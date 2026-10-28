# 高速网络流量观测

English version: [README.md](./README.md)

## 简介

Unifabric 高速网络流量观测采集 AI 应用在高速网络上的通信流量。
它把流量关联到 Pod、namespace 和顶层 workload。
观测范围覆盖 NVLink 域内 GPU 之间的 NVLink 通信,以及 Pod 经过 RDMA 网卡和交换机的跨节点通信。
结果以 Prometheus 指标、`RDMAEndpoint` 资源和 Grafana 仪表盘的形式提供。

通过本文你将学习到:

- 流量观测覆盖通信路径上的哪些位置,以及各位置当前是否支持。
- 各位置的数据如何关联到同一个 Pod 身份。
- 各观测方式的配置和验证步骤在哪里查看。

## 背景

AI 训练任务和推理任务的本质是 GPU 之间的通信。
一个任务的 Pod 运行在多个节点上。
同一个 NVLink 域内的 GPU 通过 NVLink 通信。
NVLink 域之外的 GPU 通信使用 RDMA,经过 Pod 的 RDMA 网卡、主机的 RDMA 网卡和交换机。
Pod 读写模型、数据集和 KV Cache 的流量也经过 RDMA 网卡和交换机。
这条路径上任何一处拥塞或出错,都会拖慢整个任务。

但这条路径对云原生体系几乎是黑盒:

- 常规的网络观测手段失效。

  RDMA 绕过内核协议栈,由网卡直接收发数据。
  抓包、连接跟踪和基于内核协议栈的工具只能看到 TCP/IP 流量,看不到 RDMA 流量。
  网卡上只剩设备级的累计计数器。

- 各处的数据互不相通,也没有业务身份。

  交换机的计数器在网络管理平台中,主机网卡的计数器在主机的 sysfs 中。
  Pod 使用的 SR-IOV VF 位于 Pod 的网络命名空间中,主机上的监控工具看不到它。
  host 模式的 Pod 共用主机网卡,计数器无法区分是哪个 Pod 的流量。
  这些数据只记录设备,例如 `mlx5_0` 或交换机端口,不记录 Pod。
  Kubernetes 认识 Pod,却看不到这些硬件。
  这就是云原生体系和高速网络之间的"最后一公里"问题。

- 通信关系是黑盒。

  即使知道一个 Pod 发送了多少流量,也不知道流量发给了谁。
  任务变慢时,运维无法回答"是谁的流量占用了带宽,影响了哪些任务"。
  性能调优时,也无法确认通信的对端、时序和消息大小是否符合预期。

Unifabric 的高速网络流量观测功能尝试解决这些问题：
它在通信路径的各个位置采集流量,并把每一处的数据都关联到同一个 Pod 身份上。
运维可以从一个 workload 出发,查到它的 Pod 和谁通信、经过了哪些网卡、这些网卡上有没有拥塞和错误。
这样就能判断任务变慢是否由网络引起,以及是谁的流量影响了谁。

## 前提条件

- 已按[基础安装](../../getting-started.zh.md)完成 Unifabric 部署。
- 集群已安装 Prometheus Operator。
- 仅 Grafana 仪表盘需要: 集群已安装 Grafana Operator (v5 及以上) 或 Grafana sidecar。

各观测方式的其他要求,参见对应的文档。

## 流量观测范围

下图是 AI 应用的通信路径,以及流量观测在路径上的观测位置。
同一个 NVLink 域内的 GPU 通过 NVLink 通信,NVLink 域可以跨越多个节点。
经过 RDMA 网卡的通信从 Pod 中的应用进程提交 RDMA 请求开始,经过 Pod 的 RDMA 网卡、主机的 RDMA 网卡和交换机,到达对端节点上的 Pod。

![流量观测范围](../../images/traffic-observability-scope.png)

| 观测位置 | 观测内容 | 支持情况 | 文档 |
| --- | --- | --- | --- |
| ① NVLink | NVLink 域内 GPU 之间的 NVLink 流量 | 规划中 | 无 |
| ② RDMA 提交路径 | Pod 之间的 RDMA 流: 谁和谁通信、流量多大、每次发送多大 | 已支持 | [基于 eBPF 的 RDMA tracing](./rdma-flow.zh.md)、[RDMA 发送事件存储](./rdma-flow-events.zh.md) |
| ③ 主机与 Pod 的 RDMA 网卡 | 网卡的吞吐、拥塞、错误和重传 | 已支持 | [RDMA 指标](./rdma-metrics.zh.md) |
| ④ 交换机 | 交换机端口上的 RDMA 流量 | 规划中 | 无 |

Scale-out 网络和 Storage 网络的流量经过 ② 到 ④。
Scale-up 网络的流量经过 ①。

### NVLink 流量观测

| 项目 | 说明 |
| --- | --- |
| 观测对象 | NVLink 域内 GPU 之间经过 NVLink 和 NVLink Switch 的流量 |
| 支持情况 | 规划中 |

NVLink 域的范围取决于服务器:

- 8 卡服务器 (例如 DGX B200) 中,NVLink 域就是一个节点,NVLink 流量只在节点内。
- 多节点 NVLink 系统 (例如 GB200 NVL72、GB300 NVL72) 中,一个机柜的 18 个计算节点通过 NVLink Switch 组成一个 72 GPU 的 NVLink 域,NVLink 流量会跨节点。

NVLink 流量不经过 RDMA 网卡,网卡计数器和 eBPF 探针都看不到它。
这一项规划采集 GPU 之间的 NVLink 流量,并关联到使用这些 GPU 的 Pod。

> 注意: 拓扑发现把 NVLink 域记录为 Scale-up 网络的 tier 1。参见[拓扑发现概念](../../concepts/topology-discovery.zh.md)。

### 基于 eBPF 的 RDMA tracing

| 项目 | 说明 |
| --- | --- |
| 观测对象 | CPU 通过 `libibverbs` 提交的每次 RDMA 发送 |
| 数据来源 | `libibverbs` 提交路径上的 eBPF uprobe |
| 关联到 Pod 的方式 | 源 Pod: 发送进程所在的 cgroup。目的 Pod: QP 的目的地址,RoCE 网络使用 GID,InfiniBand 网络使用 LID。 |
| 输出 | `rdma_flow_*` 指标和 `RDMAEndpoint` 资源。可选: 把每次发送存入 ClickHouse 或 Elasticsearch。 |
| 支持情况 | 已支持 |
| 文档 | [基于 eBPF 的 RDMA tracing](./rdma-flow.zh.md)、[RDMA 发送事件存储](./rdma-flow-events.zh.md) |

RDMA 通信绕过内核,网卡上只有总量计数。
这一项在应用提交 RDMA 请求的位置挂载 eBPF 探针,无需修改应用,就能还原 Pod 之间的 RDMA 流: 谁和谁通信、何时通信、每次发送多大。
它让 RDMA 通信像 TCP 连接一样可观测。

限制:

- 只覆盖 CPU 发起的 RDMA。GPU 发起的 RDMA (例如 GDAKI、IBGDA) 和内核态的 RDMA 存储协议 (例如 NVMe over RDMA) 不经过 `libibverbs`,探针看不到它们。网卡计数器仍记录这部分流量。
- InfiniBand 网络需要设置 `agent.ebpfFlow.endpoint.allPods=true`,才能解析目的 Pod。
- Storage 网络上用户态存储客户端的 RDMA 流可以观测,但该场景尚未验证。

### 主机与 Pod 的 RDMA 网卡观测

| 项目 | 说明 |
| --- | --- |
| 观测对象 | Pod 的 RDMA 网卡 (SR-IOV VF,或 host 模式下共用的主机网卡) 和主机的 RDMA 网卡 (PF) |
| 数据来源 | sysfs 中的 RDMA 计数器,ethtool 优先级计数器 |
| 关联到 Pod 的方式 | Pod 网卡: `FabricNode.status.rdmaPods` 中的 RDMA Pod。主机网卡: 节点。VF 通过 `parent_ifname` 标签关联到 PF。 |
| 输出 | `unifabric_*` 指标,RDMA 集群、节点、Pod 和 AI Workload 仪表盘 |
| 支持情况 | 已支持 |
| 文档 | [RDMA 指标](./rdma-metrics.zh.md) |

这一项读取网卡的吞吐、拥塞 (ECN、CNP)、错误与重传,以及各优先级的 pause 和丢弃计数器。
`kind` 标签记录网卡属于哪张网络: `scaleOut`、`storage` 或 `scaleUp`。
它告诉你每张网卡和每个 Pod 收发了多少、有没有拥塞和错误,但不告诉你流量发给了谁。
通信对端由[基于 eBPF 的 RDMA tracing](#基于-ebpf-的-rdma-tracing) 提供。

### 交换机 RDMA 流量观测

| 项目 | 说明 |
| --- | --- |
| 观测对象 | Scale-out 网络和 Storage 网络中,交换机端口上的 RDMA 流量 |
| 支持情况 | 规划中,参见 [#49](https://github.com/unifabric-io/unifabric/issues/49) |

这一项规划采集交换机上的流量采样,并把采样到的 RDMA 流量关联到 Pod。
这样可以回答"交换机上的这段流量属于哪个任务"。

> 注意: 目前可以用拓扑发现确定网卡连接的交换机。拓扑发现在 Node label 中记录节点所在的交换机域。使用 LLDP 发现时,`FabricNode` 还记录每张网卡连接的交换机和端口。参见[拓扑发现概念](../../concepts/topology-discovery.zh.md)。

## 关联到 Pod 身份

上面几种观测方式各自拿到一个位置的数据: eBPF tracing 拿到 Pod 之间的 RDMA 流,网卡观测拿到每张网卡的计数器。
Unifabric 在采集时就给每条数据打上 Pod 身份: `pod_namespace`、`pod_name` 和顶层 workload (`topowner_*` 或 `*_top_owner_*`)。
不同位置的数据因此可以用同一个 Pod 或同一个 workload 查询,不需要人工做关联。

### 需要配置什么

大部分场景不需要配置:

- Pod 身份自动识别。Agent 从 `FabricNode.status.rdmaPods` 识别使用 RDMA 的 Pod (SR-IOV 或 host 模式),eBPF tracing 按发送进程所在的 cgroup 识别 Pod。
- 顶层 workload 自动解析。Chart 默认授予 Agent 读取 Kubernetes 内置 workload,以及 Kubeflow、JobSet、LeaderWorkerSet、KubeRay、Spark Operator 等常见 workload 的权限。

只有一种情况需要手动配置: 你的 workload 控制器不在默认列表中,例如 Argo Workflows。
这时在 `agent.workloadOwnerRules` 中添加它的 API 组和复数资源名:

```yaml
agent:
  workloadOwnerRules:
    argoproj.io: [workflows]
```

详细的识别规则和排查方法,参见 [RDMA 指标](./rdma-metrics.zh.md#pod-与-workload-归因)。

### 效果

打开 Unifabric RDMA Pod 仪表盘,选择 namespace 和 Pod,可以在一个页面上看到这个 Pod 在各个位置的流量:

| 分组 | 来自哪个观测位置 | 内容 |
| --- | --- | --- |
| 概览、RDMA 基础统计、RDMA 错误统计 | ③ 主机与 Pod 的 RDMA 网卡 | 这个 Pod 网卡的吞吐、带宽利用率、拥塞和错误计数器 |
| RDMA 发送明细 (CPU 侧发起) | ② RDMA 提交路径 | 这个 Pod 和哪些 Pod 通信、每条流的字节数、消息大小分布、最近的发送记录 |

![Unifabric RDMA Pod 仪表盘](../../images/pods.png)

打开 Unifabric RDMA AI Workload 仪表盘,选择 workload 的 kind、namespace 和 name,可以看到这个任务下所有 Pod 的汇总:
每个 Pod 的吞吐和错误,以及任务对外的 RDMA 流和对端 Pod。

![Unifabric RDMA AI Workload 仪表盘](../../images/workloads.png)

在 Prometheus 中也可以直接按 Pod 查询。例如,查看一个 Pod 网卡的发送吞吐和它发往每个对端的字节数:

```promql
sum(rate(unifabric_port_xmit_data{pod_namespace="tenant-a", pod_name="vllm-decode-0"}[5m]))
sum by (dst_pod_namespace, dst_pod_name) (rate(rdma_flow_bytes_total{src_pod_namespace="tenant-a", src_pod_name="vllm-decode-0"}[5m]))
```

例如,一个 `PyTorchJob` 变慢时:

1. 在 AI Workload 仪表盘中选择这个任务,找到流量最大的 Pod 和它的对端。
2. 切换到 Pod 仪表盘,查看这个 Pod 网卡的拥塞和错误计数器。
3. 用 Node 的拓扑 label 找到同一台 leaf 交换机下的其他节点,在 RDMA 集群仪表盘中检查这些节点上其他任务的流量。

## 参考

- [RDMA 指标](./rdma-metrics.zh.md)、[基于 eBPF 的 RDMA tracing](./rdma-flow.zh.md)、[RDMA 发送事件存储](./rdma-flow-events.zh.md): 各观测方式的原理、配置、验证和故障排查。
- [架构](../../concepts/architecture.zh.md): Agent、Controller 和各组件之间的关系。
- [拓扑发现概念](../../concepts/topology-discovery.zh.md): 节点和网卡连接的交换机。
- [RDMA 指标模型与 Pod 归因设计](../../design/rdma-metrics.zh.md)、[eBPF RDMA 流归因设计](../../design/rdma-flow.zh.md)。
- [RDMAEndpoint 参考](../../reference/rdmaendpoint.zh.md)、[FabricNode 参考](../../reference/fabricnode.zh.md)。
- [Helm values 参考](../../../chart/README.md): 全部 `nodeMetrics.*`、`agent.ebpfFlow.*`、`ebpfFlowEventCollector.*` 和 `grafanaDashboard.*` 参数。
- [路线图](../../roadmap.zh.md): 规划中的功能。
