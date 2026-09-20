# Unikernel Migration Plan

> **Status**: Detailed work plan. Not an active work item. Agents
> should ignore this document unless specifically asked to consider it.
>
> **Prerequisite**: MCP v2 phase 6 complete -- master running, agents
> on all nodes, edge routing, snapshots, and migration all operational.

## Starting Point

The MCP agent already has a `runtime.Runtime` interface
(`mcp/internal/runtime/runtime.go`) with methods for Pull, Run, Stop,
Remove, Inspect, List, Build, Push, ImageExists, and Logs. The only
implementation is `Podman` (`mcp/internal/runtime/podman.go`). The
agent struct holds `Runtime runtime.Runtime` and all lifecycle
operations (deploy, stop, start, undeploy, status) go through this
interface.

This means the runtime abstraction layer is already in place. The
migration is primarily: implement a QEMU/Nanos backend for the
existing interface, add isolated networking, and extend service
definitions with runtime-specific fields.

## Terminology

| Term | Meaning |
|------|---------|
| **VM** | A QEMU/KVM virtual machine running a Nanos unikernel |
| **bridge** | A Linux bridge device (`mcp-br0`) on the host for VM networking |
| **TAP** | A TAP device attached to the bridge, one per VM |
| **9p mount** | QEMU's `-virtfs` passthrough for host directory access |
| **ops** | The Nanos toolchain CLI for building unikernel images |

---

## Phase 1: QEMU Runtime Implementation

**Goal**: A second `runtime.Runtime` implementation that can start and
stop Nanos unikernel VMs with basic networking. No isolation
enforcement yet -- VMs get host-forwarded ports like containers do.

### 1.1 NixOS Host Prerequisites

Add QEMU/KVM packages to the NixOS configuration on rift and orion.
svc (Debian) gets equivalent packages via apt.

Required on all nodes that will run unikernels:

- `qemu` (specifically `qemu-system-x86_64`)
- `ops` CLI (Nanos toolchain) -- install from GitHub release or build
  from source
- KVM access: the `mcp` user needs `/dev/kvm` access. On NixOS, add
  the user to the `kvm` group. On Debian, same.
- `bridge-utils` or `iproute2` for bridge management (Phase 2)

Verify KVM works: `qemu-system-x86_64 -enable-kvm -nographic
-no-reboot` should boot and exit.

**Deliverable**: All amd64 nodes can run QEMU with KVM acceleration.
RPi nodes (arm64, no KVM) are excluded from unikernel support.

### 1.2 Image Building Pipeline

The agent needs to produce a Nanos `.img` file from a Go binary. Two
paths, implemented in order:

**1.2a -- Local build from OCI image (initial approach)**

The agent already pulls OCI images via `Runtime.Pull()`. For
unikernels:

1. Pull the OCI image from MCR (reuse existing podman pull or use
   `skopeo copy` to a local directory).
2. Extract the ELF binary from the image. Convention: the binary is at
   `/usr/local/bin/<service>` in the image (same path the Dockerfiles
   use).
3. Run `ops build <binary> -c <config.json>` to produce a `.img` file.
4. Store the image at `/srv/mcp/images/<service>-<component>.img`.

The `ops` config JSON specifies:

```json
{
  "Args": ["server", "--config", "/srv/mcq/mcq.toml"],
  "Dirs": ["srv"],
  "Mounts": {
    "/srv/<service>": "/srv/<service>"
  },
  "ManifestPassthrough": {
    "mem": "256m",
    "smp": 1
  }
}
```

**1.2b -- Pre-built unikernel images in MCR (later)**

Store `.img` files as OCI artifacts in MCR with a distinct media type
(`application/vnd.metacircular.unikernel.nanos.v1`). The agent pulls
the artifact and writes it directly to
`/srv/mcp/images/<service>-<component>.img`. This skips the
extract-and-build step and ensures the deployed image is identical to
what was built.

MCR already stores OCI artifacts; this requires adding the media type
to MCR's accepted list and adding an `mcp build --unikernel` command
that builds the image locally and pushes it.

**Deliverable**: Agent can produce a bootable Nanos image from an
existing OCI container image.

### 1.3 QEMU Runtime Type

Implement `QEMURuntime` satisfying `runtime.Runtime`:

```go
type QEMURuntime struct {
    imageDir   string            // /srv/mcp/images/
    stateDir   string            // /srv/mcp/vm-state/
    opsPath    string            // path to ops binary
    qemuPath   string            // path to qemu-system-x86_64
    logger     *slog.Logger
    mu         sync.Mutex
    vms        map[string]*vmState  // name → running VM state
}

type vmState struct {
    pid       int
    qmpSocket string             // QMP control socket
    serial    string             // serial console log path
    ip        string             // VM IP on bridge (Phase 2)
    ports     map[int]int        // guest port → host port
}
```

**Method mapping:**

| Runtime Method | QEMU Implementation |
|---|---|
| `Pull(image)` | Pull OCI image, extract ELF, run `ops build`, store `.img` |
| `Run(spec)` | Start `qemu-system-x86_64` with KVM, virtio-net, 9p mounts, QMP socket |
| `Stop(name)` | Send `system_powerdown` via QMP, wait 10s, then SIGKILL |
| `Remove(name)` | Kill process if running, remove state files |
| `Inspect(name)` | Check process liveness + read QMP status |
| `List()` | Enumerate `/srv/mcp/vm-state/*/qemu.pid`, check liveness |
| `Build(...)` | Not applicable for unikernels (image built during Pull) |
| `Push(...)` | Not applicable (future: push `.img` to MCR as OCI artifact) |
| `ImageExists(image)` | Check if `.img` file exists in imageDir |
| `Logs(name)` | Read serial console log file |

**QEMU invocation** (Phase 1 -- user-mode networking with port
forwards, no bridge yet):

```
qemu-system-x86_64 \
  -enable-kvm \
  -m 256 \
  -smp 1 \
  -nographic \
  -serial file:/srv/mcp/vm-state/<name>/console.log \
  -qmp unix:/srv/mcp/vm-state/<name>/qmp.sock,server,nowait \
  -drive file=/srv/mcp/images/<name>.img,format=raw,if=virtio \
  -virtfs local,path=/srv/<service>,mount_tag=srvdata,security_model=mapped-xattr,id=srvdata \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp:127.0.0.1:<host_port>-:<guest_port>
```

This gives user-mode networking with port forwards to localhost --
functionally identical to how rootless podman works. mc-proxy routes
to `127.0.0.1:<host_port>` the same way it does for containers.

**Deliverable**: A `QEMURuntime` that passes the same interface as
`Podman`. Agent can deploy, stop, inspect, and undeploy unikernel
services using QEMU user-mode networking.

### 1.4 Service Definition Changes

Add `runtime` field to service definitions, proto specs, and registry
schema.

**TOML** (`servicedef.go`):

```toml
name    = "mcq"
runtime = "unikernel"
tier    = "worker"
active  = true

[[components]]
name    = "mcq"
image   = "mcr.svc.mcp.metacircular.net:8443/mcq:v0.4.0"
memory  = 256       # MB, required for unikernels
vcpus   = 1         # default 1
volumes = ["/srv/mcq:/srv/mcq"]
cmd     = ["server", "--config", "/srv/mcq/mcq.toml"]
```

**Proto** (`mcp.proto`):

```protobuf
message ServiceSpec {
  string name    = 1;
  bool   active  = 2;
  repeated ComponentSpec components = 3;
  string comment = 4;
  string runtime = 5;   // "container" (default) or "unikernel"
}

message ComponentSpec {
  // ... existing fields ...
  int32 memory_mb = 11;  // required for unikernel runtime
  int32 vcpus     = 12;  // default 1
}
```

**Registry schema** (new migration):

```sql
ALTER TABLE components ADD COLUMN runtime TEXT NOT NULL DEFAULT 'container';
ALTER TABLE components ADD COLUMN memory_mb INTEGER NOT NULL DEFAULT 0;
ALTER TABLE components ADD COLUMN vcpus INTEGER NOT NULL DEFAULT 1;
```

**Agent runtime selection**: In `agent.go`, the agent holds both
runtimes:

```go
type Agent struct {
    // ... existing fields ...
    ContainerRuntime runtime.Runtime  // podman
    UnikernelRuntime runtime.Runtime  // qemu (nil if not configured)
}

func (a *Agent) runtimeFor(comp *registry.Component) runtime.Runtime {
    if comp.Runtime == "unikernel" {
        return a.UnikernelRuntime
    }
    return a.ContainerRuntime
}
```

All lifecycle operations call `a.runtimeFor(comp)` instead of
`a.Runtime` directly.

**Validation rules**:
- `runtime = "unikernel"` requires `memory_mb > 0`.
- `runtime = "unikernel"` requires the node to have KVM
  (`/dev/kvm` exists). Agent rejects deploys on nodes without KVM.
- `runtime = "unikernel"` is incompatible with `exec:` and `cli`
  snapshot methods. Validation rejects these combinations.

**Deliverable**: Service definitions can declare `runtime =
"unikernel"`. The agent selects the correct runtime per component.
Container services are completely unaffected.

### 1.5 Resource Tracking

The agent needs to track allocated VM resources to avoid overcommit.
The v2 heartbeat already reports CPU, memory, and disk. Add tracking
of allocated-to-VMs resources:

```go
type ResourceTracker struct {
    mu           sync.Mutex
    totalMemMB   int64    // from /proc/meminfo
    totalCPUs    int32    // from runtime.NumCPU()
    allocMemMB   int64    // sum of running VM memory_mb
    allocCPUs    int32    // sum of running VM vcpus
}

func (r *ResourceTracker) CanFit(memMB int64, vcpus int32) bool
func (r *ResourceTracker) Allocate(memMB int64, vcpus int32)
func (r *ResourceTracker) Release(memMB int64, vcpus int32)
```

The master's placement algorithm gains a resource check: before
placing a unikernel service on a node, verify the node has enough
unallocated memory and CPUs. Container services continue to use
container-count placement.

**Deliverable**: Agent tracks VM resource allocation. Master rejects
placements that would overcommit a node.

### 1.6 Phase 1 Validation

Deploy a test service (a minimal Go HTTP server, not a real platform
service) as a unikernel:

1. Build a trivial Go binary that serves HTTP on port 8080.
2. Package it as an OCI image, push to MCR.
3. Write a service definition with `runtime = "unikernel"`.
4. `mcp deploy test-unikernel` -- verify it starts, mc-proxy routes
   to it, health checks pass.
5. `mcp undeploy test-unikernel` -- verify clean shutdown.
6. Verify container services are completely unaffected.

**Phase 1 complete when**: A unikernel service can be deployed,
health-checked, and undeployed through the normal `mcp deploy`/
`mcp undeploy` flow, alongside running container services.

---

## Phase 2: Isolated Networking

**Goal**: Replace QEMU user-mode networking with a host-only bridge.
VMs can only communicate through mc-proxy. This is the phase that
delivers the security properties -- without it, unikernels are just
heavier containers.

### 2.1 Bridge Setup

Create a persistent Linux bridge on each unikernel-capable node:

**NixOS** (`networking.bridges` in NixOS config):

```nix
networking.bridges.mcp-br0.interfaces = [];
networking.interfaces.mcp-br0.ipv4.addresses = [{
  address = "10.99.0.1";
  prefixLength = 24;
}];
```

**Debian** (svc -- if svc ever runs unikernels, which is unlikely
given its edge role, but document for completeness):

```
# /etc/network/interfaces.d/mcp-br0
auto mcp-br0
iface mcp-br0 inet static
  address 10.99.0.1/24
  bridge_ports none
  bridge_stp off
```

The bridge uses the `10.99.0.0/24` subnet. This is a host-only
network -- no default route, no NAT to the internet or Tailnet. VMs
can only reach `10.99.0.1` (the agent/mc-proxy host).

**Deliverable**: Each unikernel-capable node has a `mcp-br0` bridge
with address `10.99.0.1/24`.

### 2.2 TAP Device Management

Each VM gets a TAP device attached to the bridge. The agent creates
and destroys TAP devices as part of the VM lifecycle:

```go
func (q *QEMURuntime) createTAP(name string) (string, error) {
    tapName := fmt.Sprintf("tap-%s", name)  // max 15 chars for IFNAMSIZ
    // ip tuntap add dev <tap> mode tap user mcp
    // ip link set <tap> master mcp-br0
    // ip link set <tap> up
    return tapName, nil
}

func (q *QEMURuntime) destroyTAP(name string) error {
    tapName := fmt.Sprintf("tap-%s", name)
    // ip link del <tap>
    return nil
}
```

TAP creation requires `CAP_NET_ADMIN` or `ip tuntap` permissions for
the `mcp` user. On NixOS, grant this via a udev rule or by running
the agent with ambient capabilities:

```nix
systemd.services.mcp-agent.serviceConfig.AmbientCapabilities = [
  "CAP_NET_ADMIN"
];
```

**QEMU invocation changes** (bridge networking replaces user-mode):

```
qemu-system-x86_64 \
  ... \
  -device virtio-net-pci,netdev=net0,mac=52:54:00:xx:xx:xx \
  -netdev tap,id=net0,ifname=tap-<name>,script=no,downscript=no
```

Each VM gets a deterministic MAC address derived from the service
name (e.g., SHA-256 of service name, take 5 bytes, prepend `52:54:00`).

### 2.3 VM IP Assignment

VMs need static IPs on the bridge. No DHCP server -- the agent
assigns IPs and passes them to Nanos via the ops config.

```go
type IPAllocator struct {
    mu       sync.Mutex
    subnet   net.IPNet            // 10.99.0.0/24
    gateway  net.IP               // 10.99.0.1
    assigned map[string]net.IP    // service name → IP
    next     byte                 // next octet to try (2-254)
}
```

The ops config passes networking to Nanos:

```json
{
  "RunConfig": {
    "IPAddress": "10.99.0.5",
    "NetMask": "255.255.255.0",
    "Gateway": "10.99.0.1"
  }
}
```

Assigned IPs are persisted in the agent's registry:

```sql
ALTER TABLE components ADD COLUMN vm_ip TEXT;
```

**Deliverable**: Each VM gets a static IP on the bridge. The agent
tracks assignments in its registry.

### 2.4 mc-proxy Route Update

With bridge networking, mc-proxy routes change from
`127.0.0.1:<host_port>` to `10.99.0.<n>:<guest_port>`:

- L7 routes: mc-proxy terminates TLS, forwards to
  `10.99.0.<n>:<port>` (plaintext on the bridge).
- L4 routes: mc-proxy passes through to `10.99.0.<n>:<port>` (TLS
  end-to-end).

The `ProxyRouter.RegisterRoutes()` method needs to use the VM's
bridge IP instead of `127.0.0.1` for unikernel components. Port
allocation changes: unikernel VMs expose their actual service port
on the bridge (no random host port needed), so `host_port` equals
the route's declared port.

### 2.5 Firewall Rules

The bridge must be locked down so VMs can only reach mc-proxy:

```bash
# Allow established connections back to VMs
iptables -A FORWARD -i mcp-br0 -o mcp-br0 -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow VMs to reach the host (mc-proxy) on the bridge IP
iptables -A INPUT -i mcp-br0 -d 10.99.0.1 -j ACCEPT

# Block VM-to-VM traffic on the bridge
ebtables -A FORWARD -i tap-+ -o tap-+ -j DROP

# Block VMs from reaching anything outside the bridge
iptables -A FORWARD -i mcp-br0 ! -o mcp-br0 -j DROP
```

These rules enforce mandatory mediation: VMs can reach the host
(where mc-proxy listens) but nothing else. No Tailnet, no internet,
no other VMs. All inter-service communication goes through mc-proxy.

On NixOS, these rules go in `networking.firewall` or
`networking.nftables`. On Debian, `/etc/iptables/rules.v4`.

**Deliverable**: VMs are network-isolated. They can only reach
mc-proxy on the host. VM-to-VM and VM-to-Tailnet traffic is blocked.

### 2.6 Phase 2 Validation

1. Deploy the test unikernel from Phase 1 with bridge networking.
2. Verify mc-proxy routes to it via the bridge IP.
3. From inside the VM (via the service's own gRPC or HTTP endpoint),
   attempt to reach a Tailnet IP directly -- must fail.
4. Attempt to reach another VM on the bridge -- must fail.
5. Verify the service can reach its dependencies (MCIAS, Metacrypt)
   only via mc-proxy on the host.
6. Verify container services are completely unaffected by the bridge.

**Phase 2 complete when**: Unikernel VMs are fully network-isolated
and can only communicate through mc-proxy. The agent enforces this
structurally, not cooperatively.

---

## Phase 3: Snapshots and Observability

**Goal**: Ensure unikernel services participate in the snapshot and
monitoring systems. Adapt debugging tools for the no-shell environment.

### 3.1 Snapshot Adaptation

The default snapshot method (tar `*.toml`, `*.db`, `*.pem` from the
host-side `/srv/<service>/`) works unchanged for unikernels because
the agent tars the host directory, not the VM filesystem. The 9p
passthrough means writes from the VM appear on the host immediately.

The `grpc` snapshot method also works unchanged -- the agent calls the
service's `SnapshotService.Snapshot` RPC over mc-proxy, which reaches
the VM the same way any other gRPC call does.

**What doesn't work**: `cli` and `exec:` methods, because there is
no shell inside the VM. Validation (from Phase 1.4) already rejects
these combinations, but the snapshot scheduler should also log a
warning if it encounters a unikernel service with an incompatible
snapshot method.

**Deliverable**: Snapshots work for unikernel services using the
default or `grpc` methods.

### 3.2 Serial Console Log Collection

QEMU writes serial console output to
`/srv/mcp/vm-state/<name>/console.log`. The `Logs()` method on
`QEMURuntime` reads this file. But the agent's `Logs` gRPC RPC
currently streams from podman/journalctl.

Extend the `Logs` RPC to detect the component's runtime and read from
the serial console log instead:

```go
func (a *Agent) Logs(req *pb.LogsRequest, stream pb.McpAgent_LogsServer) error {
    comp := a.registryComponent(req)
    if comp.Runtime == "unikernel" {
        return a.streamSerialLog(comp, req, stream)
    }
    return a.streamContainerLog(comp, req, stream)
}
```

For Nanos, configure the Go binary's logging to write to stdout/stderr
(which Nanos routes to the serial console). This is the default Go
behavior, so no changes needed in the services themselves.

**Deliverable**: `mcp logs <service>` works for unikernel services,
streaming the serial console output.

### 3.3 Health Check Adaptation

The v2 health check types (tcp, grpc, http) all work over the network
and don't require shell access. No changes needed -- the agent's
monitoring loop connects to the VM's port via mc-proxy or the bridge
IP the same way it does for containers.

### 3.4 Drift Detection

The agent's `LiveCheck()` currently calls `Runtime.List()` and
reconciles with the registry. The QEMU `List()` implementation
enumerates running VMs by checking PIDs in
`/srv/mcp/vm-state/*/qemu.pid`. This needs to be reliable:

- On agent restart, rebuild the `vms` map from the state directory.
- QEMU processes started with `--daemonize` survive agent restarts.
- The QMP socket reconnects on agent restart.

**Deliverable**: Drift detection works for unikernel VMs. Agent
restart does not lose track of running VMs.

### 3.5 Phase 3 Validation

1. Deploy a unikernel service with `[snapshot] method = "grpc"`.
2. `mcp snapshot <service>` -- verify snapshot succeeds.
3. Verify scheduled snapshots include the unikernel service.
4. `mcp logs <service>` -- verify serial console output streams.
5. Kill the QEMU process manually. Verify drift detection catches it
   and reports the service as unhealthy.
6. Restart the agent. Verify it rediscovers running VMs.

---

## Phase 4: Image Attestation

**Goal**: The agent verifies that the image it boots matches what the
operator deployed. The master records expected image hashes.

### 4.1 Image Hashing

After building the `.img` file (Phase 1.2), the agent computes its
SHA-256 hash and stores it in the registry:

```sql
ALTER TABLE components ADD COLUMN image_hash TEXT;
```

Before every VM boot, the agent re-hashes the `.img` file and
compares against the stored value. If they don't match, the deploy
fails with an attestation error. This detects:

- Accidental image corruption.
- Tampering with the image file on disk.
- Stale images from a previous deploy.

### 4.2 Master-Side Hash Verification

The agent reports the image hash to the master in the deploy response
and in heartbeats. The master stores expected hashes in its placements
table:

```sql
ALTER TABLE placements ADD COLUMN image_hash TEXT;
```

On reconciliation, the master compares the agent-reported hash against
its stored value. Mismatches are flagged in `mcp status` output.

### 4.3 Build Reproducibility

For attestation to be meaningful, image builds must be reproducible:
the same ELF binary + the same ops config must produce the same `.img`
hash. Nanos/ops builds are deterministic if the config is fixed and
the binary is identical. Document and test this property.

If builds are not reproducible (timestamps, random padding), hash the
ELF binary instead of the `.img` and accept that the image-level hash
is a weaker check.

### 4.4 Phase 4 Validation

1. Deploy a unikernel service. Note the image hash in `mcp status`.
2. Manually modify the `.img` file on disk.
3. Attempt to restart the service -- must fail with attestation error.
4. Redeploy (rebuilds the image) -- must succeed with a new hash.
5. Verify master reconciliation flags hash mismatches.

---

## Phase 5: Service Migration

**Goal**: Convert real platform services from containers to
unikernels, starting with the lowest-risk services and working toward
core infrastructure.

### 5.1 Migration Order

Services are migrated in order of increasing criticality and
decreasing tolerance for disruption:

**Wave 1 -- Stateless/low-risk worker services:**

| Service | Why first | Risk |
|---|---|---|
| mcdoc | Stateless doc renderer. No database. Public-facing but read-only. Failure means docs are down, not data loss. | Very low |
| mcat | MCIAS policy tester. Internal only. No persistent state. | Very low |

**Wave 2 -- Stateful worker services:**

| Service | Why second | Risk |
|---|---|---|
| mcq | Review queue. SQLite database. Has gRPC snapshot support. Good test of 9p + SQLite under unikernel. | Low-medium |

**Wave 3 -- Core infrastructure (only after Waves 1-2 are stable):**

| Service | Considerations | Risk |
|---|---|---|
| mcns | DNS server. Failure affects all name resolution. Must validate that Nanos's network stack handles DNS UDP correctly. | Medium |
| metacrypt | Seal/unseal lifecycle. Sensitive key material in memory. The reduced TCB is most valuable here. | Medium-high |
| mcr | Container registry. Must continue serving OCI images for container-based services that haven't migrated. | Medium |
| mcias | Root dependency. Every other service authenticates through it. Last to migrate. Must be thoroughly validated. | High |

**Not migrated:**

| Service | Reason |
|---|---|
| mc-proxy | Node infrastructure, not a deployed service. Runs on the host. |
| mcp-agent | Node infrastructure. Must have host access. Unikernel isolation is the opposite of what it needs. |
| mcp-master | Same as agent -- needs full host/network access. |

### 5.2 Per-Service Migration Procedure

For each service:

1. **Validate the binary under Nanos locally.** Before touching the
   control plane, run `ops run <binary> -c config.json` on a dev
   machine. Verify:
   - The service starts and passes health checks.
   - SQLite opens in WAL mode (if applicable).
   - TLS connections work (Nanos's TLS stack handles the Metacrypt CA
     cert).
   - 9p-mounted files are readable and writable.

2. **Deploy as unikernel on a worker node alongside the container
   version.** Use a different service name (e.g., `mcq-uk`) to run
   both versions simultaneously. Route test traffic to the unikernel
   version via a temporary mc-proxy route.

3. **Validate under real traffic.**
   - Health checks pass consistently.
   - gRPC and HTTP endpoints respond correctly.
   - Snapshots succeed.
   - Logs are readable via `mcp logs`.
   - SQLite performance is acceptable under 9p (benchmark IOPS).

4. **Cut over.** Update the real service definition to `runtime =
   "unikernel"` and redeploy. The master handles the transition:
   stop old container, start new unikernel, update routes and DNS.

5. **Soak.** Run for at least one full snapshot cycle (24h) before
   declaring stable. Monitor for:
   - Memory growth (unikernels have fixed memory, no swap).
   - 9p filesystem performance under sustained writes.
   - Clock drift (Nanos uses KVM clock, should be fine).

6. **Remove the container fallback.** Once stable, remove the
   parallel container deployment.

### 5.3 Rollback

If a unikernel service fails in production:

1. Change `runtime` back to `"container"` in the service definition.
2. `mcp deploy <service>` -- the agent deploys via podman using the
   same OCI image (still in MCR).
3. Routes and DNS update automatically.

Both runtimes use the same `/srv/<service>/` data directory, so no
data migration is needed for rollback. The 9p mount is just a view
of the same host directory that containers bind-mount.

### 5.4 Phase 5 Validation

Per wave:
- All services in the wave are running as unikernels.
- Snapshots complete successfully for all migrated services.
- `mcp status` shows all services healthy.
- Edge routing works for public services (mcdoc, mcq).
- No performance regression in SQLite operations.
- Successful `mcp migrate` of a unikernel service between nodes.

---

## Phase 6: Hardening and Long-Term

**Goal**: Operational maturity. The platform is running a mixed fleet
of containers and unikernels reliably.

### 6.1 Agent Upgrade for Unikernel Nodes

`mcp agent upgrade` currently cross-compiles and SCPs the agent
binary. No changes needed -- the agent is host software, not a
unikernel. Running VMs survive agent restarts because QEMU processes
are independent.

### 6.2 Boot Sequence for Unikernel Core Services

If core services (Wave 3) are migrated to unikernels, the agent's
boot sequence config needs to handle QEMU instead of podman for those
stages. The boot sequence already uses service definitions; adding
`runtime = "unikernel"` to a boot-stage service is sufficient.

**Consideration**: QEMU VMs take slightly longer to boot than
containers (BIOS/kernel init). Adjust stage timeouts if needed.

### 6.3 MCR Unikernel Image Storage (Phase 1.2b)

Once the pipeline is stable, implement pre-built unikernel images in
MCR. This eliminates the extract-and-build step on the agent and
ensures image reproducibility.

Add `mcp build` subcommand:

```
mcp build mcq --unikernel    # build .img, push to MCR as OCI artifact
mcp build mcq --container    # existing behavior
mcp build mcq --all          # both
```

### 6.4 Monitoring Dashboard

Add unikernel-specific metrics to `mcp status`:

- VM memory usage (from QMP `query-memory`)
- VM CPU usage (from QMP `query-cpus`)
- 9p I/O statistics
- Image hash and attestation status
- Serial console tail (last N lines)

### 6.5 Future: Capability Tokens

Independent of unikernels but synergistic. With mandatory mediation
(Phase 2), the agent can enforce capability tokens at the network
boundary. This is an MCIAS redesign, not an MCP change:

- MCIAS issues operation-scoped tokens ("bearer may read from mcq
  review queue") instead of identity tokens ("bearer is kyle").
- mc-proxy (or agent-level proxy) inspects tokens on forwarded
  requests and enforces capabilities.
- Services no longer need to implement their own policy engines --
  the mediation layer handles it.

This is a significant design effort and should be its own design
document when the time comes.

---

## Dependency Graph

```
Phase 1.1 (NixOS/KVM setup)
    │
    ├── Phase 1.2 (image building)
    │       │
    │       └── Phase 1.3 (QEMU runtime)
    │               │
    │               ├── Phase 1.4 (service def changes)
    │               │
    │               └── Phase 1.5 (resource tracking)
    │                       │
    │                       └── Phase 1.6 (validation) ─── PHASE 1 DONE
    │
    └── Phase 2.1 (bridge setup)
            │
            ├── Phase 2.2 (TAP management)
            │       │
            │       └── Phase 2.3 (IP assignment)
            │               │
            │               └── Phase 2.4 (mc-proxy routes)
            │
            └── Phase 2.5 (firewall) ─── Phase 2.6 (validation) ─── PHASE 2 DONE
                                                │
                                                ├── Phase 3 (snapshots/observability) ─── PHASE 3 DONE
                                                │
                                                └── Phase 4 (attestation) ─── PHASE 4 DONE
                                                        │
                                                        └── Phase 5 (service migration)
                                                                │
                                                                └── Phase 6 (hardening)
```

Phases 1 and 2 can be partially parallelized: bridge setup (2.1) only
depends on the NixOS/KVM setup (1.1), not on the QEMU runtime being
complete. However, Phase 2 validation requires Phase 1 to be done.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Nanos doesn't support a Go stdlib feature a service uses | Service won't start | Validate each binary under Nanos before committing to migration (Phase 5.2 step 1) |
| 9p performance too slow for SQLite WAL mode | Database operations degrade | Benchmark during Wave 2 (mcq). Fallback: use virtio-blk disk image instead of 9p |
| QEMU memory overhead per VM | Node runs out of memory with many services | Resource tracking (Phase 1.5) prevents overcommit. Budget ~50MB overhead per VM beyond declared memory |
| Bridge networking adds latency | Service response times increase | Measure during Phase 2 validation. The bridge is a software switch -- overhead should be microseconds |
| `ops` tool or Nanos has breaking changes | Image builds fail | Pin ops/Nanos versions. Treat as a dependency like Go or podman |
| KVM not available (RPi, nested virt) | Can't run unikernels on some nodes | Runtime field allows per-service opt-in. Container remains the default. Nodes without KVM simply don't get unikernel placements |

## Non-Goals

- **Replacing containers entirely.** Containers remain the default
  runtime. Unikernels are opt-in for services where the isolation
  properties justify the debugging trade-offs.
- **Multi-process unikernels.** Services that need sidecars (none
  currently) stay as containers.
- **Custom Nanos kernel builds.** Use stock Nanos. If a service needs
  kernel customization, it stays as a container.
- **Internet access from VMs.** VMs communicate only through mc-proxy.
  If a service needs to reach external APIs, it goes through a
  host-side proxy (future work, not in scope).
