# Implementation Tasks: Scratchpad (Conversation-Scoped Note-Taking)

**Feature**: Scratchpad  
**Status**: Ready for Implementation  
**Branch**: `bos/bos-scratchpad-tools`

## Task List

### Phase 1: Core Logic & Types

#### Task 1.1: Create Type Definitions
**Priority**: P0  
**Dependencies**: None  
**Estimated Time**: 30 min

**Description**: Define TypeScript interfaces for notes, operations, and tool results.

**Acceptance Criteria**:
- [ ] Create `src/lib/agent/scratchpad/types.ts` with interfaces:
  - `Note`: `{ id: string, title: string, content: string, created: string, modified: string }`
  - `ToolResult`: `{ success: boolean, noteId?: string, message?: string, error?: string, notes?: Note[], note?: Note }`
  - Error codes: `NOTE_NOT_FOUND`, `NOTE_EXISTS`, `INVALID_TITLE`, `VALIDATION_ERROR`
- [ ] Add JSDoc comments for each interface
- [ ] Export types for use across modules

**Implementation Notes**:
- Use ISO 8601 format for timestamps
- Follow existing BOS TypeScript conventions

---

#### Task 1.2: Implement Per-Conversation Store
**Priority**: P0  
**Dependencies**: Task 1.1  
**Estimated Time**: 45 min

**Description**: Create in-memory state management for notes keyed by conversationId.

**Acceptance Criteria**:
- [ ] Implement module-level store: `Map<conversationId, Map<title, Note>>`
- [ ] Function `getNotes(conversationId): Map<string, Note>` - returns notes for conversation (initializes if needed)
- [ ] Function `setNote(conversationId, title, note)` - adds/updates a note
- [ ] Function `deleteNote(conversationId, title)` - removes note from Map
- [ ] Function `getNote(conversationId, title)` - retrieves single note
- [ ] Handle initialization: Create empty inner Map if conversation not seen before

**Implementation Notes**:
- Store in `src/lib/agent/scratchpad/store.ts`
- Use JavaScript `Map` for O(1) lookups
- Add debug logging for state changes (debug level)

---

#### Task 1.3: Implement Operation Extraction & Replay
**Priority**: P0  
**Dependencies**: Task 1.2  
**Estimated Time**: 1 hour

**Description**: Parse conversation history to extract and replay scratchpad tool calls.

**Acceptance Criteria**:
- [ ] Create `src/lib/agent/scratchpad/replay.ts`
- [ ] Implement `extractScratchpadOps(messages: unknown[]): ScratchpadOperation[]`:
  - Filter messages for tool calls with names: `scratchpad_write`, `scratchpad_edit`, `scratchpad_delete`
  - Extract arguments and timestamps from each tool call
  - Return operations in chronological order
- [ ] Implement `replayOperations(conversationId: string, ops: ScratchpadOperation[])`:
  - Process operations sequentially
  - `scratchpad_write`: Create note with `created` = op timestamp
  - `scratchpad_edit`: Update content, set `modified` = op timestamp (ignore if note doesn't exist)
  - `scratchpad_delete`: Remove note from Map (ignore if not found)
- [ ] Handle malformed tool calls gracefully (skip with warning log)

**Edge Cases**:
- Edit/delete on non-existent note in history → silently ignore (log warning)
- Multiple writes to same title → last one wins (natural from sequential replay)

**Implementation Notes**:
- Access conversation messages from loaded JSON structure
- Parse tool call arguments from message structure
- Add debug logging for each replayed operation

---

#### Task 1.4: Implement CRUD Handlers
**Priority**: P0  
**Dependencies**: Tasks 1.2, 1.3  
**Estimated Time**: 1 hour

**Description**: Create pure functions for note operations.

**Acceptance Criteria**:
- [ ] Create `src/lib/agent/scratchpad/handlers.ts`
- [ ] Implement `writeNote(conversationId, title, content)`:
  - Validate input (title non-empty, content is string)
  - Check for duplicate title; return error if exists
  - Generate unique ID (UUID or timestamp-based)
  - Set `created` and `modified` timestamps
  - Store note via `setNote()`
  - Return success with `noteId`
- [ ] Implement `readNotes(conversationId, title?)`:
  - If title provided: return single note or error if not found
  - If no title: return array of all notes with count
- [ ] Implement `editNote(conversationId, title, content)`:
  - Validate input
  - Check if note exists; return error if not found
  - Update content and `modified` timestamp
  - Return success
- [ ] Implement `deleteNote(conversationId, title)`:
  - Validate title
  - Check if note exists; return error if not found
  - Remove from Map
  - Return success

**Error Scenarios**:
- Missing/empty title → `INVALID_TITLE`
- Duplicate title (write) → `NOTE_EXISTS`
- Note not found → `NOTE_NOT_FOUND`

**Implementation Notes**:
- Pure functions (no side effects except state updates)
- Consistent error format: `{ success: false, error: CODE, message: "..." }`

---

#### Task 1.5: Unit Tests for Core Logic
**Priority**: P0  
**Dependencies**: Tasks 1.2–1.4  
**Estimated Time**: 1.5 hours

**Description**: Write hand-run unit tests for handlers and replay logic.

**Acceptance Criteria**:
- [ ] Create `src/lib/agent/scratchpad/__tests__/handlers.test.ts`:
  - Test `writeNote`: success, duplicate title, invalid title, empty content allowed
  - Test `readNotes`: single note (found/not found), list all (empty/non-empty)
  - Test `editNote`: success, note not found, preserve created timestamp
  - Test `deleteNote`: success, note not found
- [ ] Create `src/lib/agent/scratchpad/__tests__/replay.test.ts`:
  - Test basic replay: write → replay → verify note exists
  - Test edit sequence: write → edit → replay → verify final content
  - Test delete sequence: write → delete → replay → verify note gone
  - Test complex sequence: multiple writes/edits/deletes in order
  - Test edge cases: edit/delete non-existent notes in history
- [ ] Implement `runAll()` async entry matching `drive-adapter.test.ts` style
- [ ] All tests pass when run manually

**Implementation Notes**:
- Use mock conversation data for replay tests
- Test error paths thoroughly
- No automated coverage measurement (hand-run only)

---

### Phase 2: CopilotKit Actions

#### Task 2.1: Create Actions Component
**Priority**: P0  
**Dependencies**: Task 1.4  
**Estimated Time**: 1 hour

**Description**: Implement CopilotKit actions to expose tools to agents.

**Acceptance Criteria**:
- [ ] Create `src/components/agent/ScratchpadActions.tsx`
- [ ] Import `useCopilotAction` from CopilotKit (via `gated-action.ts` wrapper if available)
- [ ] Register four actions:
  - `scratchpad_write`: { title, content } → { success, noteId, message }
  - `scratchpad_read`: { title? } → { notes, total } or { note }
  - `scratchpad_edit`: { title, content } → { success, message }
  - `scratchpad_delete`: { title } → { success, message }
- [ ] Each action handler:
  - Gets current conversationId (from context or props)
  - Ensures notes Map is initialized (triggers replay if needed)
  - Calls corresponding handler function from `handlers.ts`
  - Returns result to CopilotKit
- [ ] Proper TypeScript typing for all parameters and return values

**Implementation Notes**:
- Follow pattern from `SkillsActions.tsx` or `WorkflowActions.tsx`
- Use `gated-action.ts` wrapper if it exists for capability checking
- Add error handling with descriptive messages

---

#### Task 2.2: Register Capabilities
**Priority**: P0  
**Dependencies**: Task 2.1  
**Estimated Time**: 30 min

**Description**: Declare scratchpad capabilities in the registry.

**Acceptance Criteria**:
- [ ] Open `src/lib/agent/capabilities-registry.ts`
- [ ] Add four capability entries:
  - `scratchpad_write`: name, description, input schema, output schema
  - `scratchpad_read`: name, description, input schema, output schema
  - `scratchpad_edit`: name, description, input schema, output schema
  - `scratchpad_delete`: name, description, input schema, output schema
- [ ] Ensure capability IDs match action names exactly
- [ ] Follow existing entry format (e.g., `skill_list`, `workflow_run`)

**Implementation Notes**:
- Schemas should match the actual function signatures
- Descriptions should be clear for agent consumption

---

#### Task 2.3: Mount Actions Component
**Priority**: P0  
**Dependencies**: Task 2.1  
**Estimated Time**: 30 min

**Description**: Integrate ScratchpadActions into the app component tree.

**Acceptance Criteria**:
- [ ] Find parent component that mounts other `*Actions` components (e.g., in `src/app/`, `src/components/`, or chat-related components)
- [ ] Import `<ScratchpadActions />`
- [ ] Mount it alongside other action components (e.g., `<SkillsActions />`, `<WorkflowActions />`)
- [ ] Verify component renders without errors

**Implementation Notes**:
- Component doesn't need visible UI; it's side-effect only (registering actions)
- Follow existing mounting pattern in codebase

---

#### Task 2.4: Integration Tests for Actions
**Priority**: P0  
**Dependencies**: Tasks 1.5, 2.3  
**Estimated Time**: 1 hour

**Description**: Test end-to-end tool invocation and conversation reload scenarios.

**Acceptance Criteria**:
- [ ] Manual test: Write note via Assistant → read note → verify content
- [ ] Manual test: Edit note via Assistant → read note → verify updated content
- [ ] Manual test: Delete note via Assistant → read notes → verify gone
- [ ] Manual test: Reload page (conversation persists) → write/read/edit/delete → verify state preserved
- [ ] Manual test: Switch conversations → new conversation starts empty → switch back → notes restored
- [ ] Document any issues found

**Implementation Notes**:
- Use Assistant chat interface for testing
- Test with multiple conversations open simultaneously
- Verify lazy replay triggers correctly on first tool call

---

### Phase 3: Integration & Polish

#### Task 3.1: Add Debug Logging
**Priority**: P1  
**Dependencies**: Phase 2 complete  
**Estimated Time**: 30 min

**Description**: Add structured logging for debugging and monitoring.

**Acceptance Criteria**:
- [ ] Log tool invocations (info level): action name, arguments, result
- [ ] Log replay events (debug level): operations extracted, state after replay
- [ ] Log errors with context (error level)
- [ ] Use BOS central logging system if available, or console with consistent format
- [ ] Include conversationId in log messages for traceability

**Implementation Notes**:
- Follow BOS logging conventions
- Avoid logging sensitive content (truncate long note contents if needed)

---

#### Task 3.2: Performance Testing
**Priority**: P1  
**Dependencies**: Phase 2 complete  
**Estimated Time**: 45 min

**Description**: Benchmark tool performance and replay speed.

**Acceptance Criteria**:
- [ ] Measure tool call latency: target < 10ms per operation
- [ ] Measure replay time with 100 operations: target < 100ms
- [ ] Measure replay time with 1000 operations: document results
- [ ] Test conversation switching performance (cache hit vs miss)
- [ ] Identify bottlenecks if targets not met

**Implementation Notes**:
- Use console.time() or profiling tools
- Create test conversations with large operation histories

---

#### Task 3.3: Documentation Updates
**Priority**: P1  
**Dependencies**: All phases complete  
**Estimated Time**: 1 hour

**Description**: Update user and developer documentation.

**Acceptance Criteria**:
- [ ] Add user-facing docs: `docs/usage/features/scratchpad.md` (or similar)
  - Explain what scratchpad is
  - List available tools with examples
  - Note behavior on reload and conversation switching
- [ ] Update developer docs: `docs/dev/features/scratchpad.md`
  - Architecture overview (lazy replay, per-conversation store)
  - Implementation details (file layout, key functions)
  - Testing guidelines
- [ ] Add tool descriptions to capability registry entries (if not already clear)

**Implementation Notes**:
- Follow existing documentation style in BOS
- Include code examples for common use cases

---

#### Task 3.4: Final QA Testing
**Priority**: P1  
**Dependencies**: All previous tasks  
**Estimated Time**: 1 hour

**Description**: Comprehensive manual testing before release.

**Acceptance Criteria**:
- [ ] All hand-run unit tests pass
- [ ] Manual testing in live BOS environment:
  - Create, read, edit, delete notes via Assistant
  - Refresh page → verify notes persist
  - Switch conversations → verify isolation
  - Test error scenarios (invalid inputs, missing notes)
- [ ] No console errors or warnings
- [ ] Performance meets targets
- [ ] Documentation is accurate and complete

**Implementation Notes**:
- Use checklist to ensure nothing is missed
- Report any issues found for fix before merge

---

## Task Dependencies Graph

```
Phase 1: Core Logic
├── 1.1 Types → 1.2 Store → 1.3 Replay → 1.4 Handlers → 1.5 Unit Tests

Phase 2: Actions
├── 1.4 Handlers → 2.1 Actions Component → 2.2 Capabilities → 2.3 Mount → 2.4 Integration Tests
│                              ↑
│                      (depends on 1.2 Store, 1.3 Replay)

Phase 3: Polish
├── 3.1 Logging (depends on Phase 2)
├── 3.2 Performance (depends on Phase 2)
├── 3.3 Documentation (depends on all)
└── 3.4 Final QA (depends on all)
```

## Notes for Developer

- **Order of Operations**: Complete tasks in numerical order within each phase
- **Testing Approach**: Hand-run tests with `runAll()` entry; no automated test runner available
- **Code Style**: Follow existing BOS conventions (TypeScript strict mode, ESLint rules)
- **Commit Strategy**: One commit per task or logical group; descriptive commit messages
- **Branch**: Working on `bos/bos-scratchpad-tools` feature branch
- **Review**: After implementation, run `analyze` and `converge` steps to verify spec/code alignment

## Acceptance Criteria Summary

The feature is complete when:
1. ✅ All four CopilotKit actions work correctly (`scratchpad_write`, `scratchpad_read`, `scratchpad_edit`, `scratchpad_delete`)
2. ✅ Notes survive conversation reloads via lazy history replay
3. ✅ Hand-run unit tests pass for handlers and replay logic
4. ✅ Integration tests pass for all scenarios (reload, conversation switching)
5. ✅ Performance targets met (<10ms/tool, <100ms replay for 100 ops)
6. ✅ Documentation complete and accurate
7. ✅ No regressions in existing BOS functionality
