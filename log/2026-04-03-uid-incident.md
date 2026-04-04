# Incident Report: UID Change Cascading Failure

**Date**: 2026-04-03
**Duration**: ~2 hours (08:45–10:45 PDT)
**Severity**: Full platform outage on rift (all containers lost)
**Root cause**: Changing the `mcp` system user UID from 995 to 850

## Timeline

### Background

Orion was being provisioned as a new worker node. Its NixOS config
imports `mcp.nix` which pins the mcp user to UID 995. On orion, UID 995
was already assigned to the `sshd` user, causing a UID collision:

```
uid=995(sshd) gid=988(mcp) groups=988(mcp),62(systemd-journal),992(sshd)
```

Both `sshd` and `mcp` had UID 995 on orion. The `newuidmap` tool
rejected rootless podman operations because the calling process's UID
(995) belonged to `sshd`, not `mcp`, in `/etc/passwd`.

### The UID Change

To resolve the collision, `mcp.nix` was updated to pin UID 850 (in
the 800–899 range, empty on all nodes). Both rift and orion were
rebuilt with `nixos-rebuild switch`.

**Problem 1: NixOS doesn't change UIDs for existing users.** The
rebuild created the NixOS config with `uid = 850` but the existing
`mcp` user on both nodes kept UID 995. Manual `usermod -u 850 mcp`
was required on each node.

**Problem 2: Rootless podman caches the UID everywhere.**
- Podman's SQLite database (`db.sql`) stores absolute paths like
  `/run/user/995/libpod/tmp` and `/run/user/995/containers`
- The systemd user session (`/run/user/995/`) is tied to the UID
- subuid/subgid mappings reference the user by name but the kernel
  checks the actual UID
- Container storage overlay directories have file ownership based on
  the old UID namespace mapping (995 → 100000)

After changing the UID, `podman` operations failed with:
```
newuidmap: write to uid_map failed: Operation not permitted
```

### The Reboot

Rift was rebooted to get a clean systemd user session for UID 850.
The reboot succeeded, but **all containers were gone**:

```
$ podman ps -a
(empty)
```

Podman's database was recreated fresh on boot because the old database
referenced paths under `/run/user/995/` which no longer existed. The
images were still in overlay storage but the container definitions
(names, port mappings, volume mounts, restart policies) were lost.

### DNS Collapse

MCNS (the authoritative DNS server for `.svc.mcp.metacircular.net`)
ran as a container on rift. When all containers were lost, DNS
resolution broke:

- `mcq.svc.mcp.metacircular.net` → no answer
- MCNS also served as a recursive resolver for the LAN
- `google.com` → NXDOMAIN on machines using MCNS as their resolver

Tailscale DNS (MagicDNS) was also affected because resolved's global
DNS config pointed to MCNS. Tailscale itself remained functional
(its coordination servers are external), but hostname resolution via
Tailscale DNS names failed.

The operator turned off Tailscale on vade (the workstation) because
Tailscale's MagicDNS was routing ALL DNS queries through the broken
MCNS resolver — external services including Claude Code and Gitea
were unreachable. Disabling Tailscale was the only way to restore
external DNS resolution. However, this also broke connectivity to
rift since the MCP agent binds to the Tailnet IP only
(`100.95.252.120:9444`).

### Recovery

**Step 1**: Turn Tailscale back on (on both rift and vade). Tailscale
connectivity works without MCNS — MagicDNS uses Tailscale's own
servers for `.ts.net` names.

**Step 2**: Start MCNS manually via `podman run`. The image was cached
in overlay storage. MCNS needed explicit port bindings (not `--network
host`) because systemd-resolved holds port 53 on localhost:

```bash
podman run -d --name mcns --restart unless-stopped \
  -p 192.168.88.181:53:53/tcp -p 192.168.88.181:53:53/udp \
  -p 100.95.252.120:53:53/tcp -p 100.95.252.120:53:53/udp \
  -p 127.0.0.1:38443:8443 \
  -v /srv/mcns:/srv/mcns \
  mcr.svc.mcp.metacircular.net:8443/mcns:v1.2.0 \
  server --config /srv/mcns/mcns.toml
```

DNS resolution restored within seconds.

**Step 3**: Start remaining services manually via `podman run`. Images
were all cached. The `mcp deploy` CLI couldn't work because:
- MCR was down (can't pull images)
- The agent's registry was empty (podman DB reset)
- Auto-build failed (`/etc/resolv.conf` permission denied in build
  containers)

Each service was started with explicit `podman run` commands matching
the service definitions in `~/.config/mcp/services/*.toml`.

**Step 4**: Fix file ownership for rootless podman. Files in `/srv/*`
were owned by UID 850 (the mcp user on the host). Inside containers,
UID 0 (root) maps to host UID 850 via subuid. But:

- `podman unshare chown -R 0:0 /srv/<service>` translated ownership
  to match the container's user namespace
- SQLite's `PRAGMA journal_mode = WAL` requires creating WAL/SHM files
  in the database directory
- modernc.org/sqlite calls `fchmod()` on the database file, which is
  denied inside rootless podman user namespaces (even for UID 0 in the
  namespace)

**Step 5**: Delete and recreate SQLite databases. The `fchmod` denial
was fatal for MCR and Metacrypt. The fix:

```bash
# Stop the container
podman stop metacrypt-api
# Delete the database (WAL and SHM too)
rm -f /srv/metacrypt/metacrypt.db*
# Restart — the service recreates the database
podman start metacrypt-api
```

The `fchmod` error still occurs on the newly created database but is
non-fatal — the service logs a warning and continues.

**Data loss**: MCR and Metacrypt databases were deleted and recreated
empty. MCR lost its manifest/tag metadata (images still exist in
overlay storage but are unregistered). Metacrypt lost its CA state
(encrypted keys, issued certs tracking). Other services (mcq, mcdoc,
etc.) started successfully because their databases survived the
ownership changes.

## Root Causes

1. **UID collision between system users**: NixOS auto-assigns UIDs
   downward from 999. Pinning UID 995 for mcp collided with sshd on
   orion.

2. **Rootless podman's deep UID dependency**: Changing a user's UID
   after rootless podman has been used requires:
   - Updating podman's internal database paths
   - Recreating the systemd user session
   - Fixing subuid/subgid mappings
   - Fixing overlay storage ownership
   - Fixing service data file ownership
   - None of these happen automatically

3. **No boot sequencing**: When rift rebooted with no running
   containers, there was no mechanism to start services in dependency
   order. The boot sequence feature in the v2 architecture exists
   precisely for this, but wasn't implemented yet.

4. **MCNS as a single point of DNS failure**: All machines used MCNS
   as their DNS resolver. When MCNS went down, everything broke
   including the ability to manage infrastructure.

5. **modernc.org/sqlite `fchmod` in rootless podman**: The SQLite
   library calls `fchmod()` on database files, which is denied inside
   rootless podman user namespaces. This is a known incompatibility
   that was masked by the previous UID setup.

## Lessons Learned

1. **Never change a rootless podman user's UID.** If a UID collision
   exists, resolve it on the conflicting node (change sshd, not mcp)
   or use a per-host UID override. Changing the UID after podman has
   been used is destructive.

2. **DNS must not be a single point of failure.** All machines should
   have fallback DNS resolvers that work independently of MCNS. The
   NixOS config should list public resolvers (1.1.1.1, 8.8.8.8) as
   fallbacks, not just MCNS.

3. **Boot sequencing is critical.** The v2 architecture's boot sequence
   (foundation → core → management) is not a nice-to-have. Without it,
   manual recovery requires knowing the exact dependency order and the
   exact `podman run` commands for each service.

4. **The MCP agent should be able to recover containers from its
   registry.** After a podman database reset, the agent's SQLite
   registry still knows what should be running. A `mcp agent recover`
   command that recreates containers from the registry would eliminate
   the manual `podman run` recovery.

5. **Service definitions must include all runtime parameters.** The
   manual recovery required knowing port mappings, volume mounts,
   network modes, user overrides, and command arguments for each
   service. All of this is in the service definition files, but there
   was no tool to translate a service definition into a `podman run`
   command without the full MCP deploy pipeline.

6. **Tailscale MagicDNS amplifies DNS failures.** When MCNS is down
   and MagicDNS routes through it, ALL DNS breaks — not just internal
   names. Disabling Tailscale restores external DNS but loses Tailnet
   connectivity. The fix is fallback resolvers that bypass MCNS, not
   disabling Tailscale.

## Action Items

- [x] Write disaster recovery runbook → `docs/disaster-recovery.md`
- [x] Add fallback DNS resolvers to NixOS config → all nodes now have
      1.1.1.1 and 8.8.8.8 as fallbacks after MCNS
- [x] Implement `mcp agent recover` command → MCP v0.10.5. Recreates
      containers from the agent registry when podman DB is lost.
- [x] Implement boot sequencing in the agent → MCP v0.10.6.
      [[boot.sequence]] config with per-stage health checks.
- [x] Fix modernc.org/sqlite `fchmod` → was our own `os.Chmod` in
      `mcdsl/db/db.go`, not sqlite. Made best-effort in mcdsl v1.8.0.
- [x] Add multi-address support to node config → MCP v0.10.4.
      Fallback addresses tried in order when primary fails.
- [x] Stabilize mcp UID → pinned at 850 with NEVER CHANGE comment
