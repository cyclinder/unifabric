# Topology Discovery

中文版: [topology-discovery.zh.md](./topology-discovery.zh.md)

## Introduction

Topology discovery converts the physical adjacency of the network into
hierarchical performance domains, writes them to Node labels, and aggregates
them into the read-only `Topology` resource for schedulers such as Kueue,
Volcano, and KAI Scheduler, and for operators. This guide describes the meaning
of the discovery results for each physical network, the switch onboarding
options for SONiC RoCE fabrics, and the verification and troubleshooting of the
discovery pipeline.

The discovery results consist of:

- Node labels: `scale-out.unifabric.io/tier-N` represents the hierarchical
  performance domains of the scale-out network. Tier 1 is closest to the Node,
  followed by `tier-2`, `tier-3`, and so on. The label value identifies the
  performance domain of the Node. The built-in Unifabric discovery uses values
  of the form `tierN-groupM`, for example
  `scale-out.unifabric.io/tier-1=tier1-group1`. NVIDIA Topograph uses values
  derived from switch GUIDs, for example `tier-1=S-fc6a1c03006636c0`. Nodes
  with the same label value are in the same performance domain.
- The `FabricNode` and `Switch` resources expose the discovery input state, and
  the `Topology` resource exposes the aggregated read-only topology.

## Applicability

| Physical network | Discovery method | Helm configuration |
| --- | --- | --- |
| SONiC RoCE | Unifabric discovery over node and switch LLDP | `topoDiscovery.scaleOut.mode=unifabric-roce` (default) |
| InfiniBand | NVIDIA Topograph over `ibnetdiscover` | `topoDiscovery.scaleOut.mode=nv-topograph` |
| Spectrum-X | NVIDIA Topograph over the NetQ API | `topoDiscovery.scaleOut.mode=nv-topograph` plus the NetQ provider |

Storage network topology currently supports the SONiC RoCE discovery method only
(`topoDiscovery.storage.mode=unifabric-roce`). Scale-up labels come from the
GPU interconnect topology inside the node, and only the `nv-topograph` mode of
the InfiniBand scenario writes them.

## Prerequisites

Make sure that each item below is true before you continue:

- [ ] Unifabric is installed through [Deploy Unifabric](../getting-started.md),
      and the Helm options for the physical network are selected in
      [Configure Topology Discovery](../getting-started.md#configure-topology-discovery).
- [ ] The RDMA devices of the nodes are visible under `/sys/class/infiniband`.

## Configuration and Usage

### SONiC RoCE: LLDP Topology Discovery

Unifabric discovers the leaf, spine, and core topology from the node RDMA NICs,
the LLDP neighbors of the `FabricNode` resources, and optional switch-side LLDP
snapshots. Make sure that LLDP is enabled on every switch.

This section contains two cases. Select one case before you start:

| Case | When to use | What you need |
| --- | --- | --- |
| Case 1: node LLDP only | Security policy forbids software on the switches, or another team maintains the switches | Nodes discover their leaf switches through node-side LLDP. An administrator declares the tiers above the leaves with `Switch` resources and a neighbors annotation. |
| Case 2: switches run the `switch-agent` (recommended) | The operations team can run the Unifabric Switch Agent on the SONiC switches and wants automatic switch-to-switch discovery | LLDP is enabled on the switches. The Kubernetes Nodes that run `unifabric-controller` can reach TCP port `8090` of every switch. |

Both cases below use the same example network. `node1` and `node2` connect
through `leaf1` and `leaf2` and form `tier1-group1`. `node3` and `node4` connect
through `leaf3` and `leaf4` and form `tier1-group2`. Their shared uplink
`spine1` forms `tier2-group1`.

![RoCE switch topology discovery example](../images/sonic-roce-topology-example.jpg)

#### Case 1: Node LLDP Only

This case runs no software on the switches. When no `Switch` resource with the
same role exists, Unifabric enters node-only mode: it treats the switch
hostnames that the `FabricNode` LLDP data reports as synthetic leaves and
produces tier 1 only.

1. Wait until the Agents report LLDP neighbors. Tier 1 needs no switch-side
   action. In the example network, discovery produces two tier 1 domains, and
   every Node receives its own `scale-out.unifabric.io/tier-1` label.

2. To describe the spine, core, and other upper tiers, create one `Switch`
   resource for every upper switch and add the
   `unifabric.io/switch-neighbors` annotation to each resource. Do not create
   resources for the leaves, and do not set `spec.mgmtIP`. For the example
   network, apply this file from a host that can reach the cluster with
   `kubectl`:

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
   <!-- TODO: capture output from a real environment -->
   ```

> **Warning:** Do not write Kubernetes Node names into
> `unifabric.io/switch-neighbors`. The annotation lists switch neighbors only.
> Node-to-switch links come from the `FabricNode` LLDP data. A Node name in the
> array corrupts the computed topology.

> **Note:** Mode selection only checks whether the annotation key exists on
> every `Switch` with the same role. An empty value or `[]` also selects this
> mode but provides no neighbor links.

After you apply the file, all four Nodes receive both the tier 1 and the tier 2
labels. The [Verification](#verification) section shows the expected `Topology`
content.

#### Case 2: Automatic Discovery with the switch-agent

This case runs the `switch-agent` on every participating switch. The agent
reports LLDP snapshots to the Controller over mutual TLS (mTLS). As soon as any
`Switch` resource with the same role lacks the `unifabric.io/switch-neighbors`
key, Unifabric enters fully automatic mode and ignores all occurrences of the
annotation. To keep the configuration unambiguous, omit the annotation on all
`Switch` resources.

> **Note:** To deploy to many switches at once, use the
> [automated switch-agent deployment script](./deploy-switch-agent-script.md)
> instead of the manual steps below.

1. Export the pinned mTLS certificates. Run this on a host that can reach the
   cluster with `kubectl`:

   ```bash
   mkdir -p ./tmp-switch-mtls

   kubectl -n unifabric-system get secret switch-controller-mtls-agent -o jsonpath='{.data.tls\.crt}' | base64 -d > ./tmp-switch-mtls/tls.crt
   kubectl -n unifabric-system get secret switch-controller-mtls-agent -o jsonpath='{.data.tls\.key}' | base64 -d > ./tmp-switch-mtls/tls.key
   kubectl -n unifabric-system get secret switch-controller-mtls-agent -o jsonpath='{.data.peer\.crt}' | base64 -d > ./tmp-switch-mtls/peer.crt
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

2. Log in to the target switch and create the certificate directory:

   ```bash
   sudo mkdir -p /opt/unifabric-switch-agent/mtls
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

   Then copy `tls.crt`, `tls.key`, and `peer.crt` to
   `/opt/unifabric-switch-agent/mtls/` on the switch.

3. On the switch, make sure that LLDP sees the expected neighbors:

   ```bash
   lldpcli show neighbors -f json0
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

4. Start the `switch-agent` on the switch. The command below uses the Docker
   bridge network and publishes the gRPC port with `-p 8090:8090`. It reads
   LLDP through the mounted `lldpd` socket and needs neither host networking
   nor privileged permissions. If the switch cannot run containers, use the
   [switch-agent systemd installation](./switch-agent-systemd.md) instead.

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
   <!-- TODO: capture output from a real environment -->
   ```

   > **Note:** If the switch runs SONiC 202006 through 202311, add
   > `-e UNIFABRIC_SWITCH_AGENT_LLDP_CLI_VERSION=1.0.4`. Keep the default value
   > `1.0.16` for other versions.

   > **Note:** If the switch cannot mount `/run/lldpd.socket`, use the
   > [hostProc collection mode](./switch-agent-host-proc.md) instead. That mode
   > needs privileged permissions and mounts the host `/proc`.

5. Make sure that the container is running and that the log shows no errors:

   ```bash
   docker ps | grep unifabric-switch-agent
   docker logs --tail 100 unifabric-switch-agent
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

6. Create one `Switch` resource for every switch. The Controller connects to
   `spec.mgmtIP` to read the LLDP snapshots. For the example network, create
   five resources: `leaf1` through `leaf4` and `spine1`. Each resource looks
   like this:

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

   Set `role` to the network that the switch serves. `ScaleOut` means that the
   switch connects the host RDMA NICs for node-to-node traffic. `Storage` means
   that the switch connects the NICs that
   `fabricNode.storageInterfaceSelector` selects. `ScaleUp` means that the
   switch connects the GPU-side interconnect NICs. The default is `ScaleOut`.
   `grpcPort` defaults to `8090`.

   Apply the files from a host that can reach the cluster with `kubectl`:

   ```bash
   kubectl apply -f leaf1.yaml
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

### InfiniBand: NVIDIA Topograph

This scenario needs no switch-side action. NVIDIA Topograph discovers the IB
fabric through `ibnetdiscover` in the node-data-broker and writes the Node
labels.

The figure below shows an example topology: a three-tier network with tier 3
core, tier 2 spine, and tier 1 leaf switches, and four nodes in two tier 1
performance domains:

![InfiniBand three-tier topology example](../images/infiniband-topology-example.png)

After scale-out topology discovery completes, the four nodes carry these labels.
The label values derive from the switch GUIDs:

| Node | Expected labels |
| --- | --- |
| `node1` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006636c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |
| `node2` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006636c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |
| `node3` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006637c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |
| `node4` | `scale-out.unifabric.io/tier-1=S-fc6a1c03006637c0`<br>`scale-out.unifabric.io/tier-2=S-fc6a1c0300afca40`<br>`scale-out.unifabric.io/tier-3=S-fc6a1c0300b03c40` |

Scale-up labels come from the GPU interconnect topology inside the node and do
not appear in the figure.

### Spectrum-X: NVIDIA Topograph with NetQ

This scenario needs no switch-side action. NVIDIA Topograph reads the fabric
topology through the NetQ API and writes the Node labels. The NetQ credentials
Secret and the provider options are configured in
[Deploy Unifabric](../getting-started.md#spectrum-x-fabric). This section has no
extra steps.

## Verification

Read the Node labels:

```bash
kubectl get nodes -L scale-out.unifabric.io/tier-1,scale-out.unifabric.io/tier-2,scale-out.unifabric.io/tier-3,kubernetes.io/hostname
```

```text
<!-- TODO: capture output from a real environment -->
```

Make sure that every Node has consecutive `scale-out.unifabric.io/tier-N`
labels, where tier 1 is closest to the Node. In SONiC RoCE Case 1 without upper
`Switch` resources, tier 1 only is the expected result. When you configure
Kueue, Volcano, or KAI Scheduler, use only the labels that really exist on the
Nodes.

For SONiC RoCE, read one `FabricNode`:

```bash
kubectl get fabricnode <node-name> -o yaml
```

```text
<!-- TODO: capture output from a real environment -->
```

Make sure that the output meets all of these conditions:

- `status.scaleOutNics` contains the expected scale-out RDMA NICs.
- `status.scaleOutNics[*].lldpNeighbor.hostname` exists.
- The `Ready` and `LLDPNeighborsReady` conditions in `status.conditions` are
  `True`.

For SONiC RoCE Case 2, read the `Switch` resources:

```bash
kubectl get switch
```

```text
NAME     MGMTIP            ROLE       HEALTHY   NEIGHBORS
leaf1    192.168.122.72    ScaleOut   true      2
leaf2    192.168.122.80    ScaleOut   true      2
spine1   192.168.122.163   ScaleOut   true      2
```

Make sure that `HEALTHY` is `true` and that `NEIGHBORS` is greater than `0` for
every switch.

Read the aggregated topology. The Controller creates the `Topology` resource
only after discovery produces a valid result. `status.domains` describes the
parent-child relations of the performance domains, and
`status.nodes[].switchDomainPath` records the full path of each node from the
highest tier down to tier 1:

```bash
kubectl get topology scaleout -o yaml
```

```text
<!-- TODO: capture output from a real environment -->
```

For the example network in SONiC RoCE Case 2, the status contains two tiers, and
`switchMember` lists the real `Switch` names. For the node-only mode of Case 1,
only tier 1 domains appear, and `switchMember` is empty because the synthetic
leaves have no `Switch` resources.

## Troubleshooting

First rule out the common `FabricNode` and RDMA metrics problems with the
[troubleshooting section of Deploy Unifabric](../getting-started.md#troubleshooting).

### `LLDPNeighborsReady` Is `False` (SONiC RoCE)

1. Make sure that LLDP is enabled on the switches.
2. Make sure that the node-side `lldpd` works and that the Agent Pod can read
   the LLDP data.
3. Make sure that `fabricNode.initialScanDelay` is long enough. A first scan
   that runs before LLDP learning completes reports missing neighbors.

### Nodes Have No Topology Labels (SONiC RoCE)

1. Make sure that `topoDiscovery.scaleOut.mode=unifabric-roce`.
2. Make sure that `FabricNode.status.nodeRole` is not `Storage`.
3. Make sure that at least one `scaleOutNics` entry has both `state=up` and
   `lldpNeighbor.hostname`.
4. In Case 2, make sure that every LLDP hostname matches a `Switch` name and
   that the switch status contains LLDP neighbors.
5. In Case 1 with upper tiers, make sure that every `Switch` with the same role
   carries the annotation key and that its neighbor names resolve to leaves
   that the nodes discovered or to other annotated `Switch` resources.
6. Read the Controller log:

   ```bash
   kubectl -n unifabric-system logs deployment/unifabric-controller
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

7. In Case 2, read the switch-agent log on the switch:

   ```bash
   docker logs --tail 100 unifabric-switch-agent
   ```

   ```text
   <!-- TODO: capture output from a real environment -->
   ```

### Nodes Have No Topology Labels (InfiniBand)

1. Make sure that NVIDIA topograph, node-observer, and node-data-broker are
   running and have permission to update Nodes.
2. Make sure that the node-data-broker Pods run on the target GPU nodes.
3. If the scale-out labels are missing, make sure that `ibnetdiscover` works in
   the node-data-broker Pod.
4. If you customized `topoDiscovery.*.nodeLabel.keyTemplate`, update the label
   keys in the scheduler configuration as well.

### Nodes Have No Topology Labels (Spectrum-X)

1. Make sure that NVIDIA topograph and node-observer are running and have
   permission to update Nodes.
2. Make sure that `nvidiaTopograph.provider.name=netq` and that
   `nvidiaTopograph.provider.params.apiUrl=<netq-url>` is correct.
3. Make sure that `nvidiaTopograph.credentialsSecretName` points to a Secret in
   the installation namespace and that the Secret contains the
   `credentials.yaml` key.
4. Make sure that the NetQ account can access the target premises and that NetQ
   holds the fabric topology data.
5. If you customized `topoDiscovery.*.nodeLabel.keyTemplate`, update the label
   keys in the scheduler configuration as well.

### Topograph Cannot Reach NetQ (Spectrum-X)

1. Make sure that the NetQ API URL, the username and password in the Secret,
   the certificates, and the network connectivity are correct.
2. Make sure that the Topograph Pod mounts the credentials at
   `/etc/topograph/credentials/credentials.yaml`.
3. Read the topograph and node-observer logs.
4. Make sure that the topograph Service is reachable.

## References

- Use the labels in a scheduler with the
  [Kueue TAS workload example](./workload-tas.md).
- Render the topology with the
  [topology visualization guide](./topology-visualization.md).
- Look up fields in the [FabricNode reference](../reference/fabricnode.md) and
  the [Topology reference](../reference/topology.md).
- Tune the installation with the [Helm values reference](../../chart/README.md).
