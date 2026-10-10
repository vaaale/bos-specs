# Design — Log Viewer: Shared Expandable Log Record Card

**Feature**: `039-log-viewer-card` · branch `bos/039-log-viewer-card`
**Spec**: `spec.md` (same dir) · **Mockup**: `mockup.html` (same dir)
**Status**: Draft

> This is the structural/technical design for the approved spec. It names the
> components, the data-flow decisions, and the integration points against the
> **real** current code. It intentionally does not restate the spec's
> user-facing requirements — those live in `spec.md`.

## Classification

- **App Target**: `bos-core` — a presentational change to two existing
  client components plus one new shared component and one small export.
- **Mechanism**: pure React + Tailwind (per the style guide, no component
  library, inline class strings). No backend, no store, no route changes.
- **Risk**: low. Blast radius is two files that both already render `LogRecord`
  lists; the new component is a leaf. The one structural risk is the
  **record-type mismatch** between the two files (below), which this design
  resolves explicitly.

## Current state (grounded in source)

Both surfaces read the same `GET /api/logs` endpoint and render a newest-first
list, but they model and render records differently:

| | Toolbar `RecentLogPopover` | Settings `LogsTab` |
|---|---|---|
| File | `src/components/desktop/VersionControls.tsx` | `src/components/apps/settings/LogsTab.tsx` |
| Row element | `<div key={i}>` — **not** clickable (left-click), right-click → Copy only | `<details key={i}><summary>` — **already** expandable, multi-open, accessible |
| Record type | local `interface LogRecord` — a **lossy subset**: `ts, level, stream, component, source, msg, message, err?: {message}` (no `data`, `err.stack`, `branch`, `versionLabel`, `sessionId`, `conversation`, `buildLog`) | local `interface LogRec` — a **superset** that *does* carry `data`, `err.stack`, `branch`, `versionLabel`, `sessionId`, `conversation`, `buildLog` |
| Expanded detail | — (none) | an **inline** `<div>` hand-rolled body (session/conversation/branch/err/data/buildLog, plain `text-white/60`), `pl-4` |
| Copy | `useLogContextMenu().openMenu` (shared hook) | same |
| Key | `key={i}` (index) | `key={i}` (index) |

**The canonical record type** is `src/lib/logging/models/log-models.ts` →
`export interface LogRecord` (with `LogLevel`, `LogStream`, `LogErr`). Both
files currently declare their *own* local types instead of importing it. The
toolbar's local type is lossy enough that, as written, the toolbar **cannot
render the full record even if it wanted to** — the fields are dropped at the
type level.

**Clipboard**: `copyToClipboard` is a **private** (non-exported) function inside
`src/components/logging/useLogContextMenu.tsx`, called only by that file's
menu.

**Build log**: `buildLog` is a relative blob path (`logs/builds/<branch>-<ts>.log`).
There is **no client route** that serves these blobs (confirmed: no
`/api/.../buildLog` or build-blob route exists). Both surfaces render it as
plain monospace/amber text today.

## The design

### 1. One shared presentational component: `LogRecordCard`

**New file**: `src/components/logging/LogRecordCard.tsx`

```ts
"use client";
import type { LogRecord } from "@/lib/logging/models/log-models";

export function LogRecordCard({ record }: { record: LogRecord }) { ... }
```

- Takes **the canonical `LogRecord`** (from `log-models.ts`), not a local copy.
  This is the single most important decision: it removes the two divergent
  local types' influence on the detail body, so toolbar and Settings render
  byte-identical detail from one render site (spec SC-004).
- **Leaf, presentational, no state of its own.** No fetch, no store, no
  polling. It renders a record it's handed. That keeps it trivially safe to use
  in both surfaces and makes it unit-testable in isolation.
- **Omission rule**: every optional section is conditionally rendered on the
  field's presence (`record.err && …`, `record.data !== undefined && …`, etc.).
  Nothing renders as `null`/empty (spec FR-006).

#### Card layout (field order)

A two-column `grid` (label / value), matching the style guide's field-row
recipe, with these rows in order — each rendered only if present:

1. **Message** — `record.msg`, `whitespace-pre-wrap break-words`, **not
   truncated** (spec FR-002).
2. **Error** — if `record.err`: `err.message` in a red tint, then `err.stack`
   in a `<pre>` that is `overflow-auto`, monospace, and **height-capped**
   (`max-h-[200px]`) so a huge stack scrolls instead of stretching the panel
   (spec FR-003). When there's no `stack`, only `message` shows.
3. **Stream** — a badge using the Settings `STREAM_COLOR` map (moved to the
   shared card; see decision D3).
4. **Component** — monospace `record.component`.
5. **Branch · Version** — `record.branch` and `record.versionLabel`,
   space-separated when either is present.
6. **Session · Conversation** — `record.sessionId` and `record.conversation`.
7. **Data** — `record.data` as pretty JSON in a `<pre>` that is `overflow-auto`
   and height-capped (`max-h-[160px]`). Reuses the same pretty-printer the
   Settings tab already has (see decision D4).
8. **Build log** — `record.buildLog` rendered as **monospace text** (not a
   hyperlink), matching current Settings behaviour, because no route serves the
   blob (see decision D5).

Footer row: a single **Copy JSON** button.

**Styling**: built entirely from tokens already used by the two files —
`border-white/10`, `bg-black/35` (the card body sits on the same deep surface
the lists already use), `font-mono` for values/paths, `text-white/40` for
labels, `LEVEL_COLOR`/`STREAM_COLOR` for the level/stream tints. The card is
the *body* of the row; the **summary line keeps each surface's existing
styling** (decision D2), so the card never changes how a closed row looks.

### 2. Row = native `<details>` wrapping the surface's summary + the card

Both surfaces render each record as:

```tsx
<details key={stableId(r)} onContextMenu={(e) => openMenu(e, r)} className="…surface row classes…">
  <summary className="…surface summary classes (UNCHANGED)…">…existing summary markup…</summary>
  <LogRecordCard record={r} />
</details>
```

- **Toolbar**: this is the real change — rows go `div` → `<details>`, and the
  existing one-line summary markup moves verbatim into `<summary>` (spec
  FR-007/FR-011, "keep it the way it is"). The summary keeps
  `onContextMenu` → `openMenu`, so right-click Copy is preserved.
- **Settings**: rows are *already* `<details>`; the only change is the **body**
  — replace the hand-rolled inline `<div>` with `<LogRecordCard record={r} />`
  (spec FR-001, the "migrate" decision). Summary markup unchanged.
- **Multi-open for free**: each `<details>` is independent; opening one never
  closes another (spec US2). No "only one open" logic anywhere.
- **Accessibility for free**: `<summary>` is natively focusable and toggles with
  Enter/Space and exposes the expanded state (spec US6). No custom keyboard
  handling. A rotating chevron (CSS `transform` on `[open]`) signals state.

### 3. Stable keys (the live-refresh fix)

Both files currently key rows by array index (`key={i}`) while re-sorting every
poll. Index keys reshuffle on re-sort, which (a) remounts rows and (b) snaps any
open `<details>` shut — the exact failure that would make the feature unusable.

**Fix**: key by a stable id derived from the record. `LogRecord` has no `id`
field, so derive a deterministic one:

```ts
function logRowId(r: LogRecord): string {
  // deterministic, collision-resistant within a 200/500-record window
  return `${r.ts}:${r.stream}:${r.component}:${(r.msg ?? "").slice(0, 40)}`;
}
```

- A small **pure helper** (`logRowId`) shared by both surfaces (colocated with
  `LogRecordCard` or in `log-models.ts` — see decision D6).
- With stable keys + native `<details>`, the **DOM owns the open state**, so an
  expanded card survives re-sort and new records arriving above it (spec US4),
  and **no React state tracks which rows are open** — when a record leaves the
  window its `<details>` unmounts and its open state goes with it (spec
  FR-009, US4 edge). This is why the earlier draft's "Set of expanded ids +
  pruning" is *dropped*: native `<details>` makes it unnecessary.

### 4. Toolbar sizing

- List scroll area: `max-h-[340px]` → `max-h-[440px]` (spec FR-010, the "a
  little" bump).
- Outer width **unchanged**: `w-[900px] max-w-[92vw]`.
- Settings panel **unchanged**: it's already `flex-1`/unconstrained.

### 5. Shared clipboard

`useLogContextMenu.tsx` currently keeps `copyToClipboard` private. Export it:

```ts
export function copyToClipboard(text: string): void { … }  // unchanged body
```

The card's **Copy JSON** calls `copyToClipboard(JSON.stringify(record, null, 2))`
— the identical call the context menu already makes — so "copy from the card"
and "right-click → Copy" produce byte-identical JSON through one code path
(spec FR-012, SC-001/US5). The context menu itself is untouched.

## File / module plan (what this feature creates or modifies)

| File | Change | Why |
|---|---|---|
| `src/components/logging/LogRecordCard.tsx` | **NEW** | The shared card (US1–US3, US5, US6); canonical `LogRecord` prop; pretty-printer + stream badge + Copy JSON |
| `src/components/logging/logRowId.ts` *(or colocated in `LogRecordCard.tsx` / `log-models.ts` — D6)* | **NEW (tiny)** | `logRowId(r)` stable-key helper shared by both surfaces |
| `src/components/desktop/VersionControls.tsx` | **MODIFIED** | Rows `div`→`<details>` + `<summary>` (summary markup unchanged); `key={i}`→`key={logRowId(r)}`; body = `<LogRecordCard>`; list `max-h-[340px]`→`[440px]`; drop the lossy local `LogRecord` in favour of the canonical type; keep errors-only, red badge, 15-min window, escape-close, right-click Copy |
| `src/components/apps/settings/LogsTab.tsx` | **MODIFIED** | Replace the inline body `<div>` with `<LogRecordCard>`; `key={i}`→`key={logRowId(r)}`; keep all filters, session picker, auto-refresh; drop the now-redundant local `LogRec` where the card replaces its use |
| `src/components/logging/useLogContextMenu.tsx` | **MODIFIED (one line)** | `export` `copyToClipboard` |
| `src/lib/logging/models/log-models.ts` | **UNCHANGED** (source of the canonical `LogRecord` type) | read-only dependency |

## Integration points (existing mechanisms this feature *calls into*, not recreates)

- **`GET /api/logs`** — the read endpoint (proxies to the Supervisor sink
  under `017-central-logging`). Consumed as-is; no change.
- **`LogRecord` type** (`log-models.ts`) — imported by the card and both
  surfaces; the single source of truth for the record shape.
- **`useLogContextMenu`** — right-click Copy stays; the card only reuses its
  exported `copyToClipboard`.
- **`STREAM_COLOR` / `LEVEL_COLOR`** — the Settings map is lifted into the card;
  the toolbar's `LEVEL_COLOR` stays local to its summary (summaries are
  unchanged). The card owns the stream badge styling for both surfaces.
- **Style-guide tokens** — opacity palette, `border-white/10`, `font-mono`,
  `rounded`, `text-xs` density (no new palette, no light-mode, no component
  lib).

## Decisions (and the alternatives rejected)

- **D1 — Canonical type, not local.** The card consumes
  `log-models.ts` `LogRecord`; both surfaces import it for the row data.
  *Rejected*: keeping each file's local type and mapping into the card — that
  re-introduces the divergence this feature exists to remove, and the toolbar's
  lossy local type would still starve the card of `data`/`err.stack`/etc.
- **D2 — Summary markup unchanged per surface.** Only the *body* becomes shared;
  each surface keeps its existing `<summary>` (toolbar: no ellipsis; Settings:
  `truncate`). *Rejected*: also unifying the summary — that violates the
  "keep it the way it is" decision and would be scope creep with no ask behind
  it.
- **D3 — `STREAM_COLOR` badge lives in the card.** The Settings stream-badge
  map is what the card renders. *Rejected*: passing a per-surface badge style —
  the whole point is one identical detail body.
- **D4 — One pretty-printer.** Reuse the Settings `safeJson` shape (string
  passthrough, else `JSON.stringify(v, null, 2)`, with a `String(v)` catch) for
  the card's `data`. *Rejected*: `JSON.stringify` only — `data` is sometimes a
  plain string and must not become `"\"...\""`.
- **D5 — `buildLog` is text, not a link.** No route serves the blob, so a
  hyperlink would 404. *Rejected*: building a new `/api/.../buildLog` route —
  that is a backend change and out of scope (spec FR-014). A clickable blob
  viewer is a candidate **follow-up** (needs a serving route), explicitly not
  part of this feature.
- **D6 — `logRowId` helper placement.** Default: colocate as a tiny exported
  helper next to `LogRecordCard` (`src/components/logging/`). *Alternative if
  the Developer prefers*: put it in `log-models.ts` beside the type. Either is
  one pure function; placement is the Developer's call at implement time.
- **D7 — No open-state React bookkeeping.** Native `<details>` owns it; stable
  keys preserve it across re-sort; unmount drops it. *Rejected*: the earlier
  draft's `Set<string>` of expanded ids + pruning logic — unnecessary and a
  source of the very bugs it tried to solve once `<details>` is used.

## Risks & mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Type narrowing across the two local→canonical migrations | Med | The card's prop type *is* the canonical `LogRecord`; both surfaces' row data must already satisfy it (Settings' `LogRec` does; the toolbar's lossy type is widened to the import). A `tsc --noEmit` pass catches any missing-field mismatch. |
| `<details>` inside a re-sorting list loses open state | Low | Stable keys prevent remount; verified by the SC-003 criterion (open survives 2+ polls). |
| A very large `data`/`err.stack` stretches the panel | Low | Both are height-capped `<pre>` with `overflow-auto` (SC-005). |
| Card styling drifts from the surface | Low | Card body uses the same `bg-black/35`/`border-white/10`/`font-mono` tokens the lists already use; summaries stay per-surface. |
| Copy JSON path diverges from context menu | Low | Single shared `copyToClipboard`; identical `JSON.stringify(record, null, 2)` (SC-001/US5). |

## Out of scope (mirrors spec, structural view)

- No backend route, no `/api/logs` change, no Supervisor change.
- No `buildLog` serving route (text-only for now; follow-up).
- No change to Settings filters / session picker / config panel, or toolbar
  errors-only / red badge / 15-min window / escape-close.
- No custom keyboard navigation beyond native `<details>` (arrow-key row
  movement not added).
- No new record fields.

## Open design questions

None that block a plan. Two are flagged for the Developer's judgement at
implement time, both already decided-with-default in D6/D5:
- exact `logRowId` placement (D6);
- whether to keep the toolbar's local `LogRecord` interface for its own
  `recentErrors` polling or import the canonical type there too (D1 leans
  "import the canonical type" — cleaner, and the polling only reads `ts`/`level`).
