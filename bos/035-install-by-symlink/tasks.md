# Tasks: Install Is a Symlink

**Spec**: `spec.md` · **Status**: implemented 2026-07-29

## Phase 1 — The shared scanner (FR-006, FR-007)

- [x] **T001** New `src/system/items/installed.ts`: `listInstalledItems()` doing the depth-2 scan, returning `{ id, itemPath, facets, origin, marketplaceId, broken }`. Plus `itemLinkPath`, `itemConfigDir`, `isItemInstalled`, `itemsFromMarketplace`, `RESERVED_ITEM_IDS`.
- [x] **T002** Provenance derived from the resolved symlink, never persisted (FR-009). Dangling links reported as `broken` rather than dropped.

## Phase 2 — Install / uninstall (FR-001 … FR-005)

- [x] **T003** `symlinkManager.ts` rewritten: six symlinks per item → `installItemLink()` (one link + `seedItemConfig()`) and `uninstallItemLink()`.
- [x] **T004** Reject the reserved id `config`; refuse to rebind an id already installed from a different source.
- [x] **T005** `seedItemConfig()` never overwrites, so a reinstall keeps the user's settings and uninstall leaves them behind.
- [x] **T006** `serviceInstaller.ts` installs via the item link and validates the manifest *through* it.
- [x] **T007** `ServiceRegistry`: `configDirPath` → `system/config/<id>`; installed services from the shared scan; its private `listNames` duplicate-scan helper deleted.
- [x] **T008** `ServiceManager` + `validateManifestAtStart` resolve entrypoints through the item link. The validator's second argument is now the service's OWN directory, not a root with the id joined on — **tests updated, 25/25 pass**.

## Phase 3 — No copies (FR-001)

- [x] **T009** `installMarketplaceService`: copy + `commitAll` removed; `installService(src, id)` symlinks the marketplace clone directly.
- [x] **T010** App facet: copy removed; `itemDirFor()` derives the ITEM directory to link (entrypoints may point below the item root, e.g. `items/lunar-lander/app/dist`).
- [x] **T011** `installBosPlugin`: copy to `dataDir()/bos-plugins/` removed; the item is linked and the loader reads its `plugin/` facet.
- [x] **T012** `installItemApp` writes nothing and links nothing — it derives and returns the manifest.
  - *Caught in testing: it still wrote `app.json` into `user-apps/items/<id>/app/` and linked THERE, which both created content in the user's own marketplace for someone else's app and collided with the correct link into the clone.*
- [x] **T013** `uninstallBosPlugin` no longer `fs.rm`s the plugin directory. That path resolves **through** the symlink, so it would have deleted the item's source out of the marketplace clone. It removes the item link instead.
- [x] **T014** `adoptSpec`'s copy deliberately kept — adoption is a fork, not an install.

## Phase 4 — Plugins as ordinary items (FR-008)

- [x] **T015** `loadAllPlugins()` scans installed items for a `plugin/` facet; `PLUGINS_DIR` and the dead `MODULES_KEY` removed.
- [x] **T016** Boot order fixed: plugins load **after** the migrations.
  - *Caught in testing: with the old ordering the plugin loaded nothing on the very boot that migrated it — `data/bos-plugins/` was no longer read and the item's `plugin/` facet did not exist yet.*

## Phase 5 — Capability grants

- [x] **T017** New `src/system/items/capabilities.ts` → `system/config/<id>/capabilities.json`. `resolveCapabilities()` migrates a legacy grant out of `app.json` on first read; `setAppCapabilities` writes BOS state and now works for marketplace apps too.

## Phase 6 — Guards and UI (FR-012, FR-013)

- [x] **T018** `removeMarketplace` refuses while installed items resolve into that clone, naming them.
- [x] **T019** Settings → Apps: "Uninstalled" section and Restore removed (`restoreApp` + `PATCH /api/apps` deleted); Purge moved next to Uninstall and shown only when `origin !== "marketplace"`; `purgeApp` enforces the same server-side.

## Phase 7 — Migration (FR-010, FR-011)

- [x] **T020** `migrate-installed-state.ts`: per-facet symlinks → one item link; seeds `system/config/<id>` from the OLD config location *before* removing the link that points at it; drops the two reader-less symlink trees; removes emptied facet dirs with `rmdir` (fails safe if anything unexpected remains).
- [x] **T021** `foldBosPlugins()`: a legacy `bos-plugins/<id>` copy is moved into its item as `plugin/` when the item is the user's own (repairing the pre-035 stub), else dropped; the move is committed; `data/bos-plugins/` is retired.
- [x] **T022** Wired into boot before the service registry. Idempotent. Never deletes from `user-apps` (FR-011).

## Phase 8 — Verification

- [x] **T023** `tsc --noEmit` clean across app + bastion + bastion UI; ESLint clean on all changed files; service tests 25/25.
- [x] **T024** Migration on real state: `system/{app,services}/` collapsed to `system/{config,live-avatar,terminal,workflows}`; `data/bos-plugins/` gone; live-avatar's plugin folded into its item and loading (`[bos-plugin:live-avatar] loaded v0.1.0`).
- [x] **T025** Services start under the new layout: `terminal` running on 3001, `workflows` running. `/apps/terminal/ -> 200`.
- [x] **T026** **SC-001 verified**: installing `welcome` from `bos-marketplace` left `user-apps` at the same commit with no new content; `data/system/welcome` → the marketplace clone; the app listed as `origin=marketplace, marketplaceId=bos-marketplace` (derived) and served 200.
- [x] **T027** **SC-003 verified**: uninstalling `welcome` removed one symlink; it vanished from `system/` and from the app list, and `data/marketplace/bos-marketplace/items/welcome/` was untouched.
- [x] **T028** **FR-012 verified**: `Cannot remove marketplace "bos-marketplace": 1 installed item(s) come from it (welcome). Uninstall them first.`
- [x] **T029** Capability grants verified BOS-side at `system/config/workflows/capabilities.json`, migrated out of `app.json` on first read.

- [x] **T030** Test suite brought onto the new model — the failures were tests asserting the *pre-035* layout:
  - `_worker-fixtures.ts`'s `installFixtureService` wrote a `system/services/<id>` directory; it now builds a real item plus the one install link (plus a new `installBrokenFixtureService` for the missing-entry case).
  - `ServiceRegistry.test.ts` laid items out flat and created per-facet links.
  - `integration.test.ts` asserted `system/services/<id>` and `system/app/<id>`; it now asserts one symlink, a real (non-symlink) seeded config dir, and the *absence* of the old facet dirs.
  - `local-marketplace.test.ts` asserted the opposite of the new contract ("no marketplace.json is ever written"); rewritten, plus new cases for reconciliation idempotency and pruning.
  - One assertion was passing for the wrong reason: `installService` autostarts, and the test expected `stopped` only because the pre-035 path could not resolve the entrypoint at all. It now expects `running`.
  - **119/119 pass.**
- [x] **T031** `isReadOnly` deleted from `/api/services/<id>/config`. It refused config edits for marketplace-sourced items — correct when installing *copied* the item into a read-only clone, wrong now that config is BOS-owned state. `readOnly: false` is retained in the response so the client contract is unchanged.
- [x] **T032** End-user docs corrected (`docs/usage/apps/settings.md`, `docs/usage/building-and-modifying/building-apps.md`) — they still described Restore and "uninstall keeps its files".

## Not verified live

- Two marketplaces offering the same item id (the refuse-to-rebind path is exercised only by an accidental double-install during development).
- A `spec`/`hooks` facet resolving through the item link — no installed item currently has one.
