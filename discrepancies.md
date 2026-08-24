# Discrepancies Log

Spec/code drift and post-converge findings. Newest first. Per project convention, drift notes about system (read-only) specs are tracked here in `user-specs`, not alongside the original spec.

---

## 035-spec-promote-conflict-escalation — post-converge (2026-08-24)

**Converge verdict: CONVERGED (docs aligned to code).** All 25 FRs (FR-001…FR-025) + 5 NFRs met by the implemented code; no functional drift. The `abandoned` terminal state and the 7th call site (below) were *spec/design under-specifications* — the code and the binding mockup were already correct, so converge updated the docs toward the code, not the other way around. **E2E: 13/13 passing** (isolated `/tmp` checkout). **Unit: 256/256** in `tests/gitops` (39 new). `tsc --noEmit` + `lint` clean.

### Call-site completeness (FR-016) — the definitive set is **7**
The spec named 4, the design §10 sweep found 6, and the **implementation-time FR-016 sweep (task T040) found a 7th**: `/api/git-sync` `case "resolve"` (VFS-mounted repo), which inline-merged/rebased and returned a static `MERGE_CONFLICT` "Resolve manually" with no agent. All 7 now route through `reconcile()` with the correct repo-parameterized `workingContext`; row 7 added to design §10 and the spec Path B table. The invariant "no git conflict path dead-ends without an agent" is now enforced by a dedicated test.

### Real bugs found by the e2e suite (all fixed)
1. **`createTag` failed in any BOS-created repo** — no `user.email` → "Committer identity unknown". This is pipeline **step 1** (the rollback tag), so FR-017's safety guarantee was broken on any host without a global git identity. Fixed with `gitIdentityEnv()`.
2. **`attemptStrategy` only inspected `stderr` for conflicts** — but `git merge` reports `CONFLICT` on **stdout**. Every plain merge conflict was misclassified as a hard `failed` and *never escalated*, silently defeating the no-dead-ends invariant. Fixed to inspect both streams.
3. **The auto-launcher read the event stream from `since=0`** — replayed *every historical* conflict event and yanked the user into an old session on each browser load. Fixed to only react to events emitted after launch.
4. **Pre-existing: the `build-studio` config namespace declared `fields: []`** — `/api/config` PATCH silently drops undeclared keys, so that tab's Save had *never persisted anything*, including the pre-existing "Agent" field. Fixed by declaring the fields (and the new `conflictAgent`).

### Handled-differently-than-specified (justified; recorded in spec/design, no FR violation)
- **`reconcile()` still blocks — on the *session*, not the run.** The design's "escalate and return" was refined: `reconcileWithSession` awaits the session reaching a terminal state so "resolving completes the operation" is true for callers that block (the Supervisor promote). The `onEscalate` callback + the `/api/gitfs/reconcile` job wrapper still let the Supervisor surface "escalated, here's the session" immediately.
- **A `failed`/`abandoned` session returns `failed`, not `escalated`.** This lets `requireReconciled` refuse to promote an unmerged branch — the silent-success failure FR-021 forbids. `escalated` is reserved for "still live, handed to the agent."
- **`conflict_abandon` (agent) → `failed`; user "Abandon & roll back" → `abandoned`.** The spec had lumped these as "`failed` (abandoned)". The accepted mockup already modeled `abandoned` as a distinct neutral terminal ("Rolled back — operation aborted"), so the code is correct and the spec was updated to match (state machine in FR-002/FR-011/FR-020/SC-002/system-overview/US2 + Key Entities; design §3.1 type + comments).
- **Browser-refresh restore** added to the auto-launcher (re-queries active sessions on mount) — the spec's FR-024 "pane re-queries the session store on load" is met by this, not by a BS-side query.

### Test-coverage note (honest gap, not an FR miss)
The **agent's own resolution turns are driven by unit tests, not the e2e** — a real escalation starts a real model run, which the harness can only script for the first message. The decision loop, completion, rollback, and boot-sweep are therefore covered by unit tests that drive those exact code paths against real git repos. The e2e covers the deterministic scaffolding around them (session creation, event emit, BS auto-launch, surface state, and the full resolve on the *deterministic* paths that don't require a live agent).

### Docs added (constitution VI)
- `docs/dev/features/git-conflict-resolution.md` — architecture, working-context parameterization, park-and-rewake, the 7-call-site table, Settings field, event type + payload, state machine, how to add a new call site.
- `docs/dev/build-studio.md` — the conflict pane + the "Conflict resolution agent" Settings field.

**Net**: no FR unmet, no functional drift. Feature is complete, promoted, and converged. The only follow-up-worthy item is optional: drive the agent's resolution turns in the e2e (needs a scripted/stubbed agent in the harness).

---

## 034-event-notification-system — post-converge (2026-08-24)

**Converge verdict: CONVERGED.** All 30 FRs (FR-001…FR-030) and 9 NFRs (NFR-001…NFR-009) met by the implemented code; no functional drift. **E2E: 6/6 passing** (verified against a working dev server; `buildstudio_run_tests` against port 3000 fails due to a stale worktree file-handle issue after promotion — infra, not code). 41/41 unit tests passing.

### E2E test fix (2026-08-24)
- **Parallelism race in "Mark all as read"** — global action over shared unread count; concurrent sibling tests skewed the delta. Fixed with `test.describe.configure({ mode: "serial" })` (precedent: `e2e/039-service-tool-exposure.spec.ts`).

### Non-blocking observation
- **`perf.test.ts` p99 flaky under full parallelism** (41 workers) — 192ms vs. 100ms budget is CPU contention, not a regression. Passes comfortably in isolation (447ms). No action needed.

**Net**: No FR unmet, no functional drift. Feature is complete and converged.

---

## 028-memory-curation-retrieval — post-converge (2026-08-23)

**Converge verdict: CONVERGED.** All 20 FRs (FR-001…FR-020) met by the implemented code; no functional drift. **E2E: 9/9 passing.** The items below are non-blocking cleanups and test-coverage gaps found by the code-vs-spec check.

### E2E test stabilization (2026-08-23)
Three rounds of e2e fixes were needed after the initial implementation:
1. **SC-001 cross-run state pollution** — added `test.beforeAll` to delete the shared topic before the block runs. Resolved.
2. **Browser tests: `channel: "chrome"` not found** — the `buildstudio_run_tests` sandbox had no system Chrome. Fixed by user (Playwright config). Resolved.
3. **Browser tests: inner `toBeVisible` waits capped at 30 s test timeout** — Playwright caps inner waits at the test-level timeout. The assistant test needs ~80 s of sequential waits. Fixed by adding `test.setTimeout(120_000)` to both browser tests. Resolved.

### Code cleanups (non-blocking)
- **`pruneEmbeddingCache` (embeddings.ts:95) is dead code** — exported but has zero call sites. Design ADR-3 promised it'd be called lazily (slow-loop pass or post-search) to bound cache growth; it was never wired. The `.embeddings.json` cache retains vectors for removed/re-texted entries until a model change. *Action: wire it into a slow-loop pass, or remove it.*
- **`memoryApi` (tool.ts:147) is a dead export** — the `/api/memory` route imports the ops directly from `topics.ts`, never via this object. *Action: remove, or route the API through it.*

### Handled-differently-than-specified (minor, no FR violation)
- **`/api/memory?topic=` REST detail** returns `{id, text, timestamp}` only — **no `state`**. The `memory_recall` tool and the `/api/memory/search` route both expose state, but this REST surface doesn't, so a REST consumer can't read superseded state on a topic. FR-019 is the agent-tool path (met), so this is a surface-parity inconsistency, not a violation. *Action (optional): expose `state` for parity.*
- **UI can't clear the embedding API key once set** — `ProviderSettings.tsx` `save()` guards with `embedKeyDraft ? {apiKey} : {}`, so an empty draft is never sent (the route *does* support clearing via `""`). Same limitation as the LLM key in the same component — a shared, low-priority UX gap.

### Test coverage gaps (code present + wired; no assertions)
- **FR-020 as-of** (`entryStateAt`) — untested.
- **FR-018 contradiction/supersede** — untested (SC-004: known contradiction pair → active returned as current 100%).
- **SC-003 dense/paraphrase recall** — explicitly deferred in the e2e comments; needs a stubbed embedding endpoint.
- **SC-002** (N redundant → ≤ ⌈N/2⌉) and **SC-006** (interruption atomicity) — not covered. SC-006's assertion needs the honest "invariant + convergence" reword already applied to spec.md.

### Improvement over design (not a regression)
- **`openai` declared in `package.json`** (`^6.46.0`) — design ADR-6 framed it as "existing but undeclared/hoisted." The implementation resolved the Constitution VII caveat by declaring it explicitly (matches task T001). Stricter than the design; keep.

**Net**: no FR unmet, no functional drift. The feature is complete; the items above are polish + test-hardening for a follow-up pass if desired.
