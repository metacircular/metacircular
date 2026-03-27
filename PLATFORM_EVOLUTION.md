# Platform Evolution

This document captures the planned evolution of the Metacircular platform
from its current manually-wired state to fully declarative deployment.
It is a living design document — not a spec, not a commitment, but a
record of where we are, where we want to be, and what's between.

Last updated: 2026-03-27 (Phase A complete)

---

## Current State

The platform works. Services run on rift, fronted by mc-proxy, with
MCIAS handling auth and Metacrypt managing secrets. MCP can deploy,
stop, start, restart, and monitor containers. This is not nothing — the
core infrastructure is real and operational.

But the wiring between services is manual:

- **Port assignment**: operators pick host ports by hand and record them
  in service definitions (`ports = ["127.0.0.1:28443:8443"]`). A mental
  register of "what port is free" is required.
- **mc-proxy routing**: routes are defined in a static TOML config file.
  Adding a service means editing `mc-proxy-rift.toml`, restarting
  mc-proxy, and hoping you didn't typo a port number.
- **TLS certificates**: provisioned manually. Certs are generated,
  placed in `/srv/mc-proxy/certs/`, and referenced by path in the
  mc-proxy config.
- **DNS**: records are manually configured in MCNS zone files.
- **Container config boilerplate**: operators specify `network`, `user`,
  `restart`, full image URLs, and port mappings per component, even
  though these are almost always the same values.
- **mcdsl build wiring**: the shared library requires `replace`
  directives or sibling directory tricks in Docker builds. It should
  be a normally-versioned Go module fetched by the toolchain.

Each new service requires touching 4-5 files across 3-4 repos. The
process works but doesn't scale and is error-prone.

## Target State

The operator writes a service definition that declares **what** they
want, not **how** to wire it:

```toml
name = "metacrypt"
node = "rift"
version = "v1.0.0"

[build.images]
metacrypt = "Dockerfile.api"
metacrypt-web = "Dockerfile.web"

[[components]]
name = "api"

[[components.routes]]
name = "rest"
port = 8443
mode = "l4"

[[components.routes]]
name = "grpc"
port = 9443
mode = "l4"

[[components]]
name = "web"

[[components.routes]]
port = 443
mode = "l7"
```

Everything else is derived from conventions:

- **Image name**: `<service>` for the first/api component,
  `<service>-<component>` for others. Resolved against the registry
  URL from global MCP config (`~/.config/mcp/mcp.toml`).
- **Version**: the service-level `version` field applies to all
  components. Can be overridden per-component when needed.
- **Volumes**: `/srv/<service>:/srv/<service>` is the agent default.
  Only declare additional mounts.
- **Network, user, restart**: agent defaults (`mcpnet`, `0:0`,
  `unless-stopped`). Override only when needed.
- **Source path**: defaults to `<service>` relative to the workspace
  root. Override with `path` if different.

`mcp deploy metacrypt` does the rest:

1. Agent assigns a free host port per route (random, check
   availability, retry on collision).
2. Agent requests TLS certs from Metacrypt CA for
   `metacrypt.svc.mcp.metacircular.net`.
3. Agent registers routes with mc-proxy via gRPC (mc-proxy persists
   them in SQLite).
4. Agent creates/updates DNS records in MCNS for
   `metacrypt.svc.mcp.metacircular.net`.
5. Agent starts containers with `$PORT_REST`, `$PORT_GRPC`, `$PORT_WEB`
   environment variables set to the assigned host ports.
6. Agent records the full state (port assignments, cert paths, route
   IDs) in its registry.

On teardown (`mcp stop`), the agent reverses the process: stops
containers, removes mc-proxy routes, cleans up DNS records.

### Port Environment Variables

Applications receive their assigned ports via environment variables:

| Components with... | Env var | Example |
|--------------------|---------|---------|
| Single route | `$PORT` | `$PORT=8913` |
| Multiple routes | `$PORT_<NAME>` | `$PORT_REST=8913`, `$PORT_GRPC=9217` |

Route names come from the `name` field in `[[components.routes]]`.
Applications read these in their config layer alongside existing env
overrides (e.g., `$MCR_SERVER_LISTEN_ADDR`).

### Hostname Convention

Every service gets `<service>.svc.mcp.metacircular.net` automatically.
Public-facing services can additionally declare external hostnames:

```toml
[[components.routes]]
name = "web"
port = 443
mode = "l7"
hostname = "docs.metacircular.net"  # optional, public DNS
```

If `hostname` is omitted, the route uses the default
`<service>.svc.mcp.metacircular.net`.

### Multi-Node Considerations

This design targets single-node (rift) but should not prevent
multi-node operation. Key design decisions that keep the door open:

- **Port assignment is per-agent.** Each node's agent manages its own
  port space. No cross-node coordination needed.
- **Route registration uses the node's address, not `127.0.0.1`.**
  When mc-proxy and the service are on the same host, the backend is
  loopback. When they're on different hosts, the backend is the node's
  network address. The agent registers the appropriate address for its
  node. The mc-proxy route API already accepts arbitrary backend
  addresses.
- **DNS can have multiple A records.** MCNS can return multiple records
  for the same hostname (one per node) for simple load distribution.
- **The CLI routes to the correct agent via the `node` field.** Adding
  a second node is `mcp node add orion <address>` and then services
  can target `node = "orion"`.

Nothing in the single-node implementation should hardcode assumptions
about one node, one mc-proxy, or loopback-only backends.

---

## Gap Analysis

### What exists today and works

| Capability | Status |
|------------|--------|
| MCP CLI + agent deploy/stop/start/restart | Working |
| MCP sync (push service definitions to agent) | Working |
| MCP status/monitoring/drift detection | Working |
| mc-proxy L4/L7 routing | Working |
| mc-proxy gRPC admin API | Working |
| MCIAS auth for all services | Working |
| Metacrypt CA (PKI engine) | Working |
| MCNS DNS serving | Working |
| MCR container registry | Working |
| Service definitions in ~/.config/mcp/services/ | Working |
| Image build pipeline (being folded into MCP) | Working |

### What needs to change

#### 1. mcdsl: Proper Module Versioning — DONE

mcdsl is already properly versioned and released:
- Tagged releases: `v0.1.0`, `v1.0.0`, `v1.0.1`
- All consuming services import by URL with pinned versions
  (mcr, mcat, mcns, mc-proxy → `v1.0.0`; metacrypt → `v1.0.1`)
- No `replace` directives anywhere
- Docker builds use standard `go mod download`
- `uses_mcdsl` eliminated from service definitions and docs

#### 2. MCP Agent: Port Assignment — DONE

Agent allocates host ports automatically at deploy time:
- Service definitions declare `[[components.routes]]` with name, port,
  mode, and optional hostname
- Agent picks random free ports (10000-60000, availability check,
  mutex-serialized), records assignments in `component_routes` table
- Containers receive `$PORT` / `$PORT_<NAME>` env vars
- Backward compatible: old-style `ports` strings still work unchanged
- Proto: `RouteSpec` message, `routes` + `env` fields on `ComponentSpec`
- Servicedef: `RouteDef` parsing and validation from TOML
- Registry: `component_routes` table with `host_port` tracking
- Runtime: `Env` field on `ContainerSpec`, `-e` flag generation

#### 3. MCP Agent: mc-proxy Route Registration

**Gap**: mc-proxy routes are static TOML. The gRPC admin API exists but
MCP doesn't use it.

**Work**:
- Agent calls mc-proxy gRPC API to register/remove routes on
  deploy/stop.
- Route registration includes: hostname, backend address (node address
  + assigned port), mode (l4/l7), TLS cert paths.

**Depends on**: port assignment (#2), mc-proxy route persistence (#5).

#### 4. MCP Agent: TLS Cert Provisioning

**Gap**: certs are manually provisioned and placed on disk. There is no
automated issuance flow.

**Work**:
- Agent requests certs from Metacrypt CA via its API.
- Certs are stored in a standard location
  (`/srv/mc-proxy/certs/<service>.pem`).
- Cert renewal is handled automatically before expiry.

**Depends on**: Metacrypt cert issuance policy (#7).

#### 5. mc-proxy: Route Persistence — DONE

mc-proxy routes are fully persisted in SQLite and survive restarts:
- SQLite `routes` table stores all listener and route state
- TOML config seeds the database on first run only (via
  `store.IsEmpty()` + `store.Seed()`); subsequent starts load from
  DB (`store.ListListeners()` + `store.ListRoutes()`)
- gRPC admin API (`AddRoute`/`RemoveRoute`) writes through to both
  DB and in-memory state
- `mcproxyctl` CLI provides full route management (add, remove, list)
- Routes added via gRPC survive mc-proxy restart
- TOML route config is vestigial — kept only for mc-proxy's own
  bootstrap before MCP is operational. The gRPC API and mcproxyctl
  are the primary route management interfaces going forward.

#### 6. MCP Agent: DNS Registration

**Gap**: DNS records are manually configured in MCNS zone files.

**Work**:
- Agent creates/updates A records in MCNS for
  `<service>.svc.mcp.metacircular.net`.
- Agent removes records on service teardown.

**Depends on**: MCNS record management API (#8).

#### 7. Metacrypt: Automated Cert Issuance Policy

**Gap**: no policy exists for automated cert issuance. The MCP agent
doesn't have a Metacrypt identity or permissions.

**Work**:
- MCP agent gets an MCIAS service account.
- Metacrypt policy allows this account to issue certs scoped to
  `*.svc.mcp.metacircular.net` (and explicitly listed public
  hostnames).
- No wildcard certs — one cert per hostname per service.

**Depends on**: MCIAS service account provisioning (exists today, just
needs the account created).

#### 8. MCNS: Record Management API

**Gap**: MCNS is a CoreDNS precursor serving static zone files. There
is no API for dynamic record management.

**Work**:
- MCNS needs an API (REST + gRPC per platform convention) for
  creating/updating/deleting DNS records.
- Records are stored in SQLite (replacing or supplementing zone files).
- MCIAS auth, scoped to allow MCP agent to manage
  `*.svc.mcp.metacircular.net` records.

**Depends on**: this is the largest gap. MCNS is currently a CoreDNS
wrapper, not a full service. This may be the right time to build the
real MCNS.

#### 9. Application $PORT Convention — DONE

mcdsl v1.1.0 adds `$PORT` and `$PORT_GRPC` env var support:
- `config.Load` checks `$PORT` → overrides `Server.ListenAddr`
- `config.Load` checks `$PORT_GRPC` → overrides `Server.GRPCAddr`
- Takes precedence over TOML and generic env overrides
  (`$MCR_SERVER_LISTEN_ADDR`) — agent-assigned ports are authoritative
- Handles both `config.Base` embedding (MCR, MCNS, MCAT) and direct
  `ServerConfig` embedding (Metacrypt) via struct tree walking
- MCR, Metacrypt, MCNS upgraded to mcdsl v1.1.0

---

## Suggested Sequencing

The dependencies form a rough order:

```
Phase A — Independent groundwork: ✓ COMPLETE
  #1  mcdsl proper module versioning ✓ DONE
  #2  MCP agent port assignment ✓ DONE
  #5  mc-proxy route persistence ✓ DONE
  #9  $PORT convention in applications ✓ DONE

Phase B — MCP route registration: ← IN PROGRESS
  #3  Agent registers routes with mc-proxy
      (depends on #2 + #5 — both done)

Phase C — Automated TLS:
  #7  Metacrypt cert issuance policy
  #4  Agent provisions certs
      (depends on #7)

Phase D — DNS:
  #8  MCNS record management API
  #6  Agent registers DNS
      (depends on #8)
```

**Phase A is complete.** Services can be deployed with agent-assigned
ports and `$PORT` env vars. mc-proxy routes must still be registered
manually via `mcproxyctl` or the gRPC API.

**Phase B is next.** The agent will call mc-proxy's gRPC API to
register/remove routes automatically during deploy/stop. This
eliminates the last manual networking step — no more `mcproxyctl`
or TOML editing.

After Phase C, cert provisioning is automatic. After Phase D,
`mcp deploy` is fully declarative.

### Immediate Next Steps

1. **Phase B: MCP agent route registration (#3)** — agent connects to
   mc-proxy via Unix socket, registers routes on deploy, removes on
   stop. TLS certs are pre-provisioned (Phase C automates this later).
2. **mcdoc implementation** — fully designed, no platform evolution
   dependency. Deployable with a manually assigned port today.

---

## Open Questions

- **Cert rotation**: when a Metacrypt-issued cert expires, does the
  agent renew it automatically? What's the renewal window? Does mc-proxy
  need to reload certs without restart?
- **Public hostnames**: services like mcdoc want `docs.metacircular.net`
  in addition to the `.svc.mcp.metacircular.net` name. Public DNS is
  managed outside MCNS (Cloudflare? registrar?). How does the agent
  handle the split between internal and external DNS?
- **mc-proxy bootstrap**: mc-proxy itself is a service that needs to be
  running before other services can be routed. Its own routes (if any)
  may need to be self-configured or seeded from a minimal static config
  at first start. Once operational, all route management goes through
  the gRPC API.
- **Rollback**: if cert provisioning fails mid-deploy, does the agent
  roll back the port assignment and mc-proxy route? What's the failure
  mode — partial deploy, full rollback, or best-effort?
- **Service discovery between components**: currently, components find
  each other via config (e.g., mcr-web knows mcr-api's gRPC address).
  With agent-assigned ports, components within a service need to
  discover each other's ports. The agent could set additional env vars
  (`$PEER_API_GRPC=127.0.0.1:9217`) or services could query the agent.
