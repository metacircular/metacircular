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
| rift | NixOS | amd64 | Master + worker -- runs mcp-master, core infra, and application services |
| orion | NixOS | amd64 | Worker |
| hyperborea | Debian | arm64 | Worker (Raspberry Pi) |
| svc | Debian | amd64 | Edge -- mc-proxy for public traffic, no containers |

Tailnet is the interconnect between all nodes. Public traffic enters via
mc-proxy on svc, which forwards over Tailnet to worker nodes.

## Key Architecture Decisions

These were resolved in the 2026-04-01 design session:

1. **Rift is the master node.** No separate straylight machine. Core infra
   stays on rift, which gains mcp-master alongside its existing agent.

2. **Master-mediated coordination.** Agents never talk to each other. All
   cross-node operations go through the master. Agents only dial the master
   (for registration and heartbeats) and respond to master RPCs.

3. **Agent self-registration.** Agents register with the master on startup
   (name, role, address, arch). The master maintains the live node registry.
   No static `[[nodes]]` config required except for bootstrap.

4. **Heartbeats with fallback probe.** Agents push heartbeats every 30s
   (with resource data). If the master misses 3 heartbeats (90s), it
   actively probes the agent. Failed probe marks the node unhealthy.

5. **Tier-based placement.** `tier = "core"` runs on the master node.
   `tier = "worker"` (default) is auto-placed on a worker with capacity.
   Explicit `node = "orion"` overrides tier for pinned services.

6. **Two separate certs for public services.** Internal cert
   (`svc.mcp.metacircular.net`) issued by worker agent. Public cert
   (`metacircular.net`) issued by edge agent. Internal names never
   appear on edge certs.

7. **`public = true` on routes.** Public routes declare intent with a
   boolean flag. The master assigns the route to an edge node (currently
   always svc). No explicit `edge` field in service definitions.

## Components

### Master (`mcp-master`)

Long-lived orchestrator on rift. Responsibilities:

- Accept CLI commands and dispatch to the correct agent
- Maintain node registry from agent self-registration
- Place services based on tier, explicit node, and resource availability
- Detect `public = true` routes and coordinate edge setup
- Validate public hostnames against allowed domain list
- Aggregate status from all agents (fleet-wide view)
- Probe agents on missed heartbeats

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

### Edge agents

svc runs an agent but does NOT run containers. Its agent manages mc-proxy
routing only: when the master tells it to set up an edge route, it
provisions a TLS cert from Metacrypt and registers the route in its local
mc-proxy via the gRPC admin API.

## Migration Plan

### Phase 1: Agent on svc
Deploy mcp-agent to svc. Verify with `mcp node list`.

### Phase 2: Edge routing RPCs
Implement SetupEdgeRoute/RemoveEdgeRoute/ListEdgeRoutes on the agent.
Test by calling directly from CLI.

### Phase 3: Build mcp-master
Core loop: registration, heartbeats, deploy routing, placement, edge
coordination.

### Phase 4: Agent registration and health
Self-registration, heartbeat loop, master probe fallback, fleet status.

### Phase 5: Cut over
Point CLI at master, add tier fields to service defs, deploy agents to
orion and hyperborea.

## What Phase E Does NOT Include

These remain future work:

- Auto-reconciliation (agent auto-restarting drifted containers)
- Live migration (snapshot streaming between nodes)
- Web UI for fleet management
- Observability / log aggregation
- Object store
- Multiple edge nodes / master HA
