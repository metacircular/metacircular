# Metacircular Infrastructure

## Background

Metacircular Dynamics is a personal infrastructure platform. The name comes
from the tradition of metacircular evaluators in Lisp — a system defined in
terms of itself — by way of SICP and Common Lisp projects that preceded this
work. The infrastructure is metacircular in the same sense: the platform
manages, secures, and hosts its own services.

The goal is sovereign infrastructure. Every component is self-hosted, every
dependency is controlled, and the entire stack is operable by one person. There
are no cloud provider dependencies, no third-party auth providers, no external
databases. When a Metacircular node boots, it connects to Metacircular services
for identity, certificates, container images, and workload scheduling.

All services are written in Go and follow a shared set of engineering standards
(see `engineering-standards.md`). The platform is designed for a small number of
machines — a personal homelab or a handful of VPSes — not for hyperscale.

## Philosophy

**Sovereignty.** You own the whole stack. Identity, certificates, secrets,
container images, DNS, networking — all self-hosted. No SaaS dependency means
no vendor lock-in, no surprise deprecations, and no trust delegation to third
parties.

**Simplicity over sophistication.** SQLite over Postgres. Stdlib `testing` over
test frameworks. Pure Go over CGo. htmx over React. Single-binary deployments
over microservice orchestrators. The right tool is the simplest one that solves
the problem without creating a new one.

**Consistency as leverage.** Every service follows identical patterns: the same
directory layout, the same Makefile targets, the same config format, the same
auth integration, the same deployment model. Knowledge of one service transfers
instantly to all others. A new service can be stood up by copying the skeleton.

**Security as structure.** Security is not a feature bolted on after the fact.
Default deny is the starting posture. TLS 1.3 is the minimum, not a goal.
Interceptor maps make "forgot to add auth" a visible, reviewable omission
rather than a silent runtime failure. Secrets are encrypted at rest behind a
seal/unseal barrier. Every service delegates identity to a single root of
trust.

**Design before code.** The architecture document is written before
implementation begins. It is the spec, not the afterthought. When the code and
the spec disagree, one of them has a bug.

## High-Level Overview

Metacircular infrastructure is built from six core components, plus a shared
standard library (**MCDSL**) that provides the common patterns all services
depend on (auth integration, database setup, config loading, HTTP/gRPC server
bootstrapping, CSRF, web session management, health checks, snapshots, and
service directory archiving):

- **MCIAS** — Identity and access. The root of trust for all other services.
  Handles authentication, token issuance, role management, and login policy
  enforcement. Every other component delegates auth here.

- **Metacrypt** — Cryptographic services. PKI/CA, SSH CA, transit encryption,
  and encrypted secret storage behind a Vault-inspired seal/unseal barrier.
  Issues the TLS certificates that every other service depends on.

- **MCR** — Container registry. OCI-compliant image storage. MCP directs nodes
  to pull images from MCR. Policy-controlled push/pull integrated with MCIAS.

- **MCNS** — Networking. DNS and address management for the platform.

- **MCP** — Control plane. The orchestrator. A master/agent architecture that
  manages workload scheduling, container lifecycle, service registry, data
  transfer, and node state across the platform.

- **MC-Proxy** — Node ingress. A TLS proxy and router that sits on every node,
  accepts outside connections, and routes them to the correct service — either
  as raw TCP passthrough or via TLS-terminating HTTP/2 reverse proxy.

These components form a dependency graph rooted at MCIAS:

```
MCIAS (standalone — the root of trust)
  ├── Metacrypt (uses MCIAS for auth; provides certs to all services)
  ├── MCR (uses MCIAS for auth; stores images pulled by MCP)
  ├── MCNS (uses MCIAS for auth; provides DNS for the platform)
  ├── MCP (uses MCIAS for auth; orchestrates everything; owns service registry)
  └── MC-Proxy (pre-auth; routes traffic to services behind it)
```

### The Node Model

The unit of deployment is the **MC Node** — a machine (physical or virtual)
that participates in the Metacircular platform.

```
                     ┌──────────────────┐    ┌──────────────┐
                     │  System / Core   │    │     MCP      │
                     │  Infrastructure  │    │    Master     │
                     │  (e.g. MCIAS)    │    │              │
                     └────────┬─────────┘    └──────┬───────┘
                              │                     │ C2
                              │                     │
    Outside     ┌─────────────▼─────────────────────▼──────────┐
    Client ────▶│                   MC Node                     │
                │                                               │
                │   ┌───────────┐                               │
                │   │ MC-Proxy  │──┬──────┬──────┐              │
                │   └───────────┘  │      │      │              │
                │              ┌───▼┐  ┌──▼─┐  ┌─▼──┐  ┌─────┐ │
                │              │ α  │  │ β  │  │ γ  │  │ MCP │ │
                │              └────┘  └────┘  └────┘  │Slave│ │
                │                                      └──┬──┘ │
                │                                    ┌────▼───┐│
                │                                    │Docker/ ││
                │                                    │etc.    ││
                │                                    └────────┘│
                └──────────────────────────────────────────────┘
```

Outside clients connect to **MC-Proxy**, which inspects the TLS SNI hostname
and routes to the correct service (α, β, γ) — either as a raw TCP relay or
via TLS-terminating HTTP/2 reverse proxy, per-route. The **MCP Agent** on each
node receives C2 commands from the **MCP Master** (running on the operator's
workstation) and manages local container lifecycle via the container runtime.
Core infrastructure services (MCIAS, Metacrypt, MCR) run on nodes like any
other workload.

### The Network Model

Metacircular nodes are connected via an **encrypted overlay network** — a
self-managed WireGuard mesh, Tailscale, or similar. No component has a hard
dependency on a specific overlay implementation; the platform requires only
that nodes can reach each other over encrypted links.

```
                  Public Internet
                        │
              ┌─────────▼──────────┐
              │  Edge MC-Proxy     │    VPS (public IP)
              │  :443              │
              └─────────┬──────────┘
                        │ PROXY protocol v2
              ┌─────────▼──────────────────────────────────┐
              │         Encrypted Overlay (e.g. WireGuard) │
              │                                            │
  ┌───────────┴──┐   ┌──────────┐   ┌──────────┐   ┌──────┴─────┐
  │ Origin       │   │  Node B  │   │  Node C  │   │  Operator  │
  │ MC-Proxy     │   │  (MCP    │   │          │   │  Workstation│
  │ + services   │   │  agent)  │   │  (MCP    │   │  (MCP      │
  │ (MCP agent)  │   │          │   │  agent)  │   │  Master)   │
  └──────────────┘   └──────────┘   └──────────┘   └────────────┘
```

**External traffic** flows from the internet through an edge MC-Proxy (on a
public VPS), which forwards via PROXY protocol over the overlay to an origin
MC-Proxy on the private network. The overlay preserves the real client IP
across the hop.

**Internal traffic** (MCP C2, inter-service communication, MCNS DNS) flows
directly over the overlay. MCP's C2 channel is gRPC over whatever link exists
between master and agent — the overlay provides the transport.

The overlay network itself is a candidate for future Metacircular management
(a self-hosted WireGuard mesh manager), consistent with the sovereignty
principle of minimizing third-party dependencies.

---

## System Catalog

### MCIAS — Metacircular Identity and Access Service

MCIAS is the root of trust for the entire platform. Every other service
delegates authentication to it; no service maintains its own user database.

**What it provides:**

- **Authentication.** Username/password with optional TOTP and FIDO2/WebAuthn.
  Credentials are verified by MCIAS and a signed JWT bearer token is returned.
  Services validate tokens by calling back to MCIAS (cached 30s by SHA-256 of
  the token).

- **Role-based access.** Three roles — `admin` (full access, policy bypass),
  `user` (policy-governed), `guest` (service-dependent restrictions). Admin
  detection comes solely from the MCIAS `admin` role; services never promote
  users locally.

- **Account types.** Human accounts (interactive users) and system accounts
  (service-to-service). Both authenticate the same way; system accounts enable
  automated workflows.

- **Login policy.** Priority-based ACL rules that control who can log into
  which services. Rules can target roles, account types, service names, and
  tags. This allows operators to restrict access per-service (e.g., deny
  `guest` from services tagged `env:restricted`) without changing the
  services themselves.

- **Token lifecycle.** Issuance, validation, renewal, and revocation.
  Ed25519-signed JWTs. Short expiry with renewal support.

**How other services integrate:** Every service includes an `[mcias]` config
section with the MCIAS server URL, a `service_name`, and optional `tags`. At
login time, the service forwards credentials to MCIAS along with this context.
MCIAS evaluates login policy against the service context, verifies credentials,
and returns a bearer token. The MCIAS Go client library
(`git.wntrmute.dev/kyle/mcias/clients/go`) handles this flow.

**Status:** Implemented. v1.0.0 complete.

---

### Metacrypt — Cryptographic Service Engine

Metacrypt provides cryptographic resources to the platform through a modular
engine architecture, backed by an encrypted storage barrier inspired by
HashiCorp Vault.

**What it provides:**

- **PKI / Certificate Authority.** X.509 certificate issuance. Root and
  intermediate CAs, certificate signing, CRL management, ACME protocol
  support. This is how every service in the platform gets its TLS
  certificates.

- **SSH CA.** SSH certificate signing for host and user certificates,
  replacing static SSH key management. Signing profiles, Key Revocation List
  (KRL) support, gRPC/REST APIs, and web UI.

- **Transit encryption.** Encrypt and decrypt data without exposing keys to
  the caller. Symmetric encryption with versioned key management, signing,
  and HMAC operations. Envelope encryption for services that need to protect
  data at rest without managing their own key material.

- **User-to-user encryption.** End-to-end encryption between users, with key
  management handled by Metacrypt. ECDH key exchange with AES-256-GCM
  encryption.

**Seal/unseal model:** Metacrypt starts sealed. An operator provides a password
which derives (via Argon2id) a key-wrapping key, which decrypts the master
encryption key (MEK), which in turn unwraps per-engine data encryption keys
(DEKs). Each engine mount gets its own DEK, limiting blast radius — compromise
of one engine's key does not expose another's data.

```
Password → Argon2id → KWK → [decrypt] → MEK → [unwrap] → per-engine DEKs
```

**Engine architecture:** Engines are pluggable providers that register with a
central registry. Each engine mount has a type, a name, its own DEK, and its
own configuration. The engine interface handles initialization, seal/unseal
lifecycle, and request routing. New engine types plug in without modifying the
core.

**Policy:** Fine-grained ACL rules control which users can perform which
operations on which engine mounts. Priority-based evaluation, default deny,
admin bypass. See Metacrypt's `POLICY.md` for the full model.

**Status:** Implemented. All four engine types complete — CA (with ACME
support), SSH CA, transit encryption, and user-to-user encryption.

---

### MCR — Metacircular Container Registry

MCR is an OCI Distribution Spec-compliant container registry. It stores and
serves the container images that MCP deploys across the platform.

**What it provides:**

- **OCI-compliant image storage.** Pull, push, tag, and delete container
  images. Content-addressed by SHA-256 digest. Manifests and tags in SQLite,
  blobs on the filesystem.

- **Authenticated access.** No anonymous access. MCR uses the OCI token
  authentication flow: clients hit `/v2/`, receive a 401 with a token
  endpoint, authenticate via MCIAS, and use the returned JWT for subsequent
  requests.

- **Policy-controlled push/pull.** Fine-grained ACL rules govern who can push
  to or pull from which repositories. Integrated with MCIAS roles.

- **Garbage collection.** Unreferenced blobs are cleaned up via the admin CLI
  (`mcrctl`).

**How it fits in:** MCP directs nodes to pull images from MCR. When a workload
is scheduled, MCP tells the node's agent which image to pull and where to get
it. MCR sits behind an MC-Proxy instance for TLS routing.

**Status:** Implemented. Phase 13 (deployment artifacts) complete.

---

### MC-Proxy — TLS Proxy and Router

MC-Proxy is the ingress layer for every MC Node. It accepts TLS connections,
extracts the SNI hostname, and routes to the correct backend. Each route is
independently configured as either **L4 passthrough** (raw TCP relay, no TLS
termination) or **L7 terminating** (terminates TLS, reverse proxies HTTP/2 and
HTTP/1.1 including gRPC). Both modes coexist on the same listener.

**What it provides:**

- **SNI-based routing.** A route table maps hostnames to backend addresses.
  Exact match, case-insensitive. Multiple listeners can bind different ports,
  each with its own route table, all sharing the same global firewall.

- **Dual-mode proxying.** L4 routes relay raw TCP — backends see the original
  TLS handshake, MC-Proxy adds nothing. L7 routes terminate TLS at the proxy
  and reverse proxy HTTP/2 to backends (plaintext h2c or re-encrypted TLS),
  with header injection (`X-Forwarded-For`, `X-Real-IP`), gRPC streaming
  support, and trailer forwarding.

- **Global firewall.** Every connection is evaluated before routing: per-IP
  rate limiting, IP/CIDR blocks, and GeoIP country blocks (MaxMind GeoLite2).
  Blocked connections get a TCP RST — no error messages, no TLS alerts.

- **PROXY protocol.** Listeners can accept v1/v2 headers from upstream proxies
  to learn the real client IP. Routes can send v2 headers to downstream
  backends. This enables multi-hop deployments — a public edge MC-Proxy on a
  VPS forwarding over the encrypted overlay to a private origin MC-Proxy —
  while preserving the real client IP for firewall evaluation and logging.

- **Runtime management.** Routes and firewall rules can be updated at runtime
  via a gRPC admin API on a Unix domain socket (filesystem permissions for
  access control, no network exposure). State is persisted to SQLite with
  write-through semantics.

**How it fits in:** MC-Proxy is pre-auth infrastructure. It sits in front of
everything on a node. Outside clients connect to MC-Proxy on well-known ports
(443, 8443, etc.) and MC-Proxy routes to the correct backend based on the
hostname the client is trying to reach. A typical production deployment uses
two instances — an edge proxy on a public VPS and an origin proxy on the
private network, connected over the overlay with PROXY protocol preserving
client IPs across the hop.

**Status:** Implemented.

---

### MCNS — Metacircular Networking Service

MCNS provides DNS for the platform. It manages two internal zones and serves
as the name resolution layer for the Metacircular network. Service discovery
(which services run where) is owned by MCP; MCNS translates those assignments
into DNS records.

**What it will provide:**

- **Internal DNS.** MCNS is authoritative for the internal zones of the
  Metacircular network. Three zones serve different purposes:

  | Zone | Example | Purpose |
  |------|---------|---------|
  | `*.metacircular.net` | `metacrypt.metacircular.net` | External, public-facing. Managed outside MCNS (existing DNS). Points to edge MC-Proxy. |
  | `*.mcp.metacircular.net` | `vade.mcp.metacircular.net` | Node addresses. Maps node names to their network addresses (e.g. Tailscale IPs). |
  | `*.svc.mcp.metacircular.net` | `metacrypt.svc.mcp.metacircular.net` | Internal service addresses. Maps service names to the node and port where they currently run. |

  The `*.mcp.metacircular.net` and `*.svc.mcp.metacircular.net` zones are
  managed by MCNS. The external `*.metacircular.net` zone is managed separately
  (existing DNS infrastructure) and is mostly static.

- **MCP integration.** MCP pushes DNS record updates to MCNS after deploy and
  migrate operations. When MCP starts service α on node X, it calls the MCNS
  API to set `α.svc.mcp.metacircular.net` to X's address. Services and clients
  using internal DNS names automatically resolve to the right place without
  config changes.

- **Record management API.** Authenticated via MCIAS. MCP is the primary
  consumer for dynamic updates. Operators can also manage records directly
  for static entries (node addresses, aliases).

**How it fits in:** MCNS answers "what is the address of X?" MCP answers "where
is service α running?" and pushes the answer to MCNS. This separation means
services can use stable DNS names in their configs (e.g.,
`mcias.svc.mcp.metacircular.net` in `[mcias] server_url`) that survive
migration without config changes.

**Status:** Not yet implemented. A CoreDNS precursor currently serves the
internal zones (`svc.mcp.metacircular.net` and `mcp.metacircular.net`) as an
interim solution until the full MCNS service is built.

---

### MCP — Metacircular Control Plane

MCP is the orchestrator. It manages what runs where across the platform. The
deployment model is operator-driven: the user says "deploy service α" and MCP
handles the rest. MCP Master runs on the operator's workstation; agents run on
each managed node.

**What it will provide:**

- **Service registry.** MCP is the source of truth for what is running where.
  It tracks every service, which node it's on, and its current state. Other
  components that need to find a service (including MC-Proxy for route table
  updates) query MCP's registry.

- **Deploy.** The operator says "deploy α". MCP checks if α is already running
  somewhere. If it is, MCP pulls the new container image on that node and
  restarts the service in place. If it isn't running, MCP selects a node
  (the operator can pin to a specific node but shouldn't have to), transfers
  the initial config, pulls the image from MCR, starts the container, and
  pushes a DNS update to MCNS (`α.svc.mcp.metacircular.net` → node address).

- **Migrate.** Move a service from one node to another. MCP snapshots the
  service's `/srv/<service>/` directory on the source node (as a tar.zst
  image), transfers it to the destination, extracts it, starts the service,
  stops it on the source, and updates MCNS so DNS points to the new location.
  The `/srv/<service>/` convention makes this uniform across all services.

- **Data transfer.** The C2 channel supports file-level operations between
  master and agents: copy or fetch individual files (push a config, pull a
  log), and transfer tar.zst archives for bulk snapshot/restore of service
  data directories. This is the foundation for both migration and backup.

- **Service snapshots.** To snapshot `/srv/<service>/`, the agent runs
  `VACUUM INTO` to create a consistent database copy, then builds a tar.zst
  that includes the full directory but **excludes** live database files
  (`*.db`, `*.db-wal`, `*.db-shm`) and the `backups/` directory. The
  temporary VACUUM INTO copy is injected into the archive as `<service>.db`.
  The result is a clean, minimal archive that extracts directly into a
  working service directory on the destination.

- **Container lifecycle.** Start, stop, restart, and update containers on
  nodes. MCP Master issues commands; agents on each node execute them against
  the local container runtime (Docker, etc.).

- **Master/agent architecture.** MCP Master runs on the operator's machine.
  Agents run on every managed node, receiving C2 (command and control) from
  Master, reporting node status, and managing local workloads. The C2 channel
  is authenticated via MCIAS. The master does not need to be always-on —
  agents keep running their workloads independently; the master is needed only
  to issue new commands.

- **Node management.** Track which nodes are in the platform, their health,
  available resources, and running workloads.

- **Scheduling.** When placing a new service, MCP selects a node based on
  available resources and any operator-specified constraints. The operator can
  override with an explicit node, but the default is MCP's choice.

**How it fits in:** MCP is the piece that ties everything together. MCIAS
provides identity, Metacrypt provides certificates, MCR provides images, MCNS
provides DNS, MC-Proxy provides ingress — MCP orchestrates all of it, owns the
map of what is running where, and pushes updates to MCNS so DNS stays current. It is the system that makes the
infrastructure metacircular: the control plane deploys and manages the very
services it depends on.

**Container-first design:** All Metacircular services are built as containers
(multi-stage Docker builds, Alpine runtime, non-root) specifically so that MCP
can deploy them. The systemd unit files exist as a fallback and for bootstrap —
the long-term deployment model is MCP-managed containers.

**Status:** Not yet implemented.

---

### MCAT — MCIAS Login Policy Tester

MCAT is a lightweight diagnostic tool, not a core infrastructure component. It
presents a web login form, forwards credentials to MCIAS with a configurable
`service_name` and `tags`, and shows whether the login was accepted or denied
by policy. This lets operators verify that login policy rules behave as
expected without touching the target service.

**Status:** Implemented.

---

## Bootstrap Sequence

Bringing up a Metacircular platform from scratch requires careful ordering
because of the circular dependencies — the infrastructure manages itself, but
must exist before it can do so. The key challenge is that nearly every service
needs TLS certificates (from Metacrypt) and authentication (from MCIAS), but
those services themselves need to be running first.

During bootstrap, all services run as **systemd units** on a single bootstrap
node. MCP takes over lifecycle management as the final step.

### Prerequisites

Before any service starts, the operator needs:

- **The bootstrap node** — a machine (VPS, homelab server, etc.) with the
  overlay network configured and reachable.
- **Seed PKI** — MCIAS and Metacrypt need TLS certs to start, but Metacrypt
  isn't running yet to issue them. The root CA is generated manually using
  `github.com/kisom/cert` and stored in the `ca/` directory in the workspace.
  Initial service certificates are issued from this root. The root CA is then
  imported into Metacrypt once it's running, so Metacrypt becomes the
  authoritative CA for the platform going forward.
- **TOML config files** — each service needs its config in `/srv/<service>/`.
  During bootstrap these are written manually. Later, MCP handles config
  distribution.

### Startup Order

```
Phase 0: Seed PKI
  Operator creates or obtains initial TLS certificates for MCIAS
  and Metacrypt. Places them in /srv/mcias/certs/ and
  /srv/metacrypt/certs/.

Phase 1: Identity
  ┌──────────────────────────────────────────────────────┐
  │ MCIAS starts (systemd)                               │
  │  - No dependencies on other Metacircular services    │
  │  - Uses seed TLS certificates                        │
  │  - Operator creates initial admin account             │
  │  - Operator creates system accounts for other services│
  └──────────────────────────────────────────────────────┘

Phase 2: Cryptographic Services
  ┌──────────────────────────────────────────────────────┐
  │ Metacrypt starts (systemd)                           │
  │  - Authenticates against MCIAS                       │
  │  - Uses seed TLS certificates initially              │
  │  - Operator initializes and unseals                  │
  │  - Operator creates CA engine, imports root CA from  │
  │    ca/, creates issuers                              │
  │  - Can now issue certificates for all other services │
  │  - Reissue MCIAS and Metacrypt certs from own CA     │
  │    (replace seed certs with Metacrypt-issued certs)  │
  └──────────────────────────────────────────────────────┘

Phase 3: Ingress
  ┌──────────────────────────────────────────────────────┐
  │ MC-Proxy starts (systemd)                            │
  │  - Static route table from TOML config               │
  │  - Routes external traffic to MCIAS, Metacrypt       │
  │  - No MCIAS auth (pre-auth infrastructure)           │
  │  - TLS certs for L7 routes from Metacrypt            │
  └──────────────────────────────────────────────────────┘

Phase 4: Container Registry
  ┌──────────────────────────────────────────────────────┐
  │ MCR starts (systemd)                                 │
  │  - Authenticates against MCIAS                       │
  │  - TLS certificates from Metacrypt                   │
  │  - Operator pushes container images for all services │
  │    (including MCIAS, Metacrypt, MC-Proxy themselves) │
  └──────────────────────────────────────────────────────┘

Phase 5: DNS
  ┌──────────────────────────────────────────────────────┐
  │ MCNS starts (systemd)                                │
  │  - Authenticates against MCIAS                       │
  │  - Operator configures initial DNS records           │
  │    (node addresses, service names)                   │
  └──────────────────────────────────────────────────────┘

Phase 6: Control Plane
  ┌──────────────────────────────────────────────────────┐
  │ MCP Agent starts on bootstrap node (systemd)         │
  │ MCP Master starts on operator workstation            │
  │  - Authenticates against MCIAS                       │
  │  - Master registers the bootstrap node               │
  │  - Master imports running services into its registry │
  │  - From here, MCP owns the service map               │
  │  - Services can be redeployed as MCP-managed         │
  │    containers (replacing the systemd units)          │
  └──────────────────────────────────────────────────────┘
```

### The Seed Certificate Problem

The circular dependency between MCIAS, Metacrypt, and TLS is resolved by
bootstrapping with a **manually generated root CA**:

1. The operator generates a root CA using `github.com/kisom/cert`. This root
   and initial service certificates live in the `ca/` directory.
2. MCIAS and Metacrypt start with certificates issued from this external root.
3. Metacrypt comes up. The operator imports the root CA into Metacrypt's CA
   engine, making Metacrypt the authoritative issuer under the same root.
4. Metacrypt can now issue and renew certificates for all services. The `ca/`
   directory remains as the offline backup of the root material.

This is a one-time process. The root CA is generated once, imported once, and
from that point forward Metacrypt is the sole CA. MCP handles certificate
provisioning for all services.

### Adding a New Node

Once the platform is bootstrapped, adding a node is straightforward:

1. Provision the machine and connect it to the overlay network.
2. Install the MCP agent binary.
3. Configure the agent with the MCP Master address and MCIAS credentials
   (system account for the node).
4. Start the agent. It authenticates with MCIAS, connects to Master, and
   reports as available.
5. The operator deploys workloads to it via MCP. MCP handles image pulls,
   config transfer, certificate provisioning, and DNS updates.

### Disaster Recovery

If the bootstrap node is lost, recovery follows the same sequence as initial
bootstrap — but with data restored from backups:

1. Start MCIAS on a new node, restore its database from the most recent
   `VACUUM INTO` snapshot.
2. Start Metacrypt, restore its database. Unseal with the original password.
   The entire key hierarchy and all issued certificates are recovered.
3. Bring up the remaining services in order, restoring their databases.
4. Start MCP, which rebuilds its registry from the running services.
5. Update DNS (MCNS or external) to point to the new node.

Every service's `snapshot` CLI command and daily backup timer exist specifically
to make this recovery possible. The `/srv/<service>/` convention means each
service's entire state is a single directory to back up and restore.

---

## Certificate Lifecycle

Every service in the platform requires TLS certificates, and Metacrypt is the
CA that issues them. This section describes how certificates flow from
Metacrypt to services, how they are renewed, and how the pieces fit together.

### PKI Structure

Metacrypt implements a **two-tier PKI**:

```
Root CA (self-signed, generated at engine initialization)
  ├── Issuer "infra"    (intermediate CA for infrastructure services)
  ├── Issuer "services" (intermediate CA for application services)
  └── Issuer "clients"  (intermediate CA for client certificates)
```

The root CA signs intermediate CAs ("issuers"), which in turn sign leaf
certificates. Each issuer is scoped to a purpose. The root CA certificate is
the trust anchor — services and clients need it (or the relevant issuer chain)
to verify certificates presented by other services.

### ACME Protocol

Metacrypt implements an **ACME server** (RFC 8555) with External Account
Binding (EAB). This is the same protocol used by Let's Encrypt, meaning any
standard ACME client can obtain certificates from Metacrypt.

The ACME flow:

1. Client authenticates with MCIAS and requests EAB credentials from Metacrypt.
2. Client registers an ACME account using the EAB credentials.
3. Client places a certificate order (one or more domain names).
4. Metacrypt creates authorization challenges (HTTP-01 and DNS-01 supported).
5. Client fulfills the challenge (places a file for HTTP-01, or a DNS TXT
   record for DNS-01).
6. Metacrypt validates the challenge and issues the certificate.
7. Client downloads the certificate chain and private key.

A **Go client library** (`metacrypt/clients/go`) wraps this entire flow:
MCIAS login, EAB fetch, account registration, challenge fulfillment, and
certificate download. Services that integrate this library can obtain and
renew certificates programmatically.

### How Services Get Certificates Today

Currently, certificates are provisioned through Metacrypt's **REST API or web
UI** and placed into each service's `/srv/<service>/certs/` directory. This is
a manual process — the operator issues a certificate, downloads it, and
deploys the files. The ACME client library exists but is not yet integrated
into any service.

### How It Will Work With MCP

MCP is the natural place to automate certificate provisioning:

- **Initial deploy.** When MCP deploys a new service, it can provision a
  certificate from Metacrypt (via the ACME client library or the REST API),
  transfer the cert and key to the node as part of the config push to
  `/srv/<service>/certs/`, and start the service with valid TLS material.

- **Renewal.** MCP knows what services are running and when their certificates
  expire. It can renew certificates before expiry by re-running the ACME flow
  (or calling Metacrypt's `renew` operation) and pushing updated files to the
  node. The service restarts with the new certificate.

- **Migration.** When MCP migrates a service, the certificate in
  `/srv/<service>/certs/` moves with the tar.zst snapshot. If the service's
  hostname changes (new node, new DNS name), MCP provisions a new certificate
  for the new name.

- **MC-Proxy L7 routes.** MC-Proxy's L7 mode requires certificate/key pairs
  for TLS termination. MCP (or the operator) can provision these from
  Metacrypt and push them to MC-Proxy's cert directory. MC-Proxy's
  architecture doc lists ACME integration and Metacrypt key storage as future
  work.

### Trust Distribution

Every service and client that validates TLS certificates needs the root CA
certificate (or the relevant issuer chain). Metacrypt serves these publicly
without authentication:

- `GET /v1/pki/{mount}/ca` — root CA certificate (PEM)
- `GET /v1/pki/{mount}/ca/chain` — full chain: issuer + root (PEM)
- `GET /v1/pki/{mount}/issuer/{name}` — specific issuer certificate (PEM)

During bootstrap, the root CA cert is distributed manually (or via the `ca/`
directory in the workspace). Once MCP is running, it can distribute the CA
cert as part of service deployment. Services reference the CA cert path in
their `[mcias]` config section (`ca_cert`) to verify connections to MCIAS and
other services.

---

## End-to-End Deploy Workflow

This traces a deployment from code change to running service, showing how every
component participates. The example deploys a new version of service α that is
already running on Node B.

### 1. Build and Push

The operator builds a new container image and pushes it to MCR:

```
Operator workstation (vade)
  $ docker build -t mcr.metacircular.net/α:v1.2.0 .
  $ docker push mcr.metacircular.net/α:v1.2.0
         │
         ▼
    MC-Proxy (edge) ──overlay──→ MC-Proxy (origin) ──→ MCR
                                                        │
                                                   Authenticates
                                                   via MCIAS
                                                        │
                                                   Policy check:
                                                   can this user
                                                   push to α?
                                                        │
                                                   Image stored
                                                   (blobs + manifest)
```

The `docker push` goes through MC-Proxy (SNI routing to MCR), authenticates
via the OCI token flow (which delegates to MCIAS), and is checked against
MCR's push policy. The image is stored content-addressed in MCR.

### 2. Deploy

The operator tells MCP to deploy:

```
Operator workstation (vade)
  $ mcp deploy α                  # or: mcp deploy α --image v1.2.0
         │
    MCP Master
         │
         ├── Registry lookup: α is running on Node B
         │
         ├── C2 (gRPC over overlay) to Node B agent:
         │     "pull mcr.metacircular.net/α:v1.2.0 and restart"
         │
         ▼
    MCP Agent (Node B)
         │
         ├── Pull image from MCR
         │     (authenticates via MCIAS, same OCI flow)
         │
         ├── Stop running container
         │
         ├── Start new container from updated image
         │     - Mounts /srv/α/ (config, database, certs all persist)
         │     - Service starts, authenticates to MCIAS, resumes operation
         │
         └── Report status back to Master
```

Since α is already running on Node B, this is an in-place update. The
`/srv/α/` directory is untouched — config, database, and certificates persist
across the container restart.

### 3. First-Time Deploy

If α has never been deployed, MCP does more work:

```
Operator workstation (vade)
  $ mcp deploy α --config α.toml
         │
    MCP Master
         │
         ├── Registry lookup: α is not running anywhere
         │
         ├── Scheduling: select Node C (best fit)
         │
         ├── Provision TLS certificate from Metacrypt
         │     (ACME flow or REST API)
         │
         ├── C2 to Node C agent:
         │     1. Create /srv/α/ directory structure
         │     2. Transfer config file (α.toml → /srv/α/α.toml)
         │     3. Transfer TLS cert+key → /srv/α/certs/
         │     4. Transfer root CA cert → /srv/α/certs/ca.pem
         │     5. Pull image from MCR
         │     6. Start container
         │
         ├── Update service registry: α → Node C
         │
         ├── Push DNS update to MCNS:
         │     α.svc.mcp.metacircular.net → Node C address
         │
         └── (Optionally) update MC-Proxy route table
              if α needs external ingress
```

### 4. Migration

Moving α from Node B to Node C:

```
Operator workstation (vade)
  $ mcp migrate α --to node-c     # or let MCP choose the destination
         │
    MCP Master
         │
         ├── C2 to Node B agent:
         │     1. Stop α container
         │     2. Snapshot /srv/α/ → tar.zst archive
         │     3. Transfer tar.zst to Master (or directly to Node C)
         │
         ├── C2 to Node C agent:
         │     1. Receive tar.zst archive
         │     2. Extract to /srv/α/
         │     3. Pull container image from MCR (if not cached)
         │     4. Start container
         │     5. Report status
         │
         ├── Update service registry: α → Node C
         │
         ├── Push DNS update to MCNS:
         │     α.svc.mcp.metacircular.net → Node C address
         │
         └── (If α had external ingress) update MC-Proxy route
              or rely on DNS change
```

### What Each Component Does

| Step | MCIAS | Metacrypt | MCR | MC-Proxy | MCP | MCNS |
|------|-------|-----------|-----|----------|-----|------|
| Build/push image | Authenticates push | — | Stores image, enforces push policy | Routes traffic to MCR | — | — |
| Deploy (update) | Authenticates pull, authenticates service on start | — | Serves image to agent | Routes traffic to service | Coordinates: registry lookup, C2 to agent | — |
| Deploy (new) | Authenticates pull, authenticates service on start | Issues TLS certificate | Serves image to agent | Routes traffic to service (if external) | Coordinates: scheduling, cert provisioning, config transfer, DNS update | Updates DNS records |
| Migrate | Authenticates service on new node | Issues new cert (if hostname changes) | Serves image (if not cached) | Routes traffic to new location | Coordinates: snapshot, transfer, DNS update | Updates DNS records |
| Steady state | Validates tokens for every authenticated request | Serves CA certs publicly, renews certs | Serves image pulls | Routes all external traffic | Tracks service health, holds registry | Serves DNS queries |

---

## Future Ideas

Components and capabilities that may be worth building but have no immediate
timeline. Listed here to capture the thinking; none are committed.

### Observability — Log Collection and Health Monitoring

Every service already produces structured logs (`log/slog`) and exposes health
checks (gRPC `Health.Check` or REST status endpoints). What's missing is
aggregation — today, debugging a cross-service issue means SSH'ing into each
node and reading local logs.

A collector could:

- Gather structured logs from services on each node and forward them to a
  central store.
- Periodically health-check local services and report status.
- Feed health data into MCP so it can make informed decisions (restart
  unhealthy services, avoid scheduling on degraded nodes, alert the operator).

This might be a standalone service or an MCP agent capability, depending on
weight. If it's just "tail logs and hit health endpoints," it fits in the
agent. If it grows to include indexing, querying, retention policies, and
alerting rules, it's its own service.

### Object Store

The platform has structured storage (SQLite), blob storage scoped to container
images (MCR), and encrypted key-value storage (Metacrypt's barrier). It does
not have general-purpose object/blob storage.

Potential uses:

- **Centralized backups.** Service snapshots currently live on each node in
  `/srv/<service>/backups/`. A central object store gives MCP somewhere to push
  tar.zst snapshots for offsite retention.
- **Artifact storage.** Build outputs, large files, anything that doesn't fit
  in a database row.
- **Data sharing between services.** Files that need to move between services
  outside the MCP C2 channel.

Prior art: [Nebula](https://metacircular.net/pages/nebula.html), a
content-addressable data store with capability-based security (SHA-256
addressed blobs, UUID entries for versioning, proxy references for revocable
access). Prototyped in multiple languages. The capability model is interesting
but may be more sophistication than the platform needs — a simpler
authenticated blob store with MCIAS integration might suffice.

### Overlay Network Management

The platform currently relies on an external overlay network (WireGuard,
Tailscale, or similar) for node-to-node connectivity. A self-hosted WireGuard
mesh manager would bring the overlay under Metacircular's control:

- Automate key exchange and peer configuration when MCP adds a node.
- Manage IP allocation within the mesh (potentially absorbing part of MCNS's
  scope).
- Remove the dependency on Tailscale's coordination servers.

This is a natural extension of the sovereignty principle but is low priority
while the mesh is small enough to manage by hand.

### Hypervisor / Isolation

A deeper exploration of environment isolation, message-passing between
services, and access mediation at a level below containers. Prior art:
[hypervisor concept](https://metacircular.net/pages/hypervisor.html). The
current platform achieves these goals through containers + MCIAS + policy
engines. A hypervisor layer would push isolation down to the OS level —
interesting for security but significant in scope. More relevant if the
platform ever moves beyond containers to VM-based workloads.

### Prior Art: SYSGOV

[SYSGOV](https://metacircular.net/pages/lisp-dcos.html) was an earlier
exploration of system management in Lisp, with SYSPLAN (desired state
enforcement) and SYSMON (service management). Many of its research questions —
C2 communication, service discovery, secure config distribution, failure
handling — are directly addressed by MCP's design. MCP is the spiritual
successor, reimplemented in Go with the benefit of the Metacircular platform
underneath it.
