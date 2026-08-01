# Implementation Plan: Generic Service Secrets & Bastion Credential Routing

**Branch**: `034-secrets-authentication` | **Date**: 2026-08-01 | **Spec**: `./spec.md`

**Input**: Feature specification from `data/specs/bos-system-specs/034-secrets-authentication/spec.md`

## Summary

Generalize 033-vfs-mount's WebDAV-only mount-token system into a generic, namespaced secret
mechanism any BOS service can use, and replace Bastion's WebDAV-specific routing special-case
(`WEBDAV_PREFIX`/`isWebdavPath` + `provider.getUser()`) with a protocol-agnostic one: Bastion
resolves routing by hashing a presented Basic-auth credential and scanning a small, non-reversible,
per-user companion index it already has filesystem access to (via the existing `/user-data`
bind-mount) — never by asking the identity provider who a username belongs to. Authentication
itself stays exactly where it is today: the target container's own encrypted secret store is the
only thing that ever validates a secret.

## Technical Context

**Language/Version**: TypeScript throughout — Next.js 16 App Router (BOS server code, Node.js
runtime) and Node/Express (Bastion, a separate sub-project at `bastion/`).

**Primary Dependencies**: No new dependencies. Reuses `src/lib/integrations/secrets/store.ts`
(`SecretsStore`, AES-256-GCM + scrypt), `src/os/data-dir.ts` (`dataDir()`), and Bastion's existing
`bastion/src/docker.ts`/`config.ts` (`cfg.volumeBase`, already bind-mounted). Node's built-in
`crypto` module covers the new fast companion hash (`sha256`); no new npm package is needed on
either side.

**Storage**: Two per-user, file-based stores, both already living under each user's `data/`
directory (no new volumes, no new database service):
1. The existing encrypted `SecretsStore` file — unchanged, holds the authoritative, salted scrypt
   hash of each secret.
2. A new plaintext JSON file, `data/system/credentials-index.json` — the non-reversible routing
   companion, one fast unsalted hash + `service` name per entry.

**Testing**: Existing project test runner (`tests/services/*.test.ts` pattern, Vitest-style
per-file unit tests) for the generic secret module and the companion-index read/write helpers;
a Node-level unit test for Bastion's new `resolveCredential()` scan (no live Docker/Keycloak needed
— it's a pure filesystem scan against fixture directories).

**Target Platform**: Linux (Docker Compose multi-user deployment via `bastion/`, and standalone
single-container / local-dev `next dev`).

**Project Type**: Web service (`src/`) + companion reverse-proxy sub-project (`bastion/`) — two
TypeScript projects in one repo, each with its own `package.json`/`tsconfig.json` (per
CLAUDE.md's existing description of `bastion/`).

**Performance Goals**: Bastion's routing lookup is a linear scan over provisioned users' single
small JSON files — must stay well under user-perceptible latency at the default
`MAX_CONCURRENT_INSTANCES` (50). No caching is required at this scale; if scale grows substantially
in the future, an in-memory reverse-index rebuilt on file-change would be the next step, but that's
explicitly not needed now.

**Constraints**: No new external services or databases (matches "all runtime state persists as
files under `./data`," CLAUDE.md). Must preserve per-user data isolation — no change may let one
user's container read or write another user's directory; only Bastion (which already has
cross-user filesystem access for provisioning) gains a new *read* path, and only over non-reversible
hashes. Must not require a Keycloak admin-API client/service account. Must not introduce a native
mount client or OAuth device-code flow (explicitly out of scope, see spec Assumptions).

**Scale/Scope**: Bounded by `MAX_CONCURRENT_INSTANCES` (default 50, operator-configurable). Two
call sites migrate to the generic mechanism in this feature (WebDAV mint + WebDAV verify); the
mechanism itself is designed for N future callers with zero Bastion changes per FR-008.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

No constitution file exists in this spec store (`bos-system-specs/.specify/memory/constitution.md`
is not present in this checkout) — this section falls back to CLAUDE.md's stated working rules,
which this plan follows: work stays on a feature branch, no changes to secrets/`package.json`/
lockfiles/build config beyond what's required, `npx tsc --noEmit` + `npm run lint` after edits, and
`docs/` gets updated for the architecture change (see Phase 6 below). No violations to record in
Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
data/specs/bos-system-specs/034-secrets-authentication/
├── spec.md      # Feature specification
├── plan.md      # This file
└── tasks.md     # Task breakdown (this feature's Phase 2 output)
```

### Source Code (repository root)

```text
src/lib/secrets/
├── service-secrets.ts      # NEW — generic createSecret/verifySecret/listSecrets/revokeSecret,
│                           #        namespaced by `service`; wraps the existing SecretsStore
└── credentials-index.ts    # NEW — writeIndexEntry/removeIndexEntry/readIndex against
                             #        data/system/credentials-index.json

src/lib/webdav/
├── tokens.ts                # MODIFIED — becomes a thin wrapper over service-secrets.ts with
│                             #            service = "vfs-webdav" (keeps createToken/verifyToken/
│                             #            listTokens/revokeToken/hasAnyToken call sites unchanged)
└── auth.ts                  # UNCHANGED — still calls verifyToken(); the wrapper absorbs the change

bastion/src/
├── credential-routing.ts    # NEW — resolveCredential(rawSecret): { username, service } | null;
│                             #        scans every dir under cfg.volumeBase for
│                             #        data/system/credentials-index.json
└── proxy.ts                 # MODIFIED — remove WEBDAV_PREFIX/isWebdavPath; the no-session branch
                              #             becomes "Basic auth header present" (any path), and its
                              #             username resolution switches from provider.getUser() to
                              #             credential-routing.ts's resolveCredential()

docs/dev/
└── (relevant architecture doc) — updated to describe the generic secret mechanism and note that
    033-vfs-mount's WebDAV tokens are now one caller of it, per CLAUDE.md's "update docs/ when
    architecture changes" rule.
```

**Structure Decision**: Single-project layout for the BOS side (`src/lib/secrets/`, alongside the
existing `src/lib/webdav/` and `src/lib/integrations/secrets/`), plus the corresponding change in
the separate `bastion/` sub-project it already has (per CLAUDE.md: "`bastion/` is a standalone
Node.js/Express sub-project with its own `package.json` and `tsconfig.json`"). No new top-level
project or directory is introduced — this generalizes existing modules in place.

## Data Model

### ServiceSecret (generalizes 033-vfs-mount's `TokenRecord`)

| Field       | Type   | Notes                                                              |
|-------------|--------|---------------------------------------------------------------------|
| `tokenId`   | string | UUID, unchanged from today                                          |
| `service`   | string | Namespace, e.g. `"vfs-webdav"` — new field; was an implicit constant |
| `hash`      | string | `"<saltHex>:<hashHex>"`, scrypt — unchanged algorithm/format         |
| `label`     | string | Unchanged                                                            |
| `createdAt` | string | ISO timestamp, unchanged                                             |

Stored via `getSecretsStore().set(service, tokenId, record)` — same store, `service` replaces the
hardcoded `NAMESPACE` constant as a parameter.

### CredentialsIndexEntry (new)

| Field       | Type   | Notes                                                                 |
|-------------|--------|------------------------------------------------------------------------|
| `hash`      | string | `sha256(rawSecret)` hex — **not** the scrypt hash; unsalted, deterministic, fast — this is the map key, not a separate field |
| `service`   | string | Which service this secret belongs to                                   |
| `createdAt` | string | ISO timestamp                                                          |

Stored as a single JSON file at `data/system/credentials-index.json`:

```json
{
  "version": 1,
  "entries": {
    "<sha256hex>": { "service": "vfs-webdav", "createdAt": "..." }
  }
}
```

**Why a second, different hash for the same secret**: `ServiceSecret.hash` is intentionally slow
and salted (scrypt) to resist offline brute-force if the encrypted store were ever exfiltrated —
correct for the authoritative check, but unusable for a fast lookup-by-value scan (you can't derive
it from a candidate secret without already knowing which salt to use). The companion index instead
uses a fast, unsalted `sha256` purely so Bastion can compute one hash from a presented credential
and do an O(1) map lookup per user directory. This is safe specifically because the secret is a
high-entropy, BOS-generated random token (never a user-chosen password) — preimage resistance
alone is sufficient; hash speed doesn't matter for brute-force resistance against 256 bits of
entropy. The index entry alone is also not sufficient to authenticate anything (FR-011) — it only
tells Bastion which user's container to try; that container's own scrypt-based check remains the
real gate.

## Routing Flow (replaces `isWebdavPath` branch in `bastion/src/proxy.ts`)

```text
request arrives, no session cookie
  │
  ├─ Authorization: Basic present? ──no──▶ clearSession(); redirect to /login   (unchanged)
  │
  yes
  │
  ▼
parse username/password from Basic header (password = the raw secret)
  │
  ▼
resolveCredential(password):
  for each dir in readdirSync(cfg.volumeBase):        # one dir per provisioned username
    read dir/data/system/credentials-index.json
    if sha256(password) in entries → return { username: dir, service: entries[hash].service }
  return null
  │
  ├─ null ──▶ 401, WWW-Authenticate: Basic realm="BrowserOS"    (unchanged shape)
  │
  found { username, service }
  │
  ▼
rewrite Authorization → "Bearer " + password
routeToContainer(username, ...)                        # unchanged — same proxy path as today
  │
  ▼
target container's own service-secrets verify (unchanged authoritative check)
```

The Basic-auth *username* the client sends is no longer used for anything (mirrors the
already-documented "elsewhere, the username is ignored" standalone behavior — this makes it true
for the Bastion+Basic-auth case too, for any auth provider).

## Migration

Existing `vfs-webdav`-namespaced `ServiceSecret` records created under 033-vfs-mount predate the
companion index and have no corresponding `credentials-index.json` entry. Two options, to be
decided during Phase 2 (see tasks.md):

- **(a) Lazy backfill**: on next successful direct verification (standalone) or on next Settings
  page load, write the missing index entry for any existing secret that lacks one.
- **(b) No backfill, document it**: existing tokens continue to work standalone (FR-004 doesn't
  depend on the index) but won't route via Bastion until regenerated; Settings UI could surface
  "regenerate to enable Bastion mounting" for pre-existing tokens.

Given how recently 033-vfs-mount shipped and how few tokens are likely to exist in the wild, (a) is
recommended as low-cost insurance, but this is a task-level decision, not a blocking one.

## Complexity Tracking

*No constitution violations — table intentionally empty.*
