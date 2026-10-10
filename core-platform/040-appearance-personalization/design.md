# Design: Appearance Personalization

**Feature Branch**: `bos/040-appearance-personalization`
**Spec**: `./spec.md`
**Mockup**: `./mockup.html`
**Classification**: `bos-core` (matches the spec's `App Target`)

> **Revision (rev 3)** — the original frosted-glass design made "off" fall back to the window's pre-existing `backdrop-blur-md` (12px) so the default stayed unchanged. That left the toggle inert: **off** rendered `blur(12px)` and **on** at the default 12px also rendered `blur(12px)` — two identical states, so the switch only revealed/hid the blur slider and never toggled the effect. Corrected contract: **off = no backdrop blur at all** (`backdropFilter: undefined`), **on = `blur(<frostedGlassBlur>px)`**, and the **default state is ON at 12px** — which still reproduces the pre-existing `backdrop-blur-md` exactly, so SC-005 (pixel-equivalent at defaults) holds and the control is now functional. This is the source of truth for the frosted-glass logic in §3; `spec.md` was updated in lockstep (default `frostedGlass: true`).

## Context

The Appearance surface is a single client component — `src/components/apps/settings/AppearanceTab.tsx` — that reads `settings` from the OS store (`useOSStore`) and writes through a single `apply(patch)` helper that calls both `applySettings(patch)` (in-memory, optimistic) and `settingsClient.patch(patch)` (`PATCH /api/settings`, persisted). It already renders a **live preview** by calling `wallpaperToCss(settings.wallpaper, settings.wallpaperFit)`.

The desktop itself is `src/components/desktop/Desktop.tsx`, which paints `wallpaperToCss(...)` onto its background. Window chrome is `src/components/desktop/Window.tsx`. `wallpaperToCss` (`src/os/wallpapers.ts`) already resolves a leading-`/` VFS path to `/api/fs/raw?path=…`, so an image that lives in the VFS renders as a wallpaper with **zero changes to the render path** — the only work is getting bytes into the VFS and pointing `settings.wallpaper` at them.

The OS settings shape is `OSSettings` in `src/os/types.ts` (line ~118). It currently has **no** opacity or blur fields.

This feature is three small, mostly client-side slices. It touches two subsystems (the Settings app + the desktop/window chrome + the OS settings type) and adds one settings field cluster, so it is scoped but worth a structural pass.

## Component design

### 1. Wallpaper upload (P1) — `AppearanceTab.tsx` + existing `fsClient.upload`

**Mechanism (grounded, not invented):** `src/lib/os-client.ts` already exposes `fsClient.upload(dirPath, files, onProgress?)`, which POSTs `FormData` to `/api/fs/upload` and resolves to `{ ok: true, uploaded: string[] }` — `uploaded` are the **resolved destination paths** the server assigned. The feature does not need a new route; it wires an existing client to an existing endpoint.

- Add an **Upload image** control (a hidden `<input type=file accept="image/*">` plus a drag-and-drop drop zone over the wallpaper area — see mockup). On a selected/dropped file:
  1. Client-side pre-validate: type is an image and size is within the cap (reject locally → show an inline amber error, do not hit the server for obvious rejects).
  2. `const res = await fsClient.upload(WALLPAPER_DIR, [file])` where `WALLPAPER_DIR = "/Pictures/wallpapers"` (a new `const` in the tab or `src/os/wallpapers.ts`). The server `mkdir`s the dir on demand (the upload endpoint already creates the target folder).
  3. Take `res.uploaded[0]` (the resolved VFS path) and `apply({ wallpaper: res.uploaded[0] })`. Because `wallpaperToCss` resolves `/…` paths, the desktop + live preview update with no further change.
- **Drag-and-drop** uses `onDrop`/`onDragOver` on the drop zone; the File object flows through the same `handleFile` as the picker.
- **Validation cap** (Assumption): image types only, ≤ 10 MB. The cap is a small `const` in the tab; the server is the final authority (it still enforces its own limits — we surface its errors too).
- **Filename collisions**: the `/api/fs/upload` endpoint returns the resolved path it actually wrote (it collision-safes the name internally, matching the `copy`/`move` behaviour that "picks a collision-safe sibling name"). We use the **returned** path, never a path we compose, so a second file with the same base name cannot clobber the first.

This slice is self-contained to the Settings tab; the only shared-surface change is the new `WALLPAPER_DIR` constant.

### 2. UI transparency slider (P1) — `OSSettings` + `Window.tsx` + `AppearanceTab.tsx`

**New settings field**: `uiOpacity: number` on `OSSettings`, **0–100, default 100 (fully opaque)**. Backward-compatible: existing persisted settings lack the field, so a default is applied at read time (the store's settings hydration must coalesce `settings.uiOpacity ?? 100`).

**Where the value is applied** — the design decision the open questions left to us:

- **Surface set (open question 3, resolved):** the **window chrome** — i.e. the elements in `Window.tsx` that sit over the wallpaper. Concretely the window container (currently `bg-[#15171e]/95 backdrop-blur-md`) and the title bar (`bg-white/5` over the same). The window **content area** (`bg-[#0f1117]`) is left opaque-by-default but its background alpha also follows the same mapping, so the whole window reads as translucent rather than a translucent frame around an opaque hole. We do **not** touch the top bar / dock / panels in v1 — that is a larger, riskier surface; the spec already scopes this to window/panel chrome and the mockup demonstrates windows. (Panels can adopt the same token in a follow-up.)
- **Mapping (open question 4 — readability floor, resolved):** chrome alpha = `FLOOR + (1 - FLOOR) * (uiOpacity/100)`, with `FLOOR = 0.45`. At 100% → alpha 1.0 (identical to today, satisfies FR-013/SC-005); at 0% → alpha 0.45, so the chrome never becomes fully transparent and text stays legible (FR-008). The floor is a named `const` so it is tunable without a UI.
- **Implementation approach (grounded):** the window already uses `backdrop-blur-md` + an `rgba` background, so we replace the **hard-coded alpha** in the window background classes with a value computed from `uiOpacity`. Two clean options, either acceptable:
  - **(a) Inline style**: `Window.tsx` reads `settings.uiOpacity` from the store and sets `style={{ background: rgba(...) , backdropFilter: ... }}` on the container + title bar, overriding the Tailwind `bg-[#15171e]/95`.
  - **(b) A single helper** `chromeBackground(uiOpacity)` in `src/os` (next to `wallpapers.ts`) that returns the `rgba(...)` string, consumed by `Window.tsx`. This centralises the floor constant so any later panel that adopts transparency uses the same math. **Recommended (b)** — one source of truth for the floor + alpha mapping.
- The **live preview** in `AppearanceTab.tsx` reflects the value by applying the same helper to its preview card (the mockup does exactly this), so the user sees the effect before it's committed.

This slice touches `types.ts` (add field), `Window.tsx` (consume it), the settings hydration (default), and `AppearanceTab.tsx` (the control + preview). It is the slice most worth a careful review because it changes what every window looks like.

### 3. Frosted glass control (P2) — `OSSettings` + `Window.tsx` + `AppearanceTab.tsx`

**New settings fields**: `frostedGlass: boolean` (default `false`) and `frostedGlassBlur: number` (px, **default 12**, range 0–24 — open question 5, resolved as toggle + adjustable slider per the mockup).

**Where applied**: the **same chrome surfaces** the transparency mapping targets (the window container in `Window.tsx`, which is the element that carries the blur). The mapping is now a true on/off gate (rev 3):
- `frostedGlass === true` → `backdropFilter: blur(<frostedGlassBlur>px)`, clamped 0–24px (the slider takes over the radius entirely).
- `frostedGlass === false` → `backdropFilter: undefined` — **no blur at all**, not a fallback radius.

The **default is `frostedGlass: true` at `frostedGlassBlur: 12`**, which renders `blur(12px)` — byte-identical to the pre-existing `backdrop-blur-md` — so the default chrome is unchanged (SC-005) *and* the toggle is no longer inert: off genuinely removes the blur, on re-applies it. (The earlier "fall back to `backdrop-blur-md` when off" idea is retired — it was the cause of the no-op toggle: off and on-at-default produced the same 12px.) At 100% transparency the opaque background covers the backdrop regardless, so the blur has no visible effect there — that remains the documented compositional no-op (US3 scenario 3), not the toggle being broken.

**Composition (grounded in CSS):** `backdrop-filter: blur()` blurs whatever is *behind* the element (the wallpaper), while the element's own `background` alpha (from slice 2) controls how much of that blurred backdrop shows through. They compose naturally:
- transparency 70% + frosted on (12px) → wallpaper shows through, blurred. ✓
- transparency 100% (opaque) + frosted on → the opaque background covers the backdrop, so no visible blur. This is the documented no-op (US3 scenario 3) — not a bug.

The `chromeBackground` helper from slice 2 can be extended to also take the frosted-glass fields and return `{ background, backdropFilter }` in one place — **recommended**, so `Window.tsx` has a single call and the floor + blur logic lives together.

## File / module plan (what this feature creates/modifies — DESIGN only, nothing is written here)

**Modified:**
- `src/os/types.ts` — add `uiOpacity`, `frostedGlass`, `frostedGlassBlur` to `OSSettings` (with JSDoc: ranges + defaults).
- `src/os/wallpapers.ts` (or a new sibling `src/os/chrome.ts`) — add `WALLPAPER_DIR` const and the `chromeBackground(settings)` helper returning `{ background, backdropFilter }` (centralises the readability floor + blur mapping).
- `src/components/apps/settings/AppearanceTab.tsx` — add the Upload control (file input + drop zone + `handleFile` via `fsClient.upload`), the UI transparency slider, and the Frosted glass toggle + blur slider; extend the live preview to apply `chromeBackground`; wire all three into `apply(patch)`.
- `src/components/desktop/Window.tsx` — consume `chromeBackground(settings)` on the window container + title bar (replace the hard-coded `bg-[#15171e]/95 backdrop-blur-md`) and the content-area alpha.
- Settings hydration / defaults (where `OSSettings` is coalesced on load — the OS store's settings init) — default the three new fields when absent from persisted settings.

**Not created (explicitly out of scope):** any new `/api/fs` route (we reuse `/api/fs/upload`), any per-window opacity state, any tiling/animation, any change to the gradient set / accent / chat font.

## Integration points (existing BOS mechanisms this feature merely calls into)

- `fsClient.upload` / `PATCH /api/settings` (`src/lib/os-client.ts`) — existing, reused as-is.
- `wallpaperToCss` (`src/os/wallpapers.ts`) — existing VFS-path → CSS resolution; uploads lean on it unchanged.
- `useOSStore` settings selectors + `applySettings` (`src/store/os-provider`) — existing optimistic-apply pattern; new fields flow through the same path as `wallpaper`/`accent` today.
- `Desktop.tsx` — no change needed; it already renders `wallpaperToCss`, so uploaded wallpapers appear automatically once `settings.wallpaper` points at them.

## ADRs

- **ADR-1: Reuse `/api/fs/upload` rather than a new wallpaper endpoint.**
  *Decision:* the upload is `fsClient.upload('/Pictures/wallpapers', [file])`. *Why:* the endpoint exists, handles mkdir + collision-safe naming + byte writes, and returns the resolved path we need to store. A bespoke route would duplicate that and a second auth/scope path. *Consequence:* uploads live under `/Pictures/wallpapers` (visible in the Files app, removable there) — acceptable and arguably a feature (the user can manage the files themselves).

- **ADR-2: Centralise the chrome appearance math in one helper.**
  *Decision:* a `chromeBackground(settings)` helper (in `src/os`) returns `{ background, backdropFilter }`, owning the readability floor and the blur mapping. *Why:* the floor + alpha + blur are one coupled concern; putting them in a helper (rather than spread across `Window.tsx` Tailwind classes) gives a single source of truth and makes the default (100% / frosted off) provably identical to today. *Consequence:* `Window.tsx` swaps a Tailwind class for an inline style on two elements — a small, reviewable diff.

- **ADR-3: Scope transparency/frosted-glass to window chrome in v1, not the whole OS shell.**
  *Decision:* apply to window containers + title bars (and the content area alpha); leave top bar / dock / panels for a follow-up. *Why:* every window looks right with far less risk than restyling the shell; the spec already frames this as window/panel chrome and the mockup demonstrates windows. *Consequence:* panels (e.g. a translucent panel) are not affected in v1 — noted for follow-up.

## Risks

- **R1 — `backdrop-filter` over iframes.** `Window.tsx` hosts iframe apps; `backdrop-filter` on an ancestor of a cross-origin/opaque iframe can be expensive or visually inconsistent across browsers. *Mitigation:* the current code already uses `backdrop-blur-md` on windows, so we are not introducing a new capability — we are only tuning its radius. Keep radii modest (0–24px); test with an iframe app open.
- **R2 — Text legibility at low transparency.** Even with the floor, very translucent chrome over a bright wallpaper can reduce contrast. *Mitigation:* the `FLOOR = 0.45` keeps a meaningful base; the live preview surfaces it before commit; the floor is a single tunable `const`.
- **R3 — Persisted settings migration.** Existing users' settings lack the new fields. *Mitigation:* default at hydration (`?? 100`, `?? false`, `?? 12`) so there is no migration step and no crash.
- **R4 — Upload size/memory for large images.** *Mitigation:* client-side cap (10 MB) before upload; the server enforces its own limits and errors surface inline.
- **R5 — Two settings controls both driving the same chrome.** If the floor + blur + alpha interact unexpectedly, debugging is harder. *Mitigation:* ADR-2 keeps the math in one helper with a unit-testable pure function; the mockup already models the composition.
