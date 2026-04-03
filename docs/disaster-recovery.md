# Disaster Recovery: Bootstrap from Zero

This document covers recovering the Metacircular platform when all
services on rift are down and no containers are running. It assumes:

- The machine boots and NixOS is functional
- The mcp-agent systemd service starts automatically
- Tailscale is configured and connects on boot
- Service data directories (`/srv/<service>/`) are intact on disk
- Container images are cached in podman's overlay storage

If images are NOT cached (fresh machine, disk wipe), see the
"Cold Start" section at the end.

## Prerequisites

Before starting recovery, verify:

```bash
# 1. Machine is up
hostname    # should print "rift"

# 2. Tailscale is connected
tailscale status --self
# Should show the Tailnet IP (100.95.252.120)

# 3. The mcp user exists
id mcp
# Should show uid=850(mcp) gid=850(mcp)

# 4. The agent is running
systemctl status mcp-agent
# Should be active

# 5. Images are cached
su -s /bin/sh mcp -c "XDG_RUNTIME_DIR=/run/user/850 HOME=/srv/mcp podman images" | wc -l
# Should be > 0
```

If Tailscale is not running: `doas systemctl start tailscaled && doas tailscale up`

If the agent is not running: check `/srv/mcp/mcp-agent` exists and
`/srv/mcp/mcp-agent.toml` is correct, then `doas systemctl restart mcp-agent`.

## Recovery Order

Services must be started in dependency order. Each stage must be
healthy before the next starts.

```
Stage 1 (Foundation): MCNS → DNS works
Stage 2 (Core):       mc-proxy, MCR, Metacrypt → routing + images + certs
Stage 3 (Management): mcp-master → orchestration
Stage 4 (Services):   mcq, mcdoc, mcat, kls, sgard, exo → applications
```

## Manual Recovery Commands

All commands run as the mcp user. Use this shell prefix:

```bash
# Set up the environment
export PODMAN_CMD='doas sh -c "cd /srv/mcp && XDG_RUNTIME_DIR=/run/user/850 HOME=/srv/mcp su -s /bin/sh mcp -c"'
# Or SSH as mcp directly (if SSH login is enabled):
ssh mcp@rift
```

For brevity, commands below show the `podman run` portion only. Prefix
with the environment setup above.

### Stage 1: MCNS (DNS)

MCNS must start first. Without it, no hostname resolution works.

```bash
podman run -d --name mcns --restart unless-stopped \
  -p 192.168.88.181:53:53/tcp \
  -p 192.168.88.181:53:53/udp \
  -p 100.95.252.120:53:53/tcp \
  -p 100.95.252.120:53:53/udp \
  -p 127.0.0.1:38443:8443 \
  -v /srv/mcns:/srv/mcns \
  mcr.svc.mcp.metacircular.net:8443/mcns:v1.2.0 \
  server --config /srv/mcns/mcns.toml
```

**Verify:**
```bash
dig @192.168.88.181 google.com +short
# Should return an IP address
dig @192.168.88.181 mcq.svc.mcp.metacircular.net +short
# Should return a Tailnet IP
```

**Note:** MCNS binds to specific IPs, not `0.0.0.0`, because
systemd-resolved holds port 53 on localhost. The explicit bindings
avoid the conflict.

### Stage 2: Core Infrastructure

#### mc-proxy (TLS routing)

```bash
podman run -d --name mc-proxy --restart unless-stopped \
  --network host \
  -v /srv/mc-proxy:/srv/mc-proxy \
  mcr.svc.mcp.metacircular.net:8443/mc-proxy:v1.2.2 \
  server --config /srv/mc-proxy/mc-proxy.toml
```

**Verify:** `curl -sk https://localhost:443/ 2>&1 | head -1`
(should get a response, even if 404)

#### MCR (Container Registry)

```bash
# API server
podman run -d --name mcr-api --restart unless-stopped \
  -v /srv/mcr:/srv/mcr \
  -p 127.0.0.1:28443:8443 \
  -p 127.0.0.1:29443:9443 \
  mcr.svc.mcp.metacircular.net:8443/mcr:v1.2.1 \
  server --config /srv/mcr/mcr.toml

# Web UI
podman run -d --name mcr-web --restart unless-stopped \
  --user 0:0 \
  -v /srv/mcr:/srv/mcr \
  -p 127.0.0.1:28080:8080 \
  mcr.svc.mcp.metacircular.net:8443/mcr-web:v1.3.2 \
  server --config /srv/mcr/mcr.toml
```

**If MCR fails with "chmod" or "readonly database":**
```bash
podman stop mcr-api
rm -f /srv/mcr/mcr.db /srv/mcr/mcr.db-wal /srv/mcr/mcr.db-shm
podman start mcr-api
```
This recreates the database empty. Image blobs in `/srv/mcr/layers/`
are preserved but tag metadata is lost. Re-push images to rebuild the
registry.

#### Metacrypt (PKI / Secrets)

```bash
# API server
podman run -d --name metacrypt-api --restart unless-stopped \
  -v /srv/metacrypt:/srv/metacrypt \
  -p 127.0.0.1:18443:8443 \
  -p 127.0.0.1:19443:9443 \
  mcr.svc.mcp.metacircular.net:8443/metacrypt:v1.3.1 \
  server --config /srv/metacrypt/metacrypt.toml

# Web UI
podman run -d --name metacrypt-web --restart unless-stopped \
  -v /srv/metacrypt:/srv/metacrypt \
  -p 127.0.0.1:18080:8080 \
  mcr.svc.mcp.metacircular.net:8443/metacrypt-web:v1.4.1 \
  --config /srv/metacrypt/metacrypt.toml
```

**If Metacrypt fails with "chmod" or "readonly database":**
Same fix as MCR — delete the database files. **Warning:** this loses
all encrypted secrets, issued certs tracking, and CA state. The CA
key itself is in the sealed vault (password-protected), not in SQLite.

### Stage 3: MCP Master

```bash
podman run -d --name mcp-master --restart unless-stopped \
  --network host \
  -v /srv/mcp-master:/srv/mcp-master \
  mcr.svc.mcp.metacircular.net:8443/mcp-master:v0.10.3 \
  server --config /srv/mcp-master/mcp-master.toml
```

**Verify:**
```bash
# From vade (operator workstation):
mcp node list
# Should show rift, svc, orion
```

### Stage 4: Application Services

Once the master is running, deploy applications through MCP:

```bash
mcp deploy mcq --direct
mcp deploy mcdoc --direct
mcp deploy mcat --direct
mcp deploy kls --direct
```

Or start them manually:

```bash
# MCQ
podman run -d --name mcq --restart unless-stopped \
  -v /srv/mcq:/srv/mcq \
  -p 127.0.0.1:48080:8080 -p 100.95.252.120:48080:8080 \
  mcr.svc.mcp.metacircular.net:8443/mcq:v0.4.2 \
  server --config /srv/mcq/mcq.toml

# MCDoc
podman run -d --name mcdoc --restart unless-stopped \
  -v /srv/mcdoc:/srv/mcdoc \
  -p 127.0.0.1:38080:8080 \
  mcr.svc.mcp.metacircular.net:8443/mcdoc:v0.1.0 \
  server --config /srv/mcdoc/mcdoc.toml

# MCAT
podman run -d --name mcat --restart unless-stopped \
  -v /srv/mcat:/srv/mcat \
  -p 127.0.0.1:48116:8443 \
  mcr.svc.mcp.metacircular.net:8443/mcat:v1.2.0 \
  server --config /srv/mcat/mcat.toml

# KLS
podman run -d --name kls --restart unless-stopped \
  -v /srv/kls:/srv/kls \
  -p 127.0.0.1:58080:8080 -p 100.95.252.120:58080:8080 \
  mcr.svc.mcp.metacircular.net:8443/kls:v0.2.0 \
  -f /srv/kls/kls.conf

# Sgard
podman run -d --name sgardd --restart unless-stopped \
  -v /srv/sgard:/srv/sgard \
  -p 127.0.0.1:19473:9473 \
  mcr.svc.mcp.metacircular.net:8443/sgardd:v3.2.0 \
  --repo /srv/sgard --authorized-keys /srv/sgard/authorized_keys \
  --tls-cert /srv/sgard/certs/sgard.pem --tls-key /srv/sgard/certs/sgard.key
```

## Verification Checklist

After all services are running:

```bash
# Fleet status
mcp ps
# All services should show "running"

# DNS
dig @192.168.88.181 google.com +short
dig @192.168.88.181 mcq.svc.mcp.metacircular.net +short

# MCIAS (runs on svc, should be unaffected by rift outage)
curl -sk https://mcias.metacircular.net:8443/v1/health

# MCR
curl -sk https://mcr.svc.mcp.metacircular.net:8443/v2/

# Metacrypt
curl -sk https://metacrypt.svc.mcp.metacircular.net:8443/v1/health

# Public routes via svc
curl -sk https://mcq.metacircular.net/
curl -sk https://docs.metacircular.net/
```

## Common Errors

### "chmod: operation not permitted"

modernc.org/sqlite calls `fchmod()` on database files. This is denied
inside rootless podman user namespaces. Fix:

```bash
# Delete the database and let the service recreate it
podman stop <container>
rm -f /srv/<service>/<service>.db*
podman start <container>
```

The `fchmod` error will still appear in logs as a warning but is
non-fatal for newly created databases.

### "address already in use" on port 53

systemd-resolved holds port 53 on localhost. MCNS must bind to
specific IPs, not `0.0.0.0:53`. Use explicit port bindings:
`-p 192.168.88.181:53:53 -p 100.95.252.120:53:53`

### "connection refused" to MCR

MCR is down. Images are cached locally — you can start services that
use cached images without MCR. MCR itself starts from its cached
image.

### Agent shows "error" for all nodes

Check:
1. Tailscale is running on both the CLI machine and the target node
2. The agent is listening: `ss -tlnp | grep 9444`
3. The CLI config has the correct addresses
4. TLS certs have the right SANs for the Tailnet IP

### "podman: executable file not found"

This warning appears for svc (which uses Docker, not podman). It's
benign — svc is an edge node that doesn't run containers.

## Cold Start (No Cached Images)

If the machine was wiped and no images are cached:

1. **MCIAS** runs on svc (Docker container), not rift. It should be
   unaffected by a rift failure. Verify: `ssh svc.metacircular.net
   "docker ps | grep mcias"`.

2. **Pre-stage images** by pulling from a backup or building locally:
   ```bash
   # On vade (operator workstation), build and push to a temp location
   cd ~/src/metacircular/mcns && make docker
   podman save mcr.svc.mcp.metacircular.net:8443/mcns:v1.2.0 | \
     ssh rift "podman load"
   ```
   Repeat for each service.

3. Alternatively, if another node has MCR access, push images there
   first, then pull from the running MCR instance.

## Service Reference

Quick reference for all services, their images, and critical flags:

| Service | Image | Network | Key Ports | Config Path |
|---------|-------|---------|-----------|-------------|
| mcns | mcns:v1.2.0 | bridge | 53/tcp+udp, 38443→8443 | /srv/mcns/mcns.toml |
| mc-proxy | mc-proxy:v1.2.2 | host | 443, 8443, 9443 | /srv/mc-proxy/mc-proxy.toml |
| mcr (api) | mcr:v1.2.1 | bridge | 28443→8443, 29443→9443 | /srv/mcr/mcr.toml |
| mcr (web) | mcr-web:v1.3.2 | bridge | 28080→8080 | /srv/mcr/mcr.toml |
| metacrypt (api) | metacrypt:v1.3.1 | bridge | 18443→8443, 19443→9443 | /srv/metacrypt/metacrypt.toml |
| metacrypt (web) | metacrypt-web:v1.4.1 | bridge | 18080→8080 | /srv/metacrypt/metacrypt.toml |
| mcp-master | mcp-master:v0.10.3 | host | 9555 | /srv/mcp-master/mcp-master.toml |
| mcq | mcq:v0.4.2 | bridge | 48080→8080 | /srv/mcq/mcq.toml |
| mcdoc | mcdoc:v0.1.0 | bridge | 38080→8080 | /srv/mcdoc/mcdoc.toml |
| mcat | mcat:v1.2.0 | bridge | 48116→8443 | /srv/mcat/mcat.toml |
| kls | kls:v0.2.0 | bridge | 58080→8080 | /srv/kls/kls.conf |
| sgard | sgardd:v3.2.0 | bridge | 19473→9473 | (flags, see above) |

All images are prefixed with `mcr.svc.mcp.metacircular.net:8443/`.
