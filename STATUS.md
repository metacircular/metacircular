# Metacircular Platform Status

Last updated: 2026-03-25

## Platform Overview

One node operational (**rift**), running core infrastructure services as
containers fronted by MC-Proxy. MCIAS runs separately (not on rift).
MCP and full MCNS are not yet built.

## Service Status

| Service | Version | SDLC Phase | Deployed | Node |
|---------|---------|------------|----------|------|
| MCIAS | v1.7.0 | Maintenance | Yes | (separate) |
| Metacrypt | untagged | Testing | Yes | rift |
| MC-Proxy | untagged | Maintenance | Yes | rift |
| MCR | untagged | Ready to deploy | No | — |
| MCAT | untagged | Complete | Unknown | — |
| MCDSL | v0.1.0 | Stable | N/A (library) | — |
| MCNS | untagged | Precursor | Yes | rift |
| MCP | — | Not started | No | — |

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

- **Version:** Untagged.
- **Phase:** Testing. All four engine types implemented (CA, SSH CA, transit,
  user-to-user). Active work on integration test coverage.
- **Deployment:** Running on rift as a container, fronted by MC-Proxy on
  ports 443 (web, L7), 8443 (API, L4), and 9443 (gRPC, L4).
- **Recent work:** ACME integration tests (60+ tests), mcdsl migration,
  security audit fixes.
- **Artifacts:** systemd units (service + web + backup timer), Docker Compose
  (standard + rift), install script, example configs.

### MC-Proxy — TLS Proxy and Router

- **Version:** Untagged. Phases 1-8 complete.
- **Phase:** Maintenance. Stable and actively routing traffic on rift.
- **Deployment:** Running on rift. Fronts Metacrypt, MCR, and sgard on ports
  443, 8443, and 9443. Prometheus metrics on 127.0.0.1:9091.
- **Recent work:** MCR route additions, Nix flake, L7 backend cert handling,
  Prometheus metrics, L7 policies.
- **Artifacts:** systemd units (service + backup timer), Docker Compose
  (standard + rift), install and backup scripts, rift config.

### MCR — Container Registry

- **Version:** Untagged. Phase 13 (deployment artifacts) complete.
- **Phase:** Ready to deploy. Implementation complete, deployment artifacts
  written, next step is first deploy to rift.
- **Deployment:** Not yet deployed. DNS record and MC-Proxy routes are
  pre-configured on rift.
- **Recent work:** Dockerfile fixes, server wiring, OCI route mounting,
  deployment artifact creation.
- **Artifacts:** systemd units (service + web + backup timer), Dockerfiles
  (API + web), Docker Compose (rift), install script, rift config.

### MCAT — Login Policy Tester

- **Version:** Untagged.
- **Phase:** Complete. Diagnostic tool, not core infrastructure.
- **Deployment:** Available for ad-hoc use. Lightweight tool for testing
  MCIAS login policy rules.
- **Recent work:** Migrated to mcdsl for auth, config, CSRF, and web.
- **Artifacts:** systemd unit, install script, example config.

### MCDSL — Standard Library

- **Version:** v0.1.0.
- **Phase:** Stable. All 9 packages implemented and tested (87 tests). Being
  adopted across the platform.
- **Deployment:** N/A (Go library, imported by other services).
- **Packages:** auth, db, config, httpserver, grpcserver, csrf, web, health,
  archive.
- **Adoption:** mcat, mc-proxy, and mcr migrated. metacrypt and mcias
  pending.

### MCNS — Networking Service

- **Version:** Untagged.
- **Phase:** Precursor. CoreDNS instance serving internal zones until the
  full MCNS service is built.
- **Deployment:** Running on rift via Docker Compose. Serves two zones:
  `mcp.metacircular.net` (node addresses) and
  `svc.mcp.metacircular.net` (service addresses).
- **Records:** rift node, metacrypt, mcr, sgard services.
- **Artifacts:** Corefile, zone files, Docker Compose (rift).

### MCP — Control Plane

- **Phase:** Not started. Design documented in `docs/metacircular.md`.
- **Blocked by:** Nothing — MCIAS, Metacrypt, MCR, MC-Proxy, and MCNS
  (precursor) are all available. MCP is the next major project.

## Node Inventory

| Node | Address (LAN) | Address (Tailscale) | Role |
|------|---------------|---------------------|------|
| rift | 192.168.88.181 | 100.95.252.120 | Infrastructure services |

## Rift Port Map

| Port | Protocol | Services |
|------|----------|----------|
| 443 | L7 (TLS termination) | metacrypt-web, mcr-web |
| 8443 | L4 (SNI passthrough) | metacrypt API, mcr API |
| 9443 | L4 (SNI passthrough) | metacrypt gRPC, mcr gRPC, sgard |
| 9091 | HTTP (loopback) | MC-Proxy Prometheus metrics |
