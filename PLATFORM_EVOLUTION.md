# Platform Evolution

This document captures the planned evolution of the Metacircular platform
from its current manually-wired state to fully declarative deployment.
It is a living design document — not a spec, not a commitment, but a
record of where we are, where we want to be, and what's between.

Last updated: 2026-03-28 (Phases A + B + C + D complete)

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
- Tagged releases: `v0.1.0`, `v1.0.0`, `v1.0.1`, `v1.1.0`, `v1.2.0`
- All consuming services import by URL with pinned versions
  (all consuming services on `v1.2.0`)
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

#### 3. MCP Agent: mc-proxy Route Registration — DONE

Agent connects to mc-proxy via Unix socket and automatically manages
routes during deploy and stop:
- Deploy: after container starts, calls `AddRoute` with hostname,
  backend (`127.0.0.1:<host_port>`), mode (l4/l7), and TLS cert paths
- Stop: calls `RemoveRoute` before stopping containers
- Config: `[mcproxy] socket` and `cert_dir` in agent config
- Nil-safe: if socket not configured, silently skipped (backward compatible)
- L7 routes: mc-proxy terminates TLS using certs at `<cert_dir>/<service>.pem`
- L4 routes: TLS passthrough, backend handles its own TLS
- Hostnames default to `<service>.svc.mcp.metacircular.net`

#### 4. MCP Agent: TLS Cert Provisioning — DONE

Agent provisions TLS certificates from Metacrypt CA automatically during
deploy for L7 routes:
- ACME client library requests certs from Metacrypt CA via its API
- Certs stored in `/srv/mc-proxy/certs/<service>.pem`
- Provisioning happens during deploy before mc-proxy route registration
- L7 routes get agent-provisioned certs; L4 routes use service-managed TLS

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

#### 6. MCP Agent: DNS Registration — DONE

Agent automatically manages DNS records during deploy and stop:
- Deploy: calls MCNS API to create/update A records for
  `<service>.svc.mcp.metacircular.net` pointing to the node's address.
- Stop/undeploy: removes DNS records before stopping containers.
- Config: `[mcns]` section in agent config with server URL, CA cert,
  token path, zone, and node address.
- Nil-safe: if MCNS not configured, silently skipped (backward compatible).
- Authorization: mcp-agent system account can manage any record name.

#### 7. Metacrypt: Automated Cert Issuance Policy — DONE

MCP agent has MCIAS credentials and Metacrypt policy for automated cert
issuance:
- MCP agent authenticates to Metacrypt with MCIAS service credentials
- Metacrypt policy allows cert issuance for
  `*.svc.mcp.metacircular.net`
- One cert per hostname per service — no wildcard certs

#### 8. MCNS: Record Management API — DONE

MCNS provides full CRUD for DNS records via REST and gRPC:
- REST: POST/GET/PUT/DELETE on `/v1/zones/{zone}/records`
- gRPC: RecordService with ListRecords, CreateRecord, GetRecord,
  UpdateRecord, DeleteRecord RPCs
- SQLite-backed with transactional writes, CNAME exclusivity enforcement,
  and automatic SOA serial bumping on mutations
- Authorization: admin can manage any record, mcp-agent system account
  can manage any record name, other system accounts scoped to own name
- MCP agent uses the REST API to register/deregister records on
  deploy/stop

#### 9. Application $PORT Convention — DONE

mcdsl v1.2.0 added `$PORT` and `$PORT_GRPC` env var support:
- `config.Load` checks `$PORT` → overrides `Server.ListenAddr`
- `config.Load` checks `$PORT_GRPC` → overrides `Server.GRPCAddr`
- Takes precedence over TOML and generic env overrides
  (`$MCR_SERVER_LISTEN_ADDR`) — agent-assigned ports are authoritative
- Handles both `config.Base` embedding (MCR, MCNS, MCAT) and direct
  `ServerConfig` embedding (Metacrypt) via struct tree walking
- All consuming services on mcdsl v1.4.0

---

## Suggested Sequencing

The dependencies form a rough order:

```
Phase A — Independent groundwork: ✓ COMPLETE
  #1  mcdsl proper module versioning ✓ DONE
  #2  MCP agent port assignment ✓ DONE
  #5  mc-proxy route persistence ✓ DONE
  #9  $PORT convention in applications ✓ DONE

Phase B — MCP route registration: ✓ COMPLETE
  #3  Agent registers routes with mc-proxy ✓ DONE

Phase C — Automated TLS: ✓ COMPLETE
  #7  Metacrypt cert issuance policy ✓ DONE
  #4  Agent provisions certs ✓ DONE
      (depends on #7)

Phase D — DNS: ✓ COMPLETE
  #8  MCNS record management API ✓ DONE
  #6  Agent registers DNS ✓ DONE
      (depends on #8)

Phase E — Multi-node agent management:
  #10 Agent binary at /srv/mcp/mcp-agent on all nodes
  #11 mcp agent upgrade (SSH-based cross-compiled push)
  #12 Node provisioning tooling (Debian + NixOS)
      (depends on #10)
```

**Phases A, B, C, and D are complete.** Services can be deployed with
agent-assigned ports, `$PORT` env vars, automatic mc-proxy route
registration, automated TLS cert provisioning from Metacrypt CA, and
automatic DNS registration in MCNS. No more manual port picking,
mcproxyctl, TOML editing, cert generation, or DNS zone editing.

### Immediate Next Steps

1. **Phase E: Multi-node agent management** — see below.
2. **mcdoc implementation** — fully designed, no platform evolution
   dependency. Deployable now with the new route system.

#### 10. Agent Binary Location Convention

**Gap**: The agent binary is currently NixOS-managed on rift (lives in
`/nix/store/`, systemd `ExecStart` points there). This doesn't work for
Debian nodes and requires a full `nixos-rebuild` for every MCP release.

**Work**:
- Standardize agent binary at `/srv/mcp/mcp-agent` on all nodes.
- NixOS config: change `ExecStart` from nix store path to
  `/srv/mcp/mcp-agent`. NixOS still owns user, systemd unit, podman,
  directories — just not the binary version.
- Debian nodes: same layout, provisioned by setup script.

#### 11. Agent Upgrade via SSH Push

**Gap**: Updating the agent requires manual, OS-specific steps. On
NixOS: update flake lock, commit, push, rebuild. On Debian: build, scp,
restart. With multiple nodes and architectures (amd64 + arm64), this
doesn't scale.

**Work**:
- `mcp agent upgrade [node]` CLI command.
- Cross-compiles agent for each target arch (`GOARCH` from node config).
- Uses `golang.org/x/crypto/ssh` to push the binary and restart the
  service. No external tool dependencies.
- Node config gains `ssh` (hostname) and `arch` (GOARCH) fields.
- Upgrades all nodes by default to prevent version skew. New RPCs cause
  `Unimplemented` errors if agent and CLI are out of sync.

**Depends on**: #10 (binary location convention).

#### 12. Node Provisioning Tooling

**Gap**: Setting up a new node requires manual steps: create user,
create directories, install podman, write config, create systemd unit.
Different for NixOS vs Debian.

**Work**:
- Go-based provisioning tool (part of MCP CLI) or standalone script.
- `mcp node provision <name>` SSHs to the node and runs setup:
  create `mcp` user with podman access, create `/srv/mcp/`, write
  systemd unit, install initial binary, start service.
- For NixOS, provisioning remains in the NixOS config (declarative).
  The provisioning tool targets Debian/generic Linux.

**Depends on**: #10 (binary location convention), #11 (SSH infra).

**Current fleet**:

| Node | OS | Arch | Status |
|------|----|------|--------|
| rift | NixOS | amd64 | Operational, single MCP agent |
| hyperborea | Debian (RPi) | arm64 | Online, needs agent provisioning |
| svc | Debian | amd64 | Runs MCIAS, needs agent for public edge services |

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
