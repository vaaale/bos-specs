# Tasks: Conversation Archive

**Input**: Design documents from `/Specs/user-specs/core-platform/038-conversation-archive/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), design.md (mechanism/ADRs), mockup.html (UI contract)

**Note**: All tasks are executed by the Developer during `implement`. Paths are repo-relative. `[P]` = parallelizable (different file, no dependency).

## Format: `[ID] [P?] [Story] Description`

## Phase 1: Foundational (Shared Infrastructure)

**Purpose**: The data-model flag + store accessor + persistence endpoint. Everything downstream depends on these. No user story can be demonstrated before this phase.

- [ ] T001 [US1] Add `archived?: boolean` to the `Conversation` and `ConversationFile` interfaces and parse it defensively in `readConversationFile` (missing → `false`) in `src/lib/agent/conversations.ts`
- [ ] T002 [US1] Carry `archived` into each conversation built in `loadFromVfs`, include it in `newConversation`'s file write, and **defensively** carry it over from `existing` in the client-side `saveConversationMessages` so a file rebuild can never silently drop the flag in `src/lib/agent/conversations.ts`
- [ ] T003 [US1] Add `setConversationArchived(id, archived)` to `src/lib/agent/conversations.ts`: optimistic `setState` map update (early-return if value unchanged, mirroring `setConversationAgent`) then `fetch` the archive endpoint
- [ ] T004 [US1] Create `src/app/api/assistant/conversations/[conversationId]/archive/route.ts`: `PATCH { archived: boolean }` → `patchConversationMeta(conversationId, { archived })` (queue-serialized), return resulting metadata; `400` on bad body, `404` if the conversation file is absent (mirror the `feature-branches` route shape)
- [ ] T005 [US1] **Confirm dead-code status of client `saveConversationMessages`** (design.md R1, reviewer S2): verify no live caller imports the *client* `saveConversationMessages` from `src/lib/agent/conversations.ts`; the defensive carry in T002 makes this moot either way — record the confirmation in the commit message

**Checkpoint**: Foundation ready — a conversation can be flagged archived, the flag survives read/rebuild, and the state is optimistically updated in the shared store.

---

## Phase 2: User Story 1 - Archive a conversation to declutter the list (Priority: P1) 🎯 MVP

**Goal**: Archiving a conversation removes it from the default list and makes it read-only, while preserving all content and keeping it reopenable.

**Independent Test**: Open the Assistant app, archive a conversation from its row menu, and verify it no longer appears in the default list, that its content is fully preserved, that it opens read-only, and that a send attempt is blocked.

### Tests for User Story 1 ⚠️

> Write first, confirm they FAIL on the unfixed base.

- [ ] T006 [P] [US1] E2E: archive a conversation, assert it is absent from the default list and present in the Archived section, content intact, and reopens read-only — in `e2e/038-conversation-archive.spec.ts`
- [ ] T007 [P] [US1] E2E (server gate): with a conversation archived, a `POST /api/assistant/runs` for it returns `409` and no run starts — in `e2e/038-conversation-archive.spec.ts`

### Implementation for User Story 1

- [ ] T008 [US1] In `src/app/api/assistant/runs/route.ts`: after the existing `conversationId`/`agentId`/`message` validation and before `startAssistantRun`, read `getConversationMeta(conversationId)` (add the import from `@/lib/assistant/conversation-store` — reviewer S1) and `return 409` if `meta?.archived === true` (design.md ADR-2)
- [ ] T009 [US1] In `src/components/agent/v2/ChatInputV2.tsx`: read the active conversation's `archived` flag; when true, replace the composer with a locked placeholder + amber "archived and read-only … Unarchive to continue" banner whose button calls `setConversationArchived(id, false)` (UX only — server gate is authoritative) (design.md ADR-2/ADR-6, mockup)
- [ ] T010 [US1] In `src/components/apps/assistant/ConversationPanel.tsx` (`ConvRow`): replace the inline Pencil/Delete hover glyphs with a single kebab (`more-vertical`) row menu — default rows show Rename / **Archive** / Delete, archived rows show Rename / **Unarchive** / Delete — with Archive/Unarchive calling `setConversationArchived(id, !archived)` (design.md ADR-5)

**Checkpoint**: US1 fully functional — archiving declutters the default list and enforces read-only, content preserved.

---

## Phase 3: User Story 2 - Review and restore archived conversations (Priority: P2)

**Goal**: The user can browse archived conversations and unarchive them back into the default list, with content fully restored.

**Independent Test**: Archive a conversation, open the Archived section, unarchive it, and verify it returns to the default list with content identical to before.

### Tests for User Story 2 ⚠️

- [ ] T011 [P] [US2] E2E: archive → open Archived section → unarchive → assert it reappears in the default list and disappears from the Archived section, content unchanged — in `e2e/038-conversation-archive.spec.ts`

### Implementation for User Story 2

- [ ] T012 [US2] In `src/components/apps/assistant/ConversationPanel.tsx`: split the single-agent view's conversation list into a default block + an always-present **"Archived" sub-section** (divider-headed, count badge, empty-state "Nothing archived yet.") at the bottom of that agent's rows (design.md ADR-4/ADR-5, mockup)
- [ ] T013 [US2] In `src/components/apps/assistant/ConversationPanel.tsx`: apply the same default-vs-archived split **per agent** in the all-agents view — each agent bucket appends its own "Archived" sub-section beneath that agent's default rows (design.md R3, **resolved: per-agent buckets, not pooled**)
- [ ] T014 [US2] Ensure unarchiving from the Archived section (row menu, T010) returns the conversation to the top of its agent's default block and clears it from the Archived sub-section (relies on the shared store re-render)

**Checkpoint**: US1 + US2 functional — the user can hide and restore conversations, all within the same agent-scoped layout.

---

## Phase 4: User Story 3 - Archive from anywhere a conversation is surfaced (Priority: P3)

**Goal**: The Build Studio surface honors the same shared state and exposes an archive affordance, so behavior is consistent across apps with no per-app duplication.

**Independent Test**: Archive a conversation in the Assistant app and confirm it is hidden in Build Studio's conversation selector (and vice versa), both reflecting the change with no manual refresh.

### Tests for User Story 3 ⚠️

- [ ] T015 [P] [US3] E2E: archive a conversation in one surface, assert it is hidden from the default list in the other surface (cross-app, no refresh) — in `e2e/038-conversation-archive.spec.ts`

### Implementation for User Story 3

- [ ] T016 [US3] In `src/components/apps/assistant/AgentSelector.tsx` (`ConversationSelector`): split the `<select>` into two `<optgroup>`s — default and **"Archived"** — fed from the shared store (the only form of FR-008 a native `<select>` supports) (design.md ADR-5)
- [ ] T017 [US3] In `src/components/apps/assistant/AgentSelector.tsx` (`ConversationSelector`): add an Archive/Unarchive **button beside the dropdown** acting on the *active* conversation (same arm-then-confirm pattern as the existing delete button), calling `setConversationArchived(id, !archived)` (design.md ADR-5)

**Checkpoint**: All three user stories independently functional and consistent across both app surfaces.

---

## Phase 5: Polish & Cross-Cutting Concerns

- [ ] T018 [P] Update `docs/dev/assistant/*` (conversation model: the `archived` flag) and a `docs/usage` page for the archive affordance in the same change (Constitution VI)
- [ ] T019 [P] Self-check the spec's edge cases against the built behavior (design.md §7): open archived URL/deep-ref still opens read-only; deleting an archived conversation removes it from the Archived section too; rapid archive/unarchive is idempotent; archiving does not change which conversation is "current"
- [ ] T020 Run the full `e2e/038-conversation-archive.spec.ts` suite green; confirm `tsc` + lint pass

**Note on FR-007 / US3 acceptance**: phrase as "archive affordance **available in the Build Studio surface**" (not per-row) — a native `<select>` cannot host a per-row kebab (reviewer S3).

---

## Dependencies & Execution Order

- **Phase 1 (Foundational)**: no dependencies — BLOCKS all user stories. T001→T002→T003 (same file, sequential); T004 independent; T005 depends on T002.
- **US1 (Phase 2)**: depends on Phase 1. T006/T007 (tests, [P]) first; T008, T009, T010 are different files → [P]-able among themselves but each depends on T003/T004.
- **US2 (Phase 3)**: depends on Phase 1 and the `ConvRow` menu from T010 (US2's unarchive action lives in that menu). T012→T013→T014 (same file, sequential).
- **US3 (Phase 4)**: depends on Phase 1 (store + `setConversationArchived`); independent of US1/US2's UI files. T016, T017 (same file, sequential).
- **Polish (Phase 5)**: depends on all stories.

### Parallel opportunities

- Within US1: T008 (runs route), T009 (ChatInputV2), T010 (ConversationPanel) touch different files.
- Across stories after Phase 1: US2's `ConversationPanel` work and US3's `AgentSelector` work are in different files.
- Test tasks per story are [P] (same spec file, different test cases).

### Implementation strategy (full-scope, one pass)

Per the user's preference for full-scope delivery: execute Phases 1→2→3→4→5 in a single implementation pass (all three user stories at once), validating at each checkpoint, rather than an MVP-only stop after US1.
