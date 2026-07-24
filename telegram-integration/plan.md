# Telegram Integration Plan

## Overview

This plan outlines the implementation strategy for the Telegram integration in BrowserOS, breaking down the specification into manageable phases with clear dependencies and milestones. The MVP focuses on bot-based interactions with **agent-driven auto-replies** as a key feature.

## Implementation Phases

### Phase 1: Core Infrastructure & Authentication
**Goal**: Establish the foundation for Telegram API communication  
**Deliverables**:
- MCP server for Telegram (or integration adapter)
- Bot token authentication flow
- Connection management with auto-reconnect
- Secrets storage integration

**Key Tasks**:
1. Design Telegram adapter interface following BrowserOS integration patterns
2. Implement connection lifecycle management (connect, disconnect, reconnect)
3. Create credential storage layer using BrowserOS secrets manager
4. Add connection status monitoring and error handling

**Dependencies**: None (foundation phase)  
**Estimated Duration**: 3-5 days

---

### Phase 2: Contact & Chat Management
**Goal**: Enable user discovery and chat listing  
**Deliverables**:
- Contact list fetching and caching (user-only features, stubbed for bot mode)
- Chat list with last message preview
- Search/filter functionality for contacts
- Profile photo handling

**Key Tasks**:
1. Implement `getContacts()` and `getChats()` API wrappers
2. Design local cache strategy for contacts/chats (TTL-based refresh)
3. Build search indexing for fast contact lookup
4. Create profile photo download and caching layer

**Dependencies**: Phase 1 (authentication must work first)  
**Estimated Duration**: 4-6 days

---

### Phase 3: Core Messaging
**Goal**: Send and receive messages  
**Deliverables**:
- Message sending (text + media attachments)
- Message receiving with real-time updates
- Delivery status tracking
- Reply and forward functionality

**Key Tasks**:
1. Implement `sendMessage()`, `sendMedia()`, `reply()`, `forward()` operations
2. Set up message receiving via MTProto updates or webhook polling
3. Create message status tracking (pending, sent, delivered, read)
4. Build local message queue for offline operation
5. Implement pagination for large chat histories

**Dependencies**: Phase 1 (connection), Phase 2 (chat resolution)  
**Estimated Duration**: 6-8 days

---

### Phase 4: Chat Management Features
**Goal**: Organize and manage conversations  
**Deliverables**:
- Pin/unpin chats
- Mark as read/unread
- Archive functionality
- Mute notifications per chat

**Key Tasks**:
1. Implement `pinChat()`, `unpinChat()`, `markAsRead()`, `archiveChat()`, `muteChat()`
2. Update local cache when chat state changes
3. Sync state changes back to Telegram server
4. Persist user preferences (pinned, muted) across sessions

**Dependencies**: Phase 2 (chat list), Phase 3 (message context)  
**Estimated Duration**: 3-4 days

---

### Phase 5: Search & Filtering
**Goal**: Enable powerful message search  
**Deliverables**:
- Full-text search across message history
- Filter by sender, date, media type
- Search result highlighting

**Key Tasks**:
1. Build local message index (SQLite full-text search or similar)
2. Implement `searchMessages(query, filters)` API
3. Create pagination for search results
4. Add result highlighting and snippet extraction

**Dependencies**: Phase 3 (message storage), Phase 2 (contact metadata)  
**Estimated Duration**: 4-5 days

---

### Phase 6: Bot Integration & Agent Routing
**Goal**: Support Telegram bot interactions with agent-driven auto-replies  
**Deliverables**:
- Bot discovery by username
- Inline keyboard rendering
- Webhook-based update handling (optional, alternative to polling)
- **Agent routing engine for incoming messages**
- Per-bot agent configuration UI
- Conversation context management per chat

**Key Tasks**:
1. Implement `getBotInfo(username)` to fetch bot profile
2. Parse and render inline keyboards from bot messages
3. Set up webhook listener for bot updates (optional)
4. Create command discovery and help menu generator
5. **Implement agent routing dispatcher**:
   - Parse incoming message to identify target bot
   - Look up configured agent for that bot
   - Assemble conversation context (last N messages from that chat)
   - Invoke the selected agent with the message + context
   - Format and send back the agent's response
6. **Build per-bot configuration UI**:
   - Agent selection dropdown (lists all installed BrowserOS agents)
   - Toggle between "manual mode" and "auto-reply mode"
   - Context depth slider (default 10, max 100 messages)
   - Optional system prompt override field
7. **Implement conversation context cache**:
   - Store last N messages per chat for agent context
   - Eviction policy (LRU or TTL-based)
   - Serialization format for agent consumption

**Dependencies**: Phase 1 (connection), Phase 3 (message handling)  
**Estimated Duration**: 6-8 days (includes agent routing logic)

---

### Phase 7: Notifications & UX Polish
**Goal**: Deliver seamless user experience  
**Deliverables**:
- Desktop notifications for new messages
- Per-chat notification preferences
- Unread count badges
- Offline mode UI indicators

**Key Tasks**:
1. Integrate with BrowserOS notification system
2. Implement notification rules engine (mute, DND, per-chat settings)
3. Add unread count tracking and badge updates
4. Create offline mode detection and queue visualization
5. Polish error messages and loading states

**Dependencies**: All previous phases  
**Estimated Duration**: 3-4 days

---

### Phase 8: Testing & Optimization
**Goal**: Ensure reliability and performance  
**Deliverables**:
- Unit tests for all adapters
- Integration tests with real Telegram API
- Performance benchmarks
- Error handling validation

**Key Tasks**:
1. Write unit tests for adapter methods (mocked Telegram API)
2. Create integration test suite with sandboxed Telegram account
3. Benchmark sync performance (message history up to 10k messages)
4. Test edge cases: network drops, rate limits, large files, **agent timeouts**
5. Optimize cache strategies and memory usage

**Dependencies**: All previous phases  
**Estimated Duration**: 4-6 days

---

## Data Flow Architecture (with Agent Routing)

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│   User (Phone)  │────▶│  Telegram    │────▶│  BrowserOS      │
│   Sends Message │     │  Bot Update  │     │  Poller/Worker  │
└─────────────────┘     └──────────────┘     └─────────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────┐
                                          │ Agent Router │
                                          │ (Dispatcher) │
                                          └──────────────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    ▼                              ▼                              ▼
           ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
           │  Context     │              │   Selected   │              │   Fallback   │
           │  Builder     │              │    Agent     │              │   Handler    │
           │ (Last N msgs)│              │(Assistant/   │              │(Timeout/Error)│
           └──────────────┘              │ Memory/...)  │              └──────────────┘
                    │                     └──────────────┘                        │
                    │                              │                              │
                    └──────────────────────────────┼──────────────────────────────┘
                                                   ▼
                                          ┌──────────────┐
                                          │ Agent        │
                                          │ Invocation   │
                                          │ (with context)│
                                          └──────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────┐
                                          │ Response     │
                                          │ Formatter    │
                                          └──────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────┐
                                          │ Telegram API │
                                          │ sendMessage()│
                                          └──────────────┘
                                                   │
                                                   ▼
                                          ┌──────────────┐
                                          │ User (Phone) │
                                          │ Receives     │
                                          │ Agent Reply  │
                                          └──────────────┘
```

## Technology Choices

### Recommended Stack
- **Adapter Pattern**: Follow BrowserOS integration framework (similar to GSuite integration)
- **Storage**: SQLite for message index + local filesystem for media cache and agent context
- **Connection**: WebSocket-based MTProto client or REST polling for bot API
- **Queue**: Local message queue with retry logic (exponential backoff)
- **Cache**: TTL-based in-memory cache with persistence to disk for agent conversation history
- **Agent Invocation**: Use existing BrowserOS agent execution engine (same as Assistant app)

### Alternatives Considered
- **Full MTProto vs Bot API**: Support both, but prioritize Bot API for simplicity and agent routing
- **Webhook vs Polling**: Default to polling for reliability; offer webhook as advanced option
- **SQLite vs IndexedDB**: SQLite for server-side/Node environment; IndexedDB if browser-only

## Risk Assessment

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Telegram API rate limits | Medium | High | Implement request queuing with backoff |
| Large message history sync | High | Medium | Progressive loading, pagination, background sync |
| Network instability | Medium | High | Offline queue, auto-reconnect, status indicators |
| Media file size limits | Low | Medium | Chunking or compression for files > 20MB |
| Multi-device state conflicts | Medium | Low | Last-write-wins with timestamp reconciliation |
| **Agent timeout** | **High** | **Medium** | **Implement 30s timeout with fallback message** |
| **Context overflow (token limits)** | **Medium** | **High** | **Configurable context depth, intelligent truncation** |

## Milestones & Deliverables

1. **MVP (Phase 1-3)**: Basic send/receive text messages, contact list, bot token auth
2. **Agent-Ready (Phase 1-6)**: Full bot integration with agent routing engine and per-bot configuration
3. **Production Ready (Phase 1-8)**: Full feature set with notifications, search, optimization, agent timeout handling

## Success Metrics (from Spec)

- Authentication < 5 seconds
- Message delivery 99% within 3 seconds
- Search results < 2 seconds for 10k messages
- Memory footprint < 100MB
- API failure rate < 1%
- **Agent response time: 95% of agent-driven responses completed within 10 seconds end-to-end**

## Open Questions

1. Should we support Telegram Premium features (larger files, faster uploads)?
2. Is there a need for custom emoji or sticker pack management?
3. Should the integration expose raw MTProto events for advanced use cases?
4. **How should agent conversation history be persisted across bot restarts?** (Answer: SQLite cache with configurable TTL)
