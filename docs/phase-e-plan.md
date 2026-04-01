# Phase E: Multi-Node Orchestration

Phase D (automated DNS registration) is complete. Phase E extends MCP from
a single-node agent on rift to a multi-node fleet with a central master
process.

## Goal

Deploy and manage services across multiple nodes from a single control
plane. The operator runs `mcp deploy` and the system places the workload on
the right node, provisions certs, registers DNS, and configures routing --
same as today on rift, but across the fleet.

## Fleet Topology

| Node | OS | Arch | Role |
|------|----|------|------|
| desktop (TBD) | NixOS | amd64 | Control plane -- runs master + MCIAS + MCNS |
| rift | NixOS | amd64 | Compute -- application services |
| orion | NixOS | amd64 | Compute |
| hyperborea | Debian | arm64 | Compute (Raspberry Pi) |
| svc | Debian | amd64 | Edge -- mc-proxy for public traffic, no containers |

Tailnet is the interconnect between all nodes. Public traffic enters via
mc-proxy on svc, which forwards over Tailnet to compute nodes.

## Components

### Master (`mcp-master`)

Long-lived orchestrator on the control plane node. Responsibilities:

- Accept CLI commands and dispatch to the correct agent
- Aggregate status from all agents (fleet-wide view)
- Node selection when `node` is omitted from a service definition
- Health-aware scheduling using agent heartbeat data

The master is stateless in the durable sense -- it rebuilds its world view
from agents on startup. If the master goes down, running services continue
unaffected; only new deploys and rescheduling stop.

### Agent upgrades

The fleet is heterogeneous (NixOS + Debian, amd64 + arm64), so NixOS flake
inputs don't work as a universal update mechanism.

**Design:** MCP owns the binary at `/srv/mcp/mcp-agent` on all nodes.

- `mcp agent upgrade [node]` -- CLI cross-compiles for the target's
  GOARCH, SCPs the binary, restarts via SSH
- Node config gains `ssh` (user@host) and `arch` (amd64/arm64) fields
- rift's NixOS `ExecStart` changes from nix store path to
  `/srv/mcp/mcp-agent`
- All nodes: binary at `/srv/mcp/mcp-agent`, systemd unit
  `mcp-agent.service`

Upgrades must be coordinated -- new RPCs cause `Unimplemented` errors on
old agents.

### Edge agents

svc runs an agent but does NOT run containers. Its agent manages mc-proxy
routing only: when the master provisions a service on a compute node, svc's
agent updates mc-proxy routes to point at the compute node's Tailnet
address.

### MCIAS migration

MCIAS moves from the svc VPS to the control plane node, running as an
MCP-managed container with an independent lifecycle. Bootstrap order:

1. MCIAS image pre-staged or pulled unauthenticated
2. MCIAS starts (L4 passthrough through mc-proxy -- manages its own TLS)
3. All other services bootstrap after MCIAS is up

## Scheduling

Three placement modes, in order of specificity:

1. `node = "rift"` -- explicit placement on a named node
2. `node = "pi-pool"` -- master picks within a named cluster
3. `node` omitted -- master picks any compute node with capacity

Resource-aware placement via agent heartbeats (CPU, memory, disk). RPis
with 4-8 GB RAM need resource tracking more than beefy servers.

## Open Questions

- **Control plane machine**: which desktop becomes the always-on node?
- **Heartbeat model**: agent push vs. master poll?
- **Cluster definition**: explicit pool config in master vs. node labels/tags?
- **MCIAS migration timeline**: when to cut over from svc to control plane?
- **Agent on svc**: what subset of agent RPCs does an edge-only agent need?

## What Phase E Does NOT Include

These remain future work:

- Auto-reconciliation (agent auto-restarting drifted containers)
- Migration (snapshot streaming between nodes)
- Web UI for fleet management
- Observability / log aggregation
- Object store
