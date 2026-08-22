# Tasks: Memory Curation & Retrieval

**Branch**: `bos/028-memory-curation-retrieval`
**Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md) | **Design**: [design.md](./design.md) | **Mockup**: [mockup.html](./mockup.html) | **Quickstart**: [quickstart.md](./quickstart.md)
**App Target**: `bos-core`

> Tasks are grouped by user story (priority order) and dependency-ordered. `[P]` = parallelizable (different files, no dependency on an in-flight task). The test strategy (spec + plan) is explicit, so test tasks are included and tagged with the success criterion / FR they verify.

---

## Phase 1: Setup

- [ ] T001 Declare `openai` explicitly in `package.json` (Constitution VII exception — already a de-facto runtime dep via `src/lib/agent/llm.ts`; `embed()` will depend on it directly). Blocking for Phase 6 (US4).

## Phase 2: Foundational (shared topic-store primitive)

- [ ] T002 Extend `Topic` with `consolidate?: boolean` — parse/serialize as a `<!-- needs-consolidation -->` marker line (a non-`>` line, invisible to `rebuildMemoryIndex`); make `currentBudget` exclude **only** the marker line (keep the existing digest+entries baseline). `src/lib/agent/memory/topics.ts`

---

## Phase 3: User Story 1 — Soft budget (Priority: P1)

**Goal**: A save to a full topic succeeds and flags the topic for consolidation instead of hard-rejecting (the reported bug).
**Independent Test**: Fill a topic to just under budget, save one more that overflows → success + flag.

- [ ] T003 [US1] Change `addTopicEntry`/`replaceTopicEntry` over-budget behavior from the hard "Over budget" reject to **accept the write + set the `consolidate` flag** (FR-001). Within-budget saves set no flag. `src/lib/agent/memory/topics.ts`
- [ ] T004 [P] [US1] Verify **SC-001**: save to an at/over-budget topic returns success (no "Over budget") and sets the flag; a within-budget save sets no flag; the marker line does not change the reported budget. (test)

## Phase 4: User Story 2 — Agent self-edit (Priority: P1)

**Goal**: The agent can replace/remove its own entries to tidy a topic (no forced `-2` shard).
**Independent Test**: Scripted "tidy this topic" completes via the new tools with no shard.

- [ ] T005 [US2] Add `memory_replace(topic, entryIdOrText, content)` and `memory_remove(topic, entryIdOrText)` backed by `replaceTopicEntry`/`removeTopicEntry` — keyed by entry id with the existing unique-substring fallback; not-found → clear error + no change; dedup on replace (FR-002, FR-003, FR-004). `src/lib/agent/memory/tool.ts` + `src/lib/assistant/tools/server/memory.ts`
- [ ] T006 [US2] Register `memory_replace`/`memory_remove` in `src/lib/agent/capabilities-registry.ts` with `context: "action"` (parity with the `memory_*` family); update the `memory_search`/`memory_recall` tool descriptions to reflect the new capabilities.
- [ ] T007 [P] [US2] Verify **SC-005** + US2 acceptance: a scripted agent "tidy this topic" completes via replace/remove with **no** `-2` shard; replace by id updates; remove by id deletes; unknown id → not-found error + no change; replace-to-duplicate leaves no duplicate text. (test)

## Phase 5: User Story 3 — Holistic consolidation (Priority: P1)

**Goal**: The slow loop can read a topic and reorganize it (merge/dedup/split + refresh digest + clear flag), and actually runs for a flagged topic even with no pending episodes.
**Independent Test**: Seeded redundant topic → after a pass, ≤ ⌈N/2⌉ entries, digest refreshed, flag cleared.

- [ ] T008 [US3] Add a `topic_read` op to the slow loop returning the full entry list of a topic (FR-005). `src/lib/agent/memory/consolidate.ts`
- [ ] T009 [US3] Add reorganization guidance to the slow-loop system prompt (merge = remove×N + add×1; split = create + add×N + remove×N; **moves, never deletes-then-recreates**; refresh digest; leave a clean, non-redundant topic substantially unchanged) + a `topic_set_digest` op (FR-006, FR-008); add `setTopicDigest(agentId, slug, digest)` to the topic store. `src/lib/agent/memory/consolidate.ts` + `src/lib/agent/memory/topics.ts`
- [ ] T010 [US3] Widen the `runSlowLoop` gating so it runs a pass when there are **pending episodes OR ≥1 flagged topic** (the M1 fix); `renderAgentPreamble`/`consolidateAgent` enumerate the flagged topics. `src/lib/agent/memory/consolidate.ts`
- [ ] T011 [US3] Clear the source topic's flag in a final write **only after** a successful reorganization (FR-007); the flag survives an interrupted pass (it lives in the source file) so the next pass converges (SC-006). `src/lib/agent/memory/consolidate.ts`
- [ ] T012 [P] [US3] Verify **SC-002** (seeded N-redundant topic → ≤ ⌈N/2⌉ entries, no two with the same meaning, digest refreshed, flag cleared), **SC-006** (inject a mid-pass fault during a split → every topic file valid, no entry lost/duplicated, flag survives, next pass converges), **FR-005** (loop can read entries), **FR-007** (flag cleared only on success), **FR-008** (clean topic unchanged), and **M1** (a flagged topic with zero pending episodes is still processed). (test)

## Phase 6: User Story 4 — Hybrid retrieval + provider embedding endpoint (Priority: P2)

**Goal**: `memory_search` ranks by fused dense + sparse + recency + importance with a floor + provenance + cache + graceful degradation; the AI provider config gains an embedding endpoint with per-field fallback; the Settings UI shows it per the mockup.
**Independent Test**: Paraphrase retrieves via dense, keyword via sparse, most-recent ranks higher, embeddings-off degrades with no error.

- [ ] T013 [US4] Extend `ProviderConfig` with `embeddings?: { baseUrl?: string; apiKey?: string; model?: string }`; add `resolveEmbeddingConfig(c)` implementing **per-field** fallback; extend `getProviderConfigView()` with `embedBaseUrl` (resolved, non-secret), `hasEmbeddingKey` (boolean), `embeddingsEnabled`; merge `embeddings` **per-field** in `updateProviderConfig` (per-field `""`→clear, `undefined`→unchange) (FR-009, FR-010). `src/lib/agent/provider.ts`
- [ ] T014 [P] [US4] Add `defaultEmbedModel?` and `embedAvailability: "available" | "unsupported" | "unknown"` to `PROVIDERS` in `src/lib/agent/provider-meta.ts` (openai/codex/responses → `text-embedding-3-small`; openai-compatible → none; anthropic → `unsupported`) so the mockup `PROVIDERS` table and real defaults can't diverge.
- [ ] T015 [US4] Add `embeddingClient(resolved)` (OpenAI-family only; anthropic → `null`) and `embed(resolved, text): Promise<number[] | null>` returning `null` on 404/auth/unsupported (the degradation signal, not an error); reuse `normalizeApiBase`. `src/lib/agent/llm.ts`
- [ ] T016 [US4] Provider API route: PATCH accepts and forwards the `embeddings` object (per-field); GET returns the extended view. `src/app/api/agent/provider/route.ts`
- [ ] T017 [US4] Provider test route: extend the test action to attempt a 1-token `embed()` and report `embeddings.available` + error **as a distinct field** from the LLM completion probe. `src/app/api/agent/provider/test/route.ts`
- [ ] T018 [P] [US4] Add `ai-provider` `fields[]` for `embeddings.{baseUrl, apiKey, model}` with `secret: true` on the key in `src/lib/config/registry.ts` (the config-registry redaction path, distinct from the view's boolean — R8, S5).
- [ ] T019 [US4] Create `src/lib/agent/memory/embeddings.ts` with `getEmbedding(agentId, { id, text })` wrapping `embed()` + a per-agent cache at `/Memories/<agentId>/.embeddings.json` (key = content-hash entry id, per-entry `model`, lazy recompute on model change, prune orphaned ids); add `agentEmbeddingsFile(agentId)` to `src/lib/agent/memory/paths.ts` (FR-013, ADR-3).
- [ ] T020 [US4] Rework `src/lib/agent/memory/search.ts`: fused score `0.4·dense + 0.3·sparse(BM25) + 0.2·recency(exp(-age_days/30), neutral 0.5) + 0.1·importance(neutral 0.5)`, **renormalized** over the available signals; relevance floor on `max(dense, sparse) ≥ ~0.15` (below → drop; none pass → empty result) (FR-012, FR-014, FR-015, FR-017); graceful degradation — if the query `embed()` is `null` or dense disabled, drop the dense term and renormalize, **no error** (FR-016); **bounded cold burst** — on a cold cache embed only the top-K (≈8–10) sparse candidates, not the whole corpus (R3); retain per-result provenance.
- [ ] T021 [US4] `src/components/apps/ProviderSettings.tsx`: add the **Embeddings** subsection — Base URL (with "Leave blank to use the LLM provider's base URL above" hint + `Uses: <embedBaseUrl>` placeholder), API key (password field + has-key/fallback indicator driven by `hasEmbeddingKey`/`hasApiKey`, value never shown), Model (per-provider default from `provider-meta`, anthropic "no first-party embeddings" note); availability indicator (`available` / `not supported` / `unknown`) from `provider-meta.embedAvailability` + Test result (**no auto-probe**); "new" badge; bind all three fields to the PATCH `embeddings` object with per-field semantics. **Must match `mockup.html`.**
- [ ] T022 [P] [US4] Verify **SC-003** (fixed set: most-relevant-and-recent is top-1 in ≥90%; a paraphrase with no shared keywords retrieves via dense; a keyword query retrieves keyword-only entries via sparse), **SC-007** (embeddings disabled → ranked results, **zero errors**), **FR-013** (2nd identical search issues no corpus `embed()` for cached entries; an entry text change → recompute), **FR-014** (no-match → empty), **FR-015** (provenance per result), **FR-011 / R8** (embedding key absent from the provider API view **and** the config-registry load), and a Settings e2e matching `mockup.html`. (test)

## Phase 7: User Story 5 — Temporal validity (Priority: P3)

**Goal**: Entries carry an active/superseded lifecycle; a detected contradiction marks the older entry superseded (not deleted); default retrieval returns current, as-of returns history.
**Independent Test**: Contradiction pair → older superseded; default returns the new one; as-of returns the historical state.

- [ ] T023 [US5] Extend `TopicEntry` with `state?: "active" | "superseded"` and `supersededBy?: { id: string; timestamp: string }`; extend the `ENTRY_LINE` parser to optionally capture a `⟦superseded by=<entryId>@<ts>⟧` tag (absent → `active`); keep the existing text-regex `superseded` heuristic as a **read-time legacy fallback** (ADR-5); pin the hash invariant `id = hashId(clean entry text)` where the tag + marker are excluded from the hash (S4). `src/lib/agent/memory/topics.ts`
- [ ] T024 [US5] Add `supersedeTopicEntry(agentId, slug, entryIdOrText, supersededBy)` — marks the entry superseded with a reference to the superseding entry, without deleting it (FR-018). `src/lib/agent/memory/topics.ts`
- [ ] T025 [US5] Add a `topic_supersede` op and contradiction-detection guidance to the slow-loop system prompt (the loop `topic_read`s both entries and marks the older superseded → newer; best-effort, background — not inline). `src/lib/agent/memory/consolidate.ts`
- [ ] T026 [US5] Default retrieval returns active entries as current and labels superseded "not current" (FR-019); add `state` to `SearchResult`; implement as-of `entryStateAt(e, T)` = active iff `e.timestamp ≤ T` AND NOT(superseded AND `supersededBy.timestamp ≤ T`), exposed via an optional `asOf` parameter on `memory_recall` (FR-020). `src/lib/agent/memory/search.ts` + `src/lib/agent/memory/tool.ts` + `src/lib/assistant/tools/server/memory.ts`
- [ ] T027 [P] [US5] Verify **SC-004** (known contradiction pair → older superseded with ref; default retrieval returns the active one as current in 100% of cases; the superseded one is never returned as current), **FR-018** (supersede marks + references without deleting), **FR-019** (default labels superseded "not current"), **FR-020** (as-of returns the correct historical state), and **ADR-5/S4** (a legacy file with no tag parses as active via the fallback; the id hash uses clean text so the tag doesn't change the id). (test)

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T028 [P] Update docs (Constitution VI): `docs/dev/memory/memory.md` (soft budget, lifecycle, hybrid search, embedding endpoint, cache) + `docs/usage/memory/*` + the Memory-app doc (agent self-cures; embeddings config; active/superseded labeling).
- [ ] T029 Author the full Playwright suite `e2e/028-memory-curation-retrieval.spec.ts` (Settings vs `mockup.html`; agent-tool round-trip: save-soft-budget / replace / remove; search + degradation) and run it via `buildstudio_run_tests` (writes `test-results.md` into this folder).
- [ ] T030 Run `/speckit.analyze` (cross-artifact consistency) + `/speckit.converge` (code-vs-spec drift); record any drift in `specs/discrepancies.md`.
- [ ] T031 Final quality gates: `npx tsc --noEmit` and `npm run lint` clean (Constitution VII).

---

## Dependencies

- **Phase 1 → Phase 6**: T001 (openai) blocks T015 (embed).
- **Phase 2 → Phases 3/5/7**: T002 (flag) is consumed by T003 (US1 sets it), T010 (US3 reads/gates on it), and coexists with T023 (US5 lifecycle tag).
- **US1 (Phase 3) → US3 (Phase 5)**: the soft-budget flag must exist before the consolidation loop reads/gates on it.
- **US4 (Phase 6)**: T013 → T015 → T016/T017; T013 → T021 (view fields); T014/T015 → T021 (defaults + availability); T015 → T019 (embed cache); T019 + T015 → T020 (search). T014, T018 are independent of the rest of the provider layer.
- **US5 (Phase 7)**: T023 → T024 → T025/T026. Independent of US4 except T026 also touches `search.ts` (after T020).
- **Phase 8**: T029 depends on all user-story phases; T030/T031 depend on T029.

## Parallel execution examples

- **Provider layer (Phase 6)**: T014 (provider-meta) ∥ T018 (config registry) ∥ T013 (provider.ts) — different files, no in-flight deps.
- **Test tasks**: every `[P]` test task (T004, T007, T012, T022, T027) is independent of the other test tasks and can run alongside its story's implementation once that story's code tasks land.

## Implementation strategy (MVP first)

- **MVP (first shippable increment)**: **US1 (soft budget) + US2 (agent self-edit)** — Phases 1–4. This is the direct fix for the reported "Over budget / forced shard" failure and is independently valuable (the plan's checkpoint after Phase 3). Promote here if desired.
- **Next**: US3 (holistic consolidation, Phase 5) — makes the soft budget safe by reorganizing flagged topics.
- **Then**: US4 (hybrid retrieval + embedding endpoint, Phase 6) — the P2 retrieval quality win.
- **Finally**: US5 (temporal validity, Phase 7) — the P3 trust win — then Polish (Phase 8).

---

## Format validation

All 31 tasks follow the checklist format (`- [ ]` + Task ID + optional `[P]` + optional `[USn]` + description with exact file path). Story-labeled tasks map to spec user stories 1–5; Setup/Foundational/Polish carry no story label.
