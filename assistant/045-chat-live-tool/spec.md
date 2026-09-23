# Feature Specification: Live Per-Tool Updates & Rich Tool-Call Rendering in Chat

**Feature Branch**: `bos/045-chat-live-tool`

**Created**: 2026-09-23

**Status**: Specified — design complete (reviewed)

**App Target**: bos-core

**Input**: User description: "When the agent runs multiple tool calls in parallel, the tool outputs are not marked as completed until ALL tool calls finish, and the outputs are not updated live while multiple tools are executing. I want each tool call's output to update live, individually — e.g. if an agent runs 4 subagents in parallel, the output from each subagent should update live and its status should flip to completed as soon as that subagent finishes."

Second request (added while authoring): "Also, the tool call input and output are currently rendered as raw/pretty JSON. Render them properly the way Claude / ChatGPT do — a human-readable action summary in the card, the arguments shown as structured data, and the result rendered as its natural content type (markdown, syntax-highlighted code, or formatted JSON) rather than a text/JSON blob."

## Context: what exists today

The v2 assistant chat is driven entirely by a per-call run-event stream (NDJSON/SSE), not by
CopilotKit message state. Each tool invocation is a first-class unit with its own identity and
status, and the client is already fully capable of updating them independently:

- **Per-call identity & status.** Every tool call has a distinct `callId`. The client store
  (`src/lib/assistant/client/chat-store.ts`) holds `toolCalls: Record<callId, ToolCallView>` where
  `ToolCallView.status ∈ { running, done, cancelled }`. On `tool_call` a call is `running`; on
  its own `tool_result` it is `done`; on `tool_cancelled` it is `cancelled`. The live card
  (`src/components/agent/v2/ToolCallCard.tsx`) flips purely off its own call's `status`.
- **Per-call live channel.** A `tool_progress` run event carries `{ callId, event }`; the store
  appends it to that call's `progress[]`, and the card renders it live while the call is running.
  This channel already re-arms the call's idle timeout, so long-running work never times out.
- **Parallel execution is deliberate.** `agent_delegate` is wrapped in `parallel(...)` and marked
  `parallelSafe`; adjacent parallel-safe calls in a turn are grouped into one batch
  (`batchToolCalls`) and run concurrently.

Given all of the above, the two reported defects are both **upstream of the client**, and the fix
is to reuse the existing per-call machinery rather than add new event types or protocol:

1. **Batch-gated completion (defect a).** In `src/lib/assistant/agent-loop.ts`, the whole parallel
   batch is awaited as a unit (`await Promise.all(batch.map(executeCall))`) and each call's
   `tool_result` is emitted in a *trailing* loop after the **slowest** call settles. The three
   fast subagents finish server-side long before, but their `tool_result` events — and therefore
   their card flips — are held until the fourth lands, then all four fire in one burst. The client
   is ready to flip each card the instant its own `tool_result` arrives; it simply never gets them
   individually today.

   The trailing loop deliberately **persists tool results in the original call order, never
   completion order**, so the persisted transcript is deterministic across runs of the same input.
   That invariant is correct and must be preserved — the fix must **decouple the order in which
   live `tool_result` events are emitted (completion order, drives the UI) from the order in which
   results are persisted (original call order, drives the transcript)**. Today those two concerns
   are fused in one loop.

2. **No live output for delegations (defect b).** For a subagent delegation, the inner loop
   (`src/lib/assistant/inner-loop.ts`) reshapes the inner `tool_call` events into nested *starts*
   and forwards them via the per-call `tool_progress` channel — so the parent card already shows a
   live list of *which nested tools have started*. But it **swallows the inner loop's
   `tool_result`/`tool_cancelled`**, and the card's live branch renders `output=""`
   (`src/components/agent/v2/ToolCallCard.tsx`). Net effect: a delegation streams a list of starts
   but shows **no nested results and no output until the whole delegation returns** — and that
   return is itself batch-gated by defect (a).

3. **Flat, opaque rendering (defect c).** The card
   (`src/components/agent/v2/ToolCallCard.tsx`) is a single-level expand/collapse: the collapsed
   header shows only the raw tool name + status; expanding it reveals a **Request** section
   (pretty-printed JSON dump via `pretty(call.args)`) and a **Response** section (a plain `<pre>`
   text blob via `resultText`). There is no independent expand/collapse on Input vs. Output,
   no human-readable action summary, no natural content-type rendering of the result (markdown /
   code / JSON), and no syntax highlighting. The only special cases are the MCP-UI iframe and the
   nested-delegation list — and even that list is a flat `<ul>` of tool-name + input rows, not
   recursively expandable cards. So a delegation's nested work is a flat list until the terminal
   `tool_result`, not a navigable tree; and a file read or web search shows as an unformatted
   text/JSON wall rather than the readable, content-aware rendering users expect.

### New vs. existing

- **Existing, reused unchanged:** per-call `ToolCallView` + `applyRunEvent` state machine; the
  `tool_progress` run event and per-call `progress[]`; `ToolContext.onEvent` as the per-call
  progress conduit; `encodeNested`/`parseNested` for serializing sub-agent events into a result;
  the NDJSON/SSE transport and the per-call `tool_result` event shape; the chat's existing
  markdown renderer (reused to render tool output that is text/markdown); and the existing
  MCP-UI iframe and nested-delegation renderers (preserved, taking precedence).
- **New (what this feature adds):** per-settle live `tool_result` emission decoupled from
  original-order persistence in the run loop; forwarding of nested *results* (not just nested
  *starts*) through the `tool_progress` channel for delegations; and a reworked card that is a
  **recursive, single-component collapsible tree**: a human-readable action summary in the header;
  independently expandable **Input** (structured key–value, primary arg highlighted, raw-JSON
  toggle) and **Output** (content-type rendered: markdown / highlighted JSON / highlighted code /
  child tool-call cards) sections; and for delegations, the Output section is a list of child
  tool-call cards — the same component, recursed at every depth. The mockup at
  `mockup.html` in this spec directory is the binding UI contract.
- **Not introduced:** no new run-event type, no transport/protocol change, no client-store change,
  no CopilotKit change.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Each parallel tool call completes the moment it finishes (Priority: P1)

A user watching the agent run several independent tool calls at once (e.g. four subagents) wants
each call's card to flip to "done" individually the instant that call finishes, instead of all of
them appearing to complete together at the end. Today the three fast calls sit as "running" even
though they have already returned, and the whole row flips at once when the slowest call lands.

**Why this priority**: This is the headline defect and the single most visible problem. It is a
self-contained backend fix (per-settle emission) with no client change required — the card already
flips on its own `tool_result` — so it delivers the biggest win with the least risk.

**Independent Test**: Trigger one agent turn that issues a batch of ≥2 parallel-safe tool calls
with deliberately staggered completion times (e.g. multiple `agent_delegate` subagents of
differing length, or a mix of a fast tool and a slow tool). Observe that each card transitions
running→done at its *own* settle time, with no card waiting for the batch's slowest call.

**Acceptance Scenarios**:

1. **Given** a batch of four parallel-safe calls where calls A and B finish before C and D,
   **When** A settles, **Then** A's card flips to "done" immediately, while B, C and D remain
   "running" — A does not wait for C or D.
2. **Given** the same batch, **When** all four have settled, **Then** every card is "done", each
   having flipped at its own settle time rather than in a single simultaneous burst.
3. **Given** the persisted transcript for the run, **When** it is read (live or on reload),
   **Then** the tool-result messages appear in the **original call order**, independent of the
   order in which the calls actually completed.

---

### User Story 2 - A running tool call shows its output live, as it happens (Priority: P1)

A user watching a subagent run wants to see that subagent's work stream into its card *while it is
running* — its nested tool results appearing as each one completes — rather than a static list of
starts followed by a single all-at-once dump of output when the delegation returns.

**Why this priority**: This is the second half of the request ("output should update live"). It is
a wiring change on the existing `tool_progress` channel (forward nested results, render them), and
it is what makes long-running delegations feel alive.

**Independent Test**: Delegate to a local/ephemeral subagent whose inner run calls several tools.
Watch the parent card while the subagent is still running: confirm the card's live progress list
shows each nested tool's *result* appearing as it completes (not only the nested tool's start),
and that the full output is not merely appended once at the very end.

**Acceptance Scenarios**:

1. **Given** a running subagent delegation that has completed 2 of 5 nested tools,
   **When** the card renders while still "running", **Then** its live progress shows the results of
   those 2 completed nested tools, plus the (still-pending) starts of the remainder.
2. **Given** the same delegation, **When** a nested tool completes, **Then** its result appears in
   the parent card's live list at that time without waiting for the delegation to return.
3. **Given** a delegation that finishes, **When** its card flips to "done", **Then** the card shows
   the full nested event list and the delegation's final output.

---

### User Story 3 - Recursive, navigable tree for tool call input and output (Priority: P2)

A user reviewing a tool call wants a **recursive, collapsible tree** — not a flat card with two
flat sections. The tool call header is expanded (as today); inside it, **Input** and **Output**
are each independently expandable sections. For a delegation, the Output section contains child
tool-call cards — the *same component*, recursed at every depth — so the user can drill into a
subagent's nested work the same way they would into any other tool call. The binding UI contract
is the mockup at `mockup.html` in this spec directory.

**Why this priority**: It is the second half of the user's request and is presentational rather
than behavioral — it does not change *when* or *whether* results arrive (US1/US2), only *how*
they are displayed. It builds on the card that US1/US2 already drive, so it lands as P2 behind
them, yet it is independently testable: any completed tool call can be checked for tree structure
without involving the live-update mechanism.

**Independent Test**: Open the mockup and click through: expand a done tool call, expand its
Input and Output independently, confirm Input shows structured key–value (not raw JSON), Output
renders by content type (markdown / JSON / code). Then expand a delegation's Output: confirm
child tool-call cards appear, and expand one of those children to confirm it has its own
Input/Output — the same structure recursed.

**Acceptance Scenarios**:

1. **Given** a completed tool call, **When** its card header is collapsed, **Then** it shows a
   human-readable one-line action summary (verb + key argument, e.g. "Read file /path", "Web
   search for …", "Delegate to <agent>"), not just the raw tool name. The tool name is the
   fallback when no per-tool summary is available.

2. **Given** a tool call with its header expanded, **When** the Input section is expanded,
   **Then** the arguments render as structured key–value rows (primary argument — path, query,
   url, id, command — highlighted as a distinct labeled value). A "raw" toggle switches the
   Input section to the full pretty-printed JSON dump.

3. **Given** a tool call with its header expanded, **When** the Output section is expanded,
   **Then** the result renders by content type in this precedence: (1) MCP-UI marker → existing
   iframe renderer; (2) nested-delegation marker → list of child tool-call cards (see scenario 5);
   (3) valid JSON object/array → formatted, syntax-highlighted JSON with a copy button; (4)
   otherwise → markdown (headings, lists, tables, inline and fenced code) using the chat's
   existing markdown renderer, with code blocks highlighted and a copy button.

4. **Given** the Input and Output sections are both visible inside an expanded card, **When** the
   user clicks the Input section header, **Then** only Input collapses/expands — Output is
   unaffected, and vice versa.

5. **Given** a delegation tool call, **When** its Output section is expanded, **Then** it shows
   a list of child tool-call cards — each a full card with its own header (action summary +
   status dot), its own independently expandable Input and Output, and its own content-type
   rendering. The nesting is visually indicated by an indent and left border. This recurses:
   a child that is itself a delegation shows its own child cards in its Output.

6. **Given** a tool call with no output (empty/null result), **When** its card is expanded,
   **Then** no Output section is shown (the Input section is unaffected).

---

### Edge Cases

- **Single-call batch.** A parallel-safe batch of one call degrades to current behavior: its
  `tool_result` is emitted on its own settle. No regression.
- **Mixed parallel and sequential in one turn.** A non-parallel-safe call (e.g. `dev_delegate`)
  is never grouped into a batch and already completes individually; the fix MUST NOT change its
  behavior. A parallel-safe batch followed by a sequential call keeps both correct.
- **Aborted/cancelled batch.** If the run is cancelled mid-batch, each still-running call flips to
  "cancelled" individually and promptly — not held until the batch would otherwise end.
- **A call that errors.** A `executeCall` that rejects emits its own result/error for that call
  individually and MUST NOT hold the remaining calls' `tool_result` emissions.
- **Replay determinism.** Replaying a completed run (events `?since=`) and reloading a saved
  conversation MUST present tool-result messages in the original call order. Two runs of the same
  input MUST produce the same persisted transcript order even if live completion order differs.
- **Non-delegation tools that never emit progress.** Tools that do not call the progress conduit
  continue to show nothing until their (now individual) `tool_result`; this feature does not
  mandate new progress emission from every tool — it only makes whatever is emitted appear live and
  per-call.
- **Claude-harness delegations.** The Claude/OpenCode CLI harness reports its final text once at the
  end; US2's live output for such delegations covers streaming the nested tool activity/results the
  harness does emit, and does not require token-by-token streaming of the harness's final prose.
- **Empty result.** A call with no output shows no Response section (current behavior);
  content-type detection MUST NOT invent content for an empty/null result.
- **Large result.** Very large outputs MUST remain performant inside the card's existing scroll
  clamp; markdown/JSON rendering MUST NOT remove the scroll/max-height behavior or render an
  unbounded document.
- **Ambiguous content.** A string that looks like JSON but is prose (or vice versa) is resolved
  best-effort; when ambiguous, prefer the markdown/text rendering so display degrades gracefully
  rather than throwing.
- **Content that resembles markup/code.** A result containing raw `<`, `>`, `&`, backticks, or
  markdown-like syntax MUST render safely — code-like content shows as code (highlighted), not as
  mangled or executed markup.
- **Args with no clear primary.** A tool whose arguments have no single "thing it acts on" shows a
  clean key–value list rather than a forced/fabricated primary argument.
- **Running delegation, nested cards.** A delegation still running shows its child cards live as
  each nested tool completes (US2); each child card is a full recursive card (US3). A child that
  is still running shows its own (empty or in-progress) Output with a spinner — the recursion
  holds in the live state, not only after completion.
- **Unbounded recursion.** There is no hard depth cap on nested cards; each level is the same
  component. The per-section scroll clamp bounds the rendered height at any depth so a very deep
  or very wide tree remains navigable.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The run loop MUST emit each parallel-safe call's `tool_result` event **as that call
  settles** (completion order), so the client receives a distinct `tool_result` per `callId` at the
  moment that call finishes, rather than a single batched burst after all calls in the batch settle.

- **FR-002**: Persistence of tool results MUST remain in the **original call order** (the order the
  calls were issued), and MUST be independent of the order in which the calls completed. Live
  emission order and persisted order are decoupled: the live event stream may follow completion
  order while the persisted transcript follows original call order.

- **FR-003**: A running tool call's card MUST reflect its own status individually: it MUST flip
  running→done on receipt of **its own** `tool_result` and MUST NOT wait for any other call in the
  same batch to complete.

- **FR-004**: A cancelled or aborted call in a running batch MUST flip to "cancelled" individually
  and promptly, without being held for the rest of the batch.

- **FR-005**: For a subagent delegation, the inner run MUST forward each nested tool **result** (and
  nested cancellation) through the per-call `tool_progress` channel, so the parent streams nested
  completions live — not only nested tool *starts* as it does today.

- **FR-006**: While a delegated call is still running, its card MUST render the live nested
  progress (including completed nested results as they arrive), not only nested tool names/inputs.
  On completion the card MUST render the full nested list plus the delegation's final output.

- **FR-007**: The fix MUST reuse the existing per-call event vocabulary (`tool_call`,
  `tool_progress`, `tool_result`, `tool_cancelled`), the per-call client store, and the
  `encodeNested`/`parseNested` serialization. It MUST NOT introduce a new run-event type, a
  transport/protocol change, a client-store schema change, or a CopilotKit dependency.

- **FR-008**: A tool call that errors MUST emit its own result/error individually and MUST NOT delay
  the `tool_result` emissions of other calls in the same batch.

- **FR-009**: Replaying a completed run and reloading a saved conversation MUST present tool-result
  messages in original call order. Two runs of the same input MUST yield the same persisted
  transcript order regardless of live completion timing.

- **FR-010**: This behavior MUST hold for every surface that renders the v2 assistant chat
  (standalone Assistant and any embed), since they share the same run-event stream and
  `ToolCallCard`.

- **FR-011**: The tool call card MUST be a single recursive component, used identically at every
  nesting depth. Each card has a collapsed header (action summary + status dot) that expands to
  reveal two independently collapsible sections: **Input** and **Output**. For a delegation, the
  Output section is a list of child cards — the same component, recursed. The card MUST NOT use
  a separate flat list component for nested delegation results; the nesting is structural (the
  same component rendered inside the parent's Output), not representational.

- **FR-012**: Each card's collapsed header MUST show a human-readable, one-line action summary
  derived from the tool name and its key argument (e.g. "Read file /path", "Web search for …",
  "Delegate to <agent>"). A tool with no recognized summarizer MUST fall back to the raw tool
  name. The header MUST NOT display a raw JSON argument blob.

- **FR-013**: The Input section (inside an expanded card) MUST render the tool's arguments as
  structured key–value rows: the primary argument (path, query, URL, id, command) as a distinct
  labeled row with visual emphasis; remaining arguments as plain key–value rows. A "raw" toggle
  MUST switch the Input section to the full pretty-printed JSON dump. Both views MUST be
  available; the structured view is the default.

- **FR-014**: The Output section (inside an expanded card) MUST render the tool's result by
  content type, in this precedence: (1) MCP-UI marker → existing iframe renderer; (2)
  nested-delegation marker → list of child tool-call cards (FR-011); (3) result that parses as
  a JSON object/array → formatted, syntax-highlighted JSON with a copy button; (4) otherwise →
  markdown (headings, lists, tables, inline and fenced code) using the chat's existing markdown
  renderer, with code blocks highlighted and a copy button. The content-type mode MUST be
  visually indicated (e.g. a small label in the Output section header: "markdown", "json",
  "code").

- **FR-015**: The Input and Output sections MUST be independently expandable: clicking the Input
  section header MUST toggle only Input (Output unaffected), and vice versa. Both MUST default
  to collapsed when the card is first expanded.

- **FR-016**: The input/output rendering in FR-013/FR-014 MUST also apply to the live nested
  results streamed by US2 (FR-005/FR-006), so that a delegated call's child cards render their
  outputs as their natural content type while the delegation is still running — not as raw JSON.

### Key Entities

- **Tool call (`callId`)**: the identity of one tool invocation; the unit of live status, live
  output, and individual completion.
- **Parallel batch**: a group of adjacent parallel-safe calls in one turn, executed concurrently.
- **Live progress event (`tool_progress`)**: an incremental, per-`callId` partial-output event on
  the existing streaming channel.
- **Live completion order vs. persisted (original call) order**: the two orderings this feature
  keeps separate — the former drives the UI, the latter drives the deterministic transcript.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In a batch of N≥2 parallel-safe calls with staggered completion, 0 cards wait for the
  batch's slowest call — each card transitions running→done at its own `tool_result` receipt.

- **SC-002**: Persisted transcript order is invariant to completion order: two runs of identical
  input produce byte-identical tool-result message ordering (0 order differences), even when live
  completion order differs between the runs.

- **SC-003**: For a subagent delegation, 100% of its nested tool results appear in the parent card's
  live progress as each completes while the delegation is still running (not only in the terminal
  `tool_result`).

- **SC-004**: On mid-batch cancellation, each still-running call flips to "cancelled" within one
  event round-trip of the cancel, individually (0 calls held for the batch's remainder).

- **SC-005**: No new run-event type, transport, or client-store schema is introduced — verified by
  a change surface that touches only run-loop emission ordering, delegation inner-loop forwarding,
  and the live card render.

- **SC-006**: 100% of completed tool-call cards show a human-readable header summary (or the tool
  name as fallback) with no raw JSON argument blob in the collapsed header.

- **SC-007**: In an expanded card, the Input section renders structured key–value rows (not a raw
  JSON dump) by default; the raw-JSON toggle is available and switches to the full dump.

- **SC-008**: A tool result that is markdown/text renders as formatted markdown (at least one of
  heading/list/table/code structure visible), not a single unstructured `<pre>` text blob. A
  valid-JSON result renders as formatted, syntax-highlighted JSON with a copy button. A code
  result renders in a highlighted code block with a copy button.

- **SC-009**: The MCP-UI iframe renderer is preserved with 0 regressions — a result matching that
  marker still renders as the iframe. The nested-delegation flat list is replaced by the recursive
  child-card tree (FR-011); child cards MUST render with the same structure as top-level cards.

- **SC-010**: In a delegation card's Output, 100% of child tool calls render as full recursive
  cards (own header, own Input, own Output, own content-type rendering). The tree is navigable
  to at least 3 levels deep without structural change.

## Assumptions

- The client store is already per-call capable and requires no schema change. The client (card)
  change is limited to: rendering live nested results while running (FR-006), the header summary
  (FR-011), structured args (FR-012), and natural content-type result rendering (FR-013/FR-014).
- `App Target` is `bos-core`: the core of the change is the server run loop
  (`src/lib/assistant/agent-loop.ts`, `inner-loop.ts`), with a presentational change to the shared
  v2 chat card (`src/components/agent/v2/ToolCallCard.tsx`) — not a standalone built-in app.
- "Live output" for a delegation means streaming the delegation's nested tool activity/results
  through the existing per-call channel. It does **not** require token-by-token streaming of a
  subagent's final answer prose; the Claude/OpenCode harness reports its final text once at the end,
  and local delegations produce their final output at return — both unchanged.
- This feature does not mandate that every tool emit live progress; it makes whatever progress a
  tool emits appear live and per-call, and it makes every tool's completion individual.
- `dev_delegate` is not parallel-safe and never batches, so it already completes individually and is
  left unchanged.
- The deterministic-transcript invariant (original call order) is treated as first-class and MUST be
  preserved by the decoupling in FR-001/FR-002.
- The chat already renders the assistant's own message text as markdown; that existing markdown
  renderer (plus syntax highlighting) is reused for tool output — not a new rendering stack.
- Content-type detection (FR-013) is best-effort and heuristic; it MUST degrade gracefully (fall
  back to markdown/text) rather than fail on ambiguous or malformed content.
- US3's header summary (FR-012) is derived per-tool from name + key argument; tools without a
  recognized summarizer fall back to the tool name rather than forcing a summary.
- The mockup at `mockup.html` in this spec directory is the binding UI contract for US3 and
  FR-011–FR-016. Implementation MUST match the mockup's structure (recursive card, independent
  Input/Output collapse, content-type rendering, child-card nesting) — visual details (exact
  spacing, icon choices) may be refined to match the live card's Tailwind classes.
