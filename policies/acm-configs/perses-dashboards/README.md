# Perses Dashboards

## Description
Deploys a set of custom `PersesDashboard` resources into the `open-cluster-management-observability`
namespace on the hub, alongside the Perses dashboards ACM ships out of the box. Each dashboard was
converted from an existing Grafana dashboard (kept in this directory as the original `*.json` source)
using the upstream `percli migrate` tool against a disposable local Perses server, then adjusted to
match this cluster's conventions:

- `apiVersion: perses.dev/v1alpha2`, with dashboard content nested under `spec.config` (the shape
  required by the installed Perses Operator CRD)
- `metadata.namespace: open-cluster-management-observability` — the same namespace ACM's own
  `PersesDashboard` resources (e.g. `k8s-apiserver`, `acm-clusters-overview`) live in
- `metadata.labels.managed-by: bry-tam-policies`
- All Prometheus queries and label-value variables reference the existing `rbac-query-proxy-datasource`
  `PersesDatasource` by name (the same datasource ACM's own dashboards use), instead of a
  dashboard-local `Datasource` template variable

Each dashboard also has a matching `ScrapeConfig` manifest (`*-metrics-scrapeconfig.yml`) that adds the
dashboard's metrics to the set MCOA federates from managed clusters up to the hub's Thanos — without
these, most panels render empty because the metrics are never forwarded past the managed cluster. See
[Implementation Details](#implementation-details) for how the metric lists were derived.

Every dashboard requires selecting a managed cluster, matching the convention used by ACM's own
built-in dashboards: a `cluster` variable (backed by `acm_managed_cluster_labels`) is the first
variable on every dashboard, every other variable's label-value query is scoped to the selected
cluster, and every panel's PromQL query is scoped to it as well. See
[Implementation Details](#implementation-details) for how this was applied.

| Dashboard | Source JSON | Resource name | ScrapeConfig |
| --- | --- | --- | --- |
| OpenShift Networking (OVN) | `ovn-dashboard.json` | `ovn-dashboard` | `ovn-metrics` |
| OpenShift Performance | `ocp-performance.json` | `ocp-performance` | `ocp-performance-metrics` |
| Node Performance | `node.json` | `node-performance` | `node-metrics` |
| etcd Cluster Info | `etcd-on-cluster-dashboard.json` | `etcd-on-cluster-dashboard` | `etcd-metrics` |
| API Performance | `api-performance-overview.json` | `api-performance-overview` | `api-performance-metrics` |

## Dependencies
  - [ACM Observability](../observability/) — requires the `MultiClusterObservability` stack and the
    `open-cluster-management-observability` namespace to exist
  - [Cluster Observability Operator](../../operators/cluster-observability/) — provides the Perses
    Operator and the `PersesDashboard` CRD
  - Requires the `rbac-query-proxy-datasource` `PersesDatasource`, which is created automatically in
    the `open-cluster-management-observability` namespace once the multicluster-observability-addon
    (MCOA) is enabled on the hub
  - [Observability Addon Config](../observability-addon-config/) — this policy watches every
    `ScrapeConfig` in `open-cluster-management-observability` and keeps the
    `multicluster-observability-addon` ClusterManagementAddOn's global metrics config in sync with
    them. The `ScrapeConfig` manifests here are picked up automatically; no changes to that policy are
    needed.

## Details
ACM Minimal Version: 2.17

Documentation: [latest](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/latest/html-single/observability/index)

Notes:
  - Deployed using the `ft-acm-observability--enabled` feature-flag placement — the same placement
    used by the [ACM Observability](../observability/) policy itself, since these dashboards only
    make sense where that stack (and its `rbac-query-proxy-datasource`) already exists
  - Each dashboard is a plain, static `PersesDashboard` manifest (no hub templating) — once applied,
    the Perses Operator reconciles it and it shows up under Observe > Dashboards (Perses) in the console
  - Converting an additional Grafana dashboard: run it through `percli migrate --online --format cr`
    against a local throwaway Perses server (e.g. `podman run -d -p 8080:8080
    docker.io/persesdev/perses:v0.54.0`), then apply the same adjustments listed above (apiVersion,
    `spec.config` nesting, namespace, label, datasource references) before adding it as a manifest here

## Implementation Details
Two recurring issues were found and fixed in the source Grafana JSON before migration (in a temporary
copy only — the original `*.json` files in this directory are untouched):
  - Collapsed Grafana `row` panels had a malformed `gridPos.y` (a nested object instead of a plain
    number), which `percli` rejects — fixed by assigning each row an incrementing numeric `y`
  - The dashboards had no top-level `uid`, which `percli`'s `--online` migration requires — fixed by
    assigning each dashboard a slug-style `uid`

After migration, `percli`'s `--format cr` output needed further adjustment to match this cluster: the
API version (`v1alpha1` → `v1alpha2`), the missing `spec.config` wrapper, the target namespace and
label, and swapping every `$Datasource` reference (and the `Datasource` template variable itself) for
a direct reference to the `rbac-query-proxy-datasource` `PersesDatasource`.

### ScrapeConfigs

MCOA does not use the `observability-metrics-(custom-)allowlist` ConfigMap mechanism from classic ACM
observability — it federates metrics from each managed cluster based on the `ScrapeConfig` objects
referenced by the `multicluster-observability-addon` ClusterManagementAddOn (see
[Observability Addon Config](../observability-addon-config/)). Each `*-metrics-scrapeconfig.yml` here
follows the same shape as ACM's own built-in ScrapeConfigs (`platform-metrics`,
`apiserver-hcp-uwl-metrics`, etc.) in that namespace: `metricsPath: /federate`, `scheme: HTTPS`,
`scrapeClass: not-configurable`, a placeholder `staticConfigs` target, and an exact-match `params.match[]`
list (`{__name__="..."}` per metric, no regex) to minimize the extra Thanos cardinality/storage cost.

The metric lists were derived by: parsing every PromQL query in the converted dashboards to extract
candidate metric names, then diffing against the full set of metric names already forwarded by ACM's 7
built-in ScrapeConfigs (165 distinct names). Only metrics not already forwarded were added. A few metric
names (e.g. `container_cpu_usage_seconds_total`) are repeated across more than one dashboard's
ScrapeConfig — that overlap is expected and harmless (duplicate `match[]` entries across separate
ScrapeConfigs don't double-scrape). When converting another dashboard in the future, repeat this
diffing process against the then-current set of forwarded metrics before writing its ScrapeConfig.

### Multicluster compatibility

The converted dashboards originally had no managed-cluster selector, unlike ACM's own built-in
dashboards. To match that convention, each dashboard gained the same `cluster` `ListVariable` ACM's
dashboards use (a `PrometheusLabelValuesVariable` against `acm_managed_cluster_labels`, via
`rbac-query-proxy-datasource`), and every other variable and panel query was scoped to it:

  - Variables whose label-value matcher already had `{...}` (e.g. `kube_node_role{role="master"}`) got
    `cluster="$cluster",` prepended inside the existing braces
  - Variables with a bare metric name and no braces (e.g. `apiserver_request_total`) got
    `{cluster="$cluster"}` appended
  - Variables with no matcher at all (e.g. api-performance-overview's `code`/`verb`/`flow_schema`/
    `priority_level`) got a matcher of `{cluster="$cluster"}`
  - `StaticListVariable`-kind variables (the `interval` variables) were left untouched — they aren't
    Prometheus-backed
  - Every PromQL panel query had `cluster="$cluster"` injected into every metric selector it
    references, including inside aggregations, vector-matching expressions, and range-vector/subquery
    syntax, while leaving function names, `by (...)`/`on (...)`-style label lists, and duration
    literals (e.g. `[5m]`, `[$interval:]`) untouched

This was done with a scripted regex-based PromQL transform (not a full parser, since PromQL tooling
isn't available in this environment) with a stoplist of aggregation operators, functions, and keywords
to avoid misidentifying them as metric names. When converting another dashboard in the future, apply
the same cluster-scoping to its variables and panel queries before committing it.
