# eBPF-based RDMA Tracing

中文版: [rdma-flow.zh.md](./rdma-flow.zh.md)

## Introduction

NIC counters record how much traffic a NIC or a Pod sends. They do not record where the traffic goes.
eBPF-based RDMA tracing attaches eBPF probes to the submit path of `libibverbs`.
It attributes each RDMA send to a flow between Pods, without application changes.
Operators use it to see the peers, the traffic volume, and the message sizes of training jobs and inference jobs.
This document tells you how to enable, verify, and troubleshoot eBPF-based RDMA tracing.

| Item | Description |
| --- | --- |
| Observation location | ② RDMA submit path |
| Applicable networks | Scale-out, Storage |
| Data source | eBPF uprobes on the submit path of `libibverbs` |
| Observed object | CPU-initiated RDMA traffic |
| Attribution levels | Source Pod and destination Pod, source device and destination device, and the top-level workloads of both ends |
| Output | `rdma_flow_*` metrics, `RDMAEndpoint` resources, and the "RDMA send details (CPU initiated)" row in the Pod and AI Workload dashboards |
| Node side requirements | Linux kernel 5.8 or later, with BPF ring buffer and uprobe support |
| Helm configuration | `agent.ebpfFlow.enabled=true`. An InfiniBand network also needs `agent.ebpfFlow.endpoint.allPods=true` |

The Agent rebuilds a flow in this way:

- Data plane: the eBPF probes record the bytes and WRs that each QP sends, and keep them in BPF maps.
- Source: the Agent finds the source Pod from the cgroup of the sending process.
- Destination: the Agent finds the destination Pod from the destination address of the QP. A RoCE network uses the GID, that is, the Pod IP or MAC. An InfiniBand network uses the LID.
- `RDMAEndpoint`: each Agent publishes the devices, GIDs, LIDs, and QPs of the RDMA Pods on its node. The Agents on other nodes use them to resolve the destination Pod.

eBPF-based RDMA tracing exports these metrics:

| Metric | Meaning | Requirement |
| --- | --- | --- |
| `rdma_flow_bytes_total` | The total bytes that one flow sent. | `agent.ebpfFlow.enabled=true` |
| `rdma_flow_wrs_total` | The total WRs that one flow sent. | `agent.ebpfFlow.enabled=true` |
| `rdma_flow_qps` | The number of active QPs of one flow now. | `agent.ebpfFlow.enabled=true` |
| `rdma_flow_sends_total` | The total number of sends of one flow. | Also enable [RDMA send event storage](./rdma-flow-events.md) |
| `rdma_flow_send_bytes` | The distribution of bytes per send (histogram). | Also enable [RDMA send event storage](./rdma-flow-events.md) |

Each sample has these labels:
`src_pod_namespace`, `src_pod_name`, `src_pod_top_owner_kind`, `src_pod_top_owner_namespace`, `src_pod_top_owner_name`,
`dst_pod_namespace`, `dst_pod_name`, `dst_pod_top_owner_kind`, `dst_pod_top_owner_namespace`, `dst_pod_top_owner_name`,
`src_device`, and `dst_device`.

Limits:

- Only CPU-initiated RDMA traffic is observable. GPU-initiated RDMA traffic does not use `libibverbs`, so the probes do not see it. See [Scope](#scope).
- Only requests that user space submits through `libibverbs` are observable. Kernel-space RDMA protocols, for example NVMe over RDMA, do not use `libibverbs`.
- The Agent records the traffic on the side that submits the request. The source Pod of a flow is the Pod that submits the WR. For RDMA Read, the data actually goes from the destination Pod to the source Pod.
- A flow whose source Pod or destination Pod is not resolved yet is not counted. After the Agent resolves it, it adds the earlier traffic in one step.

eBPF-based RDMA tracing covers ② the RDMA submit path in the traffic observability scope. For the full scope and the observation methods of the other locations, see [High-speed network traffic observability](./README.md#traffic-observability-scope).

## Scope

eBPF-based RDMA tracing supports only **CPU-initiated RDMA**.
The CPU must submit the send work requests (WRs) through `libibverbs`.
Data in GPU memory does not prevent observation.
GPUDirect RDMA is observable when the CPU submits the WRs.

**GPU-initiated RDMA** paths, for example GDAKI, GPUDirect Async, and IBGDA, use GPU kernels to drive the NIC directly. They bypass `libibverbs` and are not observable.

One framework can use both types of paths.
Check the communication path that is actually in use.

### Inference frameworks

| Inference framework | Communication scenario | RDMA flow observable | Information as of version |
| --- | --- | --- | --- |
| SGLang | TP | Uses NCCL; supported. | v0.5.21 |
| | PP | Uses NCCL; supported. | v0.5.21 |
| | EP | Depends on the SGLang startup argument `--moe-a2a-backend`:<br />- `none` (default): uses NCCL; supported.<br />- DeepEP, Mooncake, NIXL-EP, FlashInfer: not supported. | v0.5.21 |
| | KV cache transfer in Prefill/Decode disaggregation | Selected with the SGLang startup argument `--disaggregation-transfer-backend`:<br />- NIXL, Mooncake: both submit RDMA requests from the CPU; supported.<br /><br />For configuration details, see the [PD Disaggregation](https://docs.sglang.io/docs/advanced_features/pd_disaggregation) documentation. | v0.5.21 |
| vLLM | TP | Uses NCCL; supported. | v0.31.0 |
| | PP | Uses NCCL; supported. | v0.31.0 |
| | EP (MoE communication) | Depends on the vLLM startup argument `--all2all-backend`:<br />- `allgather_reducescatter`: uses NCCL collectives; supported.<br />- `deepep_high_throughput`, `deepep_low_latency`, `nixl_ep`, `flashinfer_nvlink_two_sided`, `flashinfer_nvlink_one_sided`, `mori_high_throughput`, `mori_low_latency`, `deepep_v2`, `moonep`: not supported.<br />For details, see the [moe_kernel_features](https://docs.vllm.ai/en/latest/design/moe_kernel_features/) documentation. | v0.31.0 |
| | KV cache transfer in Prefill/Decode disaggregation | Depends on the connector and backend configuration in the vLLM startup argument `--kv-transfer-config`:<br />- NixlConnector: select the backend with the `backends` array in `kv_connector_extra_config`:<br />&nbsp;&nbsp;- `UCX` (default): submits RDMA requests from the CPU; supported.<br />&nbsp;&nbsp;- `LIBFABRIC`: submits RDMA requests from the CPU; supported.<br />For configuration details, see [nixl_connector_usage](https://docs.vllm.ai/en/latest/features/nixl_connector_usage/).<br /><br />- MooncakeConnector: supported when `mooncake_protocol` is `rdma` (the default). Other values are not supported. For configuration details, see [mooncake_connector_usage](https://github.com/vllm-project/vllm/blob/main/docs/features/mooncake_connector_usage.md).<br /><br />- LMCacheConnector: supported. LMCache uses the UCX communication library, and uses the RDMA network first when RDMA is available.<br /><br />- MoRIIOConnector: supported when `backend` is `rdma` (the default). Other values are not supported. For configuration details, see [MoRIIOConnector](https://docs.vllm.ai/en/latest/features/moriio_connector_usage/#rdma-backend). | v0.31.0<br />LMCache: v0.5.5 |

### Communication libraries

| Communication library | RDMA flow observable | Information as of version |
| --- | --- | --- |
| NCCL | Depends on the actual RDMA submit path:<br />- With the [NCCL API](https://github.com/NVIDIA/nccl/blob/v2.32.3-1/src/nccl.h.in#L602), the CPU proxy thread submits RDMA requests through `libibverbs`; supported.<br />- With the [NCCL-GIN API](https://github.com/NVIDIA/nccl/blob/v2.32.3-1/src/include/nccl_device/gin.h#L356), if the `NCCL_GIN_TYPE` environment variable is empty (the default) or `3` (GDAKI), the GPU drives the NIC directly when it can and bypasses `libibverbs`; not supported.<br />- With the [NCCL-GIN API](https://github.com/NVIDIA/nccl/blob/v2.32.3-1/src/include/nccl_device/gin.h#L356), if you set `NCCL_GIN_TYPE=2` (Proxy mode), the CPU submits RDMA requests; supported. | v2.32.3 |
| UCX | Select the transport with the `UCX_TLS` environment variable:<br />- `rc`, `dc`: supported.<br />- Other transports: not supported. | v1.22.0 |
| DeepEP | Not supported. DeepEP submits RDMA requests directly from the GPU. | - |
| FlashInfer | Not supported. FlashInfer starts RDMA from the GPU, and the CPU is not on the critical path. | - |
| LIBFABRIC | Supported. | v2.7.0 |

## Prerequisites

- You meet the prerequisites in [High-speed network traffic observability](./README.md#prerequisites).
- The Linux kernel of the nodes is 5.8 or later, and supports BPF ring buffer and uprobe.
- The application submits RDMA requests through the host `libibverbs` path. An application that only uses RDMA devices or GPU buffers is not always observable.

## Configuration

Run this command to enable eBPF-based RDMA tracing:

```bash
helm upgrade --install unifabric oci://ghcr.io/unifabric-io/charts/unifabric \
  --namespace unifabric-system \
  --reuse-values \
  --set agent.ebpfFlow.enabled=true \
  --wait
```

If the Scale-out network or the Storage network is InfiniBand, also add `--set agent.ebpfFlow.endpoint.allPods=true`.
An InfiniBand network addresses the peer by LID, and the Agent cannot find the Pod from a LID.
With this setting, the Agent publishes an `RDMAEndpoint` for each RDMA Pod, and the Agents on other nodes use it to resolve the destination Pod.
By default, the Agent publishes `RDMAEndpoint` resources only for hostNetwork Pods and for Pods that are not in the IP index yet. It resolves the other Pods by Pod IP.

After you enable it, the Agent sums the data in the BPF maps into flow metrics every 5 seconds (`agent.ebpfFlow.reportInterval`).
This level does not make per-send events, and does not need a storage backend.
If you need a record of each send and the distribution of message sizes, also configure [RDMA send event storage](./rdma-flow-events.md).

For all settings, see `agent.ebpfFlow.*` in the [Helm values reference](../../../chart/README.md).

## Verification

1. Make sure that the Agent loaded the BPF programs and attached the probes:

   ```bash
   kubectl -n unifabric-system logs ds/unifabric-agent -c agent | \
     grep -E 'bpf_collection_loaded|bootstrap_probes_attached'
   ```

   <!-- TODO: capture output from a real environment -->

2. After the application makes RDMA traffic, start a port forward to an Agent Pod and view the flow metrics:

   ```bash
   AGENT_POD=$(kubectl -n unifabric-system get pod \
     -l app.kubernetes.io/component=unifabric-agent \
     --field-selector=status.phase=Running \
     -o jsonpath='{.items[0].metadata.name}')
   kubectl -n unifabric-system port-forward "pod/$AGENT_POD" 8082:8082
   ```

   In a second terminal, run:

   ```bash
   curl -s http://127.0.0.1:8082/metrics | grep '^rdma_flow_bytes_total'
   ```

   <!-- TODO: capture output from a real environment -->

3. Make sure that the `RDMAEndpoint` resources exist:

   ```bash
   kubectl get rdmaendpoints -A
   ```

   <!-- TODO: capture output from a real environment -->

4. In Grafana, open the Unifabric RDMA Pod dashboard or the Unifabric RDMA AI Workload dashboard, and expand the "RDMA send details (CPU initiated)" row.
   The panels need this configuration:

   | Panels | Configuration needed |
   | --- | --- |
   | Cumulative bytes sent, Cumulative WRs, Active QPs, Cumulative bytes from source Pod to destination Pods, Cumulative bytes by source Pod, Cumulative bytes by peer Pod, Workload outgoing flow details | The configuration in this document |
   | Cumulative sends, Peer Pods in time range, Cumulative sends and size percentiles by source and destination Pod, Cumulative traffic by device pair | Also enable [RDMA send event storage](./rdma-flow-events.md) |
   | Top 20 most common message sizes, Processes and QPs per source Pod, Latest send records | Also enable RDMA send event storage with the Elasticsearch backend |

   ![Unifabric RDMA Pod dashboard](../../images/pods.png)

## Troubleshooting

### The dashboards have no RDMA flow data

The commands below use the installation namespace `unifabric-system`.
If you used a different namespace, change `NAMESPACE`.

1. Make sure that the application meets the observation conditions:

   - The communication library actually uses the CPU-initiated `libibverbs` path. Use [Scope](#scope), and check the NCCL, UCX, or application logs.
   - The application actually makes RDMA traffic while you troubleshoot. Check the application logs, the communication test results, or [RDMA metrics](./rdma-metrics.md).

   If one of the conditions is false, there is no flow data. Do not continue to check the Agent or Grafana.

2. Make sure that the Agent runs on all target nodes, and select an Agent Pod on a node that has RDMA traffic:

   ```bash
   NAMESPACE=unifabric-system
   kubectl -n "$NAMESPACE" get daemonset,pod \
     -l app.kubernetes.io/component=unifabric-agent -o wide

   AGENT_POD=$(kubectl -n "$NAMESPACE" get pod \
     -l app.kubernetes.io/component=unifabric-agent \
     --field-selector=status.phase=Running \
     -o jsonpath='{.items[0].metadata.name}')
   echo "$AGENT_POD"
   ```

   `DESIRED`, `CURRENT`, and `READY` of the DaemonSet must be equal.
   Also make sure that `agent.ebpfFlow.enabled=true`. If it is not enabled, the Agent does not register the `rdma_flow_*` metrics.

3. View the metrics of this Agent directly. This tells a collection problem from a Prometheus or Grafana problem. In one terminal, start a port forward:

   ```bash
   kubectl -n "$NAMESPACE" port-forward "pod/$AGENT_POD" 8082:8082
   ```

   In a second terminal, run:

   ```bash
   curl -s http://127.0.0.1:8082/metrics | grep '^rdma_flow_'
   curl -s http://127.0.0.1:8082/metrics | \
     grep -E '^rdma_flow_agent_(qp_infos|qp_stats|qp_matched|qp_unmatched|qp_unresolved) '
   ```

   - There are no `rdma_flow_*` metrics: make sure that the feature is enabled, and check the BPF load logs in the next step.
   - `rdma_flow_agent_qp_stats` is `0`: the Agent did not see sends. Make sure that the application makes CPU-initiated RDMA traffic, and check that the probes are attached.
   - `rdma_flow_agent_qp_stats` is more than `0`, but `rdma_flow_agent_qp_matched` is `0`: check `qp_infos` and the provider callbacks.
   - `rdma_flow_agent_qp_unresolved` is more than `0`: the Agent found the QPs, but cannot resolve the source or destination address to a Pod. On an InfiniBand network, make sure that `agent.ebpfFlow.endpoint.allPods=true` and that all `RDMAEndpoint` resources exist.
   - `rdma_flow_bytes_total`, `rdma_flow_wrs_total`, or `rdma_flow_qps` exists: the Agent collects correctly. Continue to check Prometheus and Grafana.

4. Check the BPF objects, the probes, and the pinned BPF maps:

   ```bash
   kubectl -n "$NAMESPACE" logs "$AGENT_POD" -c agent | \
     grep -E 'bpf_collection_loaded|bootstrap_probes_attached|provider_callback_restored|callback_received|load bpf collection|attach_bootstrap_probes'

   kubectl -n "$NAMESPACE" exec "$AGENT_POD" -c agent -- \
     ls -l /sys/fs/bpf/unifabric
   ```

   The logs must show `bpf_collection_loaded` and `bootstrap_probes_attached`.
   After the application creates a QP, the logs must show `provider_callback_restored`.
   `callback_received` shows only at the debug log level.
   `/sys/fs/bpf/unifabric` must contain `qp_infos` and `qp_stats`.

   If `bpftool` is installed on the node, you can view the content of the maps:

   ```bash
   bpftool map dump pinned /sys/fs/bpf/unifabric/qp_infos
   bpftool map dump pinned /sys/fs/bpf/unifabric/qp_stats
   ```

   If `qp_infos` is empty, the QP usually did not enter the RTR state yet, or the Agent did not catch the provider callback.
   If `qp_stats` is empty, the application usually did not submit sends through a supported `libibverbs` path.

5. If the Agent has flow metrics locally, check the Prometheus scrape. See [RDMA metrics](./rdma-metrics.md#prometheus-has-no-metrics).
   Then run this query in Prometheus or Grafana Explore:

   ```promql
   sum(rate(rdma_flow_bytes_total[5m]))
   ```

   If the query has results but the dashboard is empty, check the time range of the dashboard. Also make sure that the namespace, workload, Pod, and other variables select objects that make RDMA traffic.

### Some panels have no data

See the panel table in [Verification](#verification).
The panels for send counts and message sizes need [RDMA send event storage](./rdma-flow-events.md).

### Workload labels stop at an intermediate resource

See [RDMA metrics](./rdma-metrics.md#workload-labels-are-empty-or-stop-at-an-intermediate-resource).

## References

- [High-speed network traffic observability](./README.md): the traffic observability scope, and the rules of the Pod identity association.
- [RDMA send event storage](./rdma-flow-events.md): record each RDMA send.
- [eBPF RDMA flow attribution design](../../design/rdma-flow.md): the design of the data plane, the attribution, and `RDMAEndpoint`.
- [RDMAEndpoint reference](../../reference/rdmaendpoint.md): the fields of `RDMAEndpoint`.
- [Helm values reference](../../../chart/README.md): all `agent.ebpfFlow.*` parameters.
