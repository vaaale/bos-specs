# Implementation Plan: VFS Mount Points, SpecFS, Feature Context, Provider Registry, and Marketplace

**Branch**: `027-vfs-specfs-marketplace` | **Date**: 2026-07-15 | **Spec**: [spec.md](./spec.md) | **Tasks**: [tasks.md](./tasks.md)

## Summary

Replace the siloed, scan-based spec store architecture with a uniform VFS-backed system. The VFS gains a **mount table** that routes path prefixes to pluggable `FSBackend` implementations. `Documents/Specs/` is mounted to `SpecFS` — a `GitFS` subclass that enforces a **Feature Context** on every write, manages git branches automatically, and generates LLM commit messages via debounced flush. Both specs and apps adopt a **Provider Registry** pattern (Builtin / User-or-Local / Marketplace) replacing the compiled-in app list and the `BOS_SPECS_ROOT` directory scan. A **Marketplace** built-in app lets users register remote git repos that deliver pre-built apps and adoptable spec templates. `BOS_SPECS_ROOT` is removed; all paths are fixed under `dataDir()`.

## Technical Context

**Language/Version**: TypeScript, Node ≥ 20. Next.js App Router. Zustand vanilla store.

**Primary Dependencies**:
- `simple-git` (already likely present; verify before adding) — git operations in GitFS/SpecFS. If absent, use `child_process.execFile` with `git` directly.
- No new runtime deps anticipated for Phases 1–4. Marketplace cloning reuses git.

**Storage**:
```
data/specs/system/            ← seeded at startup from seed/spec-store/, read-only
data/specs/user/              ← user-specs git repo (SpecFS backing store)
data/specs/marketplace/<id>/  ← cloned marketplace repos, read-only
data/apps/local/              ← user-developed local apps (Phase 4, scaffold only)
data/apps/marketplace/<id>/   ← cloned for offline use (Phase 5)
data/config/feature-context.json  ← active Feature Context (server-readable)
data/config/marketplaces.json     ← registered marketplace URLs
```

`data/vfs/Documents/Specs/` is a real directory (mount point stub) so it appears in directory listings. Reads and writes through it are intercepted by the mount table and routed to `data/specs/user/`.

**Server boundary**: All git operations, SpecFS logic, and feature context persistence run server-side behind API routes. The Zustand OS store holds a client-side mirror of the active Feature Context; changes are persisted via `POST /api/feature-context`.

**Testing**: `npx tsc --noEmit` and `npm run lint` after each phase. Manual verification per User Story acceptance scenarios. No new e2e spec required for Phase 1–3 (covered by existing VFS e2e infrastructure). Phase 5 (Marketplace app) warrants a new e2e spec.

**Migration**: Existing user-specs under `BOS_SPECS_ROOT` are migrated to `data/specs/user/` on first boot in `seed.ts`. One-time migration: if `data/specs/user/` is absent but `$BOS_SPECS_ROOT/user-specs` exists, copy + log. System specs are re-seeded from `seed/spec-store/` into `data/specs/system/`.

## Constitution Check

- **I. Spec-Driven — SAAP**: this plan derives from `spec.md`; implementation follows `tasks.md`. PASS.
- **II. Server Authority & SSR Boundary**: all git operations, SpecFS, and feature context I/O are server-only (behind API routes or within `src/os/fs/`). The LLM commit-message call goes through the existing AI provider client (server-side). No secrets or git internals are exposed to the client. PASS.
- **III. Always Delegate; Claude Codes**: implementation runs via the Developer sub-agent on the feature branch. PASS.
- **IV. Minimize Blast Radius**: changes are additive — new files (`src/os/fs/`, `src/lib/specs/providers/`, `src/lib/apps/providers/`, `src/lib/marketplace/`, `src/apps/marketplace/`). Existing VFS callers are unaffected (unmounted paths fall through to LocalFS, identical behaviour). `stores.ts` public API is preserved during migration. `BOS_SPECS_ROOT` removal is the only breaking change, and it is gated on a migration path. PASS.
- **V. The VFS Is Not the Source**: `Documents/Specs/` is user-spec content, not BOS source. The mount point routes to `data/specs/user/`, not `src/`. BOS source is modified only by the Developer agent on the correlated feature branch. PASS.
- **VI. Specs & Docs Stay in Sync**: closeout merges 027 into the relevant specs (018, 020, 009), updates `discrepancies.md` and `overview.md`, and updates `docs/dev/architecture-overview.md` + `docs/usage/` as needed. PASS.
- **VII. Respect Boundaries**: `package.json` / lockfiles are untouched unless `simple-git` is confirmed absent. `BOS_SPECS_ROOT` removal is explicit and intentional. PASS.

No violations. No complexity exceptions needed.

## Project Structure

```
specs/bos-system-specs/027-vfs-specfs-marketplace/
├── spec.md       ← done
├── plan.md       ← this file
└── tasks.md      ← done

src/os/
├── fs-types.ts                 ← NEW: FSBackend, MountPoint
├── fs/
│   ├── local-fs.ts             ← NEW: LocalFS implements FSBackend
│   ├── git-fs.ts               ← NEW: GitFS abstract class
│   └── spec-fs.ts              ← NEW: SpecFS extends GitFS
└── vfs.ts                      ← MODIFY: add mount table

src/os/types.ts                 ← MODIFY: FeatureContext type + AppManifest extensions

src/store/os-store.ts           ← MODIFY: featureContext slice

src/app/api/
├── feature-context/route.ts    ← NEW
└── marketplace/route.ts        ← NEW

src/lib/specs/
├── provider.ts                 ← NEW: SpecProvider interface + SpecRegistry
├── providers/
│   ├── builtin.ts              ← NEW
│   ├── user.ts                 ← NEW
│   └── marketplace.ts          ← NEW
├── promote.ts                  ← NEW: promoteFeature()
└── stores.ts                   ← REWRITE (same public API)

src/lib/apps/
├── provider.ts                 ← NEW: AppProvider interface + AppRegistry
└── providers/
    ├── builtin.ts              ← NEW
    ├── local.ts                ← NEW
    └── marketplace.ts          ← NEW

src/lib/marketplace/
├── schema.ts                   ← NEW: marketplace.json + marketplaces.json types
└── client.ts                   ← NEW: MarketplaceClient (clone, sync, adopt, install)

src/apps/marketplace/
├── manifest.ts                 ← NEW
└── index.tsx                   ← NEW

src/os/apps.ts                  ← MODIFY: use AppRegistry
src/components/apps/registry.tsx ← MODIFY: use AppRegistry
src/os/specs-dir.ts             ← DELETE
bastion/src/provision.ts        ← MODIFY: remove BOS_SPECS_ROOT, create data/specs/user/
bastion/src/docker.ts           ← MODIFY: remove BOS_SPECS_ROOT env injection
```

## Phase 1 — VFS Mount Points + Feature Context (Foundation)

Critical path. All subsequent phases depend on this.

### T001 — `src/os/fs-types.ts` (new)
Define `FSBackend` interface mirroring the existing VFS API surface:
```typescript
interface FSBackend {
  list(path: string): Promise<VfsEntry[]>
  stat(path: string): Promise<VfsEntry>
  readText(path: string): Promise<string>
  readBuffer(path: string): Promise<Buffer>
  writeText(path: string, content: string): Promise<void>
  writeBuffer(path: string, data: Buffer): Promise<void>
  mkdir(path: string): Promise<void>
  remove(path: string): Promise<void>
  rename(from: string, to: string): Promise<void>
  exists(path: string): Promise<boolean>
}
interface MountPoint { vfsPrefix: string; backend: FSBackend }
```
`path` passed to a backend is relative to the mount root.

### T002 — `src/os/fs/local-fs.ts` (new)
`LocalFS implements FSBackend`. Mechanical extraction of the current `vfs.ts` path-resolution + `fs/promises` calls. No logic change — this is a refactor target.

### T003 — `src/os/vfs.ts` (modify)
Add mount table:
```typescript
const mounts: MountPoint[] = []
export function registerMount(prefix: string, backend: FSBackend): void
function resolveMount(vfsPath: string): { backend: FSBackend; rel: string } | null
```
Update all nine public VFS functions to call `resolveMount()` first; if matched, delegate to the backend with the relative path; otherwise fall through to existing LocalFS behaviour. Unmounted paths must behave identically to today.

### T004 — `src/os/types.ts` (modify)
Add:
```typescript
interface FeatureContext {
  id: string
  branchName: string        // always "bos/feat/<id>"
  description?: string
  touchedSpecs: string[]    // VFS paths of spec folders written to
  touchedSourcePaths: string[]
  startedAt: string
}
interface FeatureContextFile { active: FeatureContext | null }
```

### T005 — `src/store/os-store.ts` (modify)
Add `activeFeature: FeatureContext | null` to `OSState`. Add actions:
- `setFeature(ctx: FeatureContext)` — sets active, persists via API
- `clearFeature()` — clears active, persists via API
- `touchSpec(vfsPath: string)` — appends to `touchedSpecs` if not present, persists
- `touchSource(filePath: string)` — appends to `touchedSourcePaths` if not present, persists

All persist actions call `fetch('/api/feature-context', { method: 'POST', … })` fire-and-forget.

### T006 — `src/app/api/feature-context/route.ts` (new)
- `GET` → read and return `data/config/feature-context.json`
- `POST` → validate body as `FeatureContextFile`, write to file
- `DELETE` → write `{ active: null }`

### T007 — Ensure `Documents/Specs/` stub exists
In the VFS initialisation that creates the default directory structure (where `Documents`, `Pictures`, `Desktop` are created), add `Documents/Specs` as a standard directory. This makes it visible in the file browser even before SpecFS is mounted.

---

## Phase 2 — GitFS + SpecFS

### T008 — `src/os/fs/git-fs.ts` (new)
`abstract class GitFS implements FSBackend` with `repoPath: string` constructor parameter.

Git operations (shell out via `execFile('git', [...], { cwd: repoPath })`):
- `ensureBranch(name)` — `git checkout <name>` || `git checkout -b <name>`
- `currentBranch()` — `git rev-parse --abbrev-ref HEAD`
- `getDiff()` — `git diff HEAD`
- `stagedDiff()` — `git diff --cached`
- `stageAll()` — `git add -A`
- `commit(message)` — `git commit -m <message> --allow-empty-message`
- `merge(from, to)` — checkout `to`, merge `from` (fast-forward preferred)
- `hasUncommitted()` — `git status --porcelain`

FSBackend methods: `fs/promises` operations rooted at `repoPath`. Identical to LocalFS but scoped to the repo directory.

### T009 — `src/os/fs/spec-fs.ts` (new)
`class SpecFS extends GitFS`

Key behaviour:
1. `writeText` / `writeBuffer`: read active feature context from `data/config/feature-context.json`. Throw `SpecFSNoContextError` if absent. Call `ensureBranch(ctx.branchName)`. Write file. Append spec folder to `touchedSpecs` in the context file (direct file write, not via the client store). Schedule debounced commit.
2. Debounce queue: `Map<branchName, { timer, changedPaths[] }>`. Each write resets the 2 s timer. On flush: `stageAll()` → `getDiff()` → `generateCommitMessage(diff)` → `commit(message)`.
3. `generateCommitMessage(diff)`: calls the configured AI provider (same client used by the assistant). Prompt: `"Write a concise git commit message (imperative mood, ≤72 chars) for this diff:\n\n<diff>"`. On any error returns `"Update specs"`.
4. `read*`, `list`, `stat`, `remove`, `rename`, `mkdir`: delegate to GitFS base (no branch enforcement on reads).

### T010 — Seed `data/specs/user/` (modify `src/lib/specs/seed.ts`)
On first run:
- If `data/specs/user/` is absent: init a new git repo there with `git init && git commit --allow-empty -m "init"`. Create a minimal `spec-store.json` (`{ "label": "My Specs", "owner": "user", "writable": true, "requiresPromote": true }`).
- One-time migration: if `BOS_SPECS_ROOT` is set and `$BOS_SPECS_ROOT/user-specs` exists, `cp -r` it to `data/specs/user/` and log the migration.
- Redirect system spec seeding from `BOS_SPECS_ROOT` to `data/specs/system/`.

### T011 — Register SpecFS mount at server startup
In the server initialisation path (called from `src/app/page.tsx` or equivalent server entry):
```typescript
registerMount('/Documents/Specs', new SpecFS(path.join(dataDir(), 'specs/user')))
```

### T012 — Update spec-write tool
`tools/server/spec-write.ts` (exact location TBD by Developer): replace direct `fs.writeFile` / `specsRoot()` calls with `vfs.writeText('Documents/Specs/<store>/<file>', content)`. The tool no longer handles git or branches.

---

## Phase 3 — Spec Provider Registry

### T013 — `src/lib/specs/provider.ts` (new)
```typescript
interface SpecProvider {
  readonly id: string
  readonly type: 'builtin' | 'user' | 'marketplace'
  readonly canWrite: boolean
  listStores(): Promise<SpecStore[]>
  getStore(id: string): Promise<SpecStore | undefined>
}
class SpecRegistry {
  register(provider: SpecProvider): void
  listAll(): Promise<SpecStore[]>        // builtin → user → marketplace
  getStore(id: string): Promise<SpecStore | undefined>
  defaultWritable(): Promise<SpecStore | undefined>
}
```

### T014 — `src/lib/specs/providers/builtin.ts` (new)
`BuiltinSpecProvider`: scans `data/specs/system/` for directories containing `spec-store.json`. `canWrite: false`. Same discovery logic as current `stores.ts` but rooted at the fixed system path.

### T015 — `src/lib/specs/providers/user.ts` (new)
`UserSpecProvider`: reads `data/specs/user/spec-store.json`. Returns a single store. `canWrite: true`.

### T016 — `src/lib/specs/providers/marketplace.ts` (new)
`MarketplaceSpecProvider`: reads `data/config/marketplaces.json`, scans each `data/specs/marketplace/<id>/` for spec-store subdirectories. `canWrite: false`.

### T017 — `src/lib/specs/stores.ts` (rewrite)
Replace directory scan with calls to `SpecRegistry.listAll()`. Public API (`listStores`, `getStore`, `defaultWritableStore`) is preserved — all callers continue to work without changes.

### T018 — Delete `src/os/specs-dir.ts`
Remove file. Update the two callers (`stores.ts` rewrite in T017 eliminates the only references).

---

## Phase 4 — App Provider Registry

### T019 — `src/lib/apps/provider.ts` + type extensions (new)
`AppProvider` interface and `AppRegistry` class (same pattern as SpecRegistry). Extend `AppManifest` in `src/os/types.ts`:
```typescript
runtime: 'native' | 'iframe'       // 'native' = compiled React component; builtin only
source: 'builtin' | 'local' | 'marketplace'
marketplaceId?: string
marketplaceItemId?: string
canAdoptSpec?: boolean
```

### T020 — `src/lib/apps/providers/builtin.ts` (new)
`BuiltinAppProvider`: wraps `_manifests.generated` import. Marks all as `runtime: 'native'`, `source: 'builtin'`.

### T021 — `src/lib/apps/providers/local.ts` (new)
`LocalAppProvider`: scans `data/apps/local/` for `app-manifest.json` files. Returns apps with `runtime: 'iframe'`, `source: 'local'`. Creates `data/apps/local/` if absent.

### T022 — `src/lib/apps/providers/marketplace.ts` (new)
`MarketplaceAppProvider`: reads cloned marketplace repos from `data/apps/marketplace/<id>/`, parses `marketplace.json`, exposes items with an `app` entry. `runtime: 'iframe'`.

### T023 — `src/os/apps.ts` + `src/components/apps/registry.tsx` (modify)
Replace `BUILTIN_APPS` import with `AppRegistry.listAll()`. `getApp(id)` delegates to `AppRegistry.resolveApp(id)`. SSR seed in `src/app/page.tsx` no longer appends runtime apps manually.

---

## Phase 5 — Marketplace

### T024 — `src/lib/marketplace/schema.ts` (new)
Types: `MarketplaceManifest`, `MarketplaceItem`, `RegisteredMarketplace`.

```typescript
interface MarketplaceManifest {
  id: string; name: string; version: string; description?: string
  items: MarketplaceItem[]
}
interface MarketplaceItem {
  id: string; name: string; description: string; tags?: string[]
  app?: { entrypoint: string; runtime: 'iframe'; version: string; icon?: string }
  spec?: { path: string; version: string }
}
interface RegisteredMarketplace {
  id: string; url: string; name: string; addedAt: string; lastSynced: string | null
}
```

### T025 — `src/lib/marketplace/client.ts` (new)
`MarketplaceClient`:
- `addMarketplace(url)` — clone to `data/specs/marketplace/<id>/` + `data/apps/marketplace/<id>/` (same repo, two symlinks or one clone), append to `marketplaces.json`, update `lastSynced`.
- `syncMarketplace(id)` — `git pull` in clone, update `lastSynced`.
- `listItems(id)` — parse `marketplace.json` from clone.
- `adoptSpec(marketplaceId, itemId)` — copy `items/<itemId>/spec/` into `data/specs/user/<itemId>-adopted/`, write `spec-store.json`, git-commit ("Adopt <name> from <marketplace>").
- `installApp(marketplaceId, itemId)` — verify `app` entry exists; `LocalAppProvider` discovery will pick it up from `data/apps/marketplace/<id>/items/<itemId>/app/`.

### T026 — `src/app/api/marketplace/route.ts` (new)
- `GET /api/marketplace` — list registered marketplaces + their items
- `POST /api/marketplace` — add by URL
- `DELETE /api/marketplace/:id` — remove (delete from `marketplaces.json`, optionally purge clone)
- `POST /api/marketplace/:id/sync`
- `POST /api/marketplace/:id/items/:itemId/adopt-spec`
- `POST /api/marketplace/:id/items/:itemId/install-app`

### T027 — `src/apps/marketplace/` (new built-in app)
`manifest.ts`: `id: "marketplace"`, `name: "Marketplace"`, `icon: "Store"`, `singleton: true`, `builtin: true`.  
`index.tsx`: Three-panel layout:
- **Left**: registered marketplaces list + "Add marketplace" (URL input + Add button).
- **Centre**: item grid with search input and tag filter chips. Each card shows name, description, tags, and action buttons appropriate to item capabilities.
- **Right**: item detail — description, version, [Run App] / [Install App] / [Adopt Spec] with confirmation where destructive.

---

## Phase 6 — Docker / Bastion

### T028 — `bastion/src/provision.ts` (modify)
Remove `BOS_SPECS_ROOT` env var injection. On first-run provisioning per user, ensure `data/config/` exists. `data/specs/user/` is created and git-initialised by BOS itself (`seed.ts`), not the bastion — no bastion change needed for that path.

### T029 — `bastion/src/docker.ts` (modify)
Remove `BOS_SPECS_ROOT` from the env array passed to container create. The per-user `data/` bind mount already covers `data/specs/` — no separate volume is needed.

### T030 — Remove `BOS_SPECS_ROOT` documentation
Update `.env.example` (if present), `docs/dev/deployment.md`, and any other references. Add a migration note in `docs/dev/architecture-overview.md` describing the new fixed paths.

---

## Phase 7 — Promotion + Developer Agent Wiring

### T031 — `src/lib/specs/promote.ts` (new)
```typescript
async function promoteFeature(ctx: FeatureContext): Promise<PromoteResult>
```
- Merge `bos/feat/<id>` → `main` in user-specs via SpecFS git operations.
- If `ctx.touchedSourcePaths.length === 0`: return `{ kind: 'spec-only' }`.
- Else: return `{ kind: 'source-included', branchName: ctx.branchName }`.
- After successful merge: write `{ active: null }` to `data/config/feature-context.json` and call `clearFeature()` via the feature-context API.

### T032 — Build Studio promote UI (modify)
Before showing the "Promote" confirmation dialog:
- Read `activeFeature.touchedSourcePaths` from OS store.
- Display: "Spec-only — no rebuild required" (fast path) or "Source files modified — branch `<name>` will remain open for PR review" (full path).
- Both paths available as a single "Promote" button; the dialog communicates consequences.

### T033 — Developer agent: feature branch wiring
When the Developer agent begins modifying BOS source files and a feature context is active:
1. Read `data/config/feature-context.json`.
2. If `active` is set: `git checkout -b <branchName>` in the BOS source repo (no-op if already on that branch).
3. After each file commit: `PATCH /api/feature-context` with the modified source file path to append to `touchedSourcePaths`.

---

## Closeout Checklist

- [ ] Merge 027 spec into `018-external-spec-store` (update FR-005/FR-007 status to Superseded)
- [ ] Merge 027 spec into `020-branch-coupled-specs` (mark Fulfilled)
- [ ] Merge 027 spec into `009-installed-apps` (update installed-apps model to three-source)
- [ ] Update `discrepancies.md` — remove any BOS_SPECS_ROOT drift entries
- [ ] Update `overview.md` — add 027 summary
- [ ] Update `docs/dev/architecture-overview.md` — new VFS, SpecFS, Feature Context, Provider Registry sections
- [ ] Update `docs/usage/` — end-user guide for Feature Context, Marketplace, and spec writing
