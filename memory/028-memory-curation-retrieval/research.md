# Research: Memory Curation & Retrieval

**Branch**: `bos/028-memory-curation-retrieval`
**Status**: Resolved (all open questions closed during specify/clarify/design)

## Resolved decisions

### 1. T3 retrieval backend — dense + sparse hybrid (user decision)
- **Decision**: `memory_search` fuses **dense (embedding) + sparse (BM25) + recency + importance**. Not dependency-free-only.
- **Evidence**: BOS core has **no existing embedding infrastructure** — `discovery-score.ts` documents "substring/score, no embeddings." The `openai` npm SDK is already imported by `src/lib/agent/llm.ts` and exposes `client.embeddings.create()`, so reusing it avoids a new runtime dependency.
- **Implication**: A provider **embedding endpoint** (base URL, API key, model) is added to the AI provider config, with per-field fallback to the LLM provider.

### 2. Per-field embedding fallback
- **Decision**: Empty embedding base URL → use LLM base URL; empty embedding API key → use LLM API key; model set independently with a per-provider default. Per-field `""` clears, `undefined` leaves unchanged.
- **Evidence**: `provider.ts` `ProviderConfig` already stores `baseUrl`/`apiKey`/`model` flat; the nested `embeddings` block mirrors this with a `resolveEmbeddingConfig()` helper. `normalizeApiBase` already trims the `/embeddings` suffix.

### 3. Per-provider embedding availability
- **Decision**:
  - OpenAI / Codex / Responses → default `text-embedding-3-small`, `available`.
  - Local (OpenAI-compatible) → no universal default (user picks via the existing model-list refresh), `unknown` until tested.
  - **Anthropic → no first-party embeddings API** → `unsupported`; dense retrieval off unless the user overrides the embedding base URL to an OpenAI-compatible server (the per-field override makes a hybrid LLM+embeddings config possible).
- **Evidence**: Anthropic's public API has no `/embeddings` endpoint; `normalizeApiBase`/`provider-meta.ts` confirm the OpenAI-compatible `/embeddings` convention.

### 4. Blank embedding model = disabled
- **Decision**: A blank embedding model is treated as embeddings **disabled** → retrieval degrades per FR-016. This resolves the apparent "required" vs "degrade" tension: required *to enable*, not required *to exist*.

### 5. Interruption semantics (SC-006) — invariant + convergence
- **Decision**: Because the ACE anti-collapse rule forbids a whole-file topic rewrite, a mid-pass failure may leave a **consistent intermediate state** (entries split across source + sibling). The guarantee is *no entry lost or duplicated* + *flag survives* ⇒ *monotonic convergence on the next pass*. SC-006 was reworded to match (user-approved).
- **Evidence**: design ADR-2; `topics.ts` ACE rule (a full rewrite could collapse a 600-line file to the model's compressed summary).

### 6. `package.json` (Constitution VII)
- **Decision**: Declare `openai` explicitly — it is already a de-facto runtime dependency (imported by `llm.ts`, resolving today as a hoisted transitive) and `embed()` will depend on it directly. One-line addition, touches the lockfile. **Flagged as the plan's single boundary exception.**
- **Evidence**: `llm.ts` line ~1 imports `openai`; `package.json` does not list it.

## Open (deferred)
- Exact fusion weights / floor are initial, config-exposable defaults (`0.4/0.3/0.2/0.1`, floor ~0.15) — to be tuned post-implementation (design R6).
- `MemoryActions.tsx` stale-duplicate retirement — **separate cleanup, out of scope** (noted in design R1).
