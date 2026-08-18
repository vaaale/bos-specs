# Implementation Plan: Generic Service Secrets & Bastion Credential Routing

**Branch**: `034-secrets-authentication` | **Date**: 2026-08-01 | **Spec**: `./spec.md`

**Input**: Feature specification from `bos-system-specs/034-secrets-authentication/spec.md`

## Summary

Introduce a generic, namespaced secret-issuing mechanism any BOS service can use, plus a
protocol-agnostic routing capability in Bastion: when a request carries no session cookie but a
parseable `Authorization: Basic` header, Bastion resolves which user's container to route it to by
hashing the presented credential and scanning a small, non-reversible, per-user companion index it
already has filesystem access to — never by asking the configured identity provider who owns a
username, and never by checking a path allowlist. Authentication of the request itself always
remains the target service's own responsibility, inside its own container. This feature is
delivered and verified entirely on its own terms, using a self-contained example service for
acceptance testing — it does not require any other feature to exist first.

## Technical Context

**Language/Version**: TypeScript throughout — the BOS server (Next.js App Router, Node.js runtime)
and Bastion (Node/Express, a separate sub-project with its own `package.json`/`tsconfig.json`).

**Primary Dependencies**: No new dependencies on either side. BOS side reuses whatever encrypted,
per-user key/value store already exists for at-rest secret storage (server-only, Node `crypto`).
Bastion side reuses its existing per-user data bind-mount and Node's built-in `crypto` (`sha256`)
for the companion-index lookup — no new npm package needed.

**Storage**: Two per-user, file-based stores, both living under each user's own data directory (no
new volumes, no new database service):
1. An encrypted, per-user secret store — holds the authoritative, salted hash of each issued
   secret. If a suitable encrypted store already exists in the codebase for other purposes, this
   feature reuses it rather than introducing a second one; otherwise this feature provides one.
2. A new plaintext JSON file — the non-reversible routing companion index — one fast, unsalted hash
   + `service` name per entry.

**Testing**: Unit tests for the generic secret module and the companion-index read/write helpers,
using a self-contained example `service` namespace (no dependency on any other feature); a
filesystem-fixture-based unit test for Bastion's routing-lookup function (no live Docker/Keycloak
required for that test); separate integration/manual verification against real Bastion deployments
in both auth-provider modes (see tasks.md).

**Target Platform**: Linux (Docker Compose multi-user deployment via `bastion/`, and standalone
single-container / local-dev).

**Project Type**: Web service (BOS server code) + companion reverse-proxy sub-project (`bastion/`)
— two independently-built TypeScript projects in one repository.

**Performance Goals**: Bastion's routing lookup is a linear scan over provisioned users' single
small JSON files — must stay well under user-perceptible latency at the deployment's configured
concurrent-user cap. No caching is required at this scale; if scale grows substantially in the
future, an in-memory reverse-index rebuilt on file-change would be the natural next step, but is
explicitly not needed for this feature.

**Constraints**: No new external services or databases — all new state is plain files under each
user's own data directory. Must preserve per-user data isolation: no change may let one user's
container read or write another user's directory; only Bastion (which already has cross-user
filesystem access for provisioning, in any deployment where this feature applies) gains a new
*read* path, and only over non-reversible hashes, never secret material. Must not require an
identity-provider admin-API client/service account. Must not introduce a native OAuth-capable
client or an OAuth device-code flow (explicitly out of scope, see spec.md Assumptions).

**Scale/Scope**: Bounded by however many users a given Bastion deployment provisions concurrently
(operator-configurable). This feature delivers the generic mechanism plus one self-contained
example consumer for its own acceptance testing; it is designed for N independent future callers
with zero Bastion changes per FR-008, and does not itself modify any pre-existing consumer as part
of its core scope (see spec.md User Story 6).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

No constitution file exists in this spec store — this section follows this codebase's stated
general working rules instead: work stays on a feature branch, no changes to secrets/lockfiles/
build config beyond what's required, type-check and lint after edits, and developer docs get
updated for the architecture change (see Phase 7 in tasks.md). No violations to record in
Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
bos-system-specs/034-secrets-authentication/
├── spec.md      # Feature specification
├── plan.md      # This file
└── tasks.md     # Task breakdown
```

### Source Code (repository root)

```text
src/lib/secrets/
├── service-secrets.ts      # NEW — generic createSecret/verifySecret/listSecrets/revokeSecret/
│                           #        hasAnySecret, namespaced by `service`; owns the authoritative
│                           #        encrypted per-user store (FR-001, FR-002)
└── credentials-index.ts    # NEW — writeIndexEntry/removeIndexEntry/readIndex against a per-user
                             #        plaintext routing-companion file (FR-003, FR-011)

bastion/src/
└── credential-routing.ts   # NEW — resolveCredential(rawSecret): { username, service } | null;
                             #        scans every provisioned user's companion index (FR-005, FR-006)

bastion/src/proxy.ts         # MODIFIED — the existing no-session branch gains (or, if a
                              #             protocol-specific special case already exists here from
                              #             some other feature, is superseded by) a generic
                              #             "Basic auth present → resolveCredential() routing"
                              #             branch (FR-005, FR-007, FR-008). This feature does not
                              #             assume what proxy.ts currently contains — see the
                              #             Integration Note below.

docs/dev/
└── (an architecture doc covering server-side services/auth) — updated to describe the generic
    secret mechanism as a reusable building block.
```

**Structure Decision**: Single-project layout for the BOS side (a new `src/lib/secrets/` module,
independent of any specific protocol), plus the corresponding new capability in the separate
`bastion/` sub-project. No new top-level project or directory is introduced.

### Integration Note (non-blocking)

This feature does not read, assume, or depend on the current contents of `bastion/src/proxy.ts`
beyond "it is where Bastion's per-request routing decision is made." If, at implementation time,
`proxy.ts` already contains a protocol-specific special case (for example, one introduced by a
prior, independently-developed feature covering a specific mount protocol), that special case
SHOULD be superseded by this feature's generic branch as part of implementation — but this feature
is fully specified, implementable, and testable even if no such prior special case exists. Nothing
in this plan requires it to.

## Data Model

### ServiceSecret

| Field       | Type   | Notes                                                              |
|-------------|--------|---------------------------------------------------------------------|
| `secretId`  | string | Unique identifier                                                    |
| `service`   | string | Namespace, e.g. `"example-protocol"` — caller-chosen                 |
| `hash`      | string | Salted, slow (scrypt-class) hash of the raw secret                   |
| `label`     | string | Optional, caller-supplied                                            |
| `createdAt` | string | ISO timestamp                                                        |

Stored in a per-user encrypted store, keyed by `(service, secretId)`.

### CredentialsIndexEntry

| Field       | Type   | Notes                                                                 |
|-------------|--------|--------------------------------------------------------------------------|
| `hash`      | string | `sha256(rawSecret)` hex — **not** `ServiceSecret.hash`; unsalted, deterministic, fast — this is the map key |
| `service`   | string | Which service this secret belongs to                                   |
| `createdAt` | string | ISO timestamp                                                          |

Stored as a single JSON file per user, e.g. `data/system/credentials-index.json`:

```json
{
  "version": 1,
  "entries": {
    "<sha256hex>": { "service": "example-protocol", "createdAt": "..." }
  }
}
```

**Why a second, different hash for the same secret**: `ServiceSecret.hash` is intentionally slow
and salted to resist offline brute-force if the encrypted store were ever exfiltrated — correct for
the authoritative check, but unusable for a fast lookup-by-value scan (you can't derive it from a
candidate secret without already knowing which salt to use). The companion index instead uses a
fast, unsalted `sha256` purely so Bastion can compute one hash from a presented credential and do an
O(1) map lookup per user directory. This is safe specifically because the secret is a high-entropy,
BOS-generated random token (never a user-chosen password) — preimage resistance alone is
sufficient; hash speed doesn't matter for brute-force resistance against a secret with that much
entropy. The index entry alone is also not sufficient to authenticate anything (FR-011) — it only
tells Bastion which user's container to try; that container's own authoritative check remains the
real gate.

## Routing Flow

```text
request arrives at Bastion, no session cookie
  │
  ├─ Authorization: Basic present? ──no──▶ existing no-session behavior (e.g. redirect to /login)
  │
  yes
  │
  ▼
parse username/password from the Basic header (password = the raw secret; username is unused)
  │
  ▼
resolveCredential(password):
  for each directory under the per-user data root:        # one per provisioned username
    read <dir>/data/system/credentials-index.json
    if sha256(password) in entries → return { username: <dir>, service: entries[hash].service }
  return null
  │
  ├─ null ──▶ 401, WWW-Authenticate: Basic realm="BrowserOS"
  │
  found { username, service }
  │
  ▼
rewrite Authorization → "Bearer " + password
route to that user's container (existing container-routing machinery, unchanged)
  │
  ▼
target container's own service-secrets verification (unchanged authoritative check — FR-007)
```

The Basic-auth *username* the client sends is never used for anything — only the presented secret
determines routing.

## Complexity Tracking

*No constitution violations — table intentionally empty.*
