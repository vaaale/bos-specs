# Discrepancies Log

Spec/code drift and post-converge findings. Newest first. Per project convention, drift notes about system (read-only) specs are tracked here in `user-specs`, not alongside the original spec.

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
