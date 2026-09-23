# Design — Live Per-Tool Updates & Recursive Tool-Call Rendering

**Feature**: `045-chat-live-tool` (branch `bos/045-chat-live-tool`)
**App Target**: `bos-core`
**Companion**: `spec.md` (requirements, FR/SC) · `mockup.html` (binding UI contract for the recursive card)

This is a structural/architectural design for the 3-part rework of the v2 assistant chat's
tool-call handling. Every path and mechanism below is verified against current source; no new
runtime concept is introduced.

---

## 1. Classification

**`bos-core`.** This changes the server-owned agent loop and a shared client component — the
definition of `bos-core` (part of BOS itself: server logic + `src/` components), not a
first-class window app (`src/apps/<id>/`) and not an installable marketplace item.

It matches the spec's `App Target` field and the spec's own Assumption that the work is
"the core of the change is the server run loop … with a presentational change to the shared v2
chat card — not a standalone built-in app." Concretely the change surface is:

- **Server**: `src/lib/assistant/agent-loop.ts` (run loop — batch execution + emission) and
  `src/lib/assistant/inner-loop.ts` (delegation inner loop), plus the Claude delegation path in
  `src/lib/assistant/tools/server/delegate-common.ts`.
- **Client**: `src/components/agent/v2/ToolCallCard.tsx` (the shared v2 chat card, consumed by
  both the Assistant chat and the Build-Studio self-heal pane).

Nothing runs in a worker-thread service, nothing under `src/apps/`, nothing under
`data/user-apps/`. There is no "Next.js can't do X" limitation being worked around — the
mechanism (per-call run events over NDJSON/SSE) already exists; the fix reorders and forwards
what is already emitted.

---

## 2. Context — what exists today (the load-bearing invariants)

The v2 chat is driven **entirely by the per-call run-event stream**; the client store is a
lossless projection of (persisted messages) + (live run events), rebuilt on replay. That is why
the fix lives upstream of the client, in the run loop, and reuses the existing per-call
vocabulary.

**Run-event vocabulary** — `src/lib/assistant/run-events.ts` (`RunEventInput`). The only tool
events are: `tool_call`, `tool_progress` (per-`callId`, `event: unknown`), `tool_result`
(per-`callId`, `result`), `tool_cancelled`. **No new type is introduced by this feature.**

**RunManager emit** — `src/lib/assistant/run-manager.ts`. `RunManager.emit(run, input)` appends
to the run's **append-only, monotonic-`seq`** event log, dispatches to attached listeners, and
caps the log at `MAX_EVENTS` (50k). The loop's `deps.emit` is wired straight to it
(`src/lib/assistant/start-run.ts:171` → `emit: (e) => manager.emit(run, e)`). Consequence for
this design: **live event order = emission order = `seq` order**; a live `tool_result` emitted
on each call's settle is a distinct, replayable, monotonic event.

**Client projection** — `src/lib/assistant/client/chat-store.ts`, `applyRunEvent`.
`tool_call` → `running`; `tool_progress` → appends to that call's `progress[]`; `tool_result` →
`status: "done"` + `result`; `tool_cancelled` → `status: "cancelled"`. **Each card flips purely
on its own `callId`'s event** — the client is already per-call capable and needs **no schema
change** (the store's `ToolCallView` already has `progress: unknown[]` and `result?`). The only
client change is *rendering*, not state.

**The batch + trailing emission loop** — `src/lib/assistant/agent-loop.ts`, `runAgentLoop`.
Today, per turn:

1. `batchToolCalls(turn.toolCalls, deps.tools, maxParallel)` groups **adjacent** `parallelSafe`
   calls into concurrent batches (`src/lib/assistant/tools.ts`: `AssistantTool.parallelSafe`;
   `src/lib/assistant/tools/server/util.ts`: `parallel(...)` sets it; `agent_delegate` is the
   canonical wrap — `src/lib/assistant/tools/server/subagents.ts`).
2. For each batch, the whole batch is awaited as a unit: `outcomes.push(…await Promise.all(batch.map(executeCall)))`.
3. **Then** a trailing `for (let i = 0; i < batch.length; i++)` loop persists the tool message,
   calls `afterToolCall`, and emits `tool_result` / `message` — in **original call order**, with
   one `saveMessages` per batch.

The two defects are exactly that loop: live emission is **fused** to persistence and to
batch-settle, so (a) every `tool_result` fires in one burst after the slowest call, and (b) the
persisted transcript stays in original order (correct) while the live cards stay frozen until
the burst.

**Delegation inner loop** — `src/lib/assistant/inner-loop.ts`, `runInnerLoop`. This is a
**second invocation of `runAgentLoop`** (blank in-memory transcript), with its `emit` reshaped:
an inner `tool_call` becomes an informational `ctx.onEvent({ tool, input })` entry and (for
frontend-exec tools) a real `manager.emit(parentRun, e)`; **but inner `tool_result`/
`tool_cancelled` are only forwarded for frontend-exec calls** (`frontendCallIds`), so a
local/ephemeral/surface delegation's server-tool results are **swallowed** — never seen by the
parent card until the terminal `encodeNested(...)` result.

**The onEvent → tool_progress conduit** — `src/lib/assistant/agent-loop.ts`, `runServerTool`.
`onEvent(event)` re-arms the per-call idle timer and calls `onProgress(event)`; the loop wires
`onProgress` to `emit({ type: "tool_progress", callId: call.id, event })`. So **any object
pushed through `ToolContext.onEvent` for a call is already streamed to that call's card and
appended to its `progress[]`** — the channel this feature extends.

**Delegation result envelope** — `src/lib/agent/nested-events.ts`: `encodeNested({ events,
output })` / `parseNested(value)` with `NESTED_MARKER` ("" — an empty sentinel, located by
`indexOf`). Today `events: { tool, input? }[]`, `output: string` — **one shared output, no
per-child result/status/callId** (a done delegation's child cards would render blank on reload:
`progress[]` is empty after reload, only the persisted transcript survives). This feature widens
`NestedEvent` (see §3.3 / ADR-3): each entry gains `result?` (string), `status?`
(`"running"|"done"|"cancelled"`) and `nested?` (its own `NestedEvent[]` child list) — recursion by
structure. The marker and the `encodeNested`/`parseNested` entry points are unchanged; only the
*payload shape* changes (legal under FR-007 — a richer `tool_result` string is neither a new event
type nor a store schema change). Produced at the end of every delegation
(`delegate-local.ts`, `delegate-common.ts` claude branch) and **consumed by the card** to render
the nested list.

**MCP-UI result marker** — `src/lib/mcp/ui.ts`: `parseMcpUi(value)` / `MCPUI_MARKER` (""). The
card already special-cases this (iframe) with highest precedence.

**Chat markdown + code renderer** — `src/components/agent/v2/ChatMarkdown.tsx` (reuses
`@copilotkit/react-ui`'s `<Markdown>` wrapper) and `src/components/agent/MarkdownRenderers.tsx`
(Prism `react-syntax-highlighter`, oneDark theme; fenced `CodeBlock` with a copy button;
`HtmlBlock` with Code/Preview; mermaid). This is the **existing** stack to reuse for tool output
— no new rendering dependency.

**Card-collapse registry** — `src/lib/agent/card-collapse.ts`: module-side, **scoped**
(`CardScopeContext`, default `"assistant"`), an **accordion** — `registerCard` opens a
never-seen id (collapsing the rest), `toggleCard` opens/closes one, `useCardOpen(scope, id)` reads
it via `useSyncExternalStore`. State is module-side (not React) so remounts during streaming don't
lose it. **Key constraint for US3**: it is a single-open *accordion keyed by a flat card id*
(`tool:<callId>`). The current card has exactly one such id. The reworked recursive card has
**three** collapsible units (header + Input section + Output section) — the registry must be
extended or the recursive card must scope its own section state.

**Card consumers** (both must keep working — FR-010/SC-009):
- `src/components/agent/v2/MessageListV2.tsx` (`AssistantTurn` → `<ToolCallCard call={…}/>`),
  virtualized with `@tanstack/react-virtual`; `cardsFor()` merges persisted `resultsByCall` with
  live `toolCalls[callId]` (status/progress before the result lands) and memoizes on per-call
  identity.
- `src/apps/build-studio/selfheal/TranscriptsSection.tsx` (`<ToolCallCard call={item.card}/>`),
  which renders headless-run transcript entries as the same card.

The current `ToolCardData` interface (`callId, name, args, status, result?, progress?`) is the
card's entire input contract; US3 must widen it to carry **nested child call data** for the
recursion, while keeping the top-level `MessageListV2.cardsFor()` shape working.

---

## 3. Container & component design

### 3.1 Where each part runs

- **US1 (per-settle emission) and US2 (nested-result forwarding)** run inside the **Next.js app
  process**, in the server agent loop. There is no new container — the inner loop is a *second
  invocation of the same `runAgentLoop`* in the same process; the Claude path spawns a CLI child
  (`src/lib/agent/subagents/claude-runner.ts`) whose stdout stream is already piped to `onEvent`.
- **US3 (recursive card)** is a **client component** in the Next.js app process, rendered inside
  the already-virtualized `MessageListV2` (Assistant) and the self-heal pane. No iframe boundary
  is crossed except the *existing* MCP-UI `srcDoc` iframe.

### 3.2 US1 — decoupled live emission vs. original-order persistence

Split the single fused trailing loop into **two concerns that keep their respective invariants**:

1. **Live emission (completion order).** Each call, the moment it settles, emits its own
   `tool_result`/`tool_cancelled` via `deps.emit`. This is the channel that flips the card. It
   may arrive at the client in any completion order — that is the point.
2. **Persistence (original call order).** After the batch has fully settled, the loop still
   appends tool messages in `batch` order and calls `afterToolCall`/`saveMessages` in that order.
   The `message` run-event (which is what drives the *persisted transcript's* ordering on
   reload) is emitted in original order here.

The structural seam is: **`tool_result` (live, per-settle) is decoupled from the `message` event
+ `io.saveMessages` (persisted, original order).** Today both live in one `for i` loop. The
decoupling moves the `tool_result` emission out to settle-time and leaves `message` + save in the
original-order loop.

- **Cancellation is preserved and *improved*.** Today the loop, on `signal.aborted`, sets
  `cancelledMidTools`, emits `tool_cancelled` for the batch, and persists `CANCELLED_RESULT`.
  In the decoupled design each still-running call settles (via its linked per-call abort,
  `runServerTool`'s `callAbort`) with `CANCELLED_RESULT` and emits its own `tool_cancelled`
  promptly — satisfying FR-004/SC-004 "within one event round-trip," not held for the batch.
- **Errors are isolated.** `executeCall` already never throws (in-band `Error: …`), and the
  defensive wrapper remains — accurately, **one `.catch` per call inside the `batch.map`**
  (`batch.map((call) => executeCall(call).catch((e) => ({ result: toolError(…) })))`, per-call),
  **not** a single `.catch` on the `Promise.all`. In the per-settle design a rejection settles
  only its own call; siblings' emissions are unaffected (FR-008). Keep the per-call shape when
  moving emission to settle-time — do not collapse it into one outer catch (N1).
- **The `tool_call` announcement stays up-front** (the whole batch is announced before awaiting
  any of it) so the browser can dispatch frontend tools together — unchanged.
- **Replay determinism (FR-009/SC-002).** The persisted transcript is the *source of truth for
  reload order* (via `resultsByCall` in `MessageListV2`, which is keyed by `callId` and rendered
  under the assistant message — independent of live `tool_result` seq order). Live replay
  (`?since=`) reorders only the *cards' flip times*, never the persisted message order. This is
  why the client already reads results from `resultsByCall` (persisted) rather than reconstructing
  from `tool_result` seq order.

**Why per-settle emission does not break `seq`:** `manager.emit` assigns `seq` at emission time,
so per-settle `tool_result`s get increasing `seq` in completion order. That is correct — `seq`
is the *event* order, and the client's per-`callId` handling (`applyRunEvent`) is order-agnostic
across calls. `lastSeq` dedupe (`applyRunEvent`) is per-run monotonic and unaffected.

### 3.3 US2 — forward nested results through the existing `tool_progress` channel

The inner loop already reshapes and forwards events via `ctx.onEvent`. The change is to **also
forward nested `tool_result`/`tool_cancelled`** (not just `tool_call` starts) for the non-frontend
(server-exec) tools of a local delegation, to widen the terminal `encodeNested` payload to carry
the per-child results (B1), and — for the Claude/OpenCode path — to surface the nested tool
*starts* the harness emits (narrowed to starts-only; see the Claude bullet below, S1).

- **Local/ephemeral/surface (inner-loop path).** In `runInnerLoop.emit`, extend the handler so a
  nested `tool_result`/`tool_cancelled` for a *server-exec* inner call is pushed through
  `ctx.onEvent` as a structured entry (e.g. `{ type: "tool_result", tool, result }` /
  `{ type: "tool_cancelled", tool }`). This lands in the **parent call's `progress[]`** via the
  existing conduit — no new event type, no store change. The live `progress[]` is the *in-flight*
  projection (starts + results as they settle); `SC-003` ("100% of nested results appear in the
  live progress while running") is met. **The terminal `encodeNested` payload is widened, not
  just the live projection (B1):** `runInnerLoop` must return the full per-child terminal result
  list — the data is already computed by the inner loop's own transcript (each inner call settles
  to a tool message keyed by `callId`), it is discarded today because
  `InnerLoopResult.toolCalls` carries only the *starts* (`{ tool, input }`). Extend
  `InnerLoopResult` with the per-child result list and thread it into `encodeNested` (see §4 /
  ADR-3) so a done delegation's child cards render from the persisted payload after reload.
- **Claude/OpenCode harness path** (`delegate-common.ts` claude branch → `runSubAgent`/
  `runClaudeAgent`). **Narrowed to starts-only (S1).** The harness
  (`src/lib/agent/subagents/claude-runner.ts`) streams only `tool_use` blocks →
  `onEvent({ tool, input })`; it never emits a per-tool `tool_result` (the CLI reports its final
  text once, at the end). The `tool_result` member of the `SubAgentEvent` union
  (`src/lib/agent/subagents/types.ts`) is documented as emitted *only by the local headless path
  (`runLocalHeadless`)* — a different runner (self-heal / Workflow-Manager service), **not** this
  interactive CLI delegation. So US2 for the Claude path surfaces the nested tool *starts* the
  harness emits (matching the spec's own Claude edge case), and the terminal `encodeNested`
  payload for a Claude delegation carries child entries with `result?` undefined and
  `status: "done"` — the card renders them as done cards with no per-child Output. Extending
  `claude-runner.ts` to parse the CLI's `tool_result` blocks is a **separate future change, out
  of scope** for this feature.
- **`encodeNested`/`parseNested` change payload shape** (the marker and entry points stay): the
  terminal `NestedEvent` gains `result?`/`status?`/`nested?` (recursion by structure, ADR-3), so
  a done delegation's child cards render from the persisted terminal payload **after reload**
  (when `progress[]` is empty) — the fix for B1. The live `progress[]` entries also become
  discriminated (start vs. result vs. cancel), which the card's recursive renderer consumes (US3).

### 3.4 US3 — a single recursive `ToolCallCard`

The card becomes **one component rendered at every depth**, mirroring the mockup's
`[data-head]`/`[data-panel]` disclosure model. The three collapsible units are: **header** (the
existing card toggle), **Input section**, **Output section** — each independently collapsible,
defaulting collapsed when the card first opens (FR-015).

- **Header**: status dot (done=emerald / running=amber-pulse / cancelled=grey, matching the
  mockup's `.dot-*`), a per-tool icon (replace the single `Wrench` with a name→icon map; tool name
  is the fallback), and a **human-readable one-line action summary** (verb + key argument, e.g.
  "Read file · agent-loop.ts", "Delegate · <name>"). The summary is derived by a small
  per-tool summarizer keyed on tool name + primary arg; **no raw JSON in the collapsed header**
  (FR-012, SC-006). The `truncate` + `text-white/45` for the secondary arg mirrors the mockup.
- **Input section**: structured key–value rows, **primary arg** (path/query/url/id/command/task)
  as a distinct labeled, emphasized row (`.kv-primary`), remaining args as plain rows. A **`raw`
  toggle** swaps to the full `pretty()` JSON dump. Both views available; structured is default
  (FR-013, SC-007).
- **Output section**: content-type rendering with the spec's precedence
  (FR-014, SC-008/SC-009):
  1. **MCP-UI marker** → existing `parseMcpUi` iframe (highest precedence, preserved).
  2. **nested-delegation marker** → a **list of child `ToolCallCard`s** (the same component,
     recursed) — *replacing* the flat `<NestedEventList>`. Children get the mockup's `.children`
     indent + left border.
  3. **valid JSON object/array** → Prism-highlighted JSON (`CodeBlock` with copy).
  4. **otherwise** → `ChatMarkdown` (existing renderer), fenced code highlighted + copy.
  A small mode label in the Output header ("markdown" / "json" / "code") indicates the chosen type
  (FR-014). **Empty/null result → no Output section** (FR, "empty → no Output section"); the
  mode label is suppressed.
  **Precedence is a change from the current card, described accurately (N3).** Today
  `ToolCallCard`'s ternary order is `liveNested` (in-flight `progress[]`) > `nested` (terminal
  marker) > `mcpUi` (MCP-UI marker) > plain `<pre>`. This design's US3 done-state chain is
  `mcp-ui` > `nested` > `json` > `markdown` — i.e. it moves **MCP-UI above the nested list**
  (currently the nested list wins over MCP-UI) and **splits the terminal plain-text path into a
  strict-JSON vs. markdown decision**. The reorder is practically harmless (a given result is one
  content type — a delegation's `tool_result` is a nested marker, an MCP tool's is an MCP-UI
  marker; they do not co-occur on the same call) but it *is* a change, noted here so the rework
  diff is accurate.
- **Recursion depth & scroll**:
- **Delegation live state**: while running, the Output section renders the live nested
  projection — child cards as each nested tool settles, in-progress children show a spinner +
  their live progress (the recursion holds in the live state, not only after completion —
  FR-006/FR-016, mockup child `c4` "live…").
- **Recursion depth & scroll**: **no hard depth cap** (spec: "There is no hard depth cap"); the
  **per-section scroll clamp** (the current `max-h-64 overflow-auto`) bounds rendered height at
  any depth so a deep/wide tree stays navigable (SC-010: navigable ≥3 levels without structural
  change).

**Collapse-state model for US3 (the one genuine design decision).** The card-collapse registry
(`src/lib/agent/card-collapse.ts`) is an *accordion* of a single open card per scope, keyed by a
flat id — and it is **shared with the reasoning cards** (`MessageListV2.tsx` registers
`reason:<messageId>` into the same scoped accordion, so a tool card and a reasoning card compete
for the one open slot). A recursive card needs independent per-section state (Input / Output)
that (a) survives streaming remounts and (b) does not fight that cross-card accordion. Two viable
shapes (see ADR-2):

- **(A) Extend the registry** to key by a *compound* id (`tool:<callId>`, `tool:<callId>:in`,
  `tool:<callId>:out`) and relax the accordion rule for section-level ids. This touches shared
  code used by reasoning cards, so the "additive section tier" invariant must hold exactly:
  section ids never open over a card-header id and never affect the reasoning accordion.
- **(B) Card-local section state** (a module-side map keyed by `callId`, owned by the card
  module), keeping the registry **strictly for the card-header + reasoning accordion** — its
  cross-card "newest card opens" rule untouched.

Both keep `useSyncExternalStore` and the remount-survival property. **The design picks B (N2).**
Option B is lower-risk to the shared accordion: it adds a private, card-scoped section map and
changes nothing about `registerCard`/`toggleCard`/`useCardOpen` semantics that the reasoning
(and tool) cards already rely on. Option A centralizes *all* collapse state in one place, but
relaxing the single-open rule *inside the shared registry* is exactly the change most likely to
leak into the reasoning accordion (Risk 7); that centralization benefit does not outweigh the
risk for sections that are inherently per-card. The `card-collapse.ts` file is therefore
**untouched**; the new section state is a new card-local module (see §4).

---

## 4. File / Module plan

Only files this feature creates or modifies. One-line purpose each.

### Create
- `src/components/agent/v2/ToolCardSummary.ts` — per-tool action-summary + icon + primary-arg
  selection (name→verb + which arg is "primary"); the fallback to the raw tool name. Pure data,
  no JSX.
- `src/components/agent/v2/ToolCardSections.tsx` — the Input section (structured key–value + raw
  toggle) and the Output section (content-type renderer: mcp-ui / child-cards / json / markdown),
  each independently collapsible; consumed by `ToolCallCard` at every depth.
- `src/lib/agent/card-collapse.tsx` (or an extension module) — *(if ADR-2 option A)* compound
  section ids + a section-local collapse layer on top of the existing card accordion.

### Modify
- `src/lib/assistant/agent-loop.ts` — **US1**: move the `tool_result`/`tool_cancelled` live
  emission to each call's settle point; keep `message` + `io.saveMessages` + `afterToolCall` in
  the original-order trailing loop; preserve per-call announce, cancel cascade, and error
  isolation.
- `src/lib/assistant/inner-loop.ts` — **US2 (local)**: forward nested `tool_result`/
  `tool_cancelled` (server-exec inner calls) through `ctx.onEvent`, so the parent card streams
  nested results live; terminal `encodeNested` unchanged.
- `src/lib/assistant/tools/server/delegate-common.ts` — **US2 (claude)**: surface nested tool
  results from the harness `onEvent` (`SubAgentEvent` `tool_result`) for the claude/OpenCode
  path; `encodeNested` unchanged.
- `src/components/agent/v2/ToolCallCard.tsx` — **US3**: rework from flat single-level to the
  single recursive component (header summary; Input/Output sections; delegation Output = child
  cards); widen `ToolCardData` to carry nested child call data + live nested projection.
- `src/components/agent/v2/MessageListV2.tsx` — **US3 (wiring only)**: build the card data's
  nested children for delegations from the existing `resultsByCall` / live `progress[]`; keep the
  virtualization + per-call memo comparator.
- `src/lib/agent/nested-events.ts` — **US3 (type only)**: *(if the live `progress[]` gains
  discriminated start/result/cancel entries)* export a typed `NestedProgressEntry` union for the
  card; `encodeNested`/`parseNested`/marker unchanged.

**Explicitly NOT touched** (hard-constraint guard, SC-005): no `run-events.ts` (no new event
type), no transport/protocol, no `chat-store.ts` (no schema change), no CopilotKit dependency,
no `card-collapse` behavior for non-tool cards (reasoning cards keep working).

---

## 5. Integration points (existing BOS mechanisms this feature calls into)

- **`RunManager.emit`** (`src/lib/assistant/run-manager.ts`) — the loop's `deps.emit`
  (`start-run.ts:171`). US1 emits per-settle `tool_result`/`tool_cancelled` through it; `seq`
  is assigned here, append-only, monotonic. US2's inner loop already routes through it for
  frontend-exec calls.
- **`ToolContext.onEvent`** (`src/lib/assistant/tools.ts`) → the `tool_progress` conduit
  (`agent-loop.ts` `runServerTool` → `onProgress` → `emit tool_progress`). US2 pushes nested
  results through this; the store appends to `progress[]`; the card renders it live. This is the
  **no-new-event-type** channel the spec relies on.
- **`encodeNested` / `parseNested`** (`src/lib/agent/nested-events.ts`, `NESTED_MARKER`) — the
  terminal delegation envelope (produced by `delegate-local.ts`/`delegate-common.ts`, consumed by
  the card). US3 replaces the flat `NestedEventList` with recursive child cards *from the same
  `parseNested` payload*; the marker is the "delegation" content-type signal.
- **`parseMcpUi`** (`src/lib/mcp/ui.ts`, `MCPUI_MARKER`) — the MCP-UI iframe content type; kept
  at the top of the Output precedence chain, unchanged.
- **Existing chat markdown renderer** — `ChatMarkdown.tsx` + `MarkdownRenderers.tsx` (Prism
  `CodeBlock`/`HtmlBlock`/mermaid + copy button). US3 reuses it for the "markdown" and
  "code"/"json" output modes (and the highlighted-JSON path). No new rendering stack.
- **Card-collapse registry** (`src/lib/agent/card-collapse.ts`) — `registerCard`/`toggleCard`/
  `useCardOpen`/`useCardScope`, scoped, remount-safe. US3's header toggle reuses it; its
  Input/Output section state extends it (ADR-2).
- **`applyRunEvent`** (`src/lib/assistant/client/chat-store.ts`) — the per-`callId` projection;
  US1's per-settle `tool_result` flips each card individually with **zero client changes** (the
  store already handles `tool_result` per callId). This is why US1 needs no client-store edit.

---

## 6. ADRs

### ADR-1 — Decouple live `tool_result` emission (completion order) from persistence (original
call order)

**Decision.** In `agent-loop.ts`, emit each call's `tool_result`/`tool_cancelled` **at its own
settle** (completion order) so the client flips cards individually; keep the `message` event +
`io.saveMessages` + `afterToolCall` in a trailing loop that runs in **original batch order**
after the batch has fully settled.

**Status.** Accepted.

**Consequences / rationale.**
- The two invariants were *fused* in one `for i` loop; splitting them is what makes both hold at
  once. Live order (drives the UI) and persisted order (drives the deterministic transcript)
  become independent by construction (FR-001/FR-002/FR-009, SC-001/SC-002).
- The **persisted transcript's** order is what reload reads: `MessageListV2.resultsByCall` is
  keyed by `callId` and rendered under the assistant message, so it is **already** independent of
  live `tool_result` `seq`. This means original-order persistence is preserved *without* the
  client having to re-sort by seq. Two runs of identical input yield identical persisted order
  regardless of live timing (SC-002).
- **Replay** of a run whose live order differed: the live `tool_result`s replay in completion
  order (cards flip in that order), but the persisted `message` events replay in original order —
  so the *transcript* is stable and the *cards* merely reflect the historical flip times. No
  invariant is violated (FR-009).
- Cancellation: per-call linked abort (`runServerTool`'s `callAbort`) settles each still-running
  call with `CANCELLED_RESULT`; emitting `tool_cancelled` per-settle satisfies FR-004/SC-004
  ("within one event round-trip," not held for the batch).
- **Cost/risk**: the per-settle path must still guarantee *every* call gets exactly one live
  terminal event (result or cancel) before the batch's persistence loop runs — otherwise a
  `tool_use` id would be stranded with no `tool_result` and wedge the next model turn (the exact
  hazard the current defensive `.catch` wrapper prevents). The settle-time emission must be
  idempotent per `callId` and ordered so a call can't be both "done" and "cancelled."
- **Alternative rejected** — "persist in completion order and re-sort the transcript": breaks the
  deterministic-transcript invariant and forces the client to sort by original order (which it
  can't know post-reload without re-deriving from the assistant message). Original-order
  persistence is the correct first-class invariant.

### ADR-2 — Single recursive component vs. a separate flat nested-list component

**Decision.** One `ToolCallCard` component, rendered identically at every depth; a delegation's
Output section is a **list of child `ToolCallCard`s** (structural recursion), *not* a separate
flat list. The flat `<NestedEventList>` is removed (SC-009).

**Status.** Accepted.

**Rationale.**
- FR-011 requires the nesting to be *structural* (same component, recursed), and the mockup is
  the binding contract for exactly this shape (a delegation's Output = child cards, each with its
  own header/Input/Output, indented + left border). A flat list cannot express the recursion the
  spec demands (SC-010: navigable ≥3 levels with no structural change).
- **Reuse, not duplication**: the Input/Output sections and content-type renderer are written
  once and shared at every depth; a delegation's child is just the parent's own Output rendering
  another card.
- **Single collapse-state source of truth**: header + Input + Output are three collapsible units
  of the *same* logical card. Keeping them in one component (with the registry extended to
  compound ids, option A) means the "newest card opens" accordion still governs the *card* while
  *sections* are local sub-state — no separate store to keep in sync.
- **Alternative rejected** — "separate `NestedDelegationList` component": duplicates the card's
  render logic, breaks the "same component at every depth" contract, and makes the live nested
  results (US2) need a second code path to render children in-flight. The recursive card handles
  live and terminal nested state in one place.
- **Cost/risk**: recursion + `useSyncExternalStore` per section multiplies subscriptions in a
  deep tree. Mitigated by (a) virtualization of the top-level message list (`MessageListV2`),
  (b) per-section scroll clamp bounding rendered height at any depth (no hard depth cap, per
  spec), and (c) keeping `ToolCallCard` `memo`-ized on its own call data so a sibling's progress
  tick doesn't re-render a collapsed ancestor's whole subtree.

### ADR-3 — Live nested results ride the existing `tool_progress` channel (no new event type)

**Decision.** US2 forwards nested `tool_result`/`tool_cancelled` as *structured entries* pushed
through `ToolContext.onEvent` (→ `tool_progress` → `progress[]`), discriminated from the existing
`{ tool, input }` start entry. The terminal `encodeNested` envelope is unchanged and remains the
canonical nested list.

**Status.** Accepted.

**Rationale.**
- FR-007 forbids a new run-event type; `tool_progress` is *exactly* the per-`callId` partial-
  output channel, and `ToolCallView.progress: unknown[]` already accepts arbitrary entries — so
  forwarding richer entries is a **zero-protocol, zero-store** change.
- The live `progress[]` is the *in-flight projection*; the terminal `encodeNested` result is the
  *settled source of truth*. The card prefers the settled payload once `status === "done"` and
  falls back to the live projection while `running` — which is how "children stream live, then
  the full list + final output on completion" (FR-006) works without ever changing the
  persistence contract.
- **Alternative rejected** — "add a nested `tool_result` run-event": violates the no-new-event-
  type constraint and would require a client-store schema change to track per-nested-call state.
- **Cost/risk**: `progress[]` is `unknown[]`, so the card must defensively discriminate entries
  (start vs. result vs. cancel) and tolerate the legacy `{ tool, input }` shape (existing runs /
  the Claude path before US2 lands). The renderer must degrade gracefully for any unrecognized
  entry (the existing `toNestedEvents` filter is the model to preserve).

---

## 7. Risks

1. **Deterministic-transcript invariant (FR-002/FR-009/SC-002) — highest severity.**
   The whole US1 design rests on *persisted* order (original call order, via `message` events +
   `saveMessages`) being independent of *live* order (per-settle `tool_result`). If the
   decoupling accidentally persists in settle order, or if the client were to order cards by
   `tool_result` `seq` on reload, SC-002 breaks. **Mitigation**: keep `saveMessages`/`message`
   emission strictly in the original-order trailing loop; keep `resultsByCall` (persisted,
   keyed by `callId`) as the reload source of order. Add a regression test: two runs of identical
   input with staggered completion produce byte-identical tool-result message ordering.

2. **Replay of a run whose live order differed (FR-009).**
   Live replay re-flips cards in completion order while the persisted transcript stays in
   original order. If any surface renders the *transcript* from live `tool_result` seq (rather
   than from persisted `message`s), it would show a wrong order on replay. **Mitigation**: audit
   both consumers (`MessageListV2`, `TranscriptsSection`) — both already read results from
   persisted/`resultsByCall`, so they are safe; the risk is a *future* consumer re-deriving order
   from `seq`. Call this out in the code near the decoupled loop.

3. **Recursion depth / scroll clamp / subscription explosion (SC-010, "unbounded recursion"
   edge case).**
   No hard depth cap + one `useSyncExternalStore` per collapsible section at every depth =
   O(nodes) subscriptions in a deep tree; a pathologically deep delegation could lag. **Mitigation**:
   per-section `max-h-64 overflow-auto` clamp (preserve the current behavior), `memo` per card on
   its own call data, top-level list virtualization already in place, and the collapse registry
   keeping collapsed subtrees out of the render path. **Residual risk**: very wide + very deep
   trees in one view — acceptable given the clamp bounds *rendered height*, but note it.

4. **Content-type detection heuristics (FR-014, "ambiguous content," "content that resembles
   markup/code").**
   The "valid JSON object/array vs. markdown" decision is best-effort; a JSON-looking string that
   is prose, or markdown with raw `<`/`&`, must degrade gracefully. **Mitigation**: precedence is
   fixed (mcp-ui marker → delegation marker → *strict* `JSON.parse`-to-object/array → markdown);
   only *strict* parse-to-object/array takes the JSON path (a bare JSON *string* or number is not
   treated as a JSON blob — it falls to markdown/text). Wrap `JSON.parse` in try/catch; any
   exception or non-object/array result falls through to `ChatMarkdown`. The existing
   `ChatMarkdown`/`MarkdownRenderers` already escape/sandbox (`CodeBlock`, `HtmlBlock` iframe),
   so markup-like content renders as code, never executed. **Residual risk**: a prose string that
   happens to be a valid JSON *array* of strings will render as JSON — acceptable per "best-
   effort, prefer graceful."

5. **Inner-loop event-shape churn (US2).**
   The inner loop's `emit` currently treats `tool_result`/`tool_cancelled` specially only for
   `frontendCallIds`. Extending it to also forward server-exec inner results must not (a) double-
   emit the terminal `encodeNested` result, (b) forward the inner loop's own *top-level* events in
   a way that confuses the parent, or (c) change what `runLocalDelegation`'s returned string is
   (the `encodeNested` envelope is the contract the card's terminal render depends on). **Mitigation**:
   forward *only* inner-tool results as `progress[]` entries; leave the `runAgentLoop` return
   path (and thus the `encodeNested` payload) untouched. Add the `inner-loop.test.ts`-style
   assertion that the parent's `progress[]` gains a result entry per nested settle while the
   terminal result string is unchanged.

6. **Claude-harness live output (US2 claude path, spec edge case).**
   The harness streams `tool_use` (starts) and reports final text once; nested tool *results*
   are not all guaranteed by every CLI version. **Mitigation**: US2 for the Claude path covers
   "the nested tool activity/results the harness *does* emit" and explicitly does **not** require
   token-by-token final prose (per spec). The card must render the live projection from whatever
   arrives and fall back to the terminal `encodeNested` list, so a harness that only emits starts
   still shows the full list on completion — no regression.

7. **Card-collapse registry extension (ADR-2 option A) leaking into the reasoning accordion.**
   Relaxing the "single open per scope" rule for section-level ids must not let a tool card's
   Input section "open" over the reasoning card or another tool card's header. **Mitigation**:
   scope section ids as sub-ids of the card id and keep the *cross-card* accordion rule
   (newest card opens) strictly on card-header ids; sections only compete *within* their own
   card. This is the cleanest invariant to test (a card's Input opening must not close a
   different card).
