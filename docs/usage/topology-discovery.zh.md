# 拓扑发现

English version: [topology-discovery.md](./topology-discovery.md)

## 简介

拓扑发现把物理网络的邻接关系整理为分层性能域,写入 Node label,并汇总到只读的
`Topology` 资源,供 Kueue、Volcano、KAI Scheduler 等调度器和运维人员消费。本文
介绍各物理网络下发现结果的含义、SONiC RoCE 网络的交换机接入方式,以及发现链路
的验证与排查。

发现结果包含:

- Node label:`scale-out.unifabric.io/tier-N` 表示 scale-out 网络的分层性能域,
  tier 1 离 Node 最近,其上依次为 `tier-2`、`tier-3` 等。label 值标识 Node 所在
  的性能域:Unifabric 内置发现使用 `tierN-groupM` 形式,例如
  `scale-out.unifabric.io/tier-1=tier1-group1`;NVIDIA Topograph 使用交换机
  GUID 派生的值,例如 `tier-1=S-fc6a1c03006636c0`。同一 label 值的 Node 位于
  同一性能域。
- `FabricNode` 和 `Switch` 资源暴露发现的输入状态,`Topology` 资源暴露聚合后的
  只读拓扑。

## 适用性

| 物理网络 | 发现方式 | Helm 配置 |
| --- | --- | --- |
| SONiC RoCE | Unifabric 通过节点与交换机 LLDP 发现 | `topoDiscovery.scaleOut.mode=unifabric-roce`(默认) |
| InfiniBand | NVIDIA Topograph 通过 `ibnetdiscover` 发现 | `topoDiscovery.scaleOut.mode=nv-topograph` |
| Spectrum-X | NVIDIA Topograph 通过 NetQ API 发现 | `topoDiscovery.scaleOut.mode=nv-topograph` + NetQ provider |

Storage 网络拓扑当前仅支持 SONiC RoCE 发现方式
(`topoDiscovery.storage.mode=unifabric-roce`)。Scale-up label 来自节点内的
GPU 互联拓扑,仅 InfiniBand 场景的 `nv-topograph` 模式写入。

## 前提条件

继续之前,确认以下各项均成立:

- [ ] 已按[部署 Unifabric](../getting-started.zh.md) 完成安装,并配置了对应
      物理网络的拓扑发现参数。
- [ ] 节点上的 RDMA 设备在 `/sys/class/infiniband` 下可见。

## 配置与使用

### SONiC RoCE:LLDP 拓扑发现

Unifabric 从节点 RDMA 网卡、`FabricNode` 的 LLDP 邻居,以及可选的交换机侧 LLDP
快照中发现 leaf、spine、core 拓扑。确认每台交换机都启用了 LLDP。

本节包含两个 case,开始前先选定一个:

| Case | 适用情况 | 需要准备 |
| --- | --- | --- |
| Case 1:仅节点 LLDP | 安全策略禁止在交换机上运行软件,或交换机由其他团队维护 | 节点通过节点侧 LLDP 发现其 leaf 交换机。管理员用 `Switch` 资源和邻居 annotation 声明 leaf 以上的层级。 |
| Case 2:交换机运行 `switch-agent`(推荐) | 运维团队可以在 SONiC 交换机上运行 Unifabric Switch Agent,并希望自动发现交换机间拓扑 | 交换机已启用 LLDP。运行 `unifabric-controller` 的 Kubernetes Node 可以访问每台交换机的 TCP `8090` 端口。 |

下面两个 case 使用同一个示例网络。`node1` 和 `node2` 通过 `leaf1` 和 `leaf2`
接入,组成 `tier1-group1`;`node3` 和 `node4` 通过 `leaf3` 和 `leaf4` 接入,
组成 `tier1-group2`;它们共同的上联 `spine1` 组成 `tier2-group1`。

![RoCE 交换机拓扑发现示例](../images/sonic-roce-topology-example.jpg)

#### Case 1:仅节点 LLDP

该 case 不在交换机上运行任何软件。当不存在同 role 的 `Switch` 资源时,Unifabric
进入 node-only 模式:把 `FabricNode` LLDP 上报的交换机 hostname 视为合成 leaf,
只生成 tier 1。

1. 等待 Agent 上报 LLDP 邻居。tier 1 不需要任何交换机侧操作。在示例网络中,发现
   会生成两个 tier 1 域,每个 Node 获得自己的 `scale-out.unifabric.io/tier-1`
   label。

2. 要描述 spine、core 等上层,为每台上层交换机创建一个 `Switch` 资源,并给每个
   资源添加 `unifabric.io/switch-neighbors` annotation。不要为 leaf 创建资源,也
   不要设置 `spec.mgmtIP`。对于示例网络,在可以用 `kubectl` 访问集群的主机上应用
   以下文件:

   ```yaml
   apiVersion: unifabric.io/v1beta1
   kind: Switch
   metadata:
     name: spine1
     annotations:
       unifabric.io/switch-neighbors: '["leaf1", "leaf2", "leaf3", "leaf4"]'
   spec:
     role: ScaleOut
   ```

   ```bash
   kubectl apply -f spine1.yaml
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

> **警告:**不要在 `unifabric.io/switch-neighbors` 中写入 Kubernetes Node 名称。
> 该 annotation 只列出交换机邻居。Node 到交换机的链路来自 `FabricNode` LLDP。
> 数组中出现 Node 名称会破坏计算出的拓扑。

> **说明:**模式选择只判断每个同 role `Switch` 上是否存在该 annotation key。
> 空值或 `[]` 也会选中此模式,但不提供任何邻居链路。

应用后,四个 Node 都会获得 tier 1 和 tier 2 两个 label。[验证](#验证)章节给出了
预期的 `Topology` 内容。

#### Case 2:使用 switch-agent 自动发现

该 case 在每台参与的交换机上运行 `switch-agent`,通过双向认证 TLS(mTLS)向
Controller 上报 LLDP 快照。只要任意一个同 role 的 `Switch` 资源缺少
`unifabric.io/switch-neighbors` key,Unifabric 就进入全自动模式,并忽略所有该
annotation。为保证配置无歧义,在所有 `Switch` 上都省略该 annotation。

> **说明:**要一次部署到多台交换机,使用
> [switch-agent 自动化部署脚本](./deploy-switch-agent-script.zh.md),
> 不必执行下面的手动步骤。

1. 导出 pinned mTLS 证书。在可以用 `kubectl` 访问集群的主机上执行:

   ```bash
   mkdir -p ./tmp-switch-mtls

   kubectl -n unifabric-system get secret switch-controller-mtls-agent -o jsonpath='{.data.tls\.crt}' | base64 -d > ./tmp-switch-mtls/tls.crt
   kubectl -n unifabric-system get secret switch-controller-mtls-agent -o jsonpath='{.data.tls\.key}' | base64 -d > ./tmp-switch-mtls/tls.key
   kubectl -n unifabric-system get secret switch-controller-mtls-agent -o jsonpath='{.data.peer\.crt}' | base64 -d > ./tmp-switch-mtls/peer.crt
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

2. 登录目标交换机,创建证书目录:

   ```bash
   sudo mkdir -p /opt/unifabric-switch-agent/mtls
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

   然后把 `tls.crt`、`tls.key` 和 `peer.crt` 复制到交换机的
   `/opt/unifabric-switch-agent/mtls/` 目录。

3. 在交换机上确认 LLDP 能看到预期邻居:

   ```bash
   lldpcli show neighbors -f json0
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

4. 在交换机上启动 `switch-agent`。下面的命令使用 Docker bridge 网络,通过
   `-p 8090:8090` 发布 gRPC 端口。它通过挂载的 `lldpd` socket 读取 LLDP,不需要
   host network,也不需要特权权限。如果交换机无法运行容器,改用
   [switch-agent systemd 安装](./switch-agent-systemd.zh.md)。

   ```bash
   export SWITCH_AGENT_IMAGE="ghcr.io/unifabric-io/unifabric-switch-agent:<release-tag>"
   export SWITCH_NAME="$(hostname)"

   docker pull "${SWITCH_AGENT_IMAGE}"

   docker rm -f unifabric-switch-agent 2>/dev/null || true

   docker run -d \
     --name unifabric-switch-agent \
     --restart unless-stopped \
     -p 8090:8090 \
     -e UNIFABRIC_SWITCH_AGENT_SWITCH_NAME="${SWITCH_NAME}" \
     -v /run/lldpd.socket:/run/lldpd.socket \
     -v /opt/unifabric-switch-agent/mtls:/etc/unifabric/switch-mtls:ro \
     "${SWITCH_AGENT_IMAGE}" \
     /usr/bin/unifabric/switch-agent
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

   > **说明:**如果交换机运行 SONiC 202006 到 202311,追加
   > `-e UNIFABRIC_SWITCH_AGENT_LLDP_CLI_VERSION=1.0.4`。其他版本保持默认值
   > `1.0.16`。

   > **说明:**如果交换机无法挂载 `/run/lldpd.socket`,改用
   > [hostProc 采集模式](./switch-agent-host-proc.zh.md)。该模式需要特权权限并
   > 挂载宿主机 `/proc`。

5. 确认容器正在运行且日志没有错误:

   ```bash
   docker ps | grep unifabric-switch-agent
   docker logs --tail 100 unifabric-switch-agent
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

6. 为每台交换机创建一个 `Switch` 资源。Controller 连接 `spec.mgmtIP` 读取 LLDP
   快照。对于示例网络,创建五个资源:`leaf1` 到 `leaf4` 和 `spine1`。每个资源
   形如:

   ```yaml
   apiVersion: unifabric.io/v1beta1
   kind: Switch
   metadata:
     name: leaf1
   spec:
     mgmtIP: 192.0.2.11
     role: ScaleOut
     grpcPort: 8090
   ```

   `role` 按交换机服务的网络设置。`ScaleOut` 表示交换机连接用于跨节点流量的主机
   RDMA 网卡。`Storage` 表示交换机连接 `fabricNode.storageInterfaceSelector`
   选中的网卡。`ScaleUp` 表示交换机连接 GPU 侧互联网卡。默认值为 `ScaleOut`。
   `grpcPort` 默认为 `8090`。

   在可以用 `kubectl` 访问集群的主机上应用文件:

   ```bash
   kubectl apply -f leaf1.yaml
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

### InfiniBand:NVIDIA Topograph

该场景不需要交换机侧操作。NVIDIA Topograph 通过 node-data-broker 中的
`ibnetdiscover` 发现 IB fabric,并写入 Node label。

下图是示例拓扑:tier 3 core、tier 2 spine、tier 1 leaf 三层网络,四个节点分别
位于两个 tier 1 性能域:

![InfiniBand 三层拓扑示例](../images/infiniband-topology-example.png)

Scale-out 拓扑发现完成后,四个节点带有以下 label,label 值来自交换机 GUID:

| 节点 | 预期 label |
| --- | --- |
| `node1` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006636c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |
| `node2` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006636c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |
| `node3` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006637c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |
| `node4` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006637c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |

Scale-up label 来自节点内的 GPU 互联拓扑,不在上图中展示。

### Spectrum-X:NVIDIA Topograph + NetQ

该场景不需要交换机侧操作。NVIDIA Topograph 通过 NetQ API 读取 fabric 拓扑并写入
Node label。NetQ 凭据 Secret 和 provider 参数在
[部署 Unifabric](../getting-started.zh.md#spectrum-x-网络) 中配置,本节没有额外
步骤。

## 验证

查看 Node label:

```bash
kubectl get nodes -L scale-out.unifabric.io/tier-1,scale-out.unifabric.io/tier-2,scale-out.unifabric.io/tier-3,kubernetes.io/hostname
```

```text
<!-- TODO: 待真实环境采集输出 -->
```

确认每个 Node 有连续的 `scale-out.unifabric.io/tier-N` label,tier 1 离 Node
最近。在 SONiC RoCE Case 1 且没有上层 `Switch` 资源时,只有 tier 1 是预期结果。
配置 Kueue、Volcano 或 KAI Scheduler 时,只使用已经真实写到 Node 上的 label。

对于 SONiC RoCE,查看一个 `FabricNode`:

```bash
kubectl get fabricnode <node-name> -o yaml
```

```text
<!-- TODO: 待真实环境采集输出 -->
```

确认输出同时满足以下各项:

- `status.scaleOutNics` 包含预期的 scale-out RDMA 网卡。
- `status.scaleOutNics[*].lldpNeighbor.hostname` 存在。
- `status.conditions` 中的 `Ready` 和 `LLDPNeighborsReady` 为 `True`。

对于 SONiC RoCE Case 2,查看 `Switch` 资源:

```bash
kubectl get switch
```

```text
NAME     MGMTIP            ROLE       HEALTHY   NEIGHBORS
leaf1    192.168.122.72    ScaleOut   true      2
leaf2    192.168.122.80    ScaleOut   true      2
spine1   192.168.122.163   ScaleOut   true      2
```

确认每台交换机的 `HEALTHY` 为 `true`,且 `NEIGHBORS` 大于 `0`。

查看聚合拓扑。只有发现产生有效结果后,Controller 才会创建 `Topology` 资源。
`status.domains` 描述性能域之间的父子关系,`status.nodes[].switchDomainPath`
按最高 tier 到 tier 1 的顺序记录节点所在的完整路径:

```bash
kubectl get topology scaleout -o yaml
```

```text
<!-- TODO: 待真实环境采集输出 -->
```

对于 SONiC RoCE Case 2 的示例网络,status 包含两个 tier,`switchMember` 列出
真实的 `Switch` 名称。对于 Case 1 的 node-only 模式,只出现 tier 1 域,且
`switchMember` 为空,因为合成 leaf 没有对应的 `Switch` 资源。

## 故障排查

先按[部署 Unifabric 的故障排查](../getting-started.zh.md#故障排查)排除
`FabricNode` 和 RDMA 指标的通用问题。

### `LLDPNeighborsReady` 为 `False`(SONiC RoCE)

1. 确认交换机已启用 LLDP。
2. 确认节点侧 `lldpd` 正常工作,且 Agent Pod 能读取 LLDP 数据。
3. 确认 `fabricNode.initialScanDelay` 足够长。首次扫描在 LLDP 学习完成前运行会
   上报邻居缺失。

### Node 没有拓扑 label(SONiC RoCE)

1. 确认 `topoDiscovery.scaleOut.mode=unifabric-roce`。
2. 确认 `FabricNode.status.nodeRole` 不是 `Storage`。
3. 确认至少一个 `scaleOutNics` 条目同时有 `state=up` 和
   `lldpNeighbor.hostname`。
4. 在 Case 2 中,确认每个 LLDP hostname 都匹配一个 `Switch` 名称,且交换机
   status 中有 LLDP 邻居。
5. 在 Case 1 且存在上层时,确认每个同 role `Switch` 都带有该 annotation key,
   且其邻居名称能解析为节点发现的 leaf 或其他带 annotation 的 `Switch` 资源。
6. 查看 Controller 日志:

   ```bash
   kubectl -n unifabric-system logs deployment/unifabric-controller
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

7. 在 Case 2 中,在交换机上查看 switch-agent 日志:

   ```bash
   docker logs --tail 100 unifabric-switch-agent
   ```

   ```text
   <!-- TODO: 待真实环境采集输出 -->
   ```

### Node 没有拓扑 label(InfiniBand)

1. 确认 NVIDIA topograph、node-observer 和 node-data-broker 正常运行,并且有
   权限更新 Node。
2. 确认 node-data-broker Pod 已运行在目标 GPU 节点上。
3. 如果 scale-out label 没有写入,确认 node-data-broker Pod 中的
   `ibnetdiscover` 可用。
4. 如果自定义了 `topoDiscovery.*.nodeLabel.keyTemplate`,同步更新调度器配置中的
   label key。

### Node 没有拓扑 label(Spectrum-X)

1. 确认 NVIDIA topograph 和 node-observer 正常运行,并且有权限更新 Node。
2. 确认 `nvidiaTopograph.provider.name=netq`,且
   `nvidiaTopograph.provider.params.apiUrl=<netq-url>` 配置正确。
3. 确认 `nvidiaTopograph.credentialsSecretName` 指向安装 namespace 中的
   Secret,且该 Secret 包含 `credentials.yaml` key。
4. 确认 NetQ 账号可以访问目标 premises,且 NetQ 中有对应的 fabric 拓扑数据。
5. 如果自定义了 `topoDiscovery.*.nodeLabel.keyTemplate`,同步更新调度器配置中
   的 label key。

### Topograph 无法访问 NetQ(Spectrum-X)

1. 确认 NetQ API URL、Secret 中的用户名和密码、证书以及网络连通性都正确。
2. 确认 Topograph Pod 已把凭据挂载到
   `/etc/topograph/credentials/credentials.yaml`。
3. 查看 topograph 和 node-observer 日志。
4. 确认 topograph Service 可达。

## 参考

- 在调度器中使用这些 label:参见 [Kueue TAS 工作负载示例](./workload-tas.zh.md)。
- 渲染拓扑:参见[拓扑可视化指南](./topology-visualization.zh.md)。
- 查阅字段:参见 [FabricNode 参考](../reference/fabricnode.zh.md)和
  [Topology 参考](../reference/topology.zh.md)。
- 调整安装配置:参见 [Helm values 参考](../../chart/README.md)。
