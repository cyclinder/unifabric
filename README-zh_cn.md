# Unifabric

[English](./README.md) | 简体中文

[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](./LICENSE)
[![Release](https://img.shields.io/github/v/release/unifabric-io/unifabric)](https://github.com/unifabric-io/unifabric/releases)

> 面向 Kubernetes AI 集群的 RDMA 网络拓扑发现与可观测系统。

## 简介

AI 集群通过 RDMA 网络在 GPU 之间传输训练数据。RDMA（服务器之间绕过 CPU 拷贝的直接
内存访问）运行在 RoCE 或 InfiniBand 网络上。Kubernetes 看不见这张网络：调度器无法
判断两个节点是否连在同一台 leaf 交换机，运维人员也无法把 RDMA 流量和错误归因到产生
它们的工作负载。结果是本可避免的跨 spine 流量、更低的 GPU 利用率，以及缓慢的网络排障。

Unifabric 填补了这个空白。它自动发现集群的 RDMA 网络拓扑，以 Kubernetes 资源和
Node label 的形式发布结果，供调度器消费。它还暴露归因到 Pod 和 workload 的 RDMA
指标，并提供预置的 Grafana dashboard。

Unifabric 适用于任何具备 RDMA 网络的 Kubernetes 集群，支持通用 SONiC RoCE 网络、
NVIDIA Spectrum-X 和 InfiniBand。

## 核心特性

- 自动发现 scale-out、scale-up 和 storage 网络拓扑
- 支持通用 SONiC RoCE、NVIDIA Spectrum-X 和 InfiniBand 网络
- 生成分层 Node label，供 Kueue TAS、Volcano、KAI Scheduler 做拓扑感知调度
- RDMA 吞吐、拥塞和错误指标，归因到 Pod、namespace 和 workload
- 在 Grafana 中交互式可视化已发现的网络拓扑
- 通过 `FabricNode`、`Switch` 和 `Topology` 资源查询网络状态

## 架构

![Unifabric 架构图](./docs/images/architecture.png)

Unifabric 位于物理 RDMA 网络与调度、观测体系之间。Agent 运行在每个节点上，把 RDMA
网卡、LLDP 邻居和 RDMA Pod 归属采集到 `FabricNode` 资源。Controller 基于
`FabricNode` 和 `Switch` 数据计算拓扑，或从 NVIDIA Topograph 导入，然后写入 Node
拓扑 label 并维护只读的 `Topology` 资源。可选的 Switch Agent 运行在 SONiC 交换机上，
上报交换机侧邻居，支撑交换机间发现。详见[架构总览](./docs/concepts/architecture.zh.md)。

## 快速开始

1. 完成[基础安装](./docs/getting-started.zh.md)。
2. 按照集群物理网络对应的文档操作：
   - [通用 SONiC RoCE](./docs/getting-started-sonic-roce.zh.md)
   - [NVIDIA Spectrum-X](./docs/getting-started-spectrum-x.zh.md)
   - [InfiniBand](./docs/getting-started-infiniband.zh.md)

要了解每种网络使用哪种发现方式，参见[拓扑发现概念](./docs/concepts/topology-discovery.zh.md)。

## 文档导航

| 分类 | 链接 |
| --- | --- |
| 文档索引 | [docs/README.zh.md](./docs/README.zh.md) |
| 架构与概念 | [架构总览](./docs/concepts/architecture.zh.md)、[拓扑发现](./docs/concepts/topology-discovery.zh.md) |
| 快速开始 | [基础安装](./docs/getting-started.zh.md)、[通用 SONiC RoCE](./docs/getting-started-sonic-roce.zh.md)、[NVIDIA Spectrum-X](./docs/getting-started-spectrum-x.zh.md)、[InfiniBand](./docs/getting-started-infiniband.zh.md) |
| 功能使用 | [拓扑可视化](./docs/usage/topology-visualization.zh.md)、[RDMA 指标](./docs/usage/rdma-metrics.zh.md)、[拓扑感知调度](./docs/usage/workload-tas.zh.md)、[Switch Agent 部署](./docs/usage/switch-agent-deployment.zh.md) |
| API 参考 | [FabricNode、Switch、Topology](./docs/reference/README.zh.md) |
| Helm values | [Chart 参考](./chart/README.md) |
| 设计文档 | [docs/design/](./docs/design/) |

## 社区与贡献

欢迎参与贡献：

- 通过 [GitHub issue](https://github.com/unifabric-io/unifabric/issues) 报告 bug 或提出功能建议。
- 按照 [NVAIR 开发环境指南](./docs/development/dev-with-nvair.md)搭建开发环境。
- 提交 pull request 前，运行 `make build`、`make test-unit` 和 `make check-license`。
- 提交 commit 时使用 `git commit -s` 签署 [Developer Certificate of Origin](https://developercertificate.org/)。

## 来源

DaoCloud 于 2026-04-19 创建本项目。溯源信息见公开仓库
[unifabric-io/unifabric](https://github.com/unifabric-io/unifabric) 的历史记录。

## 许可证

Apache License 2.0，详见 [LICENSE](./LICENSE)。
