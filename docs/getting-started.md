# Deploy Unifabric

中文版: [getting-started.zh.md](./getting-started.zh.md)

## Introduction and Deployment Goals

This guide installs the Unifabric base components with Helm: the Controller, the
Agent, and the observability components. It then selects the topology-discovery
method for the physical network of the cluster: SONiC RoCE, InfiniBand, or
Spectrum-X.

After you finish this guide, the cluster will show the following results:

- A `FabricNode` resource exists for every node and lists its RDMA NICs by
  purpose: scale-out, storage, or scale-up.
- Nodes carry `scale-out.unifabric.io/tier-N` topology labels for schedulers.
- Node RDMA metrics are exposed by the Agent metrics endpoint for Prometheus
  to collect.

## Prerequisites

Make sure that each item below is true before you continue:

- [ ] You have access to the target Kubernetes cluster.
- [ ] `kubectl` and Helm are installed on the host that you use for the
      installation.

## Installation

The command below installs the latest Unifabric release. To select a specific
version, read the [releases](https://github.com/unifabric-io/unifabric/releases)
page. Run it on the host with `kubectl` and Helm access to the cluster:

```bash
LATEST_TAG=$(curl -fsSL https://api.github.com/repos/unifabric-io/unifabric/releases/latest | grep '"tag_name":' | cut -d '"' -f4)
CHART_VERSION="${LATEST_TAG#v}"

helm upgrade --install unifabric oci://ghcr.io/unifabric-io/charts/unifabric \
  --version "${CHART_VERSION}" \
  --namespace unifabric-system \
  --create-namespace \
  --wait
```

```text
<!-- TODO: capture output from a real environment -->
```

By default, Unifabric classifies all RDMA NICs on a node as scale-out. If a node
has RDMA NICs for different purposes, add the matching selector to the
installation command with `--set`.

For example, a node has six RDMA NICs named `eth1` through `eth6`. The NICs
`eth1` through `eth4` carry scale-out traffic, and `eth5` and `eth6` carry
storage traffic. Reference architectures of standard GPU servers, such as
DGX H100/B200 and GB200/GB300 NVL72, commonly provide two dedicated storage
NICs. Add this option to the installation command:

```bash
--set fabricNode.storageInterfaceSelector="interface=eth5\,eth6"
```

Unifabric then classifies `eth5` and `eth6` as storage. The other four RDMA
NICs remain scale-out by the default rule. The selector also supports wildcards
and CIDRs, for example `interface=eth*,!eth9` or `cidr=172.17.0.0/16`. The
[Helm values reference](../chart/README.md) describes the complete selector
syntax.

If the cluster does not have Prometheus Operator or Grafana Operator, add these
options to the installation command:

```bash
--set nodeMetrics.serviceMonitor.enabled=false \
--set grafanaDashboard.enabled=false
```

In mainland China, add these options to accelerate image pulls:

```bash
--set global.registry=m.daocloud.io \
--set controller.image.repository=ghcr.io/unifabric-io/unifabric-controller \
--set agent.image.repository=ghcr.io/unifabric-io/unifabric-agent \
--set nvidiaTopograph.image.repository=ghcr.io/nvidia/topograph
```

## Configure Topology Discovery

Topology discovery converts the physical adjacency of the network into
hierarchical performance domains and writes them to Node labels.
`scale-out.unifabric.io/tier-1` is the tier closest to the Node, followed by
`tier-2`, `tier-3`, and so on. Select the section for the physical network of
the cluster and add the matching options to the installation command:

| Physical network | Discovery method | Required configuration |
| --- | --- | --- |
| [SONiC RoCE](#sonic-roce-fabric) | Built-in Unifabric LLDP discovery | Enabled by default, no extra options |
| [InfiniBand](#infiniband-fabric) | NVIDIA Topograph | Two Helm options |
| [Spectrum-X](#spectrum-x-fabric) | NVIDIA Topograph with NetQ | A NetQ credentials Secret and four Helm options |

### SONiC RoCE Fabric

Use this section when SONiC switches carry the RoCE network. RoCE is RDMA over
Converged Ethernet. Make sure that LLDP is enabled on every switch. LLDP is a
protocol that devices use to announce themselves to their link peers.

The chart sets `topoDiscovery.scaleOut.mode` to `unifabric-roce` by
default, so Unifabric discovers the scale-out topology from LLDP neighbors
without extra options. If the RoCE network also carries the storage network, add
this option to the installation command:

```bash
--set topoDiscovery.storage.mode=unifabric-roce
```

With node-side LLDP alone, discovery produces only tier 1: the leaf switches
that the nodes connect to. To discover the spine and core tiers above the
leaves, declare the upper switches or run the switch-agent on the switches after
the installation. The [topology discovery guide](./usage/topology-discovery.md)
describes both options.

### InfiniBand Fabric

Use this section when the cluster uses an NVIDIA InfiniBand fabric, for example
Mellanox NICs in IB mode connected to IB switches. Make sure that RDMA devices
in IB mode exist on the nodes.

Add these options to the installation command to select NVIDIA Topograph as the
topology writer for the scale-up and scale-out networks:

```bash
--set topoDiscovery.scaleUp.mode=nv-topograph \
--set topoDiscovery.scaleOut.mode=nv-topograph
```

### Spectrum-X Fabric

Use this section when the cluster uses Spectrum-X switches and NetQ manages the
network. NetQ is the NVIDIA network management platform for Spectrum-X fabrics,
and NVIDIA Topograph reads the fabric topology through the NetQ API. Make sure
that each item below is true:

- [ ] NetQ manages the Spectrum-X fabric, and NetQ holds the fabric topology
      data.
- [ ] The cluster can reach the NetQ API.
- [ ] You have the NetQ API URL, the username, and the password.

First create the namespace and the NetQ credentials Secret. The commands below
do not write the credentials to local files, Helm values, or ConfigMaps. The
Secret must contain a key named `credentials.yaml`:

```bash
kubectl create namespace unifabric-system --dry-run=client -o yaml | kubectl apply -f -

read -r -p "NetQ username: " NETQ_USERNAME
read -r -s -p "NetQ password: " NETQ_PASSWORD
printf '\n'
printf 'username: %s\npassword: %s\n' "${NETQ_USERNAME}" "${NETQ_PASSWORD}" |
  kubectl -n unifabric-system create secret generic netq-credentials \
    --from-file=credentials.yaml=/dev/stdin
unset NETQ_USERNAME NETQ_PASSWORD
```

```text
<!-- TODO: capture output from a real environment -->
```

A production cluster must also enable encryption at rest for Kubernetes Secrets
and restrict the RBAC permissions that read this Secret.

Then add these options to the installation command:

```bash
--set topoDiscovery.scaleOut.mode=nv-topograph \
--set nvidiaTopograph.provider.name=netq \
--set-string nvidiaTopograph.provider.params.apiUrl=https://netq.example.com \
--set-string nvidiaTopograph.credentialsSecretName=netq-credentials
```

Option description:

| Helm value | Purpose |
| --- | --- |
| `topoDiscovery.scaleOut.mode` | Set to `nv-topograph` so that NVIDIA Topograph writes the scale-out labels. |
| `nvidiaTopograph.provider.name` | Set to `netq` to discover the topology through the NetQ API. |
| `nvidiaTopograph.provider.params.apiUrl` | The NetQ API URL. |
| `nvidiaTopograph.credentialsSecretName` | The name of the existing NetQ credentials Secret. The Secret must contain the `credentials.yaml` key. The credentials are mounted read-only into the Topograph Pod. |

## Verification

### 1. Inspect FabricNode NIC Information

Make sure that the Controller and Agent Pods are running and that a `FabricNode`
exists for every node:

```bash
kubectl -n unifabric-system get pods
kubectl get fabricnodes
kubectl get fabricnode <node-name> -o yaml
```

```text
<!-- TODO: capture output from a real environment -->
```

Make sure that `status.totalNics`, `status.scaleOutNics`, and
`status.storageNics` match the real NIC configuration of the node. For the
six-NIC example above, the result looks like this:

```yaml
apiVersion: unifabric.io/v1beta1
kind: FabricNode
metadata:
  name: node1
status:
  totalNics: 6
  scaleOutNics:
    - name: eth1
      rdmaDeviceName: mlx5_0
      rdma: true
      state: up
    - name: eth2
      rdmaDeviceName: mlx5_1
      rdma: true
      state: up
    - name: eth3
      rdmaDeviceName: mlx5_2
      rdma: true
      state: up
    - name: eth4
      rdmaDeviceName: mlx5_3
      rdma: true
      state: up
  storageNics:
    - name: eth5
      rdmaDeviceName: mlx5_4
      rdma: true
      state: up
    - name: eth6
      rdmaDeviceName: mlx5_5
      rdma: true
      state: up
```

### 2. Check the Node RDMA Metrics Endpoint

Query the Agent metrics endpoint directly to make sure that the metrics
collector produces `unifabric_` metrics:

```bash
POD_IP=$(kubectl -n unifabric-system get pod \
  -l app.kubernetes.io/component=unifabric-agent \
  -o jsonpath='{.items[0].status.podIP}')

curl -s "http://${POD_IP}:8082/metrics" | grep '^unifabric_' | head
```

```text
<!-- TODO: capture output from a real environment -->
```

If the endpoint returns metrics with the `unifabric_` prefix, the Agent metrics
collection works. The cluster carries no workload traffic at this point, so do
not interpret the metric values. The [RDMA metrics guide](./usage/rdma-metrics.md)
describes the metrics and the Grafana dashboards.

### 3. Inspect Node Topology Labels

After topology discovery completes, the discovery result is written to Node
labels:

```bash
kubectl get nodes -L scale-out.unifabric.io/tier-1,scale-out.unifabric.io/tier-2,kubernetes.io/hostname
```

```text
<!-- TODO: capture output from a real environment -->
```

Make sure that every participating Node has consecutive
`scale-out.unifabric.io/tier-N` labels, where tier 1 is closest to the Node. On
a SONiC RoCE fabric with node-side LLDP alone, tier 1 only is the expected
result. The [topology discovery guide](./usage/topology-discovery.md) describes
the label semantics, the `Switch` and `Topology` resources, and the discovery of
the switch tiers above the leaves.

## Troubleshooting

### FabricNode Does Not Discover the Expected RDMA NICs

First make sure that the preconditions hold:

1. Make sure that the Agent Pods are running:

   ```bash
   kubectl -n unifabric-system get daemonset unifabric-agent
   kubectl -n unifabric-system logs daemonset/unifabric-agent -c agent
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

2. Make sure that the RDMA devices of the node are visible to the Agent:

   ```bash
   kubectl -n unifabric-system exec daemonset/unifabric-agent -c agent -- \
     ls /sys/class/infiniband
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

3. Make sure that `fabricNode.scaleOutInterfaceSelector`,
   `storageInterfaceSelector`, and `scaleUpInterfaceSelector` match the real
   interface names or CIDRs.
4. Make sure that no NIC is misclassified. NICs that the storage or scale-up
   selector matches do not appear in `status.scaleOutNics`.

### The Agent Metrics Endpoint Has No Data

Make sure that Agent metrics are enabled, then inspect the Service and the
ServiceMonitor:

```bash
helm get values unifabric -n unifabric-system
kubectl -n unifabric-system get service unifabric-agent-metrics
kubectl -n unifabric-system get servicemonitor unifabric-agent-metrics
```

```text
<!-- TODO: capture output from a real environment -->
```

If `nodeMetrics.serviceMonitor.enabled` was disabled during the installation,
the absence of a `ServiceMonitor` is expected. Then repeat the endpoint check
in [Verification](#2-check-the-node-rdma-metrics-endpoint):

- If the endpoint returns no metrics, make sure that `nodeMetrics.enabled=true`.
  Then read the Agent logs and the RDMA devices under `/sys/class/infiniband`.
- If the endpoint returns metrics but Prometheus or Grafana has no data, use the
  [RDMA metrics guide](./usage/rdma-metrics.md) to troubleshoot the
  observability pipeline.

### Nodes Have No Topology Labels

The discovery pipeline depends on the physical network. Follow the
[troubleshooting section of the topology discovery guide](./usage/topology-discovery.md#troubleshooting).

## Uninstall

Remove the Helm release:

```bash
helm uninstall unifabric --namespace unifabric-system --wait
```

```text
<!-- TODO: capture output from a real environment -->
```

If you no longer need the CRDs, delete them. This also deletes all `FabricNode`,
`Switch`, and `Topology` resources:

```bash
kubectl delete crd fabricnodes.unifabric.io switches.unifabric.io topologies.unifabric.io
```

```text
<!-- TODO: capture output from a real environment -->
```

## Next Steps

- Go deeper into topology discovery: declare upper SONiC switches, deploy the
  switch-agent, and understand the label semantics and the `Topology` resource
  with the [topology discovery guide](./usage/topology-discovery.md).
- Use the labels in a scheduler with the
  [Kueue TAS workload example](./usage/workload-tas.md).
- Render the topology with the
  [topology visualization guide](./usage/topology-visualization.md).
- Observe RDMA metrics with the [RDMA metrics guide](./usage/rdma-metrics.md).
- Tune the installation with the [Helm values reference](../chart/README.md).
