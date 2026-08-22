# Feature Specification: Memory Curation & Retrieval

**Feature Branch**: `[028-memory-curation-retrieval]`

**Created**: 2026-08-22

**Status**: Draft

**App Target**: `bos-core`

**Input**: User description: "The agent memory system fails when a topic exceeds its budget (the agent is forced to shard into a `-2` topic). Make memory self-curing and smarter: (T1) the consolidation loop should be able to read a topic's entries and merge/dedup/split/reorganize them; (T2) the per-topic budget should be a soft signal, not a hard reject; (T3) memory search should rank by relevance + recency (+ importance), not substring match; (T4) entries should carry temporal validity so contradictions don't silently coexist; (T6) the agent should be able to replace/remove its own entries to self-manage topics."

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

### User Story 4 - Memory search returns the most useful entries first (Priority: P2)

When the agent searches memory, results should be ranked by how relevant, recent, and important the matching entries are — not merely whether a substring/word appears. Today `memory_search` is a case-insensitive substring/word match.

**Why this priority**: Retrieval quality is the highest-leverage improvement for memory usefulness, but it does not unblock the write path, so it lands after the self-curing core.

**Independent Test**: Seed a topic with (a) a recent, on-point entry, (b) an older off-point entry containing the same keyword, and (c) an irrelevant entry; search for the keyword and confirm the on-point recent entry ranks first and the irrelevant entry does not appear.

**Acceptance Scenarios**:

1. **Given** multiple entries matching a query, **When** the agent searches, **Then** results are ordered by a combined score of relevance, recency, and importance (not by insertion order).
2. **Given** a recent entry and an older entry with equal textual relevance, **When** the agent searches, **Then** the more recent entry ranks higher.
3. **Given** a query that matches no entry, **When** the agent searches, **Then** an empty result is returned (no fuzzy false positives below a relevance floor).
4. **Given** a search result, **When** returned, **Then** each result includes provenance (which topic + in-file anchor) so the agent can read the full entry on demand.

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
- What happens when recency/importance data is missing for a legacy entry? (Must fall back gracefully to relevance-only scoring.)

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
- **FR-009**: `memory_search` MUST rank results by a combined score of textual relevance, recency, and importance, rather than by substring match alone.
- **FR-010**: `memory_search` MUST apply a relevance floor so that non-matching queries return no results (no low-confidence false positives).
- **FR-011**: `memory_search` results MUST include provenance (topic slug and in-file anchor) for each returned entry.
- **FR-012**: The system MUST support marking an entry as superseded (with a reference to the superseding entry) in response to a detected contradiction.
- **FR-013**: Default topic retrieval MUST return active entries as current and clearly label superseded entries as not-current.
- **FR-014**: The system MUST support an as-of view of a topic's entry state for a given time.
- **FR-015**: The system MUST degrade gracefully when an entry lacks recency/importance metadata (fall back to relevance-only scoring).

### Key Entities

- **Topic entry**: A single memory within a topic; has text, a stable id, a timestamp, and (new) a lifecycle state (active / superseded) plus an optional reference to the superseding entry.
- **Topic**: A slug-keyed collection of entries with a one-line digest; (new) carries a consolidation flag.
- **Consolidation flag**: A per-topic marker that the background pass should reorganize this topic; set on soft-budget overflow or manual request, cleared after a successful pass.
- **Search result**: A ranked entry reference with a combined relevance/recency/importance score and provenance.

## Success Criteria

### Measurable Outcomes

- **SC-001**: Saving a memory to a topic at or over its budget succeeds 100% of the time (zero hard rejections); previously this returned an "Over budget" error.
- **SC-002**: After a consolidation pass over a seeded topic with N redundant entries, the resulting topic contains ≤ ⌈N/2⌉ entries with no two entries expressing the same meaning (verified by a deterministic test fixture).
- **SC-003**: In a fixed retrieval test set, the most-relevant-and-recent entry is returned first in ≥ 90% of queries (up from order-by-substring, which has no recency signal).
- **SC-004**: Given a known contradiction pair, the active entry is returned as current in 100% of default retrievals, and the superseded entry is never returned as current.
- **SC-005**: The agent is able to complete a "tidy this topic" request using only the exposed replace/remove tools, with no need to create a `-2` shard (verified by a scripted agent trace).
- **SC-006**: Consolidation of a topic does not corrupt it on interruption: after a forced mid-pass failure, the topic is either fully unchanged or fully reorganized (no partial state), and the flag survives for re-run.

## Assumptions

- All changes are to the existing per-agent memory subsystem under `src/lib/agent/memory/` and the agent's memory tool handlers; no new top-level subsystem is introduced.
- The two-loop model (fast loop writes episodes; slow loop consolidates) is retained. This feature makes the slow loop holistic and adds a soft-budget trigger; it does not replace the loops or add a third loop.
- The per-topic budget remains a per-topic-file character budget, as today; the change is to its *behavior* on overflow (accept + flag, not reject), not to remove the budget.
- Recency and importance are derived signals. Recency comes from the existing entry timestamp. Importance is best-effort (e.g., derived at save or by the consolidation pass); when absent, scoring falls back to relevance + recency (FR-015).
- **Contradiction detection (T4) is a background consolidation operation, not an inline write-time check.** Until the pass runs, both entries may be visible, with the most recent preferred. (See Edge Cases.)
- **Hybrid retrieval relevance default (T3)**: the v1 relevance signal is a *local, dependency-free* lexical/structural score (e.g., BM25/TF-IDF over entry text) combined with recency and importance. A *dense vector* similarity backend is **[NEEDS CLARIFICATION: should v1 bring in an embedding provider for dense similarity, or stay dependency-free with lexical+recency+importance and treat dense embeddings as a later enhancement?]** — BOS core currently has no embedding infrastructure (`discovery-score.ts` deliberately uses "substring/score, no embeddings").
- Superseded entries are retained (not deleted) so as-of queries (FR-014) remain possible; hard deletion is out of scope.
- No user-facing UI is added by this feature; it is an agent-facing capability. (The Memory app, if present, would surface the new state later but is not part of this spec.)
