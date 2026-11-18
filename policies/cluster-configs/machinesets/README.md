#  MachineSets

## Description
Creates dedicated `MachineSet` resources for infra and storage node roles, scaling out nodes
beyond the default worker pool for workload isolation.

## Dependencies
  - None

## Details
ACM Minimal Version: 2.12

Documentation: [latest](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/machine_management/index#machinesets-intro)

Notes:
  - `infra-machineset` is applied to clusters labeled `inframachines=enabled` via `ft-inframachines--enabled`
  - `storage-machineset` is applied to clusters labeled `storage=odf` via `ft-storage--odf`
  - `infra-machineset` and `storage-machineset` are AWS-specific; the templates read region and infrastructure ID from the cluster `Infrastructure` object
  - `vsphere-static-machinesets` is applied to clusters labeled `vsphere-static-machinesets=enabled` via `ft-vsphere-static-machinesets--enabled`
  - `vsphere-static-machinesets` requires a cluster on vSphere with at least one existing `worker` `Machine`, and the node data in the `bry-tam.io/vsphere-nodes` annotation of the `ManagedCluster`

## Implementation Details
**`infra-machineset`** — creates one `MachineSet` per availability zone (a, b, c) in the cluster's
AWS region. The template reads the cluster `Infrastructure` object to determine the region and
infrastructure ID, then copies and merges the `providerSpec` from an existing worker `MachineSet`
as a baseline. Each infra `MachineSet` uses:
  - Instance type: `c6i.8xlarge`
  - Root volume: 200 GiB `io1` encrypted EBS
  - Node label: `node-role.kubernetes.io/infra: ""`
  - Taint: `node-role.kubernetes.io/infra: NoSchedule`

**`storage-machineset`** — follows the same template pattern as the infra `MachineSet` but uses the
`storage` role. Storage nodes carry both the infra and storage role labels so they receive infra
`MachineConfig` resources and are excluded from the infra `MachineConfigPool`. Each storage
`MachineSet` uses:
  - Instance type: `c6i.8xlarge`
  - Root volume: 200 GiB `io1` encrypted EBS
  - Node labels: `node-role.kubernetes.io/infra: ""`, `node-role.kubernetes.io/storage: ""`,
    `cluster.ocs.openshift.io/openshift-storage: ""`
  - Taint: `node.ocs.openshift.io/storage: NoSchedule`

**`vsphere-static-machinesets`** — creates one `MachineSet` per node listed in the `bry-tam.io/vsphere-nodes`
annotation of the cluster's `ManagedCluster`. Each `MachineSet` has `replicas: 1` and the static address is set
directly on the `providerSpec` (`network.devices[].ipAddrs`, `gateway`, `nameservers`), so no `IPPool` or
`IPAddressClaim` is needed. The hub template reads the annotation, and the managed cluster template then builds the
`providerSpec` from an existing `worker` `Machine`. A `Machine` is used as the baseline, not a `MachineSet`, because
workers created with static IPs from the `install-config.yaml` are standalone `Machine` objects. vCenter details
(`workspace`, `template`, `credentialsSecret`, `userDataSecret`) are inherited from that `Machine`; the
`addressesFromPools` network section is replaced.

Annotation format (JSON string):
```json
{
  "network": {
    "gateway": "10.0.250.1",
    "prefixLength": 24,
    "nameservers": ["10.0.107.102"],
    "networkName": "vlan250"
  },
  "nodes": [
    {
      "name": "infra-0",
      "role": "infra",
      "ipAddress": "10.0.250.201",
      "cpus": 8,
      "coresPerSocket": 2,
      "memoryMiB": 32768,
      "diskGiB": 200,
      "labels": {"example.com/zone": "a"},
      "taints": [{"key": "node-role.kubernetes.io/infra", "effect": "NoSchedule"}]
    }
  ]
}
```
  - `network.gateway` and `network.prefixLength` are required; `nameservers` and `networkName` default to the values on the worker `Machine`
  - `nodes[].name` and `nodes[].ipAddress` are required; `name` becomes the `MachineSet` name `<infrastructure-id>-<name>`
  - `role` defaults to `worker`; `cpus`, `coresPerSocket`, `memoryMiB` and `diskGiB` default to the worker `Machine` values; `labels` and `taints` are optional
  - Each node is its own `MachineSet` because a static address can only belong to one `Machine`, never scale a static IP `MachineSet` above 1

Set the annotation on the hub:
```
oc annotate managedcluster <cluster-name> --overwrite bry-tam.io/vsphere-nodes='{"network":{...},"nodes":[...]}'
```
