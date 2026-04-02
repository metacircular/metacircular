# ARCHITECTURE_V2.md Follow-Up Review

Reviewer: Claude (automated architecture review)
Date: 2026-04-02
Document reviewed: `mcp/ARCHITECTURE_V2.md` (revised)

---

## Review Disposition

The revised document addresses 20 of 23 original items. The remaining
gaps are minor and don't block implementation. This document is **ready
to move to implementation**.

---

## Resolution of Original High-Priority Items

### #3 ExportServiceData mode (live vs migration) -- RESOLVED

The document takes the "master owns stop/start" approach: for migration,
the master stops the container (step 2) before calling
`ExportServiceData` (step 3). For scheduled snapshots, the master calls
`ExportServiceData` without stopping. The agent behaves the same either
way -- runs the configured snapshot method, tars, streams back. No proto
change needed.

One edge case to handle during implementation: if the agent's persisted
snapshot method is `grpc` or `cli` and the container is already stopped
(migration case), the agent can't call the service or exec into it.
The shutdown handler's vacuum provides consistency, so the agent should
**detect the container is stopped and skip the method step**, falling
back to a direct tar with auto-vacuum. This doesn't need a proto change
-- just a behavioral note in the agent implementation.

### #4 Snapshot method config propagation -- PARTIALLY RESOLVED

The `ExportServiceDataRequest` comment (line 1093) now states: "Snapshot
config is stored in the agent's registry at deploy time." This is the
right approach (option b from the original review).

**Remaining gap:** `ServiceSpec` (lines 357-363) has no snapshot config
fields. The deploy flow is: CLI reads TOML (which has `[snapshot]`) →
converts to `ServiceSpec` proto → sends to master → master forwards to
agent. If `ServiceSpec` doesn't carry snapshot config, it can't reach
the agent's registry.

**Fix needed:** Add snapshot fields to `ServiceSpec`:

```protobuf
message ServiceSpec {
  string name    = 1;
  bool   active  = 2;
  repeated ComponentSpec components = 3;
  string tier    = 4;
  string node    = 5;
  SnapshotConfig snapshot = 6;  // snapshot method and excludes
}

message SnapshotConfig {
  string method            = 1;  // "grpc", "cli", "exec: <cmd>", "full", or "" (default)
  repeated string excludes = 2;  // paths to skip
}
```

This is a one-line proto addition -- not a design issue, just a gap
in the spec.

### #12 backend_tls protobuf default -- RESOLVED

Line 520: `bool backend_tls = 4; // MUST be true; agent rejects false`

The agent validates the field and rejects `false`. This is the strongest
fix -- the proto default doesn't matter because the agent enforces the
invariant. No silent cleartext is possible.

### #16 Public DNS registration -- RESOLVED

Lines 852-856: New step 6 documents that public DNS records are
pre-provisioned manually at Hurricane Electric. The master resolves the
hostname as a validation check, warns if it fails, but continues
(pragmatic for parallel setup). Clear and complete.

---

## Resolution of Original Medium-Priority Items

### #1 Bootstrap circularity -- RESOLVED

Lines 745-760: New "Bootstrap (first boot)" subsection covers image
pre-staging for stages 1-2 via `podman load`/`podman pull`, documents
that boot sequence config contains full service definitions, and notes
this is the only place definitions live on the agent. Thorough.

### #2 Destructive sync -- RESOLVED

Line 1404: `mcp sync --dry-run` added. The destructive-by-default
behavior is a deliberate design choice -- the services directory is the
source of truth, and sync enforces it. The dry-run flag provides
adequate safety. Acceptable.

### #6 MasterDeployResponse success semantics -- RESOLVED

Lines 403-407: `success` is now documented as "true only if ALL steps
succeeded," with per-step results showing partial failures. Clear.

### #8 Migration destination directory check -- RESOLVED

Line 1106: `bool force = 3` added to `ImportServiceDataChunk`. The
agent handles the directory check and enforces the overwrite guard.
Clean solution -- the check is where the filesystem is.

### #10 Master trust boundary -- RESOLVED

Lines 173-180: New "Trust Assumptions" subsection explicitly states the
master is fully trusted, documents the blast radius, and lists
mitigations. Exactly what was recommended.

### #11 Agent-to-master TLS verification -- RESOLVED

Lines 183-196: New "TLS Verification" subsection documents CA cert
usage, pre-provisioning, and startup retry behavior. Covers the edge
case where Metacrypt isn't up yet (CA cert is static, pre-provisioned).

---

## Resolution of Original Low-Priority Items

| # | Item | Status |
|---|------|--------|
| 5 | --direct mode caveat | RESOLVED (lines 1407-1412) |
| 7 | MigrateRequest validation | RESOLVED (line 1249) |
| 14 | Proto completeness | MOSTLY RESOLVED -- Snapshot RPCs added to service def, renamed to CreateSnapshot. Empty response messages unchanged (acceptable). |
| 15 | Agent schema for snapshot config | IMPLICITLY RESOLVED by line 1093 comment; schema is implementation detail |
| 17 | mc-proxy binding addresses | NOT ADDRESSED -- minor; implementable from context |
| 18 | Phase plan for snapshots | RESOLVED -- new Phase 5 (lines 1496-1503), cut over moved to Phase 6 |
| 23 | Snapshot proto naming | RESOLVED -- renamed to CreateSnapshot (line 389), disambiguation comment added (lines 1256-1258) |

---

## New Observations

### A. Boot config drift potential

Lines 741-742, 757-760: Boot sequence config contains full service
definitions for stage 1-3 services. When the operator bumps an image
version via `mcp deploy`, the boot config is NOT automatically updated.
On next reboot, the agent starts the old version from boot config; the
master then deploys the new version.

This is acceptable for a personal platform (brief window of old version
on reboot) and self-correcting (the master's placement takes over). But
the operator should know to update the boot config for foundation
services (MCIAS, MCNS) where running the wrong version could cause
authentication or DNS failures before the master is even up.

**Recommendation:** Add a note to the boot sequencing section: "When
updating foundation service images, also update the boot sequence
config. The master corrects worker service versions after startup, but
foundation services run before the master exists."

### B. Migration step 3 vs snapshot method

When the master calls `ExportServiceData` after stopping the container
(migration step 3), the agent can't execute `grpc` or `cli` snapshot
methods because the container isn't running. The agent should fall back
to a direct tar with auto-vacuum of `.db` files. This is correct
behavior (the container already vacuumed on shutdown), but should be
documented as an implementation rule: "If the container is not running
when `ExportServiceData` is called, skip the snapshot method and tar
directly."

This is the same point as item #3 above -- noting it here as an
implementation requirement rather than a design gap.

---

## Security Assessment

No new security concerns. The revised document strengthens the security
posture:

- **Trust Assumptions** section (new) makes the threat model explicit.
- **TLS Verification** section (new) closes the gap on inter-component
  TLS validation.
- **backend_tls rejection** ensures no accidental cleartext.
- **Public DNS validation** (warn-and-continue) prevents silent
  misconfiguration without blocking legitimate parallel setup.

The existing security model (identity-bound registration, cert SAN
restrictions, Tailscale ACLs, rate limiting) is unchanged and remains
sound.

---

## Readiness for Implementation

The document is ready for implementation. Summary of what remains:

| Item | Action | When |
|------|--------|------|
| Add `SnapshotConfig` to `ServiceSpec` proto | Proto file edit | Phase 5 (when implementing snapshots) |
| Agent fallback when container is stopped during export | Implementation detail | Phase 5 |
| Boot config drift note | Optional doc edit | Any time |
| mc-proxy binding addresses | Optional doc edit | Any time |

None of these block starting Phase 1 (agent on svc) or Phase 2 (edge
routing RPCs). The snapshot-related items only matter at Phase 5.

**Verdict: proceed to implementation.**
