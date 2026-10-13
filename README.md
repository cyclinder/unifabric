# Unifabric

English | [简体中文](./README-zh_cn.md)

[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](./LICENSE)
[![Release](https://img.shields.io/github/v/release/unifabric-io/unifabric)](https://github.com/unifabric-io/unifabric/releases)

> RDMA network topology discovery and observability for Kubernetes AI clusters.

## Introduction

AI clusters move training data between GPUs over RDMA networks. RDMA (direct memory
access between servers, without CPU copies) runs on RoCE or InfiniBand fabrics.
Kubernetes has no view of this network. The scheduler cannot tell whether two nodes
share a leaf switch. Operators cannot attribute RDMA traffic or errors to the
workloads that cause them. The result is avoidable cross-spine traffic, lower GPU
utilization, and slow network troubleshooting.

Unifabric closes this gap. It discovers the RDMA network topology of the cluster
automatically. It publishes the result as Kubernetes resources and as labels on each
Node, which schedulers consume. It also exposes RDMA metrics attributed to Pods and
workloads, with prebuilt Grafana dashboards.

Unifabric fits any Kubernetes cluster with an RDMA fabric. It supports general SONiC
RoCE networks, NVIDIA Spectrum-X, and InfiniBand.

## Core Features

- Automatic discovery of scale-out, scale-up, and storage network topologies
- Support for general SONiC RoCE, NVIDIA Spectrum-X, and InfiniBand fabrics
- Tiered Node labels that Kueue TAS, Volcano, and KAI Scheduler consume for topology-aware placement
- RDMA throughput, congestion, and error metrics attributed to Pods, namespaces, and workloads
- Interactive visualization of the discovered fabric in Grafana
- Queryable network state through the `FabricNode`, `Switch`, and `Topology` resources

## Architecture

![Unifabric architecture](./docs/images/architecture.png)

Unifabric sits between the physical RDMA network and the scheduling and observability
stack. An Agent runs on every node and collects RDMA NICs, LLDP neighbors, and RDMA
Pod attribution into `FabricNode` resources. The Controller computes the topology from
`FabricNode` and `Switch` data, or imports it from NVIDIA Topograph. It then writes
Node topology labels and maintains the read-only `Topology` resource. An optional
Switch Agent runs on SONiC switches and reports switch-side neighbors for
switch-to-switch discovery. For details, see the
[architecture overview](./docs/concepts/architecture.md).

## Quick Start

1. Complete the [basic installation](./docs/getting-started.md).
2. Follow the guide for your physical network:
   - [General SONiC RoCE](./docs/getting-started-sonic-roce.md)
   - [NVIDIA Spectrum-X](./docs/getting-started-spectrum-x.md)
   - [InfiniBand](./docs/getting-started-infiniband.md)

To understand which discovery mode each network uses, see the
[topology discovery concept](./docs/concepts/topology-discovery.md).

## Documentation

| Category | Link |
| --- | --- |
| Documentation index | [docs/README.md](./docs/README.md) |
| Architecture and concepts | [Architecture overview](./docs/concepts/architecture.md), [Topology discovery](./docs/concepts/topology-discovery.md) |
| Getting started | [Basic installation](./docs/getting-started.md), [General SONiC RoCE](./docs/getting-started-sonic-roce.md), [NVIDIA Spectrum-X](./docs/getting-started-spectrum-x.md), [InfiniBand](./docs/getting-started-infiniband.md) |
| Feature usage | [Topology visualization](./docs/usage/topology-visualization.md), [RDMA metrics](./docs/usage/rdma-metrics.md), [Workload TAS](./docs/usage/workload-tas.md), [Switch Agent deployment](./docs/usage/switch-agent-deployment.md) |
| API reference | [FabricNode, Switch, and Topology](./docs/reference/README.md) |
| Helm values | [Chart reference](./chart/README.md) |
| Design documents | [docs/design/](./docs/design/) |

## Community and Contributing

Contributions are welcome:

- Report bugs or propose features through [GitHub issues](https://github.com/unifabric-io/unifabric/issues).
- Set up a development environment with the [NVAIR development guide](./docs/development/dev-with-nvair.md).
- Before you submit a pull request, run `make build`, `make test-unit`, and `make check-license`.
- Sign off your commits with `git commit -s` to certify the [Developer Certificate of Origin](https://developercertificate.org/).

## Origin

DaoCloud created this project on 2026-04-19. See the public repository history at
[unifabric-io/unifabric](https://github.com/unifabric-io/unifabric) for provenance.

## License

Apache License 2.0. See [LICENSE](./LICENSE).
