# Building MCQ: A Reading Queue in a Session

*A document reading queue goes from design discussion to production
deployment — including the deployment's sharp edges.*

*Written by Claude (Opus 4.6), reflecting on a collaborative session with
Kyle. This log focuses heavily on the deployment side, including what
went wrong, what was done manually versus what should have been MCP-managed,
and why `mcp ps` doesn't know mcq exists.*

---

## The Idea

Kyle was out and about, away from his tailnet, and wanted to review
platform documentation on his phone. The existing tools — mcdoc (which
renders docs from Gitea repos) and the repos themselves — require either
tailnet access or a desktop workflow.

The concept: a **document queue**. Push raw markdown from inside the
infrastructure, read rendered HTML from anywhere via a browser. Like a
self-hosted Pocket, but for internal docs you're actively iterating on.

After a design discussion, we settled on:

- **Name**: mcq (Metacircular Document Queue)
- **Data model**: Documents keyed by slug, upsert semantics (re-push
  replaces content, resets read flag)
- **Auth**: MCIAS on everything — any user including guest can read, any
  user including system accounts can push
- **Rendering**: Goldmark with GFM + syntax highlighting, rendered on
  each page view
- **Architecture**: Single binary, REST API + gRPC + web UI

## Building the Service

### Codebase Exploration

Before writing any code, I explored the existing platform services to
understand the patterns:

- **mcat** (`~/src/metacircular/mcat/`): Reference for the web UI pattern —
  chi router, CSRF, session cookies, htmx, embedded templates, cobra CLI,
  config loading via `mcdsl/config`.
- **mcns** (`~/src/metacircular/mcns/`): Reference for REST + gRPC pattern —
  separate `internal/server/` (REST) and `internal/grpcserver/` (gRPC),
  method maps for auth interceptors, SQLite via `mcdsl/db`.
- **mcdoc** (`~/src/metacircular/mcdoc/`): Reference for goldmark rendering
  and plain HTTP serving (mcdoc doesn't use mcdsl for config or HTTP — it
  has its own, because it serves plain HTTP behind mc-proxy).
- **mcdsl** (`~/src/metacircular/mcdsl/`): The shared library — auth,
  config, db, httpserver, grpcserver, csrf, web packages.

### Implementation (on vade, Kyle's workstation)

Created `~/src/mcq/` with the standard platform layout:

```
cmd/mcq/              main.go, server.go (cobra CLI)
internal/
  config/             custom config (TLS optional, see below)
  db/                 SQLite schema, migrations, document CRUD
  server/             REST API routes and handlers
  grpcserver/         gRPC server, interceptors, service handlers
  webserver/          Web UI routes, templates, session management
  render/             goldmark markdown-to-HTML renderer
proto/mcq/v1/         Protobuf definitions
gen/mcq/v1/           Generated Go code
web/                  Embedded templates + static files
deploy/               systemd, examples
```

Key files:

- **Proto** (`proto/mcq/v1/mcq.proto`): DocumentService (ListDocuments,
  GetDocument, PutDocument, DeleteDocument, MarkRead, MarkUnread),
  AuthService (Login, Logout), AdminService (Health).
- **DB** (`internal/db/documents.go`): Single `documents` table with slug
  as unique key. PutDocument uses `INSERT ... ON CONFLICT(slug) DO UPDATE`.
- **REST** (`internal/server/routes.go`): All routes under `/v1/` —
  `PUT /v1/documents/{slug}` for upsert, standard CRUD otherwise.
- **Web UI** (`internal/webserver/server.go`): Login page, document list
  at `/`, rendered markdown reader at `/d/{slug}`.
- **gRPC** (`internal/grpcserver/`): Mirrors REST exactly. Method map puts
  all document operations in `authRequiredMethods`, nothing in
  `adminRequiredMethods`.

Proto generation ran on vade:

```bash
cd ~/src/mcq
protoc --go_out=. --go_opt=module=git.wntrmute.dev/mc/mcq \
    --go-grpc_out=. --go-grpc_opt=module=git.wntrmute.dev/mc/mcq \
    proto/mcq/v1/*.proto
```

### The .gitignore Bug

First `git add -A` missed `cmd/mcq/`, `proto/mcq/`, and `gen/mcq/`. The
`.gitignore` had:

```
mcq
srv/
```

The pattern `mcq` (without a leading slash) matches any file or directory
named `mcq` at any level — so it was ignoring `cmd/mcq/`, `gen/mcq/`, and
`proto/mcq/`. Fixed to:

```
/mcq
/srv/
```

### The TLS Decision

This was the most consequential design decision for deployment.

The standard platform pattern (mcdsl's `httpserver`) enforces TLS 1.3
minimum. But mc-proxy on svc terminates TLS at the edge and forwards to
backends as plain HTTP (for localhost services) or HTTPS (for remote
backends like rift). Gitea on svc runs plain HTTP on port 3000 behind
mc-proxy. mcdoc on rift runs plain HTTP on port 38080 behind mc-proxy.

mcdsl's `config.Load` validates that `tls_cert` and `tls_key` are present
— they're required fields. So I couldn't use `config.Base` with empty TLS
fields.

**Solution**: Created `internal/config/config.go` — mcq's own config
package, modeled after mcdoc's. Same TOML loading, env var overrides, and
validation, but TLS fields are optional. When empty, the server uses
`http.ListenAndServe()` instead of `httpserver.ListenAndServeTLS()`.

This meant giving up the mcdsl httpserver (with its logging middleware and
TLS enforcement) for the plain HTTP path. The gRPC server was also dropped
from the svc deployment since it requires TLS. The REST API and web UI
are sufficient for the use case.

### Build and Test (on vade)

```bash
cd ~/src/mcq
go mod tidy
go build ./...      # clean
go vet ./...        # clean
go test ./...       # 6 tests pass (all in internal/db)

# Production binary
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w -X main.version=v0.1.0" \
    -o mcq ./cmd/mcq

# Result: 21MB static binary
```

---

## Deployment

### Why mcq is NOT in `mcp ps`

**This is the most important thing in this log.**

mcq was deployed as a **manual systemd service on svc**, not as an
MCP-managed container. This means:

- `mcp ps` doesn't know about it
- `mcp stop mcq` won't work
- `mcp deploy mcq` won't work
- There's no service definition in `~/.config/mcp/services/`
- There's no container image in MCR
- The binary was `scp`'d directly to svc and `install`'d to `/usr/local/bin/`

**Why?** Three reasons:

1. **svc has no MCP agent.** The MCP agent (`mcp-agent`) only runs on rift.
   svc is a Debian VPS that hosts MCIAS, mc-proxy, MCNS, and Gitea — all
   deployed as manual systemd services, not via MCP. Getting mcq into MCP
   would require deploying an MCP agent to svc first (Phase E in
   PLATFORM_EVOLUTION.md, items #10-#12).

2. **mcq runs as a native binary, not a container.** MCP manages containers
   (podman). mcq on svc is a bare binary under systemd, like MCIAS and
   mc-proxy on svc. To make it MCP-managed, it would need to be
   containerized and pushed to MCR first.

3. **The deployment followed the existing svc pattern.** Every service on
   svc was deployed this way: build on vade, scp to svc, install, write
   config, write systemd unit, enable. This was a deliberate choice to
   match the existing operational model rather than block on MCP agent
   deployment.

### What MCP-managed deployment would look like

Once svc has an MCP agent, mcq could be managed like services on rift:

```toml
# ~/.config/mcp/services/mcq.toml
name = "mcq"
node = "svc"
version = "v0.1.0"

[[components]]
name = "api"

[[components.routes]]
port = 8090
mode = "l7"
hostname = "mcq.metacircular.net"
```

This would require:
- MCP agent running on svc
- mcq containerized (Dockerfile) and pushed to MCR
- Agent handles port assignment, mc-proxy route registration, lifecycle

### The Actual Deployment Steps

All commands below were run from vade (Kyle's workstation) via SSH to svc,
unless otherwise noted.

#### 1. Push repo to Gitea (from vade)

```bash
cd ~/src/mcq
git remote add origin git@git.wntrmute.dev:mc/mcq.git
git push -u origin master
```

The mc/mcq repo was created manually in Gitea (the MCP tool's API token
lacked `write:organization` scope for creating repos under the mc org).

#### 2. Copy binary to svc (from vade)

```bash
scp ~/src/mcq/mcq kyle@svc:/tmp/mcq
```

SSH to svc uses Tailscale hostname resolution — `svc` resolves to
`100.106.232.4` via tailscale. No SSH config entry was needed. Had to
accept the host key on first connection:

```bash
ssh -o StrictHostKeyChecking=accept-new kyle@svc
```

#### 3. Create user and install binary (on svc, as root via sudo)

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin mcq
sudo mkdir -p /srv/mcq
sudo chown mcq:mcq /srv/mcq
sudo chmod 0700 /srv/mcq
sudo install -m 0755 /tmp/mcq /usr/local/bin/mcq
```

Verified: `/usr/local/bin/mcq --version` → `mcq version v0.1.0`

#### 4. Write config (on svc)

Created `/srv/mcq/mcq.toml`:

```toml
[server]
listen_addr = "127.0.0.1:8090"

[database]
path = "/srv/mcq/mcq.db"

[mcias]
server_url   = "https://mcias.metacircular.net:8443"
ca_cert      = "/srv/mcq/ca.pem"
service_name = "mcq"
tags         = []

[log]
level = "info"
```

**Important detail**: The first attempt used `server_url = "https://127.0.0.1:8443"`
which failed because MCIAS's TLS cert has SANs for `mcias.wntrmute.dev`
and `mcias.metacircular.net` but **not** `127.0.0.1` or `localhost`. Token
validation returned "invalid or expired token" because the mcdsl auth
client couldn't establish a TLS connection to MCIAS.

Fixed by copying the pattern from MCNS on svc:
- `server_url = "https://mcias.metacircular.net:8443"` (uses the hostname
  that matches the cert's SAN)
- `ca_cert = "/srv/mcq/ca.pem"` (the WNTRMUTE root CA cert, copied from
  `/srv/mcns/certs/ca.pem`)

The hostname `mcias.metacircular.net` resolves to svc's public IP, so
this still connects to localhost MCIAS — it just goes through the public
IP for TLS hostname verification. (On a locked-down firewall this could
be an issue, but svc allows loopback through its public IP.)

#### 5. Create systemd unit (on svc)

Created `/etc/systemd/system/mcq.service`:

```ini
[Unit]
Description=MCQ Document Queue
After=network-online.target mcias.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mcq server --config /srv/mcq/mcq.toml
WorkingDirectory=/srv/mcq
Restart=on-failure
RestartSec=5
User=mcq
Group=mcq

NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/srv/mcq
PrivateTmp=yes
ProtectKernelTunables=yes
ProtectControlGroups=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mcq
```

Verified running: PID 3765144, memory 7.8MB, started cleanly.

#### 6. Generate TLS certificate for mc-proxy (on vade)

mc-proxy needs a TLS cert for the `mcq.metacircular.net` hostname (it
terminates TLS at the edge). Generated using the local WNTRMUTE root CA:

```bash
cd /tmp
openssl ecparam -name prime256v1 -genkey -noout -out mcq.key

openssl req -new -key mcq.key -out mcq.csr \
    -subj "/CN=mcq.metacircular.net/O=Metacircular Dynamics" \
    -addext "subjectAltName=DNS:mcq.metacircular.net"

openssl x509 -req -in mcq.csr \
    -CA ~/src/metacircular/ca/ca.pem \
    -CAkey ~/src/metacircular/ca/ca.key \
    -CAcreateserial -out mcq.pem -days 365 -sha256 \
    -extfile <(echo "subjectAltName=DNS:mcq.metacircular.net
keyUsage=digitalSignature
extendedKeyUsage=serverAuth")
```

The CA key and cert are at `~/src/metacircular/ca/` — this is the
WNTRMUTE Issuing Authority root CA. Not Metacrypt (which has its own
intermediate CA for automated issuance). The existing mc-proxy certs
(docs, git, metacrypt) were all signed by this same root CA.

Copied to svc:

```bash
scp /tmp/mcq.pem /tmp/mcq.key kyle@svc:/tmp/
```

Installed on svc:

```bash
sudo cp /tmp/mcq.pem /srv/mc-proxy/certs/mcq.metacircular.net.pem
sudo cp /tmp/mcq.key /srv/mc-proxy/certs/mcq.metacircular.net.key
sudo chown mc-proxy:mc-proxy /srv/mc-proxy/certs/mcq.metacircular.net.*
sudo chmod 0600 /srv/mc-proxy/certs/mcq.metacircular.net.key
```

#### 7. Add mc-proxy route (on svc)

mc-proxy on svc uses SQLite for route persistence. The TOML config only
seeds the database on first run (`store.IsEmpty()` check). After that,
routes are loaded from SQLite. So editing the TOML alone doesn't add a
route — you must also insert into the database.

I did both (TOML for documentation/re-seeding, SQLite for immediate effect):

**TOML** (added via `sed` to `/srv/mc-proxy/mc-proxy.toml`):

```toml
[[listeners.routes]]
hostname    = "mcq.metacircular.net"
backend     = "127.0.0.1:8090"
mode        = "l7"
tls_cert    = "/srv/mc-proxy/certs/mcq.metacircular.net.pem"
tls_key     = "/srv/mc-proxy/certs/mcq.metacircular.net.key"
backend_tls = false
```

**SQLite** (direct insert):

```bash
sudo sqlite3 /srv/mc-proxy/mc-proxy.db "
INSERT INTO routes (listener_id, hostname, backend, mode, tls_cert, tls_key, backend_tls)
VALUES (1, 'mcq.metacircular.net', '127.0.0.1:8090', 'l7',
        '/srv/mc-proxy/certs/mcq.metacircular.net.pem',
        '/srv/mc-proxy/certs/mcq.metacircular.net.key', 0);
"
```

The `listener_id = 1` is the `:443` listener (only listener on svc's
mc-proxy).

**Note on `backend_tls = false`**: mcq serves plain HTTP on localhost.
mc-proxy terminates TLS for the client and forwards as plain HTTP to
`127.0.0.1:8090`. This is the same pattern as Gitea (`127.0.0.1:3000`)
and mcdoc (`100.95.252.120:38080`). Only metacrypt uses `backend_tls = true`
because its backend is on rift over Tailscale.

#### 8. Restart mc-proxy (on svc)

```bash
sudo systemctl restart mc-proxy
```

This was messy. mc-proxy's graceful shutdown waits for in-flight
connections to drain, and the 30-second shutdown timeout was exceeded
(lingering connections from internet scanners hitting git.metacircular.net).
The shutdown hung for ~30 seconds before logging "shutdown timeout exceeded,
forcing close". systemd then moved to `deactivating (stop-sigterm)` state.

Had to force it:

```bash
sudo systemctl kill mc-proxy
sleep 2
sudo systemctl start mc-proxy
```

After restart: `routes=5` (was 4 before mcq). Confirmed:

```bash
curl -sk https://mcq.metacircular.net/v1/health
# {"status":"ok"}
```

#### 9. Push documents (from vade)

Used the mcp-agent service account token (from
`~/data/downloads/service-account-76d35a82-77ca-422f-85a3-b9f9360d5164.token`)
to authenticate API calls. This is a long-lived JWT issued by MCIAS with
`admin` role, `exp` in 2027.

```bash
TOKEN=$(cat ~/data/downloads/service-account-*.token)

# Push MCP Architecture
python3 -c "
import json
body = open('mcp/ARCHITECTURE.md').read()
print(json.dumps({'title': 'MCP Architecture', 'body': body}))
" | curl -sk -X PUT https://mcq.metacircular.net/v1/documents/mcp-architecture \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @-

# Push Platform Evolution
python3 -c "
import json
body = open('PLATFORM_EVOLUTION.md').read()
print(json.dumps({'title': 'Platform Evolution', 'body': body}))
" | curl -sk -X PUT https://mcq.metacircular.net/v1/documents/platform-evolution \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @-

# Push Packaging doc
python3 -c "
import json
body = open('docs/packaging-and-deployment.md').read()
print(json.dumps({'title': 'Packaging and Deployment', 'body': body}))
" | curl -sk -X PUT https://mcq.metacircular.net/v1/documents/packaging-and-deployment \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @-
```

Used `python3` for JSON encoding because `jq` isn't installed on vade
(NixOS — would need to add it to the system config or use `nix-shell`).

All three documents pushed successfully. The token identifies as
`mcp-agent` (the service account name), so `pushed_by` shows `mcp-agent`
on each document.

### Subsequent Update: Tufte Theme

Kyle wanted a wider reading area (70%) and a Tufte-inspired theme. Updated
`web/static/style.css`:

- Serif font stack (Georgia, Palatino)
- Cream background (`#fffff8`)
- Italic headings, small-caps labels
- `width: 70%` on `.page-container` (was `max-width: 720px`)
- Minimal chrome — document list uses ruled lines instead of cards,
  tables use bottom-borders only
- Mobile fallback: full width below 768px

Rebuilt, deployed same way:

```bash
# On vade
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w -X main.version=v0.1.1" \
    -o mcq ./cmd/mcq
scp mcq kyle@svc:/tmp/mcq

# On svc
sudo install -m 0755 /tmp/mcq /usr/local/bin/mcq
sudo systemctl restart mcq
```

---

## State After This Session

### What's running on svc

| Service | Port | Managed by | Notes |
|---------|------|------------|-------|
| MCIAS | :8443/:9443 | systemd | Identity/auth, been here longest |
| mc-proxy | :443 | systemd | L7 TLS termination, 5 routes |
| MCNS | :53/:8444/:9444 | systemd | Authoritative DNS |
| Gitea | :3000 | systemd | Git hosting |
| **mcq** | **:8090** | **systemd** | **NEW: document queue** |

None of these are MCP-managed. svc has no MCP agent.

### mc-proxy routes on svc

| Hostname | Backend | Mode | TLS Backend |
|----------|---------|------|-------------|
| metacrypt.metacircular.net | 100.95.252.120:18080 | L7 | yes (rift) |
| git.metacircular.net | 127.0.0.1:3000 | L7 | no |
| git.wntrmute.dev | 127.0.0.1:3000 | L7 | no |
| docs.metacircular.net | 100.95.252.120:38080 | L7 | no |
| **mcq.metacircular.net** | **127.0.0.1:8090** | **L7** | **no** |

### DNS

`mcq.metacircular.net` is a CNAME to `svc.metacircular.net` (set up by
Kyle at the DNS registrar before this session). mc-proxy's SNI-based
routing handles the rest.

### Documents in queue

| Slug | Title | Pushed By |
|------|-------|-----------|
| mcp-architecture | MCP Architecture | mcp-agent |
| platform-evolution | Platform Evolution | mcp-agent |
| packaging-and-deployment | Packaging and Deployment | mcp-agent |

### Git

Repo: `mc/mcq` on Gitea (`git.wntrmute.dev:mc/mcq.git`)

Commits:
1. `bc16279` — Initial implementation
2. `648e9dc` — Support plain HTTP mode for mc-proxy L7 deployment
3. `a5b90b6` — Switch to Tufte-inspired reading theme

---

## What Would Be Different with MCP

If svc had an MCP agent and mcq were containerized:

1. **No manual SSH** — `mcp deploy mcq` from vade would push the service
   definition, agent would pull the image from MCR.
2. **No manual port picking** — agent assigns a free port from 10000-60000.
3. **No manual mc-proxy route** — agent calls mc-proxy's gRPC API to
   register the route (Phase B, already working on rift).
4. **No manual TLS cert** — agent provisions from Metacrypt CA
   (Phase C, already working on rift).
5. **No manual systemd unit** — agent manages the container lifecycle.
6. **`mcp ps` would show mcq** — because the agent tracks it in its
   registry.
7. **`mcp stop mcq` / `mcp restart mcq` would work** — standard lifecycle.

The gap is: svc has no agent. That's Phase E work (items #10-#12 in
PLATFORM_EVOLUTION.md). The prerequisites are the agent binary location
convention, SSH-based upgrade tooling, and node provisioning for Debian.

---

## Rough Edges and Lessons

1. **MCIAS cert hostname**: Every new service on svc will hit this. The
   MCIAS cert doesn't include localhost as a SAN. Services must use
   `server_url = "https://mcias.metacircular.net:8443"` (which routes
   through the public IP back to localhost) and include the CA cert.
   Could fix by reissuing the MCIAS cert with a localhost SAN.

2. **mc-proxy route persistence**: The TOML-seeds-once-then-SQLite model
   means you have to touch two places (TOML for future re-seeds, SQLite
   for immediate effect). On rift this is handled by the agent's gRPC
   calls. On svc without an agent, it's manual database surgery.

3. **mc-proxy shutdown timeout**: The 30-second timeout isn't enough when
   internet scanners maintain persistent connections to git.metacircular.net.
   Had to force-kill on restart. Should increase `shutdown_timeout` or
   add a SIGKILL escalation in the systemd unit (`TimeoutStopSec=45`,
   which sends SIGKILL after 45s).

4. **No jq on vade**: NixOS doesn't have jq in the default system config.
   Used python3 as a workaround for JSON encoding. Minor friction.

5. **mcdsl httpserver assumes TLS**: Services behind mc-proxy L7 can't use
   `mcdsl/httpserver` because it enforces TLS 1.3. mcdoc solved this with
   its own config/server. mcq now does the same. This is a recurring
   pattern — might warrant adding a plain HTTP mode to mcdsl httpserver,
   or a separate `mcdsl/httpserver/plain` package.

6. **Session cookie Secure flag behind plain HTTP**: The mcdsl `web`
   package always sets `Secure: true` on session cookies. This works
   behind mc-proxy L7 because the *browser* sees HTTPS (mc-proxy
   terminates TLS) — the `Secure` flag is about the browser's view of
   the connection, not the backend. If mcq were ever accessed directly
   (not through mc-proxy), cookies would silently fail.
