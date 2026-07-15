# Implementation Plan: VFS Mount Points, SpecFS, Feature Context, Provider Registry, and Marketplace

**Branch**: `027-vfs-specfs-marketplace` | **Date**: 2026-07-15 (revised post-review) | **Spec**: [spec.md](./spec.md) | **Review**: [spec-review.md](./spec-review.md) | **Tasks**: [tasks.md](./tasks.md)

## Summary

Give the VFS a **mount table** routing path prefixes to pluggable `FSBackend` implementations. `Documents/Specs/` mounts to **SpecFS** — an *adapter* over the existing `src/lib/dev/spec-fs.ts` + Supervisor worktree engine (020), **not** a new git engine. SpecFS enforces a **Feature Context** on every write, resolves the active `bos/feat/<id>` branch, and routes writes to a worktree (Supervisor-provisioned, or self-provisioned for spec-only features). Reads are ref-pinned. Commits are debounced with bounded-diff LLM messages. Promotion force-flushes, reconciles `main` into the branch (conflicts first-class), and prunes the worktree. **User specs** live in `data/specs/user/` (VFS-writable, wipe-safe); **system specs** are read-only, edited as source via the Developer agent. Specs and apps adopt a **Provider Registry** pattern; the app side splits an async *manifest* registry from the static *native-component* map. A **Marketplace** delivers apps and adoptable spec templates from git repos; untrusted apps run in an **opaque-origin sandbox** (`sandbox` without `allow-same-origin`) so the `postMessage` capability broker is the *only* channel to BOS — no DNS/cert/port infrastructure. Because opaque origins have no browser storage, the **iframe SDK is promoted to a TypeScript library** (`src/lib/iframe-sdk/`, built to a served artifact) that adds a broker-backed `storage` capability and a `localStorage`/`sessionStorage` shim, so unmodified open-web apps still work. `BOS_SPECS_ROOT` is removed *last*, gated on migrating every consumer.

## Technical Context

**Language/Version**: TypeScript, Node ≥ 20. Next.js App Router. Zustand vanilla store.

**Primary Dependencies**: git via `execFile` (no shell). Reuse existing `store-git.ts` helpers (`readFileAtBranch`, `commitOnSave`, `DRAFT_BRANCH` regex) and `src/lib/dev/spec-fs.ts`. No new runtime deps anticipated.

**Storage** (fixed under `dataDir()`; no env var):
```
data/specs/user/                 ← user-specs canonical git repo (SpecFS backing)
data/specs/user-worktrees/<br>/  ← self-provisioned worktrees for spec-only features
data/specs/system/               ← seeded from seed/spec-store/, READ-ONLY
data/specs/marketplace/<id>/     ← cloned marketplace repos, read-only
data/apps/local/                 ← user-developed local apps (scaffold)
data/apps/marketplace/<id>/      ← canonical clone location for marketplace apps
data/config/feature-context.json ← server-owned active Feature Context
data/config/marketplaces.json    ← registered marketplace URLs
```
`data/vfs/Documents/Specs/` is a mount-point stub so it appears in listings; reads/writes route to SpecFS.

**Server boundary**: git ops, SpecFS, feature-context I/O, LLM commit-message calls are server-only. The Zustand store holds a read-only mirror of the active context; all mutations go through the server-authoritative `feature-context.ts` module.

**Testing**: unit tests (not just tsc/lint) for the security- and correctness-critical boundaries — mount path-escape, no-context error, debounce coalescing, wipe-survival, `id` sanitization, git-URL allowlist. A new e2e spec for the Marketplace app. `npx tsc --noEmit` + `npm run lint` per phase.

**Migration**: one-time in `seed.ts` — if `data/specs/user/` is absent but a legacy `BOS_SPECS_ROOT/user-specs` exists, copy + log. System specs re-seed into `data/specs/system/`. `BOS_SPECS_ROOT` removal is the final step, gated on the consumer enumeration (Phase 3).

## Constitution Check

- **I. Spec-Driven — SAAP**: plan derives from `spec.md`; tasks in `tasks.md`. PASS.
- **II. Server Authority & SSR Boundary**: all FS/git/context/LLM work is server-only; the client holds a read-only context mirror; untrusted apps run in an opaque-origin sandbox with no ambient access to BOS — every call is broker-mediated and capability-gated. PASS.
- **III. Always Delegate; Claude Codes**: system-spec edits and all source work run via the Developer sub-agent on the feature branch — this is now the *only* system-spec path (Option B), strengthening the principle. PASS.
- **IV. Minimize Blast Radius**: SpecFS *adopts* the existing worktree engine (no parallel git model); unmounted VFS paths are unchanged; `stores.ts` public API preserved; env-var removal gated on full consumer migration. PASS.
- **V. The VFS Is Not the Source**: `Documents/Specs/` routes to `data/specs/user/`, never `src/`. System specs (source) are edited only via the Developer agent. PASS.
- **VI. Specs & Docs Stay in Sync**: closeout updates `overview.md`, `docs/dev/self-modification/live-version-control.md`, and `docs/dev/repository-and-data-layout.md` *as part of* the Supervisor repoint, not after. PASS.
- **VII. Respect Boundaries**: `package.json`/lockfiles untouched (git via `execFile`); destructive marketplace ops require confirmation. PASS.

No violations.

## Phase 1 — VFS Mount Points + server-authoritative Feature Context

Critical path.

- **Mount table** (`src/os/fs-types.ts`, `src/os/vfs.ts`): `FSBackend` interface mirroring the current VFS surface; `MountPoint`; `registerMount`/`resolveMount`. All nine VFS functions check the mount first, else fall through to `LocalFS` (extracted, no behaviour change). **Unit test the path-escape jail on `resolveMount`.**
- **Feature Context module** (`src/lib/specs/feature-context.ts`, server-only): single writer over `data/config/feature-context.json` with an in-process async mutex and atomic RMW. API: `getActive`, `setActive(id)`, `clear`, `patch(fn)`. `setActive` is the one activation verb — it flushes any currently-active feature first, then ensures `id`'s branch/worktree (create-or-reuse, idempotent), then records `id` active. "Start new" vs "resume existing" is a **UI distinction only** (both call `setActive`), not separate module methods. **`id` sanitized against `^[a-z0-9-]+$` in `setActive`.**
- **Types** (`src/os/types.ts`): `FeatureContext`, `FeatureContextFile`.
- **OS store** (`src/store/os-store.ts`): `activeFeature` read-only mirror + intent actions that call the API and **await** the response before dependent writes; cross-tab sync (storage event / refresh-on-focus).
- **API** (`src/app/api/feature-context/route.ts`): `GET`/`POST`(set)/`DELETE`(clear)/`PATCH`(append touched path) — each delegates to the module (no whole-file replace from the client).

## Phase 2 — SpecFS adapter + worktree writes + Promotion

Promotion lives here (not last) because it is P1 (US3) and depends only on the branch model settled in this phase.

- **SpecFS** (`src/os/fs/spec-fs.ts`): an `FSBackend` **adapter** over `src/lib/dev/spec-fs.ts`. No `git checkout`.
  - Resolve active branch from the feature-context module; refuse writes with `SpecFSNoContextError` when none.
  - **Reads ref-pinned** via `readFileAtBranch` (active branch, else `main`).
  - **Writes → worktree**: Supervisor worktree when a preview exists; else self-provision `data/specs/user-worktrees/<branch>/` via `git worktree add`.
  - **Debounced commit** (2 s) via `commitOnSave`; message from `generateCommitMessage(diff)` with the **diff bounded** (truncate + cap file count) and deterministic fallback.
  - `patch(touchedSpecs)` through the feature-context module (shared mutex).
- **editFile stays a spec-layer op** (not on `FSBackend`), rewired to route through the active branch.
- **flushPending(branch)**: cancel debounce, synchronous `stageAll → commit`. Precondition for any committed-state read.
- **Startup sweep**: on init / first access, `hasUncommitted` on the canonical repo + active worktrees → recovery commit.
- **Mount registration** (explicit ordering): ensure `data/specs/user/` exists (seed) **before** `registerMount('/Documents/Specs', new SpecFS(...))`.
- **Promotion** (`src/lib/specs/promote.ts`): `flushPending` → reconcile `main` into the feature branch in its worktree → on conflict `git merge --abort` and return `{ kind: 'conflict', files }` → else fast-forward `main`, prune worktree, `clear()` context. Returns `spec-only | source-included | conflict`.
- **Feature-context entry points**: Build Studio "New feature" action, assistant `start_feature` tool, one-click quick-edit (pre-filled id). Behaviour-change note documented: no-branch commit-on-save is gone.
- **spec-write tool** → `vfs.writeText('Documents/Specs/…')`.
- **Tests**: no-context error; debounce coalescing (US1.3); wipe-survival (US4); promote conflict contract (US3.3).

## Phase 3 — Spec Provider Registry + BOS_SPECS_ROOT migration

- **SpecProvider / SpecRegistry** (`src/lib/specs/provider.ts`): aggregate builtin → user → marketplace.
- **BuiltinSpecProvider** — `data/specs/system/`, `canWrite: false` (**single source of truth**: provider `canWrite` derives from the store manifest `writable`; system manifest set to `writable: false`).
- **UserSpecProvider** — `data/specs/user/`, `canWrite: true`.
- **MarketplaceSpecProvider** — `data/specs/marketplace/<id>/`, `canWrite: false`.
- **`stores.ts` rewrite** — delegate to the registry; preserve public API.
- **Consumer enumeration (gate)**: grep every `BOS_SPECS_ROOT` / `specsRoot` consumer — at least `stores.ts`, `seed.ts`, `specs-dir.ts`, `pipeline.ts`, `skills/store.ts`, and `tools/supervisor/supervisor.mjs` — and migrate each to the fixed layout. **Supervisor repoint**: point its per-preview/base spec-store paths and promote-merge logic at `data/specs/…` (a repoint, not a rewrite).
- **Remove `src/os/specs-dir.ts` and `BOS_SPECS_ROOT`** — *last*, only after every consumer is migrated.

## Phase 4 — App Provider Registry (manifest vs. component split)

- **AppProvider / AppRegistry** (`src/lib/apps/provider.ts`): async, provider-aggregated **manifests only**. Extend `AppManifest` with `runtime`, `source`, `marketplaceId`, `marketplaceItemId`, `canAdoptSpec`.
- **`registry.tsx` stays static** — native builtin components from `_components.generated`, unchanged. The window renderer branches on `runtime`: `native` → `getAppComponent(id)`; `iframe` → iframe with the manifest entrypoint. Invariant enforced: `native` only for `builtin`.
- **BuiltinAppProvider / LocalAppProvider / MarketplaceAppProvider** — manifests for `data/apps/local/` and `data/apps/marketplace/<id>/`.
- **`src/os/apps.ts`** — drive the manifest list from `AppRegistry`; SSR seed uses it server-side.

## Phase 5 — Marketplace (client + providers + adopt/install)

- **Schemas** (`src/lib/marketplace/schema.ts`): `MarketplaceManifest`, `MarketplaceItem`, `RegisteredMarketplace`. **Schema-validate before use.**
- **MarketplaceClient** (`src/lib/marketplace/client.ts`): `addMarketplace` (**git-URL allowlist**: `https://` only, optional `ssh`; reject `file://`/`ext::`; `execFile` clone to the single canonical location `data/…/marketplace/<id>/`), `syncMarketplace`, `listItems`, `adoptSpec` (fork into user-specs + commit), `installApp`.
- **Lifecycle contracts**: `removeMarketplace` (unregister + delete clones; **does not touch adopted specs**), `uninstallApp` (delist + delete local copy). Adoption is a fork; un-adopt = delete a user spec store (spec management, not marketplace). All destructive ops confirmed.
- **API** (`src/app/api/marketplace/route.ts`): list / add / remove / sync / adopt-spec / install-app / uninstall-app.
- **Marketplace app** (`src/apps/marketplace/`): three-panel UI (marketplaces + add; item grid; detail with Run/Install/Adopt/Uninstall). **New e2e spec.**

## Phase 6 — App sandboxing (opaque-origin) + iframe SDK library + Docker/Bastion

The trust boundary is the **opaque-origin sandbox**, not a separate origin — no DNS, certs, or ports.

- **Opaque-origin sandbox**: untrusted apps (local + marketplace) render with `sandbox` **without** `allow-same-origin`, so each frame is a unique throwaway origin walled off from BOS and from every other frame. The `postMessage` broker is the only channel; it already authenticates by `e.source === iframe.contentWindow` (`IframeApp.tsx:83`), which is correct for opaque frames (whose `e.origin` is `"null"`). Native builtin + first-party trusted apps may keep the existing same-origin path.
- **iframe SDK as a TypeScript library** (`src/lib/iframe-sdk/`): promote the hard-coded string in `src/app/__bos/sdk.js/route.ts` to a real TS source tree, bundled to a served artifact by a build step (`tools/build-sdk.mjs` via esbuild — verify the bundler is already a dep from `src/lib/apps/build.ts` before adding). The route reads the built artifact. The SDK gains: a `storage` namespace, a ready-promise, capability introspection, and error types — over the same private `call()` transport.
- **`storage` capability + shim**: a broker method `storage:get/set/remove/keys` backed by a **per-app, BOS-assigned namespace** in the user's data volume (capability-gated). The SDK installs a `localStorage`/`sessionStorage` shim (hydrate-then-write-through: hydrate the namespace into an in-memory `Map` at load, sync reads, async write-through, flush on `pagehide`/`visibilitychange`; `sessionStorage` is pure in-memory). IndexedDB is not shimmed. Per-app namespacing gives app-to-app isolation *with* persistence despite opaque origins.
- **Wire the `[...slug]` route** (or a sibling) to serve `data/apps/local/` and `data/apps/marketplace/<id>/`, not only `appsDir()`.
- **Bastion**: remove `BOS_SPECS_ROOT` injection (`provision.ts`, `docker.ts`); ensure `data/config/` exists. **No apps-origin proxy/hostname wiring needed** — opaque-origin removes that infrastructure.
- **Escape hatch (not built)**: if hosting unmodified open-web apps that require a *stable browser origin* (service workers, native IndexedDB at scale) ever becomes necessary, a wildcard-subdomain apps origin can be added later. Documented, not implemented.

## Phase 7 — Developer-agent feature-branch wiring (via Supervisor)

- When a feature context is active and the Developer agent edits BOS source, it works through the **Supervisor** for `bos/feat/<id>` (worktree + port pool) — **never a raw checkout of the running tree** (fragile-main hazard). After each commit it PATCHes `touchedSourcePaths`.

## #8 hygiene (folded into the phases above)

- LLM commit diff bounded (Phase 2).
- `writable`/`canWrite` single source of truth (Phase 3).
- Docs (`live-version-control.md`, `repository-and-data-layout.md`) updated with the Supervisor repoint (Phase 3 / closeout).

## Closeout

- Update `overview.md`; mark 018 FR-005/FR-007 Superseded and 020 Adopted; update 009 to three-source.
- Update `docs/dev/architecture-overview.md`, `docs/dev/self-modification/live-version-control.md`, `docs/dev/repository-and-data-layout.md`, and `docs/usage/` (Feature Context workflow, Marketplace, writing specs from any app).
