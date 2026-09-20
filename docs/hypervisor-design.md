# Hypervisor-Based Service Isolation -- Design Notes

> **Status**: Brainstorming / future direction. This document is NOT
> an active work item. Agents should ignore this document unless
> specifically asked to consider it.

## Context

The metacircular platform runs Go services as rootless podman
containers orchestrated by MCP. This is a pragmatic execution of ideas
originally explored in a series of 2015 papers on security kernels,
environment isolation, and unikernels (see References). Those papers
describe a richer model than what containers provide: hardware-enforced
isolation, mandatory inter-environment communication mediation, and
capability-based access control. This document explores bridging the
gap by running services as unikernel VMs (specifically Nanos) on the
MCP control plane.

## Motivation: What Containers Don't Give Us

The current platform has the W7 security kernel's three properties --
isolated environments, inter-environment communication (IEC), and
access mediation -- but implemented cooperatively rather than enforced:

| W7 Property | Metacircular Today | Enforcement |
|---|---|---|
| Isolated environments | Rootless podman (namespaces/cgroups) | OS-cooperative -- shared kernel, escape CVEs exist |
| IEC | gRPC/TLS through mc-proxy | Application-cooperative -- services *choose* to route through mc-proxy |
| Access mediation | MCIAS tokens + per-service policies | Application-level -- services check tokens voluntarily |

The topology is right. The enforcement mechanism is weak. Containers
share a kernel, and any service could bypass mc-proxy to reach the
Tailnet directly.

## What Unikernels Buy Us

### Hardware-Enforced Isolation

Each service runs in its own VM with its own kernel. There is no
shared kernel to escape from. The security boundary is the hypervisor
(KVM), not Linux namespaces. This is the W7 "isolated environments"
model enforced by hardware, not convention.

### Mandatory IEC

This is the subtle but powerful part. A container on the Tailnet can
talk to anything. A unikernel VM with no direct network interface --
only a virtio-net device connected to a host-only bridge that the
agent controls -- cannot bypass the mediation layer. If the agent is
the only Tailnet citizen on the node and VMs can only reach the
agent's bridge, then mc-proxy stops being a routing convenience and
becomes the IEC mechanism. Communication between environments is
mediated by design, not by trust.

### Reduced TCB

Container TCB: Linux kernel + podman runtime + container image (often
a full distro). Unikernel TCB: KVM + Nanos runtime + the Go binary.
No shell, no package manager, no multi-user, no unnecessary syscalls.

## The Agent as Security Kernel

The MCP agent is already structurally positioned to be the W7 security
kernel for its node. It manages environment lifecycle, controls the
IEC layer (mc-proxy routes), provisions credentials (Metacrypt certs),
and reports to a central authority (master).

With unikernels, this role is formalized. The agent becomes the only
entity with host access. Services exist in VMs that can only
communicate through agent-controlled channels:

- **Network access**: virtio-net bridge under agent control; the agent
  decides what each VM can reach.
- **Storage access**: 9p/virtio-fs mounts; the agent controls what
  each VM sees on disk.
- **Credential access**: the agent provisions certs into the VM's
  filesystem before boot.
- **Identity**: the agent attests to the master what image hash is
  running in each VM.

## Why This Is Feasible for Metacircular

Several properties of the existing platform make this tractable:

- **Go + CGO_ENABLED=0**: Every service already produces a static ELF
  binary. Nanos needs exactly this. The `ops` tool packages them with
  minimal friction.

- **Single-process services**: Each service is one Go binary -- no
  sidecars, no shell scripts, no multi-process orchestration. That is
  the unikernel sweet spot.

- **Single-operator trust domain**: No multi-tenant capability
  delegation or federated attestation needed. The agent is the
  security kernel for its node; the master is the coordination point.

- **mc-proxy already mediates traffic**: The routing mesh is already
  in place. Making it mandatory (rather than optional) for unikernel
  VMs is an incremental change, not a new system.

## Design Sketch

### Runtime Abstraction

The agent gains a `Runtime` interface. Podman is one implementation;
QEMU/KVM is another. Service definitions gain a `runtime` field:

```toml
name    = "mcq"
runtime = "unikernel"    # or "container" (default)
tier    = "worker"
```

Both runtimes coexist. Services can be converted incrementally.

### Networking: Host-Only Bridge

Each unikernel VM gets a virtio-net device on a host-only bridge. The
agent runs on the bridge and controls forwarding. VMs cannot reach the
Tailnet directly. All external communication flows through mc-proxy on
the host.

This is structurally similar to how rootless podman already works
(container ports are localhost-only, mc-proxy routes to them), but
with the enforcement moved from convention to network topology.

### Storage: 9p Passthrough

Unikernel VMs mount `/srv/<service>/` via QEMU's `-virtfs` 9p
passthrough. Writes go directly to the host filesystem. This makes
snapshots work the same way as containers -- the agent tars the host
directory.

### Image Building

Two options (not mutually exclusive):

1. **Build on agent**: Agent extracts the ELF binary from the OCI
   image (pulled from MCR) and runs `ops build` locally.
2. **Store unikernel images in MCR**: OCI supports arbitrary media
   types. Unikernel `.img` files could be stored as OCI artifacts.

Option 1 is simpler to start with. Option 2 is cleaner long-term.

### Image Attestation

Before booting a unikernel, the agent hashes the image and reports it
to the master. The master compares against expected hashes from the
service definition. This is software attestation -- not TPM-based, but
it closes the "is this what I deployed?" question. It is a stepping
stone toward measured boot with hardware TPM.

### Snapshot Constraints

Unikernels have no shell. The `cli` and `exec:` snapshot methods
don't work. Only `grpc` snapshots are viable for unikernel services
(the service implements the standard `SnapshotService` RPC). The
default snapshot method (tar config/db/certs from the host-side 9p
mount) works unchanged since the agent tars the host directory, not
the VM filesystem.

### Debugging

No `podman exec`, no shell. Debugging relies on:

- Serial console output from QEMU
- gRPC health/status endpoints
- Structured logging to a file on the 9p mount
- The agent can snapshot and inspect VM state

This is a real loss of convenience. It is the price of proper
isolation -- as noted in the 2015 hypervisor paper, "the nature of
debugging means that isolation is broken."

## Difficulty Assessment

| Aspect | Difficulty | Notes |
|---|---|---|
| Building unikernel images from Go binaries | Easy | Already static ELF, `ops` handles it |
| QEMU lifecycle management in agent | Medium | Replace podman calls with qemu-system calls |
| Networking (host-only bridge + mc-proxy) | Medium | Similar to rootless podman model |
| Persistent storage via 9p | Medium | Well-supported in QEMU, maps to existing `/srv/` layout |
| Snapshots | Medium | `grpc` method works; `cli`/`exec` don't |
| Image attestation | Medium-Low | SHA-256 of image before boot |
| mc-proxy integration | Low | Just needs a reachable IP:port |
| Debugging/observability | Annoying | Loss of exec/shell access |

The minimum meaningful change is the runtime abstraction + isolated
networking together. Running a unikernel with full Tailnet access is
just a heavier container with worse debugging. The isolation properties
only kick in when the agent mediates all communication.

## Progression Path

1. **Runtime abstraction in the agent.** `Runtime` interface with
   podman and qemu implementations. Service definitions gain a
   `runtime` field. Both coexist.

2. **Isolated networking for unikernel VMs.** Host-only bridge per
   node, agent controls forwarding. mc-proxy becomes the mandatory
   IEC layer for unikernel services.

3. **Image attestation.** Agent hashes images before boot, reports to
   master. Master compares against expected values.

4. **Capability tokens (longer-term).** MCIAS issues operation-scoped
   tokens instead of identity tokens. The agent's mediation layer
   enforces them at the network boundary. This is independent of
   unikernels but synergizes with mandatory mediation.

## Open Questions

- **Tailscale integration**: Should unikernel VMs ever be first-class
  Tailnet citizens (via tsnet compiled into the binary), or should the
  agent always mediate? Mandatory mediation is more secure but means
  the agent is on the critical path for all traffic.

- **Resource limits**: QEMU VMs need explicit memory and CPU
  allocation. The current container model doesn't declare resource
  requirements. Unikernels would force this.

- **Mixed fleet**: During transition, some services run as containers
  and some as unikernels. mc-proxy routes to both. Does the master
  need to know the runtime type for placement decisions?

- **ARM support**: Nanos supports aarch64 but the QEMU/KVM story on
  Raspberry Pi (no KVM on all models) may limit unikernels to amd64
  nodes.

## References

- Rees, J. "A Security Kernel Based on the Lambda Calculus" (W7
  security kernel model -- isolated environments, IEC, access
  mediation)
- "Containers, isolation, and operating systems for network spaces"
  (2015) -- argues the OS must provide a security kernel; unikernels
  as viable isolation mechanism
- "A hypervisor for the modern age" (2015) -- problem statement for a
  hypervisor providing proper isolation, IEC, and access mediation
  with a programmatic administrative interface
- "A content-addressable data store with object capabilities" (Nebula,
  2015) -- capability-based access control model
- MCP v2 Architecture (`docs/architecture-v2.md`) -- current platform
  design this document builds on
