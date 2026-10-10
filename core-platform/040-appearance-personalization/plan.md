# Implementation Plan: Appearance Personalization

**Branch**: `bos/040-appearance-personalization` | **Date**: 2026-09-29 | **Spec**: `./spec.md`

**Input**: Feature specification from `/Specs/user-specs/core-platform/040-appearance-personalization/spec.md`

## Summary

Add three controls to the Settings → Appearance tab (a built-in tab inside the `settings` app):

1. **Upload image** — pick or drag-and-drop a local image file; it is written into the user's VFS under `/Pictures/wallpapers` via the existing `fsClient.upload` → `POST /api/fs/upload` path, and `settings.wallpaper` is pointed at the server-returned path. No new route: the existing `wallpaperToCss()` resolver already turns a leading-`/` VFS path into a rendered `background`, so the desktop and the tab's live preview pick the image up with zero changes to the render path.
2. **UI transparency slider** — a new persisted `OSSettings.uiOpacity` field (0–100, default 100 = fully opaque) that drives the alpha of the window chrome (container, title bar, content area) in `Window.tsx`, via a new pure helper `chromeBackground(settings)` in `src/os` that owns the readability-floor math.
3. **Frosted glass** — two more persisted fields, `frostedGlass: boolean` (default `false`) and `frostedGlassBlur: number` px (default `12`, range 0–24), that make the same helper also return a `backdrop-filter: blur(...)` value, composing with the transparency alpha (blurred wallpaper showing through translucent chrome).

All three share one settings field cluster, one helper, and one tab, so they are implemented as one branch but remain independently testable per the spec's user-story structure.

**Technical approach** (full reasoning in `./design.md`): reuse `/api/fs/upload` rather than adding a wallpaper endpoint (ADR-1); centralise the floor/alpha/blur math in one pure helper rather than spreading it across `Window.tsx` Tailwind classes (ADR-2); scope the transparency/frosted-glass surface set to the window chrome in v1, not the whole OS shell (ADR-3).

## Technical Context

**Language/Version**: TypeScript (BOS's existing TS setup, Next.js App Router)

**Primary Dependencies**: React 19, Zustand, Tailwind CSS, lucide-react — all already in the project; no new dependencies introduced

**Storage**: `data/settings.json` (via the existing `src/os/settings.ts` `getSettings`/`updateSettings` pair, persisted by `PATCH /api/settings`); uploaded wallpaper bytes live in the user's VFS under `/Pictures/wallpapers/` (via the existing `POST /api/fs/upload`)

**Testing**: Playwright e2e under `e2e/` (BOS's existing harness); `npx tsc --noEmit` + `npm run lint` as the standing quality gates. No new test framework is introduced.

**Target Platform**: BOS desktop web app (Next.js SSR + React client, running in the user's browser)

**Project Type**: Existing single-project web app — this feature modifies existing files only, creates no new app/directory tree

**Performance Goals**: No measurable performance target beyond "no visible jank": `chromeBackground()` is a pure string-computation called once per render of `Window.tsx` per window — trivial. `backdrop-filter` already exists on windows today (`backdrop-blur-md`); this feature only tunes its radius, it does not introduce a new compositing capability.

**Constraints**:
- At default values (`uiOpacity=100`, `frostedGlass=false`) the rendered chrome MUST be pixel-identical to today's (FR-013, SC-005) — the helper's math must produce exactly the current values at those inputs.
- No new `/api/**` route, no change to `wallpaperToCss()`, no change to `fsClient.upload()`'s contract — the feature is wired into existing endpoints, not added to them.
- `backdrop-filter` must degrade gracefully where unsupported (plain transparency, no crash).
- Uploads are client-pre-validated (image type, ≤10 MB) before hitting the server; the server remains the final authority and its errors surface inline.

**Scale/Scope**: 5 files modified, 1 new pure helper module (or its constant co-located in an existing module — see design.md's file plan), 1 settings field cluster of 3 fields, 1 new tab section set.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Principle | Status | Notes |
|---|-----------|--------|-------|
| I | Spec-Driven | ✅ Pass | `spec.md` exists and is agreed; `design.md` follows. No implementation without this spec. |
| II | Server Authority & SSR Boundary | ✅ Pass | No secrets, no Node-API surface, no API key handling. The only server touch is the *existing* `PATCH /api/settings` and `POST /api/fs/upload` routes, both unchanged — this feature calls them, it does not add to them. New domain types (`uiOpacity`, etc.) land in framework-free `src/os/types.ts`, exactly where the constitution says domain types live. |
| III | Always Delegate | ✅ Pass | This plan is the plan artifact only; `implement` will be delegated to the Developer sub-agent via `dev_delegate` (target: `bos-core`, matches `spec.md`'s App Target field). |
| IV | Minimize Blast Radius | ✅ Pass | Work happens on the already-active feature branch `bos/040-appearance-personalization` in an isolated worktree; no change to the running tree until Promote. |
| V | The VFS Is Not the Source | ✅ Pass | Uploaded wallpaper bytes land in the VFS (user data) — that's correct, that's what the VFS is for. No BOS *source* change touches the VFS path as a code artifact. |
| VI | Specs & Docs Stay in Sync | ✅ Pass (pending) | This plan, the design, and (at implement time) `docs/usage`/`docs/dev` updates for the Appearance tab's new controls must land in the same branch. Tracked as a Polish-phase task. |
| VII | Respect Boundaries | ✅ Pass | No changes to `package.json`, lockfiles, build config, or secrets. Quality gates (`tsc --noEmit`, `npm run lint`) apply to the changed files at implement time. |

**Gate result: PASS** — no violations to justify in Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
/Specs/user-specs/core-platform/040-appearance-personalization/
├── spec.md              # Agreed (this branch's input)
├── design.md            # Architect's structural pass (this branch's prior step)
├── mockup.html          # UI contract (Appearance tab: upload, slider, frosted-glass control, live preview)
├── plan.md              # This file
└── tasks.md             # Next pipeline step — not created here
```

No `research.md`, `data-model.md`, or `contracts/` are warranted: there are no unresolved technical unknowns (every mechanism — `fsClient.upload`, `wallpaperToCss`, the settings hydration path, the store's `apply` pattern — is confirmed present in source), no new entities beyond a 3-field extension to an existing persisted type, and no new external interface (the feature consumes two existing HTTP endpoints, it exposes none).

### Source Code (repository root)

```text
src/
├── os/
│   ├── types.ts            # MODIFY: add uiOpacity / frostedGlass / frostedGlassBlur to OSSettings
│   ├── settings.ts         # MODIFY: add the 3 new fields to DEFAULT_SETTINGS (so hydration + getSettings coalesce them for existing users)
│   └── chrome.ts           # NEW (or co-located constants + helper in wallpapers.ts): WALLPAPER_DIR const,
│                           #   chromeBackground(settings) → { background, backdropFilter }, CHROME_OPACITY_FLOOR const
├── lib/
│   └── os-client.ts        # UNCHANGED — fsClient.upload and settingsClient.patch are reused as-is
├── store/
│   ├── os-store.ts         # UNCHANGED — applySettings(patch) already handles arbitrary Partial<OSSettings>
│   └── os-provider.tsx     # UNCHANGED
└── components/
    ├── apps/settings/
    │   └── AppearanceTab.tsx   # MODIFY: Upload image control (file input + drop zone + handleFile),
    │                           #   UI transparency slider, Frosted glass toggle + blur slider, live-preview card
    │                           #   applying chromeBackground()
    └── desktop/
        └── Window.tsx          # MODIFY: consume chromeBackground(settings) on the window container,
                                #   title bar, and content area (replaces the hard-coded bg-[#15171e]/95
                                #   backdrop-blur-md / bg-[#0f1117])

e2e/
└── 040-appearance-personalization.spec.ts   # NEW: Playwright coverage for the three user stories
                                               #   (upload→persist, slider→persist+floor, frosted glass→persist+compose)
```

**Structure Decision**: single-project modification of the existing `src/` tree, per the plan above. The one genuinely *new* module (`chrome.ts`) is a small pure-function + constants file placed next to the existing `wallpapers.ts`/`settings.ts`/`types.ts` in `src/os/`, matching the design.md recommendation (ADR-2: one source of truth for the floor + alpha + blur math). If the Developer prefers co-locating `WALLPAPER_DIR` + `chromeBackground` directly inside `wallpapers.ts` (it already imports from `types.ts`), that is an acceptable alternative — the plan's constraint is "one pure helper, one place," not the exact filename.

## Design Notes

Full structural reasoning — including the three ADRs, the component-by-component mechanism walkthrough, the file/module plan, integration points, and the risk register (R1–R5) — is in `./design.md`, written by the `architect` role. This plan does not repeat it; the key decisions it inherits, summarized for a reader of `plan.md` alone:

1. **Reuse, don't rebuild** (ADR-1): the upload path is `fsClient.upload("/Pictures/wallpapers", [file])` → `res.uploaded[0]` → `apply({ wallpaper: res.uploaded[0] })`. `/api/fs/upload` already mkdirs the target dir, writes collision-safe names, and returns the resolved path — none of that is reinvented.
2. **One helper owns the appearance math** (ADR-2): `chromeBackground(settings: { uiOpacity, frostedGlass, frostedGlassBlur })` returns `{ background: string; backdropFilter: string | undefined }`. The readability floor (`FLOOR = 0.45`) is a named constant in that module. `Window.tsx` and `AppearanceTab.tsx`'s live-preview card both call it — never re-derive the math.
3. **v1 surface set is the window chrome, not the whole shell** (ADR-3): `Window.tsx` container + title bar + content area. Top bar / dock / panels are explicitly out of scope for this branch.
4. **Defaults are provably today's values**: `uiOpacity: 100`, `frostedGlass: false` in `DEFAULT_SETTINGS` (`src/os/settings.ts`) — so `chromeBackground({uiOpacity:100, frostedGlass:false, frostedGlassBlur:12})` must yield the exact same `background` alpha (1.0) and `backdropFilter` (`blur(12px)`, matching `backdrop-blur-md`) the current hard-coded classes produce. This is the concrete, checkable form of FR-013/SC-005.
5. **Settings hydration already works for missing fields**: `src/os/settings.ts` `getSettings()` is `{ ...DEFAULT_SETTINGS, ...parsed }` — adding the three fields to `DEFAULT_SETTINGS` is sufficient for both fresh installs and existing users' persisted `settings.json` that predates this feature; no separate migration step is needed (risk R3, closed by this mechanism rather than a migration script).

### Open decisions resolved here (carried from spec.md's "Assumptions")

| Spec open item | Resolution (from design.md, restated for the plan of record) |
|---|---|
| Uploads land in a dedicated VFS wallpapers folder | `/Pictures/wallpapers/`, `WALLPAPER_DIR` const |
| Upload size cap | ≤10 MB, image types only, client-pre-validated; server is final authority |
| Transparency/frosted-glass surface set | Window chrome (container + title bar + content area), global, not per-window (v1) |
| Readability floor | `CHROME_OPACITY_FLOOR = 0.45`, alpha = `FLOOR + (1-FLOOR) * (uiOpacity/100)` |
| Frosted-glass control shape | Toggle + 0–24px blur-radius slider, default 12px, replacing (not adding to) the existing fixed `backdrop-blur-md` when enabled |
