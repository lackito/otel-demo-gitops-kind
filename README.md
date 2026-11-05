# otel-demo-gitops-kind

GitOps desired state for the OpenTelemetry Demo running on the local kind
cluster. This repository is the local counterpart to `otel-demo-gitops-aws`, which
remains dedicated to AWS.

See the
[AWS and local project walkthrough](https://github.com/lackito/otel-demo-infra-kind/blob/main/docs/PROJECT_WALKTHROUGH.md)
for the shared source-to-Argo CD delivery model.

## Repository structure

```text
otel-demo-gitops-kind/
├── argocd/
│   └── applications/
│       └── otel-demo.yaml
│
└── applications/
    └── otel-demo/
        ├── values.yaml
        └── manifests/
            ├── gateway.yaml
            └── httproute.yaml
```

The local Terraform source repository, `otel-demo-infra-kind`, installs Argo CD in
its platform stage and registers an Application from its separate applications
stage. That Application watches `applications/otel-demo` on `main`. Argo CD
combines the pinned upstream OpenTelemetry Demo Helm chart with this
repository's values and local Gateway API resources.

## Responsibilities

This repository owns:

- the declarative local Argo CD Application definition;
- local Helm value overrides;
- the local `Gateway` and `HTTPRoute`;
- verified Recommendation, Product Catalog, and Ad image selection.

It does not own the kind cluster, Argo CD installation, NGINX Gateway Fabric,
application source, or AWS desired state.

## Delivery flow

Pushes to the `kind` branch of `otel-demo-apps` publish per-service ARM64
images to GHCR and update `applications/otel-demo/values.yaml` here. Local Argo
CD then detects the commit and reconciles the kind cluster.

The GitHub repository must be public for anonymous Argo CD access, or private
repository credentials must be configured in the local Argo CD installation.

## Local telemetry

Traces are exported to Jaeger, metrics to Prometheus, and application logs to
OpenSearch. In Grafana, open Explore and choose the OpenSearch data source to
search `otel-logs-*`. The local data source uses a daily index pattern so the
Grafana plugin resolves concrete index names for health checks. All three data sources are provisioned together from local values so the
upstream wildcard definition cannot overwrite it. The Collector also prints
basic batch summaries.

OpenSearch runs as one local node with zero replicas, a 400 MiB Java heap, a 768 MiB memory
request, and a 1,100 MiB memory limit. Storage is ephemeral: recreating its pod
removes indexed logs. This is a local demo configuration with the security plugin
disabled and no external OpenSearch ingress. A small Argo CD PostSync job applies
a zero-replica template to future log indices and updates existing log indices.
This keeps the log index healthy on one node without allocating a second node.

To inspect the OpenSearch API locally:

```sh
kubectl --context kind-otel-demo-kind -n opentelemetry-demo   port-forward svc/opensearch 9200:9200
```

In another terminal:

```sh
curl 'http://localhost:9200/_cluster/health?pretty'
curl 'http://localhost:9200/_cat/indices/otel-logs-*?v'
curl 'http://localhost:9200/otel-logs-*/_search?size=1&pretty'
```

Application container logs also remain available through `kubectl logs`.

## Frontend routing

The frontend keeps upstream HTTP connections alive for 65 minutes, longer than
Envoy's default one-hour idle timeout. This avoids reusing connections that
Next.js has already closed. The standalone frontend image reads the
`KEEP_ALIVE_TIMEOUT` environment variable in milliseconds.

The flag editor remains disabled locally. Requests to `/feature` and
`/feature/` are served by the existing image-provider as an explanatory page,
rather than being forwarded to the absent editor. The flagd evaluation service
and `/flagservice/` route remain available.

## Environment naming

See the shared [naming and migration guide](https://github.com/lackito/otel-demo-apps/blob/main/docs/NAMING.md) for repository names, runtime identities, and the coordinated rename sequence.

The three pinned `otel-demo-local-*` GHCR references are verified bootstrap
artifacts from before the naming migration. They stay usable until the kind
release workflows publish public `otel-demo-kind-*` packages and promote their
verified full-SHA images. Repository renames do not rename registry packages.
