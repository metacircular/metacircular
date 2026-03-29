# Metacircular Dynamics

Metacircular Dynamics is a self-hosted personal infrastructure platform. The
name comes from the tradition of metacircular evaluators in Lisp — a system
defined in terms of itself — by way of SICP and Common Lisp projects that
preceded this work. The infrastructure is metacircular in the same sense: the
platform manages, secures, and hosts its own services.

Every component is self-hosted, every dependency is controlled, and the entire
stack is operable by one person. No cloud providers, no third-party auth, no
external databases. The platform is designed for a small number of machines — a
personal homelab or a handful of VPSes — not for hyperscale.

All services are written in Go and follow shared
[engineering standards](engineering-standards.md). Full platform documentation
lives in [docs/metacircular.md](docs/metacircular.md).

## Components

| Component | Purpose | Status |
|-----------|---------|--------|
| **MCIAS** | Identity and access — the root of trust. SSO, token issuance, role management, login policy. Every other service delegates auth here. | Implemented |
| **Metacrypt** | Cryptographic services — PKI/CA, transit encryption, encrypted secret storage behind a seal/unseal barrier. Issues TLS certificates for the platform. | Implemented |
| **MCR** | Container registry — OCI-compliant image storage with MCIAS auth and policy-controlled push/pull. | Implemented |
| **MC-Proxy** | Node ingress — TLS proxy and router. L4 passthrough or L7 terminating (per-route), PROXY protocol, firewall with rate limiting and GeoIP. | Implemented |
| **MCNS** | Networking — authoritative DNS for internal platform zones, upstream forwarding. | Implemented |
| **MCP** | Control plane — operator-driven deployment, service registry, data transfer, master/agent container lifecycle. | Implemented |
| **MCDoc** | Documentation server — renders markdown from Gitea, serves public docs. | Implemented |
| **MCDeploy** | Deployment CLI — tactical bridge tool, now deprecated and archived. Superseded by MCP. | Deprecated |

Shared library: **MCDSL** — standard library for all services (auth, db,
config, TLS server, CSRF, snapshots).

Supporting tool: **MCAT** — lightweight web app for testing MCIAS login
policies.

## Architecture

```
MCIAS (standalone — the root of trust)
  ├── Metacrypt (auth via MCIAS; provides certs to all services)
  ├── MCR (auth via MCIAS; stores images pulled by MCP)
  ├── MCNS (auth via MCIAS; provides DNS for the platform)
  ├── MCP (auth via MCIAS; orchestrates everything; owns service registry)
  └── MC-Proxy (pre-auth; routes traffic to services behind it)
```

Each machine is an **MC Node**. On every node, **MC-Proxy** accepts outside
connections and routes by TLS SNI — either relaying raw TCP (L4) or
terminating TLS and reverse proxying HTTP/2 (L7), per-route. **MCP Agent** on
each node receives commands from **MCP Master** (which runs on the operator's
workstation) and manages containers via the local runtime. Core infrastructure
(MCIAS, Metacrypt, MCR) runs on nodes like any other workload.

```
                     ┌──────────────────┐    ┌──────────────┐
                     │  Core Infra      │    │  MCP Master  │
                     │  (e.g. MCIAS)    │    │              │
                     └────────┬─────────┘    └──────┬───────┘
                              │                     │ C2
    Outside     ┌─────────────▼─────────────────────▼──────────┐
    Client ────▶│                   MC Node                     │
                │  ┌───────────┐                               │
                │  │ MC-Proxy  │──┬──────┬──────┐              │
                │  └───────────┘  │      │      │              │
                │             ┌───▼┐  ┌──▼─┐  ┌─▼──┐  ┌─────┐ │
                │             │ α  │  │ β  │  │ γ  │  │ MCP │ │
                │             └────┘  └────┘  └────┘  │Agent│ │
                │                                     └──┬──┘ │
                │                                   ┌────▼───┐│
                │                                   │Container│
                │                                   │Runtime  │
                │                                   └────────┘│
                └──────────────────────────────────────────────┘
```

## Design Principles

- **Sovereignty** — self-hosted end to end; no SaaS dependencies
- **Simplicity** — SQLite over Postgres, stdlib testing, pure Go, htmx, single binaries
- **Consistency** — every service follows identical patterns (layout, config, auth, deployment)
- **Security as structure** — default deny, TLS 1.3 minimum, interceptor-map auth, encrypted-at-rest secrets
- **Design before code** — ARCHITECTURE.md is the spec, written before implementation

## Tech Stack

Go 1.25+, SQLite (modernc.org/sqlite), chi router, gRPC + protobuf, htmx +
Go html/template, golangci-lint v2, Ed25519/Argon2id/AES-256-GCM, TLS 1.3,
container-first deployment (Docker + systemd).

## Repository Structure

This root repository is a workspace container. Each subdirectory is a separate
Git repo with its own `CLAUDE.md`, `ARCHITECTURE.md`, `Makefile`, and `go.mod`:

```
metacircular/
├── mcias/          Identity and Access Service
├── metacrypt/      Cryptographic service engine
├── mcr/            Container registry
├── mc-proxy/       TLS proxy and router
├── mcp/            Control plane (master/agent)
├── mcns/           DNS server
├── mcat/           Login policy tester
├── mcdsl/          Standard library (shared packages)
├── mcdeploy/       Deployment CLI (deprecated, archived)
├── mcdoc/          Documentation server
├── ca/             PKI infrastructure (dev/test, not source code)
└── docs/           Platform-wide documentation
```
