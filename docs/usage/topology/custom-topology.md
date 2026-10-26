# Custom Topology

中文版: [custom-topology.zh.md](./custom-topology.zh.md)

## Introduction

Custom topology is the discovery method for switches that cannot run
switch-agent. The reason can be a security policy, a switch that another team
manages, or a switch OS that has no Docker and no Linux shell. The switch only
has to send LLDP.

In this method, the Unifabric Agent on each node still reads the LLDP neighbor of
each RDMA NIC. From this, the Controller discovers the leaf switches and the links
from the nodes to them. For the tiers above the leaf, you create one `Switch`
resource for each spine and core switch. In its `unifabric.io/switch-neighbors`
annotation, you declare the switches one tier below it. The Controller combines
the discovered leaf tier with the declared upper tiers. It writes the same Node
labels and `Topology` resource as the other methods.

| Item | Description |
| --- | --- |
| Applicable networks | Scale-out network, Storage network |
| Topology source | The LLDP neighbors on the node side, plus the neighbor declarations on the `Switch` resources |
| Discovered tiers | The leaf tier is discovered automatically. The spine and core tiers come from the declarations. |
| Switch side requirements | LLDP is enabled on the ports that face the nodes. Any switch OS is possible. No software runs on the switch. |
| Cluster side requirements | None. |
| Helm configuration | The same as [LLDP discovery](./lldp-discovery.md): `topoDiscovery.scaleOut.mode=unifabric-roce` or `topoDiscovery.storage.mode=unifabric-roce`. |

Limits:

- The declared links do not update when the physical links change. After you add
  a switch or move a cable, update the declarations.
- The leaf tier can come only from the LLDP on the node side. You cannot declare
  it. If the switch ports that face the nodes do not send LLDP, the nodes cannot
  find their leaf. Then you cannot use this method.

For the discovery method to use for each network and each network device, see
[Network topology discovery](./README.md#choose-a-discovery-method-by-network).

## Prerequisites

- Unifabric is deployed according to the [basic installation](../../getting-started.md),
  and `topoDiscovery.scaleOut.mode` or `topoDiscovery.storage.mode` is
  `unifabric-roce`.
- LLDP is enabled on the switch ports that face the nodes. The nodes must see
  their leaf switches. Use the command below to make sure of this. The hostname
  column must not be empty:

  ```bash
  kubectl get fabricnode node1 -o jsonpath='{range .status.scaleOutNics[*]}{.name}{"\t"}{.lldpNeighbor.hostname}{"\t"}{.lldpNeighbor.port}{"\n"}{end}'
  ```

  ```text
  eth1	leaf1	Ethernet0
  eth2	leaf2	Ethernet0
  ```

- The operator must know the physical links between the leaf, spine, and core
  switches.
- No `Switch` of the same role runs switch-agent. The annotation key selects
  Custom topology for the whole role. If one `Switch` of the role has the key
  and another does not, the Controller ignores all annotations of that role.

## Configuration and Usage

The example uses the topology from
[Network topology discovery](./README.md#view-topology-and-node-labels). It has
four nodes, four leaf switches, and one spine switch `spine1`. `spine1` connects
to `leaf1` through `leaf4`.

1. Do not create `Switch` resources for the leaf switches. The node LLDP already
   discovers them. The Controller uses the LLDP hostname of each leaf as its
   name.

2. Create one `Switch` resource for each switch at the spine tier and above.
   In the annotation, declare the switches one tier below it. Set `role` to
   `ScaleOut` for the Scale-out network or to `Storage` for the Storage network:

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

   The annotation value is a quoted JSON array of strings. Each name must be the
   LLDP hostname of a leaf that the nodes discovered. It can also be the name of
   another `Switch` resource with the same role. Do not use a Kubernetes Node
   name. A link has no direction, so declare each link only once.

3. If the network has a core tier, create one `Switch` resource for each core
   switch. Its neighbors are the spine switches that connect to it:

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

> Warning: The declared links do not update when the physical links change.
> After you add a switch or move a cable, update the annotation. If you do not,
> the Node labels do not match the real topology.

> Do not run switch-agent and set this annotation on `Switch` resources of the
> same role.

## Verification

Make sure that the `Switch` resources exist. `HEALTHY` and `NEIGHBORS` are empty
because no switch-agent reports to the Controller:

```bash
kubectl get switches -o wide
```

```text
NAME     MGMTIP   ROLE       HEALTHY   NEIGHBORS
spine1            ScaleOut
```

The final result is the Node labels and the `Topology` resource. To view them,
see [Network topology discovery](./README.md#view-topology-and-node-labels). In
the `Topology` resource, the members of the leaf domains are the leaf hostnames
from LLDP. The member of the spine domain is `spine1`.

## Troubleshooting

### Node Labels Have Only Tier 1

- Make sure that a `Switch` resource exists for each switch at the spine tier
  and above.
- Make sure that each `Switch` of the same role has the
  `unifabric.io/switch-neighbors` annotation. One `Switch` without the key
  changes the whole role back to switch-agent discovery.
- Make sure that the annotation is a quoted JSON array of strings. The array
  must contain only switch names, not Kubernetes Node names.
- Make sure that each neighbor name resolves to a leaf hostname that the node
  LLDP discovered. It can also resolve to another `Switch` resource of the same
  role. A name that does not resolve stops the reconciliation, and the old
  labels stay. Read the Controller log:

  ```bash
  kubectl -n unifabric-system logs deployment/unifabric-controller --tail 200
  ```

- Make sure that no `Switch` resource has the same name as a leaf that the nodes
  discovered.
- If the physical network has only a leaf tier, tier 1 alone is the expected
  result.

## References

- [Network topology discovery](./README.md): the discovery methods and how to
  view the result.
- [LLDP discovery](./lldp-discovery.md): automatic discovery with switch-agent.
- [Switch reference](../../reference/switch.md): the `spec`, `status`, and
  annotation of the `Switch` resource.
- [Scale-out topology design](../../design/scaleout-topology.md): how the
  Controller combines the node LLDP with the neighbor declarations.
