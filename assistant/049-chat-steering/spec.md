# Feature Specification: Chat Steering & Concurrent Conversations

**Feature Branch**: `bos/049-chat-steering`

**Created**: 2026-10-07

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Two improvements to how the chat works, primarily the assistant app but also Build Studio and any other app that uses the chat component. (1) The ability to 'inject' a user message while the agent is processing/generating the previous user message, treated as additional information, not a new task (e.g. 'try searching for X instead of Y'). (2) The ability to have multiple conversations 'running' at the same time — start one, switch to another, come back and forth without killing or truncating them."

---

## Problem Summary

The shared chat stack (Assistant app, Build Studio, any embedder of the chat component) has two gaps:

1. **Mid-run steering.** A user cannot add a course-correction to a conversation while the agent is still generating. Today the composer is disabled during a run, and the single send funnel (`POST /api/assistant/runs`) returns `409` for a plain send into an active run. There is no seam to fold a fresh user message into a loop that is already mid-turn, so an in-flight correction is either dropped or, once the run finishes, misread as a brand-new turn.

2. **Visibility of concurrent runs.** The run engine *already* supports N independent, detached, server-owned runs (one per conversation) with lossless switching — `RunManager` keys runs by conversation and only rejects a second run for the *same* conversation. What is missing is surfacing that: there is no indication in the conversation list that a run is live in the background, and no notification when a background run you navigated away from completes.

**Decisions locked in during triage (see issue #57 thread):**
- The injected message is delivered as a **plain `user` message at the next turn boundary** — no synthetic text wrapper. This matches the industry steering convention (GitHub Copilot SDK, OpenAI mid-turn steering, pi-llm-agent, Claude Code/Cursor).
- Delivery is **next-turn-boundary** (no mid-generation abort in v1).
- A distinct **"Steering" UI tag** is required so the message reads as additional info, not a new task.
- **Cross-window** steering is required (an injection reaches a conversation running in another window).
- A **finish notification** for background conversations is required.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Steer an in-flight conversation (Priority: P1)

While an agent is generating a long answer or running a tool chain, the user types a course-correction — "try searching for X instead of Y" — and presses Enter. The message appears in the transcript immediately, visibly tagged as an injected mid-run message. The agent folds it into its current task at the next model turn and does **not** restart the conversation.

**Why this priority**: This is the headline request. Without it the feature does not exist. It is a viable MVP on its own — a user can steer one conversation and immediately feel the chat is more responsive.

**Independent Test**: Start a conversation that produces a long, multi-step run. Mid-run, type a correction and send. Assert (a) the message renders immediately with the injected tag, and (b) the agent's next model turn incorporates the correction (verifiable in the transcript: a subsequent step reflects "X" rather than "Y") while the run continues without a restart.

**Acceptance Scenarios**:

1. **Given** a run is streaming a long answer, **When** I type "try searching for X instead" and press Enter, **Then** the message renders immediately and is tagged "Steering", and **Then** the agent's next model turn incorporates it and the run does not restart.
2. **Given** no run is active, **When** I type a message and press Enter, **Then** it behaves exactly as today — a fresh run is started (the steer path is a no-op fallback to normal send).
3. **Given** a run is active and two steering messages are sent within the same step window, **When** the next model turn is reached, **Then** both messages are appended in arrival order and the turn sees both before continuing.
4. **Given** a run is active and I send a steering message **while a long tool call is executing**, **When** the tool call settles and the next model turn begins, **Then** the steering message is present in the transcript at that boundary (never inserted mid-tool-call).
5. **Given** a run is streaming and I send a plain (non-steer) message with no `mode` set, **When** the request reaches the funnel, **Then** it returns `409` exactly as today (the default is unchanged).

---

### User Story 2 — Run several conversations at once, with visibility (Priority: P2)

The user starts a conversation doing a long research task, switches to another conversation, and starts a second task. Both runs proceed independently in the background. The conversation list shows a live "running" indicator on each conversation that has an active run. When a background run finishes, the user is notified even though they are looking at a different conversation.

**Why this priority**: The engine already does the hard part (concurrent, detached, losslessly-switchable runs); this story adds the *surfacing* — the status indicator and finish notification — which is what makes the capability discoverable and useful. It is independently valuable: even a single user benefits from knowing a background run is alive and when it completes.

**Independent Test**: Start run A in conversation 1. Switch to conversation 2 and start run B. Assert both runs continue (neither is killed or truncated), the conversation list shows a live indicator on both, and when A finishes a notification fires while the user is in conversation 2.

**Acceptance Scenarios**:

1. **Given** a run is active in conversation 1, **When** I switch to conversation 2 and send a new message there, **Then** a second independent run starts in conversation 2 and the conversation 1 run continues unchanged (no kill, no truncation).
2. **Given** conversation 1 has an active run, **When** I look at the conversation list, **Then** conversation 1 shows a live "running" indicator, and it is cleared when that run finishes.
3. **Given** I am viewing conversation 2 and conversation 1's background run completes, **When** the completion event fires, **Then** I receive a finish notification (toast or sidebar cue) identifying conversation 1 as finished.
4. **Given** I switch back to conversation 1 after its run finished while I was away, **When** I open it, **Then** the full transcript including the completed run's output is present (lossless switch — already true today, must remain true).

---

### User Story 3 — Steer a conversation running in another window (Priority: P3)

The user has conversation 1 running in one window (or tab) and opens it in another. From the second window the user types a steering correction into conversation 1. The injection is delivered to the running conversation and rendered live in both windows.

**Why this priority**: Cross-window steering falls out of the server-side inbox + event-log design of P1, but it is an explicit requirement and adds a distinct testable surface (broadcast across viewers). It is lower priority because a single-window user gets full value from P1; P3 extends that to multi-window.

**Independent Test**: Open the same conversation in two windows with one run active. Send a steering message from window 2. Assert the injected message appears in both windows and the running conversation folds it into the next turn.

**Acceptance Scenarios**:

1. **Given** a run is active in conversation 1 and it is open in two windows, **When** I send a steering message from window 2, **Then** the injected message is delivered to the active run and rendered live in both windows.
2. **Given** a run is active in conversation 1, **When** I send a steering message from a window that is not the one that started the run, **Then** the run does not error and the message is enqueued to that run's inbox (the inbox is server-side, keyed to the run, not to a viewer).

---

### Edge Cases

- **Run finishes before the next turn boundary drains the inbox**: the steering message is not lost — it becomes a normal queued user message in the transcript and the run ends. This mirrors GitHub Copilot's steering→queue fallback and keeps the transcript honest.
- **Steering with attachments**: a steering message may carry attachments; they are persisted with the injected user message and available at the next model turn, same as a normal message.
- **Rapid successive steering messages**: all messages arriving within one step window are drained in arrival order at the next boundary; the transcript order is deterministic regardless of timing.
- **Steering into a conversation whose run has already errored/stopped**: behaves as a normal send — starts a fresh run (graceful fallback).
- **Two concurrent runs in the *same* conversation**: still rejected as today (`409`) — concurrency is per-conversation, never two live runs in one conversation.
- **Finish notification when the user is actively viewing that conversation**: suppressed (the user can already see it complete); notification is for background (non-viewed) runs.

---

## Requirements *(mandatory)*

### Functional Requirements

**Mid-run steering (P1)**

- **FR-001**: The chat composer MUST remain enabled while a run is active, so a user can send a message into an in-flight conversation.
- **FR-002**: `POST /api/assistant/runs` MUST accept an optional delivery `mode` field with value `"steer"` (no new endpoint). The request body is `{ conversationId, agentId, message, attachments?, mode?: "steer" }`.
- **FR-003**: When a conversation has an active run and the request carries `mode: "steer"`, the system MUST enqueue the message to that run's per-run inbox instead of returning `409`, and the run MUST continue.
- **FR-004**: The `RunManager` MUST maintain a small per-run inbox (mirroring the existing `pendingFrontend` mechanism) that is drained by the agent loop **between steps**, appending each injected message as a persisted `user` message **before the next model turn**, placed after the last settled tool result so replay is deterministic.
- **FR-005**: An injected message MUST be delivered as a **plain `user` message** with **no synthetic text wrapper**. It MUST NOT abort an in-flight generation or a running tool call; it MUST fold into the ongoing task at the next model turn boundary.
- **FR-006**: When `mode: "steer"` is sent but the conversation is idle (no active run), the system MUST fall back to starting a normal run (graceful no-op of the steer path).
- **FR-007**: A plain send (no `mode`) into an active run MUST continue to return `409`, and `editOfMessageId` (edit-resubmit) MUST continue to auto-cancel + restart. The `mode` field MUST be orthogonal to `editOfMessageId`.
- **FR-008**: On each injection, the system MUST emit an `injected_message` run event (carrying the message) so all attached viewers render it live.
- **FR-009**: Injected mid-run messages MUST be rendered in the transcript with a distinct **"Steering" tag** so they read as additional information, not a new task.
- **FR-010**: If a run finishes before the next model turn drains the inbox, pending injected messages MUST be persisted to the transcript as normal queued user messages (not dropped).

**Concurrent-conversation visibility (P2)**

- **FR-011**: The run engine MUST continue to support N independent, detached, server-owned runs (one per conversation) with lossless switching. No run may be killed or truncated by switching away from its conversation.
- **FR-012**: The conversation list MUST show a live "running" indicator on each conversation that has an active run, driven by run status (e.g. an active-run query per conversation), and MUST clear it when that run finishes (`run_finished`).
- **FR-013**: When a background run (one the user has navigated away from) completes, the system MUST present a finish notification (toast or sidebar cue) identifying the finished conversation. Notifications MUST be suppressed for the conversation the user is actively viewing.

**Cross-window steering (P3)**

- **FR-014**: Because the run inbox and event log are server-side (keyed to the run, not to a viewer), a steering message sent from one window MUST reach a conversation running in another window, and the `injected_message` event MUST broadcast so all attached viewers render it live.

### Key Entities

- **Run (per-conversation)**: the existing server-owned execution. Gains a **per-run inbox** (ordered queue of pending injected messages) and continues to own the monotonic run event log (now including `injected_message` events).
- **Injected (steering) message**: a persisted `user` message with a distinguishing marker that renders a "Steering" tag in the transcript, enqueued to a run's inbox and drained at the next turn boundary.
- **Conversation "running" state**: a derived/live indicator (from active-run status) surfaced per conversation in the list, cleared on `run_finished`.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can send a steering message during an active run and see it appear in the transcript in under 1 second, tagged as injected.
- **SC-002**: 100% of steering messages sent into an active run are either (a) folded into the run's next model turn, or (b) persisted as a normal queued user message on run completion — none are silently dropped.
- **SC-003**: A steering message changes the agent's subsequent behaviour in the same run (verifiable: a later step reflects the correction) without a run restart, in the test scenario.
- **SC-004**: A user can run at least two conversations simultaneously; switching between them leaves both transcripts fully intact (no truncation, no kill), confirmed by round-trip.
- **SC-005**: A background run that finishes while the user is in another conversation produces exactly one finish notification.
- **SC-006**: A steering message sent into a conversation open in a second window is delivered and rendered live in both windows.
- **SC-007**: Default send behaviour is unchanged: a plain send into an active run still returns `409`, and edit-resubmit still auto-cancel + restarts.

---

## Assumptions

- The single send funnel `POST /api/assistant/runs` remains the transport for steering; no new endpoint is introduced (locked decision from triage).
- Steering is delivered at the next model-turn boundary (not a mid-generation abort) in v1, matching the OpenAI / GitHub Copilot convention (locked decision).
- The injected message is a plain `user` message with no synthetic prompt wrapper (Option A, locked decision) — the mid-stream position itself conveys "additional context."
- The existing `RunManager` per-conversation keying, `pendingFrontend` mechanism, and `attachToRun`/replay-with-`?since=0` lossless-switch behaviour are reused, not re-implemented.
- The finish notification is a lightweight cue (toast or sidebar) rather than a full re-architecture; the exact form is a design decision.
- Out of scope: cancelling or reordering already-executing tool calls; editing/undoing an injected message after it is persisted; a true side-by-side dual-pane chat view; cross-conversation *state* sharing (runs remain independent).
