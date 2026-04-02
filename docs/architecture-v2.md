# MCP v2 -- Multi-Node Control Plane

## Overview

MCP v2 introduces multi-node orchestration with a master/agent topology.
The CLI no longer dials agents directly. A dedicated **mcp-master** daemon
coordinates deployments across nodes, handles cross-node concerns (edge
routing, certificate provisioning, DNS), and serves as the single control
point for the platform.

### Motivation

v1 deployed successfully on a single node (rift) but exposed operational
pain points as services needed public-facing routes through svc:

- **Manual edge routing**: Exposing mcq.metacircular.net required hand-editing
  mc-proxy's TOML config on svc, provisioning a TLS cert manually, updating
  the SQLite database when the config and database diverged, and debugging
  silent failures. Every redeployment risked breaking the public route.

- **Dynamic port instability**: The route system assigns ephemeral host ports
  that change on every deploy. svc's mc-proxy pointed at a specific port
  (e.g., `100.95.252.120:48080`), which went stale after redeployment.
  Container ports are also localhost-only under rootless podman, requiring
  explicit Tailscale IP bindings for external access.

- **$PORT env override conflict**: The mcdsl config loader overrides
  `listen_addr` from `$PORT` when routes are present. This meant containers
  ignored their configured port and listened on the route-allocated one
  instead, breaking explicit port mappings that expected the config port.

- **Cert chain issues**: mc-proxy requires full certificate chains (leaf +
  intermediates). Certs provisioned outside the standard metacrypt flow
  were leaf-only and caused silent TLS handshake failures (`client_bytes=7
  backend_bytes=0` with no error logged).

- **mc-proxy database divergence**: mc-proxy persists routes in SQLite.
  Routes added via the admin API override the TOML config. Editing the TOML
  alone had no effect until the database was manually updated -- a failure
  mode that took hours to diagnose.

- **No cross-node coordination**: The v1 CLI talks directly to individual
  agents. There is no mechanism for one agent to tell another "set up a
  route for this service." Every cross-node operation was manual.

v2 addresses all of these by making the master the single coordination
point for deployments, with agents handling local concerns (containers,
mc-proxy routes, cert provisioning) on instruction from the master.

### What Changes from v1

| Concern | v1 | v2 |
|---------|----|----|
| CLI target | CLI dials agents directly | CLI dials the master |
| Node awareness | CLI routes by `node` field in service defs | Master owns the node registry |
| Service placement | Explicit `node` required | `tier` field; master auto-places workers |
| Edge routing | Manual mc-proxy config on svc | Master coordinates edge setup |
| Cert provisioning | Agent provisions for local mc-proxy only | Edge agent provisions its own public certs |
| DNS registration | Agent registers records on deploy | Master coordinates DNS across zones |
| Auth model | Token validation only | Per-RPC role-based authorization |

### What Stays the Same

The agent's core responsibilities are unchanged: it manages containers via
podman, stores its local registry in SQLite, monitors for drift, and alerts
the operator. The agent gains new RPCs for edge routing and health reporting
but does not become aware of other nodes -- the master handles all
cross-node coordination. Agents never communicate with each other.

---

## Topology

```
Operator workstation (vade)
  ┌──────────────────────────┐
  │  mcp (CLI)               │
  │                          │
  │  gRPC ───────────────────┼─── Tailnet ──┐
  └──────────────────────────┘              │
                                            ▼
Master + worker node (rift)
  ┌──────────────────────────────────────────────────────┐
  │  mcp-master                                          │
  │    ├── node registry (agents self-register)          │
  │    ├── service placement (tier-aware)                │
  │    ├── edge routing coordinator                      │
  │    └── SQLite state (edge routes, placements)        │
  │                                                      │
  │  mcp-agent                                           │
  │    ├── mcias container                               │
  │    ├── mcns container                                │
  │    ├── metacrypt container                           │
  │    ├── mcr container                                 │
  │    ├── mcq, mcdoc, exo, sgard, kls ...              │
  │    └── mc-proxy (rift)                               │
  └──────────┬──────────────────┬───────────┬────────────┘
             │                  │           │
          Tailnet           Tailnet      Tailnet
             │                  │           │
             ▼                  ▼           ▼
Worker (orion)                  Edge (svc)
  ┌──────────────────┐           ┌─────────────────────┐
  │  mcp-agent       │           │  mcp-agent          │
  │    ├── services  │           │    ├── mc-proxy     │
  │    └── mc-proxy  │           │    └── (routes only)│
  └──────────────────┘           └─────────────────────┘
  NixOS / amd64                  Debian / amd64
```

### Node Roles

| Role | Purpose | Nodes |
|------|---------|-------|
| **master** | Runs mcp-master + mcp-agent. Hosts core infrastructure. Single coordination point. | rift |
| **worker** | Runs mcp-agent. Hosts application services. | orion |
| **edge** | Runs mcp-agent. Terminates public TLS, forwards to internal services. No application containers. | svc |

Every node runs an mcp-agent. Rift also runs mcp-master. The master's
local agent manages the infrastructure services (MCIAS, mcns, metacrypt,
mcr) the same way other agents manage application services.

### mc-proxy Mesh

Each node runs its own mc-proxy instance. They form a routing mesh:

```
mc-proxy (rift)
  ├── :443  L7 routes for internal .svc.mcp hostnames
  ├── :8443 L4 passthrough for API servers (MCIAS, metacrypt, mcr)
  └── :9443 L4 passthrough for gRPC services

mc-proxy (orion)
  ├── :443  L7 routes for services hosted on this node
  └── :8443 L4/L7 routes for internal APIs

mc-proxy (svc)
  └── :443  L7 termination for public hostnames
            → forwards to internal .svc.mcp endpoints over Tailnet
```

---

## Security Model

### Authentication and Authorization

All gRPC channels (CLI↔master, master↔agent, agent→master) use TLS 1.3
with MCIAS bearer tokens. Every entity has a distinct MCIAS identity:

| Entity | MCIAS Identity | Account Type |
|--------|---------------|--------------|
| Operator CLI | `kyle` (or personal account) | human |
| mcp-master | `mcp-master` | service |
| Agent on rift | `agent-rift` | service |
| Agent on orion | `agent-orion` | service |
| Agent on svc | `agent-svc` | service |

RPCs are authorized by **caller role**, not just authentication:

| RPC Category | Allowed Callers | Rejected |
|--------------|-----------------|----------|
| CLI→master (Deploy, Undeploy, Status, Sync) | human accounts, `mcp-master` (for self-management) | agent service accounts |
| Agent→master (Register, Heartbeat) | `agent-*` service accounts | human accounts, `mcp-master` |
| Master→agent (Deploy, SetupEdgeRoute, HealthCheck) | `mcp-master` only | all others |

The auth interceptor on both master and agent validates the bearer token
via MCIAS, then checks the caller's account type and service name against
the RPC's allowed-caller list. Unauthorized calls return
`PermissionDenied`.

### Trust Assumptions

The master is a **fully trusted** component. A compromised master can
control the entire fleet: deploy arbitrary containers, exfiltrate data
via snapshots, redirect traffic via edge routes. This is inherent to
the master/agent topology and acceptable for a single-operator personal
platform. Mitigations: the master runs on the operator's always-on
machine (rift) behind Tailscale, authenticates to MCIAS with its own
service identity, and all communication is TLS 1.3.

### TLS Verification

All gRPC connections verify the peer's TLS certificate against the
Metacrypt CA cert. Agents configure the CA cert path in their config:

```toml
[tls]
ca_cert = "/srv/mcp/certs/metacircular-ca.pem"
```

When an agent starts before the master is available (e.g., svc's agent
starts before rift's boot sequence completes), the TLS connection fails
and the agent retries with exponential backoff. The CA cert itself is
pre-provisioned on all nodes — it does not depend on Metacrypt being
running.

### Registration Security

Agents self-register with the master, but registration is **identity-bound**:

1. The master extracts the caller's MCIAS service name from the validated
   token (e.g., `agent-rift`).
2. The expected node name is derived by stripping the `agent-` prefix.
3. The `RegisterRequest.name` must match. `agent-rift` can only register
   `name = "rift"`. A rogue agent cannot impersonate another node.
4. The master maintains an allowlist of permitted agent identities:

```toml
[registration]
allowed_agents = ["agent-rift", "agent-svc", "agent-orion"]
```

Registration from unknown identities is rejected. Re-registration from the
same identity updates the entry (handles restarts) and logs a warning with
the previous address for audit.

### Edge Route Validation

When the master sets up an edge route, it validates both ends:

- **Public hostname**: must fall under an allowed domain
  (`metacircular.net`, `wntrmute.net`). Validation uses proper domain
  label matching — `evilmetacircular.net` is rejected. Implementation:
  the hostname must equal the allowed domain or be preceded by a `.`
  (e.g., `mcq.metacircular.net` matches, `metacircular.net` matches,
  `xmetacircular.net` does not).

- **Backend hostname**: must end with `.svc.mcp.metacircular.net`
  (the internal DNS zone). The edge agent resolves it and verifies the
  result is a Tailnet IP (100.64.0.0/10). Non-Tailnet backends are
  rejected.

### Certificate Issuance Policies

Per-identity restrictions in Metacrypt limit what each agent can issue:

| Agent | Allowed SANs | Denied SANs |
|-------|-------------|-------------|
| `agent-rift`, `agent-orion` | `*.svc.mcp.metacircular.net` | public domains |
| `agent-svc` | `*.metacircular.net`, `*.wntrmute.net` | `.svc.mcp.` names |

This ensures a compromised edge agent cannot issue certs for internal
names, and a compromised worker agent cannot issue certs for public
names. The Metacrypt CA is not publicly trusted, which limits blast
radius further.

### Rate Limiting

The master rate-limits agent RPCs:

- `Register`: 1 per minute per identity.
- `Heartbeat`: 1 per 10 seconds per identity.
- Maximum registered nodes: 16 (configurable).

Excess calls return `ResourceExhausted`.

### Tailscale ACLs

Network-level restriction (configured in Tailscale admin, not MCP):

- rift (master): can reach all agent gRPC ports (9444) on all nodes.
  The master process needs this to forward deploys and set up edge
  routes.
- svc: can reach master gRPC (9555), backend service ports (443, 8443,
  9443), and Metacrypt (8443). Blocked from MCIAS management, MCR push,
  and agent gRPC on other nodes.
- Workers: can reach master gRPC, MCR (pull), Metacrypt, MCIAS. Blocked
  from other workers' agent ports and svc's agent port.

---

## Service Placement

Services declare a **tier** that determines where they run:

- **`tier = "core"`** — scheduled on the master node. Used for platform
  infrastructure: MCIAS, metacrypt, mcr, mcns.
- **`tier = "worker"`** (default) — auto-placed on a worker node. The
  master selects the node based on container count and health.

Explicit node pinning is still supported via `node = "orion"` for cases
where a service must run on a specific machine. When `node` is set, it
overrides `tier`.

### Placement Algorithm

Worker placement is deliberately simple:

1. Filter eligible nodes: healthy workers.
2. Select the node with the fewest running containers.
3. Break ties alphabetically by node name (deterministic).

All v2 nodes are amd64, so architecture filtering is not needed.
Services do not declare resource requirements for v2. The heartbeat
reports available resources (CPU, memory, disk) which the master uses
for health assessment, but placement is container-count based. Resource-
aware bin-packing is future work.

### Service Definition

```toml
name   = "mcq"
tier   = "worker"                    # default; placed by master
active = true

[[components]]
name    = "mcq"
image   = "mcr.svc.mcp.metacircular.net:8443/mcq:v0.4.0"
volumes = ["/srv/mcq:/srv/mcq"]
cmd     = ["server", "--config", "/srv/mcq/mcq.toml"]

# Internal route: handled by the local node's mc-proxy.
[[components.routes]]
name     = "internal"
port     = 8443
mode     = "l7"

# Public route: master sets up edge routing on svc.
[[components.routes]]
name     = "public"
port     = 8443
mode     = "l7"
hostname = "mcq.metacircular.net"
public   = true
```

Core service example:

```toml
name   = "mcias"
tier   = "core"                      # always on master node
active = true

[[components]]
name    = "mcias"
image   = "mcr.svc.mcp.metacircular.net:8443/mcias:v1.10.5"
volumes = ["/srv/mcias:/srv/mcias"]
cmd     = ["mciassrv", "-config", "/srv/mcias/mcias.toml"]
```

### v1 Compatibility

Existing v1 service definitions with `node = "rift"` continue to work
(explicit pinning). New v2 fields (`tier`, `public`) default to their
zero values (`"worker"`, `false`) when absent. The validation rule
changes from "node required" to "either node or tier must be set;
tier defaults to worker if both are empty."

---

## Proto Definitions

### ServiceSpec and RouteSpec Updates

```protobuf
message ServiceSpec {
  string name    = 1;
  bool   active  = 2;
  repeated ComponentSpec components = 3;  // unchanged from v1
  string tier    = 4;  // "core" or "worker" (default: "worker")
  string node    = 5;  // explicit node pin (overrides tier)
  SnapshotConfig snapshot = 6;  // snapshot method and excludes
}

message SnapshotConfig {
  string method            = 1;  // "grpc", "cli", "exec: <cmd>", "full", or "" (default)
  repeated string excludes = 2;  // paths relative to /srv/<service>/ to skip
}

message RouteSpec {
  string name     = 1;
  int32  port     = 2;
  string mode     = 3;  // "l4" or "l7"
  string hostname = 4;
  bool   public   = 5;  // triggers edge routing
}
```

### McpMasterService

```protobuf
service McpMasterService {
  // CLI operations.
  rpc Deploy(MasterDeployRequest) returns (MasterDeployResponse);
  rpc Undeploy(MasterUndeployRequest) returns (MasterUndeployResponse);
  rpc Status(MasterStatusRequest) returns (MasterStatusResponse);
  rpc Sync(MasterSyncRequest) returns (MasterSyncResponse);
  rpc Migrate(MigrateRequest) returns (MigrateResponse);

  // Fleet management.
  rpc ListNodes(ListNodesRequest) returns (ListNodesResponse);

  // Snapshots (CLI-triggered).
  rpc CreateSnapshot(CreateSnapshotRequest) returns (CreateSnapshotResponse);
  rpc ListSnapshots(ListSnapshotsRequest) returns (ListSnapshotsResponse);

  // Agent registration and health (called by agents).
  rpc Register(RegisterRequest) returns (RegisterResponse);
  rpc Heartbeat(HeartbeatRequest) returns (HeartbeatResponse);
}

message MasterDeployRequest {
  ServiceSpec service = 1;
}

message MasterDeployResponse {
  string node    = 1;  // node the service was placed on
  bool   success = 2;  // true only if ALL steps succeeded
  string error   = 3;
  // Per-step results for operator visibility. Partial failure is
  // possible: deploy succeeds but edge routing fails. The CLI shows
  // exactly what worked and what didn't.
  StepResult deploy_result     = 4;
  StepResult edge_route_result = 5;
  StepResult dns_result        = 6;
}

message StepResult {
  string step    = 1;
  bool   success = 2;
  string error   = 3;
}

message MasterUndeployRequest {
  string service_name = 1;
}

message MasterUndeployResponse {
  bool   success = 1;
  string error   = 2;
}

message MasterStatusRequest {
  string service_name = 1;  // empty = all services
}

message MasterStatusResponse {
  repeated ServiceStatus services = 1;
}

message ServiceStatus {
  string name   = 1;
  string node   = 2;
  string tier   = 3;
  string status = 4;  // "running", "stopped", "unhealthy", "unknown"
  repeated EdgeRouteStatus edge_routes = 5;
}

message EdgeRouteStatus {
  string hostname   = 1;
  string edge_node  = 2;
  string cert_expires = 3;
}

message MasterSyncRequest {
  repeated ServiceSpec services = 1;
}

message MasterSyncResponse {
  repeated StepResult results = 1;
}

message ListNodesRequest {}

message ListNodesResponse {
  repeated NodeInfo nodes = 1;
}

message NodeInfo {
  string name     = 1;
  string role     = 2;
  string address  = 3;
  string arch     = 4;
  string status   = 5;  // "healthy", "unhealthy", "unknown"
  int32  containers = 6;
  string last_heartbeat = 7;  // RFC3339
}

message RegisterRequest {
  string name    = 1;
  string role    = 2;
  string address = 3;
  string arch    = 4;
}

message RegisterResponse {
  bool accepted = 1;
}

message HeartbeatRequest {
  string name           = 1;
  int64  cpu_millicores = 2;
  int64  memory_bytes   = 3;
  int64  disk_bytes     = 4;
  int32  containers     = 5;
}

message HeartbeatResponse {
  bool acknowledged = 1;
}
```

### Agent RPC Additions

```protobuf
// Health probe -- called by master on missed heartbeats.
rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);

// Edge routing -- called by master on edge nodes.
rpc SetupEdgeRoute(SetupEdgeRouteRequest) returns (SetupEdgeRouteResponse);
rpc RemoveEdgeRoute(RemoveEdgeRouteRequest) returns (RemoveEdgeRouteResponse);
rpc ListEdgeRoutes(ListEdgeRoutesRequest) returns (ListEdgeRoutesResponse);

message HealthCheckRequest {}

message HealthCheckResponse {
  string status     = 1;  // "healthy" or "degraded"
  int32  containers = 2;
}

message SetupEdgeRouteRequest {
  string hostname         = 1;  // public hostname
  string backend_hostname = 2;  // internal .svc.mcp hostname
  int32  backend_port     = 3;  // port on worker's mc-proxy
  bool   backend_tls      = 4;  // MUST be true; agent rejects false
}

message SetupEdgeRouteResponse {}

message RemoveEdgeRouteRequest {
  string hostname = 1;
}

message RemoveEdgeRouteResponse {}

message ListEdgeRoutesRequest {}

message ListEdgeRoutesResponse {
  repeated EdgeRoute routes = 1;
}

message EdgeRoute {
  string hostname         = 1;
  string backend_hostname = 2;
  int32  backend_port     = 3;
  string cert_serial      = 4;
  string cert_expires     = 5;
}
```

---

## Agent Registration and Health

### Registration

Agents self-register with the master on startup by calling
`McpMasterService.Register`. The master validates the caller's MCIAS
identity (see Security Model) and adds the node to its registry (SQLite).

If the master is unreachable at startup, the agent retries with
exponential backoff (1s, 2s, 4s, ... capped at 60s). Running containers
are unaffected — registration is a management concern, not a runtime one.

### Heartbeats

Agents send heartbeats every 30 seconds via `McpMasterService.Heartbeat`.
Each heartbeat includes resource data (CPU, memory, disk, container count).
The master derives the agent's node name from the authenticated MCIAS
identity (same as registration) — the `name` field in the heartbeat is
verified against the token, not trusted blindly.

If the master has not received a heartbeat from an agent in 90 seconds
(3 missed intervals), it probes the agent with `HealthCheck`. If the
probe fails (5-second timeout), the agent is marked unhealthy. Unhealthy
nodes are excluded from placement but their services continue running.

When a previously unhealthy agent sends a heartbeat, the master marks it
healthy again.

### Node Identity

Each agent authenticates to MCIAS as a distinct service user:
`agent-rift`, `agent-svc`, `agent-orion`. Benefits:

- **Audit**: logs show which node performed an action.
- **Least privilege**: edge agents don't need image pull access.
- **Revocation**: a compromised node's credentials can be revoked
  without affecting the fleet.

---

## mcp-master

### Responsibilities

1. **Accept CLI commands** via gRPC (deploy, undeploy, status, sync).
2. **Maintain node registry** from agent self-registration (SQLite).
3. **Place services** on nodes based on tier, explicit node, and
   container count.
4. **Detect public routes** (`public = true`) and coordinate edge routing.
5. **Validate public hostnames** against allowed domain list.
6. **Assign edge nodes** for public routes (currently always svc).
7. **Coordinate undeploy** across nodes.
8. **Aggregate status** from all agents for fleet-wide views.

### What the Master Does NOT Do

- Store container state (agents own their registries).
- Manage container lifecycle directly (agents do this).
- Run containers (the co-located agent does).
- Replace the agent on any node.
- Talk to agents on behalf of other agents.

### Master State (SQLite)

The master maintains a SQLite database at `/srv/mcp-master/master.db`
with three tables:

```sql
-- Registered nodes. Populated by agent Register RPCs.
-- Rebuilt from agent re-registration on master restart.
CREATE TABLE nodes (
    name        TEXT PRIMARY KEY,
    role        TEXT NOT NULL,
    address     TEXT NOT NULL,
    arch        TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'unknown',
    containers  INTEGER NOT NULL DEFAULT 0,
    last_heartbeat TEXT
);

-- Service placements. Records which node hosts which service.
-- Populated on deploy, removed on undeploy.
CREATE TABLE placements (
    service_name TEXT PRIMARY KEY,
    node         TEXT NOT NULL REFERENCES nodes(name),
    tier         TEXT NOT NULL,
    deployed_at  TEXT NOT NULL
);

-- Edge routes. Records public routes for undeploy cleanup.
CREATE TABLE edge_routes (
    hostname         TEXT PRIMARY KEY,
    service_name     TEXT NOT NULL REFERENCES placements(service_name),
    edge_node        TEXT NOT NULL REFERENCES nodes(name),
    backend_hostname TEXT NOT NULL,
    backend_port     INTEGER NOT NULL,
    created_at       TEXT NOT NULL
);

-- Snapshot metadata. The archive files live on disk at
-- /srv/mcp-master/snapshots/<service>/<timestamp>.tar.zst.
CREATE TABLE snapshots (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    service_name TEXT    NOT NULL,
    node         TEXT    NOT NULL,
    filename     TEXT    NOT NULL,
    size_bytes   INTEGER NOT NULL,
    created_at   TEXT    NOT NULL
);
CREATE INDEX idx_snapshots_service ON snapshots(service_name, created_at DESC);
```

On master restart, the node registry is rebuilt as agents re-register
(within 30s via heartbeat). Placements and edge routes persist across
restarts. The master reconciles placements against actual agent state
on startup (see Reconciliation).

### Master Configuration

```toml
[server]
grpc_addr = "100.x.x.x:9555"     # master listens on Tailnet
tls_cert  = "/srv/mcp-master/certs/cert.pem"
tls_key   = "/srv/mcp-master/certs/key.pem"

[database]
path = "/srv/mcp-master/master.db"

[mcias]
server_url   = "https://mcias.metacircular.net:8443"
service_name = "mcp-master"

[edge]
allowed_domains = ["metacircular.net", "wntrmute.net"]

[registration]
allowed_agents = ["agent-rift", "agent-svc", "agent-orion"]
max_nodes      = 16

[timeouts]
deploy       = "5m"
edge_route   = "30s"
health_check = "5s"
undeploy     = "2m"
snapshot     = "10m"

# Bootstrap: master's own agent (can't self-register before master starts).
[[nodes]]
name    = "rift"
address = "100.95.252.120:9444"
role    = "master"
```

### Boot Sequencing

The master node runs core infrastructure that other services depend on.
On boot, these services must start in dependency order. Only the master
needs sequencing -- worker and edge nodes start their agent and wait
for registration with the master.

The master's agent config declares boot stages:

```toml
[[boot.sequence]]
name     = "foundation"
services = ["mcias", "mcns"]
timeout  = "120s"
health   = "tcp"

[[boot.sequence]]
name     = "core"
services = ["metacrypt", "mcr"]
timeout  = "60s"
health   = "tcp"

[[boot.sequence]]
name     = "management"
services = ["mcp-master"]
timeout  = "30s"
health   = "grpc"
```

**Stage 1 -- Foundation**: MCIAS and MCNS start first. Every other
service needs authentication (MCIAS) and DNS resolution (MCNS).

**Stage 2 -- Core**: Metacrypt and MCR start once auth and DNS are
available. Agents need Metacrypt for cert provisioning and MCR for
image pulls.

**Stage 3 -- Management**: MCP-Master starts last. It requires all
infrastructure services to be running before it can coordinate the fleet.

mcp-master runs as a container managed by the agent, just like any
other service. This means updates are a normal `mcp deploy` (or image
bump in the bootstrap config), and the agent handles restarts via
podman's `--restart unless-stopped` policy.

**Bootstrap (first boot):** On initial cluster setup, no images exist
in MCR yet. The boot sequence config references container images, but
MCR doesn't start until stage 2. Resolution:

- Stage 1 and 2 images (MCIAS, MCNS, Metacrypt, MCR) must be
  **pre-staged** into the local podman image store before first boot
  (`podman load` or `podman pull` from an external source).
- Once MCR is running (stage 2), stage 3 (mcp-master) can pull its
  image from MCR normally.
- Subsequent boots use cached images. Image updates go through the
  normal `mcp deploy` flow (which pulls from MCR).

The boot sequence config contains full service definitions (image,
volumes, cmd, routes) — not just service names. This is the only
place where service definitions live on the agent rather than being
pushed from the CLI via the master.

**Health check types:**
- `tcp` — connect to the container's mapped port. Success = connection
  accepted. Used for most services.
- `grpc` — call the gRPC health endpoint. Used for services with gRPC.
- `http` — GET a health endpoint. Future option.

**Timeout behavior:** Depends on the stage:
- **Foundation** (MCIAS, MCNS): failure **blocks** boot. The agent
  retries indefinitely with backoff and alerts the operator. All
  downstream services depend on auth and DNS — proceeding is futile.
- **Core and management**: failure logs an error and proceeds. The
  operator can fix the failed service manually. Partial boot is
  better than no boot for non-foundation services.

The agent treats boot sequencing as a startup concern only. Once all
stages complete, normal operations proceed. If a foundation service
crashes at runtime, the agent restarts it independently via the
`--restart unless-stopped` podman policy.

**Boot config drift:** The boot sequence config contains pinned image
versions. When the operator updates a service via `mcp deploy`, the
boot config is NOT automatically updated. On reboot, the agent starts
the old version; the master then deploys the current version. This is
self-correcting for core and management services, but foundation
services (MCIAS, MCNS) run before the master exists. **When updating
foundation service images, also update the boot sequence config.**

### Reconciliation

On startup, the master actively probes all nodes it knows about from
its persisted `nodes` table — it does not wait for agents to
re-register. This means the master has a fleet-wide view within seconds
of starting, rather than waiting up to 30s per agent heartbeat cycle.

The initial probe cycle is a **warm-up** phase: the master builds its
fleet view but does not emit health alerts. Once all known nodes have
been probed (or the probe timeout expires), the master transitions to
**ready** and begins normal health alerting. This avoids noisy
"unhealthy" warnings for agents that simply haven't started yet.

1. **Probe known nodes**: For each node in the `nodes` table, the
   master calls `HealthCheck` (5s timeout). Nodes that respond are
   marked healthy; nodes that don't respond are marked unhealthy.
   Agent self-registration still runs in the background and updates
   addresses or adds new nodes, but reconciliation does not depend
   on it.
2. **Check placements**: For each placement in the database, query
   the hosting agent's `Status` RPC (bulk — one call per agent, not
   per service). If the agent reports a service is not running, mark
   the placement as stale (log warning, do not auto-redeploy).
3. **Detect orphans**: For each service running on an agent that has
   no matching placement record, log it as an orphan. Orphans may
   result from failed deploys, manual `podman run`, or v1 leftovers.
4. **Check edge routes**: For each edge route in the database, query
   the edge agent for route status.
5. **Check snapshot freshness**: Flag any service whose latest
   snapshot is older than 2x the snapshot interval (e.g., older than
   48 hours with a 24-hour cycle). Stale snapshots are a disaster
   recovery risk.
6. **Report**: All discrepancies (stale placements, orphans, missing
   edge routes, unhealthy nodes, stale snapshots) are reported via
   `mcp status` and structured logs.

Reconciliation is read-only — it detects drift but does not
auto-remediate. The operator reviews `mcp status` output and takes
action. Auto-reconciliation is future work.

---

## Edge Routing

The core v2 feature: when a service declares `public = true` on a route,
the master automatically provisions the edge route.

### Deploy Flow with Edge Routing

When the master receives `Deploy(mcq)`:

1. **Place service**: Master selects the target node based on tier/node/
   container count. For mcq (tier=worker), master picks the least-loaded
   healthy worker.

2. **Deploy to worker**: Master sends `Deploy` RPC to the worker's agent
   (timeout: 5m). The agent deploys the container, provisions a TLS cert
   for `mcq.svc.mcp.metacircular.net` from Metacrypt, and registers the
   internal mc-proxy route.

3. **Register DNS**: Master registers an A record for the internal
   hostname (`mcq.svc.mcp.metacircular.net`) pointing to the worker's
   Tailnet IP via MCNS. This is the backend address that edge and
   internal clients resolve.

4. **Detect public routes**: Master inspects the service spec for routes
   with `public = true`.

5. **Validate hostname**: Master checks that `mcq.metacircular.net` falls
   under an allowed domain using proper domain label matching.

6. **Check public DNS**: Master resolves `mcq.metacircular.net` to
   verify it points to the edge node's public IP. Public DNS records
   are pre-provisioned manually at Hurricane Electric. If the hostname
   does not resolve, the master warns but continues — the operator
   may be setting up DNS in parallel.

7. **Validate backend hostname**: Master verifies the internal hostname
   (`mcq.svc.mcp.metacircular.net`) ends with `.svc.mcp.metacircular.net`.
   The internal hostname is derived from the service and component name
   using the convention `<component>.svc.mcp.metacircular.net`.

8. **Assign edge node**: Master selects an edge node (currently svc).

9. **Set up edge route**: Master sends `SetupEdgeRoute` RPC to svc's
   agent (timeout: 30s):
   ```
   SetupEdgeRoute(
     hostname:         "mcq.metacircular.net"
     backend_hostname: "mcq.svc.mcp.metacircular.net"
     backend_port:     8443
     backend_tls:      true
   )
   ```

10. **Svc agent provisions**: On receiving `SetupEdgeRoute`, svc's agent:
    a. Validates that `backend_hostname` ends with `.svc.mcp.metacircular.net`.
    b. Resolves `backend_hostname` — verifies result is a Tailnet IP
       (100.64.0.0/10).
    c. Provisions a TLS certificate from Metacrypt for the **public**
       hostname `mcq.metacircular.net` only. Internal names never appear
       on edge certs.
    d. Registers an L7 route in its local mc-proxy:
       `mcq.metacircular.net:443 → <worker-tailnet-ip>:8443`
       with `backend_tls = true`.

11. **Master records the edge route** in its SQLite database.

12. **Master returns structured result** to CLI with per-step status.

**Failure handling:** If any step fails, the master returns the error
to the CLI with the step that failed. If the deploy succeeded but
edge routing failed, the service is running internally but not publicly
reachable. The operator can retry with `mcp deploy` (idempotent) or
fix the issue and run `mcp sync`.

If cert provisioning fails during deploy (step 2 or 8), the deploy
**fails** — the agent does not register an mc-proxy route pointing to
a nonexistent cert. This prevents the silent TLS failure from v1.

### Undeploy Flow

1. **Undeploy on worker first**: Master sends `Undeploy` RPC to the
   worker agent (timeout: 2m). The agent tears down the container,
   routes, DNS, and certs. This stops the backend, ensuring no traffic
   is served during edge cleanup.
2. **Remove edge route**: Master sends `RemoveEdgeRoute` to svc's agent.
   Svc removes the mc-proxy route and cleans up the cert.
3. **Master removes records** from placements and edge_routes tables.

Ordering rationale: undeploy the backend first so that if edge cleanup
fails, the service is already stopped and the edge route returns a
502 rather than serving stale content.

### Certificate Model

Two separate certs per public service — internal names never appear on
edge certs:

| Cert | Provisioned by | SAN | Used on |
|------|---------------|-----|---------|
| Internal | Worker agent → Metacrypt | `mcq.svc.mcp.metacircular.net` | Worker's mc-proxy |
| Public | Edge agent → Metacrypt | `mcq.metacircular.net` | Edge's mc-proxy |

Edge cert renewal is the edge agent's responsibility. The agent runs
the same `renewWindow` check as worker agents, renewing certs before
they expire (90-day TTL, renew at 30 days remaining).

---

## Snapshots

The master maintains periodic snapshots of every service's data.
Snapshots are the foundation for both migration and disaster recovery —
if a node dies, the master can restore a service to a new node from its
latest snapshot without the source node being alive.

All nodes have LUKS-encrypted disks. Snapshots are stored on the
master's encrypted disk, so service data is encrypted at rest at both
source and destination. An existing backup service on rift replicates
to external storage, covering the case where rift itself is lost.

### Snapshot Mechanism

The service definition declares how the agent should trigger a
consistent snapshot via the `method` field:

```toml
[snapshot]
method  = "grpc"                       # preferred for Metacircular services
exclude = ["layers/", "uploads/"]      # paths to skip (optional)
```

**Methods:**

| Method | How it works | Best for |
|--------|-------------|----------|
| `grpc` | Agent calls the standard `Snapshot` gRPC RPC on the service's gRPC port. The service vacuums databases and confirms. Agent then tars. | Metacircular services with gRPC servers |
| `cli` | Agent runs `podman exec <container> <service> snapshot` (the engineering standard's snapshot CLI command). Agent then tars. | Metacircular services without gRPC |
| `exec: <cmd>` | Agent runs `podman exec <container> <cmd>`. Agent then tars. | Non-standard services with custom backup scripts |
| `full` | Agent tars the entire `/srv/<service>/` directory, auto-vacuuming any `.db` files found. | Services that need everything backed up |
| *(omitted)* | Agent collects only `*.toml`, `*.db`, and `*.pem` files from `/srv/<service>/` — config, database, and certs. `.db` files are auto-vacuumed. | Default — covers the essentials without configuration |

The **default** (no `[snapshot]` section) captures the minimum needed
to restore a service: config, database, and TLS certs. This keeps
snapshot sizes small and predictable. Services that need more data
(e.g., file uploads, state directories) opt into `full` or specify
paths explicitly.

**`exclude`** works with any method. MCR uses `exclude` to skip layer
blobs (which can be rebuilt from git) while still capturing its
database and config.

**Database consistency:** For `grpc` and `cli` methods, the service
owns its own vacuum logic. For `full` and the default, the agent
detects `.db` files and runs `VACUUM INTO` to a temp copy before
including them in the tar. WAL and SHM files are excluded (the
vacuumed copy is self-contained).

### Standard Snapshot gRPC Service (mcdsl)

The `grpc` snapshot method uses a standard RPC that Metacircular
services implement via the `mcdsl/snapshot` package — same pattern as
`mcdsl/health`:

```protobuf
service SnapshotService {
  rpc Snapshot(SnapshotRequest) returns (SnapshotResponse);
}

message SnapshotRequest {}

message SnapshotResponse {
  bool   success = 1;
  string error   = 2;
  string path    = 3;  // path to the vacuumed backup (e.g. /srv/mcq/backups/...)
}
```

Services register the `SnapshotService` on their gRPC server. The
`mcdsl/snapshot` package provides a default implementation that reads
the database path from the service's config, runs `VACUUM INTO`, and
returns the backup path. Services with custom snapshot needs can
override the handler.

### Service Definition Examples

Metacircular service with gRPC (preferred):
```toml
[snapshot]
method = "grpc"
```

MCR (skip layer blobs):
```toml
[snapshot]
method = "grpc"
exclude = ["layers/", "uploads/"]
```

Non-Metacircular service with custom backup:
```toml
[snapshot]
method = "exec: /usr/local/bin/backup.sh"
```

Service with no snapshot config (default — captures *.toml, *.db, *.pem):
```toml
# No [snapshot] section needed
```

### Snapshot Storage

Snapshots are stored as flat files on the master node:

```
/srv/mcp-master/snapshots/
  mcq/
    2026-04-01T00:00:00Z.tar.zst
  mcias/
    2026-04-01T00:00:00Z.tar.zst
```

Format: tar.zst (tar archive with zstandard compression). One file per
snapshot, named by UTC timestamp.

### Snapshot Scheduling

The master runs a scheduled job that snapshots all services every 24
hours. The master iterates over all placements and for each one:

1. Acquires a per-service lock (skips if deploy/migrate/undeploy is
   in progress).
2. Sends `ExportServiceData(service_name)` to the hosting agent
   (timeout: 10m).
3. The agent runs the snapshot command (if configured), creates a
   tar.zst archive of `/srv/<service>/` (respecting excludes), and
   streams it back.
4. The master writes the archive to the snapshots directory.
5. The master prunes old snapshots (keep last N, configurable).

Scheduled snapshots are **live** — the service keeps running. Database
consistency is ensured by the vacuum step, not by stopping the
container. Migration snapshots use a different flow (stop first, then
tar) for perfect consistency.

**Agent fallback rule:** If `ExportServiceData` is called and the
container is not running (migration case), the agent skips the
configured snapshot method (`grpc`/`cli`/`exec`) and falls back to a
direct tar with auto-vacuum of `.db` files. This is correct because
the container already vacuumed on shutdown (SIGTERM handler).

For v2, the master always requests a full snapshot — no change
detection. Intelligence about dirty vs. clean services is future
optimization.

### Concurrency

The master holds a per-service lock for all operations that touch a
service (deploy, undeploy, migrate, snapshot). If a scheduled snapshot
overlaps with a deploy or migration, the snapshot waits. This prevents
capturing partial state during multi-step operations.

### Snapshot RPCs

```protobuf
// Service data export -- called by master on any agent.
// Authorization: mcp-master only.
rpc ExportServiceData(ExportServiceDataRequest)
    returns (stream DataChunk);

// Service data import -- called by master on any agent.
// Authorization: mcp-master only.
rpc ImportServiceData(stream ImportServiceDataChunk)
    returns (ImportServiceDataResponse);

message ExportServiceDataRequest {
  string service_name = 1;
  // Snapshot config is stored in the agent's registry at deploy time.
  // The agent uses its persisted config to determine the snapshot method
  // (grpc, cli, exec, full, default) and exclude patterns.
}

message DataChunk {
  bytes data = 1;
}

message ImportServiceDataChunk {
  // First message sets the service name; subsequent messages carry data.
  string service_name = 1;
  bytes  data         = 2;
  bool   force        = 3;  // overwrite existing /srv/<service>/ (first message only)
}

message ImportServiceDataResponse {
  int64 bytes_written = 1;
}
```

Note: `ExportServiceData`/`ImportServiceData` transfer full directory
archives. The existing `PushFile`/`PullFile` RPCs transfer individual
files and serve a different purpose (config distribution, cert
provisioning).

### Master Snapshot Config

```toml
[snapshots]
dir      = "/srv/mcp-master/snapshots"
interval = "24h"
retain   = 7    # keep last 7 snapshots per service
```

---

## Service Migration

Services can be migrated between nodes with `mcp migrate`. This is
essential for moving workloads off rift (which starts as both master
and worker) onto dedicated workers like orion as they come online.

Migration uses snapshots for data transfer. This means migration works
even if the source node is down (disaster recovery).

### Constraints

- **Core services cannot be migrated.** `tier = "core"` services are
  bound to the master node. Moving core services means designating a
  new master — a manual, deliberate operation outside the scope of
  `mcp migrate`.
- **Edge nodes are not migration targets.** Edge nodes run mc-proxy
  only, not application containers.

### Migration Flow

```
mcp migrate mcq --to orion
```

When the master receives `Migrate(mcq, orion)`:

1. **Validate**: Master verifies `orion` is a healthy worker. Rejects
   migration of `tier = "core"` services and migration to edge nodes.

2. **Stop on source** (if source is alive): Master sends `Stop` RPC
   to the source agent. The agent gracefully stops the container
   (SIGTERM). The service runs its shutdown handler, which vacuums
   databases per the engineering standard. If the source is down,
   skip this step.

3. **Snapshot** (if source is alive): Agent tars `/srv/<service>/`
   (now consistent — the service vacuumed on shutdown) and streams
   it to the master. If the source is down, the master uses the most
   recent stored snapshot.

4. **Push snapshot to destination**: Master streams the snapshot to
   the destination agent via `ImportServiceData`. The agent creates
   `/srv/<service>/` (with correct permissions) and extracts the
   archive.

5. **Deploy on destination**: Master sends `Deploy` RPC to the
   destination agent (orion). The agent deploys the container using
   the restored data. Provisions internal TLS cert and registers
   mc-proxy route on the new node.

6. **Update DNS**: Master updates the internal A record
   (`mcq.svc.mcp.metacircular.net`) to point to orion's Tailnet IP.

7. **Update edge route** (if public): Master sends `SetupEdgeRoute`
   to svc's agent with the updated backend. The edge agent updates
   the mc-proxy route. No new cert needed — the public hostname
   hasn't changed.

8. **Clean up source** (if source is alive): Master sends `Undeploy`
   to the source agent to remove the stopped container, old routes,
   old certs, and old DNS records.

9. **Update placement**: Master updates the `placements` table to
   reflect the new node. This step runs regardless of whether source
   cleanup succeeded.

### Disaster Recovery

If a node dies, the operator migrates its services to another node:

```
mcp migrate mcq --to orion          # source is down, uses latest snapshot
mcp migrate --all --from rift --to orion  # evacuate all services
```

The master detects the source is unreachable (unhealthy in node
registry), skips the stop and cleanup steps, and restores from
the stored snapshot. Data loss is bounded by the snapshot interval
(24 hours).

### Batch Migration

Full node evacuation for decommissioning or disaster recovery:

```
mcp migrate --all --from rift --to orion
```

The master migrates each service sequentially. Core services are
skipped (they cannot be migrated). The operator sees per-service
progress. If any migration fails, the master stops and reports which
service failed — the operator can fix the issue and resume with
`--all` (already-migrated services are skipped since they no longer
have placements on the source node).

### Migration Safety

- The source data is not deleted until step 8 (cleanup). If migration
  fails mid-transfer, the source still has the complete data and the
  operator can retry or roll back.
- The master rejects migration if the destination already has a
  `/srv/<service>/` directory (prevents accidental overwrite).
  Use `--force` to override.
- Downtime window: from stop (step 2) to the new container starting
  (step 5). For a personal platform this is acceptable.
- Migration snapshots use stop-then-tar for perfect consistency.
  Scheduled daily snapshots use live vacuum (no downtime).

### Migration Proto

```protobuf
rpc Migrate(MigrateRequest) returns (MigrateResponse);

message MigrateRequest {
  string service_name = 1;
  string target_node  = 2;
  bool   force        = 3;  // overwrite existing /srv/<service>/ on target
  bool   all          = 4;  // migrate all services from source
  string source_node  = 5;  // required when all=true
  // Validation: reject if all=true AND service_name is set (ambiguous).
}

message MigrateResponse {
  repeated StepResult results = 1;
}

// Note: CreateSnapshot/ListSnapshots are master CLI commands.
// The mcdsl SnapshotService.Snapshot RPC is a separate, service-level
// RPC called by agents on individual services.
rpc CreateSnapshot(CreateSnapshotRequest) returns (CreateSnapshotResponse);
rpc ListSnapshots(ListSnapshotsRequest) returns (ListSnapshotsResponse);

message CreateSnapshotRequest {
  string service_name = 1;
}

message CreateSnapshotResponse {
  string filename   = 1;
  int64  size_bytes = 2;
}

message ListSnapshotsRequest {
  string service_name = 1;
}

message ListSnapshotsResponse {
  repeated SnapshotInfo snapshots = 1;
}

message SnapshotInfo {
  string service_name = 1;
  string node         = 2;  // node the snapshot was taken from
  string filename     = 3;
  int64  size_bytes   = 4;
  string created_at   = 5;  // RFC3339
}
```

### CLI

```
mcp migrate <service> --to <node>              # migrate single service
mcp migrate <service> --to <node> --force      # overwrite existing data
mcp migrate --all --from <node> --to <node>    # evacuate all services
mcp snapshot <service>                         # take an on-demand snapshot
mcp snapshot list <service>                    # list available snapshots
```

---

## Agent Changes for v2

### New RPCs

See Proto Definitions section above for full message definitions.

- `HealthCheck` — called by master on missed heartbeats.
- `SetupEdgeRoute` — called by master on edge nodes.
- `RemoveEdgeRoute` — called by master on edge nodes.
- `ListEdgeRoutes` — called by master on edge nodes.

All new RPCs require the caller to be `mcp-master` (authorization check).

### Cert Provisioning on All Agents

All agents need Metacrypt configuration:

```toml
[metacrypt]
server_url = "https://metacrypt.svc.mcp.metacircular.net:8443"
ca_cert    = "/srv/mcp/certs/metacircular-ca.pem"
mount      = "pki"
issuer     = "infra"
token_path = "/srv/mcp/metacrypt-token"
```

Worker agents provision certs for internal hostnames. Edge agents
provision certs for public hostnames. Both use the same Metacrypt API
but with different identity-scoped policies.

### mc-proxy Management

The agent is the sole manager of mc-proxy routes via the gRPC admin API.
TOML config is not used for route management — this avoids the
database/config divergence problem from v1. mc-proxy's TOML config
only sets listener addresses and TLS defaults.

On mc-proxy restart, routes survive in mc-proxy's own SQLite database.
If mc-proxy's database is lost, the agent detects missing routes during
its monitoring cycle and re-registers them.

### Deploy Failure on Cert Error

If cert provisioning fails during deploy, the agent **must** fail the
deploy — do not register an mc-proxy route pointing to a nonexistent
cert. Return an error to the master, which reports it to the CLI. The
current v1 behavior (log warning, continue) is a bug.

---

## CLI Changes for v2

The CLI gains a `[master]` section and retains `[[nodes]]` for direct
access:

```toml
[master]
address = "100.x.x.x:9555"

# Retained for --direct mode (bypass master when it's down).
[[nodes]]
name    = "rift"
address = "100.95.252.120:9444"

[[nodes]]
name    = "svc"
address = "100.x.x.x:9444"

[mcias]
server_url   = "https://mcias.metacircular.net:8443"
service_name = "mcp"

[auth]
token_path = "/home/kyle/.config/mcp/token"

[services]
dir = "/home/kyle/.config/mcp/services"
```

By default, all commands go through the master. The `--direct` flag
bypasses the master and dials agents directly (v1 behavior):

```
mcp deploy mcq              # → master
mcp deploy mcq --direct -n rift  # → agent on rift (v1 mode)
mcp ps                      # → master aggregates all agents
mcp ps --direct             # → each agent individually (v1 mode)
```

`--direct` is the escape hatch when the master is down. In direct mode,
deploy requires an explicit `--node` flag (the CLI cannot auto-place
without the master).

### Sync Semantics

`mcp sync` is **declarative**: the service definitions on the operator's
workstation are the source of truth. The master converges the fleet:

- New definitions → deploy.
- Changed definitions → redeploy.
- Definitions present in the master's placement table but absent from
  the sync request → undeploy.

This makes the services directory a complete, auditable declaration of
what should be running. Use `mcp sync --dry-run` to preview what sync
would do without executing.

### Direct Mode Caveat

Services deployed via `--direct` (bypassing the master) are invisible
to the master — no placement record exists. Reconciliation detects
them as orphans. To bring a directly-deployed service under master
management, redeploy it through the master.

### New Commands

```
mcp edge list               # list all public edge routes
mcp edge status             # health of edge routes (cert expiry, backend reachable)
mcp node list               # fleet status from master
```

Service definition files remain on the operator's workstation. The CLI
pushes them to the master on `mcp deploy` and `mcp sync`.

---

## Agent Upgrades

The fleet is heterogeneous (NixOS + Debian, amd64 + arm64). NixOS flake
inputs don't work as a universal update mechanism.

MCP owns the binary at `/srv/mcp/mcp-agent` on all nodes.

```
mcp agent upgrade [node]    # cross-compile, SCP, restart via SSH
```

- CLI cross-compiles for the target's GOARCH.
- Copies via SCP to `/srv/mcp/mcp-agent.new`.
- Restarts via SSH. The restart command is OS-aware: `doas` on NixOS
  (rift, orion), `sudo` on Debian (svc). Configurable per node.
- Running containers survive the restart — rootless podman containers
  are independent of the agent process. `--restart unless-stopped` means
  podman handles liveness.
- The upgrade window (agent down for ~2s) only affects management
  operations. The master marks the agent as temporarily unhealthy until
  the next heartbeat.

All nodes: binary at `/srv/mcp/mcp-agent`, systemd unit
`mcp-agent.service`.

---

## Migration Plan

### Phase 1: Agent on svc

Deploy mcp-agent to svc (Debian):

- Create `mcp` user, install binary via SCP, configure systemd.
- Configure with Metacrypt access and mc-proxy gRPC socket access.
- Migrate existing mc-proxy TOML routes to agent-managed routes:
  export current routes from mc-proxy SQLite, import via agent
  `AddProxyRoute` RPCs.
- Verify with `mcp node list` (svc shows up).

### Phase 2: Edge routing RPCs

Implement `SetupEdgeRoute`, `RemoveEdgeRoute`, `ListEdgeRoutes` on the
agent. Test by calling directly from the CLI (temporary `mcp edge setup`
scaffolding command, removed after phase 3).

### Phase 3: Build mcp-master

Core coordination loop. Uses bootstrap `[[nodes]]` config for agent
addresses (dynamic registration comes in phase 4):

1. gRPC server with `McpMasterService`.
2. SQLite database for placements and edge routes.
3. Accept `Deploy` / `Undeploy` from CLI.
4. Place service on a node (tier / container-count).
5. Forward deploy to the correct agent.
6. Register DNS via MCNS.
7. Detect `public = true` routes, validate, call `SetupEdgeRoute`.
8. Return structured per-step results to CLI.

### Phase 4: Agent registration and health

- Agents self-register on startup (identity-bound).
- Heartbeat loop (30s interval, resource data).
- Master probe on missed heartbeats (90s threshold, 5s timeout).
- Fleet status aggregation for `mcp ps` and `mcp node list`.
- Reconciliation on master startup.
- Master transitions from bootstrap `[[nodes]]` to dynamic registry.

### Phase 5: Snapshots and migration

- Implement `ExportServiceData` / `ImportServiceData` on agents.
- Implement `mcdsl/snapshot` standard gRPC service.
- Add snapshot scheduling to master (24h cycle, retention pruning).
- Implement `CreateSnapshot`, `ListSnapshots`, `Migrate` on master.
- Add `mcp snapshot`, `mcp snapshot list`, `mcp migrate` CLI commands.
- Test migration between rift and orion.

### Phase 6: Cut over

- Update CLI config to add `[master]` section.
- Update service definitions with `tier` and `public` fields.
- Deploy agent to orion.
- Verify all services via `mcp ps` and public endpoint tests.
- Keep `[[nodes]]` config and `--direct` flag as escape hatch.

---

## Hostname Convention for Public Services

Services with public routes have two hostnames:

| Hostname | Purpose | Example |
|----------|---------|---------|
| `<svc>.metacircular.net` | Public — browser access, SSO login | `mcq.metacircular.net` |
| `<svc>.svc.mcp.metacircular.net` | Internal — API clients, service-to-service | `mcq.svc.mcp.metacircular.net` |

**SSO always uses the public hostname.** The service's `[sso].redirect_uri`
and the MCIAS SSO client registration both point to the public hostname
(e.g., `https://mcq.metacircular.net/sso/callback`). SSO state cookies
are bound to the domain they are set on, so the entire browser-based
login flow must stay on a single hostname.

**API clients use the internal hostname.** Service-to-service calls,
CLI tools, and MCP server communication authenticate with bearer tokens
(not SSO) and use the internal `.svc.mcp.` hostname. These do not
involve browser cookies and are unaffected by the SSO hostname
constraint.

This means:
- Human users bookmark `mcq.metacircular.net`, not the `.svc.mcp.` URL.
- The web UI's SSO "Sign in" button always initiates the flow on the
  public hostname.
- API endpoints on both hostnames accept the same bearer tokens —
  the hostname distinction is a routing and cookie concern, not an
  auth concern.

---

## Superseded Documents

`docs/edge-routing-design.md` is superseded by this document. It used
agent-to-agent communication, a single shared cert, private key
transmission over gRPC, and an `edge` field instead of `public`. None
of these design choices carried forward to v2.

---

## Open Questions

1. **Master HA**: mcp-master is a single point of failure. For v2, this
   is acceptable — the operator can use `--direct` to bypass the master.
   Future work could add master replication.

2. **Auto-reconciliation**: The master detects drift but does not
   auto-remediate. Future work could add automatic redeploy on drift.

## v2 Scope

v2 targets amd64 nodes only: rift (master+worker), orion (worker),
svc (edge). All images are single-arch amd64.

## Fast-Follow: arm64 Support

Immediate follow-up after v2 to onboard Raspberry Pi workers
(hyperborea and others):

1. **MCR manifest list support**: Accept and serve OCI image indexes
   (`application/vnd.oci.image.index.v1+json`) so a single tag
   references both amd64 and arm64 variants.
2. **`mcp build` multi-arch**: Build `linux/amd64` + `linux/arm64`
   images and push manifest lists to MCR.
3. **Onboard RPi workers**: Deploy agents, add to registration
   allowlist. Placement remains arch-agnostic — podman pulls the
   correct variant automatically.

## What v2 Does NOT Include

These remain future work beyond the arm64 fast-follow:

- Auto-reconciliation (master-driven redeploy on drift)
- Zero-downtime live migration (v2 migration stops the service)
- Web UI for fleet management
- Observability / log aggregation
- Object store
- Multiple edge nodes with load-based assignment
- Master replication / HA
- Resource-aware bin-packing (requires resource declarations in service defs)
