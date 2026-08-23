# Implementation Plan: Memory Curation & Retrieval

**Branch**: `bos/028-memory-curation-retrieval`
**Spec**: [spec.md](./spec.md) | **Design**: [design.md](./design.md) | **Mockup**: [mockup.html](./mockup.html)
**App Target**: `bos-core`

> **Purpose**: Translate the approved design into a phased, independently-testable implementation plan with explicit gate criteria. This is the input to `/speckit.tasks`; it does not restate requirements (those live in `spec.md`) nor re-derive architecture (those live in `design.md`).

---

## Technical Context

**Language/Version**: TypeScript (strict) on the existing Next.js (App Router) + React codebase — no new runtime language.
**Primary Dependencies**: `openai` npm SDK (already imported by `src/lib/agent/llm.ts`; **to be explicitly declared in `package.json`** — see Gate VII below). No other new dependencies.
**Storage**: Per-agent files under `/Memories/<agentId>/` (VFS) — topic `.md` files (unchanged schema + new marker/tag), a new `.embeddings.json` sidecar cache. Provider config in `provider.json` (existing store). No new database.
**Testing**: Playwright e2e (via `buildstudio_run_tests`, `e2e/028-memory-curation-retrieval.spec.ts`) for the settings surface + agent-tool round-trip; focused unit-style assertions on the pure functions (fusion scoring, fallback resolution, topic serializer round-trip) where a pure function is the unit. Quality gates: `npx tsc --noEmit` and `npm run lint` (Constitution VII).
**Target Features**: The five slices — T2 soft budget (P1), T6 agent replace/remove (P1), T1 holistic consolidation (P1), T3 hybrid retrieval + provider embedding endpoint (P2), T4 temporal validity (P3).
**Performance Goals**: A cold `memory_search` issues ≤ 1 query-embed + ≤ K corpus-embeds (K ≈ 8–10, the top-K sparse candidates) — bounded, not O(corpus) (design §5.5, R3). Steady-state search is 1 query-embed + cached vectors.
**Constraints**: No whole-file topic rewrite (ACE anti-collapse rule — design ADR-2). Embedding API key never leaves the server (FR-011). Graceful degradation when a provider can't serve embeddings (FR-016). `rebuildMemoryIndex` stays untouched.

### Constitution compliance notes
- **VII (package.json)**: declaring `openai` explicitly is a *one-line, explicitly-flagged* tradeoff the design calls out (ADR-6, §Constitution VII). It touches the lockfile. This is the single boundary exception in the plan; it is justified by the fact that `openai` is already a de-facto runtime dependency (imported by `llm.ts`, resolving today as a hoisted transitive) and the new `embed()` depends on it directly. If the user prefers zero lockfile churn, the fallback is to keep it undeclared — but that perpetuates a fragile hoist. **Decision recorded here: declare it explicitly.**
- **II (server authority)**: `embed()` lives in `src/lib/agent/` (server-only, beside `provider.ts` which is `"server-only"`). The embedding key is returned only as `hasEmbeddingKey` (boolean). ✅
- **V (VFS not source)**: all new runtime state (`.embeddings.json`, flag, tags) is under `/Memories/<agentId>/`, never `src/`. ✅
- **VI (docs)**: Phase 6 updates `docs/dev/memory/memory.md` + `docs/usage/memory/*` in the same change. ✅

---

## Phase 1: Provider embedding endpoint (foundation for T3)

**Purpose**: Add the embedding endpoint to the AI provider config layer so everything else that needs embeddings (search, test) can call into a resolved config. This is the substrate; nothing else in the plan depends on it being *used* yet — it just needs to be *configurable and resolvable*.

**Complexity**: None.

**Tasks**:
- **T1.1** (`src/lib/agent/provider.ts`): Extend `ProviderConfig` with `embeddings?: { baseUrl?: string; apiKey?: string; model?: string }`. Add `resolveEmbeddingConfig(c)` implementing the per-field fallback (design §5.1). Extend `getProviderConfigView()` with `embedBaseUrl` (resolved, non-secret), `hasEmbeddingKey` (boolean), `embeddingsEnabled`. Extend `updateProviderConfig` to merge `embeddings` **per-field** (per-field `""` → clear, `undefined` → unchanged) — *not* an all-or-nothing object merge (design S3).
- **T1.2** (`src/lib/agent/provider-meta.ts`): Add per-provider `defaultEmbedModel?` (openai/codex/responses → `text-embedding-3-small`; openai-compatible → none; anthropic → none) and `embedAvailability: "available" | "unsupported" | "unknown"` (anthropic → `unsupported`). Keep this file the single source so the mockup's `PROVIDERS` table and real defaults can't diverge (design §8).
- **T1.3** (`src/lib/agent/llm.ts`): Add `embeddingClient(resolved)` (OpenAI-family only; anthropic → `null`) and `embed(resolved, text): Promise<number[] | null>` returning `null` on 404/auth/unsupported (the degradation signal, not an error). Reuse `normalizeApiBase`.
- **T1.4** (`src/app/api/agent/provider/route.ts`): PATCH accepts and forwards the `embeddings` object; GET returns the extended view.
- **T1.5** (`src/app/api/agent/provider/test/route.ts`): Extend the test action to attempt a 1-token `embed()` and report `embeddings.available` + error **as a distinct field** from the LLM completion probe (design N1).
- **T1.6** (`src/lib/config/registry.ts`): Add the `ai-provider` `fields[]` for `embeddings.{baseUrl,apiKey,model}` with `secret: true` on the key (design S5 — this is the config-registry redaction path, distinct from the view's boolean).
- **T1.7** (`package.json`): Declare `openai` explicitly (Gate VII).

**Files**: `provider.ts`, `provider-meta.ts`, `llm.ts`, `app/api/agent/provider/route.ts`, `app/api/agent/provider/test/route.ts`, `lib/config/registry.ts`, `package.json`.

**Verification**:
- `resolveEmbeddingConfig` returns the right base URL/key/model/enabled for: both fields empty (→ LLM values), only base URL empty, only key empty, both set, anthropic (→ `enabled: false`), blank model (→ `enabled: false`). (Pure-function assertions.)
- `getProviderConfigView()` never contains the embedding key value (only `hasEmbeddingKey`); the config-registry load redacts it via `secret: true`. (FR-011, R8 — both paths.)
- PATCH round-trips `embeddings` per-field: clearing the key to `""` clears only the key, leaving the base URL intact. (SC edge: per-field fallback.)
- Typecheck + lint pass.

**Checkpoint**: ✅ Embedding endpoint is configurable, resolvable, and redacted. *Do not proceed to Phase 2 until this is green.*

---

## Phase 2: Settings UI — Embeddings subsection (user-facing, mockup-bound)

**Purpose**: Make the embedding endpoint visible and editable in Settings → AI Provider, matching `mockup.html` exactly (the binding UI contract).

**Complexity**: None.

**Tasks**:
- **T2.1** (`src/components/apps/ProviderSettings.tsx`): Add the **Embeddings** subsection — Base URL (with "Leave blank to use the LLM provider's base URL above" hint + `Uses: <resolved>` placeholder from `embedBaseUrl`), API key (password field + has-key indicator driven by `hasEmbeddingKey`/`hasApiKey`, value never shown), Model (per-provider default from `provider-meta`, with the "no first-party embeddings" note for anthropic). "New" badge on the subsection.
- **T2.2** (same file): Availability indicator (`available` / `not supported` / `unknown`) driven by `provider-meta.embedAvailability` **and** updated by the Test connection result (not an auto-probe). Wire the Test button to the extended `/api/agent/provider/test` and surface the distinct embeddings probe result.
- **T2.3** (same file): Bind all three fields to the PATCH `embeddings` object; ensure per-field semantics (a blank field is sent as `""` to clear, an untouched field is omitted).

**Files**: `src/components/apps/ProviderSettings.tsx`.

**Verification** (Playwright e2e, mockup as visual reference):
- The Embeddings subsection renders with all three fields, the per-field fallback hints, the has-key indicator, the "new" badge, and the availability indicator — matching `mockup.html`.
- The API key field never displays a value (set → indicator shows "set"; blank + LLM key present → "fallback"). (FR-011.)
- Saving persists `embeddings`; a blank Base URL resolves to the LLM base URL in the placeholder. (FR-010.)
- Changing provider to Anthropic shows the "no first-party embeddings" note and `not supported` availability.
- Typecheck + lint pass.

**Checkpoint**: ✅ Settings surface matches the mockup and round-trips the embedding config. *Do not proceed to Phase 3 until this is green.*

---

## Phase 3: Soft budget + agent self-edit (T2 + T6 — the reported-bug fix)

**Purpose**: Stop the hard "Over budget" rejection and give the agent the tools to tidy topics. This is the direct fix for the failure that started this feature, and it is the safe-to-ship core (independent of retrieval and consolidation).

**Complexity**: None.

**Tasks**:
- **T3.1** (`src/lib/agent/memory/topics.ts`): Add `Topic.consolidate?: boolean` parsed/serialized as a `<!-- needs-consolidation -->` marker line (a non-`>` line, invisible to `rebuildMemoryIndex`). In `addTopicEntry`/`replaceTopicEntry`, replace the hard budget reject with **accept + set the flag** (FR-001). `currentBudget` keeps its existing baseline and **excludes only the marker line** (design S2), so usage reporting is unaffected.
- **T3.2** (`src/lib/agent/memory/tool.ts` + `src/lib/assistant/tools/server/memory.ts`): Add `memory_replace(topic, entryIdOrText, content)` and `memory_remove(topic, entryIdOrText)` backed by the existing `replaceTopicEntry`/`removeTopicEntry`, keyed by id with the existing unique-substring fallback. Not-found → clear error, no change. Dedup on replace (FR-004). (T6.)
- **T3.3** (`src/lib/agent/capabilities-registry.ts`): Register `memory_replace`/`memory_remove` with `context: "action"` (parity with the `memory_*` family).
- **T3.4** (both memory tool files): Update the `memory_search`/`memory_recall` tool descriptions to reflect the new capabilities.

**Files**: `topics.ts`, `memory/tool.ts`, `assistant/tools/server/memory.ts`, `capabilities-registry.ts`.

**Verification**:
- **SC-001**: Save to a topic at/over budget returns success (not "Over budget") and sets the flag. (Direct regression test for the reported bug.)
- **SC-005**: A scripted agent "tidy this topic" completes via `memory_replace`/`memory_remove` with **no** `-2` shard created.
- US2 acceptance: replace by id updates; remove by id deletes; unknown id → not-found error, no change; replace-to-duplicate handled.
- Topic serializer round-trips the flag (set → present line; cleared → absent line) and the flag does not change the reported budget.
- Typecheck + lint pass.

**Checkpoint**: ✅ The reported bug is fixed and the agent can self-edit. **This is the first shippable increment** — consider promoting here if desired. *Do not proceed to Phase 4 until this is green.*

---

## Phase 4: Holistic consolidation (T1 — makes the soft budget safe)

**Purpose**: Give the slow loop the ability to *read* a topic and reorganize it (merge/dedup/split + refresh digest + clear flag), and widen its driver so a flagged topic is actually processed even with zero pending episodes.

**Complexity**: **Justified** — this is the highest-risk phase (interruption semantics, ADR-2). It is scoped to the existing atomic single-entry ops (no new whole-file primitive), which is *why* it's bounded. The complexity is in the loop's gating + the invariant-enforcing op sequences, both fully specified in design §5.4.

**Tasks**:
- **T4.1** (`src/lib/agent/memory/topics.ts`): Add `setTopicDigest(agentId, slug, digest)`.
- **T4.2** (`src/lib/agent/memory/consolidate.ts`): Add `topic_read` op (returns the full entry list, FR-005), `topic_set_digest` op, and the reorganization guidance (merge = remove×N + add×1; split = create + add×N + remove×N; **moves, never deletes-then-recreates**; refresh digest). The preamble tells the loop to leave well-organized topics unchanged (FR-008).
- **T4.3** (`src/lib/agent/memory/consolidate.ts` — **driver, M1**): Widen `runSlowLoop` gating so it runs a pass when there are pending episodes **OR ≥1 flagged topic** (enumerate the agent's topic set for the `consolidate` flag). `renderAgentPreamble`/`consolidateAgent` enumerate the flagged topics.
- **T4.4** (`src/lib/agent/memory/consolidate.ts`): Clear the source topic's flag in a final write **only after** a successful reorganization (FR-007). The flag survives interruption (it's in the source file) ⇒ the next pass converges (design ADR-2, SC-006).

**Files**: `topics.ts` (T4.1), `consolidate.ts` (T4.2–T4.4).

**Verification**:
- **SC-002**: A seeded topic with N redundant entries → after one pass, ≤ ⌈N/2⌉ entries, no two expressing the same meaning (deterministic fixture).
- **SC-006**: Force a mid-pass failure (inject a fault after a partial split); assert every topic file is valid, no entry lost/duplicated, the flag survives, and a subsequent pass converges. (The invariant + convergence guarantee — *not* the binary "unchanged-or-fully-reorganized".)
- FR-005: the loop can see a topic's entries (assert `topic_read` returns them in a test).
- FR-007: flag cleared only on success; a flagged topic with zero pending episodes is still processed (the M1 fix — assert the driver runs it).
- FR-008: a clean, non-redundant topic is left substantially unchanged after a pass.
- Typecheck + lint pass.

**Checkpoint**: ✅ Consolidation is holistic and interruption-safe. *Do not proceed to Phase 5 until this is green.*

---

## Phase 5: Hybrid retrieval (T3 — search quality)

**Purpose**: Upgrade `memory_search` from substring match to fused dense + sparse + recency + importance with a relevance floor, provenance, per-entry embedding cache, and graceful degradation.

**Complexity**: **Justified** — new retrieval algorithm + a new cache file, but the design fully specifies the fusion (design §5.5), the cache (ADR-3), and the bounded cold burst (N2/R3). The weights/floor are initial, config-exposable defaults (R6).

**Tasks**:
- **T5.1** (`src/lib/agent/memory/embeddings.ts` — **new**): `getEmbedding(agentId, {id, text})` wrapping `embed()` + the per-agent cache at `/Memories/<agentId>/.embeddings.json` (key = content-hash id, per-entry `model`, lazy recompute on model change, prune of orphaned ids). (ADR-3, FR-013.)
- **T5.2** (`src/lib/agent/memory/paths.ts`): Add `agentEmbeddingsFile(agentId)`.
- **T5.3** (`src/lib/agent/memory/search.ts`): Replace the substring ranking with the fused score `0.4·dense + 0.3·sparse + 0.2·recency + 0.1·importance` (renormalized over the available signals when dense is off). BM25 over the agent's topic corpus. Recency `exp(-age_days/30)`, neutral 0.5 when missing. Importance neutral 0.5 default. (FR-012, FR-017.)
- **T5.4** (same file): Relevance floor on the *support* signal `max(dense, sparse) ≥ ~0.15` before ranking; below floor → dropped; none pass → empty result (FR-014, ADR-7). Add `state` to each `SearchResult` (FR-019). Provenance retained (FR-015).
- **T5.5** (same file): Graceful degradation — if the query `embed()` returns `null` (or dense disabled), drop the dense term and renormalize; **no error** (FR-016, SC-007). **Bounded cold burst (N2)**: on a cold cache, embed only the top-K (≈8–10) sparse candidates to seed dense, not the whole corpus (R3).
- **T5.6** (`src/lib/agent/memory/tool.ts` + `assistant/tools/server/memory.ts`): `memory_recall` labels superseded entries "not current" (FR-019); expose an optional `asOf` on `memory_recall` (FR-020).

**Files**: `embeddings.ts` (new), `paths.ts`, `search.ts`, `tool.ts`, `server/memory.ts`.

**Verification**:
- **SC-003**: On a fixed retrieval set — most-relevant-and-recent is top-1 in ≥90%; a paraphrased query (no shared keywords) retrieves the intended entry via dense; a keyword query retrieves keyword-only entries via sparse. (Pure-function assertions on the fusion + a mocked `embed()`; plus an e2e with a real/stubbed endpoint.)
- **SC-007**: With embeddings disabled (blank model / anthropic / 404), search returns ranked results (sparse+recency+importance) with **zero errors** across the fixed set. (FR-016.)
- FR-013: caching — a second identical search issues no corpus `embed()` for already-cached entries; changing an entry's text yields a cache miss (recompute-by-construction via the content-hash id).
- FR-014: a query matching nothing above the floor returns empty.
- FR-015: each result carries `source` provenance + `state`.
- Typecheck + lint pass.

**Checkpoint**: ✅ Search is hybrid, ranked, and degrades gracefully. *Do not proceed to Phase 6 until this is green.*

---

## Phase 6: Temporal validity (T4 — trust) + docs

**Purpose**: Add the entry lifecycle (active/superseded + superseding ref), contradiction detection as a background slow-loop op, default + as-of retrieval views, and update the docs (Constitution VI).

**Complexity**: **Justified** — largest data-model change (entry state + ref), but it's additive (a tag on the entry line, ADR-5) with a read-time legacy fallback, and contradiction detection is deliberately best-effort LLM judgment (R4, matching the spec's "background pass, not inline" assumption).

**Tasks**:
- **T6.1** (`src/lib/agent/memory/topics.ts`): `TopicEntry` gains `state?: "active" | "superseded"` + `supersededBy?: { id; timestamp }`. The parser extends `ENTRY_LINE` to optionally capture the `⟦superseded by=<entryId>@<ts>⟧` tag; absent → `active`. Keep the existing text-regex `superseded` heuristic as a **read-time fallback** for legacy files (ADR-5). **Pin the hash invariant (S4)**: `id = hashId(CLEAN text)` where the tag + marker are excluded from the hash. Add `supersedeTopicEntry(agentId, slug, entryIdOrText, supersededBy)`.
- **T6.2** (`src/lib/agent/memory/consolidate.ts`): Add `topic_supersede` op + contradiction-detection guidance in the slow-loop system prompt (the loop `topic_read`s both entries, marks the older one superseded pointing at the newer). (T4, background op.)
- **T6.3** (`src/lib/agent/memory/search.ts` + recall path): Default view returns active entries as current, labels superseded "not current" (FR-019). As-of: `entryStateAt(e, T)` = active iff `e.timestamp ≤ T` AND NOT(`superseded` AND `supersededBy.timestamp ≤ T`); exposed via the `memory_recall(asOf?)` param (FR-020).
- **T6.4** (docs, Constitution VI): Update `docs/dev/memory/memory.md` (soft budget, lifecycle, hybrid search, embedding endpoint, cache) and `docs/usage/memory/*` + the Memory-app doc (agent self-cures; embeddings config; active/superseded labeling).

**Files**: `topics.ts`, `consolidate.ts`, `search.ts`, recall path, `docs/dev/memory/memory.md`, `docs/usage/memory/*`.

**Verification**:
- **SC-004**: A known contradiction pair → after the slow-loop pass, the older entry is superseded (ref to the newer); default retrieval returns the active one as current in 100% of cases; the superseded one is never returned as current.
- FR-018: `supersedeTopicEntry` marks + references without deleting.
- FR-020: as-of returns the correct state at a historical T.
- FR-019: default retrieval labels superseded "not current."
- ADR-5/S4: a legacy file (no tag) parses as active via the fallback; hashing an entry with a tag uses the clean text (tag doesn't change the id).
- Docs reflect all five slices (Constitution VI).
- Typecheck + lint pass.

**Checkpoint**: ✅ Full feature complete. Run the full e2e suite via `buildstudio_run_tests`.

---

## Phase 7: Integration, e2e, and cross-artifact consistency

**Purpose**: End-to-end validation, the Playwright suite, and keeping spec/docs authoritative.

**Complexity**: None.

**Tasks**:
- **T7.1**: Author `e2e/028-memory-curation-retrieval.spec.ts` covering the settings surface (mockup-bound), the agent-tool round-trip (save-soft-budget, replace, remove), and a search scenario. Run via `buildstudio_run_tests` (writes `test-results.md` into the spec folder).
- **T7.2**: Run the full verification for each success criterion (SC-001…SC-007) end-to-end; confirm the reported bug is gone.
- **T7.3**: `/speckit.analyze` (cross-artifact consistency) + `/speckit.converge` (code-vs-spec drift); record any drift in `specs/discrepancies.md`.
- **T7.4**: Final quality gates: `npx tsc --noEmit` + `npm run lint` clean.

**Verification**: All SCs pass end-to-end; e2e suite green; analyze/converge clean.

**Checkpoint**: ✅ Feature complete, tested, and docs-in-sync — ready for preview/promote.

---

## Test Strategy

**Unit (pure functions)** — `resolveEmbeddingConfig` fallback matrix; fusion scoring + relevance floor (with a mocked `embed()`); topic serializer round-trip (flag + lifecycle tag + budget-excludes-marker); as-of `entryStateAt`. These are deterministic and need no server.

**E2E (Playwright, `e2e/028-memory-curation-retrieval.spec.ts`)** — Settings → AI Provider: the Embeddings subsection renders per `mockup.html`, the key is never shown, per-field fallback works, saving persists. Agent tools: save-to-full-topic succeeds (SC-001), replace/remove tidy a topic (SC-005), search ranks + degrades (SC-003/SC-007). Use a stubbed/real embedding endpoint for the dense-signal assertions.

**Interruption (SC-006)** — inject a mid-pass fault in a consolidation split; assert file validity + no loss/dup + surviving flag + convergence on re-run.

**Security (R8)** — assert the embedding key is absent from both the provider API view *and* the config-registry load (both redaction paths).

**Degradation (FR-016)** — run the fixed retrieval set with embeddings disabled; assert ranked results + zero errors.

---

## Complexity Tracking

| Complexity | Justification (Constitution: "justified or rejected") |
|---|---|
| Phase 4 — holistic consolidation | High-risk (interruption semantics). Scoped to existing atomic ops (no whole-file primitive), which is precisely what bounds it and satisfies SC-006 (ADR-2). Fully specified in design §5.4. |
| Phase 5 — hybrid retrieval + cache | New algorithm + sidecar cache, but the fusion, cache shape, and bounded cold burst are all specified (design §5.5, ADR-3, N2). Weights/floor are config-exposable defaults, not hidden complexity (R6). |
| Phase 6 — temporal validity | Largest data-model change, but additive (an entry-line tag, ADR-5) with a legacy fallback; contradiction detection is best-effort by explicit spec assumption (R4). |
| Gate VII — `package.json` | One-line explicit declaration of an already-present (`openai`) de-facto dependency; touches the lockfile. Flagged as the single boundary exception (ADR-6). |

---

## Source Accuracy Verification

Verified against the design's cited source paths (per the design's Integration-points + file plan): `provider.ts` (ProviderConfig/updateProviderConfig), `provider-meta.ts` (PROVIDERS), `llm.ts` (`openaiClient`/`normalizeApiBase`), `app/api/agent/provider/{route,test}/route.ts`, `config/registry.ts` (ai-provider namespace), `lib/agent/memory/{topics,search,consolidate,tool,paths,agent-memory}.ts`, `assistant/tools/server/memory.ts`, `capabilities-registry.ts`, `os/atomic-write.ts` (temp+rename), `components/apps/ProviderSettings.tsx`. `openai` confirmed imported by `llm.ts` but not declared in `package.json` (the Gate VII item). No stale-path assumptions carried into the task phases.
