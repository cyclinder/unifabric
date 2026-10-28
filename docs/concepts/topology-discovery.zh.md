# 拓扑发现

English version: [topology-discovery.md](./topology-discovery.md)

## 概述

拓扑发现探测 AI 集群中节点与交换机、交换机与交换机之间的连接关系。它把每个节点在每张
网络中的位置记录为分层 Node label，并把结果聚合为只读的 `Topology` 资源。拓扑感知调度器
根据 label 把同一个任务的 Pod 放在网络中相近的位置。运维人员通过 `Topology` 资源和
Grafana 拓扑图查看整个网络。

本文介绍拓扑模型、发现模式和发现方式。组件说明参见[架构总览](./architecture.zh.md)。
发现方式的选择、配置和验证步骤，参见[网络拓扑发现指南](../usage/topology/README.zh.md)。

## 拓扑模型

Unifabric 对三张网络分别建模。一个节点可以同时属于三张网络。每张网络有自己的 Node label
前缀和 `Topology` 资源：

| 网络 | Node label key | `Topology` 资源 |
| --- | --- | --- |
| Scale-out 网络 | `scale-out.unifabric.io/tier-1`、`tier-2`、`tier-3` | `scaleout` |
| 存储网络 | `storage.unifabric.io/tier-1`、`tier-2`、`tier-3` | `storage` |
| Scale-up 网络 | `scale-up.unifabric.io/tier-1` | `scaleup` |

性能域是同一张网络中某一层级上的一组节点，以及为它们服务的交换机。tier 1 是 leaf 层，
tier 2 是 spine 层，tier 3 是 core 层。层级编号越大，越靠近网络顶层。在 Scale-up 网络中，
tier 1 是 NVLink 域。

label 的值是域的名称。两个节点某个 label 的值相同，表示它们在该层级属于同一个域，
彼此之间的流量不经过更高层级。例如，`scale-out.unifabric.io/tier-1` 值相同的两个节点
连接到同一个 leaf 组。`unifabric-roce` 模式把域命名为 `tier<N>-group<M>`。NVIDIA Topograph
使用自己的命名，例如基于 InfiniBand 交换机 GUID 生成的名称。Helm 值
`topoDiscovery.<network>.nodeLabel.keyTemplate` 决定 label key。

## 发现模式与发现方式

每张网络有一个发现模式，由 Helm 值 `topoDiscovery.<network>.mode` 设置。各网络相互独立，
同一集群的不同网络可以使用不同模式。模式决定拓扑来源：

- `unifabric-roce`：Controller 基于 LLDP 数据计算拓扑。这是 Scale-out 网络和存储网络的默认值。
- `nv-topograph`：由 NVIDIA Topograph 提供拓扑。Topograph 是 NVIDIA 开源的服务，它从厂商
  数据源读取拓扑并写入 Node label。
- `manual`：由管理员写入 label。这是 Scale-up 网络的默认值。

以下四种发现方式基于这些模式：

| 发现方式 | 模式 | 网络 | 拓扑来源 |
| --- | --- | --- | --- |
| [RoCE 网络中 LLDP 发现](../usage/topology/roce-lldp-discovery.zh.md) | `unifabric-roce` | Scale-out、存储 | 节点和交换机上的 LLDP |
| [自定义网络拓扑](../usage/topology/custom-network-topology.zh.md) | `unifabric-roce` | Scale-out、存储 | 节点上的 LLDP，以及 `Switch` 资源上的邻居声明 |
| [RoCE 网络中 NVIDIA NetQ 发现](../usage/topology/roce-netq-discovery.zh.md) | `nv-topograph` | Scale-out、Scale-up | 经 Topograph 读取的 NetQ API |
| [InfiniBand 网络发现](../usage/topology/infiniband-discovery.zh.md) | `nv-topograph` | Scale-out、Scale-up | 经 Topograph 读取的 `ibnetdiscover` 结果和 GPU Operator clique label |

### RoCE 网络中 LLDP 发现

LLDP 是一种链路层协议。设备通过 LLDP 向每条链路对端的设备通告自己的名称和端口。每个
节点上的 Agent 读取每块 RDMA 网卡的 LLDP 邻居，写入 `FabricNode` 资源，由此得到节点到
leaf 交换机的链路。switch-agent 运行在每台交换机上，读取交换机的 LLDP 邻居。Controller
通过 mTLS gRPC 订阅每个 switch-agent，并把邻居写入 `Switch` status，由此得到交换机之间的
链路。Controller 合并两者，计算 leaf、spine 和 core 域。

该方式要求交换机能以容器或 systemd 服务的形式运行 switch-agent。SONiC 交换机满足这一要求。

### 自定义网络拓扑

该方式适用于无法运行 switch-agent 的交换机，例如受安全策略限制，或交换机操作系统没有
Linux shell。交换机仍需在面向节点的端口上发送 LLDP。Agent 照常通过节点侧 LLDP 发现 leaf 层。
对于 leaf 以上的层级，由管理员在 `Switch` 资源的 `unifabric.io/switch-neighbors` annotation
中列出直连的交换机。Controller 把发现的 leaf 层与声明的上层合并，结果与使用 switch-agent 时相同。

### RoCE 网络中 NVIDIA NetQ 发现

NVIDIA NetQ 管理 Spectrum-X 网络，已经掌握其拓扑。Topograph 通过 NetQ API 读取 fabric
拓扑，并写入 Scale-out label。如果 NetQ 同时管理多节点 NVLink 系统，Topograph 还会读取
NVLink 域并写入 Scale-up label。

### InfiniBand 网络发现

在 InfiniBand 网络中，Topograph 使用 `infiniband-k8s` provider。它在一个 GPU 节点上运行
`ibnetdiscover`，读取 IB 子网中的交换机、HCA 和链路，并写入 Scale-out label。对于 Scale-up
网络，它读取 NVIDIA GPU Operator 写在每个节点上的 `nvidia.com/gpu.clique` label。

### 当前限制

- `nv-topograph` 方式不区分存储网络和 Scale-out 网络，结果只作为 Scale-out 拓扑使用。
  以太网存储网络请使用某种 `unifabric-roce` 方式。InfiniBand 存储网络暂不支持拓扑发现。
- `nv-topograph` 方式不创建 `Switch` 资源。Node label 和 `Topology` 资源是完整的，但在创建
  `Switch` 资源之前，拓扑图中不显示交换机成员。
- 不向节点发送 LLDP 的交换机无法被发现。

各网络与网络设备组合的支持情况，参见
[网络拓扑发现指南](../usage/topology/README.zh.md#按网络选择发现方式)。

## 发现结果

所有发现方式产生相同的结果：

- Node 拓扑 label：每个节点在每张网络中的域路径。Kueue、Volcano、KAI Scheduler 等拓扑
  感知调度器直接读取。物理拓扑变化时，label 随之更新。
- `Switch` 域 label：在 `unifabric-roce` 模式下，Controller 在每个 `Switch` 资源上写入
  `unifabric.io/domain` label，标明交换机所在的性能域。
- `Topology` 资源：Controller 把 Node 和 `Switch` 的 label 聚合为 `scaleout`、`scaleup` 和
  `storage` 三个资源。每个资源列出性能域、各域的父域，以及每个节点的域路径。参见
  [Topology 参考](../reference/topology.zh.md)。
- 拓扑可视化：Grafana 拓扑插件通过 Controller 的 Topology HTTP API 读取 `Topology` 资源，
  在一张图中展示三张网络。参见[拓扑可视化](../usage/topology/topology-visualization.zh.md)。

## 延伸阅读

- [网络拓扑发现指南](../usage/topology/README.zh.md)：发现方式选择、验证、调度器集成与故障排查。
- [Scale-out 拓扑发现设计](../design/scaleout-topology.zh.md)：Controller 如何计算 Scale-out 域。
- [Topology CRD 设计](../design/topology-crd.zh.md)：性能域与节点路径的数据模型。
- [拓扑可视化设计](../design/topology-visualization.zh.md)：Topology HTTP API 与 Grafana 插件。
