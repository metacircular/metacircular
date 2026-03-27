# Metacircular Dynamics — Engineering Standards

Source: https://metacircular.net/roam/20260314210051-metacircular_dynamics.html

This document describes the standard repository layout, tooling, and software
development lifecycle (SDLC) for services built at Metacircular Dynamics. It
incorporates the platform-wide project guidelines and codifies the conventions
established in Metacrypt as the baseline for all services.

## Platform Rules

These four rules apply to every Metacircular service:

1. **Data Storage**: All service data goes in `/srv/<service>/` to enable
   straightforward migration across systems.
2. **Deployment Architecture**: Services require systemd unit files but
   prioritize container-first design to support deployment via the
   Metacircular Control Plane (MCP).
3. **Identity Management**: Services must integrate with MCIAS (Metacircular
   Identity and Access Service) for user management and access control. Three
   role levels: `admin` (full administrative access), `user` (full
   non-administrative access), `guest` (service-dependent restrictions).
4. **API Design**: Services expose both gRPC and REST interfaces, kept in
   sync. Web UIs are built with htmx.

## Table of Contents

0. [Platform Rules](#platform-rules)
1. [Repository Layout](#repository-layout)
2. [Language & Toolchain](#language--toolchain)
3. [Build System](#build-system)
4. [API Design](#api-design)
5. [Authentication & Authorization](#authentication--authorization)
6. [Database Conventions](#database-conventions)
7. [Configuration](#configuration)
8. [Web UI](#web-ui)
9. [Testing](#testing)
10. [Linting & Static Analysis](#linting--static-analysis)
11. [Deployment](#deployment)
12. [Documentation](#documentation)
13. [Security](#security)
14. [Development Workflow](#development-workflow)

---

## Repository Layout

Every service follows a consistent directory structure. Adjust the
service-specific directories (e.g. `engines/` in Metacrypt) as appropriate,
but the top-level skeleton is fixed.

```
.
├── cmd/
│   ├── <service>/              CLI entry point (server, subcommands)
│   └── <service>-web/          Web UI entry point (if separate binary)
├── internal/
│   ├── auth/                   MCIAS integration (token validation, caching)
│   ├── config/                 TOML configuration loading & validation
│   ├── db/                     Database setup, schema migrations
│   ├── server/                 REST API server, routes, middleware
│   ├── grpcserver/             gRPC server, interceptors, service handlers
│   ├── webserver/              Web UI server, template routes, HTMX handlers
│   └── <domain>/               Service-specific packages
├── proto/<service>/
│   └── v<N>/                   Current proto definitions (start at v1;
│                               increment only on breaking changes)
├── gen/<service>/
│   └── v<N>/                   Generated Go gRPC/protobuf code
├── web/
│   ├── embed.go                //go:embed directive for templates and static
│   ├── templates/              Go HTML templates
│   └── static/                 CSS, JS (htmx)
├── deploy/
│   ├── <service>-rift.toml     MCP service definition (reference)
│   ├── docker/                 Docker Compose configuration
│   ├── examples/               Example config files
│   ├── scripts/                Install, backup, migration scripts
│   └── systemd/                systemd unit files and timers
├── docs/                       Internal engineering documentation
├── Dockerfile.api              API server container (if split binary)
├── Dockerfile.web              Web UI container (if split binary)
├── Makefile
├── buf.yaml                    Protobuf linting & breaking-change config
├── .golangci.yaml              Linter configuration
├── .gitignore
├── CLAUDE.md                   AI-assisted development instructions
├── ARCHITECTURE.md             Full system specification
└── <service>.toml.example      Example configuration
```

### Key Principles

- **`cmd/`** contains only CLI wiring (cobra commands, flag parsing). No
  business logic.
- **`internal/`** contains all service logic. Nothing in `internal/` is
  importable by other modules — this is enforced by Go's module system.
- **`proto/`** is the source of truth for gRPC definitions. Generated code
  lives in `gen/`, never edited by hand. Versions start at `v1`; a new
  version directory is only created when a breaking change is required — not
  as a naming convention or initial setup step.
- **`deploy/`** contains everything needed to run the service in production.
  A new engineer should be able to deploy from this directory alone.
- **`web/`** is embedded into the binary via `//go:embed`. No external file
  dependencies at runtime.

### What Does Not Belong in the Repository

- Runtime data (databases, certificates, logs) — these live in `/srv/<service>`
- Real configuration files with secrets — only examples are committed
- IDE configuration (`.idea/`, `.vscode/`) — per-developer, not shared
- Vendored dependencies — Go module proxy handles this

---

## Language & Toolchain

| Tool | Version | Purpose |
|------|---------|---------|
| Go | 1.25+ | Primary language |
| protoc + protoc-gen-go | Latest | Protobuf/gRPC code generation |
| buf | Latest | Proto linting and breaking-change detection |
| golangci-lint | v2 | Static analysis and linting |
| Docker | Latest | Container builds |

### Go Conventions

- **Pure-Go dependencies** where possible. Avoid CGo — it complicates
  cross-compilation and container builds. Use `modernc.org/sqlite` instead
  of `mattn/go-sqlite3`.
- **`CGO_ENABLED=0`** for all production builds. Statically linked binaries
  deploy cleanly to Alpine containers.
- **Stripped binaries**: Build with `-trimpath -ldflags="-s -w"` to remove
  debug symbols and reduce image size.
- **Version injection**: Pass `git describe --tags --always --dirty` via
  `-X main.version=...` at build time. Every binary must report its version.

### Module Path

Services hosted on `git.wntrmute.dev` use:

```
git.wntrmute.dev/kyle/<service>
```

---

## Build System

Every repository has a Makefile with these standard targets:

```makefile
.PHONY: build test vet lint proto-lint clean docker all

LDFLAGS := -trimpath -ldflags="-s -w -X main.version=$(shell git describe --tags --always --dirty)"

<service>:
	go build $(LDFLAGS) -o <service> ./cmd/<service>

build:
	go build ./...

test:
	go test ./...

vet:
	go vet ./...

lint:
	golangci-lint run ./...

proto:
	protoc --go_out=. --go_opt=module=<module> \
		--go-grpc_out=. --go-grpc_opt=module=<module> \
		proto/<service>/v2/*.proto

proto-lint:
	buf lint
	buf breaking --against '.git#branch=master,subdir=proto'

clean:
	rm -f <service>

docker:
	docker build -t <service> -f Dockerfile.api .

all: vet lint test <service>
```

### Target Semantics

| Target | When to Run | CI Gate? |
|--------|-------------|----------|
| `vet` | Every change | Yes |
| `lint` | Every change | Yes |
| `test` | Every change | Yes |
| `proto-lint` | Any proto change | Yes |
| `proto` | After editing `.proto` files | No (manual) |
| `all` | Pre-push verification | Yes |

The `all` target is the CI pipeline: `vet → lint → test → build`. If any
step fails, the pipeline stops.

---

## API Design

Services expose two synchronized API surfaces:

### gRPC (Primary)

- Proto definitions live in `proto/<service>/v<N>/`, where N starts at 1.
- **Versioning policy**: proto packages are versioned to protect existing
  clients from breaking changes. A new version directory (`v2/`, `v3/`, …)
  is only introduced when a breaking change is unavoidable. Non-breaking
  additions (new fields, new RPCs) are made in-place to the current version.
- Use strongly-typed, per-operation RPCs. Avoid generic "execute" patterns.
- Use `google.protobuf.Timestamp` for all time fields (not RFC 3339 strings).
- Run `buf lint` and `buf breaking` against master before merging proto
  changes.
- **Input validation**: gRPC handlers must validate input fields (non-empty
  required strings, positive IDs, valid enum values) and return
  `codes.InvalidArgument` with a descriptive message on failure. This mirrors
  the validation that REST handlers perform and ensures both API surfaces
  reject bad input consistently.

### REST (Secondary)

- JSON over HTTPS. Routes live in `internal/server/routes.go`.
- Use `chi` for routing (lightweight, stdlib-compatible).
- Standard error format: `{"error": "description"}`.
- Standard HTTP status codes: `401` (unauthenticated), `403` (unauthorized),
  `412` (precondition failed), `503` (service unavailable).

### API Sync Rule

**Every REST endpoint must have a corresponding gRPC RPC, and vice versa.**
When adding, removing, or changing an endpoint in either surface, the other
must be updated in the same change. This is enforced in code review.

### gRPC Interceptors

Access control is enforced via interceptor maps, not per-handler checks:

| Map | Effect |
|-----|--------|
| `sealRequiredMethods` | Returns `UNAVAILABLE` if the service is sealed/locked |
| `authRequiredMethods` | Validates MCIAS bearer token, populates caller info |
| `adminRequiredMethods` | Requires admin role on the caller |

Adding a new RPC means adding it to the correct interceptor maps. Forgetting
this is a security defect.

---

## Authentication & Authorization

### Authentication

All services delegate authentication to **MCIAS** (Metacircular Identity and
Access Service). No service maintains its own user database.

- Client sends credentials to the service's `/v1/auth/login` endpoint.
- The service forwards them to MCIAS via the client library
  (`git.wntrmute.dev/kyle/mcias/clients/go`).
- On success, MCIAS returns a bearer token. The service returns it to the
  client and optionally sets it as a cookie for the web UI.
- Subsequent requests include the token via `Authorization: Bearer <token>`
  header or cookie.
- Token validation calls MCIAS `ValidateToken()`. Results should be cached
  (keyed by SHA-256 of the token) with a short TTL (30 seconds or less).

### Authorization

Three role levels:

| Role | Meaning |
|------|---------|
| `admin` | Full access to everything. Policy bypass. |
| `user` | Access governed by policy rules. Default deny. |
| `guest` | Service-dependent restrictions. Default deny. |

Admin detection is based solely on the MCIAS `admin` role. The service never
promotes users locally.

Services that need fine-grained access control should implement a policy
engine (priority-based ACL rules stored in encrypted storage, default deny,
admin bypass). See Metacrypt's implementation as the reference.

---

## Database Conventions

### SQLite

SQLite is the default database for Metacircular services. It is simple to
operate, requires no external processes, and backs up cleanly with
`VACUUM INTO`.

Connection settings (applied at open time):

```go
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
```

File permissions: `0600`. Created by the service on first run.

### Migrations

- Migrations are Go functions registered in `internal/db/` and run
  sequentially at startup.
- Each migration is idempotent — `CREATE TABLE IF NOT EXISTS`,
  `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`.
- Seed data migrations must use `INSERT OR IGNORE` (not plain `INSERT`)
  to ensure idempotency when the migration runs against a database that
  already contains the seed rows.
- Applied migrations are tracked in a `schema_migrations` table.
- Never modify a migration that has been deployed. Add a new one.

### Backup

Every service must provide a `snapshot` CLI command that creates a consistent
backup using `VACUUM INTO`. Automated backups run via a systemd timer
(daily, with retention pruning).

---

## Configuration

### Format

TOML. Parsed with `go-toml/v2`. Environment variable overrides via
`SERVICENAME_*` (e.g. `METACRYPT_SERVER_LISTEN_ADDR`).

### Standard Sections

```toml
[server]
listen_addr = ":8443"           # HTTPS API
grpc_addr   = ":9443"           # gRPC (optional; disabled if unset)
tls_cert    = "/srv/<service>/certs/cert.pem"
tls_key     = "/srv/<service>/certs/key.pem"

[web]
listen_addr  = "127.0.0.1:8080" # Web UI (optional; disabled if unset)
vault_grpc   = "127.0.0.1:9443" # gRPC address of the API server
vault_ca_cert = ""               # CA cert for verifying API server TLS

[database]
path = "/srv/<service>/<service>.db"

[mcias]
server_url   = "https://mcias.metacircular.net:8443"
ca_cert      = ""                # Custom CA for MCIAS TLS
service_name = "<service>"       # This service's identity, as registered in MCIAS
tags         = []                # Tags sent with every login request (e.g. ["env:restricted"])
                                 # MCIAS evaluates auth:login policy against these tags,
                                 # enabling per-service login restrictions via policy rules.

[log]
level = "info"                   # debug, info, warn, error
```

#### Service context and login policy

`service_name` and `tags` in `[mcias]` are sent with every `POST /v1/auth/login`
request. MCIAS evaluates the `auth:login` action with the resource set to
`{service_name, tags}`. This allows operators to write deny rules that restrict
which roles or account types can log into specific services.

Example: deny `guest` and `viewer` human accounts from any service tagged
`env:restricted`:

```json
{
  "effect": "deny",
  "roles": ["guest", "viewer"],
  "account_types": ["human"],
  "actions": ["auth:login"],
  "required_tags": ["env:restricted"]
}
```

A service can also be targeted by name instead of (or in addition to) tags:

```json
{
  "effect": "deny",
  "roles": ["guest"],
  "actions": ["auth:login"],
  "service_names": ["meta-money-printer"]
}
```

MCIAS enforces the policy after credentials are verified; a policy-denied
login returns HTTP 403 (not 401) so the client can distinguish a bad password
from a service access restriction.

### Validation

Required fields are validated at startup. The service refuses to start if
any are missing. Do not silently default required values.

### Data Directory

All runtime data lives in `/srv/<service>/`:

```
/srv/<service>/
├── <service>.toml        Configuration
├── <service>.db          SQLite database
├── certs/                TLS certificates
└── backups/              Database snapshots
```

This convention enables straightforward service migration between hosts:
copy `/srv/<service>/` and the binary.

---

## Web UI

### Technology

- **Go `html/template`** for server-side rendering. No JavaScript frameworks.
- **htmx** for dynamic interactions (form submission, partial page updates)
  without full page reloads.
- Templates and static files are embedded in the binary via `//go:embed`.

### Structure

- `web/templates/layout.html` — shared HTML skeleton, navigation, CSS/JS
  includes. All page templates extend this.
- Page templates: one `.html` file per page/feature.
- `web/static/` — CSS, htmx. Keep this minimal.

### Architecture

The web UI runs as a separate binary (`<service>-web`) that communicates
with the API server via its gRPC interface. This separation means:

- The web UI has no direct database access.
- The API server enforces all authorization.
- The web UI can be deployed independently or omitted entirely.

### Security

- CSRF protection via signed double-submit cookies on all mutating requests
  (POST/PUT/PATCH/DELETE).
- Session cookie: `HttpOnly`, `Secure`, `SameSite=Strict`.
- All user input is escaped by `html/template` (the default).

---

## Testing

### Philosophy

Tests are written using the Go standard library `testing` package. No test
frameworks (testify, gomega, etc.) — the standard library is sufficient and
keeps dependencies minimal.

### Patterns

```go
func TestFeatureName(t *testing.T) {
    // Setup: use t.TempDir() for isolated file system state.
    dir := t.TempDir()
    database, err := db.Open(filepath.Join(dir, "test.db"))
    if err != nil {
        t.Fatalf("open db: %v", err)
    }
    defer func() { _ = database.Close() }()
    db.Migrate(database)

    // Exercise the code under test.
    // ...

    // Assert with t.Fatal (not t.Error) for precondition failures.
    if !bytes.Equal(got, want) {
        t.Fatalf("got %q, want %q", got, want)
    }
}
```

### Guidelines

- **Use `t.TempDir()`** for all file-system state. Never write to fixed
  paths. Cleanup is automatic.
- **Use `errors.Is`** for error assertions, not string comparison.
- **No mocks for databases.** Tests use real SQLite databases created in
  temp directories. This catches migration bugs that mocks would hide.
- **Test files** live alongside the code they test: `barrier.go` and
  `barrier_test.go` in the same package.
- **Test helpers** call `t.Helper()` so failures report the caller's line.

### What to Test

| Layer | Test Strategy |
|-------|---------------|
| Crypto primitives | Roundtrip encryption/decryption, wrong-key rejection, edge cases |
| Storage (barrier, DB) | CRUD operations, sealed-state rejection, concurrent access |
| API handlers | Request/response correctness, auth enforcement, error codes |
| Policy engine | Rule matching, priority ordering, default deny, admin bypass |
| CLI commands | Flag parsing, output format (lightweight) |

---

## Linting & Static Analysis

### Configuration

Every repository includes a `.golangci.yaml` with this philosophy:
**fail loudly for security and correctness; everything else is a warning.**

### Required Linters

| Linter | Category | Purpose |
|--------|----------|---------|
| `errcheck` | Correctness | Unhandled errors are silent failures |
| `govet` | Correctness | Printf mismatches, unreachable code, suspicious constructs |
| `ineffassign` | Correctness | Dead writes hide logic bugs |
| `unused` | Correctness | Unused variables and functions |
| `errorlint` | Error handling | Proper `errors.Is`/`errors.As` usage |
| `gosec` | Security | Hardcoded secrets, weak RNG, insecure crypto, SQL injection |
| `staticcheck` | Security | Deprecated APIs, mutex misuse, deep analysis |
| `revive` | Style | Go naming conventions, error return ordering |
| `gofmt` | Formatting | Standard Go formatting |
| `goimports` | Formatting | Import grouping and ordering |

### Settings

- `errcheck`: `check-type-assertions: true` (catch `x.(*T)` without ok check).
- `govet`: all analyzers enabled except `shadow` (too noisy for idiomatic Go).
- `gosec`: severity and confidence set to `medium`. Exclude `G104` (overlaps
  with errcheck).
- `max-issues-per-linter: 0` — report everything. No caps.
- Test files: allow `G101` (hardcoded credentials) for test fixtures.

---

## Deployment

### Container-First

Services are designed for container deployment but must also run as native
systemd services. Both paths are first-class.

### Docker

Multi-stage builds:

1. **Builder**: `golang:1.23-alpine`. Compile with `CGO_ENABLED=0`, strip
   symbols.
2. **Runtime**: `alpine:3.21`. Non-root user (`<service>`), minimal attack
   surface.

Runtime images MUST include `ca-certificates` (for TLS verification) and
`tzdata` (for timezone-aware logging and scheduling):

```dockerfile
RUN apk add --no-cache ca-certificates tzdata \
    && addgroup -S <service> \
    && adduser -S -G <service> -h /srv/<service> -s /sbin/nologin <service> \
    && mkdir -p /srv/<service> && chown <service>:<service> /srv/<service>
```

The image should declare `VOLUME /srv/<service>` (to document the data
mount point) and `WORKDIR /srv/<service>` (so relative paths resolve
correctly).

If the service has separate API and web binaries, use separate Dockerfiles
(`Dockerfile.api`, `Dockerfile.web`) and a `docker-compose.yml` that wires
them together with a shared data volume.

### systemd

Every service ships with:

| File | Purpose |
|------|---------|
| `<service>.service` | Main service unit (API server) |
| `<service>-web.service` | Web UI unit (if applicable) |
| `<service>-backup.service` | Oneshot backup unit |
| `<service>-backup.timer` | Daily backup timer (02:00 UTC, 5-minute jitter) |

#### Security Hardening

All service units must include these security directives:

```ini
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
RestrictNamespaces=true
LockPersonality=true
MemoryDenyWriteExecute=true
RestrictRealtime=true
ReadWritePaths=/srv/<service>
```

The web UI unit should use `ReadOnlyPaths=/srv/<service>` instead of
`ReadWritePaths` — it has no reason to write to the data directory.

### Install Script

`deploy/scripts/install.sh` handles:

1. Create system user/group (idempotent).
2. Install binary to `/usr/local/bin/`.
3. Create `/srv/<service>/` directory structure.
4. Install example config if none exists.
5. Install systemd units and reload the daemon.

### Deployment with MCP

The Metacircular Control Plane (MCP) is the standard deployment tool for
container-based services. It manages container lifecycle on target nodes
using rootless Podman.

#### Service Definition Format

MCP service definitions are TOML files stored at
`~/.config/mcp/services/<service>.toml` on the operator workstation. Each
file defines a service with one or more container components:

```toml
name = "metacrypt"
node = "rift"
active = true
path = "metacrypt"

[build]
uses_mcdsl = false

[build.images]
metacrypt = "Dockerfile.api"
metacrypt-web = "Dockerfile.web"

[[components]]
name = "api"
image = "mcr.svc.mcp.metacircular.net:8443/metacrypt:v1.0.0"
network = "mcpnet"
user = "0:0"
restart = "unless-stopped"
ports = ["127.0.0.1:18443:8443", "127.0.0.1:19443:9443"]
volumes = ["/srv/metacrypt:/srv/metacrypt"]
cmd = ["server", "--config", "/srv/metacrypt/metacrypt.toml"]
```

Top-level fields:

| Field | Purpose |
|-------|---------|
| `name` | Service name (matches the project name) |
| `node` | Target host to deploy to |
| `active` | Whether MCP should keep this service running |
| `path` | Source directory relative to the workspace (for builds) |

Build fields:

| Field | Purpose |
|-------|---------|
| `build.uses_mcdsl` | Whether the build requires the mcdsl module |
| `build.images.<name>` | Maps image name to its Dockerfile path |

Component fields:

| Field | Purpose |
|-------|---------|
| `name` | Component name within the service (e.g. `api`, `web`) |
| `image` | Full image reference including MCR registry and version tag |
| `network` | Podman network to attach to |
| `user` | Container user:group |
| `restart` | Restart policy |
| `ports` | Host-to-container port mappings |
| `volumes` | Host-to-container volume mounts |
| `cmd` | Command and arguments passed to the entrypoint |

#### Convention

Projects should include a reference service definition in
`deploy/<service>-rift.toml` as the canonical deployment example. This
file is committed to the repository and kept in sync with the actual
deployment.

#### Key Commands

| Command | Purpose |
|---------|---------|
| `mcp build <service>` | Build and push images for a service |
| `mcp sync` | Push all service definitions to agents; builds missing images if source tree is available |
| `mcp deploy <service>` | Pull image and (re)create containers |
| `mcp stop <service>` | Stop running containers |
| `mcp restart <service>` | Restart containers in place |
| `mcp ps` | List all managed containers and their status |
| `mcp status [service]` | Show detailed status for a specific service |

#### Container User Convention

All containers run as `--user 0:0` (root inside the container). Security
isolation is provided by rootless Podman (the container engine runs as an
unprivileged host user) combined with systemd hardening on the host. This
avoids file permission issues with mounted volumes while maintaining a
strong security boundary at the host level.

#### Image Convention

Container images are pulled from the Metacircular Container Registry (MCR):

```
mcr.svc.mcp.metacircular.net:8443/<service>:<tag>
```

Tags follow semver. Service definitions MUST pin an explicit version tag
(e.g., `v1.1.0`), never `:latest`. This ensures deployments are
reproducible and `mcp status` shows the actual running version.

#### Build and Release Workflow

The standard workflow for releasing a service:

1. **Tag** the release in git (`git tag -a v1.1.0`).
2. **Build** the container images (`mcp build <service>`).
3. **Update** the service definition with the new version tag.
4. **Sync and deploy** (`mcp sync` or `mcp deploy <service>`).

`mcp build` reads the `[build]` section of the service definition to
locate Dockerfiles and the source tree. The workspace root is configured
in `~/.config/mcp/mcp.toml`:

```toml
[build]
workspace = "~/src/metacircular"
```

Each service's `path` field is relative to the workspace. For example,
`path = "mcr"` resolves to `~/src/metacircular/mcr`.

`mcp sync` checks whether each component's image tag exists in the
registry. If a tag is missing and the source tree is available, it
builds and pushes automatically. If the source tree is not available,
it fails with a clear error directing the operator to build first.

### TLS

- **Minimum TLS version: 1.3.** No exceptions, no fallback cipher suites.
  Go's TLS 1.3 implementation manages cipher selection automatically.
- **Timeouts**: read 30s, write 30s, idle 120s.
- Certificate and key paths are required configuration — the service refuses
  to start without them.

### Graceful Shutdown

Services handle `SIGINT` and `SIGTERM`, shutting down cleanly:

1. Stop accepting new connections.
2. Drain in-flight requests (with a timeout).
3. Clean up resources (close databases, zeroize secrets if applicable).
4. Exit.

---

## Documentation

### Required Files

| File | Purpose | Audience |
|------|---------|----------|
| `README.md` | Project overview, quick-start, and contributor guide | Everyone |
| `CLAUDE.md` | AI-assisted development context | Claude Code |
| `ARCHITECTURE.md` | Full system specification | Engineers |
| `RUNBOOK.md` | Operational procedures and incident response | Operators |
| `deploy/examples/<service>.toml` | Example configuration | Operators |

### Suggested Files

These are not required for every project but should be created where applicable:

| File | When to Include | Purpose |
|------|-----------------|---------|
| `AUDIT.md` | Services handling cryptography, secrets, PII, or auth | Security audit findings with issue tracking and resolution status |
| `POLICY.md` | Services with fine-grained access control | Policy engine documentation: rule structure, evaluation algorithm, resource paths, action classification, common patterns |

### README.md

The README is the front door. A new engineer or user should be able to
understand what the service does and get it running from this file alone.
It should contain:

- Project name and one-paragraph description.
- Quick-start instructions (build, configure, run).
- Link to `ARCHITECTURE.md` for full technical details.
- Link to `RUNBOOK.md` for operational procedures.
- License and contribution notes (if applicable).

Keep it concise. The README is not the spec — that's `ARCHITECTURE.md`.

### CLAUDE.md

This file provides context for AI-assisted development. It should contain:

- Project overview (one paragraph).
- Build, test, and lint commands.
- High-level architecture summary.
- Project structure with directory descriptions.
- Ignored directories (runtime data, generated code).
- Critical rules (e.g. API sync requirements).

Keep it concise. AI tools read this on every interaction.

### ARCHITECTURE.md

This is the canonical specification for the service. It should cover:

1. System overview with a layered architecture diagram.
2. Cryptographic design (if applicable): algorithms, key hierarchy.
3. State machines and lifecycle (if applicable).
4. Storage design.
5. Authentication and authorization model.
6. API surface (REST and gRPC, with tables of every endpoint).
7. Web interface routes.
8. Database schema (every table, every column).
9. Configuration reference.
10. Deployment guide.
11. Security model: threat mitigations table and security invariants.
12. Future work.

This document is the source of truth. When the code and the spec disagree,
one of them has a bug.

### RUNBOOK.md

The runbook is written for operators, not developers. It covers what to do
when things go wrong and how to perform routine maintenance. It should
contain:

1. **Service overview** — what the service does, in one paragraph.
2. **Health checks** — how to verify the service is healthy (endpoints,
   CLI commands, expected responses).
3. **Common operations** — start, stop, restart, seal/unseal, backup,
   restore, log inspection.
4. **Alerting** — what alerts exist, what they mean, and how to respond.
5. **Incident procedures** — step-by-step playbooks for known failure
   modes (database corruption, certificate expiry, MCIAS outage, disk
   full, etc.).
6. **Escalation** — when and how to escalate beyond the runbook.

Write runbook entries as numbered steps, not prose. An operator at 3 AM
should be able to follow them without thinking.

### AUDIT.md (Suggested)

For services that handle cryptography, secrets, PII, or authentication,
maintain a security audit log. Each finding gets a numbered entry with:

- Description of the issue.
- Severity (critical, high, medium, low).
- Resolution status: open, resolved (with summary), or accepted (with
  rationale for accepting the risk).

The priority summary table at the bottom provides a scannable overview.
Resolved and accepted items are struck through but retained for history.
See Metacrypt's `AUDIT.md` for the reference format.

### POLICY.md (Suggested)

For services with a policy engine or fine-grained access control, document
the policy model separately from the architecture spec. It should cover:

- Rule structure (fields, types, semantics).
- Evaluation algorithm (match logic, priority, default effect).
- Resource path conventions and glob patterns.
- Action classification.
- API endpoints for policy CRUD.
- Common policy patterns with examples.
- Role summary (what each MCIAS role gets by default).

This document is aimed at administrators who need to write policy rules,
not engineers who need to understand the implementation.

### Engine/Feature Design Documents

For services with a modular architecture, each module gets its own design
document (e.g. `engines/sshca.md`). These are detailed implementation plans
that include:

- Overview and core concepts.
- Data model and storage layout.
- Lifecycle (initialization, teardown).
- Operations table with auth requirements.
- API definitions (gRPC and REST).
- Implementation steps (file-by-file).
- Security considerations.
- References to existing code patterns to follow.

Write these before writing code. They are the blueprint, not the afterthought.

---

## Security

### General Principles

- **Default deny.** Unauthenticated requests are rejected. Unauthorized
  requests are rejected. If in doubt, deny.
- **Fail closed.** If the service cannot verify authorization, it denies the
  request. If the database is unavailable, the service is unavailable.
- **Least privilege.** Service processes run as non-root. systemd units
  restrict filesystem access, syscalls, and capabilities.
- **No local user databases.** Authentication is always delegated to MCIAS.

### Cryptographic Standards

| Purpose | Algorithm | Notes |
|---------|-----------|-------|
| Symmetric encryption | AES-256-GCM | 12-byte random nonce per operation |
| Symmetric alternative | XChaCha20-Poly1305 | For contexts needing nonce misuse resistance |
| Key derivation | Argon2id | Memory-hard; tune params to hardware |
| Asymmetric signing | Ed25519, ECDSA (P-256, P-384) | Prefer Ed25519 |
| CSPRNG | `crypto/rand` | All keys, nonces, salts, tokens |
| Constant-time comparison | `crypto/subtle` | All secret comparisons |

- **Never use RSA for new designs.** Ed25519 and ECDSA are faster, produce
  smaller keys, and have simpler security models.
- **Zeroize secrets** from memory when they are no longer needed. Overwrite
  byte slices with zeros, nil out pointers.
- **Never log secrets.** Keys, passwords, tokens, and plaintext must never
  appear in log output.

### Web Security

- CSRF tokens on all mutating requests.
- `SameSite=Strict` on all cookies.
- `html/template` for automatic escaping.
- Validate all input at system boundaries.

---

## Development Workflow

### Local Development

```bash
# Build and run both servers locally:
make devserver

# Or build everything and run the full pipeline:
make all
```

The `devserver` target builds both binaries and runs them against a local
config in `srv/`. The `srv/` directory is gitignored — it holds your local
database, certificates, and configuration.

### Pre-Push Checklist

Before pushing a branch:

```bash
make all          # vet → lint → test → build
make proto-lint   # if proto files changed
```

### Proto Changes

1. Edit `.proto` files in `proto/<service>/v2/`.
2. Run `make proto` to regenerate Go code.
3. Run `make proto-lint` to check for linting violations and breaking changes.
4. Update REST routes to match the new/changed RPCs.
5. Update gRPC interceptor maps for any new RPCs.
6. Update `ARCHITECTURE.md` API tables.

### Adding a New Feature

1. **Design first.** Write or update the relevant design document. For a new
   engine or major subsystem, create a new doc in `docs/` or `engines/`.
2. **Implement.** Follow existing patterns — the design doc should reference
   specific files and line numbers.
3. **Test.** Write tests alongside the implementation.
4. **Update docs.** Update `ARCHITECTURE.md`, `CLAUDE.md`, and route tables.
5. **Verify.** Run `make all`.

### CLI Commands

Every service uses cobra for CLI commands. Standard subcommands:

| Command | Purpose |
|---------|---------|
| `server` | Start the service |
| `init` | First-time setup (if applicable) |
| `status` | Query a running instance's health |
| `snapshot` | Create a database backup |

Add service-specific subcommands as needed (e.g. `migrate-aad`, `unseal`).
Each command lives in its own file in `cmd/<service>/`.
