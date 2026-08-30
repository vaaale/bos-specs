# Tasks: Media Support in `web_view` (Images & Video)

**Input**: [spec.md](./spec.md), [plan.md](./plan.md), [design.md](./design.md)

**Organization**: Grouped by user story. All tasks are for the Developer (`dev_delegate`) on branch `web-view-media`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)

---

## Phase 1: Foundational (Shared Infrastructure)

**Purpose**: Core infrastructure that all user stories depend on.

- [ ] T001 [P] Create `src/lib/apps/media.ts` — framework-free module exporting `IMAGE_EXTENSIONS` (Set: png, jpg, jpeg, gif, webp, svg, avif), `VIDEO_EXTENSIONS` (Set: mp4, ogv, webm, mov, m4v, avi), `classifyMediaTarget(src: string): "image" | "video" | null` (handles `/api/fs/raw?path=…`, VFS paths, `data:` URIs via MIME prefix, and `http(s)://` via pathname extension with query stripped), and `mediaTypeFromMime(mime: string)` helper.
- [ ] T002 [P] Extend `src/app/api/fs/raw/route.ts` MIME map: add `.m4v` → `video/mp4` and `.avi` → `video/x-msvideo`. Additive only; no change to streaming/404/branch-scoping.

**Checkpoint**: Classifier and MIME map ready — user story implementation can begin.

---

## Phase 2: User Story 1 — Preview an Image (P1) 🎯 MVP

**Goal**: An agent can call `web_view` with an image VFS path or URL and the window shows it centered, fit-to-frame, on a neutral `#1a1a1a` stage.

**Independent Test**: Call `web_view` with a known image path (e.g. `/workspace/test.png`). Confirm the window shows the image centered, scaled to fit, no crop, no white background.

### Implementation

- [ ] T003 [US1] Modify `src/apps/html-viewer/index.tsx`: read `params.mode`, `params.src` from the app params. When `mode === "image"`, render a native `<img src={params.src}>` inside a `#1a1a1a` stage div (`position: relative; overflow: hidden; flex; items-center; justify-center; padding: 16px`) with `object-fit: contain; max-width: 100%; max-height: 100%; width: auto; height: auto; border-radius: 8px; box-shadow: 0 25px 50px -12px rgba(0,0,0,0.5)`. Key the `<img>` on `src`. On `onError`, show a centered error card on the same stage. Leave the existing `html`/`url` iframe path byte-for-byte unchanged.
- [ ] T004 [US1] Modify `src/components/agent/v2/FrontendToolsV2.tsx` (`web_view` handler): after resolving the target URL (existing logic), call `classifyMediaTarget(resolvedUrl)`. If it returns `"image"` or `"video"`, build `params = { mode, src: resolvedUrl, title?, poster?, autoplay?, loop?, muted? }` and launch `html-viewer` with those params. Default `title` to the target's basename when omitted. For non-media, leave existing `{ html }` / `{ url, title }` params unchanged. The existing verify-fetch for `/api/fs/raw` targets already covers media raw URLs (FR-007 tool-failure half).
- [ ] T005 [P] [US1] Update `src/lib/assistant/tools/frontend-declarations.ts` — `web_view` declaration: update the `description` to state that images and video are supported and document `poster`/`autoplay`/`loop`/`muted` parameters. Add the four new parameter properties to the schema.
- [ ] T006 [P] [US1] Update `src/lib/agent/capabilities-registry.ts` — `web_view` entry: update the one-line description to mention image and video preview.

**Checkpoint**: User Story 1 fully functional — image preview works end-to-end.

---

## Phase 3: User Story 2 — Preview a Video (P1)

**Goal**: An agent can call `web_view` with a video VFS path or URL and the window shows a controllable video player (play/pause/seek/volume/fullscreen), centered, fit-to-frame.

**Independent Test**: Call `web_view` with a known video path (e.g. `/workspace/test.mp4`). Confirm the window shows a video player with native controls, user can play/seek/fullscreen.

### Implementation

- [ ] T007 [US2] Modify `src/apps/html-viewer/index.tsx` (media branch, alongside T003's image branch): when `mode === "video"`, render a native `<video src={params.src} controls playsinline preload="metadata">` with the same `#1a1a1a` stage and fit-to-frame CSS as the image branch. Apply `poster`, `autoplay`, `loop`, `muted` attributes from params when present. Key on `src`. On `onError`, show the same centered error card.
- [ ] T008 [US2] The v2 handler (T004) already passes `mode: "video"` + media options for video targets. No additional handler change needed beyond T004 (which handles both image and video).

**Checkpoint**: User Stories 1 AND 2 fully functional — image and video preview work end-to-end.

---

## Phase 4: User Story 3 — Media Playback Options (P2)

**Goal**: `poster`, `autoplay`, `loop`, `muted` parameters are accepted by `web_view` and applied to the video player.

**Independent Test**: Call `web_view` with a video path + `muted: true` + `autoplay: true`. Confirm the video starts playing immediately. Call with `loop: true`. Confirm it restarts on end. Call with `poster: "/workspace/thumb.jpg"`. Confirm the poster shows before playback.

### Implementation

- [ ] T009 [P] [US3] The v2 handler (T004) already extracts `poster`/`autoplay`/`loop`/`muted` from the tool call args and passes them in params. Confirm `poster` VFS paths are rewritten to raw URLs (same `toRawUrl` rule as `url`/`filePath`) before being passed (design risk R4). If not already handled in T004, add the poster rewrite.
- [ ] T010 [P] [US3] The `frontend-declarations.ts` update (T005) already documents the four new params. Confirm they are typed correctly (booleans, poster is string).

**Checkpoint**: All playback options functional.

---

## Phase 5: User Story 4 — Robust Errors + Honest Contract (P2)

**Goal**: The tool description advertises media support; missing/unrenderable media produces a clear in-window error and a non-success tool return.

**Independent Test**: (a) Read the `web_view` tool description — confirm it mentions images and video. (b) Call `web_view` with a nonexistent VFS path — confirm tool returns an error, no window opens. (c) Call `web_view` with a valid path to a file with an unsupported extension — confirm in-window error card.

### Implementation

- [ ] T011 [US4] The v2 handler's existing verify-fetch (for `/api/fs/raw` targets) already returns a tool failure for missing VFS files. Confirm media raw URLs flow through it (they do — they resolve to `/api/fs/raw`). No additional handler change needed.
- [ ] T012 [US4] The in-window error card (implemented in T003/T007 via `onError`) handles unrenderable media (unsupported codec, external URL 404, data URI with bad MIME). Confirm the error message is clear and identifies the target.
- [ ] T013 [P] [US4] The tool declaration updates (T005, T006) already state image/video support in the description. Confirm the language is clear to the model.

**Checkpoint**: All 4 user stories independently functional.

---

## Phase 6: Cross-Cutting — v1 Consistency + Docs

**Purpose**: Keep all declaration surfaces and docs in sync (Constitution VI).

- [ ] T014 [P] Modify `src/components/agent/OSActions.tsx` — v1 CopilotKit `web_view` action: (a) update `description` to mention image/video, (b) add `poster`/`autoplay`/`loop`/`muted` to the `parameters` array, (c) update the `handler` to import `classifyMediaTarget` from `src/lib/apps/media.ts` and mirror the v2 handler's media classification logic (build `{ mode, src, title?, poster?, autoplay?, loop?, muted? }` for media targets).
- [ ] T015 [P] Update `docs/dev/assistant/actions-and-tools.md` — the `web_view` row: update description to mention image/video, add the new params.
- [ ] T016 [P] Create or update the `docs/usage` page for `web_view`. Per reviewer finding: no dedicated usage page exists. Create `docs/usage/assistant/web-view.md` documenting the tool (including media params), or add a section to `docs/usage/assistant/using-the-assistant.md` if that page is more appropriate. Check the existing docs structure and place it consistently.

**Checkpoint**: All declaration surfaces, docs, and code are in sync.

---

## Phase 7: Quality Gates

- [ ] T017 Run `npx tsc --noEmit` — fix any type errors.
- [ ] T018 Run `npm run lint` — fix any lint errors.

**Checkpoint**: Feature complete, all gates pass.

---

## Dependencies & Execution Order

```
T001, T002 (Phase 1)
  └─► T003, T004, T005, T006 (Phase 2 — US1)
        └─► T007, T008 (Phase 3 — US2)
              └─► T009, T010 (Phase 4 — US3)
                    └─► T011, T012, T013 (Phase 5 — US4)
                          └─► T014, T015, T016 (Phase 6 — cross-cutting)
                                └─► T017, T018 (Phase 7 — quality gates)
```

**Parallel opportunities**: T001 ∥ T002; T005 ∥ T006; T009 ∥ T010; T011 ∥ T012; T013 ∥ T014 ∥ T015 ∥ T016; T017 ∥ T018.

**Note**: This is a single `dev_delegate` handoff — the Developer executes all tasks in one pass on the `web-view-media` branch. The task breakdown is for tracking and verification, not for parallel team assignment.

---

## Implementation Strategy

Single `dev_delegate` call with the spec directory path. The Developer:
1. Reads spec.md, design.md, plan.md, tasks.md from the spec directory.
2. Implements all tasks in dependency order.
3. Runs typecheck + lint.
4. Reports back with a summary of changes.

The result is a preview on the `web-view-media` feature branch — the user promotes or discards from the Topbar.
