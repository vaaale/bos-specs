# Tasks: user-apps is a Marketplace (Layout & Manifest Parity)

**Spec**: `spec.md` · **Status**: implemented 2026-07-29

## Phase 1 — Layout (FR-001)

- [x] **T001** Add `userAppsItemsDir()` = `dataDir()/user-apps/items` in `src/lib/marketplace/client.ts`. `cloneDir(LOCAL)` keeps returning the repo **root**, so manifest-relative entrypoints (`items/<id>/app`) resolve identically for local and remote marketplaces.
- [x] **T002** Point `itemsRoot()` in `src/lib/apps/store.ts` at `user-apps/items` — this is what redirects Build Studio's app output (FR-009).
- [x] **T003** Point the user-apps scan in `src/core/service/ServiceRegistry.ts` at `user-apps/items`, making it symmetric with the `marketplace/<id>/items` branch immediately below it.
- [x] **T004** Move install destinations under `items/`: the app-facet copy and `installMarketplaceService`'s `dest`.

## Phase 2 — Manifest (FR-002, FR-003, FR-004)

- [x] **T005** Replace `scanUserAppsManifest()` (returned a synthesized manifest) with `scanLocalItems()` (returns inferred items only), emitting `items/<id>/…` entrypoints.
- [x] **T006** Add `reconcileLocalManifest()`: read the on-disk manifest, scan `items/`, keep declared entries untouched, append newly discovered ones, prune vanished ones.
- [x] **T007** `readManifest(LOCAL)` reconciles then returns; the local special-case no longer synthesizes.
- [x] **T008** Refuse to overwrite a malformed `marketplace.json` — throw and name the file. It is the user's curated content.
- [x] **T009** Decide "changed" by comparing the **serialized** manifest to the file's bytes, then write + commit only if different.
  - *First implementation compared inferred facet shapes and committed a manifest whose only difference was a trailing newline. Caught in testing; replaced with the content comparison.*
- [x] **T010** Commit summaries reflect what happened: `initialise` / `sync (N added, M removed)` / `repair`.

## Phase 3 — Identity (FR-006, FR-007)

- [x] **T011** `requestUsername()` reads the bastion-injected `x-bos-username` via `next/headers`, returning null outside a request scope. No new plumbing — the header already existed and is already consumed by `/api/system/session`.
- [x] **T012** `defaultLocalMarketplaceName()` → `<username>-marketplace`, falling back to the OS account name, then `my-marketplace`. Only used when no manifest exists, so naming happens lazily on first catalog read, never at boot.
- [x] **T013** The local catalog entry reports the repo manifest's own `name` instead of a hardcoded "My Apps".
- [x] **T014** `addMarketplace` rejects a remote whose manifest id equals the local marketplace's id, and still rejects the literal reserved `user-apps` (which `cloneDir` would otherwise alias onto the local directory).

## Phase 4 — Migration (FR-008)

- [x] **T015** New `src/lib/marketplace/migrate-user-apps.ts`: move `user-apps/<id>/` → `user-apps/items/<id>/` for any top-level directory carrying an item marker.
- [x] **T016** Re-point installed-state symlinks (`system/{services,app,settings,hooks}/<id>`, `config/<id>`, `specs/external-specs/<id>`, `docs/external-docs/<id>`), rewriting only links that actually pointed into the old location.
- [x] **T017** Repair pre-034 manifest entrypoints in reconciliation: if a declared path is absent but its `items/`-prefixed equivalent exists, rewrite it.
  - *Found in testing — the migration moved the directories but left `terminal/app` in the manifest, so every local install would have failed on a missing source path.*
- [x] **T018** Run the migration at boot from `src/instrumentation.ts`, before the service registry scans `items/`. Idempotent.

## Phase 5 — Verification

- [x] **T019** `tsc --noEmit` and ESLint clean.
- [x] **T020** Migration on a real flat repo: 3 items moved, 4 symlinks re-pointed, **0 dangling**.
- [x] **T021** Entrypoint repair: all declared paths resolve after migration.
- [x] **T022** Curated fields preserved — `terminal` and `workflows` keep their authored descriptions and facets.
- [x] **T023** Idempotency: repeated catalog reads produce no commit and leave a clean working tree.
- [x] **T024** Auto-discovery: a hand-created `items/scratch-probe/app/index.html` is added (`sync … (1 added)`); deleting it prunes the entry (`sync … (1 removed)`).

- [x] **T025** Recognise `app/app.json` (not just `app/index.html`) as an app facet, inferring `runtime: "plugin-served"` from its `appUrl` plus the declared `name`/`icon`.
  - *Found after implementation: `live-avatar` sat in `user-apps/items/` — installed and running — yet was absent from the "My Apps" catalog, because a plugin-served app has no `index.html` of its own.*
- [x] **T026** Marketplace app comments corrected; the app itself needed no functional change (it consumes items from the API and never sees paths). Install from the local marketplace verified end-to-end.

## Known gap (documented, not fixed)

Installing a plugin item from a remote marketplace leaves a **stub** in
`user-apps/items/<id>/` (just `app/app.json`); the plugin itself goes to
`dataDir()/bos-plugins/`. Re-installing from the local marketplace registers the
app but cannot restore the plugin, and the `integration`/`voiceEngine` facet is
not inferable from disk. Pre-existing.

## Not verified live

- **FR-006** naming only fires when `user-apps` is uninitialised (a wipe or first login); the code path is typechecked but was not exercised against a fresh data dir.
- **FR-007** duplicate rejection is in place but not exercised against a real remote whose manifest id collides.
- **FR-009** Build Studio output landing in `items/` follows from T002 but was not exercised by an actual build.
