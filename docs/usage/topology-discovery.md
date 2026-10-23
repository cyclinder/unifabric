# Topology Discovery

中文版: [topology-discovery.zh.md](./topology-discovery.zh.md)

## Introduction

An AI cluster depends on three high-speed networks. The scale-out network carries
GPU traffic between nodes. The scale-up network connects the GPUs inside one node.
The storage network carries traffic to high-speed storage. Topology discovery
detects the links between the hosts and the switches in these three networks. It
turns the physical topology, which is spread across the network devices of many
vendors, into standard Kubernetes resources:

- Tiered Node labels. Unifabric writes the position of each node in each network,
  such as the leaf tier or the spine tier, to Node labels. Schedulers read these
  labels directly.
- The `Topology` resource. This cluster-level resource shows the performance
  domains of each network and the path of each node. Grafana renders it, and
  operators query it.

In this document you learn:

- How topology discovery and topology visualization work for three network types
  in an AI cluster: generic SONiC RoCE, NVIDIA Spectrum-X, and InfiniBand.
- How schedulers such as Kueue, Volcano, and KAI Scheduler consume the discovery
  result.

## Core Value

For the cloud-native stack, these three networks are almost a black box. The
physical links are spread across the switches, NICs, and management platforms of
many vendors. The monitoring tools of each vendor do not talk to each other, and
none of them connect to the Kubernetes resource model. This causes two direct
problems:

- The scheduler cannot see the network topology. It does not know which nodes
  connect to the same leaf switch. The Pods of one training job spread across the
  whole cluster at random. Their traffic crosses the spine tier or the core tier,
  where bandwidth is limited and traffic from other jobs interferes.
- Operators cannot see the full set of links. To find the nodes and links that a
  fault affects, they must log in to each switch. In a cluster with hundreds of
  nodes and tens of switches, this is slow and easy to get wrong.

Topology discovery solves these two problems:

- Better training and inference efficiency. Topology-aware schedulers such as
  Kueue, Volcano, and KAI Scheduler read the Node labels that Unifabric writes.
  They place the Pods of one job in the same switch domain. This shortens the
  communication path and reduces cross-tier traffic and interference.
- Faster troubleshooting. The `Topology` resource and the Grafana topology graph
  show the links of the whole network. Operators see which nodes and jobs a switch
  or a link affects, without a login to the switch.
- One topology model for all vendors. Whether the network is SONiC, Spectrum-X, or
  InfiniBand, the result uses the same label model and the same `Topology`
  resource. The scheduler configuration and the operations procedures do not
  change with the network vendor.

## Prerequisites

- Unifabric is deployed according to the [basic installation](../getting-started.md).
  The topology discovery parameters for your network are set according to its
  "Other Network Environments" section.
- In `kubectl get fabricnodes`, every node shows `READY` as `True`, and the NIC
  classification is correct. The verification section of the basic installation
  covers this step.
- Only for topology visualization: Grafana Operator and its CRDs are installed in
  the cluster.

## Supported Scenarios

Unifabric supports three topology discovery scenarios:

- Generic SONiC RoCE switches (LLDP discovery)
- NVIDIA Spectrum-X
- InfiniBand (import from NVIDIA Topograph)

Select the scenario that matches the physical network of the cluster. The scenarios
differ only in the discovery mechanism. The result is the same, and you examine it
in [View Topology and Node Labels](#view-topology-and-node-labels) and
[View Topology Visualization](#view-topology-visualization).

The following sections describe each scenario.

### Scenario 1: Generic SONiC RoCE

SONiC is an open-source network operating system that runs on white-box switches.
RoCE is the protocol that carries RDMA over Ethernet. In the generic SONiC RoCE
scenario, the scale-out network or the storage network is an Ethernet fabric of
SONiC switches, and the nodes connect with RoCE NICs. This network has no vendor
management platform that provides the topology, so Unifabric discovers it with
LLDP. LLDP is a protocol with which a device announces its identity and its port
to the device at the other end of a link.

| Item | Description |
| --- | --- |
| Networks | scale-out, storage |
| Topology source | LLDP neighbors on the nodes, plus LLDP neighbors on the switches or a declaration by the administrator |
| Helm configuration | `topoDiscovery.scaleOut.mode=unifabric-roce` (the default). The storage network also needs a NIC selector. See the [basic installation](../getting-started.md#installation). |

For the configuration and usage, see
[SONiC RoCE topology discovery](./topology/sonic-switch-onboarding.md).

### Scenario 2: NVIDIA Spectrum-X

Spectrum-X is the NVIDIA Ethernet RoCE network platform for AI clusters. It
consists of Spectrum switches and BlueField or ConnectX NICs. The NVIDIA NetQ
network management platform manages the fabric. NetQ already knows the full fabric
topology, so Unifabric does not use LLDP here. It imports the topology from NetQ
through NVIDIA Topograph. Topograph is an open-source NVIDIA service that reads the
network topology from vendor data sources and writes Node labels.

| Item | Description |
| --- | --- |
| Networks | scale-out |
| Topology source | NVIDIA Topograph reads the fabric topology through the NetQ API |
| Helm configuration | `topoDiscovery.scaleOut.mode=nv-topograph` and the NetQ credentials. See the [basic installation](../getting-started.md#spectrum-x-fabric). |

The Controller imports the result from Topograph directly and does not compute it
from LLDP. The fabric topology data in NetQ decides which tiers are discovered.
Usually the result includes the leaf tier, the spine tier, and all tiers above.

Limitations:

- This scenario creates no `Switch` resources. The `Topology` resource and the
  Node labels are complete, but in topology visualization the switch member card
  of each performance domain is empty.
- This scenario does not discover the scale-up network or the storage network.

#### Configuration

The Node labels and the `Topology` resource appear after the basic installation.
No more steps are needed. If you want topology visualization to show the switches,
create label-only `Switch` resources according to
[Add Switch Resources for Topograph Scenarios](./topology/topograph-switch-resources.md).

### Scenario 3: InfiniBand

InfiniBand is a high-speed interconnect that is independent of Ethernet. It
consists of IB switches and NICs that run in IB mode, for example Mellanox
ConnectX. The Subnet Manager keeps the topology of the whole fabric. Unifabric
discovers the topology through the `infiniband-k8s` provider of NVIDIA Topograph.
Topograph runs `ibnetdiscover` on a node to read the full IB fabric topology. It
also reads the GPU interconnect inside each node to get the scale-up domains.

| Item | Description |
| --- | --- |
| Networks | scale-out, scale-up |
| Topology source | NVIDIA Topograph. Scale-out: `ibnetdiscover` reads the IB fabric. Scale-up: the GPU interconnect topology inside the node. |
| Helm configuration | `topoDiscovery.scaleOut.mode=nv-topograph` and `topoDiscovery.scaleUp.mode=nv-topograph`. See the [basic installation](../getting-started.md#infiniband-fabric). |

Scale-out discovery finds the leaf, spine, and core tiers. The label value is a
name derived from the GUID of the IB switch, for example `S-fc6a1c03006636c0`.
Scale-up discovery finds the GPU interconnect domain inside the node and writes it
to `scale-up.unifabric.io/tier-1`.

Limitations:

- As in Scenario 2, this scenario creates no `Switch` resources, and the switch
  member cards in topology visualization are empty.
- This scenario does not discover the storage network.

#### Configuration

The same as Scenario 2: no more steps are needed. If you want topology
visualization to show the switches, see
[Add Switch Resources for Topograph Scenarios](./topology/topograph-switch-resources.md).

## Verification

### View Topology and Node Labels

This section uses a SONiC RoCE cluster as the example. `node1` and `node2` connect
to both `leaf1` and `leaf2`. `node3` and `node4` connect to both `leaf3` and
`leaf4`. The four leaf switches connect to `spine1`:

![Example SONiC RoCE topology](../images/sonic-roce-topology-example.jpg)

After discovery completes, view the Node labels:

```bash
kubectl get nodes \
  -L scale-out.unifabric.io/tier-1,scale-out.unifabric.io/tier-2
```

```text
NAME    STATUS   ROLES    AGE   VERSION   SCALE-OUT.UNIFABRIC.IO/TIER-1   SCALE-OUT.UNIFABRIC.IO/TIER-2
node1   Ready    <none>   30d   v1.32.0   tier1-group1                    tier2-group1
node2   Ready    <none>   30d   v1.32.0   tier1-group1                    tier2-group1
node3   Ready    <none>   30d   v1.32.0   tier1-group2                    tier2-group1
node4   Ready    <none>   30d   v1.32.0   tier1-group2                    tier2-group1
```

`node1` and `node2` have the same `tier-1` value, so they are in the same leaf
domain. All four nodes have the same `tier-2` value, so they share one spine
domain. If a node has `tier-1` but no `tier-2`, the tiers above the leaf were not
discovered. See [Troubleshooting](#troubleshooting).

The Node labels follow one model for SONiC RoCE, NVIDIA Spectrum-X, and
InfiniBand:

| Node label | Meaning |
| --- | --- |
| `scale-out.unifabric.io/tier-1` | The scale-out leaf switch domain that the node connects to |
| `scale-out.unifabric.io/tier-2` | The spine domain above the leaf |
| `scale-out.unifabric.io/tier-3` | The core domain above the spine |
| `scale-up.unifabric.io/tier-1` | The GPU interconnect domain inside the node, for example an NVLink domain |
| `storage.unifabric.io/tier-N` | The matching tier of the storage network. The rules are the same as for scale-out. |

A higher tier number means a tier closer to the top of the network. When two nodes
have the same value for a label, they are in the same performance domain.

The Helm value `topoDiscovery.<topology>.nodeLabel.keyTemplate` defines the label
key. The defaults are `scale-out.unifabric.io/tier-{{ .Tier }}`,
`scale-up.unifabric.io/tier-{{ .Tier }}`, and
`storage.unifabric.io/tier-{{ .Tier }}`. You can change the template, but it must
contain exactly one `{{ .Tier }}`.

> **Note:** If you change the key template, change the label keys in the scheduler
> configuration too. Otherwise the scheduler reads no topology information.

The `Topology` resource summarizes the hierarchy of the performance domains and
the domain path of each node:

```bash
kubectl get topologies.unifabric.io scaleout -o yaml
```

```yaml
apiVersion: unifabric.io/v1beta1
kind: Topology
metadata:
  name: scaleout
status:
  domains:
    - members:
        - spine1
      name: tier2-group1
      tier: 2
    - members:
        - leaf1
        - leaf2
      name: tier1-group1
      parent: tier2-group1
      tier: 1
    - members:
        - leaf3
        - leaf4
      name: tier1-group2
      parent: tier2-group1
      tier: 1
  nodes:
    - domainPath:
        - tier2-group1
        - tier1-group1
      nodes:
        - node1
        - node2
    - domainPath:
        - tier2-group1
        - tier1-group2
      nodes:
        - node3
        - node4
```

In `status.domains`, `members` lists the switches in a domain, and `parent` points
to the domain one tier above. In `status.nodes`, each group of nodes shares one
domain path from top to bottom. For the full field description, see the
[Topology reference](../reference/topology.md).

#### Use the Node Labels in a Scheduler

A topology-aware scheduler arranges the Node labels into tiers from top to bottom.
It then places the Pods of one job tier by tier. The table maps the Unifabric
labels to the configuration entry of each scheduler. For the installation and
configuration of the scheduler itself, follow its official documentation:

| Scheduler | Configuration entry | Unifabric labels to fill in |
| --- | --- | --- |
| Kueue Topology Aware Scheduling | `spec.levels` of the `Topology` resource | From top to bottom: `scale-out.unifabric.io/tier-3`, `tier-2`, `tier-1`, then `kubernetes.io/hostname` |
| Volcano network topology aware scheduling | The `HyperNode` resource | Create one HyperNode per tier. The member selector references the matching label. |
| KAI Scheduler topology aware scheduling | `spec.levels` of the `Topology` resource | The same as Kueue |

References: [Kueue TAS](https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/),
[Volcano network topology](https://volcano.sh/en/docs/network_topology_aware_scheduling/),
[KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler).

### View Topology Visualization

The Grafana topology dashboard renders the `Topology` resource as an interactive
graph. The graph matches the Node labels above. Each tier 1 domain is a leaf group.
Each tier 2 domain is a spine group above the leaf groups. Each node hangs under
its leaf group.

![Topology visualization dashboard](../images/topology-visualization.png)

For the dashboard installation, the connection to an external Grafana, and the
details of the graph, see
[Topology Visualization](./topology/topology-visualization.md).

## Troubleshooting

### A Node Has No Topology Labels

First make sure that the prerequisites are met:

1. Make sure that the NICs in the `FabricNode` received LLDP neighbors:

   ```bash
   kubectl get fabricnode node1 -o jsonpath='{range .status.scaleOutNics[*]}{.name}{"\t"}{.lldpNeighbor.hostname}{"\t"}{.lldpNeighbor.port}{"\n"}{end}'
   ```

   ```text
   eth1	leaf1	Ethernet0
   eth2	leaf2	Ethernet0
   ```

   If the hostname column is empty, make sure that LLDP is enabled on the switch
   and that the switch port of the node NIC sends LLDP frames. The
   `LLDPNeighborsReady` condition of the `FabricNode` also reports the reason
   `LLDPNeighborMissing`.

2. In the SONiC RoCE scenario, make sure that `topoDiscovery.scaleOut.mode` is
   `unifabric-roce`. In the Topograph scenarios, make sure that `mode` is
   `nv-topograph` and that the Topograph Deployment runs:

   ```bash
   kubectl -n unifabric-system get deployment unifabric-nvidia-topograph
   kubectl -n unifabric-system logs deployment/unifabric-nvidia-topograph --tail 100
   ```

   In the Spectrum-X scenario, examine the NetQ credentials and the API URL. In
   the InfiniBand scenario, make sure that the log shows that `ibnetdiscover`
   runs.

3. Read the Controller log:

   ```bash
   kubectl -n unifabric-system logs deployment/unifabric-controller --tail 200
   ```

### Node Labels Have Only Tier 1

LLDP on the nodes discovers only the leaf tier. This symptom means that the
information above the leaf tier did not reach the Controller:

- If the switches run switch-agent: make sure that every switch has a `Switch`
  resource and that `HEALTHY` is `true` in `kubectl get switch -o wide`. If it is
  `false`, make sure that the node that runs the Controller can reach port `8090`
  of the switch and that the mTLS certificates match.
- If the switches do not run switch-agent: make sure that a `Switch` resource
  exists for each switch at the spine tier and above. Make sure that the
  `unifabric.io/switch-neighbors` annotation is a quoted JSON array of strings.
  Each name in the array must match a leaf that the nodes discovered, or another
  `Switch` resource.
- If the physical network has only a leaf tier, tier 1 alone is the expected
  result.

For the detailed steps for both forms, see
[SONiC switch onboarding](./topology/sonic-switch-onboarding.md).

### Switch Members Are Empty in Topology Visualization

This is the expected behavior in the Topograph scenarios (Spectrum-X and
InfiniBand). See
[Add Switch Resources for Topograph Scenarios](./topology/topograph-switch-resources.md).

## References

- [Topology discovery concepts](../concepts/topology-discovery.md): logical
  topology, discovery methods, and discovery results.
- [SONiC switch onboarding](./topology/sonic-switch-onboarding.md): switch-agent
  deployment and `Switch` resource configuration.
- [Add Switch Resources for Topograph Scenarios](./topology/topograph-switch-resources.md):
  the optional step for the Spectrum-X and InfiniBand scenarios.
- [Topology Visualization](./topology/topology-visualization.md): installation and
  reading of the Grafana dashboard.
- [Topology reference](../reference/topology.md),
  [FabricNode reference](../reference/fabricnode.md),
  [Switch reference](../reference/switch.md).
- [Scale-out topology discovery design](../design/scaleout-topology.md),
  [Topology CRD design](../design/topology-crd.md).
- [Helm values reference](../../chart/README.md): all `topoDiscovery.*`
  parameters.
