# Feature Specification: Log Viewer — Expandable Log Record Card (Toolbar + Settings)

**Feature Branch**: `039-log-viewer-card`

**Created**: 2026-09-29

**Status**: Draft

**App Target**: bos-core

**Input**: GitLab issue #51 (Improvement): "I want to improve the Log view in the top toolbar. When clicking on a log item, the log messages should be expanded and the log message should be displayed in an appropriate 'Card'." — approved by @alex on 2026-09-29 (note #96), after two revision rounds that settled on: **multi-open** rows, a **shared card** extracted and applied in **both** the toolbar and Settings → Logs, a **small** size bump, summary rows **unchanged**, and native accessible controls that **fit the BOS theme**.

> Pairs with `017-central-logging` (the `LogRecord` schema and the `/api/logs` read endpoint this renders — the source of every field the card displays). This is a **pure client-side** change: `/api/logs` already returns the full `LogRecord`, so no backend, Supervisor, or endpoint work is in scope.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Expand a toolbar log row to read the full record (Priority: P1)

A developer debugging from the topbar clicks the **Log** button, sees the newest-first list, and clicks a row whose one-line summary is not enough. That row expands **in place** into a bordered **card** showing the complete, structured record — the full `msg` (word-wrapped, not truncated), `err.message` **and** the full `err.stack` in a scrollable monospace `<pre>`, and the fields the list currently throws away (`stream`, `component`, `branch`, `versionLabel`, `sessionId`, `conversation`, `data`, `buildLog`).

**Why this priority**: This is the exact ask in the issue and the core value — today the toolbar lets you *copy* a record but never *read* it; a full stack trace, the main debugging signal, is not rendered at all.

**Independent Test**: Open the toolbar Log popover, find a record with a long `err.stack`, click its row, and confirm the card renders the full record with the complete stack — no copy, no leaving the popover.

**Acceptance Scenarios**:

1. **Given** the toolbar Log popover is open showing records, **When** the user clicks a row, **Then** that row expands into a card below it showing the full structured record; the summary line for that row is preserved.
2. **Given** a record with an `err` (message + stack), **When** its card is open, **Then** `err.message` is shown and `err.stack` is shown in full in a monospace `<pre>`, scrollable, **not** limited to the last line.
3. **Given** a record with no `err`, **When** its card is open, **Then** the Error section is simply absent — no "null", no empty placeholder.
4. **Given** a record present in the store but missing optional fields (`branch`, `versionLabel`, `conversation`, `data`, `buildLog`), **When** its card is open, **Then** only the fields the record actually carries are shown.

---

### User Story 2 — Several rows open at once (Priority: P1)

The user opens more than one row to compare two records side by side (e.g. the `error` and the `info` that bracketed it). Each row toggles **independently**; opening one does not close another.

**Why this priority**: Settled explicitly during review (@alex, note #92: "Let's go with allowing to expand multiple lines at a time"). A single-open accordion would force closing one to read another.

**Independent Test**: Open two different rows and confirm both cards are visible simultaneously; toggle one closed and confirm the other stays open.

**Acceptance Scenarios**:

1. **Given** rows A and B are both collapsed, **When** the user opens A then opens B, **Then** both cards are visible at the same time.
2. **Given** rows A and B are both open, **When** the user clicks A's row again, **Then** A collapses and B remains open.
3. **Given** several cards are open and make the list tall, **When** the user scrolls, **Then** the list's scroll area is the primary scroller and cards stay full-width within it.

---

### User Story 3 — One shared card, in both the toolbar and Settings → Logs (Priority: P1)

The expanded detail is a single component (`LogRecordCard`) used by **both** the toolbar popover and the Settings → Logs timeline, so the two surfaces show the identical in-place detail and never diverge. Settings already expands rows inline with hand-rolled markup; that inline body is **migrated to the shared card**.

**Why this priority**: Settled explicitly during review (@alex, note #94 pt. 3: "Extract it so that it can be used in Settings->Logs too. In fact, let's apply it there as well with this change."). Consistency across the two surfaces is also the answer to the "fit the theme" request (note #94 pt. 5).

**Independent Test**: Open the same record's detail in the toolbar and in Settings → Logs and confirm the field layout and content are identical (only the summary line differs per surface, as today).

**Acceptance Scenarios**:

1. **Given** the shared `LogRecordCard` component, **When** it is rendered by the toolbar row and by a Settings → Logs row, **Then** the card body is identical in field set, order, and styling of the detail sections.
2. **Given** the Settings → Logs timeline's current inline expanded body, **When** this change lands, **Then** that inline body is replaced by `LogRecordCard` (no second, divergent detail markup remains).

---

### User Story 4 — An open card survives the live 4-second refresh (Priority: P2)

The toolbar list (and the auto-refreshing Settings timeline) re-sorts and re-renders on every poll. A row the user has expanded must **stay open and stay in place** for the same record across polls, rather than snapping shut.

**Why this priority**: Both files currently key rows by array index (`key={i}`); with a 4 s re-sort, index keys reshuffle and any expanded row closes the moment the list re-renders — which would make the whole feature unusable in practice.

**Independent Test**: Open a row, wait through at least two poll cycles (or trigger a new record that re-sorts the list), and confirm the open card is still open and still attached to the same record.

**Acceptance Scenarios**:

1. **Given** a row is expanded, **When** a poll re-sorts the list (new records arrive above it), **Then** the expanded card remains open and remains the card for the same record.
2. **Given** a record falls out of the visible window (older than the limit, or filtered out), **When** it unmounts, **Then** its card goes with it — no orphaned open state, no growth of an in-memory open set.
3. **Given** rows are keyed by a stable record id (derived when no id field is present), **When** the list re-renders with the same records, **Then** React/DOM reuses each row so the open state is preserved without any imperative bookkeeping.

---

### User Story 5 — Copy the record's JSON from the card (Priority: P2)

The user copies the full record as pretty JSON directly from the expanded card, reusing the same clipboard path the toolbar's right-click *Copy* already uses.

**Why this priority**: Preserves the one interaction the toolbar has today (right-click → Copy) and gives the card an equivalent action, so "copy" and "read in place" coexist.

**Independent Test**: Open a card, trigger its *Copy JSON* action, paste, and confirm the result is the record pretty-printed (consistent with the right-click *Copy* output).

**Acceptance Scenarios**:

1. **Given** a card is open, **When** the user triggers *Copy JSON*, **Then** the full record is copied to the clipboard as pretty-printed JSON, via the **same** helper the context menu uses (one shared `copyToClipboard`).
2. **Given** the toolbar right-click *Copy* on a summary row, **When** used, **Then** it still works and produces the same pretty JSON as the card's action.

---

### User Story 6 — Accessible, theme-consistent interaction (Priority: P3)

Rows are genuinely focusable and toggle with the keyboard, and the card is built from the styling tokens the two surfaces already use, so it reads as native BOS rather than a foreign widget.

**Why this priority**: Settled during review (@alex, note #94 pt. 5: "I trust your judgement on this. Just make sure it looks good and fits the theme of BOS"). Native `<details>` gives focusability and Enter/Space toggling for free.

**Independent Test**: Tab to a row, press Enter/Space to expand, confirm a screen reader announces the expanded/collapsed state, and confirm the card's colours and type match the surrounding surface.

**Acceptance Scenarios**:

1. **Given** a row, **When** the user tabs to it and presses Enter or Space, **Then** it expands/collapses without a mouse.
2. **Given** an open row, **When** inspected, **Then** it exposes the standard `aria-expanded`-equivalent state that native `<details>` provides.
3. **Given** the card, **When** rendered in either surface, **Then** it uses the existing BOS colour/type tokens (level colours, stream badges, `border-white/10`, `font-mono`, the surface's background treatment) rather than a new palette.

---

### Edge Cases

- **Absent optional fields** — a record carrying no `err`, `data`, `branch`, `versionLabel`, `conversation`, or `buildLog` shows only the fields it has; absent sections are omitted, never rendered as `null`/empty rows.
- **Huge `data` / `err.stack`** — both are rendered in height-capped, independently scrollable regions so one large record cannot blow up the popover or the Settings panel.
- **Several cards open at once** — the combined height is handled by the list's existing scroll area (toolbar) / the panel's `flex-1` scroll (Settings), not by growing the whole popover unbounded.
- **Record leaves the window** — when a record scrolls out of the (toolbar) 200-record limit or is filtered out, its `<details>` unmounts and its open state disappears with it; no leak, no pruning logic to maintain.
- **Live re-sort while open** — index-keyed rows must not close on re-render (see User Story 4); this is the reason for stable keys.
- **No Supervisor (dev fallback)** — the data still arrives as the same `LogRecord` shape via the existing `/api/logs` fallback; the card renders identically, so no branch-specific behaviour is introduced.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A new shared component `src/components/logging/LogRecordCard.tsx` MUST render a full, structured `LogRecord` as a bordered card, and MUST be the expanded detail for **both** the toolbar `RecentLogPopover` and the Settings → Logs timeline.
- **FR-002**: The card MUST render the record's `msg` in full, **word-wrapped and not truncated**.
- **FR-003**: When the record has an `err`, the card MUST show `err.message` and `err.stack`; `err.stack` MUST be rendered in a monospace, **scrollable, height-capped** region showing the **full** stack (not only the last line).
- **FR-004**: When the record has `data`, the card MUST render it as **pretty-printed JSON** in a **scrollable, height-capped** region so a single large payload cannot expand the container unbounded.
- **FR-005**: The card MUST render, **only when present**, the record's scalar/identity fields: `stream`, `component`, `branch`, `versionLabel`, `sessionId`, `conversation`, and the `buildLog` path (rendered as monospace **text** — no client route serves the blob yet, so it is not a hyperlink; a clickable blob viewer is an explicit follow-up) when present.
- **FR-006**: Every optional section MUST be omitted (not rendered as empty/`null`) when the corresponding field is absent on the record.
- **FR-007**: Each log row in **both** surfaces MUST be a **native `<details>` element** whose expanded body is `LogRecordCard`, so that (a) multiple rows can be open simultaneously, independently, and (b) rows are focusable and toggle with Enter/Space.
- **FR-008**: Rows in both surfaces MUST be keyed by a **stable record id** (falling back to a deterministic hash of `ts + stream + component + msg` when no id field is present) instead of the array index, so an expanded row survives the periodic re-sort/re-render.
- **FR-009**: Because open state lives in the DOM `<details>` (not React state), the feature MUST NOT maintain a growing in-memory set of expanded ids; an unmounted record's open state MUST disappear with it.
- **FR-010**: The toolbar popover's list scroll area MUST be enlarged from its current `max-h-[340px]` to `max-h-[440px]`; its outer width MUST remain `w-[900px] max-w-[92vw]`. The Settings → Logs timeline MUST remain unconstrained (`flex-1`).
- **FR-011**: The summary line for each row MUST be **unchanged** from today on each surface (toolbar: one-line, no ellipsis; Settings: existing `truncate`), per the review decision to "keep it the way it is."
- **FR-012**: The card MUST provide a **Copy JSON** action that reuses the existing clipboard helper (export `copyToClipboard` from `src/components/logging/useLogContextMenu.tsx`) so the card and the right-click *Copy* share one code path and produce identical pretty JSON.
- **FR-013**: The card MUST be styled from the tokens already used by the two files (level colours, stream badges, `border-white/10`, `font-mono`, the surface background treatment) so it reads as native to each surface; the right-hand chevron MUST rotate to indicate open/closed.
- **FR-014**: The change MUST be **client-side only**: it MUST NOT add a backend endpoint, modify `/api/logs`, or change the Supervisor log sink. All fields rendered MUST already exist on the `LogRecord` the endpoint returns.
- **FR-015**: The existing toolbar behaviour that is **out of scope** — *errors only* toggle, the red error badge, the 15-minute window on the Log button, escape-to-close, and right-click *Copy* — MUST be preserved unchanged.

### Key Entities

- **LogRecordCard** — the new shared presentational component; renders a `LogRecord` in full. Used by both surfaces as the expanded body of a row.
- **LogRecord** — the existing record schema (from `017-central-logging` FR-004 / `log-models.ts`): `ts`, `level`, `stream`, `sessionId?`, `versionLabel`, `component`, `conversation?`, `msg`, `data?`, `err?` (`message` + `stack`), `branch?`, plus `buildLog`/`rxts` where present. Rendered read-only by the card.
- **Expanded log row** — a `<details>` element wrapping the (unchanged) summary line and the `LogRecordCard` body.
- **copyToClipboard** — the existing clipboard helper (currently private to `useLogContextMenu`), to be **exported** so the card shares it.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Any log row in the toolbar can be expanded in place to show the full record — including a complete, scrollable `err.stack` — with no copy or navigation. (Measurable by expanding an error record and reading its full stack.)
- **SC-002**: At least two rows can be open at the same time, and toggling one does not affect the other. (Measurable by opening two rows.)
- **SC-003**: An expanded card remains open and attached to the same record across at least two 4-second polls, with no user action. (Measurable by opening a row and watching through refreshes.)
- **SC-004**: The expanded detail is produced by exactly **one** component (`LogRecordCard`) and is byte-for-byte the same markup in the toolbar and in Settings → Logs (only the summary line differs per surface). (Measurable by code inspection — a single render site; and by opening the same record in both.)
- **SC-005**: A single record with a very large `data` and/or `err.stack` does not grow the popover or Settings panel beyond their capped, scrollable regions. (Measurable by loading a record with a multi-KB stack/payload.)
- **SC-006**: A row can be expanded and collapsed with the keyboard alone (Tab to focus, Enter/Space to toggle). (Measurable without a mouse.)
- **SC-007**: The feature ships with no backend change — `/api/logs` and the Supervisor are untouched. (Measurable by the diff: only `VersionControls.tsx`, `LogsTab.tsx`, the new `LogRecordCard.tsx`, and an export in `useLogContextMenu.tsx`.)

## Assumptions

- **No backend change.** `/api/logs` (and the Supervisor read endpoint behind it) already returns the full `LogRecord`; the card renders only what the record already carries. Confirmed against `017-central-logging`.
- **"A little" size bump = `340px → 440px`.** This is the one number chosen on @alex's behalf in the review (their word was "increase the size a little"); the exact target is called out as the single adjustable class in case they want more/less.
- **Summary rows are unchanged.** Per note #94 pt. 4 ("Let's keep it the way it is"), the toolbar row stays a one-line summary with no added ellipsis and the Settings row keeps its existing `truncate`.
- **Native `<details>` is the interaction primitive.** It provides the required multi-open + accessibility (focus, Enter/Space, expanded state) without custom keyboard handling, and the DOM owns the open state — so no React `Set` of expanded ids and no pruning logic.
- **Scope is the two existing surfaces.** The card is extracted and applied in the toolbar and Settings → Logs (both in scope). It is *not* wired into any other surface in this change.
- **Theme via existing tokens.** "Fit the theme" is satisfied by reusing each surface's existing colour/type tokens and the shared card, rather than introducing a new design language.

## Out of scope

- No new backend endpoint, no change to `/api/logs`, no Supervisor log-sink change.
- No change to the toolbar *errors only* toggle, red error badge, 15-minute window, or escape-to-close.
- No change to Settings filters, session picker, or config panel.
- No new `LogRecord` fields — the card renders what the store already has.
- Keyboard navigation beyond what native `<details>` already provides (arrow-key moving between rows is not added).

## Notes

- **Related spec:** `017-central-logging` — the source of the `LogRecord` schema and the read endpoint.
- **Change surface (for the reviewer):** new `src/components/logging/LogRecordCard.tsx`; modified `src/components/desktop/VersionControls.tsx` (toolbar popover rows + sizing) and `src/components/apps/settings/LogsTab.tsx` (timeline rows); minor — export `copyToClipboard` from `src/components/logging/useLogContextMenu.tsx`.
- **Review decisions folded in** (@alex, notes #92 and #94): multi-open (native `<details>`), shared card applied in both surfaces, small toolbar size bump (340→440px), summary rows unchanged, accessible + on-theme.
