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

| Dashboard | Source JSON | Resource name |
| --- | --- | --- |
| OpenShift Networking (OVN) | `ovn-dashboard.json` | `ovn-dashboard` |
| OpenShift Performance | `ocp-performance.json` | `ocp-performance` |
| Node Performance | `node.json` | `node-performance` |
| etcd Cluster Info | `etcd-on-cluster-dashboard.json` | `etcd-on-cluster-dashboard` |
| API Performance | `api-performance-overview.json` | `api-performance-overview` |

## Dependencies
  - [ACM Observability](../observability/) — requires the `MultiClusterObservability` stack and the
    `open-cluster-management-observability` namespace to exist
  - [Cluster Observability Operator](../../operators/cluster-observability/) — provides the Perses
    Operator and the `PersesDashboard` CRD
  - Requires the `rbac-query-proxy-datasource` `PersesDatasource`, which is created automatically in
    the `open-cluster-management-observability` namespace once the multicluster-observability-addon
    (MCOA) is enabled on the hub

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
