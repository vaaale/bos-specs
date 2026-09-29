# Implementation Plan: Log Viewer — Shared Expandable Log Record Card

**Branch**: `bos/039-log-viewer-card` (spec feature id `039-log-viewer-card`) | **Date**: 2026-09-29 | **Spec**: [`spec.md`](./spec.md)

**Input**: Feature specification from `/Specs/user-specs/core-platform/039-log-viewer-card/spec.md`, approved by @alex (note #100 "Looks good! Proceed", 2026-09-29) at the `Design Complete` gate.

**Note**: Technical design lives in [`design.md`](./design.md) and the static UI mockup in [`mockup.html`](./mockup.html) — this plan references them rather than repeating them.

## Summary

Add one **shared, presentational** `LogRecordCard` component that renders a full, structured `LogRecord`, and use it as the **expanded body** of a log row in **both** the toolbar `RecentLogPopover` (`VersionControls.tsx`) and the Settings → Logs timeline (`LogsTab.tsx`). Rows become native `<details>` (multi-open + accessible for free), keyed by a **stable record id** so an open card survives the 4 s re-sort/poll. The toolbar list grows `max-h-[340px] → max-h-[440px]`; the summary line is **unchanged** on each surface. `copyToClipboard` is exported from `useLogContextMenu` so the card's *Copy JSON* shares the context menu's code path. **Pure client-side** — no backend, no `/api/logs`, no Supervisor change.

## Technical Context

**Language/Version**: TypeScript (BOS repo), React 18 (client components, `"use client"`).

**Primary Dependencies**: None new. Existing: `react`, `lucide-react` (icons already in use), Tailwind (inline class strings per the style guide — **no** component library).

**Storage**: N/A (the card renders `LogRecord`s already fetched from `GET /api/logs`; no persistence, no store).

**Testing**: Playwright e2e (BOS self-test convention — `e2e/<feature-id>.spec.ts`); component logic is trivially pure so a small Node-based unit check of `logRowId` is acceptable. TDD per the approval.

**Target Platform**: Next.js (App Router) desktop web surface; dark theme.

**Project Type**: `bos-core` (client-side component change).

**Performance Goals**: No new work — the card is a leaf render of data already in memory; must not add a fetch or a re-render trigger. The `data`/`err.stack` regions are height-capped + scrollable so a multi-KB payload cannot stretch the panel.

**Constraints**:
- Reuse the **canonical** `LogRecord` type from `src/lib/logging/models/log-models.ts`; do not introduce a third local record type.
- Built entirely from tokens the two surfaces already use (`border-white/10`, `bg-black/35`, `font-mono`, `text-white/40` labels, `LEVEL_COLOR`/`STREAM_COLOR`) — no new palette.
- Summary markup stays **per-surface and unchanged** (toolbar: one-line, no ellipsis; Settings: `truncate`).
- `buildLog` renders as monospace **text**, not a hyperlink (no client route serves the blob — a serving route is an out-of-scope follow-up).

**Scale/Scope**: 2 modified files, 1–2 new files, 1 one-line export. ~3 screens of surface, 1 shared component.

## Constitution Check

*GATE: Must pass before implementation. Re-checked after design.*

| Principle | Status |
|---|---|
| **I. Spec-Driven** | ✅ Approved `spec.md` + this plan + `tasks.md` precede implementation. |
| **II. Server Authority & SSR Boundary** | ✅ No server code touched; all fields already returned by `/api/logs`. |
| **III. Always Delegate; Claude Codes** | ✅ Implementation delegated to the `developer` (Claude) agent on the active feature branch. |
| **IV. Minimize Blast Radius** | ✅ On feature branch `bos/039-log-viewer-card`; isolated worktree; not the running tree. |
| **V. The VFS Is Not the Source** | ✅ Source edits happen in the Developer's `src/` worktree, never via VFS file tools. |
| **VI. Specs & Docs Stay in Sync** | ✅ `spec.md` already updated for the approved design; Developer updates `docs/usage` (Settings → Logs / toolbar) to note in-place expansion in the same change. No code/spec drift expected. |
| **VII. Respect Boundaries** | ✅ No changes to secrets/`package.json`/lockfiles/build config. Quality gates `npx tsc --noEmit` + `npm run lint` must pass for changed code. |

**Result: PASS — no violations, no Complexity Tracking required.**

## Project Structure

### Documentation (this feature)

```text
/Specs/user-specs/core-platform/039-log-viewer-card/
├── spec.md         # Approved specification (source of truth)
├── design.md       # Technical design (components, data-flow, decisions D1–D7)
├── mockup.html     # Static UI mockup of both surfaces
├── plan.md         # This file
└── tasks.md        # Dependency-ordered task list (TDD, per user story)
```

No `research.md` / `data-model.md` / `contracts/` are warranted: there are no open unknowns (the design resolved every one against source), no new data model (the card renders the existing canonical `LogRecord` read-only), and no external interface (no API surface change).

### Source Code (repository root) — the scope contract

> **This file list is the scope contract.** The Developer edits **only** these paths. Nothing outside this list without coming back. `docs/usage` may be touched for the §VI sync obligation.

```text
src/
├── components/
│   ├── logging/
│   │   ├── LogRecordCard.tsx      # NEW — shared presentational card (canonical LogRecord prop)
│   │   ├── logRowId.ts            # NEW (tiny) — stable-key helper logRowId(r)  [D6: may be colocated in LogRecordCard.tsx instead]
│   │   └── useLogContextMenu.tsx  # MODIFY — `export` copyToClipboard (one line; body unchanged)
│   ├── desktop/
│   │   └── VersionControls.tsx    # MODIFY — RecentLogPopover rows div→<details>, stable key, card body, max-h 340→440, import canonical type
│   └── apps/settings/
│       └── LogsTab.tsx            # MODIFY — replace inline body <div> with <LogRecordCard>, stable key, import canonical type
└── lib/logging/models/
    └── log-models.ts              # READ-ONLY — canonical LogRecord type; NOT modified

# Tests (TDD)
e2e/039-log-viewer-card.spec.ts    # NEW (optional but preferred) — Playwright self-test of expand/copy/multi-open/stable-open
```

**Structure Decision**: Single-project. The change is a leaf component plus two call-sites; it lands exactly where the two existing log surfaces live and shares the canonical type from `src/lib/logging/models/`. No new directory beyond the two small files in the existing `src/components/logging/` folder.

### Grounding notes (verified against source on this branch)

These are facts the Developer can rely on (read-only context, not new requirements):

- **Canonical type** (`log-models.ts`): `LogRecord { ts:number; level:LogLevel; stream:LogStream; component:string; conversation?:string; msg:string; data?:unknown; err?:LogErr{message:string;stack?:string}; branch?:string; sessionId?:string; versionLabel?:string; rxts?:number; buildLog?:string }`.
- **Toolbar (`VersionControls.tsx`)** today: local lossy `interface LogRecord`; rows are `<div key={i}>` (no left-click; right-click `onContextMenu → openMenu`); list `max-h-[340px] overflow-auto ... bg-black/35`; width `w-[900px] max-w-[92vw]`. Migration to the canonical type means **dropping `r.message` references** (e.g. `r.msg ?? r.message` → `r.msg`; `r.level ?? "info"` / `r.ts ? …` guards become redundant since `level`/`ts` are required) — the `recentErrors` polling can keep reading the canonical type. `LEVEL_COLOR` stays local to the toolbar (summaries unchanged).
- **Settings (`LogsTab.tsx`)** today: local `interface LogRec`; rows already `<details key={i}><summary>`; the expanded body is a hand-rolled `<div>` (`session/conversation/branch/err/data/buildLog`); `safeJson(v)` is a **local (non-exported)** function; `STREAM_COLOR` map lives here. Migration: body → `<LogRecordCard>`; import canonical type; `STREAM_COLOR` moves into the card (D3) so both surfaces share one badge.
- **`useLogContextMenu.tsx`**: `copyToClipboard(text)` is **private**; `openMenu(e, record: unknown)` accepts `unknown`, so passing the canonical `LogRecord` needs no change; `menuNode` renders the *Copy* button calling `copyToClipboard(JSON.stringify(menu.record, null, 2))`.
- **Card pretty-printer**: `safeJson` is not shared; the card implements the same shape locally (string passthrough, else `JSON.stringify(v, null, 2)`, `String(v)` on throw) — decision D4.

## Complexity Tracking

None — Constitution Check passed with no violations.
