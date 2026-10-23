# SONiC Switch Onboarding

中文版: [sonic-switch-onboarding.zh.md](./sonic-switch-onboarding.zh.md)

## Introduction

In the generic SONiC RoCE scenario, the Unifabric Agent discovers the links from
the nodes to the leaf switches through LLDP on the nodes. The spine tier and the
core tier above the leaf need information from the switch side. This document
describes the two forms that bring SONiC switches into topology discovery:

| Form | What you do | Tiers discovered |
| --- | --- | --- |
| [Form 1: the switches run switch-agent](#form-1-the-switches-run-switch-agent) | Run the Unifabric switch-agent on each switch. It collects the LLDP snapshot on the switch and serves it to the Controller over mTLS gRPC on port `8090`. Create one `Switch` resource per switch in Kubernetes. | Node, leaf, spine, core. All tiers are discovered automatically. |
| [Form 2: the switches cannot run switch-agent](#form-2-the-switches-cannot-run-switch-agent) | Deploy nothing on the switches. Create a `Switch` resource for each switch at the spine tier and above, and declare its neighbors. | Automatic discovery stops at the leaf. The spine tier and above come from the declaration. |

If you can run a container or a binary on the switches, select Form 1. If a
security policy forbids it, or another team manages the switches, select Form 2.
For the result and the limitations of each form, see
[Topology Discovery](../topology-discovery.md#scenario-1-generic-sonic-roce).

## Prerequisites

- Unifabric is deployed according to the [basic installation](../../getting-started.md),
  and `topoDiscovery.scaleOut.mode` is `unifabric-roce`.
- LLDP is enabled on every switch. Run the command on the switch. A leaf must see
  the nodes and the spine. A spine must see all leaf switches:

  ```bash
  lldpcli show neighbors -f json0
  ```

  ```text
  <!-- TODO: collect the output from a real environment -->
  ```

- Form 1 also needs: the Kubernetes Node that runs `unifabric-controller` can reach
  TCP port `8090` on the management IP of every switch, and a host with a
  `kubectl` context is available to export the mTLS certificates.

## Form 1: The Switches Run switch-agent

Complete Step 1 to Step 3 for every switch in the topology (leaf, spine, and core).

### Step 1: Export the mTLS Certificates

All deployment variants use the same three files from the Helm-generated Secret
`switch-controller-mtls-agent`: `tls.crt`, `tls.key`, and `peer.crt`. On the host
with `kubectl` access, export them to a local directory:

```bash
mkdir -p ./tmp-switch-mtls

kubectl -n unifabric-system get secret switch-controller-mtls-agent \
  -o jsonpath='{.data.tls\.crt}' | base64 -d > ./tmp-switch-mtls/tls.crt
kubectl -n unifabric-system get secret switch-controller-mtls-agent \
  -o jsonpath='{.data.tls\.key}' | base64 -d > ./tmp-switch-mtls/tls.key
kubectl -n unifabric-system get secret switch-controller-mtls-agent \
  -o jsonpath='{.data.peer\.crt}' | base64 -d > ./tmp-switch-mtls/peer.crt

chmod 0644 ./tmp-switch-mtls/tls.crt ./tmp-switch-mtls/peer.crt
chmod 0600 ./tmp-switch-mtls/tls.key
```

Variant A reads this directory through `CERT_SOURCE_DIR` and installs the files on
each switch. For the main path, Variant B, and Variant C, you copy the files to
each switch yourself. Each procedure names the target directory.

### Step 2: Run switch-agent on the Switch

The default `socket` mode is the main path. It mounts only the `lldpd` socket and
the mTLS directory. It does not need host network, host UTS, or privileged
permissions. First copy the three files from Step 1 to
`/opt/unifabric-switch-agent/mtls/` on the switch:

```bash
sudo mkdir -p /opt/unifabric-switch-agent/mtls
sudo chmod 700 /opt/unifabric-switch-agent/mtls
```

Then start the container on the switch. Replace `<release-tag>` with the same
release tag as the Controller. `SWITCH_NAME` must match the LLDP hostname of the
switch:

```bash
export SWITCH_AGENT_IMAGE="ghcr.io/unifabric-io/unifabric-switch-agent:<release-tag>"
export SWITCH_NAME="leaf1"

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

After the start, make sure that the container runs and that the log has no errors:

```bash
docker ps | grep unifabric-switch-agent
docker logs --tail 100 unifabric-switch-agent
```

```text
<!-- TODO: collect the output from a real environment -->
```

The three variants below apply when the main path is not usable:

| Variant | Use it when |
| --- | --- |
| Variant A: deployment script | The switches run Docker, and you deploy several switches from one management host. |
| Variant B: manual container in `hostProc` mode | The switch runs Docker, but the switch OS does not expose `/run/lldpd.socket`, or the socket permissions block container access. |
| Variant C: systemd service | The switch cannot run Docker or another container runtime. |

#### Variant A: Deploy Several Switches with the Script

The project provides
[deploy-switch-agent.sh](../../../tools/deploy-switch-agent/deploy-switch-agent.sh).
It copies the exported mTLS certificates from the management host and deploys the
switch-agent container on several switches. The script does not store target
addresses or passwords, and it does not pull images from a registry. When it
starts, it does the following:

1. Makes sure that `peer.crt`, `tls.crt`, and `tls.key` exist on the management host.
2. Copies the certificates to each target over SSH and SCP.
3. Installs the certificates to `/opt/unifabric-switch-agent/mtls` with sudo. The
   owner of the directory and the files is `root:root`. The certificates get mode
   `0644` and the private key gets mode `0600`.
4. Makes sure that the given image exists in Docker on the target switch.
5. Removes and recreates an existing `unifabric-switch-agent` container. If the
   container does not exist, the script creates it.
6. Continues with the other switches and reports all failed targets at the end.

The management host can be a Kubernetes node or a separate Linux host. It must
reach the management network of the target switches, and it must have Bash, `ssh`,
and `scp`. It needs `sshpass` only for password authentication. Each switch needs:

- SONiC OS 2025+ or NVIDIA Cumulus Linux 4.x.x+.
- Docker that the remote user can operate with sudo.
- The switch-agent image preloaded in local Docker.
- Non-interactive SSH commands that reach a Linux shell.

Run all commands in this variant on the management host, unless a step says
otherwise.

1. Select the LLDP collection mode. The script passes the mode to the container on
   each target switch through environment variables.

   The default is `socket` mode. `LLDP_SOCKET_PATH` is the path of the `lldpd`
   Unix socket on each target switch, not on the management host. Before the
   deployment, log in to each switch and make sure that the host exposes the
   socket:

   ```bash
   ls -l /run/lldpd.socket
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

   The first character of the permission string must be `s`, which marks a Unix
   socket. Set the mode and the socket path explicitly:

   ```bash
   export LLDP_COLLECTION_MODE=socket
   export LLDP_SOCKET_PATH=/run/lldpd.socket
   ```

   The script first makes sure that the path on each switch is a socket. Then it
   mounts the socket into the container at the same path and publishes gRPC with
   `-p 8090:8090`. If a switch cannot expose or mount the `lldpd` socket, select
   `hostProc`:

   ```bash
   export LLDP_COLLECTION_MODE=hostProc
   ```

   In `hostProc` mode, the script creates the container with host network, host
   UTS, `--privileged`, and the host `/proc`. `GRPC_PORT` changes the listen port
   of switch-agent. In socket mode it also updates the Docker port mapping.

2. Give the targets and run the script. Pass a comma-separated list of switch
   management IPs in `HOSTS`. The commands below use the RFC 5737 documentation
   address range. Replace them with the real management IPs. Select one SSH
   authentication method. The selection affects only the SSH login and the remote
   sudo authentication.

   Method A, SSH key, is the default:

   ```bash
   SSH_USER=your-ssh-user \
   HOSTS="192.0.2.11,192.0.2.12,192.0.2.21" \
   CERT_SOURCE_DIR=./tmp-switch-mtls \
   SWITCH_AGENT_IMAGE="ghcr.io/unifabric-io/unifabric-switch-agent:<release-tag>" \
   ./tools/deploy-switch-agent/deploy-switch-agent.sh
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

   > **Note:** The SSH key authenticates only the login. The script still runs
   > remote `sudo` to install the certificates and to operate Docker. If the remote
   > user has passwordless sudo, no more options are needed. If sudo needs a
   > password, add `SUDO_AUTH_MODE=password`. The script asks for the password
   > once, and the terminal hides the input.

   Method B, SSH password. Set `SSH_AUTH_MODE=password`. If `SSH_PASSWORD` is not
   set in advance, the script asks for it in the terminal. It does not store the
   password in the script or in the target list:

   ```bash
   SSH_AUTH_MODE=password \
   SSH_USER=your-ssh-user \
   HOSTS="192.0.2.11,192.0.2.12,192.0.2.21" \
   CERT_SOURCE_DIR=./tmp-switch-mtls \
   SWITCH_AGENT_IMAGE="ghcr.io/unifabric-io/unifabric-switch-agent:<release-tag>" \
   ./tools/deploy-switch-agent/deploy-switch-agent.sh
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

   > **Note:** Password mode assumes that the SSH password is also the sudo
   > password. If they differ, set `SUDO_PASSWORD` separately before the run.

   The script accepts these environment variables:

   | Environment variable | Default | Description |
   | --- | --- | --- |
   | `SSH_USER` | none | Required. The SSH user on the switches. |
   | `SSH_PORT` | `22` | The SSH port. |
   | `SSH_AUTH_MODE` | `key` | `key` or `password`. |
   | `SUDO_AUTH_MODE` | derived from the SSH mode | `passwordless` by default with an SSH key, `password` by default with an SSH password. |
   | `HOSTS` | none | Required. A comma-separated list of switch management IPs. |
   | `CERT_SOURCE_DIR` | `./tmp-switch-mtls` | The local certificate directory. |
   | `SWITCH_AGENT_IMAGE` | none | Required. The full image name that already exists on each target. |
   | `LLDP_COLLECTION_MODE` | `socket` | `socket` or `hostProc`. |
   | `LLDP_SOCKET_PATH` | `/run/lldpd.socket` | The path of the `lldpd` Unix socket on each target switch. In socket mode it is mounted into the container at the same path. |
   | `GRPC_PORT` | `8090` | The listen port of switch-agent. In socket mode the script also maps this port. |
   | `REMOTE_UPLOAD_DIR` | `/tmp/unifabric-switch-agent-<user>` | The temporary upload directory for the certificates. |
   | `REMOTE_CERT_DIR` | `/opt/unifabric-switch-agent/mtls` | The final certificate directory. |
   | `CONTAINER_NAME` | `unifabric-switch-agent` | The Docker container name. |

3. Log in to each target switch. Make sure that the container runs and that the
   log has no errors:

   ```bash
   docker ps --filter name=unifabric-switch-agent
   docker logs --tail 100 unifabric-switch-agent
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

#### Variant B: Run the Container Manually in hostProc Mode

`hostProc` mode mounts the host `/proc` and runs the host `lldpcli` with `nsenter`
in the mount and network namespaces of the host. `nsenter` is a Linux tool that
runs a command inside the namespaces of another process. This mode uses the
`lldpcli` and `lldpd` versions that the switch OS provides, not the CLI binary in
the switch-agent image. This mode does not read
`UNIFABRIC_SWITCH_AGENT_LLDP_CLI_VERSION`.

Run all commands in this variant on the target switch.

1. Install the exported mTLS files to `/opt/unifabric-switch-agent/mtls`:

   ```bash
   sudo install -d -m 0755 /opt/unifabric-switch-agent/mtls
   sudo install -m 0644 tls.crt peer.crt /opt/unifabric-switch-agent/mtls/
   sudo install -m 0600 tls.key /opt/unifabric-switch-agent/mtls/
   ```

2. Start the container:

   ```bash
   export SWITCH_AGENT_IMAGE="ghcr.io/unifabric-io/unifabric-switch-agent:<release-tag>"

   docker pull "${SWITCH_AGENT_IMAGE}"

   docker rm -f unifabric-switch-agent 2>/dev/null || true

   docker run -d \
     --name unifabric-switch-agent \
     --restart unless-stopped \
     --network host \
     --uts host \
     --privileged \
     -e UNIFABRIC_SWITCH_AGENT_LLDP_COLLECTION_MODE=hostProc \
     -v /proc:/host/proc:ro \
     -v /opt/unifabric-switch-agent/mtls:/etc/unifabric/switch-mtls:ro \
     "${SWITCH_AGENT_IMAGE}" \
     /usr/bin/unifabric/switch-agent
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

   > **Warning:** The container runs with `--privileged`, host network, host UTS,
   > and the host `/proc`. If the switch OS exposes `/run/lldpd.socket` reliably,
   > use the default `socket` mode in
   > [Step 2](#step-2-run-switch-agent-on-the-switch) instead. It has a smaller
   > permission surface.

3. Make sure that the container runs and that the log has no errors:

   ```bash
   docker ps | grep unifabric-switch-agent
   docker logs --tail 100 unifabric-switch-agent
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

#### Variant C: Install as a systemd Service

Use this variant when the switch cannot run Docker or another container runtime.
switch-agent runs as a native Linux binary, and systemd manages it. It calls the
local `lldpcli` on the switch directly, so it does not mount `/run/lldpd.socket`.
The switch needs `lldpcli`, and the systemd `PATH` must find it.

Run all commands in this variant on the target switch.

1. Download and install the binary that matches the CPU architecture of the switch
   from the GitHub Release page:

   ```bash
   LATEST_TAG=$(curl -fsSL https://api.github.com/repos/unifabric-io/unifabric/releases/latest | grep '"tag_name":' | cut -d '"' -f4)
   ARCH="$(uname -m)"
   case "${ARCH}" in
     x86_64) RELEASE_ARCH=amd64 ;;
     aarch64|arm64) RELEASE_ARCH=arm64 ;;
     *) echo "unsupported architecture: ${ARCH}" >&2; exit 1 ;;
   esac

   curl -fLO "https://github.com/unifabric-io/unifabric/releases/download/${LATEST_TAG}/unifabric-switch-agent_${LATEST_TAG}_linux_${RELEASE_ARCH}.tar.gz"
   curl -fLO "https://github.com/unifabric-io/unifabric/releases/download/${LATEST_TAG}/unifabric-switch-agent_${LATEST_TAG}_linux_${RELEASE_ARCH}.tar.gz.sha256"
   sha256sum -c "unifabric-switch-agent_${LATEST_TAG}_linux_${RELEASE_ARCH}.tar.gz.sha256"

   tar -xzf "unifabric-switch-agent_${LATEST_TAG}_linux_${RELEASE_ARCH}.tar.gz"
   sudo install -m 0755 "unifabric-switch-agent_${LATEST_TAG}_linux_${RELEASE_ARCH}/unifabric-switch-agent" /usr/local/bin/unifabric-switch-agent
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

2. Install the exported mTLS files to `/etc/unifabric/switch-mtls`:

   ```bash
   sudo install -d -m 0755 /etc/unifabric/switch-mtls
   sudo install -m 0644 tls.crt peer.crt /etc/unifabric/switch-mtls/
   sudo install -m 0600 tls.key /etc/unifabric/switch-mtls/
   ```

3. Create `/etc/systemd/system/unifabric-switch-agent.service`. systemd renders
   `%H` as the hostname of the switch:

   ```ini
   [Unit]
   Description=Unifabric switch-agent
   After=network-online.target lldpd.service
   Wants=network-online.target

   [Service]
   Type=simple
   Environment=UNIFABRIC_SWITCH_AGENT_SWITCH_NAME=%H
   Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   ExecStart=/usr/local/bin/unifabric-switch-agent
   Restart=always
   RestartSec=5s

   [Install]
   WantedBy=multi-user.target
   ```

4. Start the service:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now unifabric-switch-agent
   ```

5. Make sure that the service runs and that the log has no errors:

   ```bash
   systemctl status unifabric-switch-agent
   journalctl -u unifabric-switch-agent -n 100 --no-pager
   ```

   ```text
   <!-- TODO: collect the output from a real environment -->
   ```

### Step 3: Create the Switch Resources

After the agent runs, create one Kubernetes `Switch` resource for each physical
switch on the host with `kubectl` access. The Controller connects to
`spec.mgmtIP` and `spec.grpcPort` to read the LLDP snapshot. The `192.0.2.0/24`
addresses below are examples. Replace them with the real management IP of each
switch:

```bash
kubectl apply -f - <<'EOF'
apiVersion: unifabric.io/v1beta1
kind: Switch
metadata:
  name: leaf1
spec:
  mgmtIP: 192.0.2.11
  role: ScaleOut
  grpcPort: 8090
---
apiVersion: unifabric.io/v1beta1
kind: Switch
metadata:
  name: spine1
spec:
  mgmtIP: 192.0.2.21
  role: ScaleOut
  grpcPort: 8090
EOF
```

Repeat for the other switches. For the switches of the storage network, set `role`
to `Storage`.

> **Note:** `metadata.name` does not have to match the switch hostname. The
> Controller uses `Switch.status.hostname` to match the switch-side data with the
> hostname that the node LLDP discovered. Do not add the
> `unifabric.io/switch-neighbors` annotation in this form.

## Form 2: The Switches Cannot Run switch-agent

LLDP on the nodes already discovered the leaf switches. Do not create `Switch`
resources for the leaf switches. Create one `Switch` resource only for each switch
at the spine tier and above. Declare the switches one tier below it with the
`unifabric.io/switch-neighbors` annotation. The example declares that `spine1`
connects to `leaf1` to `leaf4`:

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

The annotation value is a quoted JSON array of strings. Each name must be the LLDP
hostname of a leaf that the nodes discovered, or the name of another `Switch`
resource. Do not use a Kubernetes Node name. If the network has a core tier,
create one more `Switch` resource for each core switch. Its neighbors are the
names of the spine switches that connect to it.

> **Warning:** The declared neighbors do not update when the physical links
> change. After you add a switch or move a cable, update the annotation. Otherwise
> the Node labels do not match the real topology. Do not run switch-agent and set
> this annotation on the same switch.

## Verification

Form 1: make sure that the Controller connected to the switch-agent on every
switch:

```bash
kubectl get switches -o wide
```

```text
NAME     MGMTIP       ROLE       HEALTHY   NEIGHBORS
leaf1    192.0.2.11   ScaleOut   true      3
leaf2    192.0.2.12   ScaleOut   true      3
leaf3    192.0.2.13   ScaleOut   true      3
leaf4    192.0.2.14   ScaleOut   true      3
spine1   192.0.2.21   ScaleOut   true      4
```

The switch side is complete when every `Switch` shows `HEALTHY` as `true` and
`NEIGHBORS` matches the real neighbor count of the switch.

The final result of both forms is the Node labels and the `Topology` resource. To
view them, see
[Topology Discovery](../topology-discovery.md#view-topology-and-node-labels).

## Troubleshooting

### The Log Shows lldpcli command not found

First make sure that the prerequisites are met. Log in to the switch and run:

```bash
lldpcli -f json0 show neighbors
```

If the command fails, install `lldpd` and `lldpcli` on the switch. If the command
works in an interactive shell but the systemd service still fails, make sure that
the `PATH` line in the service file includes the directory of `lldpcli`.

### Switch.status.healthy Stays false

1. Make sure that the Controller can reach port `8090`, or the configured
   `grpcPort`, on the management IP of the switch.
2. Make sure that the mTLS files on the switch match the Helm-generated Secret
   `switch-controller-mtls-agent`. After a Helm reinstall, export the files again.

### The Script Reports a Missing Local Image

The script never runs `docker pull`. Import the correct image version into Docker
on that switch, then run the script again.

### The Script Fails at Remote sudo

Make sure that the remote user has root permission for Docker and for the
certificate directory. If the sudo password differs from the SSH password, set
`SUDO_PASSWORD` before the run.

### Form 1 Shows No Spine Tier

- Make sure that every physical switch in the topology has a `Switch` resource and
  a running switch-agent.
- Make sure that no `unifabric.io/switch-neighbors` annotation remains.
- Make sure that `Switch.status.hostname` or the resource name matches the switch
  hostname that the node LLDP discovered.
- Read the Controller log and the switch log:

  ```bash
  kubectl -n unifabric-system logs deployment/unifabric-controller
  docker logs --tail 100 unifabric-switch-agent
  ```

### Form 2 Shows No Spine Tier

- Make sure that every `Switch` with the same role has the
  `unifabric.io/switch-neighbors` annotation.
- Make sure that no `Switch` resource has the same name as a leaf that the nodes
  discovered.
- Make sure that the annotation is a quoted JSON array of strings, and that the
  array contains only switch names, not Kubernetes Node names.
- Make sure that each neighbor name resolves to a leaf that the node LLDP
  discovered, or to another `Switch` resource with the same role.
- If no `Switch` resource exists, or the physical network has only a leaf tier,
  Node labels with only tier 1 are the expected result.

## References

- [Topology Discovery](../topology-discovery.md): the scenarios and how to view the
  result.
- [Switch reference](../../reference/switch.md): the `spec` and `status` fields of
  the `Switch` resource.
- [Scale-out topology design](../../design/scaleout-topology.md): how the
  Controller uses the switch-side LLDP snapshot and the neighbor declaration.
- [deploy-switch-agent.sh](../../../tools/deploy-switch-agent/deploy-switch-agent.sh):
  the batch deployment script.
