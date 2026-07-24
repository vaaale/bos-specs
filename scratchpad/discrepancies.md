# Scratchpad — Discrepancies & Deferred Work

Snapshot at converge (2026-07-05). All P1 requirements implemented; items below are
deferred or minor and do NOT block promotion.

## Deferred (optional, per spec)

- **FR-005 — `read_notes` search.** Marked optional in the spec ("can be added later
  if needed"). Current implementation supports list-all and read-by-title only. Add
  if/when a real use case appears.
- **FR-005 — metadata-only flag.** Listing already returns metadata by default; a
  separate `metadata_only` flag on read-by-title is not implemented. Revisit only
  if notes grow large enough that returning full content on list becomes costly.
- **FR-010 — tag support.** `NoteMetadata` reserves a spot for tags but no tool
  parameter is exposed to set/query them. Wire `tags` into `write_note` / `edit_note`
  and add a filter to `read_notes` when tagging is actually requested by users.

## Spec-internal contradiction (documentation only)

- **US7 scenario 2 vs. Clarifications.** US7 scenario 2 alludes to a persistent
  storage layer; the Clarifications section supersedes this by mandating
  history-derived state (FR-003). Code follows the clarification (correct behavior).
  Fix by rewording US7 scenario 2 on the next spec pass — no code change needed.

## Code-quality notes (non-blocking)

- **Duplicated `newNoteId()`** in `src/lib/agent/scratchpad/handlers.ts` and
  `src/lib/agent/scratchpad/replay.ts`. Extract to a shared helper (e.g. `ids.ts`)
  in a future cleanup pass.
- **Timestamp determinism.** Replay uses wall-clock timestamps at reconstruction
  time rather than the original tool-call timestamps. Fine for current UX; revisit
  if timestamps ever surface in the UI or drive ordering beyond insertion order.

## Test coverage

- P1 scenarios covered by `handlers.test.ts` and `replay.test.ts`.
- No tests exist for the deferred P2 features (they're not implemented).
