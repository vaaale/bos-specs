# Feature Specification: Appearance Personalization

**Feature Branch**: `bos/040-appearance-personalization`

**Created**: 2026-09-29

**Status**: Draft

**App Target**: `bos-core`

**Input**: User description (GitLab issue #55, "Improve appearance"): In Settings → Appearance, (1) make it possible to upload an image to use as a background, and (2) add a slider that controls the transparency of the UI. A follow-up comment by the reporter (@alex) additionally requested that the previously-out-of-scope blur / `backdrop-filter` "frosted glass" control be **included as a separate control**. That request was accepted into scope (spec rev 2) and is the authoritative scope of this feature.

## User Scenarios & Testing

<!--
  User stories are PRIORITIZED as independent, independently-testable journeys.
  Each P1 is an MVP that delivers value on its own.
-->

### User Story 1 — Upload my own image as the desktop background (Priority: P1)

Today a user can only set a custom wallpaper by typing a URL or VFS path they already know. There is no file picker, drag-and-drop, or in-Settings browsing. This story gives a user a first-class way to import an image from disk and make it the active wallpaper without leaving the Appearance tab.

**Why this priority**: This is the reporter's primary explicit ask (#1). It is the highest-value, lowest-risk slice — the existing `wallpaperToCss()` plumbing already resolves a VFS path to a rendered background, so an upload just needs to get bytes into the VFS and point `settings.wallpaper` at them. It is independently valuable and deployable on its own.

**Independent Test**: Open Settings → Appearance, click **Upload image**, choose a JPEG/PNG, confirm it appears in the live preview and on the desktop, then close and restart the app and confirm it is still the wallpaper.

**Acceptance Scenarios**:

1. **Given** a user on Settings → Appearance, **When** they click **Upload image** and select a supported image file, **Then** the file is saved into the user's VFS under a wallpapers location, `settings.wallpaper` is set to that VFS path, and the new image is rendered immediately in the tab's live preview and on the desktop.
2. **Given** an uploaded wallpaper, **When** the user selects a built-in gradient, **Then** the desktop background switches to that gradient and the uploaded file remains in the VFS unchanged.
3. **Given** a supported upload, **When** the app restarts, **Then** the same uploaded image is the active wallpaper (persisted).
4. **Given** the user, **When** they select a non-image file, an over-size file, or a disallowed format, **Then** the upload is rejected with a visible error near the action and nothing is written to the VFS.
5. **Given** a user, **When** they drag an image file onto the wallpaper area, **Then** it is treated identically to a picked file (saved, set active, rendered).

---

### User Story 2 — Control how see-through the UI is with a transparency slider (Priority: P1)

The OS chrome is currently fully opaque and offers no control over translucency. This story adds a single **UI transparency** slider so the wallpaper shows through the window/panel chrome to produce a translucent, "see-through" look.

**Why this priority**: This is the reporter's second explicit ask (#2) and is equally core to the feature. It is independently valuable: even without uploads or frosted glass, a translucency control changes the feel of the OS. It is a small, well-contained change to how the chrome's background is computed.

**Independent Test**: In Settings → Appearance, drag the **UI transparency** slider from 100% toward a lower value, observe open windows/panels rendering more see-through over the wallpaper, then restart and confirm the value is restored.

**Acceptance Scenarios**:

1. **Given** the default value (100% = fully opaque), **When** the user drags the **UI transparency** slider to 70%, **Then** open windows and panels render ~30% see-through over the wallpaper, and the value persists.
2. **Given** a transparency value, **When** the app restarts, **Then** the same value is restored and applied to the chrome.
3. **Given** a very low transparency value, **When** the user views the chrome, **Then** a readability floor applies so UI text remains legible (the exact floor is a design decision — see Assumptions).
4. **Given** the slider at its default (100%), **When** no other setting has changed, **Then** the chrome looks exactly as it does today (backward-compatible no-op).

---

### User Story 3 — Apply a frosted-glass (blur) finish to the UI as a separate control (Priority: P2)

The reporter explicitly requested that the previously-out-of-scope blur / `backdrop-filter` "frosted glass" control be included. This story adds a **separate** Frosted glass control — a toggle plus a blur-radius slider — independent of, and composing with, the transparency slider. Transparency makes the wallpaper visible *through* the chrome; the blur decides whether that view is sharp or frosted.

**Why this priority**: Valuable and explicitly requested, but it is a *second* appearance control layered on top of the two P1 slices, so it is P2. It has no value if transparency is off (frosted glass at 100% opacity is a visual no-op) — it composes with US2 — but is independently shippable and testable as a distinct control.

**Independent Test**: In Settings → Appearance, enable **Frosted glass** with a blur radius while transparency is below 100%, observe the wallpaper behind open chrome appearing blurred rather than sharp, then restart and confirm the toggle and radius are restored.

**Acceptance Scenarios**:

1. **Given** the default (Frosted glass off), **When** the user enables it, **Then** chrome over the wallpaper renders with `backdrop-filter: blur(<radius>px)` at the chosen radius, and the setting persists.
2. **Given** transparency 70% and Frosted glass on (e.g. 12px), **When** a window is open over the wallpaper, **Then** the wallpaper shows through the window chrome blurred rather than sharp.
3. **Given** Frosted glass on and transparency at 100% (opaque), **When** the user views a window, **Then** no visible blur change is expected (an opaque surface covers the backdrop) — this is compositional behaviour, documented and surfaced in the live preview, not an error.
4. **Given** a Frosted glass setting, **When** the app restarts, **Then** the toggle and blur radius are restored.
5. **Given** Frosted glass, **When** the user disables it, **Then** the blur is removed and the chrome returns to the plain (possibly translucent) appearance.

---

### Edge Cases

- **Upload over an existing wallpaper**: selecting a new image replaces the *active* wallpaper pointer; the previously uploaded file is not deleted.
- **Filename collisions**: uploading two files with the same base name must not silently overwrite an unrelated wallpaper (see FR-011).
- **Non-image / oversized / disallowed file**: rejected with a visible error; nothing written (FR-005).
- **Transparency at an extreme low value**: the readability floor clamps effective chrome opacity so text stays legible (FR-008).
- **Frosted glass on but opacity 100%**: no visible change — documented no-op, not an error (US3, scenario 3).
- **Large / hi-res images**: the existing `cover`/`contain` fit toggle applies to uploaded images exactly as it does to path/URL images.
- **`backdrop-filter` support**: where the rendering surface cannot apply `backdrop-filter`, Frosted glass degrades gracefully to plain transparency with no crash.

## Requirements

### Functional Requirements

- **FR-001**: The Appearance tab MUST provide an **Upload image** affordance (a file picker) in addition to the existing URL/VFS-path input.
- **FR-002**: On selecting a supported image, the system MUST write the file into the user's VFS under a dedicated wallpapers location and set `settings.wallpaper` to that VFS path so it renders through the existing `wallpaperToCss()` path with no parallel mechanism.
- **FR-003**: The system MUST support drag-and-drop of an image file onto the wallpaper area, behaving identically to a picked file.
- **FR-004**: The system MUST validate uploaded files by type (image only) and by a size cap, and MUST reject disallowed files with a visible error near the action, writing nothing on rejection.
- **FR-005**: An uploaded wallpaper MUST behave exactly like an existing path/URL wallpaper: re-selectable by its path, controllable by the existing `cover`/`contain` fit toggle, replaceable by choosing a built-in gradient.
- **FR-006**: The Appearance tab MUST provide a single **UI transparency** slider bound to a new persisted settings field, with **100 = fully opaque** as the default.
- **FR-007**: The transparency value MUST change how the UI chrome (window bodies / panels — the exact surface set is a design decision) renders over the desktop/wallpaper so the wallpaper shows through.
- **FR-008**: A **readability floor** MUST clamp the effective chrome opacity at a minimum so UI text remains legible at low transparency values.
- **FR-009**: The Appearance tab MUST provide a **separate Frosted glass** control, independent of the transparency slider: a toggle plus a blur-radius slider revealed when enabled, bound to new persisted settings fields.
- **FR-010**: When Frosted glass is enabled, the system MUST apply `backdrop-filter: blur(<radius>px)` to the **same chrome surfaces the transparency setting affects**, so the wallpaper behind them appears blurred. Frosted glass and transparency MUST compose.
- **FR-011**: The system MUST persist all three settings (wallpaper pointer, transparency, frosted-glass toggle + radius) so they survive an app restart.
- **FR-012**: The Appearance tab MUST render a **live preview** that reflects the current wallpaper, transparency, and frosted-glass values.
- **FR-013**: At the default values (no upload, transparency 100%, Frosted glass off) the OS chrome MUST be visually identical to today's behaviour (backward compatible).

### Key Entities

- **OSSettings (extended)**: the existing persisted settings object. This feature adds fields: `uiOpacity` (number, 0–100, default 100), `frostedGlass` (boolean, default false), `frostedGlassBlur` (number, px, default set in design). The existing `wallpaper` (string) now additionally holds VFS paths produced by uploads.
- **Wallpaper file**: an image stored in the VFS under the wallpapers location, referenced by its VFS path in `settings.wallpaper`.

## Success Criteria

### Measurable Outcomes

- **SC-001**: A user can import a local image and have it appear as the active desktop wallpaper in under ~15 seconds, without leaving the Appearance tab.
- **SC-002**: A user can make the UI visibly translucent (and, optionally, frosted) by moving at most two controls, and see the result live before the change is committed.
- **SC-003**: All three settings (upload, transparency, frosted glass) persist across an app restart with zero reconfiguration.
- **SC-004**: Disallowed or oversized uploads are rejected with a clear, in-place error and leave the VFS unchanged (no partial writes).
- **SC-005**: At all default values the chrome is pixel-equivalent to the pre-feature appearance (no visual regression for users who never touch the new controls).
- **SC-006**: Text in translucent chrome remains legible at the lowest user-reachable transparency (readability floor holds).

## Assumptions

- **Uploads land in a dedicated VFS wallpapers folder** (e.g. `/Pictures/wallpapers/`), not a user-chosen location. This keeps the feature simple and the existing `wallpaperToCss()` VFS-path resolution working unchanged. *(This was open question 1 in the draft; it is assumed here and flagged for confirmation in design/review.)*
- **Upload limits**: image types only (e.g. PNG, JPEG, WebP), with a reasonable size cap (a specific MB cap is a design decision).
- **The transparency/frosted-glass surface set is the window/panel chrome over the desktop** (window bodies and panels), applied **globally** (a single setting, not per-window). Per-window opacity is explicitly out of scope. *(Open question 3.)*
- **A readability floor exists** (a minimum effective chrome opacity, e.g. not allowing fully transparent chrome) so text stays legible. The exact floor is a design decision. *(Open question 4.)*
- **Frosted glass is a toggle + adjustable blur-radius slider** (0–24px, default ~12px) rather than a fixed radius. *(Open question 5.)*
- **Existing `backdrop-blur` on the window chrome** (already present in `Window.tsx`) is the base the new controls tune; the feature extends rather than replaces it.
- **Default transparency is 100%** (fully opaque) so existing behaviour is unchanged until a user opts in.

## Out of scope

- Per-window independent opacity (this is a single global setting per control).
- Editing, cropping, or replacing the wallpaper image in place (the user picks a new image instead).
- Any change to the built-in gradient set, accent colour, or chat font/size behaviour.
- Tiling/scale/parallax animation of the wallpaper.
