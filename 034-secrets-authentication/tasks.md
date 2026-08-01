---
description: "Task list for 034-secrets-authentication"
---

# Tasks: Generic Service Secrets & Bastion Credential Routing

**Input**: Design documents from `data/specs/bos-system-specs/034-secrets-authentication/`

**Prerequisites**: plan.md, spec.md

**Tests**: Included — this feature changes an authentication/routing path, so regression coverage
for each user story is not optional.

**Organization**: Tasks are grouped by user story so each can be verified independently, per
spec.md's priorities (US1 and US4 are both P1; US2 is P2; US3 is P3).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to

---

## Phase 1: Setup

- [ ] T001 Confirm working on branch `034-secrets-authentication`; no new dependencies to install
      on either `package.json` (BOS) or `bastion/package.json` — this feature only uses Node's
      built-in `crypto` beyond what's already installed.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The generic secret mechanism + companion index must exist before any user story can
be verified.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T002 [P] Create `src/lib/secrets/service-secrets.ts`: port `createToken`/`verifyToken`/
      `listTokens`/`revokeToken`/`hasAnyToken` from `src/lib/webdav/tokens.ts`, generalized to
      `createSecret(service, label?)` / `verifySecret(service, candidate)` /
      `listSecrets(service)` / `revokeSecret(service, tokenId)` / `hasAnySecret(service)`. Keep the
      existing scrypt hash format and `getSecretsStore()` usage unchanged — `service` replaces the
      hardcoded `NAMESPACE` constant as a parameter.
- [ ] T003 [P] Create `src/lib/secrets/credentials-index.ts`: `readIndex()`, `writeIndexEntry(hash,
      service)`, `removeIndexEntry(hash)` against `data/system/credentials-index.json` (use
      `writeFileAtomic`, matching the pattern in `src/lib/integrations/secrets/store.ts`). Entries
      keyed by `sha256(rawSecret)` hex (fast, unsalted — see plan.md's Data Model for why this is a
      different hash than `ServiceSecret.hash`).
- [ ] T004 [US-foundational] Wire `service-secrets.ts`'s `createSecret`/`revokeSecret` to also call
      `credentials-index.ts`'s `writeIndexEntry`/`removeIndexEntry` (FR-003, FR-009).
- [ ] T005 [US-foundational] Update `src/lib/webdav/tokens.ts` to become a thin wrapper calling
      `service-secrets.ts` with `service = "vfs-webdav"`, preserving every existing exported
      function name/signature so `src/lib/webdav/auth.ts` and
      `src/app/api/vfs/webdav-tokens/route.ts` require zero changes (FR-010).
- [ ] T006 Implement the migration decision from plan.md (lazy backfill recommended): when
      `verifySecret()` succeeds against a `ServiceSecret` that has no matching
      `credentials-index.json` entry, write one. Covers existing 033-vfs-mount tokens minted before
      this feature.
- [ ] T007 [P] Unit tests for `service-secrets.ts` (create/verify/revoke/list round-trip, wrong
      candidate rejected, per-service namespace isolation) in a new
      `tests/services/service-secrets.test.ts` (or co-located per project convention).
- [ ] T008 [P] Unit tests for `credentials-index.ts` (write/read/remove round-trip; corrupted or
      missing file doesn't throw, matches spec.md's Edge Cases).

**Checkpoint**: Generic mechanism exists and is fully covered by unit tests — user story work can
begin.

---

## Phase 3: User Story 1 - Mount WebDAV behind Bastion+Keycloak (Priority: P1) 🎯 MVP

**Goal**: Fix the reported 302 bug and the Keycloak `getUser()` gap, generically.

**Independent Test**: Bastion with `AUTH_PROVIDER=keycloak`, no Keycloak admin client configured;
generate a token via Settings; mount with `davfs2`; confirm it works.

### Tests for User Story 1

- [ ] T009 [P] [US1] Unit test for `bastion/src/credential-routing.ts`'s `resolveCredential()`
      against fixture directories under a temp `cfg.volumeBase` (no live Docker/Keycloak needed) —
      covers: match found, no match, corrupted index file for one user doesn't break others'
      lookup (spec.md Edge Cases).
- [ ] T010 [US1] Integration test / manual verification: `docker-compose.keycloak.yml` stack,
      generate a token via Settings while logged in through Keycloak, mount via `davfs2`, confirm
      file read/write — with `KEYCLOAK_CLIENT_SECRET`/admin scope untouched (proves SC-001).

### Implementation for User Story 1

- [ ] T011 [US1] Create `bastion/src/credential-routing.ts`: `resolveCredential(rawSecret):
      Promise<{ username: string; service: string } | null>` — `readdirSync(cfg.volumeBase)`,
      for each directory read `<dir>/data/system/credentials-index.json`, hash `rawSecret` with
      `sha256`, return the first match's directory name as `username`.
- [ ] T012 [US1] Modify `bastion/src/proxy.ts`: remove the `WEBDAV_PREFIX` constant and
      `isWebdavPath()` function entirely (SC-004). Replace the branch condition with "no session AND
      `parseBasicAuth(req.headers.authorization)` succeeds" (any path — FR-008). Replace the
      `provider.getUser(creds.username)` call with `resolveCredential(creds.password)`. On no
      match, keep the existing `401` + `WWW-Authenticate: Basic realm="BrowserOS"` response
      unchanged. On match, keep the existing `Authorization` rewrite to `Bearer <password>` and
      `routeToContainer(username, ...)` call unchanged.
- [ ] T013 [US1] `npx tsc --noEmit` + lint on `bastion/` after the proxy.ts change.

**Checkpoint**: WebDAV mounting works behind Bastion+Keycloak with no admin-API dependency.

---

## Phase 4: User Story 4 - Standalone/local-dev is unaffected (Priority: P1)

**Goal**: Prove the refactor in Phase 2 didn't change standalone behavior at all.

**Independent Test**: Run BOS standalone (no Bastion process running), generate + mount a token.

### Tests for User Story 4

- [ ] T014 [P] [US4] Regression test: with no Bastion involved, `verifySecret("vfs-webdav", token)`
      (via the `tokens.ts` wrapper) still accepts a freshly-created token and rejects a wrong one —
      confirms FR-004 (standalone path never touches the companion index to authenticate).
- [ ] T015 [US4] Manual verification: `run-dev-supervisor.sh` (or equivalent standalone start),
      generate a token via Settings, mount directly against `http://localhost:3000/api/vfs/webdav`
      with no Bastion in front.

**Checkpoint**: Standalone/local-dev behavior confirmed unchanged.

---

## Phase 5: User Story 2 - Mount WebDAV behind Bastion+simple auth (Priority: P2)

**Goal**: Confirm no regression for the previously-working case now that routing goes through
`resolveCredential()` instead of `provider.getUser()`.

**Independent Test**: Bastion with `AUTH_PROVIDER=simple`; generate, mount, then revoke a token.

- [ ] T016 [US2] Integration test / manual verification: `docker-compose.yml` default (simple
      auth), generate a token, mount via `davfs2`, confirm success (SC-002).
- [ ] T017 [US2] Integration test / manual verification: revoke the token via Settings, confirm the
      next WebDAV request is rejected — checks both the encrypted record and the companion index
      entry were removed (FR-009, SC-005).

**Checkpoint**: Both auth providers behave identically from the WebDAV client's perspective.

---

## Phase 6: User Story 3 - A new protocol adopts headless auth with zero Bastion changes (Priority: P3)

**Goal**: Prove the generalization actually generalizes.

- [ ] T018 [P] [US3] Add a throwaway test-only namespace (e.g. `service = "test-protocol"`), mint a
      secret via `service-secrets.ts`, present it via Basic auth to an arbitrary path Bastion has
      never special-cased, and confirm `resolveCredential()` + `proxy.ts` route it correctly with
      no changes to `proxy.ts` beyond what Phase 3 already introduced (SC-003).
- [ ] T019 [P] [US3] Document, in `docs/dev/` (the architecture doc covering the assistant/WebDAV
      area, or a new short section), how a future BOS service adopts headless Basic-auth support:
      call `service-secrets.ts` under its own namespace; nothing else to do on the Bastion side.

**Checkpoint**: A second protocol works through the same mechanism with zero Bastion edits.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T020 Update `docs/dev/` to describe the generic secret mechanism and note that 033-vfs-mount's
      WebDAV tokens are now one caller of it (CLAUDE.md's "update docs/ when architecture changes"
      rule).
- [ ] T021 [P] `npx tsc --noEmit` + `npm run lint` on the BOS side (`src/lib/secrets/`,
      `src/lib/webdav/tokens.ts`).
- [ ] T022 Grep for any other reference to `WEBDAV_PREFIX`/`isWebdavPath`/the old
      `provider.getUser()`-for-routing pattern to confirm full removal (SC-004).
- [ ] T023 Rebuild and redeploy the `bos-bastion` image wherever it actually runs (the reported bug
      was, in the end, a stale deployed image — this feature's code changes only take effect once
      that image is rebuilt and redeployed).

---

## Dependencies & Execution Order

- **Setup (Phase 1)** → **Foundational (Phase 2)**: blocks everything below.
- **US1 (Phase 3)** and **US4 (Phase 4)**: both P1, independent of each other, can proceed in
  parallel once Phase 2 is done.
- **US2 (Phase 5)**: depends on Phase 3's `credential-routing.ts`/`proxy.ts` changes existing, but
  is otherwise independently testable.
- **US3 (Phase 6)**: depends on Phase 2's generic mechanism and Phase 3's Bastion routing change,
  since it's proving the *combination* generalizes.
- **Polish (Phase 7)**: last; T023 (redeploy) is the final step that actually resolves the
  originally-reported issue in any live deployment.

## Notes

- Every task above names concrete files — no task should require guessing a path.
- Commit after each phase checkpoint, not after every task.
- T023 is the one task with real-world consequence beyond this repo (a production redeploy) —
  confirm with the operator before running it, per this project's working rules on hard-to-reverse,
  shared-state actions.
