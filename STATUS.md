# Metacircular Platform Status

Last updated: 2026-03-26

## Platform Overview

One node operational (**rift**), running core infrastructure services as
containers fronted by MC-Proxy. MCIAS runs separately (not on rift).
Bootstrap phases 0–4 complete (MCIAS, Metacrypt, MC-Proxy, MCR all
operational). MCP is deployed and managing all platform containers. Full MCNS is not
yet built.

## Service Status

| Service | Version | SDLC Phase | Deployed | Node |
|---------|---------|------------|----------|------|
| MCIAS | v1.7.0 | Maintenance | Yes | (separate) |
| Metacrypt | v1.0.0 | Production | Yes | rift |
| MC-Proxy | v1.0.0 | Maintenance | Yes | rift |
| MCR | v1.0.0 | Production | Yes | rift |
| MCAT | v1.0.0 | Complete | Unknown | — |
| MCDSL | v1.0.0 | Stable | N/A (library) | — |
| MCNS | v1.0.0 | Built, pending deploy | No | rift (planned) |
| MCP | v0.1.0 | Production | Yes | rift |
| MCDeploy | v0.1.0 | Active dev | N/A (CLI tool) | — |

## Service Details

### MCIAS — Identity and Access Service

- **Version:** v1.7.0 (client library: clients/go/v0.1.0)
- **Phase:** Maintenance. Phases 0-14 complete. Feature-complete with active
  refinement.
- **Deployment:** Running in production. All other services authenticate
  against it.
- **Recent work:** WebAuthn/FIDO2 passkeys, TOTP 2FA, service-context login
  policies, Nix flake for CLI tools.
- **Artifacts:** systemd units (service + backup timer), install script,
  Dockerfile, example configs.

### Metacrypt — Cryptographic Service Engine

- **Version:** v1.0.0.
- **Phase:** Production. All four engine types implemented (CA, SSH CA, transit,
  user-to-user). Active work on integration test coverage.
- **Deployment:** Running on rift as a container, fronted by MC-Proxy on
  ports 443 (web, L7), 8443 (API, L4), and 9443 (gRPC, L4).
- **Recent work:** ACME integration tests (60+ tests), mcdsl migration,
  security audit fixes.
- **Artifacts:** systemd units (service + web + backup timer), Docker Compose
  (standard + rift), install script, example configs.

### MC-Proxy — TLS Proxy and Router

- **Version:** v1.0.0. Phases 1-8 complete.
- **Phase:** Maintenance. Stable and actively routing traffic on rift.
- **Deployment:** Running on rift. Fronts Metacrypt, MCR, and sgard on ports
  443, 8443, and 9443. Prometheus metrics on 127.0.0.1:9091.
- **Recent work:** MCR route additions, Nix flake, L7 backend cert handling,
  Prometheus metrics, L7 policies.
- **Artifacts:** systemd units (service + backup timer), Docker Compose
  (standard + rift), install and backup scripts, rift config.

### MCR — Container Registry

- **Version:** v1.0.0. All implementation phases complete.
- **Phase:** Production. Deployed on rift, serving container images.
- **Deployment:** Running on rift as two containers (mcr API + mcr-web),
  fronted by MC-Proxy on ports 443 (web, L7), 8443 (API, L4), and
  9443 (gRPC, L4). Metacrypt is already pulling images from MCR.
- **Recent work:** Manifest push bug fix (LastInsertId unreliable after
  upsert), structured slog error logging in OCI handlers, first production
  deploy, Dockerfile fixes, server wiring, OCI route mounting.
- **Artifacts:** systemd units (service + web + backup timer), Dockerfiles
  (API + web), Docker Compose (rift), install script, rift config.

### MCAT — Login Policy Tester

- **Version:** v1.0.0.
- **Phase:** Complete. Diagnostic tool, not core infrastructure.
- **Deployment:** Available for ad-hoc use. Lightweight tool for testing
  MCIAS login policy rules.
- **Recent work:** Migrated to mcdsl for auth, config, CSRF, and web.
- **Artifacts:** systemd unit, install script, example config.

### MCDSL — Standard Library

- **Version:** v1.0.0.
- **Phase:** Stable. All 9 packages implemented and tested (87 tests). Being
  adopted across the platform.
- **Deployment:** N/A (Go library, imported by other services).
- **Packages:** auth, db, config, httpserver, grpcserver, csrf, web, health,
  archive.
- **Adoption:** mcat, mc-proxy, and mcr migrated. metacrypt and mcias
  pending.

### MCNS — Networking Service

- **Version:** v1.0.0.
- **Phase:** Implementation complete, pending deployment. Custom Go DNS
  server replacing CoreDNS precursor. Authoritative DNS with SQLite-backed
  zone/record storage and gRPC+REST management API.
- **Deployment:** Not yet deployed (replacing CoreDNS on rift).
- **Features:** A, AAAA, CNAME records; CNAME exclusivity; upstream
  forwarding with caching; MCIAS auth; SOA auto-serial.
- **Artifacts:** Dockerfile, Docker Compose (rift), example config, proto
  definitions.

### MCP — Control Plane

- **Version:** v0.1.0.
- **Phase:** Production. Phases 0-4 complete. Deployed to rift, managing all
  platform containers.
- **Deployment:** Running on rift. Agent as systemd service under `mcp` user
  with rootless podman. Manages metacrypt, mc-proxy, mcr, and mcns containers.
- **Architecture:** Two components — `mcp` CLI (thin client on vade) and
  `mcp-agent` (per-node daemon with SQLite registry, podman management,
  monitoring with drift/flap detection). gRPC-only (no REST).
- **Recent work:** Full v1 implementation (12 RPCs, 15 CLI commands),
  deployment to rift, container migration from kyle→mcp user, service
  definition authoring.
- **Artifacts:** systemd service (NixOS), TLS cert from Metacrypt, service
  definition files, design docs.

### MCDeploy — Deployment CLI

- **Version:** v0.1.0.
- **Phase:** Active development. Tactical bridge tool for deploying services
  while MCP is being built.
- **Deployment:** N/A (local CLI tool, not a server).
- **Recent work:** Initial implementation, Nix flake.
- **Description:** Single-binary CLI that shells out to podman/ssh/scp/git
  for build, push, deploy, cert renewal, and status. TOML-configured.

## Node Inventory

| Node | Address (LAN) | Address (Tailscale) | Role |
|------|---------------|---------------------|------|
| rift | 192.168.88.181 | 100.95.252.120 | Infrastructure services |

## Rift Port Map

| Port | Protocol | Services |
|------|----------|----------|
| 53 | DNS (LAN + Tailscale) | mcns-coredns |
| 443 | L7 (TLS termination) | metacrypt-web, mcr-web |
| 8080 | HTTP (all interfaces) | exod |
| 8443 | L4 (SNI passthrough) | metacrypt API, mcr API |
| 9090 | HTTP (all interfaces) | exod |
| 9443 | L4 (SNI passthrough) | metacrypt gRPC, mcr gRPC, sgard |
| 9091 | HTTP (loopback) | MC-Proxy Prometheus metrics |

Non-platform services also running on rift: **exod** (ports 8080/9090),
**sgardd** (port 19473, fronted by MC-Proxy on 9443).
