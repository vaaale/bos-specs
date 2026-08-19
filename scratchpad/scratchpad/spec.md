# Feature Specification: Scratchpad (Conversation-Scoped Note-Taking)

**Feature Branch**: `scratchpad`

**Created**: 2026-01-XX

**Status**: Ready for Implementation (Architecture Aligned with BOS Patterns)

**Input**: User request for a "scratchpad" feature with MCP tools: `write_note`, `read_notes`, `edit_note`, `delete_note`. The scratchpad is strictly **conversation-scoped** and **in-memory only**; notes exist only for the duration of the current chat session.

> This feature provides an ephemeral note-taking system accessible via MCP tools, allowing agents and users to create, read, edit, and delete notes within a single conversation session. Notes are stored exclusively in memory and are discarded when the conversation ends or reloads. Ideal for temporary context, drafting, or mid-session collaboration between agents.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Write a new note (Priority: P1)

A user or agent creates a new note with a title and content using the `write_note` tool.

**Acceptance Scenarios**:

1. **Given** the user/agent calls `write_note` with a unique title and content, **When** the note is created, **Then** it is saved to persistent storage and returns success with the note ID.
2. **Given** the user/agent calls `write_note` with a title that already exists, **When** the operation completes, **Then** it returns an error indicating the note already exists (unless overwrite flag is set).
3. **Given** the user/agent calls `write_note` without a title, **When** validation runs, **Then** it returns an error requiring a title.
4. **Given** the user/agent calls `write_note` with empty content, **When** the note is created, **Then** it accepts empty content (allowing blank notes).

### User Story 2 - Read existing notes (Priority: P1)

A user or agent retrieves notes using the `read_notes` tool, either listing all notes or reading a specific note by title.

**Acceptance Scenarios**:

1. **Given** there are multiple notes saved, **When** the user/agent calls `read_notes` without parameters, **Then** it returns a list of all notes with their titles, creation dates, and content previews.
2. **Given** a specific note exists, **When** the user/agent calls `read_notes` with a title parameter, **Then** it returns the full content of that note.
3. **Given** no notes exist, **When** the user/agent calls `read_notes`, **Then** it returns an empty list or appropriate message.
4. **Given** a non-existent note title is requested, **When** read_notes is called, **Then** it returns an error indicating the note was not found.
5. **Given** many notes exist (>50), **When** listing notes, **Then** results are paginated or limited with an option to fetch more.

### User Story 3 - Edit an existing note (Priority: P1)

A user or agent modifies the content of an existing note using the `edit_note` tool.

**Acceptance Scenarios**:

1. **Given** a note exists, **When** the user/agent calls `edit_note` with the title and new content, **Then** the note is updated and the modification timestamp is refreshed.
2. **Given** a non-existent note title is provided, **When** edit_note is called, **Then** it returns an error indicating the note was not found.
3. **Given** the user/agent calls `edit_note` with only partial fields (e.g., just content without title), **When** validation runs, **Then** it requires at minimum the title and one field to update.
4. **Given** a note is edited multiple times, **When** reading its history, **Then** all modifications are tracked with timestamps (if versioning is enabled).

### User Story 4 - Delete a note (Priority: P1)

A user or agent removes a note using the `delete_note` tool.

**Acceptance Scenarios**:

1. **Given** a note exists, **When** the user/agent calls `delete_note` with the title, **Then** the note is permanently deleted and success is returned.
2. **Given** a non-existent note title is provided, **When** delete_note is called, **Then** it returns an error indicating the note was not found.
3. **Given** the user/agent calls `delete_note` on a note, **When** confirmation is required, **Then** the tool either requires explicit confirmation flag or asks for confirmation (configurable).
4. **Given** a note is deleted, **When** listing notes afterward, **Then** the deleted note no longer appears in results.

### User Story 5 - Search and filter notes (Priority: P2)

A user or agent can search for notes by content or filter by date range.

**Acceptance Scenarios**:

1. **Given** notes contain various content, **When** the user/agent calls `read_notes` with a search query, **Then** it returns notes matching the search term in title or content.
2. **Given** notes have different creation dates, **When** filtering by date range, **Then** only notes within that range are returned.
3. **Given** no notes match the search criteria, **When** searching, **Then** an empty result set is returned.

### User Story 6 - List note metadata (Priority: P2)

Users/agents can view note metadata without retrieving full content.

**Acceptance Scenarios**:

1. **Given** multiple notes exist, **When** calling a "list" operation, **Then** it returns titles, creation dates, modification dates, and content sizes without full content.
2. **Given** a specific note is requested with metadata-only flag, **When** read_notes is called, **Then** it returns only the metadata fields.

### User Story 7 - Persistence via operation replay (Priority: P1)

Notes survive conversation reloads by replaying an operation log stored with the conversation metadata.

**Acceptance Scenarios**:

1. **Given** notes are created/edited in a conversation, **When** the user refreshes the page, **Then** all notes are reconstructed by replaying the operation log and appear exactly as before.
2. **Given** multiple operations (write/edit/delete) occurred, **When** the file system is checked, **Then** an operation log (`data/scratchpad/{conversationId}/ops.json`) exists containing the history of changes.
3. **Given** a conversation is reloaded, **When** initialization completes, **Then** the scratchpad state matches the last known state before the reload.
4. **Given** a new conversation starts, **When** it initializes, **Then** the operation log is empty and the scratchpad starts fresh.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The scratchpad MUST provide four agent-accessible tools via CopilotKit actions (namespaced as `scratchpad_*`):
  - `scratchpad_write(title: string, content: string)` → `{ success: boolean, noteId: string, message: string }`
  - `scratchpad_read(title?: string)` → `{ notes: Array<{title, created, modified, size}> }` or `{ title, content, created, modified }` for single note
  - `scratchpad_edit(title: string, content: string)` → `{ success: boolean, message: string }`
  - `scratchpad_delete(title: string)` → `{ success: boolean, message: string }`

- **FR-002**: Each **Note** entity MUST contain: `id` (unique identifier), `title` (unique string within session), `content` (text), `created` (ISO timestamp), `modified` (ISO timestamp). Tags are optional.

- **FR-003**: The scratchpad MUST derive its persistent state exclusively from the **conversation history**. On conversation load, the system MUST scan the conversation for all scratchpad tool calls (`write_note`, `edit_note`, `delete_note`), replay them in chronological order, and reconstruct the current note state. NO separate operation log or secondary storage is allowed.

- **FR-004**: Note titles MUST be unique within the current session; attempts to create duplicate titles MUST fail (no overwrite flag needed for ephemeral notes).

- **FR-005**: The `read_notes` tool MUST support:
  - Listing all notes (returns metadata only)
  - Reading a specific note by title (returns full content)
  - Simple search is optional (can be added later if needed)
  - No pagination required for typical small note sets

- **FR-006**: The `edit_note` tool MUST require both title and new content; partial updates are not supported for simplicity.

- **FR-007**: The `delete_note` tool MUST delete immediately without confirmation (ephemeral data, user accepts risk).

- **FR-008**: All operations MUST validate input parameters and return descriptive error messages for invalid requests.

- **FR-009**: The system MUST handle concurrent access gracefully (file locking or atomic writes).

- **FR-010**: Notes MUST support optional tagging via metadata for better organization.

### Key Entities

- **Note** — the core entity containing: `id`, `title`, `content`, `created`, `modified`. Stored in memory only.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An agent can create a new note using `write_note` and retrieve it immediately with `read_notes`.
- **SC-002**: An agent can list all notes, search for specific content, and read individual notes.
- **SC-003**: An agent can modify existing notes using `edit_note` without affecting other notes.
- **SC-004**: An agent can permanently delete notes using `delete_note`.
- **SC-005**: All notes persist across BOS restarts and are accessible after reload.
- **SC-006**: Error handling provides clear messages for invalid operations (duplicate titles, not found, etc.).
- **SC-007**: The tool interface is intuitive and follows MCP conventions.

## Assumptions & Dependencies

- Depends on **CopilotKit actions** (`useCopilotAction`) for exposing tools to agents (not MCP gateway).
- Depends on conversation JSON files (`/Documents/Chats/<id>.json`) being persisted and readable.
- Depends on **conversation history** (`messages[]` array) being fully preserved and queryable.
- Notes are plain text; rich text or markdown support is optional (can be added later).
- No authentication/authorization is required (all agents in the conversation can access all notes).
- **Persistence Strategy**: **Conversation-History Replay** via lazy initialization on first tool call. No secondary storage; history is the source of truth.
- Implementation pattern: CopilotKit actions in `src/components/agent/ScratchpadActions.tsx` + logic in `src/lib/agent/scratchpad/`.

## Design notes (non-normative)

**Architecture overview:**
```
┌─────────────────────────────────────────────────────────────┐
│                   Conversation History                      │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Message 1: User: "Write a note"                      │  │
│  │  Message 2: Assistant [Tool Call: scratchpad_write("Ideas", "...")] │
│  │  Message 3: Tool Result: { success: true }            │  │
│  │  ... (all tool calls stored in conversation.messages) │  │
│  └───────────────────────────────────────────────────────┘  │
│                          ▼                                   │
│              Lazy Replay on First Tool Call                  │
│              (extract from messages[] → reconstruct Map)     │
│                          ▼                                   │
┌─────────────────────────────────────────────────────────────┐
│         Client-Side Store (per conversationId)              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Module-level Map<conversationId, Map<title, Note>>   │  │
│  │  In: src/lib/agent/scratchpad/store.ts                │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
               ┌─────────────────────────────┐
               │ CopilotKit Actions          │
               │ src/components/agent/       │
               │   ScratchpadActions.tsx     │
               │ - scratchpad_write          │
               │ - scratchpad_read           │
               │ - scratchpad_edit           │
               │ - scratchpad_delete         │
               └─────────────────────────────┘
```

**Storage location:**
- **In-Memory**: Module-level `Map<conversationId, Map<title, Note>>` in `src/lib/agent/scratchpad/store.ts`
- **Persistent Source of Truth**: **Conversation History** (`/Documents/Chats/<id>.json` → `messages[]`)
- **NO secondary storage**: No log files; history is the operation log

**Note structure (in-memory):**
```typescript
interface Note {
  id: string;        // UUID or timestamp-based unique ID
  title: string;     // Unique within conversation
  content: string;   // Full note text
  created: string;   // ISO timestamp (from first scratchpad_write call)
  modified: string;  // ISO timestamp (from last scratchpad_edit call)
}
```

**Tool implementations (CopilotKit actions):**
- `scratchpad_write(title, content)`: 
  - Ensure notes Map is initialized for current conversationId (replay if needed)
  - Create new Note entry or update if exists
  - Set `created` and `modified` timestamps
  - Tool call is automatically recorded in `conversation.messages[]` by BOS
- `scratchpad_read(title?)`: 
  - Ensure notes Map is initialized (replay if needed)
  - Read directly from in-memory Map
  - Return single note or list of all notes
- `scratchpad_edit(title, content)`: 
  - Ensure notes Map is initialized (replay if needed)
  - Validate note exists; update content and `modified` timestamp
  - Tool call recorded in history automatically
- `scratchpad_delete(title)`: 
  - Ensure notes Map is initialized (replay if needed)
  - Validate note exists; remove from Map
  - Tool call recorded in history automatically

**Initialization (lazy replay on first tool call):**
1. On any scratchpad tool invocation, check if notes Map exists for current `conversationId`
2. If not: 
   - Load conversation JSON (`/Documents/Chats/<id>.json`)
   - Extract all `scratchpad_*` tool calls from `messages[]` in order
   - Replay operations to reconstruct `notes` Map
   - Cache Map in module-level store for this conversationId
3. If yes: use cached Map (no re-play needed)
4. Handle edge cases gracefully (edit/delete on non-existent note in history → ignore with warning log)

**Error handling:**
- All tools return consistent error format: `{ success: false, error: "ERROR_CODE", message: "Human-readable" }`
- Common errors: `NOTE_NOT_FOUND`, `NOTE_EXISTS`, `INVALID_TITLE`, `VALIDATION_ERROR`
- **No `LOG_CORRUPTED` error** - conversation history is trusted; malformed calls are skipped

**Conversation lifecycle:**
- New conversation: empty Map, no history yet
- Tool call triggers lazy replay if not already initialized
- Notes survive reloads because history persists in conversation JSON
- Conversation switch: new conversationId → fresh initialization/replay
- **Zero data loss**: As long as conversation JSON is preserved, notes survive

## Clarifications

The following ambiguities were resolved during the specification process:

1. **Persistence Model**: Initially considered file-based persistence, but clarified that notes must survive reloads without secondary storage. **Decision**: Use **conversation-history replay** - extract tool calls from existing conversation history (`messages[]` in `/Documents/Chats/<id>.json`) and replay them to reconstruct state. No log files or separate persistence layer required.

2. **Scope**: Clarified that the scratchpad is strictly **conversation-scoped** (not global). Notes exist only within a single conversation thread but persist across reloads of that same conversation via history replay.

3. **Tool Interface & Naming**: 
   - Initially proposed generic names (`write_note`, `read_notes`, etc.), but clarified that BOS uses **namespaced capability IDs**. **Decision**: Use `scratchpad_write`, `scratchpad_read`, `scratchpad_edit`, `scratchpad_delete` to follow BOS conventions and avoid collisions.
   - Simplified signatures by removing unnecessary flags (`overwrite`, `confirm`, pagination) since notes are ephemeral and conversation-bound.

4. **Implementation Architecture**: 
   - Initially proposed "MCP server" pattern, but clarified that BOS uses **CopilotKit actions** via `useCopilotAction`. **Decision**: Implement as `src/components/agent/ScratchpadActions.tsx` with core logic in `src/lib/agent/scratchpad/`.
   - Initially proposed `conversationState.scratchpad.notes` store, but clarified that BOS has no global conversation state slice. **Decision**: Use module-level `Map<conversationId, Map<title, Note>>` with lazy initialization on first tool call.
   - Initially proposed global conversation init hook, but clarified there's no single init point. **Decision**: Implement **lazy replay** inside action handlers (idempotent, cached per conversation).

5. **Test Framework**: Initially proposed Jest coverage targets, but clarified that BOS uses hand-written `runAll()` tests. **Decision**: Write tests matching existing patterns (`drive-adapter.test.ts`) without automated coverage measurement.

## Notes

- This is a minimal, agent-focused note-taking system without a dedicated UI (notes are managed via CopilotKit actions: `scratchpad_write`, `scratchpad_read`, `scratchpad_edit`, `scratchpad_delete`).
- **Persistence Strategy**: **Conversation-History Replay** with lazy initialization. No secondary storage; conversation JSON history is the source of truth.
- Notes are **conversation-scoped** but **survive reloads** because tool calls persist in `/Documents/Chats/<id>.json`.
- Implementation follows BOS patterns: CopilotKit actions (`src/components/agent/ScratchpadActions.tsx`) + core logic (`src/lib/agent/scratchpad/`).
- This design eliminates persistence complexity: no log files, no state-log sync, no corruption risks.
- Future enhancements could include: scratchpad UI app, rich text support, note templates, or cross-conversation sharing.
- The feature complements the agent model by providing durable, conversation-bound storage with zero additional infrastructure.
