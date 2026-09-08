# Design: Tool State Color Coding in the Assistant Info Panel

**Spec**: `042-tool-color-coding`
**Feature Branch**: `bos/042-tool-color-coding`
**App Target (spec.md)**: `bos-core`
**UI contract**: `mockup.html` (sibling to this file)

---

## 1. Classification

**App Target: `bos-core`** — agrees with `spec.md`.

This is a change to a first-class, already-compiled UI component (`InfoPanelV2.tsx`) plus a new
framework-free domain module and a test. It is not a self-contained installable item (no `app/`
iframe, no `services/` worker, no own port) and not a standalone window app. It touches `src/`
directly and ships inside BOS itself. `builtin-app` does not apply (it's not a new `src/apps/<id>/`
window — it modifies the existing Assistant component tree). So `bos-core`, no disagreement to flag.

---

## 2. Constitution check

| Principle | Verdict |
|---|---|
| **II — Server Authority & SSR Boundary** | ✅ Compliant, and this is the one that needs explicit reasoning (see below). The client derives the revealed set with `deriveRevealedIds` (`src/lib/assistant/messages.ts`) over the client's own `ChatMessage[]` projection. `messages.ts` is **framework-free — no `import "server-only"`, no Node APIs, zero imports** (its only dependency is the `ChatMessage` type defined in the same file). No secret is exposed: the messages it reads are the user's *own* transcript, which the server already delivers to this exact client via the run event stream / `openConversation`. Client-side derivation here is the same computation the server performs, over data the client legitimately holds. |
| **IV — Minimize Blast Radius** | ✅ On `bos/042-tool-color-coding`. No secrets/lockfile/build changes. One new framework-free module + one test + edits to two existing files. |
| **VI — Specs & Docs Stay in Sync** | Flagged as an implementation task (not a design decision): the Assistant end-user doc gains one sentence that the Tools tab is now agent- and conversation-aware. See §7 open questions. |
| Others | No conflict. |

The only genuine tension is **mockup vs. source**, not the constitution: the mockup draws a "Core"
group containing `find_tools`/`find_agent`, but those ids are **not** in `CAPABILITIES`
(`src/lib/agent/capabilities-registry.ts`) and are therefore **not** in `assistantToolsManifest()`
(`src/lib/agent/tool-manifest.ts`) — they render no rows today. The spec is explicit that this feature
"*only restyles rows that already render*" and "MUST NOT invent rows." **Decision: do not add a Core
group / discovery rows.** The classifier carries a defensive `isDiscovery` branch so that *if* a
discovery tool were ever rendered it would be neutral; today that branch never fires. See ADR-2.

---

## 3. Architecture

### The hinge: three single sources, shared by the run and the panel

The whole feature is sound only if the panel's per-row classification is derived from **the same
three sources the run uses**, so a color can never disagree with what the agent can actually call.
All three verified in source this session:

| Source | Run (server) uses | Panel (client) uses | Same object/field? |
|---|---|---|---|
| **Granted allowlist** | `gateFromAgent` → `allow: new Set(agent?.tools ?? [])` (`src/lib/assistant/gate.ts`, called from `start-run.ts` via `gateFor(agentId)`) | `GET /api/assistant/agent` → `tools: a.tools ?? []` (`src/app/api/assistant/agent/route.ts`), fetched by the existing InfoPanelV2 effect | ✅ Same `Agent.tools` field from `src/lib/agent/subagents/store.ts` |
| **Deferred** | `deferred: new Set(agent?.deferredTools ?? [])` (registry default `deferredCapabilityIds()` is empty) | Same route → `deferredTools: a.deferredTools ?? []` | ✅ Same `Agent.deferredTools` field |
| **Revealed** | `deriveRevealedIds(messages)` → `visibleTools(tools, gate, revealed)` per step (`src/lib/assistant/agent-loop.ts`); `deriveRevealedIds` imported from **`src/lib/assistant/messages.ts`** | `deriveRevealedIds(messages)` from the **same `messages.ts`**, over the chat store's `ChatMessage[]` projection | ✅ Same pure function, same canonical transcript shape |

**The critical disambiguation (do not get this wrong in implementation):** there are **two** functions
named `deriveRevealedIds`.

- `src/lib/assistant/messages.ts` — **framework-free, client-importable, zero imports**, operates on
  the persisted `ChatMessage[]` shape (`m.role`, `m.toolCalls`, `m.toolCallId`, `JSON.parse(m.content)`).
  **This is the one the run loop calls and the one the panel must call.**
- `src/lib/agent/tool-gate.ts` — **`import "server-only"`, NOT client-importable**, operates on the
  in-memory *model-prompt* shape (`content[].part.toolName/output`). This is the legacy sub-agent gate
  path. **The panel must not import this** (it would break the client build, and it reads a different
  shape).

`agent-loop.ts` line `const revealed = deriveRevealedIds(messages);` resolves to the `messages.ts`
version (its import statement is `from "./messages"`). The panel imports `deriveRevealedIds` from
`@/lib/assistant/messages` — the identical symbol. SC-001 follows from identity of function *and*
identity of input (the client store is a projection of the server-owned transcript).

### Context / Container

One container: the **Next.js (App Router) app process**. There is no new service, daemon, port, or
worktree. The change is entirely within the client bundle (the `"use client"` `InfoPanelV2` subtree)
plus a shared framework-free module that the server, client, and tests all import.

```
mermaid
graph LR
  subgraph AppProcess["Next.js app process"]
    subgraph Client["Client bundle"]
      AC["AssistantChatV2 (memo)"] -->|"agentId + conversationId (primitives)"| IP["InfoPanelV2"]
      IP -->|"tools/deferredTools (arrays|null) + conversationId"| TT["ToolsTab"]
      CS["chat store<br/>(useChatSelector)"] -. "s.messages (ChatMessage[])" .-> TT
      TT -->|"deriveRevealedIds + classifyToolState"| COLOR["colored rows"]
    end
    API["GET /api/assistant/agent"] -->|"tools/deferredTools/skills/mcp"| IP
    subgraph Shared["Framework-free modules (client+server+tests)"]
      MSG["src/lib/assistant/messages.ts<br/>deriveRevealedIds, ChatMessage"]
      TS["src/lib/agent/tool-state.ts  (NEW)<br/>classifyToolState, ToolState, DISCOVERY_TOOL_IDS"]
      MF["src/lib/agent/tool-manifest.ts<br/>assistantToolsManifest"]
    end
    TT --> MSG
    TT --> TS
    TT --> MF
    subgraph Server["Server (run) — SAME sources"]
      GATE["gate.ts gateFor(agentId)"]
      LOOP["agent-loop.ts: deriveRevealedIds + visibleTools"]
    end
    LOOP --> MSG
  end
```

### Component

- **`InfoPanelV2`** — unchanged structure; its existing `/api/assistant/agent` effect is extended to
  also capture `tools` + `deferredTools` into the same `caps` state it already holds for skills/mcp.
  It passes `tools`/`deferredTools` (nullable while loading) + `conversationId` into `ToolsTab`.
- **`ToolsTab`** — the only component that becomes agent- *and* conversation-aware. It owns the
  `useChatSelector(conversationId, s => s.messages)` subscription, derives the revealed set, and colors
  each row via `classifyToolState`.
- **`tool-state.ts` (NEW)** — the pure classifier + types + discovery-id set. No React, no
  server-only.

### Re-render / reactivity (FR-007, FR-008, SC-002, SC-003)

The `useChatSelector` subscription lives **inside `ToolsTab`**, not in `InfoPanelV2` or
`AssistantChatV2`. This is deliberate and load-bearing:

- `AssistantChatV2` is `memo`ized specifically so unrelated re-renders (Build Studio file tree, etc.)
  don't reconcile the whole chat subtree, *and* the codebase's own `useChatSelector` docstring warns
  that a whole-object store subscription re-renders "dozens of times per second during a streaming
  reply." If the panel read `s.messages` at the `AssistantChatV2`/`InfoPanelV2` level, every transcript
  append would re-render the memoized chat. Placing it in `ToolsTab` scopes the subscription to the one
  component that consumes it, and `ToolsTab` is **mounted only while the Tools tab is active** — switch
  to Skills/MCP and the subscription (and its re-renders) stop entirely.
- `useChatSelector` returns a stable `s.messages` reference on `text_delta`/`tool_progress` (those
  events change `streamText`/`toolCalls`, not the `messages` array), so `ToolsTab` re-renders **only**
  on a genuine transcript append (`message` event), an agent change, or a conversation change. That is
  exactly the set of moments the revealed/allow/deferred inputs can change.

The three triggers map to concrete signals:

| Trigger | Signal | Where it lands |
|---|---|---|
| **Agent change (FR-008 / SC-002)** | `resolvedAgentId` prop on `InfoPanelV2` changes → its `useEffect([agentId])` refetches `/api/assistant/agent` → `caps.tools`/`caps.deferredTools` change | `ToolsTab` receives new arrays → reclassifies every row from scratch (no per-row state persists, so no stale color — SC-002) |
| **Conversation change (FR-007)** | `conversationId` prop on `ToolsTab` changes → `useChatSelector(conversationId, ...)` re-binds to the new conversation's `messages` | revealed set becomes the *new* conversation's → a tool revealed in conv A is orange again in conv B (SC-003 revert) |
| **Reveal within a conversation (SC-003)** | a `find_tools` `tool` message is appended to the store's `messages` for the active conversation → `s.messages` reference changes | `deriveRevealedIds` now includes that id → the row flips orange → blue |

**Loading (the "empty vs. unknown" trap).** `ToolsTab` receives `tools`/`deferredTools` as
`string[] | null` where `null` means *the agent fetch hasn't resolved yet*. While `null`, render every
row **neutral** (today's look) — do **not** apply the lenient `allows()` helper that the Skills/MCP
tabs use (that helper treats empty/undefined as "allow all" to avoid a disabled-flash on load, which is
the *opposite* of what the tool-state feature needs). The spec is explicit that an **empty** allowlist
means **zero** granted → all neutral, mirroring the run's strict `gate.allow.has(id)`. The two are
distinguished: `null` = unknown → neutral (loading, safe); `[]` = known-empty → neutral (strict).
Both happen to render neutral, but they take different code paths and the classifier itself is strict.
This prevents a momentary green flash on tools the selected agent doesn't actually have.

---

## 4. Concrete file/module plan

**Creates:**
- `src/lib/agent/tool-state.ts` — framework-free. Exports:
  - `type ToolState = "granted" | "deferredHidden" | "deferredRevealed" | "neutral"`
  - `const DISCOVERY_TOOL_IDS: ReadonlySet<string>` — `{"find_tools","find_agent"}` (mirrors the
    non-importable server-only sets in `tools.ts`/`gate.ts`; see ADR-2)
  - `classifyToolState(toolId: string, ctx: { allow: ReadonlySet<string>; deferred: ReadonlySet<string>; revealed: ReadonlySet<string>; isDiscovery: boolean }): ToolState`
  - (The `STATE_COLOR` class map + legend belong in the component, not here — keep the module pure.)
- `src/lib/agent/__tests__/tool-state.test.ts` — the SC-001 test (and the §5 edge-case table).

**Modifies:**
- `src/components/agent/v2/InfoPanelV2.tsx`
  - Extend the existing `caps` state to `{ skills: string[]; mcp: string[]; tools: string[]; deferredTools: string[] }` (still `null` while loading) and read `tools`/`deferredTools` from the same `/api/assistant/agent` response it already fetches (no new fetch).
  - Accept a new prop `conversationId?: string`; pass `conversationId`, `caps?.tools ?? null`, and `caps?.deferredTools ?? null` into `<ToolsTab/>`.
  - `ToolsTab` signature becomes `({ tools, deferredTools, conversationId }: { tools: string[] | null; deferredTools: string[] | null; conversationId: string })`; it computes the revealed set + set-ification and colors the `<Wrench>`; adds the legend block at the top. Skills/MCP tabs and the `allows()` helper are untouched.
- `src/components/agent/v2/AssistantChatV2.tsx`
  - One line: `{props.showInfo && <InfoPanelV2 agentId={resolvedAgentId} conversationId={conversationId} />}` (`conversationId` is already in scope: `const conversationId = conv.activeId ?? ""`).

**Deliberately NOT changed** (they are integration points, §6): `tool-manifest.ts`, `messages.ts`,
`tool-gate.ts`, `gate.ts`, `tools.ts`, `agent-loop.ts`, `/api/assistant/agent/route.ts`.

### The classifier, precisely (predicate order is normative)

```
classifyToolState(toolId, { allow, deferred, revealed, isDiscovery }):
  1. if isDiscovery                     -> "neutral"            // discovery bypasses everything
  2. if !allow.has(toolId)              -> "neutral"            // not granted (incl. empty allow)
  3. if deferred.has(toolId) && revealed.has(toolId) -> "deferredRevealed"  // blue
  4. if deferred.has(toolId)            -> "deferredHidden"     // orange
  5. else                               -> "granted"            // green (granted, non-deferred)
```

Checks 3–5 are only reached when the tool *is* granted (checks 1–2 fell through), so "granted" is the
implicit precondition for the color states. Order matters: **neutral first** (so revealed-but-not-granted
→ neutral, and discovery → neutral), then blue before orange (so a revealed deferred tool is blue, not
orange), else green.

### Edge cases (spec "Edge Cases") mapped to the predicate

| Edge case | allow / deferred / revealed / isDiscovery | Result | Why |
|---|---|---|---|
| Empty allowlist | `allow={}`, any | `neutral` (all rows) | Check 2: `allow.has` false for everything → all neutral. (Strict membership, **not** `allows()`.) |
| Revealed but not granted | `allow`∌id, `revealed`∋id | `neutral` | Check 2 fires before any color check. |
| Revealed and non-deferred (granted) | `allow`∋id, `deferred`∌id, `revealed`∋id | `granted` (green) | Checks 3–4 fail (not deferred) → falls to 5. **Never blue.** |
| Discovery tool | `isDiscovery=true` | `neutral` | Check 1, first. (Defensive — see ADR-2; not a manifest row today.) |
| Service tool, dynamic | same path as built-ins, by id | (granted/hidden/blue per sets) | Sets are plain id sets; `assistantToolsManifest()` returns service tools via `listCapabilities()`'s dynamic layer. When the service stops the tool leaves the manifest → the row is gone → no stale color (classification is recomputed fresh each render, nothing persisted). |
| No active conversation yet | `conversationId=""` → `s.messages=[]` → `revealed={}` | granted non-deferred = green, granted deferred = orange | `deriveRevealedIds([])` = empty set. |

---

## 5. Integration points (existing mechanisms this design *calls into*, does not create/modify)

Each is cited to the real file that makes it exist:

- **`GET /api/assistant/agent`** — `src/app/api/assistant/agent/route.ts`. Already returns
  `tools: a.tools ?? []` and `deferredTools: a.deferredTools ?? []` for every agent; InfoPanelV2 already
  fetches this route (today it reads only `skills`/`mcp`). The panel now also reads `tools`/
  `deferredTools` from the same response. **No new API route, no new fetch, no server change.**
- **The chat store** — `src/lib/assistant/client/chat-store.ts`. `useChatSelector(conversationId,
  s => s.messages)` returns the per-conversation `ChatMessage[]` projection (fed by `openConversation`
  history load + live `message` run events). This is the same data the server is the single writer of.
- **`deriveRevealedIds` + `ChatMessage`** — `src/lib/assistant/messages.ts` (framework-free). The exact
  function the run loop uses (`agent-loop.ts`), so the client's revealed set cannot diverge from the
  run's.
- **`assistantToolsManifest`** — `src/lib/agent/tool-manifest.ts`. Already called by `ToolsTab`; it
  rebuilds from `listCapabilities()` each render so dynamic service tools are present/absent live.
- **The run's gate semantics** — `src/lib/assistant/tools.ts` `visibleTools` (the per-step decision the
  SC-001 test compares against) and `src/lib/assistant/gate.ts` (how `allow`/`deferred` are built from
  the `Agent`).

---

## 6. ADRs

### ADR-1 — Compute the revealed set client-side from the store; do not add an API route

- **Context**: The panel is not conversation-aware today. It needs the active conversation's revealed
  set. Two options: (a) fetch a new endpoint that returns the revealed set, or (b) derive it client-side
  from the messages the chat store already holds.
- **Options**:
  - (a) New `/api/assistant/revealed?conversationId=` route → server runs `deriveRevealedIds` and
    returns the set.
  - (b) Reuse the existing client store: `useChatSelector(conversationId, s => s.messages)` +
    client-side `deriveRevealedIds`.
- **Decision**: **(b)**.
- **Consequences / rationale**:
  - It reuses a real, existing mechanism (the store is the UI's source of truth for the transcript) and
    adds **zero** surface area — no route, no new server code, no extra network round-trip per reveal.
  - It is **inherently in sync**: the store's `messages` are a projection of the server-owned
    transcript, and `deriveRevealedIds` is the *same pure function* the run uses. There is no second
    copy of the logic to drift (which is exactly the failure SC-001 guards against).
  - It is **reactive for free**: a `find_tools` result appends a `message` event → the store's
    `s.messages` reference changes → the panel re-renders → orange flips to blue (SC-003) with no polling
    or explicit invalidation.
  - Safety: no secret is exposed (the transcript is the user's own, already delivered to this client);
    `messages.ts` is framework-free (no `server-only`), so importing it on the client is legal.
  - The alternative (a) would only be warranted if the revealed set were *not* derivable from data the
    client already holds, or if it required server-only inputs. Neither is true.

### ADR-2 — Keep the classifier pure and put `DISCOVERY_TOOL_IDS` in the new module (defensive)

- **Context**: The classifier needs to know whether a row is a discovery tool. The existing
  `DISCOVERY_TOOLS` set lives in `src/lib/assistant/tools.ts` (a module-level `const`, **not exported**)
  and `ALWAYS_AVAILABLE` in `src/lib/assistant/gate.ts` (**`import "server-only"`**, not exported) —
  neither is importable from a framework-free client module.
- **Options**:
  - (a) Export a discovery set from a server-only module and import it. — Rejected: `gate.ts` is
    server-only; `tools.ts`'s set isn't exported and exporting it from there couples the panel to the
    run-registry module.
  - (b) Define `DISCOVERY_TOOL_IDS` in the new framework-free `tool-state.ts`, as a mirror.
  - (c) Fold the discovery check *inside* `classifyToolState` by hardcoding the two ids.
- **Decision**: **(b)**, with the set passed in as the `isDiscovery` parameter (the task's recommended
  signature keeps the function maximally pure and trivially testable — the *caller* computes
  `isDiscovery: DISCOVERY_TOOL_IDS.has(toolId)`).
- **Consequences / rationale**:
  - The discovery branch is **defensive, not load-bearing**: `find_tools`/`find_agent` are not in
    `CAPABILITIES`, so `assistantToolsManifest()` never emits them and the branch cannot fire for any
    currently-rendered row. It exists so the classifier's contract ("discovery tools are always
    neutral") holds *regardless* of what the manifest returns — including if a future manifest change
    ever surfaces a discovery row. This honors the spec's edge case without inventing rows (the mockup's
    "Core" group is illustrative; the spec forbids new rows).
  - The drift risk of a duplicated two-element set is accepted and bounded: it mirrors an explicitly
    constant pair that has not changed across 025/039/041, and the SC-001 test pins the panel's behavior
    against the real gate, so if the server's notion of "discovery" ever grew, the test would fail and
    surface the mismatch. The set is commented to point at its two server-side twins.

### ADR-3 — Scope the conversation subscription to `ToolsTab`, not `InfoPanelV2`/`AssistantChatV2`

- **Context**: `AssistantChatV2` is `memo`ized to keep unrelated re-renders out of the chat subtree; the
  codebase's own `useChatSelector` docstring warns against whole-store subscriptions re-rendering per
  streamed token. Where should the `s.messages` subscription (needed only for the Tools tab) live?
- **Decision**: Inside `ToolsTab`, which is mounted only when the Tools tab is active.
- **Consequences**:
  - On Skills/MCP tabs the subscription is unmounted → no wasted re-renders from transcript appends.
  - A transcript append re-renders `ToolsTab` only (its own reconciliation), never `AssistantChatV2`,
    so the memo contract is intact.
  - `useChatSelector` already returns a stable reference on non-`message` events (see §3), so even the
    Tools tab re-renders only on real appends. This directly discharges the flagged
    "memo/perf of InfoPanelV2 under memoized AssistantChatV2" risk.

---

## 7. Risks / open questions

1. **`deriveRevealedIds` identity is the load-bearing assumption.** The design only holds if the panel
   imports the `messages.ts` version (client-safe, `ChatMessage[]`-shaped) and *not* the `tool-gate.ts`
   version (server-only, model-prompt-shaped). **Mitigation**: state it explicitly in the test and a
   comment at the import site; the SC-001 test asserts the panel's colors match `visibleTools` for the
   same (agent, messages) input, which only passes if the same function/shape is used. If the two ever
   diverge, the test is the canary.
2. **Mockup's "Core" group vs. the manifest not rendering discovery tools** — resolved by ADR-2 (defensive
   branch, no new rows). If a reviewer believes discovery tools *should* be rendered as neutral rows,
   that is a separate manifest/feature decision outside this spec's "restyle only" scope — flag it to
   Build Studio, don't fold it in silently.
3. **Client-side revealed derivation is a duplicate of server computation.** It is cheap (linear over
   the transcript's tool messages) and correctness-safe (same function, same data), but it *is* a second
   execution site. **Mitigation**: it reads only the store's existing `messages`; there is no new shared
   state, no caching to invalidate, and it is recomputed on the (rare) transcript append. Acceptable.
4. **Doc sync (Constitution VI).** The Assistant user doc should note the Tools tab is now
   agent-/conversation-aware. One sentence; assign in `plan.md`/`tasks.md`.

---

## 8. UI mockup reference

**Path**: `mockup.html` (sibling to this file). It is the visual contract; the behavioral decisions it
fixes are honored, not re-litigated:

- **Color cue = the wrench icon's color class only** (no per-row swatch/dot). The mockup sets
  `data-state` on the row and colors `.wrench`; in real code this is the `<Wrench>` `className`.
- **Palette on `#0f1117`** (BOS named accents, per the style guide — the mockup itself notes "real code
  should use the named Tailwind classes"): `granted` → `text-emerald-400`, `deferredHidden` →
  `text-amber-400`, `deferredRevealed` → `text-sky-400`, `neutral` → `text-white/40` (today's grey).
- **Legend**: compact 4-row block at the top of the Tools tab — 8px dot + 10px label: *Granted /
  Deferred · hidden / Deferred · revealed / Not granted*. Static JSX above the group list;
  `mb-3 border-b border-white/10 pb-2 space-y-1` per the mockup.
- **Unchanged by this feature** (FR-012, confirmed by the mockup): name/description text, group
  headings, row layout, tab chrome, and the Skills/MCP panes.
- **The mockup's "Core" group (find_tools/find_agent)** maps to ADR-2: it illustrates the *neutral*
  treatment of discovery tools, but those rows do not exist in the live manifest today, so no code
  renders them — the classifier's `isDiscovery` branch covers the contract without adding rows.
- **Upgrade path (not designed in, noted per instruction)**: if accessibility review later demands a
  non-color cue, the one-line change is a thin 2px left-edge swatch on each row (a `border-l-2` with the
  state color) added alongside the icon color — no change to the classifier, data flow, or any other
  section of this design.
