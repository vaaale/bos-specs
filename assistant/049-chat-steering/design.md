# Technical Design — Chat Steering & Concurrent Conversations (#57)

**Feature Branch**: `bos/049-chat-steering`
**Spec**: `spec.md` (this directory)
**Mockup**: `mockup.html` (this directory)
**App Target**: `bos-core`

## Classification

A change to the **shared assistant run stack** (`src/lib/assistant/`) and the **shared chat UI** (used by the Assistant app, Build Studio, and any other embedder). It is *not* a new app and *not* a marketplace item. It modifies BOS platform source on a feature branch; the Developer builds it via `dev_delegate`.

Two independent halves:
- **P1 — Mid-run steering**: a new delivery `mode` on the existing send funnel + a per-run inbox drained by the agent loop between steps + an `injected_message` run event + an `injected` marker on the persisted message.
- **P2/P3 — Concurrent visibility + cross-window steering**: surface the *already-present* per-conversation run concurrency (live "running" indicator + background finish notification); cross-window steering falls out for free because the inbox and event log are server-side.

---

## Context / Container

The assistant already runs **server-owned, detached agent loops** (one per conversation). `RunManager` (`src/lib/assistant/run-manager.ts`) is the single owner of runs; `runAgentLoop` (`src/lib/assistant/agent-loop.ts`) is the loop that is the *only writer of the transcript*; `POST /api/assistant/runs` (`src/app/api/assistant/runs/route.ts`) is the **single send funnel** (textarea, Enter, edit-resubmit, voice all converge on it). Runs append to a monotonic event log that viewers replay (`?since=0`) and tail as NDJSON — "N tabs are N viewers, never N executors."

The two halves fit the existing container without new infrastructure:

| Concern | Where it lives today | What we add |
|---|---|---|
| Per-conversation active run | `RunManager.byConversation` | — (already per-conversation) |
| Frontend tool-call pending queue | `Run.pendingFrontend` | `Run.inbox` (parallel structure) |
| Loop transcript writes | `runAgentLoop` via `io` | a drain seam that appends injected messages |
| Send funnel + 409 | `POST /api/assistant/runs` + `startAssistantRun` | an optional `mode: "steer"` branch |
| Event vocabulary | `run-events.ts` | `injected_message` event |
| Conversation list | chat-stack panel | live "running" indicator + finish notification |
| Composer enable/disable | `ChatInputV2` | stays enabled mid-run; send uses `mode: "steer"` |

---

## Component design

### 1. Per-run inbox (`run-manager.ts`)

Add to the `Run` interface, parallel to `pendingFrontend`:

```ts
inbox: QueuedInjection[]; // ordered, arrival order
```

where `QueuedInjection = { message: ChatMessage }`. Add manager methods:

- `enqueueInjection(run, message)` — push onto `run.inbox` and `emit(run, { type: "injected_message", message })`. Emitted **at enqueue** so every attached viewer renders the injected message live *immediately* (per FR-008/FR-009 and the mockup), while the transcript append happens later at the drain seam (deterministic order per FR-004).
- `drainInbox(run): ChatMessage[]` — return and clear all pending injected messages in arrival order.

On `finish(run, …)`, the loop (see below) is responsible for flushing any not-yet-drained injections into the transcript before the run is marked finished — so a message injected during a final text-only turn is not lost (FR-010).

**Cross-window (P3)**: the inbox and event log are keyed to the run (server-side), not to a viewer, so a `mode:"steer"` send from any window targets the same `activeFor(conversationId)` run and the `injected_message` broadcast reaches all attached viewers. No extra work.

### 2. Drain seam (`agent-loop.ts`)

`AgentLoopDeps` gains an optional callback, mirroring `awaitFrontendResult`:

```ts
drainInjections?: () => ChatMessage[];  // wired to manager.drainInbox(run)
```

In `runAgentLoop`, drain in two places:

- **Top of each step**, right after `emit({ type: "step_started", step })` (before the model turn): for each drained message, append it to **both** `messages` (the persisted transcript) and `contextMessages` (the model's view), `await io.saveMessages(messages)`. Because the `injected_message` event was already emitted at enqueue, the drain does **not** re-emit a `message` event (that would double-render for live viewers). Place the injection after the last settled tool result of the prior step — appending at the top of the next step does exactly this, so replay reads the same order regardless of when the message arrived (FR-004).
- **On exit** (the `finish` helper, before returning): drain once more so any injection that landed after the last step-start but before the run ended is persisted as a normal queued user message (FR-010; the Copilot steering→queue fallback).

Because the loop is the *only transcript writer*, and it appends the injected message as a real `user` `ChatMessage` with **no wrapper** (Option A, FR-005), the transcript stays deterministic and honest.

### 3. Send funnel + 409 (`route.ts` + `start-run.ts`)

`POST /api/assistant/runs` body gains `mode?: "steer"` (FR-002). `startAssistantRun` gains the same and rewrites its active-run branch:

```
const active = manager.activeFor(conversationId);
if (active) {
  if (mode === "steer" && !editOfMessageId) {
    manager.enqueueInjection(active, userMessage);   // emit injected_message
    return active;                                    // 200/201 { runId: active.id }
  }
  if (!editOfMessageId) throw new ActiveRunError(active.id);   // unchanged 409
  // ... existing edit-resubmit auto-cancel path unchanged
}
// idle → create run as today (a mode:"steer" send with no active run is a graceful fallback)
```

- Plain send (no `mode`) into an active run → **still 409** (FR-007, SC-007).
- `editOfMessageId` → **still** auto-cancel + restart; `mode` is orthogonal to it (FR-007).
- `mode:"steer"` into an idle conversation → starts a normal run (FR-006).
- The route returns the active run's id on the steer-into-active path so the client can attach/render live.

### 4. Event vocabulary (`run-events.ts`)

Add one variant to `RunEventInput`:

```ts
| { type: "injected_message"; message: ChatMessage }
```

### 5. Injected-message marker (`messages.ts` + transcript render)

Add `injected?: boolean` to `ChatMessage`. The loop sets it when persisting a drained injection, so a **fresh conversation reload** (which reads the persisted transcript, not the event log) still renders the "injected (mid-run)" tag. Live viewers render the tag from the `injected_message` event. One rendering path: message has `injected === true` → distinct tag (FR-009).

### 6. Composer stays enabled (`ChatInputV2`)

Today `canSend = !busy && …` disables the composer mid-run. Change: the composer is **always enabled**; when a run is active (`busy`), a send issues `POST /api/assistant/runs` with `mode: "steer"`; when idle, a plain send. Add the subtle "inject while running" affordance (the mockup's hint line). The send path (`sendMessage` in the chat client) picks the `mode` based on run state.

### 7. Conversation list — live running indicator + finish notification

The engine already supports N concurrent runs (FR-011 — no change). The new work is surfacing:

- **Data source.** Add a lightweight `GET /api/assistant/runs?all=1` → `{ activeRuns: [{ runId, conversationId, startedAt }] }`, built from `runManager()`'s active runs (the same scan the existing `?branch=` path already does). The conversation panel polls it at a modest interval (≈2s) and on window focus, mapping `conversationId → running`. (Alternative: a push/SSE channel broadcasting run-start/finish — more work; the poll+diff is the minimal surface that reuses `activeFor`. Flagged as a design option, not decided.)
- **Running dot.** Each conversation row shows a live indicator when its `conversationId` is in the active set (FR-012); cleared when it drops out (i.e. on `run_finished`).
- **Finish notification.** The panel diffs the active set across polls: a conversation that was running and is now idle has finished. If it is **not** the currently-viewed conversation, show a finish cue (toast / sidebar dot identifying the conversation) — FR-013; suppressed for the viewed conversation (edge case). This is derived, not event-driven, which is what makes it work for runs the viewer never attached to.

---

## File / module plan (what this feature creates or modifies)

**Creates**
- (none as new files) — all changes are edits to existing modules + one new API query param.

**Modifies**
- `src/lib/assistant/run-manager.ts` — `Run.inbox`, `enqueueInjection`, `drainInbox`.
- `src/lib/assistant/agent-loop.ts` — `AgentLoopDeps.drainInjections`; drain at step-start and at finish.
- `src/lib/assistant/start-run.ts` — `mode: "steer"` handling in the active-run branch.
- `src/app/api/assistant/runs/route.ts` — accept `mode` in the POST body; add `?all=1` to GET.
- `src/lib/assistant/run-events.ts` — `injected_message` event variant.
- `src/lib/assistant/messages.ts` — `ChatMessage.injected?` marker.
- Shared chat UI (chat stack): composer (`ChatInputV2`) enable-while-running + steer send; message renderer (`injected` tag); conversation panel (running indicator + finish notification).
- Client run attach (e.g. `run-client.ts`) — handle the steer-into-active response (attach to the returned active run; render the `injected_message` event).

**Tests** (per BOS convention — Playwright runner, `tests/<area>/`, NOT co-located `__tests__`):
- `tests/assistant/steering.test.ts` — loop drains inbox between steps; injection is a plain user message, no wrapper; deterministic order; run-finish flush; plain send still 409; edit-resubmit unchanged; steer-into-idle starts a run.
- `tests/assistant/concurrent-runs.test.ts` — N conversations run concurrently, lossless switch; `?all=1` reflects active runs.
- (UI) render of the `injected` tag and the running indicator.

---

## Integration points (existing BOS mechanisms this calls into, not modifies)

- `RunManager` singleton (`globalThis.__bosRunManager`) — reused for run ownership and the `?branch=` active-run scan.
- `runAgentLoop` / `conversationIO` — the loop remains the sole transcript writer; we only add a drain seam.
- `ActiveRunError` → `409` contract — preserved for the default send path.
- `attachToRun` / `?since=0` lossless replay — reused for cross-window and background-run rendering.
- `pendingFrontend` pattern — the inbox mirrors it (additive-only, settles on finish).
- Shared chat component used by Assistant + Build Studio + other embedders — the UI changes are made once, in the shared stack, so all embedders inherit them.

---

## ADRs

**ADR-1 — Reuse the send funnel, not a new endpoint.**
A dedicated `POST …/inject` endpoint would split the single send funnel and duplicate auth/archive/validation. Instead, one optional `mode: "steer"` on the existing endpoint preserves one authoritative gate (archived-conversation check, validation, run ownership) and keeps the default 409 behaviour intact. *Decision taken in triage (Alex: "the logic in the existing endpoint should handle it").*

**ADR-2 — Inject as a plain user message at the next turn boundary (Option A), no wrapper.**
A synthetic marker (Option B) degrades the transcript into a non-user message and risks the model parsing the wrapper as instructions. Every surveyed steering implementation (GitHub Copilot SDK, OpenAI mid-turn steering, pi-llm-agent, Claude Code/Cursor) delivers a plain user message at the next turn boundary. *Locked in triage.*

**ADR-3 — Next-turn delivery, no mid-generation abort (v1).**
Aborting a streaming provider turn to splice in context requires a turn restart and complicates the loop's "a stop here discards the partial turn" invariant. Next-boundary delivery is the mainstream convention and satisfies the "additional information, not a new task" requirement. An interrupt-current-turn mode is an explicit out-of-scope extension. *Locked in triage (Q1: "same as OpenAI, GitHub, etc").*

**ADR-4 — Emit `injected_message` at enqueue; persist at drain.**
Emitting at enqueue gives immediate live render across all viewers (including a second window) at zero latency; persisting at the drain seam keeps transcript order deterministic (after the last settled tool result). The drain does not re-emit, avoiding double render. *New decision — flagged for review.*

**ADR-5 — Poll+diff for the running indicator / finish notification.**
A lightweight `?all=1` active-run scan polled by the conversation panel is the minimal surface that reuses `activeFor` and correctly detects background completions a viewer never attached to. A push/SSE channel is the upgrade path if polling cadence proves too coarse. *New decision — flagged for review; Alex can prefer SSE if he wants tighter timing.*

---

## Risks

- **Race: drain vs finish.** If the run finishes in the same tick an injection is enqueued, the finish-path drain must still flush it. Mitigated by draining in the `finish` helper before the run is marked finished; test the "inject during final text-only turn" case (FR-010).
- **Poll cadence vs perceived latency** for the finish notification (ADR-5). A 2s poll means up to ~2s of notification lag. Acceptable for a "background run finished" cue; tighten or move to SSE if it feels stale.
- **UI change surface.** The composer/message/panel changes live in the *shared* chat stack, so they touch Assistant, Build Studio, and any embedder at once — regression risk is broader. Mitigate with the shared-stack tests above and verify in both Assistant and Build Studio.
- **Double-render.** If the drain re-emits a `message` event on top of the `injected_message` event, live viewers render the injection twice. The design forbids the re-emit; covered by a render test.
- **Determinism.** Injections must not reorder tool results. Appending at the top of the next step (after the prior step's settled tools) preserves order; the deterministic-order test guards this.
