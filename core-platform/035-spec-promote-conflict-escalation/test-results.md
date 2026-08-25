# Test Results: 035 Git Conflict Resolution System

**Status**: PASSED

**Date**: 2026-08-24
**Branch**: `bos/035-spec-promote-conflict`
**Environment**: isolated `/tmp` checkout (live `:3000`/`:3002` servers are a different checkout / stale build)

## Unit tests

**256/256 passing** in `tests/gitops` (39 new tests for this feature).

- `reconcile()` generalization: each step's outcome (success, conflict, failed, escalated), the new session creation path, and the `onEscalate` callback
- `readFileAtRef`: normal read, add/add base-side catch → `null`, non-existent ref
- Session store: create/get/list/update/transition, `writeFileAtomic` atomicity, `TERMINAL_STATUSES` enforcement
- `inFlightEscalations` guard: concurrent reconcile on same repo re-points to existing session; `awaiting-user` sessions are treated as active
- `recoverSessions()`: working session with dead run → re-launch + re-emit; awaiting-user → re-emit only; double-emit is benign
- Session API: transitions, decision answer recording, park-and-rewake re-launch
- Config: `build-studio.conflictAgent` read at escalation time; non-default agent path
- FR-016 completeness: no git conflict path returns a static "resolve manually" without routing through `reconcile()`

## E2E tests (Playwright)

**13/13 passing** — `e2e/035-conflict-resolution.spec.ts`

| # | Scenario | Result |
|---|---|---|
| S1 | User-specs add/add on Supervisor feature-promote (the reported bug) | ✅ |
| S2 | Source-repo promote escalation does not regress (FR-023) | ✅ |
| S3 | Pull (git-remotes fetch) divergence (FR-013) | ✅ |
| S4 | Push-recovery non-FF (FR-014) | ✅ |
| S5 | App-candidate promote (user-apps) (FR-015) | ✅ |
| S6 | VFS-mounted repo (cross-repo generality, FR-003/004) | ✅ |
| S7 | Autonomous-first: agent resolves unambiguous, asks on ambiguity (D2) | ✅ |
| S8 | Binary file loud-fail (FR-022) | ✅ |
| S9 | Restart recovery (FR-024) — working session re-launch + awaiting-user restore | ✅ |
| S10 | Rollback / abandon (rollback affordance) | ✅ |
| S11 | Configurable agent takes effect (FR-025) | ✅ |
| S12 | Concurrent-op guard (FR-022 / design S3) | ✅ |
| — | FR-016 completeness invariant (no dead-end without agent) | ✅ |

## Quality gates

| Gate | Result |
|---|---|
| `tsc --noEmit` | ✅ clean |
| `lint` | ✅ clean |
| Unit tests (256) | ✅ all passing |
| E2E tests (13) | ✅ all passing |

## Notes

- The agent's own resolution turns are driven by **unit tests** (a real escalation starts a real model run, which the harness can only script for the first message). The decision loop, completion, rollback, and boot-sweep are covered by unit tests that drive those exact code paths against real git repos.
- The e2e ran against an **isolated `/tmp` copy** (since removed), not the live `:3000`/`:3002` servers.
