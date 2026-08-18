# Implementation Plan: Scratchpad (Conversation-Scoped Note-Taking)

**Feature**: Scratchpad  
**Status**: Planned  
**Implementation Approach**: CopilotKit Actions with Conversation-History Replay

## Overview

This document outlines the implementation strategy for the Scratchpad feature, which provides four CopilotKit actions (`scratchpad_write`, `scratchpad_read`, `scratchpad_edit`, `scratchpad_delete`) for conversation-scoped note management. Notes persist across reloads by replaying tool calls from the conversation history stored in `/Documents/Chats/<id>.json`.

## Architecture Decisions

### 1. Storage Model: Conversation-History Replay
- **Decision**: Use existing conversation JSON files as the source of truth
- **Rationale**: Eliminates secondary storage, ensures consistency, leverages existing BOS infrastructure
- **Impact**: No new file I/O required; state reconstruction happens lazily on first tool call

### 2. Tool Design: CopilotKit Actions (Namespaced)
- **Decision**: Four actions with namespaced IDs (`scratchpad_*`) following BOS conventions
- **Rationale**: Matches existing patterns (`skill_list`, `workflow_run`), avoids naming collisions
- **Impact**: Tools registered via `useCopilotAction` in `ScratchpadActions.tsx`; declared in `capabilities-registry.ts`

### 3. State Management: Per-Conversation Module Store
- **Decision**: Store current note state in module-level `Map<conversationId, Map<title, Note>>`
- **Rationale**: No global conversationState slice exists; lazy initialization per conversation
- **Impact**: Replay triggered on first tool call for each conversation; cached thereafter

### 4. Persistence Strategy: Automatic via Conversation JSON
- **Decision**: Rely on BOS's existing conversation persistence (`/Documents/Chats/<id>.json`)
- **Rationale**: Tool calls are automatically recorded in `messages[]`; no custom serialization needed
- **Impact**: Replay extracts tool calls from `messages[]` array

## Component Breakdown

### A. CopilotKit Actions Component
**Location**: `src/components/agent/ScratchpadActions.tsx`

**Responsibilities**:
- Register four actions using `useCopilotAction` (via `gated-action.ts` wrapper)
- Implement action handlers that call core logic functions
- Ensure proper typing and error handling
- Mount component alongside other `*Actions` components

**Key Functions**:
```typescript
// Action registrations
useCopilotAction({ name: 'scratchpad_write', ... })
useCopilotAction({ name: 'scratchpad_read', ... })
useCopilotAction({ name: 'scratchpad_edit', ... })
useCopilotAction({ name: 'scratchpad_delete', ... })

// Handler implementations (delegate to src/lib/agent/scratchpad/handlers.ts)
```

### B. Core Logic Module
**Location**: `src/lib/agent/scratchpad/`

**Files**:
- `types.ts`: TypeScript interfaces (Note, ToolResult, error codes)
- `store.ts`: Per-conversation state management (`Map<conversationId, Map<title, Note>>`)
- `replay.ts`: Operation extraction and replay logic
- `handlers.ts`: Pure functions for CRUD operations on notes Map

**Responsibilities**:
- Define data structures
- Manage in-memory state per conversation
- Extract tool calls from conversation history
- Replay operations to reconstruct state
- Implement CRUD logic (independent of CopilotKit)

### C. Capability Registry Entry
**Location**: `src/lib/agent/capabilities-registry.ts`

**Responsibilities**:
- Declare four scratchpad capabilities with proper namespaced IDs
- Define input/output schemas for each action
- Enable capability gating if needed

### D. Test Suite
**Location**: `src/lib/agent/scratchpad/__tests__/`

**Files**:
- `handlers.test.ts`: Unit tests for CRUD operations
- `replay.test.ts`: Unit tests for extraction and replay logic

**Format**: Hand-written `runAll()` async tests (matching `drive-adapter.test.ts` style)

## Implementation Phases

### Phase 1: Core Logic & Types (P0)
**Goal**: Implement data structures, state management, and CRUD handlers

**Tasks**:
1. Create `src/lib/agent/scratchpad/types.ts` with interfaces
2. Implement `store.ts` with per-conversation Map management
3. Implement `replay.ts` with extraction and replay logic
4. Implement `handlers.ts` with pure CRUD functions
5. Write unit tests (`handlers.test.ts`, `replay.test.ts`)

**Success Criteria**:
- All data structures defined
- State management works correctly for multiple conversations
- Replay accurately reconstructs state from history
- Unit tests pass (hand-run verification)

### Phase 2: CopilotKit Actions (P0)
**Goal**: Wire up actions to expose tools to agents

**Tasks**:
1. Create `src/components/agent/ScratchpadActions.tsx`
2. Register four actions with proper names and schemas
3. Implement handlers that call core logic functions
4. Mount component in appropriate parent (find existing Actions mounting pattern)
5. Add entries to `capabilities-registry.ts`

**Success Criteria**:
- Actions discoverable via agent capability system
- Tools callable by Assistant chat
- Proper error handling and response formatting

### Phase 3: Integration & Polish (P1)
**Goal**: Ensure seamless integration with BOS ecosystem

**Tasks**:
1. Verify lazy replay triggers correctly on first tool call
2. Test conversation switching (cache invalidation)
3. Add debug logging (component: 'scratchpad')
4. Performance testing with large histories
5. Documentation updates (user and developer docs)
6. Final QA testing

**Success Criteria**:
- Tools work across multiple conversations
- No performance degradation with 100+ operations
- Clear error messages in all failure scenarios
- Documentation complete

## Dependencies

### Internal BOS Dependencies
- **CopilotKit**: For action registration (`useCopilotAction`)
- **Conversation System**: For reading/writing `/Documents/Chats/<id>.json`
- **Capabilities Registry**: For tool discovery and gating
- **Central Logging** (optional): For debug logs

### External Dependencies
- None (pure TypeScript implementation)

## Risk Assessment

### Low Risk
- **Core Logic**: Straightforward CRUD operations with well-defined interfaces
- **State Management**: Simple Map-based storage with clear lifecycle

### Medium Risk
- **Lazy Replay**: Requires careful parsing of conversation history; edge cases must be handled
- **Mitigation**: Comprehensive unit tests for replay logic

### Potential Issues
1. **Conversation Format Changes**: If BOS changes how tool calls are stored in `messages[]`
   - **Mitigation**: Abstract extraction logic; add version detection if needed
2. **Performance with Large Histories**: Replaying thousands of operations could be slow
   - **Mitigation**: Benchmark early; consider optimization (e.g., periodic snapshots) if needed
3. **Concurrent Modifications**: Multiple agents modifying same note simultaneously
   - **Mitigation**: Last-write-wins semantics (natural from replay order); document behavior

## Testing Strategy

### Unit Tests (Hand-Run)
- Each handler tested in isolation with mock data
- Replay logic tested with various operation sequences
- Edge cases: edit/delete on non-existent notes, malformed history entries

### Integration Tests (Manual)
- Full conversation lifecycle: create → modify → reload → verify
- Multi-conversation scenarios (switching between conversations)
- Edge case sequences (delete then edit, etc.)

### Manual Testing
- Interactive testing via Assistant chat
- Verify tool discovery in capability system
- Test with real-world usage patterns

## Success Metrics

1. **Functional**: All four tools work as specified with no data loss on reload
2. **Performance**: < 100ms replay time for 100 operations; < 10ms per tool call
3. **Reliability**: All hand-run tests pass; no crashes in stress testing
4. **Usability**: Clear error messages; intuitive tool behavior

## Future Enhancements (Out of Scope)

- Note search/filtering functionality
- Rich text or markdown support
- Note tagging/organization
- Cross-conversation note sharing
- UI app for manual note management
- Export/import functionality

These can be added in future iterations if requested.

## Timeline Estimate

- **Phase 1 (Core Logic)**: 2-3 hours
- **Phase 2 (Actions)**: 1-2 hours
- **Phase 3 (Integration)**: 1-2 hours
- **Testing & Polish**: 1 hour

**Total**: ~5-8 hours of development time

## Rollback Plan

If issues arise:
1. Remove `<ScratchpadActions />` mount from parent component
2. Revert `capabilities-registry.ts` entries
3. Delete `src/lib/agent/scratchpad/` directory and `src/components/agent/ScratchpadActions.tsx`
4. No data persistence cleanup required (no secondary storage)

The feature is isolated and can be removed without affecting other BOS components.
