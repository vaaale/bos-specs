---
description: "Task list for 034-secrets-authentication"
---

# Tasks: Generic Service Secrets & Bastion Credential Routing

**Input**: Design documents from `bos-system-specs/034-secrets-authentication/`

**Prerequisites**: plan.md, spec.md

**Tests**: Included — this feature introduces an authentication/routing mechanism, so coverage for
each user story is not optional.

**Organization**: Tasks are grouped by user story per spec.md's priorities (US1, US2, US3 are P1;
US4 is P2; US5 and US6 are P3, with US6 explicitly optional/non-blocking).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to

---

## Phase 1: Setup

- [ ] T001 Confirm working on branch `034-secrets-authentication`; no new dependencies to install
      on either the BOS side or `bastion/package.json` — this feature only uses Node's built-in
      `crypto` beyond what's already installed.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: The generic secret mechanism + companion index must exist before any user story can
be verified. Nothing here references or depends on any other feature.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T002 [P] Create `src/lib/secrets/service-secrets.ts`: `createSecret(service, label?)`,
      `verifySecret(service, candidate)`, `listSecrets(service)`, `revokeSecret(service, secretId)`,
      `hasAnySecret(service)`. Each secret is a random, high-entropy value shown once at creation;
      only a salted, slow (scrypt-class) hash is ever persisted, in a per-user encrypted store keyed
      by `(service, secretId)` (FR-001, FR-002). If a suitable encrypted per-user store already
      exists in the codebase for other purposes, reuse it; otherwise implement one scoped to this
      feature.
- [ ] T003 [P] Create `src/lib/secrets/credentials-index.ts`: `readIndex()`,
      `writeIndexEntry(hash, service)`, `removeIndexEntry(hash)` against a per-user plaintext file
      (e.g. `data/system/credentials-index.json`), written atomically. Entries keyed by
      `sha256(rawSecret)` hex (fast, unsalted — see plan.md's Data Model for why this must be a
      different hash than `ServiceSecret.hash`) (FR-003, FR-011).
- [ ] T004 Wire `service-secrets.ts`'s `createSecret`/`revokeSecret` to also call
      `credentials-index.ts`'s `writeIndexEntry`/`removeIndexEntry` (FR-003, FR-009).
- [ ] T005 [P] Unit tests for `service-secrets.ts` in a new `tests/services/service-secrets.test.ts`
      (or co-located per project convention): create/verify/revoke/list round-trip, wrong candidate
      rejected, per-service namespace isolation (spec.md User Story 1's acceptance scenarios).
- [ ] T006 [P] Unit tests for `credentials-index.ts`: write/read/remove round-trip; a missing or
      corrupted file doesn't throw and is treated as empty (spec.md Edge Cases).

**Checkpoint**: Generic mechanism exists, fully covered by unit tests, with no dependency on
anything outside this feature. User story work can begin.

---

## Phase 3: User Story 2 - Standalone BOS authenticates directly (Priority: P1)

**Goal**: Prove the standalone path never depends on Bastion or the companion index.

**Independent Test**: With no Bastion process running, mint and verify a secret directly.

- [ ] T007 [US2] Regression/behavior test: with no Bastion involved, `verifySecret(service, raw)`
      accepts a freshly-created secret and rejects an incorrect one, using only the encrypted store
      from Phase 2 — confirms FR-004 (the companion index is never consulted to authenticate).

**Checkpoint**: Standalone behavior confirmed self-sufficient.

---

## Phase 4: User Story 3 - Bastion routes a headless request behind Keycloak (Priority: P1) 🎯 MVP

**Goal**: Prove Bastion can route a headless Basic-auth request to the correct user's container
under `AUTH_PROVIDER=keycloak`, with no identity-provider admin-API dependency.

**Independent Test**: Bastion with `AUTH_PROVIDER=keycloak` and no Keycloak admin client configured
anywhere; mint a secret (via Phase 2's mechanism, under a self-contained example `service`
namespace) for a real, Keycloak-authenticated user; present it via Basic auth; confirm routing.

### Tests for User Story 3

- [ ] T008 [P] [US3] Unit test for `bastion/src/credential-routing.ts`'s `resolveCredential()`
      against fixture directories representing several provisioned users (no live Docker/Keycloak
      needed) — covers: match found, no match, one user's corrupted index file doesn't break the
      scan for others (spec.md Edge Cases).
- [ ] T009 [US3] Integration test / manual verification against a real `AUTH_PROVIDER=keycloak`
      Bastion deployment with no Keycloak admin client configured: mint a secret for a
      Keycloak-authenticated user via the example service from Phase 2, present it via Basic auth,
      confirm it routes to that user's container (SC-001).

### Implementation for User Story 3

- [ ] T010 [US3] Create `bastion/src/credential-routing.ts`: `resolveCredential(rawSecret):
      Promise<{ username: string; service: string } | null>` — enumerate every directory under
      Bastion's per-user data root, read each one's `data/system/credentials-index.json`, hash
      `rawSecret` with `sha256`, and return the first match's directory name as `username` plus its
      `service`.
- [ ] T011 [US3] Modify `bastion/src/proxy.ts`'s no-session branch: when the request carries a
      parseable `Authorization: Basic` header, call `resolveCredential(password)` instead of asking
      the identity provider to resolve the Basic-auth username. On no match, respond `401` with
      `WWW-Authenticate: Basic realm="BrowserOS"`. On a match, rewrite `Authorization` to
      `Bearer <password>` and route to the resolved user's container using the existing
      container-routing path. Per plan.md's Integration Note: if a protocol-specific special case
      already exists here from some other, independently-developed feature, supersede it with this
      generic branch; do not assume it exists.
- [ ] T012 [US3] Type-check + lint `bastion/` after the `proxy.ts` change.

**Checkpoint**: Headless routing works behind Bastion+Keycloak with no admin-API dependency —
this feature's core problem is solved and independently verifiable.

---

## Phase 5: User Story 4 - Bastion routes a headless request under simple auth (Priority: P2)

**Goal**: Confirm parity across auth providers now that routing goes through `resolveCredential()`.

**Independent Test**: Bastion with `AUTH_PROVIDER=simple`; mint, route, then revoke a secret.

- [ ] T013 [US4] Integration test / manual verification: `AUTH_PROVIDER=simple` Bastion, mint a
      secret via the example service, present it via Basic auth, confirm routing succeeds (SC-002).
- [ ] T014 [US4] Integration test / manual verification: revoke the secret, confirm the next
      request using it is rejected — checks both the encrypted record and the companion index entry
      were removed (FR-009, SC-005).

**Checkpoint**: Both auth providers behave identically for headless routing.

---

## Phase 6: User Story 5 - A second, independent service adopts the mechanism (Priority: P3)

**Goal**: Prove the generalization actually generalizes, using a service distinct from whatever
example was used in Phases 2-5.

- [ ] T015 [P] [US5] Introduce a second, independent test-only `service` namespace, mint a secret
      via `service-secrets.ts`, present it via Basic auth to a path Bastion has never special-cased,
      and confirm `resolveCredential()` + `proxy.ts` route it correctly with no changes to
      `proxy.ts` beyond what Phase 4 already introduced (SC-003, SC-004).
- [ ] T016 [P] [US5] Document, in `docs/dev/`, how a BOS service adopts headless Basic-auth support:
      call `service-secrets.ts` under its own namespace; nothing else to do on the Bastion side.

**Checkpoint**: A second, unrelated service works through the same mechanism with zero Bastion
edits — the generalization claim (FR-008) is proven, not assumed.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T017 Update `docs/dev/` to describe the generic secret mechanism as a reusable building block
      for any BOS service needing headless-client auth.
- [ ] T018 [P] Type-check + lint the BOS-side changes (`src/lib/secrets/`).
- [ ] T019 Confirm Bastion's routing code contains no protocol-specific constant or path check
      anywhere (SC-004) — this is a property of what Phase 4 built, not a cleanup of any assumed
      pre-existing code.
- [ ] T020 Rebuild and redeploy the `bastion` image wherever this feature's Bastion-side change
      needs to take effect in a live deployment.

---

## Phase 8: User Story 6 - Optional integration with an existing consumer (Priority: P3, non-blocking)

**Not required for this feature's completion.** Include only if a compatible headless-client
consumer already exists elsewhere in the codebase at implementation time.

- [ ] T021 [US6] [OPTIONAL] If such a consumer exists, identify its own bespoke secret store and any
      protocol-specific Bastion special case tied to it.
- [ ] T022 [US6] [OPTIONAL] Migrate that consumer to call `service-secrets.ts` under its own
      `service` namespace, preserving its existing user-facing behavior (FR-010).
- [ ] T023 [US6] [OPTIONAL] Decide and implement a migration path for that consumer's
      previously-issued credentials (e.g. lazy-backfill their companion index entry on next
      successful verification), or document that they must be reissued.
- [ ] T024 [US6] [OPTIONAL] Remove that consumer's now-superseded protocol-specific Bastion special
      case, if Phase 4 hasn't already made it dead code.

---

## Dependencies & Execution Order

- **Setup (Phase 1)** → **Foundational (Phase 2)**: blocks everything below.
- **US2 (Phase 3)** and **US3 (Phase 4)**: both P1, independent of each other, can proceed in
  parallel once Phase 2 is done.
- **US4 (Phase 5)**: depends on Phase 4's `credential-routing.ts`/`proxy.ts` change existing, but is
  otherwise independently testable.
- **US5 (Phase 6)**: depends on Phase 2's generic mechanism and Phase 4's Bastion routing change,
  since it's proving the *combination* generalizes.
- **Polish (Phase 7)**: after all required (P1/P2/P3-required) stories.
- **US6 (Phase 8)**: entirely optional and independent of the rest of this feature's completion —
  only relevant if/when a suitable existing consumer is identified.

## Notes

- Every task above names concrete files or a concrete new module — no task should require guessing
  a path, and none assumes a specific pre-existing file's contents beyond Phase 8, which is itself
  marked optional for exactly that reason.
- Commit after each phase checkpoint, not after every task.
- T020 is the one task with real-world consequence beyond this repository (a live redeploy) —
  confirm with the operator before running it.
