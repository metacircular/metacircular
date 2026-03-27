# Packaging and Deploying to the Metacircular Platform

This guide provides everything needed to build, package, and deploy a
service to the Metacircular platform. It assumes no prior knowledge of
the platform's internals.

---

## Platform Overview

Metacircular is a multi-service infrastructure platform. Services are
Go binaries running as containers on Linux nodes, managed by these core
components:

| Component | Role |
|-----------|------|
| **MCP** (Control Plane) | Deploys, monitors, and manages container lifecycle via rootless Podman |
| **MCR** (Container Registry) | OCI container registry at `mcr.svc.mcp.metacircular.net:8443` |
| **mc-proxy** (TLS Proxy) | Routes traffic to services via L4 (SNI passthrough) or L7 (TLS termination) |
| **MCIAS** (Identity Service) | Central SSO/IAM — all services authenticate through it |
| **MCNS** (DNS) | Authoritative DNS for `*.svc.mcp.metacircular.net` |

The operator workflow is: **build image → push to MCR → write service
definition → deploy via MCP**. MCP handles port assignment, route
registration, and container lifecycle.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Go | 1.25+ |
| Container engine | Docker or Podman (for building images) |
| `mcp` CLI | Installed on the operator workstation |
| MCR access | Credentials to push images to `mcr.svc.mcp.metacircular.net:8443` |
| MCP agent | Running on the target node (currently `rift`) |
| MCIAS account | For `mcp` CLI authentication to the agent |

---

## 1. Build the Container Image

### Dockerfile Pattern

All services use a two-stage Alpine build. This is the standard
template:

```dockerfile
FROM golang:1.25-alpine AS builder

RUN apk add --no-cache git
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .

ARG VERSION=dev
RUN CGO_ENABLED=0 go build -trimpath \
    -ldflags="-s -w -X main.version=${VERSION}" \
    -o /<binary> ./cmd/<binary>

FROM alpine:3.21

RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /<binary> /usr/local/bin/<binary>

WORKDIR /srv/<service>
EXPOSE <ports>

ENTRYPOINT ["<binary>"]
CMD ["server", "--config", "/srv/<service>/<service>.toml"]
```

### Dockerfile Rules

- **`CGO_ENABLED=0`** — all builds are statically linked. No CGo in
  production.
- **`ca-certificates` and `tzdata`** — required in the runtime image
  for TLS verification and timezone-aware logging.
- **No `USER` directive** — containers run as `--user 0:0` under MCP's
  rootless Podman. UID 0 inside the container maps to the unprivileged
  `mcp` host user. A non-root `USER` directive creates a subordinate
  UID that cannot access host-mounted volumes.
- **No `VOLUME` directive** — causes layer unpacking failures under
  rootless Podman. The host volume mount is declared in the service
  definition, not the image.
- **No `adduser`/`addgroup`** — unnecessary given the rootless Podman
  model.
- **`WORKDIR /srv/<service>`** — so relative paths resolve correctly
  against the mounted data directory.
- **Version injection** — pass the git tag via `--build-arg VERSION=...`
  so the binary can report its version.
- **Stripped binaries** — `-trimpath -ldflags="-s -w"` removes debug
  symbols and build paths.

### Split Binaries

If the service has separate API and web UI binaries, create separate
Dockerfiles:

- `Dockerfile.api` — builds the API/gRPC server
- `Dockerfile.web` — builds the web UI server

Both follow the same template. The web binary communicates with the API
server via gRPC (no direct database access).

### Makefile Target

Every service includes a `make docker` target:

```makefile
docker:
	docker build --build-arg VERSION=$(shell git describe --tags --always --dirty) \
	    -t <service> -f Dockerfile.api .
```

---

## 2. Write a Service Definition

Service definitions are TOML files that tell MCP what to deploy. They
live at `~/.config/mcp/services/<service>.toml` on the operator
workstation.

### Minimal Example (Single Component)

```toml
name = "myservice"
node = "rift"
version = "v1.0.0"

[build.images]
myservice = "Dockerfile"

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
```

### Full Example (API + Web)

```toml
name = "myservice"
node = "rift"
version = "v1.0.0"

[build.images]
myservice = "Dockerfile.api"
myservice-web = "Dockerfile.web"

[[components]]
name = "api"
volumes = ["/srv/myservice:/srv/myservice"]
cmd = ["server", "--config", "/srv/myservice/myservice.toml"]

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
volumes = ["/srv/myservice:/srv/myservice"]
cmd = ["server", "--config", "/srv/myservice/myservice.toml"]

[[components.routes]]
port = 443
mode = "l7"
```

### Convention-Derived Defaults

Most fields are optional — MCP derives them from conventions:

| Field | Default | Override when... |
|-------|---------|------------------|
| Image name | `<service>` (api), `<service>-<component>` (others) | Image name differs from convention |
| Image registry | `mcr.svc.mcp.metacircular.net:8443` (from global MCP config) | Never — always use MCR |
| Version | Service-level `version` field | A component needs a different version |
| Volumes | `/srv/<service>:/srv/<service>` | Additional mounts are needed |
| Network | `mcpnet` | Service needs host networking or a different network |
| User | `0:0` | Never change this for standard services |
| Restart | `unless-stopped` | Service should not auto-restart |
| Source path | `<service>` relative to workspace root | Directory name differs from service name |
| Hostname | `<service>.svc.mcp.metacircular.net` | Service needs a public hostname |

### Service Definition Reference

**Top-level fields:**

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Service name (matches project name) |
| `node` | Yes | Target node to deploy to |
| `version` | Yes | Image version tag (semver, e.g. `v1.0.0`) |
| `active` | No | Whether MCP keeps this running (default: `true`) |
| `path` | No | Source directory relative to workspace (default: `name`) |

**Build fields:**

| Field | Purpose |
|-------|---------|
| `build.images.<name>` | Maps image name to Dockerfile path |

**Component fields:**

| Field | Purpose |
|-------|---------|
| `name` | Component name (e.g. `api`, `web`) |
| `image` | Full image reference override |
| `version` | Version override for this component |
| `volumes` | Volume mounts (list of `host:container` strings) |
| `cmd` | Command override (list of strings) |
| `network` | Container network override |
| `user` | Container user override |
| `restart` | Restart policy override |

**Route fields (under `[[components.routes]]`):**

| Field | Purpose |
|-------|---------|
| `name` | Route name — determines `$PORT_<NAME>` env var |
| `port` | External port on mc-proxy (e.g. `8443`, `9443`, `443`) |
| `mode` | `l4` (TLS passthrough) or `l7` (TLS termination by mc-proxy) |
| `hostname` | Public hostname override |

### Routing Modes

| Mode | TLS handled by | Use when... |
|------|----------------|-------------|
| `l4` | The service itself | Service manages its own TLS (API servers, gRPC) |
| `l7` | mc-proxy | mc-proxy terminates TLS and proxies HTTP to the service (web UIs) |

### Version Pinning

Service definitions **must** pin an explicit semver tag (e.g. `v1.1.0`).
Never use `:latest`. This ensures deployments are reproducible and
`mcp status` shows the actual running version.

---

## 3. Build, Push, and Deploy

### Tag the Release

```bash
git tag -a v1.0.0 -m "v1.0.0"
git push origin v1.0.0
```

### Build and Push Images

```bash
mcp build <service>
```

This reads the `[build.images]` section of the service definition,
builds each Dockerfile, tags the images with the version from the
definition, and pushes them to MCR.

The workspace root is configured in `~/.config/mcp/mcp.toml`:

```toml
[build]
workspace = "~/src/metacircular"
```

Each service's source is at `<workspace>/<path>` (where `path` defaults
to the service name).

### Sync and Deploy

```bash
# Push all service definitions to agents, auto-build missing images
mcp sync

# Deploy (or redeploy) a specific service
mcp deploy <service>
```

`mcp sync` checks whether each component's image tag exists in MCR. If
missing and the source tree is available, it builds and pushes
automatically.

`mcp deploy` pulls the image on the target node and creates or
recreates the containers.

### What Happens During Deploy

1. Agent assigns a free host port (10000–60000) for each declared route.
2. Agent starts containers with `$PORT` / `$PORT_<NAME>` environment
   variables set to the assigned ports.
3. Agent registers routes with mc-proxy (hostname → `127.0.0.1:<port>`,
   mode, TLS cert paths).
4. Agent records the full state in its SQLite registry.

On stop (`mcp stop <service>`), the agent reverses the process: removes
mc-proxy routes, then stops containers.

---

## 4. Data Directory Convention

All runtime data lives in `/srv/<service>/` on the host. This directory
is bind-mounted into the container.

```
/srv/<service>/
├── <service>.toml        # Configuration file
├── <service>.db          # SQLite database (created on first run)
├── certs/                # TLS certificates
│   ├── cert.pem
│   └── key.pem
└── backups/              # Database snapshots
```

This directory must exist on the target node before the first deploy,
owned by the `mcp` user (which runs rootless Podman). Create it with:

```bash
sudo mkdir -p /srv/<service>/certs
sudo chown -R mcp:mcp /srv/<service>
```

Place the service's TOML configuration and TLS certificates here before
deploying.

---

## 5. Configuration

Services use TOML configuration with environment variable overrides.

### Standard Config Sections

```toml
[server]
listen_addr = ":8443"
grpc_addr   = ":9443"
tls_cert    = "/srv/<service>/certs/cert.pem"
tls_key     = "/srv/<service>/certs/key.pem"

[database]
path = "/srv/<service>/<service>.db"

[mcias]
server_url   = "https://mcias.metacircular.net:8443"
ca_cert      = ""
service_name = "<service>"
tags         = []

[log]
level = "info"
```

For services with a web UI, add:

```toml
[web]
listen_addr  = "127.0.0.1:8080"
vault_grpc   = "127.0.0.1:9443"
vault_ca_cert = ""
```

### $PORT Convention

When deployed via MCP, the agent assigns host ports and passes them as
environment variables. **Applications should not hardcode listen
addresses** — they will be overridden at deploy time.

| Env var | When set |
|---------|----------|
| `$PORT` | Component has a single route |
| `$PORT_<NAME>` | Component has multiple named routes |

Route names are uppercased: `name = "rest"` → `$PORT_REST`,
`name = "grpc"` → `$PORT_GRPC`.

Services built with **mcdsl v1.1.0+** handle this automatically —
`config.Load` checks `$PORT` → overrides `Server.ListenAddr`, and
`$PORT_GRPC` → overrides `Server.GRPCAddr`. These take precedence over
TOML values.

Services not using mcdsl must check these environment variables in their
own config loading.

### Environment Variable Overrides

Beyond `$PORT`, services support `$SERVICENAME_SECTION_KEY` overrides.
For example, `$MCR_SERVER_LISTEN_ADDR=:9999` overrides
`[server] listen_addr` in MCR's config. `$PORT` takes precedence over
these.

---

## 6. Authentication (MCIAS Integration)

Every service delegates authentication to MCIAS. No service maintains
its own user database.

### Auth Flow

1. Client sends credentials to the service's `POST /v1/auth/login`.
2. Service forwards them to MCIAS via the client library
   (`git.wntrmute.dev/mc/mcias/clients/go`).
3. MCIAS validates and returns a bearer token.
4. Subsequent requests include `Authorization: Bearer <token>`.
5. Service validates tokens via MCIAS `ValidateToken()`, cached for 30s
   (keyed by SHA-256 of the token).

### Roles

| Role | Access |
|------|--------|
| `admin` | Full access, policy bypass |
| `user` | Access governed by policy rules, default deny |
| `guest` | Service-dependent restrictions, default deny |

Admin detection comes solely from the MCIAS `admin` role. Services
never promote users locally.

---

## 7. Networking

### Hostnames

Every service gets `<service>.svc.mcp.metacircular.net` automatically.
Public-facing services can declare additional hostnames:

```toml
[[components.routes]]
port = 443
mode = "l7"
hostname = "docs.metacircular.net"
```

### TLS

- **Minimum TLS 1.3.** No exceptions.
- L4 services manage their own TLS — certificates go in
  `/srv/<service>/certs/`.
- L7 services have TLS terminated by mc-proxy — certs are stored at
  `/srv/mc-proxy/certs/<service>.pem`.
- Certificate and key paths are required config — the service refuses
  to start without them.

### Container Networking

Containers join the `mcpnet` Podman network by default. Services
communicate with each other over this network or via loopback (when
co-located on the same node).

---

## 8. Command Reference

| Command | Purpose |
|---------|---------|
| `mcp build <service>` | Build and push images to MCR |
| `mcp sync` | Push all service definitions to agents; auto-build missing images |
| `mcp deploy <service>` | Pull image, (re)create containers, register routes |
| `mcp stop <service>` | Remove routes, stop containers |
| `mcp start <service>` | Start previously stopped containers |
| `mcp restart <service>` | Restart containers in place |
| `mcp ps` | List all managed containers and status |
| `mcp status [service]` | Detailed status for a specific service |

---

## 9. Complete Walkthrough

Deploying a new service called `myservice` from scratch:

```bash
# 1. Prepare the target node
ssh rift
sudo mkdir -p /srv/myservice/certs
sudo chown -R mcp:mcp /srv/myservice
# Place myservice.toml and TLS certs in /srv/myservice/
exit

# 2. Tag the release
cd ~/src/metacircular/myservice
git tag -a v1.0.0 -m "v1.0.0"
git push origin v1.0.0

# 3. Write the service definition
cat > ~/.config/mcp/services/myservice.toml << 'EOF'
name = "myservice"
node = "rift"
version = "v1.0.0"

[build.images]
myservice = "Dockerfile.api"

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
EOF

# 4. Build and push the image
mcp build myservice

# 5. Deploy
mcp deploy myservice

# 6. Verify
mcp status myservice
mcp ps
```

The service is now running, with mc-proxy routing
`myservice.svc.mcp.metacircular.net` traffic to the agent-assigned
ports.

---

## Appendix: Repository Layout

Services follow a standard directory structure:

```
.
├── cmd/<service>/          CLI entry point (server, subcommands)
├── cmd/<service>-web/      Web UI entry point (if separate)
├── internal/               All service logic (not importable externally)
│   ├── auth/               MCIAS integration
│   ├── config/             TOML config loading
│   ├── db/                 Database setup, migrations
│   ├── server/             REST API server
│   ├── grpcserver/         gRPC server
│   └── webserver/          Web UI server (if applicable)
├── proto/<service>/v1/     Protobuf definitions
├── gen/<service>/v1/       Generated gRPC code
├── web/                    Templates and static assets (embedded)
├── deploy/
│   ├── <service>-rift.toml Reference MCP service definition
│   ├── docker/             Docker Compose files
│   ├── examples/           Example config files
│   └── systemd/            systemd units
├── Dockerfile.api          API server container
├── Dockerfile.web          Web UI container (if applicable)
├── Makefile                Standard build targets
└── <service>.toml.example  Example configuration
```

### Standard Makefile Targets

| Target | Purpose |
|--------|---------|
| `make all` | vet → lint → test → build (the CI pipeline) |
| `make build` | `go build ./...` |
| `make test` | `go test ./...` |
| `make vet` | `go vet ./...` |
| `make lint` | `golangci-lint run ./...` |
| `make docker` | Build the container image |
| `make proto` | Regenerate gRPC code from .proto files |
| `make devserver` | Build and run locally against `srv/` config |

---

## Appendix: Currently Deployed Services

For reference, these services are operational on the platform:

| Service | Version | Node | Purpose |
|---------|---------|------|---------|
| MCIAS | v1.8.0 | (separate) | Identity and access |
| Metacrypt | v1.1.0 | rift | Cryptographic service, PKI/CA |
| MC-Proxy | v1.1.0 | rift | TLS proxy and router |
| MCR | v1.2.0 | rift | Container registry |
| MCNS | v1.1.0 | rift | Authoritative DNS |
| MCP | v0.3.0 | rift | Control plane agent |
