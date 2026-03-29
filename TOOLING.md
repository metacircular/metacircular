# Metacircular Platform Tooling

CLI tools for interacting with the Metacircular platform. All tools are
Go binaries built with `CGO_ENABLED=0` and installed via Nix flakes.

## Tool Inventory

| Tool | Project | Purpose | Install target |
|------|---------|---------|---------------|
| `mcp` | mcp | Control plane CLI — deploy, status, lifecycle, file transfer | vade, orion |
| `mcp-agent` | mcp | Control plane agent — per-node container management daemon | rift (systemd) |
| `mciasctl` | mcias | MCIAS admin CLI — accounts, tokens, policies | vade, orion, rift |
| `mciasgrpcctl` | mcias | MCIAS gRPC debug CLI | vade, orion, rift |
| `mcproxyctl` | mc-proxy | MC-Proxy admin CLI — routes, firewall, status | vade, orion, rift |
| `mcrctl` | mcr | MCR admin CLI — repositories, policies, audit | vade, orion, rift |
| `mcdeploy` | mcdeploy | Deployment CLI — deprecated, superseded by MCP (archived) | — |

### Server-only binaries (not installed as tools)

These run inside containers and are not installed on operator workstations:

| Binary | Project | Purpose |
|--------|---------|---------|
| `mciassrv` | mcias | MCIAS server |
| `metacrypt` | metacrypt | Metacrypt server (includes init, unseal, status, snapshot subcommands) |
| `metacrypt-web` | metacrypt | Metacrypt web UI server |
| `mcrsrv` | mcr | MCR API server |
| `mcr-web` | mcr | MCR web UI server |
| `mc-proxy` | mc-proxy | TLS proxy server |
| `mcns` | mcns | DNS server |
| `mcat` | mcat | Login policy tester web app |

## Installation

All tools are packaged as Nix flakes and installed as system packages
via `mcpkg.nix` in the NixOS configuration. Adding a tool:

1. Create or update `flake.nix` in the project repo.
2. Add the flake as an input in `/home/kyle/src/nixos/flake.nix`.
3. Add the package to `configs/mcpkg.nix`.
4. `sudo nixos-rebuild switch` on target machines.

### Nix flake conventions

- Channel: `nixos-25.11` (match the system nixpkgs).
- Build: `pkgs.buildGoModule` with `vendorHash = null` (vendored deps).
- ldflags: `-s -w -X main.version=${version}`.
- `subPackages`: list only the client binaries, not servers.
- `system`: `x86_64-linux` (all machines are x86_64).

### MCP agent

The `mcp-agent` is a special case: it runs as a systemd service on
managed nodes (not as a container, since it manages containers). Its
flake exposes both `mcp` (client) and `mcp-agent` (server). The NixOS
config in `configs/mcp.nix` references the Nix package path for the
agent binary.

## Flake status

| Project | flake.nix | Packages | In mcpkg.nix | Notes |
|---------|-----------|----------|-------------|-------|
| mcias | Yes | mciasctl, mciasgrpcctl | Yes | |
| mc-proxy | Yes | mcproxyctl | Yes | |
| mcr | Yes | mcrctl | Yes | |
| mcdeploy | Yes | mcdeploy | No | Deprecated, remove from mcpkg.nix |
| mcp | Yes | mcp, mcp-agent | Yes | Agent also used by mcp.nix systemd unit |
| mcns | No | — | No | Server-only, no client tool yet |
| metacrypt | No | — | No | Server-only, no client tool yet |
| mcat | No | — | No | Server-only, no client tool yet |
| mcdsl | No | — | No | Library, no binaries |
