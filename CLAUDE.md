# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Metacircular is a multi-service personal infrastructure platform. This root repository is a workspace container — each subdirectory is a separate Git repo (gitignored here). The authoritative platform-wide standards live in `engineering-standards.md`.

## Project Map

| Directory | Purpose | Language |
|-----------|---------|----------|
| `mcias/` | Identity and Access Service — central SSO/IAM, all other services delegate auth here | Go |
| `metacrypt/` | Cryptographic service engine — encrypted secrets, PKI/CA, SSH CA, transit encryption | Go |
| `mc-proxy/` | TLS proxy and router — L4 passthrough or L7 terminating, PROXY protocol, firewall | Go |
| `mcr/` | OCI container registry — integrated with MCIAS for auth and policy-based push/pull | Go |
| `mcat/` | MCIAS login policy tester — lightweight web app to test and audit login policies | Go |
| `mcdsl/` | Standard library — shared packages for auth, db, config, TLS servers, CSRF, snapshots | Go |
| `ca/` | PKI infrastructure and secrets for dev/test (not source code, gitignored) | — |

Each subproject has its own `CLAUDE.md`, `ARCHITECTURE.md`, `Makefile`, and `go.mod`. When working in a subproject, read its own CLAUDE.md first.

## Service Dependencies

MCIAS is the root dependency — every other service authenticates through it. No service maintains its own user database. The dependency graph:

```
mcias (standalone — no MCIAS dependency)
  ├── metacrypt (uses MCIAS for auth)
  ├── mc-proxy (uses MCIAS for admin auth)
  ├── mcr (uses MCIAS for auth + policy)
  └── mcat (tests MCIAS login policies)
```

## Standard Build Commands (all subprojects)

```bash
make all         # vet → lint → test → build (the CI pipeline)
make build       # go build ./...
make test        # go test ./...
make vet         # go vet ./...
make lint        # golangci-lint run ./...
make proto       # regenerate gRPC code from .proto files
make proto-lint  # buf lint + buf breaking
make devserver   # build and run locally against srv/ config
make docker      # build container image
make clean       # remove binaries
```

Run a single test: `go test ./internal/auth/ -run TestTokenValidation`

## Critical Rules

1. **REST/gRPC sync**: Every REST endpoint must have a corresponding gRPC RPC, updated in the same change.
2. **gRPC interceptor maps**: New RPCs must be added to `authRequiredMethods`, `adminRequiredMethods`, and/or `sealRequiredMethods`. Forgetting this is a security defect.
3. **No CGo in production**: All builds use `CGO_ENABLED=0`. Use `modernc.org/sqlite`, not `mattn/go-sqlite3`.
4. **No test frameworks**: Use stdlib `testing` only. Real SQLite in `t.TempDir()`, no mocks for databases.
5. **Default deny**: Unauthenticated and unauthorized requests are always rejected. Admin detection comes solely from the MCIAS `admin` role.
6. **Proto versioning**: Start at v1. Only create v2 for breaking changes. Non-breaking additions go in-place.

## Architecture Patterns

- **Seal/Unseal**: Metacrypt starts sealed and requires a password to unlock (Vault-like pattern). Key hierarchy: Password → Argon2id → KWK → MEK → per-engine DEKs.
- **Web UI separation**: Web UIs run as separate binaries communicating with the API server via gRPC. No direct DB access from the web tier.
- **Config**: TOML with env var overrides (`SERVICENAME_*`). All runtime data in `/srv/<service>/`.
- **Policy engines**: Priority-based ACL rules, default deny, admin bypass. See metacrypt's implementation as reference.
- **Auth flow**: Client → service `/v1/auth/login` → MCIAS client library → MCIAS validates → bearer token returned. Token validation cached 30s keyed by SHA-256 of token.

## Tech Stack

- Go 1.25+, chi router, cobra CLI, go-toml/v2
- SQLite via modernc.org/sqlite (pure Go), WAL mode, foreign keys on
- gRPC + protobuf, buf for linting
- htmx + Go html/template for web UIs
- golangci-lint v2 with errcheck, gosec, staticcheck, revive
- TLS 1.3 minimum, AES-256-GCM, Argon2id, Ed25519
