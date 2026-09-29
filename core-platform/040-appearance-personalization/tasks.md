# Tasks: Appearance Personalization

**Input**: Design documents from `/Specs/user-specs/core-platform/040-appearance-personalization/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), design.md, mockup.html
**Tests**: e2e test tasks are included — BOS's standing requirement for a promotable change is that self-tests (Playwright) pass (Constitution IV + Development Workflow). No separate unit-test framework task is created; the `chromeBackground` pure helper is covered by the e2e assertions on computed styles.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

Single project (BOS repository root): `src/`, `e2e/`, `docs/` as in `plan.md`'s source tree.

**File-naming note (T020)**: the feature id is `040-appearance-personalization`, but `e2e/040-assistant-broker-capability.spec.ts` already exists in `e2e/`. To avoid any chance of the runner resolving feature `040` to the wrong file, this feature's spec file is named **`e2e/040-appearance.spec.ts`** (distinct prefix, same directory convention). The Developer should confirm this resolves cleanly under `buildstudio_run_tests` for `user-specs/core-platform/040-appearance-personalization` and rename if the harness expects the full slug.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization — nothing new for this feature

- [ ] T001 No new project structure needed — this feature modifies existing files only. Verify the active branch is `bos/040-appearance-personalization` and `mockup.html` is the UI contract in the spec directory before starting.

---

## Phase 2: Foundational (Blocking Prerequisites) 🎯

**Purpose**: The settings field cluster + the `chromeBackground` helper + `Window.tsx` consumption. This phase alone delivers the complete **US2** (transparency) behaviour and the window-side of **US3** (frosted glass), and it is the no-op-regression gate (SC-005): at defaults it must render today's chrome exactly.

**⚠️ CRITICAL**: US1/US2/US3's *tab controls* (Phases 3–5) depend on T002's helper existing.

- [ ] T002 Extend `OSSettings` in `src/os/types.ts` with `uiOpacity: number` (0–100, default 100), `frostedGlass: boolean` (default false), `frostedGlassBlur: number` (px, 0–24, default 12), each with JSDoc stating range + default + purpose (transparency of window chrome over the desktop; frosted-glass finish; its blur radius).
- [ ] T003 Add the same three fields to `DEFAULT_SETTINGS` in `src/os/settings.ts` (values: 100 / false / 12) — this is the entire persistence migration: `getSettings()` already coalesces `{ ...DEFAULT_SETTINGS, ...parsed }`, so existing users' persisted `settings.json` gains the fields by default on next read (design.md risk R3, closed here).
- [ ] T004 Create `src/os/chrome.ts` exporting: `WALLPAPER_DIR = "/Pictures/wallpapers"`, `CHROME_OPACITY_FLOOR = 0.45`, and a pure `chromeBackground(settings: Pick<OSSettings, "uiOpacity" | "frostedGlass" | "frostedGlassBlur">): { background: string; backdropFilter: string | undefined }` implementing: alpha = `FLOOR + (1 - FLOOR) * (uiOpacity/100)`; `background` = `rgba(21, 23, 30, alpha)` (container/title-bar base colour `#15171e` at computed alpha) plus the content-area variant if kept separate (design.md §2: the content area `#0f1117` follows the same alpha mapping so the whole window reads translucent, not a translucent frame around an opaque hole); `backdropFilter` = `blur(<frostedGlassBlur>px)` when `frostedGlass` is on, else `blur(12px)` (the current `backdrop-blur-md` baseline, so off ≡ today). **Acceptance for this task**: `chromeBackground({uiOpacity:100, frostedGlass:false, frostedGlassBlur:12})` yields alpha 1.0 and a 12px blur — exactly today's hard-coded values (FR-013/SC-005).
- [ ] T005 In `src/components/desktop/Window.tsx`: read the three fields via `useOSStore((s) => s.settings)`, compute `const chrome = chromeBackground(settings)` once, and apply `style` `background` + `backdropFilter` to (a) the window container (replacing `bg-[#15171e]/95 backdrop-blur-md`), (b) the title bar (its `bg-white/5` overlays the container's new background — keep or drop per the mockup; container + content are the two surfaces whose alpha changes), and (c) the content area (replacing `bg-[#0f1117]`). Remove the now-duplicate Tailwind `backdrop-blur-md` class from the container (the inline `backdropFilter` supersedes it). No other `Window.tsx` behaviour (drag/resize/pin/z) changes.
- [ ] T006 **Checkpoint — no-op regression**: with default settings (fresh profile or values explicitly at 100/false), visually confirm every open window looks identical to pre-feature (container, title bar, content area, focus ring, resize handles) against the mockup's default state. Any visible difference is a T004/T005 defect — fix before continuing.

**Checkpoint**: Foundation ready — the transparency + frosted-glass *mechanism* exists and is backward-compatible; user-story UI work can now begin.

---

## Phase 3: User Story 1 — Upload my own image as the desktop background (Priority: P1) 🎯 MVP

**Goal**: first-class import of a local image as the active wallpaper — file picker **and** drag-and-drop — validated, written to the VFS, persisted.

**Independent Test**: Open Settings → Appearance, click **Upload image**, choose a JPEG/PNG, confirm it appears in the live preview and on the desktop, then close and restart the app and confirm it is still the wallpaper (spec US1).

### Implementation for User Story 1

- [ ] T007 [P] [US1] In `src/components/apps/settings/AppearanceTab.tsx`: add the **Upload image** control — a hidden `<input type="file" accept="image/*">` behind an **Upload image** button, plus a drop zone over the wallpaper area (`onDragOver` preventDefault, `onDrop` reading `e.dataTransfer.files`) — per `mockup.html`'s layout. Both routes call one `handleFile(file: File)`.
- [ ] T008 [US1] In `src/components/apps/settings/AppearanceTab.tsx` (`handleFile`, depends on T007): client-side pre-validation — reject non-image types and files over 10 MB with an inline amber error near the action (FR-004), writing nothing and not calling the server for obvious rejects. On valid: `const res = await fsClient.upload(WALLPAPER_DIR, [file])` (import `WALLPAPER_DIR` from `src/os/chrome.ts`, `fsClient` from `src/lib/os-client.ts`), then `apply({ wallpaper: res.uploaded[0] })` — always the **server-returned** path, never a composed one (collision-safety, design.md §1 / FR-011 edge). Surface server-side errors inline as well.
- [ ] T009 [US1] In `src/components/apps/settings/AppearanceTab.tsx`: mark the active uploaded wallpaper — when `settings.wallpaper` is a `/Pictures/wallpapers/...` path, show it as the selected wallpaper state (thumbnail/label) so the existing built-in/URL rows and the upload are visually distinguishable (FR-005: re-selectable, fit-toggleable, replaceable by a gradient — no new mechanism needed, `wallpaperToCss` already resolves the path).
- [ ] T010 [US1] **Checkpoint — US1 independently testable**: upload a valid image → appears live in the tab's preview **and** on the desktop immediately; restart the app → same wallpaper (persistence via `PATCH /api/settings`); a non-image / oversized file → inline error, no new file under `/Pictures/wallpapers/`; a built-in gradient selected afterwards → desktop switches, uploaded file remains (spec US1 scenarios 1–5).

---

## Phase 4: User Story 2 — UI transparency slider (Priority: P1)

**Goal**: a single **UI transparency** slider (100 = fully opaque default) that makes window chrome see-through over the wallpaper, with a readability floor and a live preview.

**Independent Test**: In Settings → Appearance, drag the **UI transparency** slider from 100% toward a lower value, observe open windows/panels rendering more see-through over the wallpaper, then restart and confirm the value is restored (spec US2).

### Implementation for User Story 2

- [ ] T011 [US2] In `src/components/apps/settings/AppearanceTab.tsx`: add the **UI transparency** slider (`<input type="range" min=0 max=100>`), bound to `settings.uiOpacity`, applied via `apply({ uiOpacity: value })` — live (on change), matching the tab's existing optimistic `apply` pattern (design.md §2, FR-006).
- [ ] T012 [US2] In `src/components/apps/settings/AppearanceTab.tsx`: extend the **live preview** card so it applies `chromeBackground(settings)` to a sample window-chrome swatch over `wallpaperToCss(...)`, so the user sees transparency (and later frosted glass) compose before/while committing (FR-012; the mockup demonstrates exactly this card).
- [ ] T013 [US2] **Checkpoint — US2 independently testable**: at default 100 the chrome is unchanged (SC-005); at 70 open windows render ~30% see-through over the wallpaper; at 0 the chrome is translucent but not see-through (floor 0.45 holds — FR-008, SC-006); restart → value restored (FR-011).

---

## Phase 5: User Story 3 — Frosted glass as a separate control (Priority: P2)

**Goal**: a **Frosted glass** toggle + blur-radius slider (0–24px, default 12), independent of and composing with the transparency slider, applied to the same chrome surfaces.

**Independent Test**: In Settings → Appearance, enable **Frosted glass** with a blur radius while transparency is below 100%, observe the wallpaper behind open chrome appearing blurred rather than sharp, then restart and confirm the toggle and radius are restored (spec US3).

### Implementation for User Story 3

- [ ] T014 [US3] In `src/components/apps/settings/AppearanceTab.tsx`: add the **Frosted glass** toggle (`frostedGlass`) plus a blur-radius slider (`frostedGlassBlur`, 0–24, revealed/enabled when the toggle is on), each applied via `apply(...)` (FR-009, design.md §3). The window-side effect requires no further `Window.tsx` change — T005 already consumes `chromeBackground`, which owns the blur mapping.
- [ ] T015 [US3] In `src/components/apps/settings/AppearanceTab.tsx`: extend the live preview (T012) so the sample swatch also reflects `backdrop-filter` blur over the preview wallpaper, and document the compositional no-op in the UI when transparency is 100% and frosted glass is on (a caption/hint per spec US3 scenario 3 — documented no-op, not an error).
- [ ] T016 [US3] **Checkpoint — US3 independently testable**: frosted on + transparency 70 → wallpaper shows through the window chrome blurred (FR-010); frosted on + transparency 100 → no visible blur change (opaque background covers the backdrop); frosted off → exactly today's baseline (T006 no-op state); restart → toggle + radius restored (FR-011); an iframe-hosting window with frosted glass on renders without crash (risk R1).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T017 [P] Create `e2e/040-appearance.spec.ts` (see the file-naming note above) covering, at minimum: (a) default-state no-op regression — window container/title-bar computed `background-color` alpha is 1.0 and `backdrop-filter` is 12px at default settings (SC-005); (b) US1 — upload a small fixture image via the file input, assert a file exists under `/Pictures/wallpapers/` and `settings.wallpaper` points at it, plus the reject path (non-image → inline error, no write); (c) US2 — set the slider to 70, assert the window container's computed background alpha matches `0.45 + 0.55*0.70`, and restart restores it (FR-011); (d) US3 — enable frosted glass at 12px with transparency 70, assert `backdrop-filter` on the window container, restart restores toggle + radius. Model the structure on `e2e/desktop.spec.ts` / `e2e/036-file-type-handlers.spec.ts` (fixture + settings-patch patterns in `e2e/fixtures.ts`).
- [ ] T018 [P] Update documentation in the same change (Constitution VI): the `settings`/Appearance usage docs (`docs/usage/` — locate the existing Appearance/settings page) with the three new controls, the 10 MB / image-only upload limits, the readability floor's existence (not its exact constant), and that transparency/frosted glass apply to window chrome in v1 (panels/top bar/dock explicitly not yet). Note any developer-facing fact worth recording (the `chromeBackground` helper as the single source of truth for chrome appearance math) in the matching `docs/dev/` page.
- [ ] T019 Run quality gates on the branch: `npx tsc --noEmit` and `npm run lint` pass for all changed files (Constitution VII); `npx playwright test e2e/040-appearance.spec.ts` green. Do not run `npm run build` while `next dev` is running.

**Checkpoint**: All three user stories independently functional, e2e green, docs in sync — ready for Promote (the user's decision).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies
- **Foundational (Phase 2)**: T002 → T003 (same file, sequential) → T004 (needs T002's types) → T005 (needs T004's helper) → T006. **Blocks all user stories' tab work.**
- **User Story 1 (Phase 3)**: depends on Phase 2 (uses `WALLPAPER_DIR` from T004); T007 → T008 → T009 → T010. No dependency on US2/US3.
- **User Story 2 (Phase 4)**: depends on Phase 2 (the slider is UI-only over T004/T005's existing mechanism); T011 → T012 → T013. No dependency on US1.
- **User Story 3 (Phase 5)**: depends on Phase 2 (window-side already done in T005) and on T012 (preview extension builds on it); T014 → T015 → T016.
- **Polish (Phase 6)**: depends on all chosen stories. T017 and T018 are parallel; T019 is last.

### User Story Dependencies

- **US1 (P1)**: independent once Phase 2 lands — its own mechanism is `fsClient.upload` + `wallpaperToCss`, which touch none of the Phase 2 files' logic (only `WALLPAPER_DIR`).
- **US2 (P1)**: independent once Phase 2 lands — Phase 2 *is* its mechanism; US2 adds only the slider + preview.
- **US3 (P2)**: independent once Phase 2 lands; composes with US2's preview (T012) but not with US1.

### Within Each User Story

- Tab control (input/drop zone or slider/toggle) → handler logic → selection state → checkpoint.
- Checkpoint passes before moving to the next story.

### Parallel Opportunities

- **T017 (e2e) and T018 (docs)** — different files, no dependencies; run together.
- **Phase 3 (US1) and Phase 4 (US2)** — different control regions of `AppearanceTab.tsx`; *not* [P]-marked because they edit the same file and the same preview card. Sequence them (US1 → US2 → US3) in one branch, checkpointing each.
- Everything in Phase 2 is same-file-adjacent (types/settings/chrome are different files, but T004 needs T002's types) — sequential as listed.

---

## Implementation Strategy

### MVP First (Foundational + US1)

1. Phase 1: confirm branch + mockup contract.
2. Phase 2: Foundational — **checkpoint T006 (no-op regression) is the MVP gate**: chrome must be provably unchanged at defaults before any new control is added.
3. Phase 3: US1 (the reporter's primary ask) — checkpoint T010.
4. **STOP and VALIDATE**: US1 independently testable per its Independent Test.

### Incremental Delivery

1. US2 next (the reporter's second explicit ask) — checkpoint T013 → the two P1 asks are complete and demo-able.
2. US3 (the accepted follow-up scope) — checkpoint T016.
3. Phase 6: e2e + docs + quality gates → Promote.

Each story adds value without breaking the previous ones: US1 never touches chrome math, US2/US3 never touch the upload path, and T006's no-op checkpoint protects everyone who never touches the new controls (SC-005).

---

## Notes

- [P] tasks = different files, no dependencies on incomplete work.
- [Story] label maps each task to its spec.md user story for traceability.
- The three settings fields are written in one `OSSettings` extension (T002) but consumed story-by-story — the field cluster is shared infrastructure (Phase 2), not per-story work.
- **`mockup.html` is the binding UI contract** for every `AppearanceTab.tsx` task (T007, T009, T011, T012, T014, T015); where the mockup and this file disagree, the mockup wins and the discrepancy is reported, not silently resolved.
- `chromeBackground()` is the single source of truth for chrome appearance math (ADR-2) — no task may re-derive the floor/alpha/blur in `Window.tsx` or the preview card.
- Commit after each checkpoint (T006, T010, T013, T016) or logical group.
