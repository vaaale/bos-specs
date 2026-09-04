# Implementation Plan: Tool State Color Coding in the Assistant Info Panel

**Branch**: `bos/042-tool-color-coding` | **Date**: 2026-09-02 | **Spec**: `/Specs/user-specs/assistant/042-tool-color-coding/spec.md` | **Design**: `/Specs/user-specs/assistant/042-tool-color-coding/design.md` | **UI contract**: `/Specs/user-specs/assistant/042-tool-color-coding/mockup.html`

## Summary

Add state color coding to the rows in the Assistant Info Panel's **Tools** tab. Each tool row is colored by its state relative to the currently selected agent and the active conversation: **green** (granted, non-deferred), **orange / dark yellow** (granted, deferred, not yet discovered), **blue** (granted, deferred, discovered this conversation), **neutral** (the existing grey — not granted, and discovery tools). A compact legend at the top of the tab explains the colors.

Technical approach: make the panel's `ToolsTab` **agent- and conversation-aware** and classify each row with a new **pure, framework-free** helper `classifyToolState`. All three inputs come from the *same single sources the run uses* — the selected agent's `tools`/`deferredTools` (already returned by `GET /api/assistant/agent`, which the panel fetches today) and the conversation's revealed set (`deriveRevealedIds` from the client-safe `src/lib/assistant/messages.ts`, computed over the chat store's transcript) — so a row's color can never disagree with what the agent can actually call. No new API route, no new fetch, no server-side change.

## Technical Context

**Language/Version**: TypeScript (BOS's existing Next.js/React toolchain; no version bump)

**Primary Dependencies**: React, Tailwind CSS, lucide-react (all already in use by `InfoPanelV2.tsx`). No new runtime dependencies. The new module `src/lib/agent/tool-state.ts` is **framework-free** (no React, no `server-only`), so it imports into client, server, and tests alike.

**Storage**: N/A — no persisted state. The classification is recomputed fresh each render from in-memory inputs (agent record + transcript); nothing is written.

**Testing**: **Hand-run unit tests, no test runner** (BOS convention — no `vitest`/`jest`/`playwright` is wired into `package.json` for unit tests; see the header of `src/lib/assistant/__tests__/inner-loop.test.ts`). The new test file exports `test*()` functions plus a `runAll()`, using a local `assert` helper. The SC-001 test is a **predicate-identity check**: for every (allow, deferred, revealed) combination, `classifyToolState` returns a *callable* state (`granted` or `deferredRevealed`) if and only if the run's `visibleTools` predicate (`allow.has(id) && !(deferred.has(id) && !revealed.has(id))`) admits the tool. See `Testing` below.

**Target Platform**: BOS client bundle (`"use client"` React subtree) + shared framework-free module.

**Project Type**: Web application (Next.js App Router) — a change to an existing component tree plus one new shared module.

**Performance Goals**: The `useChatSelector(conversationId, s => s.messages)` subscription is scoped to `ToolsTab` (mounted only while the Tools tab is active) so the memoized `AssistantChatV2` is never re-rendered per streamed token; `ToolsTab` re-renders only on a genuine transcript append, agent change, or conversation change. (design §3, ADR-3.)

**Constraints**:
- The four row colors MUST be distinguishable on the `#0f1117` dark theme (fixed by `mockup.html`: `text-emerald-400` / `text-amber-400` / `text-sky-400` / `text-white/40`).
- The color cue sits **only on the wrench icon** — tool name/description text, group headings, row layout, tab chrome, and the Skills/MCP tabs are unchanged (FR-009, FR-012).
- Strict allowlist semantics: `null` (agent not yet fetched) and `[]` (fetched, empty) both render neutral, but via different code paths; the classifier itself is strict `allow.has`, NOT the lenient `allows()` helper the Skills/MCP tabs use (design §3 "Loading").
- Do **not** import the server-only `deriveRevealedIds` from `src/lib/agent/tool-gate.ts`; use the client-safe one from `src/lib/assistant/messages.ts` (design §3 — the load-bearing disambiguation).
- Do **not** add a "Core" group or rows for `find_tools`/`find_agent` — they are not in the capability registry and render no rows today; the classifier's `isDiscovery` branch is defensive-only (design ADR-2).

**Scale/Scope**: One feature; ~1 new framework-free module, 1 new test file, edits to 2 existing component files. No new routes, services, ports, or windows.

## Constitution Check

*Gates from `/Specs/bos-system-specs/.specify/memory/constitution.md`.*

| Principle | Status | Notes |
|---|---|---|
| **I. Spec-Driven** | ✅ PASS | `spec.md` exists; pipeline specify → design → plan followed. |
| **II. Server Authority & SSR Boundary** | ✅ PASS | The client derives the revealed set with `deriveRevealedIds` (`src/lib/assistant/messages.ts` — verified framework-free, zero imports) over the client's own `ChatMessage[]` transcript, which the server already delivers to this client. No secret, Node API, or fs access; no new route. (design ADR-1.) |
| **III. Always Delegate; Claude Codes** | ✅ PASS | `implement` delegates to the Developer (Claude) sub-agent via `dev_delegate`; Build Studio writes no source. |
| **IV. Minimize Blast Radius** | ✅ PASS | Active feature branch `bos/042-tool-color-coding`; developer edits an isolated worktree; no secrets/lockfile/build changes. |
| **V. VFS Is Not the Source** | ✅ PASS | Spec/plan authored in VFS; all `src/` edits are by the Developer on the branch. |
| **VI. Specs & Docs Stay in Sync** | ⚠️ **ACTION** | The Assistant end-user doc must note the Tools tab is now agent-/conversation-aware with the four-state colors. Carried as an explicit task (T-DOC) and verified at `converge`. If any code/spec divergence is found, record it in `/Specs/user-specs/discrepancies.md`. |
| **VII. Respect Boundaries** | ✅ PASS | No changes to `package.json`, lockfiles, or build config. `npx tsc --noEmit` and `npm run lint` must pass for changed files; do not run `npm run build` while `next dev` is up. |

**GATE: PASS** (no unjustified violations). No `Complexity Tracking` entry is required — nothing here violates a principle.

## Project Structure

### Documentation (this feature)

```text
/Specs/user-specs/assistant/042-tool-color-coding/
├── spec.md      # Feature specification (authoritative)
├── design.md    # Structural design + ADRs (written by architect, reviewed)
├── mockup.html  # UI contract (written by ui-designer)
├── plan.md      # This file
└── tasks.md     # Created in the tasks step (NOT by plan)
```

No `research.md` / `data-model.md` / `contracts/` — the feature is a single-project, no-new-storage, no-external-interface UI change; `design.md` already carries the resolved unknowns (the `deriveRevealedIds` disambiguation, the revealed-set transport, and the classifier contract).

### Source Code (repository root — created/modified at `implement`)

```text
src/
├── lib/agent/
│   ├── tool-state.ts                    # NEW — framework-free: ToolState, DISCOVERY_TOOL_IDS, classifyToolState
│   └── __tests__/
│       └── tool-state.test.ts           # NEW — hand-run unit tests incl. the SC-001 predicate-identity check
└── components/agent/v2/
    ├── InfoPanelV2.tsx                  # MODIFY — capture tools/deferredTools into caps state; add conversationId prop; ToolsTab signature + legend + row coloring
    └── AssistantChatV2.tsx              # MODIFY — one line: pass conversationId to InfoPanelV2
```

**Structure Decision**: single-project change, matching the existing layout. The new classifier lives beside the other framework-free agent modules under `src/lib/agent/` (next to `tool-manifest.ts`), so the client, the server, and the test all import the same symbol — the basis of SC-001. The two component edits are the minimal wiring. Integration points that are **deliberately NOT changed**: `src/lib/assistant/messages.ts`, `src/lib/assistant/tools.ts`, `src/lib/assistant/gate.ts`, `src/lib/agent/tool-manifest.ts`, `src/lib/agent/tool-gate.ts`, and `src/app/api/assistant/agent/route.ts` (design §5).

## Design Notes (folded from `design.md`)

**Classification**: `bos-core` — agrees with `spec.md`. A change to the already-compiled `InfoPanelV2` component plus one new framework-free module and a test.

**The classifier contract** (predicate order is normative — design §4):

```
classifyToolState(toolId, { allow, deferred, revealed, isDiscovery }):
  1. isDiscovery                        -> "neutral"
  2. !allow.has(toolId)                 -> "neutral"
  3. deferred.has(toolId) && revealed   -> "deferredRevealed"   // blue
  4. deferred.has(toolId)               -> "deferredHidden"     // orange
  5. else                               -> "granted"            // green
```

The color-state set (`granted` ∪ `deferredRevealed`) is exactly the run's `visibleTools` admission predicate, so the SC-001 test reduces to a single boolean identity.

**ADRs** (full rationale in `design.md` §6):
- **ADR-1** — derive the revealed set **client-side from the chat store** (`useChatSelector` + `deriveRevealedIds`), not via a new API route: zero new surface, inherently in sync (same pure function + same data), reactive for free.
- **ADR-2** — keep the classifier pure; `DISCOVERY_TOOL_IDS` is a framework-free mirror passed in as `isDiscovery` (defensive-only; discovery tools render no rows today).
- **ADR-3** — scope the `s.messages` subscription to `ToolsTab` (mounted only on the Tools tab) so the memoized `AssistantChatV2` never re-renders per streamed token.

**Reactivity** (design §3): agent change → `InfoPanelV2`'s existing `useEffect([agentId])` refetches → `caps.tools`/`caps.deferredTools` change → reclassify. Conversation change / in-conversation reveal → `useChatSelector` re-binds or `s.messages` reference changes → revealed set changes → orange↔blue.

**UI** (binding `mockup.html`): wrench-icon color only; palette `text-emerald-400` / `text-amber-400` / `text-sky-400` / `text-white/40`; 4-row legend at the top of the Tools tab. Accessibility upgrade path (thin 2px left-edge swatch) is **noted, not designed in**.

## Testing

Follow the repo's hand-run test convention (no runner). `src/lib/agent/__tests__/tool-state.test.ts`:

- **SC-001 (predicate identity — the core test)**: for a matrix of (allow, deferred, revealed, isDiscovery) cases, assert `classifyToolState` returns a *callable* state iff `allow.has(id) && !(deferred.has(id) && !revealed.has(id))` — i.e. identical to `visibleTools`' admission. This is the canary that the panel and the run can never diverge (design risk 1; it also pins the use of the `messages.ts`-shaped revealed set).
- **Edge-case table** (design §4): empty allowlist → all neutral; revealed-but-not-granted → neutral; revealed + non-deferred + granted → green (never blue); `isDiscovery` → neutral; no active conversation (`revealed = ∅`) → granted non-deferred green, granted deferred orange.
- **Loading strictness**: with `allow = null` (not yet fetched) every state is neutral — assert the classifier/caller path never treats `null` as "allow all" (guards against the `allows()`-style spurious green flash).

Component behavior (agent switch SC-002, in-conversation reveal SC-003, per-conversation revert) is exercised in `converge` against the live Assistant, since the repo has no React component test harness.

## Out of Scope (carried from spec)

Informational only — does not change which tools the agent can call or how deferral/revealing works. No surface/elicitation-tool rows, no new "Core" group, no colorblind swatch, and no change to the Skills/MCP tabs or group layout.
