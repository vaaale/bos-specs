# Quickstart: Memory Curation & Retrieval

**Branch**: `bos/028-memory-curation-retrieval` — how to verify the feature end-to-end after each phase.

> Run the full suite with `buildstudio_run_tests` (writes `test-results.md` into this folder). Below is the manual/traceable recipe per success criterion.

## Prereqs
- Branch `bos/028-memory-curation-retrieval` checked out.
- `npx tsc --noEmit` and `npm run lint` pass (quality gate per Constitution VII).
- An embedding-capable provider reachable (OpenAI-compatible), OR be prepared to test the degradation path (Anthropic / blank model).

## 1. Provider embedding endpoint (Phase 1)
- **Fallback matrix** (pure function `resolveEmbeddingConfig`):
  - LLM base `https://api.openai.com/v1`, key `sk-…`, embedding fields blank → resolved base `https://api.openai.com/v1`, resolved key `sk-…`, model `text-embedding-3-small`, `enabled: true`.
  - Embedding base set to a local URL, key blank → base = local URL, key = LLM key.
  - Provider = anthropic, fields blank → `enabled: false` (no `/embeddings` on the Anthropic base).
  - Embedding model blank (any provider) → `enabled: false`.
- **Key hygiene (FR-011, R8)**: `GET /api/agent/provider` returns `hasEmbeddingKey` (boolean) and **never** the key value; the config-registry load redacts it via `secret: true`.
- **Test connection** (`/api/agent/provider/test`): reports `embeddings.available` distinctly from the LLM probe (1-token `embed()`).

## 2. Settings UI (Phase 2) — open Settings → AI Provider
- The **Embeddings** subsection is present with the "new" badge, Base URL + API key + Model fields.
- Base URL blank shows "Leave blank to use the LLM provider's base URL above" and the `Uses: …` placeholder.
- API key is a password field with a has-key indicator (set / fallback) — the value is never displayed.
- Availability indicator shows `available` / `not supported` / `unknown`; switching to Anthropic shows the "no first-party embeddings" note.
- Save → fields persist; a blank Base URL resolves to the LLM base URL.
- Compare the screen to `mockup.html` (the binding contract).

## 3. Soft budget + self-edit (Phase 3) — the reported-bug regression
- **SC-001**: Fill a topic to just under its 4000-char budget, `memory_save` one more entry that pushes it over → **success** (no "Over budget"), topic flagged `consolidate`.
- **SC-005**: Ask the agent to "tidy this topic" → it uses `memory_replace` / `memory_remove` and completes **without** creating a `-2` shard.
- US2: replace by id, remove by id, unknown id → clear not-found error, no change.

## 4. Holistic consolidation (Phase 4)
- **SC-002**: Seed a topic with N redundant entries + flag; run the slow loop; result ≤ ⌈N/2⌉ entries, no two with the same meaning, digest refreshed, flag cleared.
- **Driver (M1)**: A flagged topic with **zero pending episodes** is still processed (assert the loop ran).
- **SC-006**: Inject a mid-pass fault during a split → every topic file valid, no entry lost/duplicated, flag survives, next pass converges.
- FR-008: a clean topic is left substantially unchanged.

## 5. Hybrid retrieval (Phase 5)
- **SC-003**: Fixed set — most-relevant-and-recent is top-1 in ≥90%; a paraphrase (no shared keywords) retrieves via dense; a keyword query retrieves keyword-only entries via sparse.
- **FR-013**: Second identical search issues no corpus `embed()` for cached entries; editing an entry's text → cache miss (recompute).
- **FR-014**: A query matching nothing above the floor → empty result.
- **FR-015**: Each result carries `source` provenance + `state`.
- **SC-007 / FR-016**: With embeddings disabled (blank model / anthropic / 404) → ranked results (sparse+recency+importance), **zero errors**.

## 6. Temporal validity (Phase 6)
- **SC-004**: Store "user works at X" then "user works at Y"; slow-loop pass marks X superseded (ref → Y). Default retrieval returns Y as current in 100% of cases; X is never returned as current.
- FR-018: the supersede op marks + references without deleting.
- FR-020: `memory_recall(asOf: <past T>)` returns the historically-correct state.
- ADR-5/S4: a legacy file (no tag) parses as active via the fallback; the id hash uses clean text (tag excluded).

## 7. Docs (Constitution VI)
- `docs/dev/memory/memory.md` + `docs/usage/memory/*` updated to cover all five slices.

## Promote
Once all SCs pass and tsc/lint/e2e are clean: this is a **preview candidate** — promote via the top-bar **Active ▾** menu. (The `MemoryActions.tsx` stale-duplicate retirement is tracked as a separate, out-of-scope cleanup.)
