# SVC Public Edge Deployment Plan

## Goal

Deploy mc-proxy and MCNS on `svc.metacircular.net` to make Metacircular
platform services publicly accessible and delegate internal DNS zones.

## Current State

- **svc** (`svc.metacircular.net`): Debian 12, public IP, on tailnet as
  `svc`. Runs MCIAS only (systemd).
- **rift** (`192.168.88.181` / `100.95.252.120`): NixOS, LAN + tailnet.
  Runs metacrypt, mc-proxy, mcr, mcns, mcp (all containers under mcp user).
- **DNS**: `metacircular.net` managed at Hurricane Electric. Internal zones
  (`mcp.metacircular.net`, `svc.mcp.metacircular.net`) served by MCNS on
  rift, resolved via systemd-resolved split DNS on LAN clients.

## Architecture After Deployment

```
Internet
  │
  ▼
svc.metacircular.net (public IP)
  ├── MCIAS (:8443) ─── existing, no change
  ├── mc-proxy (:443) ── NEW: L7 TLS termination
  │     └── metacrypt.metacircular.net → rift metacrypt-web (Tailscale)
  └── MCNS (:53) ─────── NEW: authoritative DNS for delegation
        └── mcp.metacircular.net, svc.mcp.metacircular.net zones

         ┌─── Tailscale ───┐
         ▼                  │
rift (100.95.252.120)       │
  ├── mc-proxy (:443/:8443/:9443)
  ├── metacrypt-web (:18080) ◄── forwarded from svc mc-proxy
  ├── metacrypt-api (:18443)
  ├── mcr, mcns, mcp, etc.
```

## Phase 1: Build Binaries

Both mc-proxy and mcns need to be built for svc (x86_64 Linux, Debian).
Since svc doesn't have Go, build on vade and scp.

```bash
# On vade
cd ~/src/metacircular/mc-proxy
CGO_ENABLED=0 make mc-proxy
scp mc-proxy svc:/tmp/

cd ~/src/metacircular/mcns
CGO_ENABLED=0 make mcns
scp mcns svc:/tmp/
```

## Phase 2: TLS Certificates

Need a cert for `metacrypt.metacircular.net` (used by mc-proxy on svc for
L7 termination). Also need a cert for MCNS management API on svc.

**Option A**: Issue from Metacrypt CA via the API.
**Option B**: Generate self-signed from the wntrmute CA on rift.

Certs needed:
- `metacrypt.metacircular.net` — for mc-proxy L7 route
- `mcns-svc.svc.mcp.metacircular.net` — for MCNS management API (or self-signed)

The wntrmute CA cert (`wntrmute-ca.pem`) must be distributed to external
users who want to trust the platform's TLS.

## Phase 3: Deploy mc-proxy on svc

### 3.1 Install

```bash
# On svc (as root)
useradd --system --no-create-home --shell /usr/sbin/nologin mc-proxy
mkdir -p /srv/mc-proxy/{certs,backups}
chown -R mc-proxy:mc-proxy /srv/mc-proxy
chmod 0700 /srv/mc-proxy

install -m 0755 /tmp/mc-proxy /usr/local/bin/mc-proxy
```

### 3.2 Configuration

Create `/srv/mc-proxy/mc-proxy.toml`:

```toml
[database]
path = "/srv/mc-proxy/mc-proxy.db"

# Public HTTPS — L7 termination for metacrypt web UI.
# Backend is metacrypt-web on rift via Tailscale, re-encrypted with TLS.
# mc-proxy skips backend cert verification (InsecureSkipVerify: trusted
# internal backend), so SNI mismatch between the public hostname and
# metacrypt-web's cert is not an issue.
[[listeners]]
addr = ":443"

  [[listeners.routes]]
  hostname    = "metacrypt.metacircular.net"
  backend     = "100.95.252.120:18080"
  mode        = "l7"
  tls_cert    = "/srv/mc-proxy/certs/metacrypt.metacircular.net.pem"
  tls_key     = "/srv/mc-proxy/certs/metacrypt.metacircular.net.key"
  backend_tls = true

# Firewall — internet-facing, aggressive blocking.
[firewall]
geoip_db          = "/srv/mc-proxy/GeoLite2-Country.mmdb"
blocked_ips       = []
blocked_cidrs     = []
blocked_countries = []
rate_limit        = 20
rate_window       = "10s"

# No gRPC admin API on svc (managed via config file only).

[proxy]
connect_timeout  = "5s"
idle_timeout     = "300s"
shutdown_timeout = "30s"

[log]
level = "info"
```

Notes:
- `backend_tls = true` — metacrypt is security-sensitive; all hops must
  use TLS. mc-proxy's backend transport uses `InsecureSkipVerify: true`
  for trusted internal backends, so the public hostname / backend cert
  mismatch is not a problem.
- **Rift prerequisite**: metacrypt-web must serve HTTPS and be exposed
  on rift's Tailscale interface. Currently mapped to `127.0.0.1:18080`.
  Two changes needed:
  1. Configure `web.tls_cert` and `web.tls_key` in metacrypt's config
     (metacrypt-web already supports TLS if these are set).
  2. Update the MCP service definition to expose the port on Tailscale:
     change `127.0.0.1:18080:8080` to `100.95.252.120:18080:8080`.
  3. Redeploy with `mcp deploy metacrypt`.

### 3.3 GeoIP Database

Download MaxMind GeoLite2-Country database (free, requires account):
```bash
# Download and install
wget -O /srv/mc-proxy/GeoLite2-Country.mmdb <maxmind-url>
chown mc-proxy:mc-proxy /srv/mc-proxy/GeoLite2-Country.mmdb
```

Populate `blocked_countries` with appropriate list after initial
deployment. Start empty, add based on access logs.

### 3.4 User Agent Blocking

UA blocking is L7-only and managed via the gRPC admin API or direct
database insertion. Common bots to block:
- Scanners: `zgrab`, `masscan`, `Nuclei`, `httpx`
- AI scrapers: `GPTBot`, `CCBot`, `ClaudeBot`, `Bytespider`

Since we're not exposing the gRPC admin socket on svc, UA policies can
be pre-seeded in the SQLite database or added later via gRPC.

### 3.5 Systemd Unit

Copy from mc-proxy deploy, adapting for svc:

```ini
[Unit]
Description=MC-Proxy TLS Router (svc)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mc-proxy
Group=mc-proxy
ExecStart=/usr/local/bin/mc-proxy server --config /srv/mc-proxy/mc-proxy.toml
Restart=on-failure
RestartSec=5

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
ReadWritePaths=/srv/mc-proxy
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

### 3.6 Enable

```bash
systemctl daemon-reload
systemctl enable --now mc-proxy
systemctl enable --now mc-proxy-backup.timer
```

## Phase 4: Deploy MCNS on svc

### 4.1 Install

```bash
useradd --system --no-create-home --shell /usr/sbin/nologin mcns
mkdir -p /srv/mcns/{certs,backups}
chown -R mcns:mcns /srv/mcns
chmod 0700 /srv/mcns

install -m 0755 /tmp/mcns /usr/local/bin/mcns
```

### 4.2 Configuration

Create `/srv/mcns/mcns.toml`:

```toml
[server]
listen_addr = ":8444"
grpc_addr   = ":9444"
tls_cert    = "/srv/mcns/certs/mcns.pem"
tls_key     = "/srv/mcns/certs/mcns.key"

[database]
path = "/srv/mcns/mcns.db"

[dns]
listen_addr = ":53"
upstreams   = ["1.1.1.1:53", "8.8.8.8:53"]

[mcias]
server_url   = "https://127.0.0.1:8443"
ca_cert      = ""
service_name = "mcns"
tags         = []

[log]
level = "info"
```

Notes:
- DNS on :53 (public-facing for NS delegation)
- Management API on :8444/:9444 (different from rift's 8443/9443 to
  avoid confusion; not publicly exposed)
- MCIAS at localhost since MCIAS runs on svc

### 4.3 Seed Data

MCNS creates the database and runs migrations on first start. Migration
v2 seeds the same zones and records as rift's instance:
- `svc.mcp.metacircular.net` zone (metacrypt, mcr, sgard, mcp-agent)
- `mcp.metacircular.net` zone (rift, ns)

The `ns` record needs updating: for public delegation, `ns` should also
have svc's public IP so external resolvers can reach it.

After first start, add svc's public IP to the ns record via the API:
```bash
# Get svc's public IP
SVC_IP=$(curl -s ifconfig.me)

# Add ns record pointing to svc
curl -k -X POST https://localhost:8444/v1/zones/mcp.metacircular.net/records \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"name\": \"ns\", \"type\": \"A\", \"value\": \"$SVC_IP\", \"ttl\": 300}"
```

### 4.4 Systemd Unit

```ini
[Unit]
Description=MCNS Authoritative DNS (svc)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=mcns
Group=mcns
ExecStart=/usr/local/bin/mcns server --config /srv/mcns/mcns.toml
Restart=on-failure
RestartSec=5

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
ReadWritePaths=/srv/mcns
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

### 4.5 Enable

```bash
systemctl daemon-reload
systemctl enable --now mcns
systemctl enable --now mcns-backup.timer
```

## Phase 5: DNS Changes at Hurricane Electric

### 5.1 A Record for metacrypt

```
metacrypt.metacircular.net.  A  <svc-public-ip>
```

### 5.2 NS Delegation for mcp.metacircular.net

```
mcp.metacircular.net.        NS   ns.mcp.metacircular.net.
ns.mcp.metacircular.net.     A    <svc-public-ip>
```

This tells external resolvers: "for anything under mcp.metacircular.net,
ask ns.mcp.metacircular.net (which is svc)." MCNS on svc then serves the
authoritative answers.

**Note**: NS delegation requires the glue record (the A record for ns)
to be in the parent zone at HE. Both records must be added.

### 5.3 Verification

```bash
# From an external machine (not on tailnet)
dig metacrypt.metacircular.net
dig rift.mcp.metacircular.net
dig metacrypt.svc.mcp.metacircular.net

# Test the web UI
curl -k https://metacrypt.metacircular.net/
```

## Phase 6: Verification

### 6.1 mc-proxy on svc

```bash
# From vade
curl -k https://metacrypt.metacircular.net/
# Should show metacrypt web UI (login page)

# Check mc-proxy logs
ssh svc journalctl -u mc-proxy -f
```

### 6.2 MCNS on svc

```bash
# Direct query to svc
host rift.mcp.metacircular.net <svc-public-ip>
host metacrypt.svc.mcp.metacircular.net <svc-public-ip>

# Forwarding works
host google.com <svc-public-ip>

# Delegation (from external, after HE changes propagate)
host rift.mcp.metacircular.net 8.8.8.8
```

### 6.3 GeoIP and UA blocking

```bash
# Check firewall is active
ssh svc journalctl -u mc-proxy | grep -i "blocked\|firewall"

# Test UA blocking (after adding policies)
curl -k -A "zgrab/0.x" https://metacrypt.metacircular.net/
# Should get 403 Forbidden
```

## Open Questions (Resolved)

1. **mc-proxy backend routing**: RESOLVED. mc-proxy uses
   `InsecureSkipVerify: true` for backend TLS — no SNI verification, no
   cert hostname checking. svc mc-proxy connects directly to
   metacrypt-web on rift's Tailscale IP with `backend_tls = true`. No
   SNI mismatch issue.

2. **GeoIP country list**: Start empty, add based on access logs after
   deployment. Review periodically.

3. **Record sync between MCNS instances**: Manual for now. Both instances
   get the same seed data from migration v2. Future: AXFR/IXFR zone
   transfers from rift (primary) to svc (secondary).

4. **MCIAS on svc for MCNS auth**: MCNS config uses
   `server_url = "https://127.0.0.1:8443"`. MCIAS cert is for
   `svc.metacircular.net`. Set `ca_cert` to the wntrmute CA cert and
   check if MCIAS cert includes localhost as a SAN. If not, either add
   localhost SAN or use `svc.metacircular.net` as the server_url.

5. **Backup coordination**: Stagger timers. mc-proxy at 02:00 UTC,
   mcns at 02:15 UTC, mcias at 02:30 UTC (check existing mcias timer).

## Rollback

If anything goes wrong:
1. `systemctl stop mc-proxy mcns` on svc
2. Remove DNS records at HE
3. All traffic returns to existing paths (LAN via rift, MCIAS via svc)

No changes are made to rift's configuration. This deployment is purely
additive on svc.

## Security Note: Backend TLS Verification

mc-proxy uses `InsecureSkipVerify: true` for backend TLS connections.
This is acceptable because backend IPs are hardcoded operator-configured
Tailscale addresses — WireGuard provides cryptographic peer
authentication, so TLS cert hostname validation is redundant.

**Reconsider later**: when services have publicly-facing FQDNs (e.g.,
`metacrypt.metacircular.net`), issue certs that include both the public
FQDN and the internal name (`metacrypt.svc.mcp.metacircular.net`) as
SANs. Then mc-proxy could enable backend cert verification for
defense-in-depth. This is a low-priority improvement — Tailscale's
identity guarantee is strong.

## Future Considerations

- **Let's Encrypt**: mc-proxy has ACME on its roadmap. Once implemented,
  public-facing routes could use trusted certs automatically.
- **MCIAS migration to rift**: Once mc-proxy on svc handles public
  routing, MCIAS could move to rift. svc mc-proxy would forward auth
  traffic too. Eliminates the separate VPS dependency for MCIAS.
- **Additional public services**: mcr web UI, MCP status dashboard, etc.
  Each would be an additional L7 route on svc's mc-proxy.
