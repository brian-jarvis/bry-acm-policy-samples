---
# finegrained-dynamic-rbac

## Description
Dynamically discovers VirtualMachines labeled with `user` on virt-enabled clusters and enforces fine-grained RBAC so each user has operator-level access to their assigned VMs. Hub-side resources aggregate the VM inventory and create per-cluster `MulticlusterRoleAssignment` objects to bind the generated ClusterRoles.

## Dependencies
- None

## Details
ACM Minimal Version: 2.16

Documentation: [latest](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/latest/html/secure_clusters/securing-cluster-intro#fine-grain-intro)

Notes:
  - Targets clusters labeled `ocp-virt=enabled` via the `ft-ocp-virt--enabled` placement
  - Hub resources are created in the `virt-rbac-hub` namespace
  - ClusterRoles are named `<vm-name>-<vm-namespace>-operator` (lowercase)
  - Placements tolerate unreachable and unavailable cluster conditions

## Implementation Details

**`finegrained-dynamic-rbac`** (managed clusters) — two ConfigurationPolicies enforced on each virt-enabled cluster:
- `record-vm-user` — creates a ConfigMap `virt-user-details` in `default`, with a key named after the ManagedCluster containing a YAML list of all VMs with a `user` label (name, namespace, user value)
- `clusterrole` — dynamically generates a `ClusterRole` per VM (named `<vm-name>-<vm-namespace>-operator`) granting get/update/patch on the VM and update on its start/stop/restart subresources

**`finegrained-dynamic-rbac-hub`** (hub only) — four ConfigurationPolicies enforced on the hub:
- `namespace-hub` — ensures the `virt-rbac-hub` namespace exists
- `clusterset-bindings` — creates a `ManagedClusterSetBinding` in `virt-rbac-hub` for each unique ClusterSet that contains a virt-enabled cluster
- `placements-hub` — creates a `Placement` named `virt-<cluster-name>` in `virt-rbac-hub` per virt-enabled cluster, selecting only that cluster; tolerates unreachable and unavailable conditions
- `mcra-virt-user` — reads each `ManagedClusterView` labeled `virt-user-details` to retrieve the per-cluster VM inventory, then creates a `MulticlusterRoleAssignment` per VM binding `<vm-name>-<vm-namespace>-operator` to the VM's user; depends on `placements-hub` being compliant
