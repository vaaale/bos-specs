# Tasks: 027 VFS Mount Points, SpecFS, Feature Context, Provider Registry, and Marketplace

Feature branch: `027-vfs-specfs-marketplace`. Tasks are grouped by phase; `[P]` marks work that can proceed in parallel with siblings within the phase.

## Phase 1 — VFS Mount Points + Feature Context (Foundation)

- [ ] T001. `src/os/fs-types.ts` (new) — Define `FSBackend` interface and `MountPoint` type. Path passed to a backend is relative to the mount root.
- [ ] T002. `src/os/fs/local-fs.ts` (new) — `LocalFS implements FSBackend`. Mechanical extraction of current `vfs.ts` path resolution + `fs/promises` calls. No logic change.
- [ ] T003. `src/os/vfs.ts` (modify) — Add `registerMount(prefix, backend)`, `resolveMount(vfsPath)`, and update all nine public VFS functions to delegate to the backend if a mount matches; fall through to LocalFS behaviour otherwise.
- [ ] T004. `src/os/types.ts` (modify) — Add `FeatureContext` and `FeatureContextFile` types.
- [ ] T005. `src/store/os-store.ts` (modify) — Add `activeFeature` state field and `setFeature`, `clearFeature`, `touchSpec`, `touchSource` actions; persist changes via `POST /api/feature-context`.
- [ ] T006. `src/app/api/feature-context/route.ts` (new) — GET / POST / DELETE handlers for `data/config/feature-context.json`.
- [ ] T007. Ensure `data/vfs/Documents/Specs/` stub directory is created during VFS initialisation so it appears in directory listings.
- [ ] T008. `npx tsc --noEmit` and `npm run lint` green.

## Phase 2 — GitFS + SpecFS

- [ ] T009. `src/os/fs/git-fs.ts` (new) — `abstract class GitFS implements FSBackend`. Git operations via `execFile('git', …)`: `ensureBranch`, `currentBranch`, `getDiff`, `stageAll`, `commit`, `merge`, `hasUncommitted`. FSBackend read/write methods rooted at `repoPath`.
- [ ] T010. `src/os/fs/spec-fs.ts` (new) — `class SpecFS extends GitFS`. `writeText`/`writeBuffer` enforce active feature context (throw `SpecFSNoContextError` if absent), call `ensureBranch`, write file, update `touchedSpecs` in context file, schedule 2 s debounced commit. Flush: `stageAll → getDiff → generateCommitMessage → commit`. Read/list/stat/remove/rename delegate to GitFS base.
- [ ] T011. `generateCommitMessage(diff)` helper — call configured AI provider; fallback to `"Update specs"` on any error.
- [ ] T012. `src/lib/specs/seed.ts` (modify) — Init `data/specs/user/` as a git repo with `spec-store.json` on first run. One-time migration from `BOS_SPECS_ROOT/user-specs` if present. Redirect system spec seeding to `data/specs/system/`.
- [ ] T013. Register SpecFS mount at server startup: `registerMount('/Documents/Specs', new SpecFS(path.join(dataDir(), 'specs/user')))`.
- [ ] T014. Update spec-write tool — replace `specsRoot()` + direct `fs.writeFile` calls with `vfs.writeText('Documents/Specs/…')`.
- [ ] T015. `npx tsc --noEmit` and `npm run lint` green.

## Phase 3 — Spec Provider Registry

- [ ] T016. `src/lib/specs/provider.ts` (new) — `SpecProvider` interface and `SpecRegistry` class aggregating providers in order: builtin → user → marketplace.
- [ ] T017. [P] `src/lib/specs/providers/builtin.ts` (new) — `BuiltinSpecProvider`. Scans `data/specs/system/` for `spec-store.json`. `canWrite: false`.
- [ ] T018. [P] `src/lib/specs/providers/user.ts` (new) — `UserSpecProvider`. Reads `data/specs/user/spec-store.json`. `canWrite: true`.
- [ ] T019. [P] `src/lib/specs/providers/marketplace.ts` (new) — `MarketplaceSpecProvider`. Reads `data/config/marketplaces.json`, scans `data/specs/marketplace/<id>/`. `canWrite: false`.
- [ ] T020. `src/lib/specs/stores.ts` (rewrite) — Replace directory scan with `SpecRegistry.listAll()`. Preserve public API: `listStores`, `getStore`, `defaultWritableStore`.
- [ ] T021. Delete `src/os/specs-dir.ts` — remove file and all imports.
- [ ] T022. `npx tsc --noEmit` and `npm run lint` green.

## Phase 4 — App Provider Registry

- [ ] T023. `src/lib/apps/provider.ts` (new) — `AppProvider` interface and `AppRegistry` class. Extend `AppManifest` in `src/os/types.ts` with `runtime`, `source`, `marketplaceId`, `marketplaceItemId`, `canAdoptSpec`.
- [ ] T024. [P] `src/lib/apps/providers/builtin.ts` (new) — `BuiltinAppProvider`. Wraps `_manifests.generated`; marks all `runtime: 'native'`, `source: 'builtin'`.
- [ ] T025. [P] `src/lib/apps/providers/local.ts` (new) — `LocalAppProvider`. Scans `data/apps/local/` for `app-manifest.json` files. Creates dir if absent.
- [ ] T026. [P] `src/lib/apps/providers/marketplace.ts` (new) — `MarketplaceAppProvider`. Parses `marketplace.json` from cloned repos in `data/apps/marketplace/<id>/`.
- [ ] T027. `src/os/apps.ts` + `src/components/apps/registry.tsx` (modify) — Replace `BUILTIN_APPS` direct import with `AppRegistry.listAll()`. `getApp` delegates to `AppRegistry.resolveApp`.
- [ ] T028. `npx tsc --noEmit` and `npm run lint` green.

## Phase 5 — Marketplace

- [ ] T029. `src/lib/marketplace/schema.ts` (new) — `MarketplaceManifest`, `MarketplaceItem`, `RegisteredMarketplace` types.
- [ ] T030. `src/lib/marketplace/client.ts` (new) — `MarketplaceClient`: `addMarketplace`, `syncMarketplace`, `listItems`, `adoptSpec`, `installApp`.
- [ ] T031. `src/app/api/marketplace/route.ts` (new) — REST handlers: list, add, remove, sync, adopt-spec, install-app.
- [ ] T032. `src/apps/marketplace/manifest.ts` (new) — id `"marketplace"`, name `"Marketplace"`, icon `"Store"`, singleton, builtin.
- [ ] T033. `src/apps/marketplace/index.tsx` (new) — Three-panel UI: marketplace list + add, item grid with search/filter, item detail with Run/Install/Adopt actions.
- [ ] T034. `npx tsc --noEmit` and `npm run lint` green.

## Phase 6 — Docker / Bastion

- [ ] T035. [P] `bastion/src/provision.ts` (modify) — Remove `BOS_SPECS_ROOT` env var injection.
- [ ] T036. [P] `bastion/src/docker.ts` (modify) — Remove `BOS_SPECS_ROOT` from container env array. No separate specs volume needed.
- [ ] T037. [P] Remove `BOS_SPECS_ROOT` from `.env.example`, `docs/dev/deployment.md`, and any other documentation.
- [ ] T038. Bastion `tsc --noEmit` green.

## Phase 7 — Promotion + Developer Agent Wiring

- [ ] T039. `src/lib/specs/promote.ts` (new) — `promoteFeature(ctx)`: merge spec branch → main; return `{ kind: 'spec-only' }` or `{ kind: 'source-included', branchName }` based on `touchedSourcePaths`; clear feature context on success.
- [ ] T040. Build Studio promote UI (modify) — Read `touchedSourcePaths` before showing confirm dialog; display fast-path or full-path message accordingly.
- [ ] T041. Developer agent: when active feature context exists, checkout `bos/feat/<id>` in BOS source before modifying files; PATCH `/api/feature-context` with touched source paths after each commit.
- [ ] T042. `npx tsc --noEmit` and `npm run lint` green.

## Closeout

- [ ] C1. Merge 027 into `018-external-spec-store` — mark FR-005/FR-007 Superseded.
- [ ] C2. Merge 027 into `020-branch-coupled-specs` — mark Fulfilled.
- [ ] C3. Merge 027 into `009-installed-apps` — update to three-source model.
- [ ] C4. Update `discrepancies.md` — remove BOS_SPECS_ROOT drift entries.
- [ ] C5. Update `overview.md` — add 027 summary entry.
- [ ] C6. Update `docs/dev/architecture-overview.md` — VFS, SpecFS, Feature Context, Provider Registry.
- [ ] C7. Update `docs/usage/` — end-user guide: Feature Context workflow, Marketplace, spec writing from any app.
