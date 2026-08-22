# Feature Specification: Memory Curation & Retrieval

**Feature Branch**: `[028-memory-curation-retrieval]`

**Created**: 2026-08-22

**Status**: Draft

**App Target**: `bos-core`

**Input**: User description: "The agent memory system fails when a topic exceeds its budget (the agent is forced to shard into a `-2` topic). Make memory self-curing and smarter: (T1) the consolidation loop should be able to read a topic's entries and merge/dedup/split/reorganize them; (T2) the per-topic budget should be a soft signal, not a hard reject; (T3) memory search should rank by relevance + recency (+ importance), not substring match — using BOTH embeddings (dense) and BM25 (sparse); (T4) entries should carry temporal validity so contradictions don't silently coexist; (T6) the agent should be able to replace/remove its own entries to self-manage topics. To support (T3), the AI Provider must also define an embedding endpoint (base url, api key, model name); if base url and api key are left empty, use the same as the LLM provider."

## User Scenarios & Testing

### User Story 1 - A full topic no longer blocks a new memory (Priority: P1)

The agent wants to save a new memory to a topic that is already at or near its size budget. Today the save is rejected and the agent is forced to create a parallel `-2` shard. Instead, the save should succeed, and the topic should be marked so the background consolidation pass can reorganize it.

**Why this priority**: This is the direct fix for the reported failure (the "Over budget … create a shard" error). It unblocks the core agent workflow immediately.

**Independent Test**: Fill a topic to just under its budget, then save one more entry that pushes it over. The save returns success (not an error), and the topic is flagged as needing consolidation.

**Acceptance Scenarios**:

1. **Given** a topic whose serialized size is within budget, **When** the agent saves an entry, **Then** the entry is stored and no flag is set.
2. **Given** a topic at or over its budget, **When** the agent saves another entry, **Then** the entry is still stored, the save succeeds, and the topic is flagged for consolidation (no rejection, no forced shard).
3. **Given** a topic flagged for consolidation, **When** a later consolidation pass runs, **Then** the flag is cleared after the topic has been reorganized.

---

### User Story 2 - The agent can edit its own memories (Priority: P1)

The agent should be able to update an existing memory in place and remove an entry that is wrong or outdated — not only append. Today the agent's only memory-write tool is append-only, so it cannot correct itself or tidy a topic.

**Why this priority**: Directly complements Story 1 — once a topic can grow past budget, the agent needs a way to shrink/repair it rather than fork it.

**Independent Test**: Create a topic with two entries; use the new update tool to change one entry's text and the new remove tool to delete the other. Confirm the topic reflects both changes and the index is consistent.

**Acceptance Scenarios**:

1. **Given** a topic with an existing entry, **When** the agent issues a replace with the entry's id and new text, **Then** the entry's text is updated (id/timestamp updated as appropriate) and the result confirms the change.
2. **Given** a topic with an entry, **When** the agent issues a remove by entry id, **Then** the entry is gone and remaining entries are intact.
3. **Given** an unknown entry id, **When** the agent issues replace/remove, **Then** a clear not-found error is returned and no change is made.
4. **Given** an entry whose new text duplicates an existing entry in the same topic, **When** the agent issues a replace, **Then** the duplicate is handled (no duplicate text remains in the topic).

---

### User Story 3 - The consolidation pass makes topics coherent (Priority: P1)

The background consolidation pass (slow loop) should be able to *see* the entries of a topic it is consolidating — not just the topic's slug and one-line digest — so it can merge near-duplicates, drop stale entries, split an over-fat topic into more focused sub-topics, and refresh the digest. Today it can only add entries it learns about from episodes; it has no view of what is already in a topic.

**Why this priority**: This is what makes the soft budget (Story 1) safe — it is the mechanism that actually reorganizes a flagged topic instead of letting it grow unbounded.

**Independent Test**: Seed a topic with several redundant/overlapping entries and flag it; run the consolidation pass; confirm the topic now has fewer, merged, coherent entries and an updated digest, and the flag is cleared.

**Acceptance Scenarios**:

1. **Given** a topic with two entries that say substantially the same thing, **When** the consolidation pass processes the topic, **Then** they are merged into a single coherent entry (no duplicate meaning remains).
2. **Given** a topic containing an entry that is clearly stale or superseded, **When** the consolidation pass runs, **Then** the stale entry is removed or marked accordingly and the rest are preserved.
3. **Given** an over-budget topic flagged for consolidation, **When** the pass runs, **Then** the topic ends within budget (entries merged/removed or split into a focused sibling topic) and the digest reflects the content.
4. **Given** a topic with well-organized, non-redundant entries, **When** the pass runs, **Then** it is left substantially unchanged (no gratuitous rewrites).

---

### User Story 4 - Memory search is semantic + lexical, and the provider can serve embeddings (Priority: P2)

When the agent searches memory, results should be ranked by how relevant, recent, and important the matching entries are — using both **dense (embedding)** similarity and **sparse (keyword/BM25)** matching — not merely whether a substring appears. To get dense similarity, the **AI Provider configuration grows an embedding endpoint**: a base URL, an API key, and a model name. When the embedding base URL and API key are left empty, they fall back to the LLM provider's base URL and API key.

**Why this priority**: Retrieval quality is the highest-leverage improvement for memory usefulness, but it does not unblock the write path, so it lands after the self-curing core.

**Independent Test**: (a) Configure the embedding endpoint; (b) seed a topic with (i) a recent paraphrased match (no shared keywords), (ii) an older off-point entry sharing the keyword, and (iii) an irrelevant entry; search and confirm the paraphrased recent entry ranks first (dense recall) while a pure-keyword query still finds keyword-only entries (sparse recall), and the irrelevant entry does not appear.

**Acceptance Scenarios**:

1. **Given** a provider with an embedding endpoint configured (or falling back to the LLM base URL/key), **When** an embedding is requested, **Then** a vector is obtained from that endpoint and cached per entry.
2. **Given** an embedding base URL and API key both left empty, **When** the embedding client resolves its endpoint, **Then** it uses the LLM provider's base URL and API key.
3. **Given** multiple entries matching a query, **When** the agent searches, **Then** results are ordered by a fused score of dense similarity, sparse (BM25) similarity, recency, and importance.
4. **Given** a query that is a paraphrase of an entry (no shared keywords), **When** the agent searches, **Then** the entry is still retrieved (dense signal).
5. **Given** a query with exact keywords and no close paraphrase, **When** the agent searches, **Then** keyword-only entries are still retrieved (sparse signal).
6. **Given** a recent entry and an older entry with equal similarity, **When** the agent searches, **Then** the more recent entry ranks higher.
7. **Given** a query that matches no entry above a relevance floor, **When** the agent searches, **Then** an empty result is returned (no low-confidence false positives).
8. **Given** a search result, **When** returned, **Then** each result includes provenance (topic slug + in-file anchor) so the agent can read the full entry on demand.
9. **Given** a configured provider whose endpoint does not support embeddings, **When** the agent searches, **Then** retrieval degrades gracefully to sparse + recency + importance (no error, no crash).
10. **Given** the provider config view is read by the UI, **When** returned, **Then** the embedding API key is never exposed (consistent with the LLM key's has-key handling).

---

### User Story 5 - Conflicting memories don't silently coexist (Priority: P3)

When a new entry contradicts an existing one (e.g., the user's job changed), the old entry should be marked as superseded rather than left alongside the new one, and the system should be able to answer "what is true now" vs "what was true at time T."

**Why this priority**: Highest trust value for a personal assistant but the largest data-model change, so it is the last slice; the first four ship value without it.

**Independent Test**: Store "user works at X"; store "user works at Y" (a contradiction the consolidation pass detects); confirm the X entry is marked superseded (pointing to Y), a default retrieval returns Y, and an as-of query returns the historically-correct entry.

**Acceptance Scenarios**:

1. **Given** an active entry, **When** a contradicting entry is recorded and the consolidation pass marks the conflict, **Then** the older entry is marked superseded with a reference to the newer one (not deleted).
2. **Given** an active entry and a superseded entry on the same subject, **When** the agent recalls the topic, **Then** the active entry is returned as current and the superseded one is clearly labeled as not-current.
3. **Given** a topic with entries spanning time, **When** the agent requests an as-of view, **Then** the entry state as of that time is returned.
4. **Given** an entry with no recorded contradiction, **When** the topic is read, **Then** it is treated as active (default).

---

### Edge Cases

- What happens when a consolidation pass is interrupted mid-reorganization of a topic? (The topic must not be left corrupted; the flag must survive so a re-run can finish.)
- How is the budget measured once entries can be split into a sibling topic — is the budget per-topic-file as today?
- What happens when the agent's replace produces text that is an exact duplicate of another entry?
- What happens when two entries contradict each other but the consolidation pass is not yet scheduled? (Contradiction marking is a background pass, not inline — confirm that is acceptable and that retrieval still surfaces both until then, with the most recent preferred.)
- What happens when recency/importance data is missing for a legacy entry? (Must fall back gracefully to similarity + recency scoring.)
- What happens when only the embedding base URL (and not the key) is left empty? (Per-field fallback: each empty field uses the LLM provider's value; a set field is used as-is.)
- What happens when the configured provider's endpoint does not support embeddings? (Retrieval degrades to sparse + recency + importance; no error, and a diagnostic is available.)
- What happens for a pure-Anthropic provider (no first-party embeddings API)? (The embedding endpoint falls back to the Anthropic base URL, which has no `/embeddings`; dense retrieval is disabled and the availability indicator reflects "not supported". The user can re-enable it by pointing the embedding base URL at an OpenAI-compatible server.)
- What happens when the embedding model is left blank? (Treated as embeddings disabled; retrieval degrades per FR-016; the availability indicator shows disabled/not-supported rather than an error.)
- What happens when an entry's text changes after its embedding is cached? (The embedding is recomputed so retrieval stays accurate.)

## Requirements

### Functional Requirements

- **FR-001**: The system MUST accept a `memory_save` to a topic even when the resulting serialized topic size exceeds its budget, instead of rejecting the write; the save MUST succeed and mark the topic for consolidation.
- **FR-002**: The system MUST expose an update (replace) operation for an existing topic entry, keyed by entry id, to the agent's memory toolset.
- **FR-003**: The system MUST expose a remove operation for a topic entry, keyed by entry id, to the agent's memory toolset.
- **FR-004**: The system MUST preserve exact-duplicate detection on save (an identical entry is a no-op, not a duplicate append).
- **FR-005**: The consolidation pass MUST be able to read the full entry list of a topic it is processing, not only the topic slug and digest.
- **FR-006**: The consolidation pass MUST be able to merge near-duplicate entries, remove stale/superseded entries, and split an over-budget topic into more focused sibling topics, and MUST refresh the topic digest afterward.
- **FR-007**: The consolidation pass MUST clear a topic's consolidation flag only after the topic has been successfully reorganized.
- **FR-008**: The consolidation pass MUST leave a well-organized, non-redundant topic substantially unchanged (no gratuitous rewrites).
- **FR-009**: The AI provider configuration MUST support an embedding endpoint consisting of a base URL, an API key, and a model name.
- **FR-010**: When the embedding base URL and/or API key are left empty, they MUST fall back to the LLM provider's base URL / API key (per-field); the embedding model name MUST be independently configurable, with a sensible per-provider default. The embedding model is required *in order to enable* dense retrieval; if it is left blank, embeddings are treated as **disabled** and retrieval degrades per FR-016 (so a blank model and the graceful-degradation path are the same thing, not a contradiction).
- **FR-011**: The embedding API key MUST NOT be exposed in the provider config view (only a has-key indicator, consistent with the existing LLM key handling).
- **FR-012**: `memory_search` MUST rank results by a fused score of dense (embedding) similarity, sparse (BM25/keyword) similarity, recency, and importance.
- **FR-013**: The system MUST compute and cache an embedding per topic entry, recomputing only when the entry text changes, so search does not recompute embeddings on every query.
- **FR-014**: `memory_search` MUST apply a relevance floor so that non-matching queries return no results (no low-confidence false positives).
- **FR-015**: `memory_search` results MUST include provenance (topic slug and in-file anchor) for each returned entry.
- **FR-016**: When the configured provider's endpoint does not support embeddings, `memory_search` MUST degrade gracefully to sparse + recency + importance scoring (no error).
- **FR-017**: The system MUST degrade gracefully when an entry lacks recency/importance metadata (fall back to similarity + recency scoring).
- **FR-018**: The system MUST support marking an entry as superseded (with a reference to the superseding entry) in response to a detected contradiction.
- **FR-019**: Default topic retrieval MUST return active entries as current and clearly label superseded entries as not-current.
- **FR-020**: The system MUST support an as-of view of a topic's entry state for a given time.

### Key Entities

- **Topic entry**: A single memory within a topic; has text, a stable id, a timestamp, a lifecycle state (active / superseded) plus an optional reference to the superseding entry, and (new) a cached embedding vector.
- **Topic**: A slug-keyed collection of entries with a one-line digest; (new) carries a consolidation flag.
- **Consolidation flag**: A per-topic marker that the background pass should reorganize this topic; set on soft-budget overflow or manual request, cleared after a successful pass.
- **Embedding endpoint**: An AI provider setting (base URL, API key, model name) used to obtain dense vectors for entries; falls back to the LLM provider's base URL / API key when empty.
- **Search result**: A ranked entry reference with a fused dense/sparse/recency/importance score and provenance.

## Success Criteria

### Measurable Outcomes

- **SC-001**: Saving a memory to a topic at or over its budget succeeds 100% of the time (zero hard rejections); previously this returned an "Over budget" error.
- **SC-002**: After a consolidation pass over a seeded topic with N redundant entries, the resulting topic contains ≤ ⌈N/2⌉ entries with no two entries expressing the same meaning (verified by a deterministic test fixture).
- **SC-003**: In a fixed retrieval test set, the most-relevant-and-recent entry is returned first in ≥ 90% of queries; a paraphrased query (no shared keywords) still retrieves the intended entry via the dense signal, and a keyword query retrieves keyword-only entries via the sparse signal (up from substring match, which has neither a semantic nor a recency signal).
- **SC-004**: Given a known contradiction pair, the active entry is returned as current in 100% of default retrievals, and the superseded entry is never returned as current.
- **SC-005**: The agent is able to complete a "tidy this topic" request using only the exposed replace/remove tools, with no need to create a `-2` shard (verified by a scripted agent trace).
- **SC-006**: Consolidation of a topic does not corrupt it on interruption: after a forced mid-pass failure, the topic is either fully unchanged or fully reorganized (no partial state), and the flag survives for re-run.
- **SC-007**: When the configured provider does not support embeddings, `memory_search` returns ranked results (sparse + recency + importance) with zero errors across the fixed retrieval test set.

## Assumptions

- All core changes are to the existing per-agent memory subsystem (`src/lib/agent/memory/`), the agent's memory tool handlers, **and** the AI provider configuration layer (provider config + its Settings surface) to add the embedding endpoint. No new top-level subsystem is introduced.
- The two-loop model (fast loop writes episodes; slow loop consolidates) is retained. This feature makes the slow loop holistic and adds a soft-budget trigger; it does not replace the loops or add a third loop.
- The per-topic budget remains a per-topic-file character budget, as today; the change is to its *behavior* on overflow (accept + flag, not reject), not to remove the budget.
- Recency and importance are derived signals. Recency comes from the existing entry timestamp. Importance is best-effort (e.g., derived at save or by the consolidation pass); when absent, scoring falls back to similarity + recency (FR-017).
- **Contradiction detection (T4) is a background consolidation operation, not an inline write-time check.** Until the pass runs, both entries may be visible, with the most recent preferred. (See Edge Cases.)
- **Hybrid retrieval (T3) uses BOTH dense (embedding) and sparse (BM25/keyword) similarity**, fused with recency and importance (FR-012). Dense similarity requires the new provider embedding endpoint (FR-009).
- **Embedding endpoint fallback (per-field)**: when the embedding base URL is empty it uses the LLM provider's base URL; when the embedding API key is empty it uses the LLM provider's API key. The embedding model name is set independently, with a sensible default. (FR-010)
- **Graceful degradation**: if the configured provider's endpoint does not support embeddings, retrieval degrades to sparse + recency + importance without erroring (FR-016).
- **Embeddings are cached** per entry and recomputed only when the entry text changes (FR-013).
- The embedding endpoint is configured in the **existing AI Provider settings** (a small addition to that surface), not a new app.
- **Per-provider embedding defaults**: OpenAI / OpenAI Codex / OpenAI Responses → `text-embedding-3-small`. Local (OpenAI-compatible) → no universal default; the user selects via the form's existing model-list refresh (a placeholder such as `nomic-embed-text` is only a UI hint, not a stored default). Anthropic → **no first-party embeddings API**, so a pure-Anthropic provider has no embeddings endpoint to fall back to; dense retrieval is unavailable for it unless the user points the embedding base URL at an OpenAI-compatible server (the per-field override makes it possible to keep the LLM on Anthropic while running embeddings on a separate/local server).
- **Embedding availability indicator**: the settings UI indicates embedding status (available / not supported / unknown) via an explicit **Test connection** action plus provider inference (e.g. Anthropic → not supported); it does NOT auto-probe the endpoint on every save.
- Superseded entries are retained (not deleted) so as-of queries (FR-020) remain possible; hard deletion is out of scope.
