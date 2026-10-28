# RDMA Metrics

中文版: [rdma-metrics.zh.md](./rdma-metrics.zh.md)

## Introduction

The Agent on each node collects the counters of the RDMA NICs and exports them as Prometheus metrics.
The metrics cover host NICs and Pod NICs.
Each sample has a network type label. Pod-level samples also have Pod and workload labels.
This document tells you how to configure, verify, and view RDMA metrics.

| Item | Description |
| --- | --- |
| Observation location | ③ Host and Pod RDMA NICs |
| Applicable networks | RDMA NICs in the Scale-out, Storage, and Scale-up networks |
| Data sources | `counters` and `hw_counters` in `/sys/class/infiniband`, NIC attributes in `/sys/class/net`, and ethtool priority counters |
| Attribution levels | Node, NIC, Pod, workload |
| Output | `unifabric_*` metrics, and the RDMA Cluster, Node, Pod, and AI Workload dashboards |
| Node side requirements | Linux nodes. RDMA devices are visible in `/sys/class/infiniband`. Pod attribution needs containerd. |
| Helm configuration | `nodeMetrics.enabled=true` (default), `grafanaDashboard.enabled=true` (default) |

The metrics are in these categories:

| Category | Examples | Source |
| --- | --- | --- |
| Throughput | `unifabric_port_xmit_data`, `unifabric_port_rcv_data`, `unifabric_unicast_xmit_packets` | `counters` |
| Congestion | `unifabric_np_ecn_marked_roce_packets`, `unifabric_np_cnp_sent`, `unifabric_rp_cnp_handled`, `unifabric_out_of_sequence` | `hw_counters` |
| Errors and retransmission | `unifabric_local_ack_timeout_err`, `unifabric_packet_seq_err`, `unifabric_req_cqe_error`, `unifabric_out_of_buffer` | `hw_counters` |
| NIC attributes | `unifabric_port_speed_mbps`, `unifabric_port_mtu`, `unifabric_port_oper_state`, `unifabric_rdma_device_tos` | `/sys/class/net`, device traffic class |
| Priority | `unifabric_rx_pause`, `unifabric_tx_pause`, `unifabric_rx_discards`, `unifabric_tx_discards` | ethtool |

The NIC driver supplies the counters in `hw_counters`.
Different NICs and driver versions supply different counters.
RoCE-only counters, for example ECN and CNP, are only on RoCE NICs.

All samples use the same label set. The main labels are:

| Label | Meaning |
| --- | --- |
| `node_name` | The node name. |
| `device` | The RDMA device, for example `mlx5_0`. |
| `ifname` | The NIC of the RDMA device. |
| `parent_ifname` | The PF NIC of a VF. For a PF, it is the same as `ifname`. |
| `kind` | The network of the NIC: `scaleOut`, `storage`, `scaleUp`, or empty. |
| `scope` | `host` or `pod`. |
| `pod_namespace`, `pod_name` | The Pod of a Pod-level sample. |
| `topowner_kind`, `topowner_namespace`, `topowner_name` | The top-level workload of the Pod. |
| `host_rdma` | Whether the Pod uses RDMA in host mode. |

For all labels, see the [RDMA metric model and Pod attribution design](../../design/rdma-metrics.md#metric-model).

Limits:

- RDMA metrics record how much traffic each NIC sends and receives. They do not record where the traffic goes. To see the peers, use [eBPF-based RDMA tracing](./rdma-flow.md).
- The metrics are the counter values at scrape time. The Agent does not keep history. Prometheus keeps the history.
- Only RDMA Pods have Pod-level samples. For the conditions, see [Identify RDMA Pods](#identify-rdma-pods).
- For a Pod in host mode, the Pod-level samples are copies of the host NIC samples. When many Pods in host mode share one NIC, their samples are the same.

RDMA metrics cover ③ the host and Pod RDMA NICs in the traffic observability scope. For the full scope and the observation methods of the other locations, see [High-speed network traffic observability](./README.md#traffic-observability-scope).

## Prerequisites

- You meet the prerequisites in [High-speed network traffic observability](./README.md#prerequisites).
- The RDMA devices are visible on the node:

  ```bash
  kubectl -n unifabric-system exec ds/unifabric-agent -c agent -- ls /sys/class/infiniband
  ```

## Configuration

RDMA metrics are enabled by default.
After the basic installation, the Agent already exports the metrics, and the chart already created the `ServiceMonitor` and the dashboards.
Change the configuration only in the scenarios below.

### Separate the network types

By default, the Agent classifies all RDMA NICs as `scaleOut`.
If the node has storage NICs or Scale-up NICs, set the related selector.
For example, to classify `eth5` and `eth6` as `storage`:

```bash
helm upgrade --install unifabric oci://ghcr.io/unifabric-io/charts/unifabric \
  --namespace unifabric-system \
  --reuse-values \
  --set fabricNode.storageInterfaceSelector="interface=eth5\,eth6" \
  --wait
```

The Agent sets `kind` in this order:

1. A NIC that matches `fabricNode.storageInterfaceSelector` is `storage`.
2. A NIC that matches `fabricNode.scaleUpInterfaceSelector` is `scaleUp`.
3. A NIC that matches `fabricNode.scaleOutInterfaceSelector` is `scaleOut`.
4. If `scaleOutInterfaceSelector` is not set, the other NICs are `scaleOut`.

The Agent matches `parent_ifname` first, then `ifname`.
A selector can be `interface=eth*,!eth9` or `cidr=172.17.0.0/16`.

> Note: These selectors also set the NIC classification for topology discovery.

### Configure Prometheus scraping

The chart creates the `ServiceMonitor` only when `nodeMetrics.enabled` and `serviceMonitor.enabled` are both `true`. Both are `true` by default.

If Prometheus selects `ServiceMonitor` objects by label, add those labels in the values:

```yaml
nodeMetrics:
  serviceMonitor:
    labels:
      release: prometheus
    interval: 15s
```

### Configure the dashboards

The `grafanaDashboard.*` parameters control how the dashboards are installed:

| Parameter | Default | Description |
| --- | --- | --- |
| `grafanaDashboard.kind` | `GrafanaDashboard` | Grafana Operator imports `GrafanaDashboard`. The Grafana sidecar imports `ConfigMap`. |
| `grafanaDashboard.language` | `en` | `en` installs the English dashboards. `zh` installs the Chinese dashboards with a `-zh` suffix on the uid and the resource name. `all` installs both. |
| `grafanaDashboard.instanceSelector` | `{}` | Selects the target instance when Grafana Operator manages many Grafana instances. |
| `grafanaDashboard.labels` | `{}` | Labels added to the dashboard resources. Use them when the Grafana sidecar finds `ConfigMap` objects by label. |

For example, to install the Chinese dashboards:

```bash
helm upgrade --install unifabric oci://ghcr.io/unifabric-io/charts/unifabric \
  --namespace unifabric-system \
  --reuse-values \
  --set grafanaDashboard.language=zh \
  --wait
```

## Pod and workload attribution

The Agent first attributes a sample to a Pod. Then it follows the `ownerReferences` of the Pod up to the top-level workload, for example a `PyTorchJob` or a `LeaderWorkerSet`.
The `pod_namespace`, `pod_name`, and `topowner_*` labels record the result.
The `*_top_owner_*` labels of [eBPF-based RDMA tracing](./rdma-flow.md) use the same resolution rules.

### Identify RDMA Pods

Only RDMA Pods have Pod-level samples.
The Agent identifies these two types of Pods as RDMA Pods and writes them to `FabricNode.status.rdmaPods`:

- SR-IOV mode: the `k8s.v1.cni.cncf.io/network-status` annotation of the Pod contains `rdma-device`. Multus and the SR-IOV CNI write this annotation.
- Host mode: the Pod sets `hostNetwork: true`, and at least one container is privileged and mounts `/dev/infiniband`.

To see the RDMA Pods that the Agent found on a node:

```bash
kubectl get fabricnode node1 -o jsonpath='{range .status.rdmaPods[*]}{.namespace}{"\t"}{.name}{"\t"}{.hostRDMA}{"\t"}{.topOwner.kind}/{.topOwner.name}{"\n"}{end}'
```

<!-- TODO: capture output from a real environment -->

> Note: Pods that get RDMA devices through DRA, for example with DraNet, are not identified as RDMA Pods yet. See [#47](https://github.com/unifabric-io/unifabric/issues/47).

> Note: Pod attribution in SR-IOV mode needs containerd. Pods on other container runtimes do not get Pod-level samples. Host-level samples are not affected.

### Resolve the top-level workload

The Agent reads each object in the owner chain, so it needs the `get` permission on these resources.
By default, the chart gives read permission on these resources:

- Kubernetes built-in resources: `Deployment`, `ReplicaSet`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`.
- The third-party resources in `agent.workloadOwnerRules`. The default value includes the workloads of Kubeflow Training Operator, Kubeflow Trainer, JobSet, LeaderWorkerSet, KubeRay, and Spark Operator.

If your workload controller is not in the list, add its API group and plural resource names to `agent.workloadOwnerRules`.
The values that you give are merged with the default values.
For example, to add Argo Workflows:

```yaml
agent:
  workloadOwnerRules:
    argoproj.io: [workflows]
```

The chart renders these rules into the `<release>-agent-workload-owners` ClusterRole.

When a permission is missing, the two observation methods behave differently:

- RDMA metrics: if the Agent cannot read any one object in the owner chain, the `topowner_*` labels are empty.
- eBPF-based RDMA tracing: the Agent follows only the resource types in [owners.go](../../../pkg/agent/ebpfflow/owners.go). When it finds a type that is not in the list, it uses that level as the top-level workload. To follow the chain further, you must add the type to `owners.go`.

## Verification

1. Make sure that the Service and the `ServiceMonitor` exist:

   ```bash
   kubectl -n unifabric-system get service unifabric-agent-metrics
   kubectl -n unifabric-system get servicemonitor unifabric-agent-metrics
   ```

2. In one terminal, start a port forward to an Agent Pod:

   ```bash
   AGENT_POD=$(kubectl -n unifabric-system get pod \
     -l app.kubernetes.io/component=unifabric-agent \
     --field-selector=status.phase=Running \
     -o jsonpath='{.items[0].metadata.name}')
   kubectl -n unifabric-system port-forward "pod/$AGENT_POD" 8082:8082
   ```

3. In a second terminal, make sure that the NIC classification is correct:

   ```bash
   curl -s http://127.0.0.1:8082/metrics | grep '^unifabric_port_xmit_data' | grep 'scope="host"'
   ```

   <!-- TODO: capture output from a real environment -->

   The `kind` of each NIC must agree with the selector configuration.

4. If the node has RDMA Pods, make sure that Pod-level samples exist:

   ```bash
   curl -s http://127.0.0.1:8082/metrics | grep '^unifabric_port_xmit_data' | grep 'scope="pod"'
   ```

   <!-- TODO: capture output from a real environment -->

5. Make sure that the dashboard resources exist:

   ```bash
   kubectl -n unifabric-system get grafanadashboard -l app.kubernetes.io/component=unifabric
   ```

   If `grafanaDashboard.kind=ConfigMap`, run:

   ```bash
   kubectl -n unifabric-system get configmap -l app.kubernetes.io/component=unifabric
   ```

## View the dashboards

RDMA metrics use these four dashboards.
Start from the cluster overview, then go down to a node, a Pod, or a workload.

### Unifabric RDMA Cluster

The overview of the cluster. It has these rows:

- Summary: the number of nodes, workloads, and Pods with RDMA, and the read and write throughput and throughput usage of the cluster and of each node.
- Hotspot: the 10 nodes, Pods, and workloads with the highest read and write throughput.
- Node Status: pause, discards, ECN marks, CNP, and out-of-sequence packets of the PF on each node.
- Pod Status: the 10 Pods with the most out-of-sequence packets and discards.

![Unifabric RDMA Cluster dashboard](../../images/cluster.png)

### Unifabric RDMA Node

The details of each NIC on one node. It has these rows:

- Throughput: throughput usage of each device, and read and write throughput of host devices and Pod devices.
- Basic (ECN/CNP/SEQ): pause and discards of each priority, and ECN marks, CNP, and out-of-sequence packets of each device.
- Detail: received Write, Read, and Atomic requests, and the error counters.
- RoCE: RoCE adaptive retransmission and slow restart.
- QoS: the traffic class of each device.
- Interface: the list of network devices.

![Unifabric RDMA Node dashboard](../../images/nodes.png)

### Unifabric RDMA Pod

The NIC data of one Pod. It has these rows:

- Summary, RDMA basic statistics: the send and receive throughput of the Pod NICs.
- RDMA error statistics: the errors and retransmissions of the Pod NICs.
- RDMA send details (CPU initiated): comes from eBPF-based RDMA tracing. See [eBPF-based RDMA tracing](./rdma-flow.md#verification).

![Unifabric RDMA Pod dashboard](../../images/pods.png)

### Unifabric RDMA AI Workload

The data summed by top-level workload.
Use the `kind`, `namespace`, and `name` variables to select one workload, for example a `PyTorchJob`.
The dashboard shows the traffic of all Pods of this workload.
The "RDMA send details (CPU initiated)" row comes from eBPF-based RDMA tracing.

![Unifabric RDMA AI Workload dashboard](../../images/workloads.png)

## Troubleshooting

### There are no RDMA metrics

1. Make sure that the Agent runs correctly:

   ```bash
   kubectl -n unifabric-system get daemonset unifabric-agent
   kubectl -n unifabric-system logs ds/unifabric-agent -c agent --tail 200
   ```

2. Make sure that the RDMA devices are visible on the node:

   ```bash
   kubectl -n unifabric-system exec ds/unifabric-agent -c agent -- ls /sys/class/infiniband
   ```

   If the directory is empty, check the NIC driver and the RDMA kernel modules.

3. Make sure that the `FabricNode` of the node exists:

   ```bash
   kubectl get fabricnodes
   ```

   If the `FabricNode` does not exist, the Agent returns empty metrics.

4. If the Agent has metrics locally but Prometheus has no data, see [Prometheus has no metrics](#prometheus-has-no-metrics).

### Prometheus has no metrics

1. Make sure that the Agent returns metrics locally. See [Verification](#verification).

2. Make sure that the Service and the `ServiceMonitor` exist:

   ```bash
   kubectl -n unifabric-system get service,servicemonitor \
     -l app.kubernetes.io/component=unifabric-agent
   ```

3. On the Targets page of Prometheus, make sure that the scrape target of `unifabric-agent-metrics` is `UP`.
   If Prometheus selects `ServiceMonitor` objects by label, add those labels to `nodeMetrics.serviceMonitor.labels`.

### There are no Pod-level metrics

1. Make sure that the Pod is in `FabricNode.status.rdmaPods`. See [Identify RDMA Pods](#identify-rdma-pods).
2. If the Pod is not in the list, make sure that it meets the conditions of SR-IOV mode or host mode.
3. If the Pod is in the list but has no Pod-level samples, make sure that the node uses containerd.

### Workload labels are empty or stop at an intermediate resource

- The `topowner_*` labels are empty: add read permission for the resources in the owner chain. See [Resolve the top-level workload](#resolve-the-top-level-workload).
- The `*_top_owner_kind` of `rdma_flow_*` stops at a Kubernetes built-in resource, for example `ReplicaSet`: make sure that the Agent can read the resource at the next level.
- The `*_top_owner_kind` of `rdma_flow_*` is a third-party resource, but it is not the top level: this type is not in `owners.go`.

### The kind label is empty or wrong

- Check `fabricNode.scaleOutInterfaceSelector`, `fabricNode.storageInterfaceSelector`, and `fabricNode.scaleUpInterfaceSelector`.
- If `scaleOutInterfaceSelector` is set, a NIC that matches no selector has an empty `kind`.
- The selectors match `parent_ifname` first. For a VF, use the PF name in the selector.

### The dashboards have no data

1. In Grafana Explore, query `unifabric_node_info` to make sure that the Prometheus data source has Unifabric metrics.
2. Make sure that the `cluster`, `node`, `namespace`, and other variables of the dashboard select objects that make RDMA traffic.
3. Make sure that there is RDMA traffic in the time range of the dashboard.

## References

- [High-speed network traffic observability](./README.md): the traffic observability scope, and the rules of the Pod identity association.
- [eBPF-based RDMA tracing](./rdma-flow.md): see the peers of RDMA traffic.
- [RDMA metric model and Pod attribution design](../../design/rdma-metrics.md): the data sources, the collection flow, and the metric model.
- [FabricNode reference](../../reference/fabricnode.md): the `status.rdmaPods` field.
- [Helm values reference](../../../chart/README.md): all `nodeMetrics.*`, `fabricNode.*`, and `grafanaDashboard.*` parameters.
