# otel-demo-gitops-kind

Local kind desired state for the
[OpenTelemetry Demo](https://opentelemetry.io/docs/demo/). Argo CD combines the
upstream chart with these values and Gateway API manifests.

## Files and ownership

| Path | Purpose |
| --- | --- |
| [Application](argocd/applications/otel-demo.yaml) | Chart version, sources, destination, and sync policy |
| [Values](applications/otel-demo/values.yaml) | Local resources, telemetry, and Ad/Product Catalog/Recommendation images |
| [Manifests](applications/otel-demo/manifests/) | Gateway, HTTPRoute, and OpenSearch log-index settings |

[otel-demo-infra-kind](https://github.com/lackito/otel-demo-infra-kind) creates
the cluster, installs the platform, and registers the Application through
Terraform. Keep its registration and the declarative Application YAML aligned
when changing chart sources or sync settings. This repository owns local
application configuration; source and releases live in
[otel-demo-apps](https://github.com/lackito/otel-demo-apps).

## Releases

Qualifying changes on `otel-demo-apps/kind` trigger per-service ARM64 builds.
Each workflow publishes `ghcr.io/lackito/otel-demo-kind-<service>:<full-sha>`,
verifies public access, and updates that service's image override here.
Argo CD watches `main` and reconciles Application `otel-demo-kind`.

All three custom services use these GHCR packages: Ad, Product Catalog, and
Recommendation. The repository must be public for anonymous Argo CD access, or
Argo CD must have repository credentials. Images must also be accessible.

From `otel-demo-infra-kind`, run `make application-validate` and compare its
three service image references with their release SHAs. Check running platform
digests against the corresponding image-index members when comparing image IDs.
See [local operations](https://github.com/lackito/otel-demo-infra-kind/blob/main/docs/OPERATIONS.md)
for setup, validation, browser access, and teardown.

## Telemetry

Traces go to Jaeger, metrics to Prometheus, and logs to OpenSearch. Grafana
provisions all three data sources together. In Explore, select OpenSearch and
search `otel-logs-*`, filtering by service. The data source uses daily indices.

OpenSearch runs as one node with zero log-index replicas, a 400 MiB heap, a
768 MiB request, and a 1,100 MiB limit. A PostSync job applies zero replicas to
current and future log indices. Security is disabled for this internal local
demo, with no external OpenSearch ingress. Storage is ephemeral: recreating the
pod removes indexed logs.

For direct index checks, keep this running:

```bash
kubectl --context kind-otel-demo -n opentelemetry-demo port-forward svc/opensearch 9200:9200
```

In another terminal:

```bash
curl 'http://localhost:9200/_cluster/health?pretty'
curl 'http://localhost:9200/_cat/indices/otel-logs-*?v'
curl 'http://localhost:9200/otel-logs-*/_search?size=1&pretty'
```

## Local settings

The frontend keeps upstream HTTP connections alive for 65 minutes to exceed
Envoy's one-hour idle timeout. `KEEP_ALIVE_TIMEOUT` is in milliseconds.

The optional flag editor is disabled locally. `/feature` and `/feature/` show an
explanatory page from image-provider. Flagd still evaluates GitOps-managed flags,
and `/flagservice/` remains available.

See [Names and repositories](https://github.com/lackito/otel-demo-apps/blob/main/docs/NAMING.md)
for the shared naming reference and
[Branching](https://github.com/lackito/otel-demo-apps/blob/main/docs/BRANCHING.md)
for the application development routine.
