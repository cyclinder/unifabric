# API 参考

English version: [README.md](./README.md)

Unifabric 提供 3 个 `unifabric.io/v1beta1` 集群级自定义资源。

| 资源 | 说明 | 参考 |
| --- | --- | --- |
| `FabricNode` | 节点 Agent 上报的 RDMA 网卡、LLDP 邻居和 RDMA Pod 状态 | [fabricnode.zh.md](./fabricnode.zh.md) |
| `Switch` | 交换机声明、switch-agent 连接状态和交换机 LLDP 邻居 | [switch.zh.md](./switch.zh.md) |
| `Topology` | scale-out、scale-up 和 storage 网络的只读汇总拓扑 | [topology.zh.md](./topology.zh.md) |

CRD 的 OpenAPI schema 以 [`chart/crds`](../../chart/crds/) 中生成的清单为准。
