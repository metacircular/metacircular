# svc.metacircular.net — Phase 1 Work Log

Date: 2026-04-02
Purpose: Deploy mcp-agent to svc (edge node) for MCP v2 Phase 1.

## Changes Made

### 1. Created `mcp` system user
```
useradd --system --home-dir /srv/mcp --create-home --shell /usr/sbin/nologin mcp
usermod -aG mc-proxy mcp
```
- UID 992, GID 991
- Member of `mc-proxy` group for socket access

### 2. Created `/srv/mcp/` directory structure
```
/srv/mcp/
├── mcp-agent          # binary (v0.8.3-1-gfa8ba6f, linux/amd64)
├── mcp-agent.toml     # agent config
├── mcp.db             # SQLite registry (created on first run)
└── certs/
    ├── cert.pem       # TLS cert (SAN: IP:100.106.232.4, DNS:svc.svc.mcp.metacircular.net)
    ├── key.pem        # TLS private key
    └── ca.pem         # Metacircular CA cert
```
- Owned by `mcp:mcp`, key file mode 0600

### 3. TLS certificate
- Issued from the Metacircular CA (`ca/ca.pem` + `ca/ca.key`)
- Subject: `CN=mcp-agent-svc`
- SANs: `IP:100.106.232.4`, `DNS:svc.svc.mcp.metacircular.net`
- Validity: 365 days
- Stored at `/srv/mcp/certs/{cert,key,ca}.pem`

### 4. Agent configuration
- File: `/srv/mcp/mcp-agent.toml`
- gRPC listen: `100.106.232.4:9555` (port 9444 in use by MCNS)
- MCIAS: `https://mcias.metacircular.net:8443`
- mc-proxy socket: `/srv/mc-proxy/mc-proxy.sock`
- Node name: `svc`
- Runtime: `podman` (not used on edge, but required by config)

### 5. systemd unit
- File: `/etc/systemd/system/mcp-agent.service`
- Runs as `mcp:mcp`
- Security hardened (NoNewPrivileges, ProtectSystem=strict, etc.)
- ReadWritePaths: `/srv/mcp`, `/srv/mc-proxy/mc-proxy.sock`
- Enabled and started

### 6. mc-proxy directory permissions
- Changed `/srv/mc-proxy/` from `drwx------` to `drwxr-x---` (group traversal)
- Changed `/srv/mc-proxy/mc-proxy.sock` from `srw-------` to `srw-rw----` (group read/write)
- Required for `mcp` user (in `mc-proxy` group) to access the socket

### 7. MCP CLI config update (on rift)
- Added svc node to `~/.config/mcp/mcp.toml`:
  ```toml
  [[nodes]]
  name = "svc"
  address = "100.106.232.4:9555"
  ```

## Verification
```
$ mcp node list
NAME  ADDRESS              VERSION
rift  100.95.252.120:9444  v0.8.3-dirty
svc   100.106.232.4:9555   v0.8.3-1-gfa8ba6f

$ mcp route list -n svc
NODE: svc
mc-proxy v1.2.1-2-g82fce41-dirty
  :443  routes=6
    l7 git.wntrmute.dev → 127.0.0.1:3000
    l7 kls.metacircular.net → 100.95.252.120:58080
    l7 mcq.metacircular.net → 100.95.252.120:48080
    l7 metacrypt.metacircular.net → 100.95.252.120:18080 (re-encrypt)
    l7 docs.metacircular.net → 100.95.252.120:38080
    l7 git.metacircular.net → 127.0.0.1:3000
```

## Agent Cert Reissue (2026-04-02)

Both agent certs reissued with comprehensive SANs:

**Rift agent** (`/srv/mcp/certs/cert.pem`):
- DNS: `rift.scylla-hammerhead.ts.net`, `mcp-agent.svc.mcp.metacircular.net`
- IP: `100.95.252.120`, `192.168.88.181`

**Svc agent** (`/srv/mcp/certs/cert.pem`):
- DNS: `svc.scylla-hammerhead.ts.net`, `svc.svc.mcp.metacircular.net`
- IP: `100.106.232.4`

Both agents upgraded to v0.10.0 (Phase 2 edge routing RPCs + v2 proto fields).

## MCP Master Deployment (2026-04-02)

**Binary**: `/srv/mcp-master/mcp-master` (v0.10.0) on rift
**Config**: `/srv/mcp-master/mcp-master.toml`
**Database**: `/srv/mcp-master/master.db`
**Certs**: `/srv/mcp-master/certs/{cert,key,ca}.pem`
  - SAN: `rift.scylla-hammerhead.ts.net`, `mcp-master.svc.mcp.metacircular.net`, IP `100.95.252.120`
**Service token**: `/srv/mcp-master/mcias-token` (MCIAS identity: `mcp-master`, expires 2027-04-03)
**Listen**: `100.95.252.120:9555`
**Bootstrap nodes**: rift (master), svc (edge)

**Status**: Running via `doas` (ad-hoc). NixOS read-only /etc prevents
direct systemd unit creation — needs NixOS config update for persistent
service.

**Tested**:
- `mcp deploy mcq` → master places on rift, forwards to agent ✓
- `mcp undeploy mcq` → master forwards to agent, cleans up placement ✓
- `mcp ps` → fleet-wide status through agents ✓
- `mcp node list` → both nodes visible with versions ✓

## CLI Config Changes (vade)

Updated `~/.config/mcp/mcp.toml`:
- Added `[master]` section: `address = "rift.scylla-hammerhead.ts.net:9555"`
- All node addresses switched to Tailscale DNS names
- Added CA cert path

## Known Limitations
- ~~mc-proxy socket permissions will reset on restart~~ **FIXED**: mc-proxy
  now creates the socket with 0660 (was 0600). Committed to mc-proxy master.
- Master runs ad-hoc via `doas` on rift. Needs NixOS systemd config for
  persistent service (rift has read-only /etc).
- DNS registration not configured on master (MCNS config omitted for now).
- Edge routing not yet tested end-to-end through master (svc cert provisioning
  not configured).
- The TLS cert was issued from the local CA directly, not via Metacrypt API.
  Should be re-issued via Metacrypt once the agent has cert provisioning.
- Container runtime is set to `podman` but podman is not installed on svc
  (Docker is). Edge agents don't run containers so this is benign.
- Metacrypt and MCNS integrations not configured (not needed for edge role).
