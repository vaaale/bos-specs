# Tasks: Tool State Color Coding in the Assistant Info Panel

**Input**: spec.md, plan.md, design.md, mockup.html — all in `/Specs/user-specs/assistant/042-tool-color-coding/`

**Prerequisites**: plan.md ✅, spec.md ✅ (US1, US2 are P1)

**Tests**: Required — SC-001 (predicate-identity check), edge-case table, loading strictness. Written hand-run (no test runner) per repo convention.

**Organization**: Grouped by user story. US2 depends on US1 (both modify the same file `InfoPanelV2.tsx`; US2 adds the conversation-awareness layer on top of US1's agent-awareness layer).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: US1 or US2
- Paths are repo-relative

---

## Phase 1: Foundational (Blocking — all stories depend on this)

**Purpose**: Create the pure classifier module and its test. No user story work can begin until the classifier exists.

- [ ] **T001** [US1] Write the hand-run test `src/lib/agent/__tests__/tool-state.test.ts` with all cases (SC-001 predicate-identity matrix, edge-case table, loading strictness). Import from `../tool-state` (not yet created — test will fail).
- [ ] **T002** [US1] Create `src/lib/agent/tool-state.ts` — framework-free module. Export:
  - `type ToolState = "granted" | "deferredHidden" | "deferredRevealed" | "neutral"`
  - `const DISCOVERY_TOOL_IDS: ReadonlySet<string>` (`{"find_tools", "find_agent"}`)
  - `function classifyToolState(toolId, ctx: { allow, deferred, revealed, isDiscovery }): ToolState`
  - Follow the exact predicate order from design §4 (neutral checks first, then blue, then orange, else green).
  - Add a comment at the top pointing at the `messages.ts` `deriveRevealedIds` as the intended revealed-set source (design risk 1).
  - Verify T001 passes after this task.

**Checkpoint**: Classifier is pure, tested, and importable by the component. SC-001 test passes.

---

## Phase 2: User Story 1 — Granted / Not-Granted Color Coding (P1) 🎯 MVP

**Goal**: The Tools tab shows granted tools in green and non-granted tools in neutral grey. Switching agents updates the colors.

**Independent Test**: Select two agents with different `tools` allowlists; open Tools tab for each; confirm granted rows are green (wrench icon), non-granted rows are neutral (existing grey).

### Implementation for User Story 1

- [ ] **T003** [US1] In `src/components/agent/v2/InfoPanelV2.tsx`: extend the existing `caps` state to include `tools: string[]` and `deferredTools: string[]` (both nullable while loading). Read `agent?.tools ?? []` and `agent?.deferredTools ?? []` from the same `/api/assistant/agent` response the component already fetches. No new fetch, no new route.
- [ ] **T004** [US1] In `InfoPanelV2.tsx` `ToolsTab`: accept `tools: string[] | null` and `deferredTools: string[] | null` as props (from `caps`). Compute `allowSet = new Set(tools ?? [])` and `deferredSet = new Set(deferredTools ?? [])`. For each row, call `classifyToolState(t.name, { allow: allowSet, deferred: deferredSet, revealed: new Set(), isDiscovery: DISCOVERY_TOOL_IDS.has(t.name) })` and map the state to the icon's `className`:
  - `"granted"` → `text-emerald-400`
  - `"deferredHidden"` → `text-amber-400`
  - `"deferredRevealed"` → `text-sky-400`
  - `"neutral"` → `text-white/40` (the existing class)
  - Pass the state color to the `<Wrench>` component's `className` (replace the static `text-white/40`). Keep all other row content (name, description, layout) unchanged.
  - While `tools === null` (loading), render every row neutral (do NOT use the `allows()` helper).
- [ ] **T005** [US1] In `src/components/agent/v2/AssistantChatV2.tsx`: add `conversationId={conversationId}` to the `<InfoPanelV2>` call (one line — `conversationId` is already in scope at line 55). `InfoPanelV2` accepts the new prop but does not use it yet (that's US2).

**Checkpoint**: US1 fully functional — granted tools green, non-granted neutral, agent switch updates colors. `deferredHidden` and `deferredRevealed` rows exist in the classifier but all show as `deferredHidden` (amber) since `revealed` is always `∅` at this point. The orange→blue transition is US2.

---

## Phase 3: User Story 2 — Deferred Hidden vs. Revealed Color Coding (P1)

**Goal**: A deferred tool that has been revealed by a prior `find_tools` call in the active conversation is blue; one not yet revealed is orange. Opening a different conversation reverts the state.

**Independent Test**: Configure an agent with a deferred tool. Open its Tools tab in a fresh conversation (orange). Run a turn that calls `find_tools` and reveals it. Confirm the row is now blue. Switch to a conversation without the reveal — back to orange.

### Implementation for User Story 2

- [ ] **T006** [US2] In `InfoPanelV2.tsx` `ToolsTab`: accept `conversationId: string` as a prop. Add `const messages = useChatSelector(conversationId, (s) => s.messages)` (import `useChatSelector` from `@/lib/assistant/client/chat-store`). Compute `const revealed = deriveRevealedIds(messages)` (import from `@/lib/assistant/messages` — the client-safe version, NOT `src/lib/agent/tool-gate.ts`). Pass `revealed: new Set(revealed)` into every `classifyToolState` call (replacing the `new Set()` from T004).
- [ ] **T007** [US2] In `InfoPanelV2.tsx`: pass `conversationId` from the component's new prop to `<ToolsTab conversationId={conversationId ?? ""} …>`.

**Checkpoint**: US2 fully functional — deferred-hidden rows are orange, deferred-revealed rows are blue, conversation change reverts, in-conversation reveal flips orange→blue. All four states are live.

---

## Phase 4: Polish & Cross-Cutting Concerns

- [ ] **T008** [US1] In `InfoPanelV2.tsx` `ToolsTab`: add the 4-row legend block at the top of the Tools tab content (above the first group). Per the mockup: a `div` with `mb-3 space-y-1 border-b border-white/10 pb-2`, four rows each an 8px dot + 10px label:
  - `<span className="h-2 w-2 rounded-full bg-emerald-400" />` — "Granted"
  - `<span className="h-2 w-2 rounded-full bg-amber-400" />` — "Deferred · hidden"
  - `<span className="h-2 w-2 rounded-full bg-sky-400" />` — "Deferred · revealed"
  - `<span className="h-2 w-2 rounded-full bg-white/40" />` — "Not granted"
  - Labels use `text-[10px] text-white/55`.
- [ ] **T009** [P] [US1] Update the Assistant end-user doc: add one sentence noting the Tools tab colors reflect the selected agent's allowlist and the active conversation's discovered tools (green = granted, orange = deferred/hidden, blue = deferred/discovered, grey = not granted). Find the doc under `/Docs/usage/` (search for "Assistant" or "Info Panel"). This is the Constitution VI (Specs & Docs in sync) action.
- [ ] **T010** [US1] Run `npx tsc --noEmit` and `npm run lint` on all changed files. Fix any errors. Do NOT run `npm run build` while `next dev` is running.

**Checkpoint**: Feature complete. All four states visible, legend present, doc updated, typecheck and lint pass.

---

## Dependencies & Execution Order

```
T001 (test, fails)
  └─ T002 (classifier module)  ← blocks everything
       └─ T003 (extend caps state)
            └─ T004 (ToolsTab coloring)
                 └─ T005 (pass conversationId prop — can be [P] with T003–T004)
                      └─ T006 (useChatSelector + deriveRevealedIds in ToolsTab)
                           └─ T007 (pass conversationId to ToolsTab)
                                └─ T008 (legend)
                                └─ T009 [P] (doc update — can run in parallel with T006–T008)
                                └─ T010 (typecheck + lint — final gate)
```

**Parallel opportunities**:
- T005 and T009 are [P] with their respective phases (different files / no dependencies).
- Once T002 is done, T003–T007 are sequential (same file).
- T009 (doc) can run in parallel with T006–T008 (different files).

**MVP checkpoint**: After T002 + T003 + T004 + T005 — the panel shows green/neutral (US1) and amber for all deferred (orange, since revealed is always ∅). This is a viable, demonstrable increment.

**Full feature**: All tasks through T010.

---

## Notes

- The classifier module (`tool-state.ts`) is the single source of truth for the four-state predicate. The SC-001 test pins it against `visibleTools`. If the test fails after implementation, the classifier (not the test) is wrong.
- The `deriveRevealedIds` import in `ToolsTab` MUST come from `@/lib/assistant/messages` (client-safe, zero imports). A comment at the import site should note this (design risk 1, ADR-2).
- The `allows()` helper in `InfoPanelV2.tsx` (used by Skills/MCP tabs) treats `null`/`[]` as "allow all" — do NOT reuse it for the Tools tab. The classifier uses strict `allow.has` semantics.
- No new files are created outside `src/lib/agent/tool-state.ts` and its test. No new routes, no new dependencies, no server-side changes.
