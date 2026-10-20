# Unifabric Documentation

中文版：[README.zh.md](./README.zh.md)

Unifabric turns RDMA network hierarchy, node proximity, and NIC state in Kubernetes AI clusters
into data that Kubernetes can query and consume:

- Agents collect node RDMA NICs, LLDP neighbors, and Pod network attribution.
- The Controller or NVIDIA Topograph discovers topology and writes the result to Node labels.
- `FabricNode`, `Switch`, and `Topology` CRs expose cluster-level network state.
- Prometheus metrics and Grafana dashboards expose RDMA traffic, congestion, errors, and Pod
  attribution.

Schedulers can use these results to place communication-intensive AI workloads on nearby Nodes,
while operators can diagnose RDMA network and workload traffic problems.

## Quick Start

Follow [Deploy Unifabric](./getting-started.md) to install the Controller, the
Agent, and the observability components. The default installation works on a
SONiC RoCE fabric and discovers the topology over LLDP. For an InfiniBand or
Spectrum-X fabric, the guide lists the Helm options to change:

| Scenario | Primary topology | Discovery method |
| --- | --- | --- |
| SONiC RoCE (default) | Scale-out and RoCE storage | Node and switch topology discovery over LLDP |
| InfiniBand | Scale-out and Scale-up | NVIDIA Topograph |
| Spectrum-X | Scale-out | NVIDIA Topograph with NetQ |

For the SONiC switch onboarding options, the label semantics, and discovery
troubleshooting, see the [topology discovery guide](./usage/topology-discovery.md).
For other Chart parameters, see the defaults and descriptions in the
[Helm values reference](../chart/README.md).

## Core Features

### Topology Discovery

Topology discovery converts physical adjacency into hierarchical performance domains and writes
them to Kubernetes Node labels. Unifabric models three logical topologies:

- **Scale-out network:** Node-to-node communication through leaf, spine, and core tiers. See the
  [topology discovery guide](./usage/topology-discovery.md) for configuration and label semantics.
- **Scale-up network:** High-bandwidth GPU interconnect domains such as NVLink and NVSwitch.
  Currently, only the InfiniBand scenario writes scale-up labels. See the
  [topology discovery guide](./usage/topology-discovery.md) for configuration.
- **Storage network:** A dedicated RDMA network between compute Nodes and storage services.
  Currently, only the SONiC RoCE discovery method supports it. See the
  [topology discovery guide](./usage/topology-discovery.md) for configuration.

Discovery results are exposed to schedulers as Node labels and aggregated into read-only
`Topology` CRs. Use the
[FabricNode, Switch, and Topology API reference](./reference/README.md) to query the results.

### Topology-Aware Scheduling

Kueue, Volcano, KAI Scheduler, and similar systems can consume the discovered Node labels and place
Pods from one workload in the same or nearby performance domains, reducing traffic across switch
tiers.

- [Topology-aware scheduling and Kueue TAS example](./usage/workload-tas.md)
- [Topology API reference](./reference/topology.md)

### RDMA Traffic and Health Observability

Unifabric Agents collect RDMA device, port, priority, throughput, congestion, and error metrics and
attribute traffic to Pods, namespaces, and top-level workloads. Grafana dashboards help operators
diagnose RDMA traffic and health by cluster, Node, and workload.

- [RDMA observability usage guide](./usage/rdma-metrics.md)
- [eBPF RDMA flow attribution usage guide](./usage/rdma-flow.md): attribute RDMA sends to
  Pod-to-Pod flows, publish `RDMAEndpoint`, and store per-send events.
- [Topology visualization usage guide](./usage/topology-visualization.md)

## Development

- [NVAIR development guide](./development/dev-with-nvair.md): Build a local topology and end-to-end
  development environment.

## Design

- [Unifabric API reference](./reference/README.md): Query `FabricNode`, `Switch`, `Topology`, and
  `RDMAEndpoint`.
- [Topology CRD design](./design/topology-crd.md): Understand performance domains, Node paths, and
  the topology data model.
- [Scale-out topology discovery design](./design/scaleout-topology.md): Understand how Scale-out
  topologies are discovered and constructed.
- [RDMA metric model and Pod attribution design](./design/rdma-metrics.md): Understand metric
  definitions and workload attribution.
- [eBPF RDMA flow attribution design](./design/rdma-flow.md): Understand how `libibverbs`
  uprobes, `RDMAEndpoint` and the send event pipeline attribute traffic between Pods.
- [Topology visualization design](./design/topology-visualization.md): Understand the Topology
  HTTP API and the Grafana datasource/panel plugins that render it.
