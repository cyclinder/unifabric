# Topology Discovery

中文版：[topology-discovery.zh.md](./topology-discovery.zh.md)

## Overview

Topology discovery finds the links between the nodes and the switches, and between
the switches, in an AI cluster. It records the position of each node in each network
as tiered Node labels. It also aggregates the result into the read-only `Topology`
resources. Topology-aware schedulers use the labels to place the Pods of a job close
together in the network. Operators use the `Topology` resources and the Grafana
topology graph to see the whole fabric.

This page explains the topology model, the discovery modes, and the discovery
methods. For the components, see the [architecture overview](./architecture.md). For
the selection of a discovery method, the configuration, and the verification steps,
see the [network topology discovery guide](../usage/topology/README.md).

## Topology Model

Unifabric models three networks separately. One node can belong to all three. Each
network has its own Node label prefix and its own `Topology` resource:

| Network | Node label keys | `Topology` resource |
| --- | --- | --- |
| Scale-out network | `scale-out.unifabric.io/tier-1`, `tier-2`, `tier-3` | `scaleout` |
| Storage network | `storage.unifabric.io/tier-1`, `tier-2`, `tier-3` | `storage` |
| Scale-up network | `scale-up.unifabric.io/tier-1` | `scaleup` |

A performance domain is a group of nodes, and of the switches that serve them, at one
tier of one network. Tier 1 is the leaf tier, tier 2 is the spine tier, and tier 3 is
the core tier. A higher tier number means a tier closer to the top of the network. In
the scale-up network, tier 1 is the NVLink domain.

The label value names the domain. Two nodes with the same value for a label are in
the same domain at that tier, and their traffic does not go to a higher tier. For
example, two nodes with the same `scale-out.unifabric.io/tier-1` value connect to the
same leaf group. The `unifabric-roce` mode names the domains `tier<N>-group<M>`.
NVIDIA Topograph uses its own names, for example a name that comes from an
InfiniBand switch GUID. The Helm value `topoDiscovery.<network>.nodeLabel.keyTemplate`
sets the label keys.

## Discovery Modes and Methods

Each network has a discovery mode, set by the Helm value `topoDiscovery.<network>.mode`.
The networks are independent, so one cluster can use different modes for different
networks. The mode defines the topology source:

- `unifabric-roce`: the Controller calculates the topology from LLDP data. This is the
  default for the scale-out and storage networks.
- `nv-topograph`: NVIDIA Topograph supplies the topology. Topograph is an NVIDIA
  open-source service that reads the topology from vendor sources and writes the
  Node labels.
- `manual`: an administrator writes the labels. This is the default for the scale-up
  network.

Four discovery methods use these modes:

| Discovery method | Mode | Networks | Topology source |
| --- | --- | --- | --- |
| [LLDP discovery in RoCE networks](../usage/topology/roce-lldp-discovery.md) | `unifabric-roce` | Scale-out, storage | LLDP on the nodes and on the switches |
| [Custom network topology](../usage/topology/custom-network-topology.md) | `unifabric-roce` | Scale-out, storage | LLDP on the nodes, and neighbor declarations on `Switch` resources |
| [NVIDIA NetQ discovery in RoCE networks](../usage/topology/roce-netq-discovery.md) | `nv-topograph` | Scale-out, scale-up | The NetQ API, through Topograph |
| [InfiniBand network discovery](../usage/topology/infiniband-discovery.md) | `nv-topograph` | Scale-out, scale-up | `ibnetdiscover` and the GPU Operator clique label, through Topograph |

### LLDP discovery in RoCE networks

LLDP is a link-layer protocol. With LLDP, each device advertises its name and its port
to the device at the other end of each link. The Agent on each node reads the LLDP
neighbor of each RDMA NIC and writes it to the `FabricNode` resource. This gives the
links from the nodes to the leaf switches. switch-agent runs on each switch and reads
the LLDP neighbors of the switch. The Controller subscribes to each switch-agent over
mTLS gRPC and writes the neighbors to the `Switch` status. This gives the links
between the switches. The Controller combines both and calculates the leaf, spine, and
core domains.

This method needs switches that can run switch-agent, as a container or as a systemd
service. SONiC switches meet this requirement.

### Custom network topology

This method is for switches that cannot run switch-agent, for example because of a
security policy or a switch OS without a Linux shell. The switches must still send
LLDP on the ports that face the nodes. The Agents discover the leaf tier from node-side
LLDP as before. For the tiers above the leaf, an administrator lists the directly
connected switches in the `unifabric.io/switch-neighbors` annotation of the `Switch`
resources. The Controller combines the discovered leaf tier with the declared upper
tiers. The result is the same as with switch-agent.

### NVIDIA NetQ discovery in RoCE networks

NVIDIA NetQ manages Spectrum-X fabrics and already has their topology. Topograph
reads the fabric topology through the NetQ API and writes the scale-out labels. If
NetQ also manages a multi-node NVLink system, Topograph reads the NVLink domains and
writes the scale-up labels.

### InfiniBand network discovery

On an InfiniBand fabric, Topograph uses its `infiniband-k8s` provider. It runs
`ibnetdiscover` on a GPU node to read the switches, the HCAs, and the links of the IB
subnet, and writes the scale-out labels. For the scale-up network, it reads the
`nvidia.com/gpu.clique` label that the NVIDIA GPU Operator writes on each node.

### Current limitations

- The `nv-topograph` methods do not separate the storage network from the scale-out
  network. Their result is used only as the scale-out topology. For an Ethernet
  storage network, use one of the `unifabric-roce` methods. Topology discovery for an
  InfiniBand storage network is not supported yet.
- The `nv-topograph` methods do not create `Switch` resources. The Node labels and the
  `Topology` resources are complete, but the topology graph shows no switch members
  until you create the `Switch` resources.
- A switch that does not send LLDP to the nodes cannot be discovered.

For the support status of each combination of network and network device, see the
[network topology discovery guide](../usage/topology/README.md#choose-a-discovery-method-by-network).

## Discovery Results

All methods produce the same results:

- Node topology labels: the domain path of each node in each network. Topology-aware
  schedulers, such as Kueue, Volcano, and KAI Scheduler, read them directly. When the
  physical topology changes, the labels change too.
- `Switch` domain labels: in the `unifabric-roce` mode, the Controller writes the
  `unifabric.io/domain` label on each `Switch` resource. The label names the
  performance domain of the switch.
- `Topology` resources: the Controller aggregates the Node and `Switch` labels into
  the `scaleout`, `scaleup`, and `storage` resources. Each one lists the performance
  domains, their parent domains, and the domain path of each node. See the
  [Topology reference](../reference/topology.md).
- Topology visualization: the Grafana topology plugins read the `Topology` resources
  through the Topology HTTP API of the Controller. They show the three networks in one
  graph. See [topology visualization](../usage/topology/topology-visualization.md).

## Further Reading

- [Network topology discovery guide](../usage/topology/README.md): method selection,
  verification, scheduler integration, and troubleshooting.
- [Scale-out topology discovery design](../design/scaleout-topology.md): how the
  Controller calculates the scale-out domains.
- [Topology CRD design](../design/topology-crd.md): the data model of performance
  domains and node paths.
- [Topology visualization design](../design/topology-visualization.md): the Topology
  HTTP API and the Grafana plugins.
