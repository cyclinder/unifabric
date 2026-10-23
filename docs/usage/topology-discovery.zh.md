# 拓扑发现

English version: [topology-discovery.md](./topology-discovery.md)

## 简介

AI 集群依赖三张高速网络。scale-out 网络承载 GPU 跨节点通信。scale-up 网络连接
节点内的 GPU。存储网络承载到高速存储的流量。拓扑发现探测这三张网络中主机与交换机
之间的连接关系。物理拓扑原本散落在多家厂商的网络设备里,拓扑发现把它变成标准的
Kubernetes 资源:

- 分层 Node label。Unifabric 把每个节点在每张网络中的位置(例如 leaf 层、spine 层)
  写入 Node label。调度器直接读取这些 label。
- `Topology` 资源。这个集群级资源呈现每张网络的性能域和每个节点的路径。Grafana 用它
  渲染拓扑图,运维人员用它查询。

通过本篇文章你将学习到:

- 在 AI 集群的通用 SONiC RoCE、NVIDIA Spectrum-X、InfiniBand 三种网络场景下,拓扑
  发现和拓扑可视化如何工作。
- Kueue、Volcano、KAI Scheduler 等调度器如何消费拓扑发现的结果。

## 核心价值

对云原生体系来说,这三张网络几乎是黑盒。物理连接关系散落在多家厂商的交换机、网卡和
网络管理平台里。各厂商的观测工具互不相通,也都不与 Kubernetes 的资源模型对接。这带来
两个直接的问题:

- 调度器看不到网络拓扑。调度器不知道哪些节点接在同一台 leaf 交换机下。一个训练任务
  的 Pod 被随机摊开到整个集群。它们的流量跨越 spine 层甚至 core 层,那里带宽受限,
  并且受其他任务的流量干扰。
- 运维看不到全局连接。要确认一次故障影响了哪些节点和链路,运维必须逐台登录交换机。
  在数百台节点、几十台交换机的集群里,这个过程缓慢且容易出错。

拓扑发现解决这两个问题:

- 提升训练与推理效率。Kueue、Volcano、KAI Scheduler 等拓扑感知调度器读取 Unifabric
  写入的 Node label。它们把一个任务的 Pod 放在同一个交换机域内。这缩短了通信路径,
  减少了跨层流量和相互干扰。
- 缩短排障时间。`Topology` 资源和 Grafana 拓扑图呈现全网的连接关系。运维不登录
  交换机,就能看到某台交换机或某条链路影响到哪些节点和任务。
- 统一多厂商拓扑。无论网络是 SONiC、Spectrum-X 还是 InfiniBand,结果都使用同一套
  label 模型和同一个 `Topology` 资源。调度器配置和运维流程不随网络厂商变化。

## 前提条件

- 已按[基础安装](../getting-started.zh.md)完成部署,并按其中「其他网络环境」小节为
  集群的网络设置了拓扑发现参数。
- `kubectl get fabricnodes` 中每个节点的 `READY` 为 `True`,网卡归类正确。基础安装
  的验证小节已覆盖这一步。
- 仅拓扑可视化需要:集群已安装 Grafana Operator 及其 CRD。

## 拓扑发现支持场景

Unifabric 支持三种拓扑发现场景:

- 通用 SONiC RoCE 交换机(LLDP 发现)
- NVIDIA Spectrum-X
- InfiniBand(从 NVIDIA Topograph 导入)

选择与集群物理网络匹配的场景。各场景只有发现机制不同。结果是一样的,都在
[查看 Topology 与 Node label](#查看-topology-与-node-label)和
[查看拓扑可视化](#查看拓扑可视化)中查看。

下面依次介绍每个场景。

### 场景一:通用 SONiC RoCE

SONiC 是运行在白盒交换机上的开源网络操作系统。RoCE 是在以太网上承载 RDMA 的协议。
在通用 SONiC RoCE 场景中,scale-out 网络或存储网络是由 SONiC 交换机组成的以太网
fabric,节点通过 RoCE 网卡接入。这类网络没有提供拓扑的厂商管理平台,所以 Unifabric
用 LLDP 发现拓扑。LLDP 是一种协议,设备用它向链路对端的设备通告自己的身份和端口。

| 项目 | 说明 |
| --- | --- |
| 覆盖网络 | scale-out、storage |
| 拓扑来源 | 节点侧 LLDP 邻居,加上交换机侧 LLDP 邻居或管理员声明 |
| Helm 配置 | `topoDiscovery.scaleOut.mode=unifabric-roce`(默认)。storage 网络还需要网卡 selector。参见[基础安装](../getting-started.zh.md#安装)。 |

配置和使用参见 [SONiC RoCE 拓扑发现](./topology/sonic-switch-onboarding.zh.md)。

### 场景二:NVIDIA Spectrum-X

Spectrum-X 是 NVIDIA 面向 AI 集群的以太网 RoCE 网络平台。它由 Spectrum 交换机和
BlueField 或 ConnectX 网卡组成。NVIDIA NetQ 网络管理平台管理这个 fabric。NetQ 已经
掌握完整的 fabric 拓扑,所以 Unifabric 在这里不用 LLDP。它通过 NVIDIA Topograph 从
NetQ 导入拓扑。Topograph 是 NVIDIA 开源的服务,从厂商数据源读取网络拓扑并写入
Node label。

| 项目 | 说明 |
| --- | --- |
| 覆盖网络 | scale-out |
| 拓扑来源 | NVIDIA Topograph 通过 NetQ API 读取 fabric 拓扑 |
| Helm 配置 | `topoDiscovery.scaleOut.mode=nv-topograph` 并配置 NetQ 凭据。参见[基础安装](../getting-started.zh.md#spectrum-x-网络)。 |

Controller 直接导入 Topograph 的结果,不从 LLDP 计算。NetQ 中的 fabric 拓扑数据
决定能发现哪些层级。通常结果包含 leaf 层、spine 层及以上全部层级。

限制:

- 本场景不创建 `Switch` 资源。`Topology` 资源和 Node label 完整,但拓扑可视化中每个
  性能域的交换机成员卡片为空。
- 本场景不发现 scale-up 网络和 storage 网络。

#### 配置

Node label 和 `Topology` 资源在基础安装完成后出现。不需要更多步骤。如果需要拓扑
可视化显示交换机,按[为 Topograph 场景补充 Switch 资源](./topology/topograph-switch-resources.zh.md)
创建仅含标签的 `Switch` 资源。

### 场景三:InfiniBand

InfiniBand 是独立于以太网的高速互联技术。它由 IB 交换机和运行在 IB 模式下的网卡
(例如 Mellanox ConnectX)组成。子网管理器(Subnet Manager)维护整个 fabric 的
拓扑。Unifabric 通过 NVIDIA Topograph 的 `infiniband-k8s` provider 发现拓扑。
Topograph 在节点上运行 `ibnetdiscover` 读取 IB fabric 的完整拓扑。它还读取每个节点
内的 GPU 互联信息,得到 scale-up 域。

| 项目 | 说明 |
| --- | --- |
| 覆盖网络 | scale-out、scale-up |
| 拓扑来源 | NVIDIA Topograph。scale-out:`ibnetdiscover` 读取 IB fabric。scale-up:节点内的 GPU 互联拓扑。 |
| Helm 配置 | `topoDiscovery.scaleOut.mode=nv-topograph` 和 `topoDiscovery.scaleUp.mode=nv-topograph`。参见[基础安装](../getting-started.zh.md#infiniband-网络)。 |

scale-out 发现 leaf、spine、core 层级。label 值是由 IB 交换机 GUID 派生的名称,
例如 `S-fc6a1c03006636c0`。scale-up 发现节点内的 GPU 互联域,写入
`scale-up.unifabric.io/tier-1`。

限制:

- 与场景二相同,本场景不创建 `Switch` 资源,拓扑可视化中的交换机成员卡片为空。
- 本场景不发现 storage 网络。

#### 配置

与场景二相同:不需要更多步骤。如果需要拓扑可视化显示交换机,参见
[为 Topograph 场景补充 Switch 资源](./topology/topograph-switch-resources.zh.md)。

## 验证

### 查看 Topology 与 Node label

本节以一个 SONiC RoCE 集群为例。`node1` 和 `node2` 同时接入 `leaf1` 和 `leaf2`。
`node3` 和 `node4` 同时接入 `leaf3` 和 `leaf4`。四台 leaf 交换机都接入 `spine1`:

![SONiC RoCE 示例拓扑](../images/sonic-roce-topology-example.jpg)

发现完成后,查看 Node label:

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

`node1` 和 `node2` 的 `tier-1` 值相同,所以它们在同一个 leaf 域。四个节点的 `tier-2`
值相同,所以它们共享一个 spine 域。如果节点只有 `tier-1` 没有 `tier-2`,说明 leaf
以上的层级没有被发现。参见[故障排查](#故障排查)。

在 SONiC RoCE、NVIDIA Spectrum-X 和 InfiniBand 场景中,Node label 遵循同一套模型:

| Node label | 含义 |
| --- | --- |
| `scale-out.unifabric.io/tier-1` | 节点直连的 scale-out leaf 交换机域 |
| `scale-out.unifabric.io/tier-2` | leaf 之上的 spine 域 |
| `scale-out.unifabric.io/tier-3` | spine 之上的 core 域 |
| `scale-up.unifabric.io/tier-1` | 节点内的 GPU 互联域,例如 NVLink 域 |
| `storage.unifabric.io/tier-N` | 存储网络的对应层级。规则与 scale-out 相同。 |

tier 数字越大,表示越靠近网络上层。两个节点的某个 label 值相同,表示它们在同一个
性能域。

Helm value `topoDiscovery.<topology>.nodeLabel.keyTemplate` 定义 label key。默认值
为 `scale-out.unifabric.io/tier-{{ .Tier }}`、`scale-up.unifabric.io/tier-{{ .Tier }}`
和 `storage.unifabric.io/tier-{{ .Tier }}`。模板可以修改,但必须且只能包含一个
`{{ .Tier }}`。

> **注意:** 如果修改了 key 模板,同时修改调度器配置中的 label key。否则调度器读不到
> 任何拓扑信息。

`Topology` 资源汇总性能域的层级关系和每个节点的域路径:

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

在 `status.domains` 中,`members` 列出该域包含的交换机,`parent` 指向上一层的域。在
`status.nodes` 中,每一组节点共享一条自上而下的域路径。字段的完整说明参见
[Topology 参考](../reference/topology.zh.md)。

#### 在调度器中使用 Node label

拓扑感知调度器把 Node label 组织成自上而下的层级。然后它按层级放置一个任务的 Pod。
下表把 Unifabric 的 label 对应到各调度器的配置入口。调度器本身的安装和配置,以其
官方文档为准:

| 调度器 | 配置入口 | 填写的 Unifabric label |
| --- | --- | --- |
| Kueue Topology Aware Scheduling | `Topology` 资源的 `spec.levels` | 自上而下:`scale-out.unifabric.io/tier-3`、`tier-2`、`tier-1`,最后是 `kubernetes.io/hostname` |
| Volcano network topology aware scheduling | `HyperNode` 资源 | 每个 tier 创建一个 HyperNode。成员选择器引用对应的 label。 |
| KAI Scheduler topology aware scheduling | `Topology` 资源的 `spec.levels` | 与 Kueue 相同 |

参考:[Kueue TAS](https://kueue.sigs.k8s.io/docs/concepts/topology_aware_scheduling/)、
[Volcano network topology](https://volcano.sh/en/docs/network_topology_aware_scheduling/)、
[KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler)。

### 查看拓扑可视化

Grafana 拓扑仪表盘把 `Topology` 资源渲染成交互式图形。图形与上面的 Node label
对应。每个 tier 1 域是一个 leaf 分组。每个 tier 2 域是 leaf 分组之上的 spine 分组。
每个节点挂在它所属的 leaf 分组下。

![拓扑可视化仪表盘](../images/topology-visualization.png)

仪表盘的安装、外部 Grafana 的接入以及图形的详细解读,参见
[拓扑可视化](./topology/topology-visualization.zh.md)。

## 故障排查

### Node 上没有任何拓扑 label

先确认前提条件成立:

1. 确认 `FabricNode` 的网卡已收到 LLDP 邻居:

   ```bash
   kubectl get fabricnode node1 -o jsonpath='{range .status.scaleOutNics[*]}{.name}{"\t"}{.lldpNeighbor.hostname}{"\t"}{.lldpNeighbor.port}{"\n"}{end}'
   ```

   ```text
   eth1	leaf1	Ethernet0
   eth2	leaf2	Ethernet0
   ```

   如果 hostname 列为空,确认交换机已启用 LLDP,并且节点网卡对应的交换机端口发送
   LLDP 报文。`FabricNode` 的 `LLDPNeighborsReady` condition 也会报告
   `LLDPNeighborMissing` 原因。

2. SONiC RoCE 场景,确认 `topoDiscovery.scaleOut.mode` 为 `unifabric-roce`。
   Topograph 场景,确认 `mode` 为 `nv-topograph`,并且 Topograph Deployment 在运行:

   ```bash
   kubectl -n unifabric-system get deployment unifabric-nvidia-topograph
   kubectl -n unifabric-system logs deployment/unifabric-nvidia-topograph --tail 100
   ```

   Spectrum-X 场景,检查 NetQ 凭据和 API 地址。InfiniBand 场景,确认日志显示
   `ibnetdiscover` 正常运行。

3. 查看 Controller 日志:

   ```bash
   kubectl -n unifabric-system logs deployment/unifabric-controller --tail 200
   ```

### Node label 只有 tier 1

节点侧 LLDP 只发现 leaf 层。这个现象说明 leaf 以上的信息没有到达 Controller:

- 如果交换机运行 switch-agent:确认每台交换机都有 `Switch` 资源,并且
  `kubectl get switch -o wide` 中 `HEALTHY` 为 `true`。如果为 `false`,确认运行
  Controller 的节点能访问交换机的 `8090` 端口,并且 mTLS 证书匹配。
- 如果交换机不运行 switch-agent:确认 spine 层及以上每台交换机都有 `Switch` 资源。
  确认 `unifabric.io/switch-neighbors` annotation 是带引号的 JSON 字符串数组。数组中
  的每个名称必须对应节点发现的某台 leaf,或另一个 `Switch` 资源。
- 如果物理网络只有 leaf 一层,只有 tier 1 是预期结果。

两种形态的详细步骤参见 [SONiC 交换机接入](./topology/sonic-switch-onboarding.zh.md)。

### 拓扑可视化中交换机成员为空

这是 Topograph 场景(Spectrum-X 和 InfiniBand)的预期行为。参见
[为 Topograph 场景补充 Switch 资源](./topology/topograph-switch-resources.zh.md)。

## 参考

- [拓扑发现概念](../concepts/topology-discovery.zh.md):逻辑拓扑、发现方式与发现结果。
- [SONiC 交换机接入](./topology/sonic-switch-onboarding.zh.md):switch-agent 部署与 `Switch` 资源配置。
- [为 Topograph 场景补充 Switch 资源](./topology/topograph-switch-resources.zh.md):Spectrum-X 和 InfiniBand 场景的可选步骤。
- [拓扑可视化](./topology/topology-visualization.zh.md):Grafana 仪表盘的安装与解读。
- [Topology 参考](../reference/topology.zh.md)、[FabricNode 参考](../reference/fabricnode.zh.md)、[Switch 参考](../reference/switch.zh.md)。
- [Scale-out 拓扑发现设计](../design/scaleout-topology.zh.md)、[Topology CRD 设计](../design/topology-crd.zh.md)。
- [Helm values 参考](../../chart/README.md):`topoDiscovery.*` 全部参数。
