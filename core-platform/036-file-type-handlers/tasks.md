# Tasks: File-Type Handler Registry & "Open With"

**Input**: Design documents from `/Specs/user-specs/core-platform/036-file-type-handlers/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), design.md (required — ADRs and file plan)

**Tests**: No explicit test framework required by the spec. Verification is via acceptance scenarios (manual / Playwright e2e at `implement` time). Quality gates: `npx tsc --noEmit` + `npm run lint`.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4, US5)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: No new packages or project init needed. This feature uses only existing BOS dependencies.

- [ ] T001 Run `npx tsc --noEmit` to establish a clean baseline before changes

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented. This phase creates the manifest type, the shared client-safe helpers, the server registry, the selection store, the API route, and the store.ts mirroring.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T002 Add `AppFileHandlerDeclaration` interface and `AppManifest.fileHandlers?` field in `src/os/types.ts`. The interface: `{ type: string; capabilities: ("render" | "edit")[]; label?: string; default?: boolean; paramShape?: { url?: "raw"; title?: "basename" } }`. Mirror the `AppEventHandlerDeclaration` / `AppManifest.eventHandlers` pattern (spec 034). (design.md §4, FR-001)
- [ ] T003 [P] Create `src/os/file-handlers.ts` — framework-free, client-safe (no React, no Node imports, no `server-only`). Export: (a) `baseMime(mime: string): string` — strip `; param` suffix, lowercase; (b) the authoritative matching MIME map (moved from `src/lib/mime.ts`) + `mimeForPath(p: string): string` + `fileBaseMime(p: string): string` (mimeForPath then baseMime); (c) `rawUrlFor(path: string): string` = `"/api/fs/raw?path=" + encodeURIComponent(path)`; (d) `buildLaunchParams(decl: AppFileHandlerDeclaration, path: string, action: "open" | "edit"): Record<string, unknown>` — applies `paramShape`: if `paramShape.url === "raw"` set `url = rawUrlFor(path)`; if `paramShape.title === "basename"` set `title = basename(path)` (use `path.split("/").pop()`); always set `path` and `action`. (design.md §4, ADR-3, ADR-7, ADR-8, FR-014)
- [ ] T004 [P] Modify `src/lib/mime.ts` — replace the MIME map and `mimeForPath` body with re-exports from `src/os/file-handlers.ts` so the three existing server-side importers (`/api/services/[id]/config-app`, `/app/apps/[...slug]`, `assistant/tools/server/view-image`) are untouched. Keep the `import path from "path"` (Node builtin, used only for `extname`); the file remains Node-only. (design.md §4, ADR-8, S-1)
- [ ] T005 [P] Create `src/lib/file-handlers/registry.ts` — `server-only`. Export: (a) `matchDeclared(declType: string, baseMime: string): boolean` — exact base-type match OR `type/` prefix match (e.g. `image/` matches `image/png`); (b) `handlersFor(baseMime: string): Promise<{ appId, name, icon, label, capabilities, isDefault, selected }[]>` — reads `BUILTIN_APPS` (`src/os/apps.ts`) + `listInstalledManifests()` (`src/lib/apps/store.ts`), filters manifests with `fileHandlers` entries where `matchDeclared` is true, maps to the view shape; (c) `effectiveSelected(baseMime: string): Promise<{ appId: string; decl: AppFileHandlerDeclaration } | null>` — checks user selection (from `selection.ts`) first (only if that app is installed AND render-capable for the type), else manifest `default: true` (only if installed AND render-capable), else null. Imports `baseMime`/`fileBaseMime` from `src/os/file-handlers.ts` (NOT from `src/lib/mime.ts`). (design.md §4, ADR-1, ADR-4, FR-002, FR-003)
- [ ] T006 [P] Create `src/lib/file-handlers/selection.ts` — `server-only`. Export: (a) `readSelection(): Promise<Record<string, string>>` — reads `data/system/file-handlers.json` (keyed by base MIME → appId); returns `{}` if file missing; (b) `writeSelection(mime: string, appId: string | null): Promise<void>` — reads current, sets/deletes the key, writes atomically via `writeFileAtomic` (`src/os/atomic-write.ts`) to `path.join(dataDir(), "system", "file-handlers.json")` (`src/os/data-dir.ts`); creates the `data/system/` dir if missing. (design.md §4, ADR-5, FR-010, FR-011)
- [ ] T007 [P] Create `src/app/api/file-handlers/route.ts` — Next.js App Router route handler. `GET ?mime=<type>` → `{ handlers: [...], selected: appId | null }` (calls `registry.handlersFor` + `registry.effectiveSelected` after normalizing the query param via `baseMime`). `POST { mime: string, appId?: string }` → if `appId` present, `selection.writeSelection(mime, appId)`; if absent, `selection.writeSelection(mime, null)` (clear, revert to manifest default). Returns 200 with the updated view. (design.md §4, FR-003, FR-010)
- [ ] T008 [P] Modify `src/lib/apps/store.ts` — mirror `fileHandlers` through the three places `eventHandlers` is handled: (a) the `InstalledApp` type (add `fileHandlers?: AppFileHandlerDeclaration[]`); (b) `readApp` (read `fileHandlers` from `app.json`); (c) `toManifest` (emit `fileHandlers` onto the `AppManifest`). The `eventHandlers` lines are the template. (design.md §4, FR-002)
- [ ] T009 [P] Modify `src/lib/os-client.ts` — add `fileHandlersClient` with two methods: `list(mime: string)` → `GET /api/file-handlers?mime=<mime>` returning `{ handlers, selected }`; `setSelected(mime: string, appId?: string)` → `POST /api/file-handlers { mime, appId }`. Same shape as the existing `fsClient`/`settingsClient`. (design.md §4)
- [ ] T010 [P] Modify `src/components/apps/IframeApp.tsx` — add `withFileParams(url: string, params: Record<string, unknown>): string` (sibling to the existing `withEventParams`). Encodes `path` → `bosFilePath`, `action` → `bosFileAction`, and `title` (when present) → `bosFileTitle` as `bos*`-prefixed query params appended to the iframe `src` URL. Does NOT encode `url` (the iframe's `src` is always `manifest.url`; ADR-3). Must not clobber the app's own query params (R-2). (design.md §4, ADR-2, ADR-3, FR-014, R-2)

**Checkpoint**: Foundation ready — all types, helpers, registry, selection, API, and delivery plumbing exist. User story implementation can now begin.

---

## Phase 3: User Story 1 - App declares file types it can handle (Priority: P1) 🎯 MVP

**Goal**: The registry works end-to-end: a manifest declaration is discoverable and queryable via the API. This is the platform primitive everything else builds on.

**Independent Test**: Query `GET /api/file-handlers?mime=text/html` and confirm the response includes a handler entry (once US2 adds the html-viewer declaration). Before US2, query a type with no handlers and confirm an empty list (no crash, no phantom default).

### Implementation for User Story 1

- [ ] T011 [US1] Verify the registry correctly derives handler lists from `BUILTIN_APPS` + `listInstalledManifests()` by confirming that a manifest with a `fileHandlers` declaration is returned by `handlersFor` for the matching MIME type, and that uninstalled apps' declarations are NOT returned. (This is a code-verification step: read the registry implementation, confirm the filter logic matches FR-002. No new code — validates T005.)
- [ ] T012 [US1] Verify `matchDeclared` handles exact base-type matches AND `type/` prefix matches correctly (e.g. `image/` matches `image/png` but not `image/pngfoo`; `text/html` matches `text/html` but not `text/plain`). (Code verification of T005's `matchDeclared`.)

**Checkpoint**: US1 mechanism verified — the registry correctly reflects installed apps' declarations.

---

## Phase 4: User Story 2 - HTML files open in web_view by default (Priority: P1)

**Goal**: `html-viewer` is registered as the default render handler for `text/html` via its manifest. No code change to `html-viewer/index.tsx` — it already reads `params.url` / `params.title`.

**Independent Test**: After this phase, `GET /api/file-handlers?mime=text/html` returns `html-viewer` as a render-capable, default handler. The `effectiveSelected` for `text/html` returns `html-viewer`.

### Implementation for User Story 2

- [ ] T013 [US2] Add the `fileHandlers` declaration to `src/apps/html-viewer/manifest.ts`: `fileHandlers: [{ type: "text/html", capabilities: ["render"], label: "Web View", default: true, paramShape: { url: "raw", title: "basename" } }]`. No change to `src/apps/html-viewer/index.tsx`. (design.md §4, FR-005)
- [ ] T014 [US2] Verify `buildLaunchParams` with html-viewer's declaration produces the correct params: for a file at `/Documents/report.html` with `action="open"`, the result is `{ path: "/Documents/report.html", action: "open", url: "/api/fs/raw?path=%2FDocuments%2Freport.html", title: "report.html" }`. (Code verification of T003 against FR-005 / design.md §6.)

**Checkpoint**: US2 complete — `html-viewer` is the default `text/html` handler. Combined with US3, double-clicking any `.html` file opens it in web_view.

---

## Phase 5: User Story 3 - Double-click opens a file with the selected handler (Priority: P1)

**Goal**: In the Files app, double-click on a file resolves its MIME type, looks up the selected handler, and launches it. No handler → existing in-app behavior (backward-compatible).

**Independent Test**: In the Files app, double-click a `.html` file → a web_view preview window opens showing the rendered HTML. Double-click a `.txt` file → the existing in-app text editor (no handler registered for `text/plain`).

### Implementation for User Story 3

- [ ] T015 [US3] Modify `src/apps/files/index.tsx` — in the `openEntry` function (the double-click handler), for file entries (gated on `entry.type === "file"`; directories keep their existing `setCwd` behavior): (a) compute the file's base MIME via `fileBaseMime(entry.path)` (from `src/os/file-handlers.ts`); (b) call `fileHandlersClient.list(baseMime)` (from `src/lib/os-client.ts`); (c) if the response has a `selected` handler, call `buildLaunchParams(selectedDecl, entry.path, "open")` (from `src/os/file-handlers.ts`) and `launch(selected.appId, params)` via `useOSStore`; (d) if `launch` returns `null` (app not launchable, R-3) OR no handler is selected, fall through to the existing in-app behavior (image preview / text editor) — byte-for-byte unchanged (SC-003). (design.md §4, ADR-6, FR-006, FR-007, FR-012, R-3)

**Checkpoint**: US3 complete — double-click on an HTML file opens it in web_view. Double-click on other files is unchanged.

---

## Phase 6: User Story 4 - Right-click "Open with" context menu (Priority: P1)

**Goal**: Right-clicking a file in the Files app shows an "Open with" group listing every registered handler for the file's MIME type, each as "Open with \<label\>" with the target app's icon. Picking one launches that app with the file. No handlers → no "Open with" group (menu unchanged).

**Independent Test**: Right-click a `.html` file → the menu shows "Open with Web View" (with html-viewer's `Code2` icon) and "Open with Editor" (if registered). Picking one opens the file in that app. Right-click a `.txt` file → no "Open with" group, just the existing Download action.

### Implementation for User Story 4

- [ ] T016 [US4] Modify `src/apps/files/index.tsx` — in the context menu render (the `menu &&` block), for file entries only (gated on `menu.entry.type === "file"`, FR-012): (a) fetch the handler list via `fileHandlersClient.list(fileBaseMime(menu.entry.path))` (triggered when the menu opens on a file); (b) if the handler list is non-empty, render an **"Open with" section** ABOVE the existing actions: a section header (uppercase, `text-[10px]`, `text-white/40`), then one row per handler — each row: the target app's manifest `icon` (lucide, 14px), the label `"Open with " + (handler.label ?? handler.name)`, and a checkmark (`lucide Check`, 14px, `text-white/70`, right-aligned) on the row where `handler.selected` is true; (c) a hairline divider (`border-t border-white/10`) between the "Open with" group and the pre-existing actions; (d) on row click: determine `action` = `"open"` if the handler is render-capable, `"edit"` if edit-only (ADR-6); call `buildLaunchParams(handler.decl, menu.entry.path, action)` and `launch(handler.appId, params)`; (e) if the handler list is empty, render the menu exactly as it does today (no section, no divider) — SC-003. (design.md §4, ADR-6, FR-008, FR-009, FR-012, mockup.html)
- [ ] T017 [US4] Verify the "Open with" menu correctly shows hidden handler apps (e.g. `html-viewer`, `hidden: true`) by their own label and icon — the Files app must NOT filter handlers by `hidden` (R-4). (Code verification of T016.)

**Checkpoint**: US4 complete — right-click shows "Open with" entries for registered handlers; picking one opens the file in that app.

---

## Phase 7: User Story 5 - Choose which app is selected (default) for a type (Priority: P2)

**Goal**: The user can change the selected handler for a MIME type. Picking a render-capable "Open with" entry both opens the file now AND sets it as the new selected (default) handler. The checkmark moves accordingly.

**Independent Test**: Change the selected handler for `text/html` from `html-viewer` to another registered handler, then double-click an `.html` file — it now opens in the newly selected app. The checkmark in the "Open with" menu reflects the current selection.

### Implementation for User Story 5

- [ ] T018 [US5] Modify `src/apps/files/index.tsx` — in the "Open with" row click handler (T016d), after the `launch` call: if the picked handler is **render-capable** (its `capabilities` array includes `"render"`), call `fileHandlersClient.setSelected(baseMime, handler.appId)` to persist the new selection (FR-010 Option A: "always open with"). If the picked handler is **edit-only**, do NOT call `setSelected` (ADR-6: selection must be render-capable). This is a fire-and-forget POST (no need to await before the launch). (design.md §4, ADR-6, FR-010)
- [ ] T019 [US5] Verify the selection persistence round-trip: after `setSelected("text/html", "some-app")`, a subsequent `fileHandlersClient.list("text/html")` returns `selected: "some-app"`. After uninstalling `some-app`, `effectiveSelected` for `text/html` falls back to the manifest default (or null). (Code verification of T006 + T005 + T007, FR-010, FR-011.)

**Checkpoint**: US5 complete — the user's "always open with" choice persists and affects subsequent double-clicks.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Documentation, quality gates, and cross-cutting verification.

- [ ] T020 [P] Update `docs/dev/apps/built-in-apps.md` — document the new `fileHandlers` manifest field (shape, `capabilities`, `label`, `default`, `paramShape` vocabulary). Reference design.md §6 (the open-file launch contract).
- [ ] T021 [P] Create `docs/dev/apps/file-handlers.md` — the open-file launch contract for handler app authors: what params a handler receives (built-in: direct `params`; installed iframe: `bos*` query params), the `paramShape` vocabulary, and the handler's obligation (read/preview/edit the file when launched with one). Reference design.md §6 and ADR-2/ADR-3.
- [ ] T022 [P] Update `docs/dev/extending-bos.md` — add an "Add a file handler" recipe: (1) declare `fileHandlers` in your manifest, (2) implement the contract (read `params.path` / `params.action` for built-in, or `bosFilePath` / `bosFileAction` for iframe), (3) for built-in render handlers, optionally declare `paramShape: { url: "raw" }` to receive the raw-file URL.
- [ ] T023 Run `npx tsc --noEmit` — must pass with zero errors.
- [ ] T024 Run `npm run lint` — must pass with zero errors.
- [ ] T025 [P] Verify SC-003 (backward compatibility): for a file type with no registered handler (e.g. `.txt`), the Files app's open and right-click behavior is byte-for-byte the existing in-app behavior. No "Open with" section, no divider, double-click opens the text editor, right-click shows only Download.
- [ ] T026 [P] Verify SC-004 (no per-app code in core): registering a new test handler (add a `fileHandlers` entry to a test app's manifest) makes it appear in the "Open with" menu with zero changes to the Files app or the registry code.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — **BLOCKS all user stories**
  - T002 blocks T003, T004, T005 (all reference the new type)
  - T003 blocks T004 (mime.ts re-exports from the shared module)
  - T005 depends on T002, T003 (imports types + helpers from the shared module)
  - T006 is independent of T005 (separate file)
  - T007 depends on T005, T006 (calls both)
  - T008 depends on T002 (uses the new type)
  - T009 is independent (thin fetch wrapper)
  - T010 is independent (IframeApp change)
- **User Stories (Phases 3–7)**: All depend on Foundational phase completion
  - US1 (Phase 3): verification only — no new code
  - US2 (Phase 4): depends on US1 (the registry must work before a declaration is added)
  - US3 (Phase 5): depends on US2 (the selected handler must exist for double-click to launch it)
  - US4 (Phase 6): depends on US2 (the "Open with" list is populated from the registry)
  - US5 (Phase 7): depends on US4 (the selection change happens on the "Open with" pick)
- **Polish (Phase 8)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) — verification only
- **User Story 2 (P1)**: Depends on US1 — the registry must correctly derive handlers before a declaration is meaningful
- **User Story 3 (P1)**: Depends on US2 — needs a registered handler (html-viewer) to launch on double-click
- **User Story 4 (P1)**: Depends on US2 — needs registered handlers to list in the menu
- **User Story 5 (P2)**: Depends on US4 — the selection change is triggered from the "Open with" pick

### Within Each Phase

- Types before modules (T002 before T003–T005)
- Shared helpers before server modules (T003 before T004, T005)
- Registry + selection before API route (T005, T006 before T007)
- Manifest declaration before Files app changes (T013 before T015, T016)
- Double-click before right-click (T015 before T016)
- Right-click before selection persistence (T016 before T018)

### Parallel Opportunities

- Within Phase 2: T003, T004, T005, T006, T007, T008, T009, T010 are all [P] (different files, no cross-dependencies after T002 is done)
- Within Phase 8: T020, T021, T022, T025, T026 are [P] (docs and verification, different files)

---

## Implementation Strategy

### MVP First (Phases 1–4)

1. Complete Phase 1: Setup (baseline typecheck)
2. Complete Phase 2: Foundational (types, helpers, registry, selection, API, store, IframeApp)
3. Complete Phase 3: US1 (verify the registry works)
4. Complete Phase 4: US2 (html-viewer declares text/html)
5. **STOP and VALIDATE**: `GET /api/file-handlers?mime=text/html` returns html-viewer as the selected handler. `effectiveSelected("text/html")` returns html-viewer.

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. US1 + US2 → Registry works, html-viewer registered (MVP!)
3. US3 → Double-click opens HTML in web_view
4. US4 → Right-click "Open with" menu
5. US5 → Selection persistence ("always open with")
6. Polish → Docs, typecheck, lint, SC verification
7. Each story adds value without breaking previous stories

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
- **Constitution gates**: `npx tsc --noEmit` + `npm run lint` must pass at every checkpoint (T023, T024)
- **SC-003 is the critical backward-compatibility invariant**: the no-handler path must be byte-for-byte the existing behavior. T025 verifies this.
- **SC-004 is the extensibility invariant**: adding a handler is manifest-only. T026 verifies this.
