# Tasks: 027 VFS Mount Points, SpecFS, Feature Context, Provider Registry, and Marketplace

Feature branch: `027-vfs-specfs-marketplace`. Task ids match the plan phases. `[P]` = parallelizable within its phase. `[T]` = test task (unit/e2e), first-class per the "high-quality code is essential" note.

## Phase 1 — VFS Mount Points + server-authoritative Feature Context

- [ ] P1.1. `src/os/fs-types.ts` (new) — `FSBackend` (mirrors the current VFS surface) + `MountPoint`. Backend paths are mount-relative.
- [ ] P1.2. `src/os/fs/local-fs.ts` (new) — `LocalFS implements FSBackend`; mechanical extraction of current `vfs.ts` resolution + `fs/promises`. No behaviour change.
- [ ] P1.3. `src/os/vfs.ts` (modify) — `registerMount`, `resolveMount`; all nine functions delegate on match, else fall through to `LocalFS`.
- [ ] P1.4. [T] Unit test — `resolveMount` path-escape jail (`..`, absolute, encoded traversal) at the mount boundary.
- [ ] P1.5. `src/os/types.ts` (modify) — `FeatureContext`, `FeatureContextFile`.
- [ ] P1.6. `src/lib/specs/feature-context.ts` (new, server-only) — single-writer module over `data/config/feature-context.json`: in-process async mutex + atomic RMW. `getActive`/`setActive(id)`/`clear`/`patch`. `setActive` flushes any current feature, then create-or-reuses `id`'s branch/worktree, then records it active (one verb; "start new" vs "resume existing" is UI-only). Sanitize `id` against `^[a-z0-9-]+$` in `setActive`.
- [ ] P1.7. [T] Unit test — `id` sanitization rejects invalid ids; concurrent `patch` calls do not lose updates.
- [ ] P1.8. `src/app/api/feature-context/route.ts` (new) — GET / POST(set) / DELETE(clear) / PATCH(append touched path); all via the module. No whole-file replace from the client.
- [ ] P1.9. `src/store/os-store.ts` (modify) — `activeFeature` read-only mirror; intent actions that **await** the API; cross-tab sync (storage event / refresh-on-focus).
- [ ] P1.10. `npx tsc --noEmit` + `npm run lint` green.

## Phase 2 — SpecFS adapter + worktree writes + Promotion

- [ ] P2.1. `src/os/fs/git-fs.ts` (new) — thin `GitFS` helpers over `execFile('git', …)` reused by SpecFS: worktree add/prune, `hasUncommitted`, `stageAll`, `commit`, `merge --abort`, fast-forward. **No base checkout.**
- [ ] P2.2. `src/os/fs/spec-fs.ts` (new) — `FSBackend` adapter over `src/lib/dev/spec-fs.ts`: resolve active branch from the feature-context module; `SpecFSNoContextError` when none; **ref-pinned reads** via `readFileAtBranch`.
- [ ] P2.3. Worktree write routing — Supervisor worktree when a preview exists; else self-provision `data/specs/user-worktrees/<branch>/` via `git worktree add`.
- [ ] P2.4. Debounced commit (2 s) via `commitOnSave`; `generateCommitMessage(diff)` with **bounded diff** (truncate + file-count cap) and deterministic fallback.
- [ ] P2.5. `touchedSpecs` appended via the feature-context module `patch` (shared mutex) — no direct file write.
- [ ] P2.6. Rewire `editFile` (spec layer, not `FSBackend`) to route through the active branch.
- [ ] P2.7. `flushPending(branch)` — cancel debounce + synchronous `stageAll → commit`; precondition helper for committed-state reads.
- [ ] P2.8. Startup sweep — `hasUncommitted` on canonical repo + active worktrees → recovery commit.
- [ ] P2.9. `src/lib/specs/seed.ts` (modify) — init `data/specs/user/` git repo + `spec-store.json` (`writable: true`); one-time migrate legacy `BOS_SPECS_ROOT/user-specs`; re-seed system specs into `data/specs/system/`.
- [ ] P2.10. Register SpecFS mount **after** ensuring `data/specs/user/` exists (explicit ordering).
- [ ] P2.11. `src/lib/specs/promote.ts` (new) — `flushPending` → reconcile `main` into feature branch → conflict ⇒ `{ kind: 'conflict', files }` (abort, `main` untouched) → else fast-forward `main`, prune worktree, `clear()`. Returns `spec-only | source-included | conflict`.
- [ ] P2.12. Feature-context entry points — Build Studio "New feature", assistant `start_feature` tool, one-click quick-edit (pre-filled id).
- [ ] P2.13. Update spec-write tool → `vfs.writeText('Documents/Specs/…')`.
- [ ] P2.14. [T] Unit tests — no-context error; debounce coalescing (US1.3); wipe-survival (US4); promote conflict contract (US3.3).
- [ ] P2.15. `npx tsc --noEmit` + `npm run lint` green.

## Phase 3 — Spec Provider Registry + BOS_SPECS_ROOT migration

- [ ] P3.1. `src/lib/specs/provider.ts` (new) — `SpecProvider` + `SpecRegistry` (builtin → user → marketplace).
- [ ] P3.2. [P] `src/lib/specs/providers/builtin.ts` — scans `data/specs/system/`; `canWrite` derived from manifest `writable: false` (single source of truth).
- [ ] P3.3. [P] `src/lib/specs/providers/user.ts` — `data/specs/user/`, `canWrite: true`.
- [ ] P3.4. [P] `src/lib/specs/providers/marketplace.ts` — `data/specs/marketplace/<id>/`, `canWrite: false`.
- [ ] P3.5. `src/lib/specs/stores.ts` (rewrite) — delegate to `SpecRegistry`; preserve public API.
- [ ] P3.6. **Consumer enumeration (gate)** — grep every `BOS_SPECS_ROOT`/`specsRoot` consumer (`stores.ts`, `seed.ts`, `specs-dir.ts`, `pipeline.ts`, `skills/store.ts`, `tools/supervisor/supervisor.mjs`, …); migrate each to the fixed layout.
- [ ] P3.7. **Supervisor repoint** (`tools/supervisor/supervisor.mjs`) — per-preview/base spec-store paths and promote-merge logic point at `data/specs/…`. Repoint, not rewrite.
- [ ] P3.8. Delete `src/os/specs-dir.ts` and remove `BOS_SPECS_ROOT` — **only after** P3.6/P3.7 land.
- [ ] P3.9. `npx tsc --noEmit` + `npm run lint` green (BOS + bastion).

## Phase 4 — App Provider Registry (manifest vs. component split)

- [ ] P4.1. `src/lib/apps/provider.ts` (new) — `AppProvider` + `AppRegistry` (manifests only). Extend `AppManifest` (`runtime`, `source`, `marketplaceId`, `marketplaceItemId`, `canAdoptSpec`).
- [ ] P4.2. [P] `providers/builtin.ts` — wraps `_manifests.generated`; `runtime: 'native'`, `source: 'builtin'`.
- [ ] P4.3. [P] `providers/local.ts` — scans `data/apps/local/` `app-manifest.json`; `runtime: 'iframe'`.
- [ ] P4.4. [P] `providers/marketplace.ts` — parses `marketplace.json` from `data/apps/marketplace/<id>/`; `runtime: 'iframe'`.
- [ ] P4.5. `src/os/apps.ts` (modify) — manifest list from `AppRegistry`; SSR seed uses it server-side.
- [ ] P4.6. Window renderer — branch on `runtime`: `native` → `getAppComponent(id)` (static, **`registry.tsx` unchanged**); `iframe` → iframe with entrypoint. Enforce `native ⇒ builtin`.
- [ ] P4.7. `npx tsc --noEmit` + `npm run lint` green.

## Phase 5 — Marketplace (client + providers + adopt/install)

- [ ] P5.1. `src/lib/marketplace/schema.ts` (new) — `MarketplaceManifest`, `MarketplaceItem`, `RegisteredMarketplace`.
- [ ] P5.2. `src/lib/marketplace/client.ts` (new) — `addMarketplace` (**git-URL allowlist** `https://`/optional `ssh`; reject `file://`/`ext::`; `execFile` clone to canonical `data/…/marketplace/<id>/`; **schema-validate** before use), `syncMarketplace`, `listItems`, `adoptSpec`, `installApp`.
- [ ] P5.3. Lifecycle — `removeMarketplace` (unregister + delete clones; leaves adopted specs), `uninstallApp` (delist + delete local copy). Confirmations on destructive ops.
- [ ] P5.4. [T] Unit test — git-URL allowlist rejects `file://`/`ext::`/unknown; malformed `marketplace.json` rejected before clone.
- [ ] P5.5. `src/app/api/marketplace/route.ts` (new) — list / add / remove / sync / adopt-spec / install-app / uninstall-app.
- [ ] P5.6. `src/apps/marketplace/manifest.ts` + `index.tsx` (new) — three-panel UI: marketplaces + add; item grid (search/filter); detail with Run/Install/Adopt/Uninstall.
- [ ] P5.7. [T] e2e — register a local repo as marketplace; list items; adopt a spec; install/run an app.
- [ ] P5.8. `npx tsc --noEmit` + `npm run lint` green.

## Phase 6 — App sandboxing (opaque-origin) + iframe SDK library + Docker/Bastion

- [ ] P6.1. Opaque-origin sandbox — render untrusted (local + marketplace) apps with `sandbox` **without** `allow-same-origin`. Keep native/first-party trusted apps on the existing same-origin path. Confirm the broker's `e.source` identity check (`IframeApp.tsx:83`) holds for opaque frames.
- [ ] P6.2. Promote the iframe SDK to a TS library — new `src/lib/iframe-sdk/` source tree; `tools/build-sdk.mjs` bundles it to a served artifact (esbuild — verify bundler already a dep via `src/lib/apps/build.ts` before adding). Rewrite `src/app/__bos/sdk.js/route.ts` to read the built artifact instead of a hard-coded string.
- [ ] P6.3. SDK surface — `storage` namespace, ready-promise, capability introspection, error types, over the existing private `call()` transport.
- [ ] P6.4. `storage` capability (server) — broker methods `storage:get/set/remove/keys` backed by a per-app BOS-assigned namespace in the user's data volume; capability-gated (extend `CAP_FOR_METHOD` + `dispatch` in `IframeApp.tsx`).
- [ ] P6.5. `localStorage`/`sessionStorage` shim (SDK) — `Object.defineProperty(window, 'localStorage', …)`; hydrate namespace → in-memory `Map` at load; sync reads; async write-through; flush on `pagehide`/`visibilitychange`. `sessionStorage` pure in-memory. IndexedDB not shimmed.
- [ ] P6.6. Serving route — extend `src/app/apps/[...slug]/route.ts` (or a sibling) to serve `data/apps/local/` and `data/apps/marketplace/<id>/`, not only `appsDir()`.
- [ ] P6.7. [P] `bastion/src/provision.ts` + `docker.ts` — remove `BOS_SPECS_ROOT` injection; ensure `data/config/`. (No apps-origin proxy needed — opaque-origin removes it.)
- [ ] P6.8. [T] Tests — an opaque-origin app cannot reach BOS except via the broker; an ungranted `storage` call is rejected; the `localStorage` shim round-trips (hydrate → get, set → persist → reload → get).
- [ ] P6.9. BOS + bastion `tsc --noEmit` + `npm run lint` green.

## Phase 7 — Developer-agent feature-branch wiring (via Supervisor)

- [ ] P7.1. When a feature context is active and the Developer agent edits BOS source, route through the **Supervisor** for `bos/feat/<id>` (worktree + port pool) — never a raw checkout of the running tree.
- [ ] P7.2. After each source commit, PATCH `/api/feature-context` to append `touchedSourcePaths`.
- [ ] P7.3. `npx tsc --noEmit` + `npm run lint` green.

## Closeout

- [ ] C1. Update `overview.md`; mark 018 FR-005/FR-007 Superseded, 020 Adopted, 009 → three-source.
- [ ] C2. Update `docs/dev/architecture-overview.md`, `docs/dev/self-modification/live-version-control.md`, `docs/dev/repository-and-data-layout.md`.
- [ ] C3. Update `docs/usage/` — Feature Context workflow, Marketplace, writing specs from any app.
