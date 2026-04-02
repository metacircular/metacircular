# Metacircular Platform Status

Last updated: 2026-04-02

## Platform Overview

Two nodes operational (**rift** + **svc**), with **orion** provisioned but
offline for maintenance. Core infrastructure services run as containers on
rift, fronted by MC-Proxy. Svc operates as an MCP edge node managing
mc-proxy routing only (no containers); MCIAS runs on svc separately as a
systemd service. Bootstrap phases 0–4 complete (MCIAS, Metacrypt, MC-Proxy,
MCR all operational). MCP is deployed and managing all platform containers
on rift, with multi-node capability (svc as edge node). MCNS is deployed on
rift, serving authoritative DNS. Platform evolution Phases A–D complete
(automated port assignment, route registration, TLS cert provisioning, and
DNS registration). Phase E (multi-node expansion) is in planning, with v2
architecture in development.

## Service Status

| Service | Version | SDLC Phase | Deployed | Node |
|---------|---------|------------|----------|------|
| MCIAS | v1.10.5 | Maintenance | Yes | svc (systemd) |
| Metacrypt | v1.4.1 | Production | Yes | rift |
| MC-Proxy | v1.2.2 | Maintenance | Yes | rift |
| MCR | v1.3.2 | Production | Yes | rift |
| MCAT | v1.2.0 | Production | Yes | rift |
| MCDSL | v1.7.0 | Stable | N/A (library) | — |
| MCNS | v1.2.0 | Production | Yes | rift |
| MCDoc | v0.1.0 | Production | Yes | rift |
| MCQ | v0.4.2 | Production | Yes | rift |
| MCP | v0.9.0 | Production | Yes | rift |

## Service Details

### MCIAS — Identity and Access Service

- **Version:** v1.10.5 (client library: clients/go/v0.2.0)
- **Phase:** Maintenance. Phases 0-14 complete. Feature-complete with active
  refinement.
- **Deployment:** Running in production on svc as a systemd service. All
  other services authenticate against it.
- **Recent work:** WebAuthn/FIDO2 passkeys, TOTP 2FA, service-context login
  policies, Nix flake for CLI tools.
- **Artifacts:** systemd units (service + backup timer), install script,
  Dockerfile, example configs.

### Metacrypt — Cryptographic Service Engine

- **Version:** v1.4.1 (API v1.3.1, Web v1.4.1).
- **Phase:** Production. All four engine types implemented (CA, SSH CA, transit,
  user-to-user). Active work on integration test coverage.
- **Deployment:** Running on rift as a container, fronted by MC-Proxy on
  ports 443 (web, L7), 8443 (API, L4), and 9443 (gRPC, L4).
- **Recent work:** ACME integration tests (60+ tests), mcdsl migration,
  security audit fixes.
- **Artifacts:** systemd units (service + web + backup timer), Docker Compose
  (standard + rift), install script, example configs.

### MC-Proxy — TLS Proxy and Router

- **Version:** v1.2.2.
- **Phase:** Maintenance. Stable and actively routing traffic on rift and svc.
- **Deployment:** Running on rift. Fronts Metacrypt, MCR, and sgard on ports
  443, 8443, and 9443. Prometheus metrics on 127.0.0.1:9091. Routes persisted
  in SQLite and managed via gRPC API. Svc runs its own mc-proxy on :443 with
  public-facing routes.
- **Recent work:** Route persistence (SQLite), idempotent AddRoute (upsert),
  golangci-lint v2 compliance, module path migration to mc/ org.
- **Artifacts:** systemd units (service + backup timer), Docker Compose
  (standard + rift), install and backup scripts, rift config.

### MCR — Container Registry

- **Version:** v1.3.2 (API v1.2.1, Web v1.3.2). All implementation phases
  complete.
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

- **Version:** v1.2.0.
- **Phase:** Production. Deployed on rift as a container managed by MCP.
- **Deployment:** Running on rift. Lightweight tool for testing MCIAS login
  policy rules.
- **Recent work:** Migrated to mcdsl for auth, config, CSRF, and web.
- **Artifacts:** systemd unit, install script, example config.

### MCDSL — Standard Library

- **Version:** v1.7.0.
- **Phase:** Stable. All 9 packages implemented and tested. Being adopted
  across the platform.
- **Deployment:** N/A (Go library, imported by other services).
- **Packages:** auth, db, config, httpserver, grpcserver, csrf, web, health,
  archive.
- **Adoption:** All services except mcias on v1.7.0. mcias pending.

### MCNS — Networking Service

- **Version:** v1.2.0.
- **Phase:** Production. Custom Go DNS server replacing CoreDNS precursor.
- **Deployment:** Running on rift as a container managed by MCP. Serves two
  authoritative zones plus upstream forwarding. REST + gRPC APIs with MCIAS
  auth and name-scoped system account authorization.
- **Recent work:** v1.0.0 implementation (custom Go DNS server), engineering
  review, deployed to rift replacing CoreDNS.
- **Artifacts:** Dockerfile, Docker Compose (rift), MCP service definition,
  systemd units, install script, example config.

### MCDoc — Documentation Server

- **Version:** v0.1.0.
- **Phase:** Production. Fetches and renders markdown documentation from Gitea.
- **Deployment:** Running on rift as a container, fronted by MC-Proxy on
  port 443 (L7).
- **Recent work:** Initial implementation, Gitea content fetching, goldmark
  rendering with syntax highlighting, webhook-driven refresh.
- **Artifacts:** Dockerfile, MCP service definition.

### MCQ — Document Review Queue

- **Version:** v0.4.2.
- **Phase:** Production. Document review queue with MCP server for Claude
  integration.
- **Deployment:** Running on rift as a container managed by MCP.
- **Recent work:** Claude MCP server integration, document review workflow.
- **Artifacts:** Dockerfile, MCP service definition.

### MCP — Control Plane

- **Version:** v0.9.0 (agent on rift: v0.8.3-dirty, agent on svc: v0.9.0).
- **Phase:** Production. Phases A–D complete. Multi-node capable with svc
  operating as an edge node. V2 architecture in development, Phase E planning
  underway.
- **Deployment:** Running on rift. Agent as systemd service under `mcp` user
  with rootless podman. Manages metacrypt, mc-proxy, mcr, mcns, mcdoc, mcat,
  mcq, and non-platform containers. Svc runs an MCP agent for edge mc-proxy
  route management.
- **Architecture:** Two components — `mcp` CLI (thin client on vade) and
  `mcp-agent` (per-node daemon with SQLite registry, podman management,
  monitoring with drift/flap detection, route registration with mc-proxy,
  automated TLS cert provisioning for L7 routes via Metacrypt CA, automated
  DNS registration in MCNS). gRPC-only (no REST). 15 RPCs, 17+ CLI commands.
- **Recent work:** Phase C (automated TLS cert provisioning), Phase D
  (automated DNS registration via MCNS), undeploy command, logs command,
  edit command, auto-login to MCR, system account auth model, module path
  migration, multi-node edge support (svc).
- **Artifacts:** systemd service (NixOS), TLS cert from Metacrypt, service
  definition files, design docs.

## Node Inventory

| Node | Address (LAN) | Address (Tailscale) | Role |
|------|---------------|---------------------|------|
| rift | 192.168.88.181 | 100.95.252.120 | Infrastructure services |
| svc | — | 100.106.232.4 | Edge — public mc-proxy, MCIAS (systemd) |
| orion | 192.168.88.214 | — | Worker (provisioned, offline for maintenance) |

## Rift Port Map

Note: Services deployed via MCP receive dynamically assigned host ports
(10000–60000). The ports below are for infrastructure services with static
assignments or well-known ports.

| Port | Protocol | Services |
|------|----------|----------|
| 53 | DNS (LAN + Tailscale) | mcns |
| 443 | L7 (TLS termination) | metacrypt-web, mcr-web, mcdoc, mcat, kls |
| 8080 | HTTP (all interfaces) | exod |
| 8443 | L4 (SNI passthrough) | metacrypt API, mcr API |
| 9090 | HTTP (all interfaces) | exod |
| 9443 | L4 (SNI passthrough) | metacrypt gRPC, mcr gRPC, sgard |
| 9091 | HTTP (loopback) | MC-Proxy Prometheus metrics |

Svc runs its own mc-proxy on :443 (L7) with public-facing routes:
kls.metacircular.net, mcq.metacircular.net, metacrypt.metacircular.net,
docs.metacircular.net, git.metacircular.net, git.wntrmute.dev.

Non-platform services also running on rift: **exod** (v0.1.0, ports
8080/9090), **sgardd** (v3.2.0, port 19473, fronted by MC-Proxy on 9443),
**kls** (v0.2.0).
