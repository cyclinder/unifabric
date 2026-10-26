# 自定义拓扑

English version: [custom-topology.md](./custom-topology.md)

## 简介

自定义拓扑是面向不能运行 switch-agent 的交换机的发现方式。原因可以是安全策略、
交换机由其他团队管理,或交换机 OS 没有 Docker 也没有 Linux shell。交换机只需要发送
LLDP。

在本方式中,每个节点上的 Unifabric Agent 仍然读取每张 RDMA 网卡的 LLDP 邻居。
Controller 由此发现 leaf 交换机以及节点到它们的连接。对于 leaf 以上的层级,你为每台
spine 和 core 交换机创建一个 `Switch` 资源。在它的 `unifabric.io/switch-neighbors`
annotation 中,声明它下一层的交换机。Controller 把发现的 leaf 层与声明的上层合并。
它写出与其他方式相同的 Node label 和 `Topology` 资源。

| 项目 | 说明 |
| --- | --- |
| 适用网络 | Scale-out 网络、Storage 网络 |
| 拓扑来源 | 节点侧的 LLDP 邻居,加上 `Switch` 资源上的邻居声明 |
| 能发现的层级 | leaf 层自动发现。spine 层和 core 层来自声明。 |
| 交换机侧要求 | 面向节点的端口启用 LLDP。交换机 OS 不限。交换机上不运行任何软件。 |
| 集群侧要求 | 无。 |
| Helm 配置 | 与 [LLDP 发现](./lldp-discovery.zh.md)相同:`topoDiscovery.scaleOut.mode=unifabric-roce` 或 `topoDiscovery.storage.mode=unifabric-roce`。 |

限制:

- 声明的连接不会随物理连接变化而更新。新增交换机或换线之后,更新声明。
- leaf 层只能来自节点侧的 LLDP。你不能声明它。如果交换机面向节点的端口不发送 LLDP,
  节点找不到自己的 leaf。此时不能使用本方式。

各网络和各网络设备应使用哪种发现方式,参见
[网络拓扑发现](./README.zh.md#按网络选择发现方式)。

## 前提条件

- 已按[基础安装](../../getting-started.zh.md)部署 Unifabric,并且 `topoDiscovery.scaleOut.mode`
  或 `topoDiscovery.storage.mode` 为 `unifabric-roce`。
- 交换机面向节点的端口已启用 LLDP。节点必须看到它们的 leaf 交换机。用下面的命令确认。
  hostname 列不能为空:

  ```bash
  kubectl get fabricnode node1 -o jsonpath='{range .status.scaleOutNics[*]}{.name}{"\t"}{.lldpNeighbor.hostname}{"\t"}{.lldpNeighbor.port}{"\n"}{end}'
  ```

  ```text
  eth1	leaf1	Ethernet0
  eth2	leaf2	Ethernet0
  ```

- 操作员必须了解 leaf、spine、core 交换机之间的物理连接。
- 同 role 的 `Switch` 都不运行 switch-agent。annotation key 为整个 role 选择自定义拓扑。
  如果该 role 中一个 `Switch` 有该 key 而另一个没有,Controller 忽略该 role 的全部
  annotation。

## 配置与使用

示例使用[网络拓扑发现](./README.zh.md#查看-topology-与-node-label)中的拓扑。它有四个
节点、四台 leaf 交换机和一台 spine 交换机 `spine1`。`spine1` 连接 `leaf1` 到 `leaf4`。

1. 不要为 leaf 交换机创建 `Switch` 资源。节点 LLDP 已经发现它们。Controller 用每台
   leaf 的 LLDP hostname 作为它的名称。

2. 为 spine 层及以上的每台交换机创建一个 `Switch` 资源。在 annotation 中声明它下一层
   的交换机。Scale-out 网络把 `role` 设为 `ScaleOut`,Storage 网络设为 `Storage`:

   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: unifabric.io/v1beta1
   kind: Switch
   metadata:
     name: spine1
     annotations:
       unifabric.io/switch-neighbors: '["leaf1", "leaf2", "leaf3", "leaf4"]'
   spec:
     role: ScaleOut
   EOF
   ```

   annotation 的值是带引号的 JSON 字符串数组。每个名称必须是节点发现的某台 leaf 的
   LLDP hostname。它也可以是另一个同 role `Switch` 资源的名称。不要使用 Kubernetes
   Node 名称。连接没有方向,所以每条连接只声明一次。

3. 如果网络有 core 层,为每台 core 交换机创建一个 `Switch` 资源。它的邻居是与它相连
   的 spine 交换机:

   ```bash
   kubectl apply -f - <<'EOF'
   apiVersion: unifabric.io/v1beta1
   kind: Switch
   metadata:
     name: core1
     annotations:
       unifabric.io/switch-neighbors: '["spine1", "spine2"]'
   spec:
     role: ScaleOut
   EOF
   ```

> 警告:声明的连接不会随物理连接变化而更新。新增交换机或换线之后,更新 annotation。
> 否则 Node label 与真实拓扑不一致。

> 对同 role 的 `Switch` 资源,不要既运行 switch-agent 又设置该 annotation。

## 验证

确认 `Switch` 资源存在。因为没有 switch-agent 向 Controller 上报,`HEALTHY` 和
`NEIGHBORS` 为空:

```bash
kubectl get switches -o wide
```

```text
NAME     MGMTIP   ROLE       HEALTHY   NEIGHBORS
spine1            ScaleOut
```

最终结果是 Node label 和 `Topology` 资源。查看方法参见
[网络拓扑发现](./README.zh.md#查看-topology-与-node-label)。在 `Topology` 资源中,
leaf 域的成员是 LLDP 发现的 leaf hostname。spine 域的成员是 `spine1`。

## 故障排查

### Node label 只有 tier 1

- 确认 spine 层及以上的每台交换机都有 `Switch` 资源。
- 确认同 role 的每个 `Switch` 都有 `unifabric.io/switch-neighbors` annotation。一个
  没有该 key 的 `Switch` 会把整个 role 改回 switch-agent 发现。
- 确认 annotation 是带引号的 JSON 字符串数组。数组只能包含交换机名称,不能包含
  Kubernetes Node 名称。
- 确认每个邻居名称都能解析到节点 LLDP 发现的某台 leaf hostname。它也可以解析到另一个
  同 role 的 `Switch` 资源。无法解析的名称会停止本次协调,旧 label 保留。查看 Controller
  日志:

  ```bash
  kubectl -n unifabric-system logs deployment/unifabric-controller --tail 200
  ```

- 确认没有 `Switch` 资源与节点发现的某台 leaf 同名。
- 如果物理网络只有 leaf 一层,只有 tier 1 是预期结果。

## 参考

- [网络拓扑发现](./README.zh.md):各发现方式,以及如何查看结果。
- [LLDP 发现](./lldp-discovery.zh.md):用 switch-agent 自动发现。
- [Switch 参考](../../reference/switch.zh.md):`Switch` 资源的 `spec`、`status` 和
  annotation。
- [Scale-out 拓扑设计](../../design/scaleout-topology.zh.md):Controller 如何合并节点
  LLDP 与邻居声明。
