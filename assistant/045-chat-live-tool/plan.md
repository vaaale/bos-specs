# Implementation Plan: Live Per-Tool Updates & Rich Recursive Tool-Call Rendering

**Branch**: `bos/045-chat-live-tool` | **Date**: 2026-09-23 | **Spec**: `spec.md` | **Design**: `design.md`

**Input**: Feature specification from `spec.md` (status: Specified — design complete).

## Summary

Three-part rework of the v2 assistant chat's tool-call handling, all built on primitives that
already exist per-call:

1. **US1 (P1, backend)** — Decouple live `tool_result` emission (**completion order**) from
   persistence (`message` + `saveMessages`, **original call order**) in the run loop, so each
   parallel-safe call flips done individually on its own settle while the persisted transcript
   stays deterministic.
2. **US2 (P1, backend)** — Forward nested tool **results** (local/ephemeral/surface path)
   through the existing per-call `tool_progress` channel; the Claude/OpenCode CLI path stays
   **starts-only** (the harness never emits per-tool `tool_result`).
3. **US3 (P2, frontend)** — Rework `ToolCallCard.tsx` into a single **recursive** collapsible
   component (header action summary; independently-collapsible Input / Output; content-type
   Output rendering; child cards recurse), per the binding mockup at `mockup.html`.

The full structural design, the three ADRs, the file/module plan, integration points, and risks
are in `design.md` — this plan references that rather than repeating it.

## Technical Context

**Language/Version**: TypeScript (Next.js App Router, React 18, Zustand).

**Primary Dependencies**: existing run-event stream (NDJSON/SSE), `RunManager.emit`,
`applyRunEvent`/`ToolCallView` (client store), `encodeNested`/`parseNested`,
`ToolContext.onEvent`, `card-collapse.ts`, `parseMcpUi`, the chat markdown renderer
(`ChatMarkdown`/`MarkdownRenderers`, Prism code blocks). No new dependency.

**Storage**: conversation messages persisted as files under `./data` (existing
`io.saveMessages`); the run log under `./data/assistant/runs`. No new store.

**Testing**: Playwright e2e for the run/loop behavior (deterministic-replay invariant, per-settle
completion, cancellation) and the card recursion; component/unit tests for the content-type
renderer, summary resolver, and the widened `NestedEvent` round-trip.

**Target Platform**: Next.js server (run loop, delegation) + browser (chat card).

**Project Type**: in-place modification of the existing `src/` tree — no new app, no new package.

**Performance Goals**: card stays responsive for deep/wide nested trees (per-section scroll clamp
bounds height at any depth); live nested streaming re-arms the idle timeout (existing behavior).

**Constraints** (from `spec.md` FR-007 — hard): no new run-event type, no transport/protocol
change, no client-store schema change, no CopilotKit. `run-events.ts`, `chat-store.ts` schema,
`run-manager.ts`, `run-client.ts`, `Message.tsx`, CopilotKit are all **not-touched**.

**Scale/Scope**: 3 new files, 6 modified files (see Project Structure). One shared card component
rendered at every nesting depth.

## Constitution Check

*GATE: must pass before implementation. Re-checked against the ratified constitution (v1.0.0).*

| Principle | Status | Note |
|---|---|---|
| **I. Spec-Driven (Non-negotiable)** | ✅ Pass | `spec.md` agreed and design-complete; this plan precedes `tasks`/`implement`. Code is a derivative of the spec. |
| **II. Server Authority & SSR Boundary** | ✅ Pass | Backend changes stay in server code (`agent-loop.ts`, `inner-loop.ts`, `delegate-common.ts`); client renders over the existing `fetch`-based NDJSON stream. No secrets/Node leak, no API key returned. |
| **III. Always Delegate; Claude Codes** | ✅ Pass | `implement` delegates to the Developer (Claude) sub-agent via `dev_delegate`; Build Studio writes no source. |
| **IV. Minimize Blast Radius (Non-negotiable)** | ✅ Pass | Feature branch `bos/045-chat-live-tool` already active; the developer edits an isolated worktree; the candidate self-tests before promote. |
| **V. VFS Is Not the Source** | ✅ Pass | Spec/plan/design/mockup are VFS artifacts; all code changes go through `src/` via the Developer — never via VFS file tools. |
| **VI. Specs & Docs Stay in Sync (Non-negotiable)** | ⚠️ Action required | This change alters **user-visible** behavior (chat tool-call cards) **and** developer-facing internals (run-loop emission order, inner-loop forwarding, `NestedEvent` payload shape). The `implement` change MUST update `docs/usage` (tool-call rendering) and `docs/dev` (agent-loop/inner-loop/nested-events) in the same change. See Project Structure. |
| **VII. Respect Boundaries** | ✅ Pass | No `package.json`/lockfile/secret/build-config changes; `npx tsc --noEmit` and `npm run lint` must pass for changed code; no `npm run build` while `next dev` is running. |

**Gate result: PASS**, with one **action item** (VI — docs sync) carried into `tasks`. No
Complexity Tracking is required (no unjustified violations).

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/assistant/045-chat-live-tool/
├── spec.md          # Agreed feature spec (source of truth)
├── design.md        # Structural design + ADRs + file plan (this step's predecessor)
├── plan.md          # This file
├── mockup.html      # Binding UI contract for US3
└── tasks.md         # Next step (NOT created by plan)
```

No `research.md`, `data-model.md`, or `contracts/` are warranted: the design already resolved the
mechanism unknowns against real source (no open NEEDS CLARIFICATION), the feature has no external
interface (it reuses the existing run-event stream — no new API route, protocol, or event type),
and the "data model" change is confined to the already-documented `NestedEvent` payload shape
(design ADR-3 / B1).

### Source code (repository root) — what `implement` creates/modifies

```text
src/
├── lib/assistant/
│   ├── agent-loop.ts            # [MODIFY] US1 — decouple per-settle tool_result emission from original-order message+saveMessages; cancel de-dup; per-call .catch isolation
│   ├── inner-loop.ts            # [MODIFY] US2 — forward nested tool_result/tool_cancelled via ctx.onEvent; carry per-callId result+status list on InnerLoopResult
│   └── tools/server/
│       └── delegate-common.ts   # [MODIFY] US2 — pass the per-child result list into the terminal encodeNested call (local path)
├── lib/agent/
│   └── nested-events.ts         # [MODIFY] US2/US3 — widen NestedEvent { result?, status?, nested? }; encodeNested/parseNested shape change (marker unchanged); export type
└── components/agent/v2/
    ├── ToolCallCard.tsx         # [MODIFY] US3 — recursive single component: header summary + collapsible Input/Output + content-type Output + child cards
    ├── MessageListV2.tsx        # [MODIFY] US3 — per-card header scope; pass live progress + resultText to the card
    ├── ToolCardSummary.ts       # [NEW]    US3 — per-tool action summary (FR-012); tool-name fallback
    ├── ToolCardSections.tsx     # [NEW]    US3 — Input (key-value + raw toggle) + Output (content-type) collapsible sections
    └── ToolCardSectionState.ts  # [NEW]    US3 — card-local per-callId Input/Output open state (ADR-2 option B)

# docs — Constitution VI (same change as code)
docs/
├── usage/<chat|assistant>/…     # [UPDATE] tool-call rendering: recursive card, per-tool completion, live output
└── dev/<assistant>/…            # [UPDATE] agent-loop emission order, inner-loop forwarding, NestedEvent payload shape

# e2e — regression coverage (see tasks)
e2e/
└── 045-chat-live-tool.spec.ts   # [NEW] deterministic-replay invariant; per-settle completion; cancellation; card recursion (Playwright)
```

**Structure Decision**: in-place modification of the existing tree, mirroring the exact layout
`design.md` captured from real source. Three new component files sit beside the card they extend;
the two backend legs stay in their existing modules. No new package, app folder, or service.

## Design Notes

The three decisions that bound the implementation (full rationale + alternatives in `design.md`):

- **ADR-1 — Decouple live emission from persistence.** Live `tool_result` fires per-settle
  (completion order, drives the UI); `message` + `saveMessages` stay in the trailing original-order
  loop (drives deterministic replay). Cancelled calls emit `tool_cancelled` exactly once at settle;
  the trailing loop only *persists* them. The per-call `.catch` isolation and `tool_use`/`tool_result`
  pairing the next model turn depends on are preserved.
- **ADR-2 — Card-local section state, not a shared-registry extension.** The header accordion
  still uses `card-collapse.ts` (scoped per card); Input/Output use a new per-callId local map so
  the shared single-open accordion (also used by reasoning cards) is not at risk.
- **ADR-3 — Live rides `tool_progress`; terminal payload widened.** `NestedEvent` gains
  `result?`/`status?`/`nested?` so a done delegation's child-card tree rebuilds from the persisted
  string alone on reload (the B1 fix). Claude/OpenCode stays starts-only.

**Constitution VI follow-on**: `tasks.md` MUST include documentation updates (usage + dev) and the
e2e regression spec alongside the code, so the change is complete per the Development Workflow
gate (spec + docs updated, typecheck + lint pass, self-tests pass).

## Done When

- [x] Plan workflow executed; design folded in; no research/data-model/contracts warranted (justified above).
- [ ] Constitution gate passed (VI action item carried to tasks).
- [ ] Completion reported → next step is `tasks`.
