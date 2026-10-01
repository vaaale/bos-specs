# Design: Project development — a registered repository is developed on its own terms

**Feature**: `053-project-development`
**Branch**: `prj/`-prefixed branches are what this feature introduces; its own work rides `bos/project-development`
**Spec**: `spec.md` (same directory)
**App Target**: `bos-core`

---

## 1. Classification

A **capability completion with a naming change at its centre**. `050` specified that BOS develops in the whole of a registered repository; only the spec half was built. Nothing here is a new subsystem — every piece is an existing mechanism that was never told a project exists.

The feature has one structural idea and four consequences:

| | What |
|---|---|
| **Structural** | A project feature branch is named `prj/*`, which BOS's own machinery is incapable of naming — so exclusion from preview restore, the toolbar, reclamation and the Supervisor guard is a consequence, not an exemption |
| **C1** | The Supervisor gains a *workspace* concern separate from its *preview* concern — including the **commit** and **merge+push** steps that today exist only inside the BOS build |
| **C2** | Branch→git-root resolution becomes injectable — in the VFS filesystem AND in Build Studio's separate, function-shaped spec-fs module, which resolves a branch mount a different way |
| **C3** | The branch list is derived from refs that already exist per repository, and the stale pointer that resurrects dead branches is cleared at source |
| **C4** | Credential resolution collapses to one implementation **per process** — BOS's is URL-keyed; the Supervisor has its own and cannot import BOS's |

> **The trap this design walked into once already.** "A project branch needs no
> build" is true. "A project branch needs nothing the build does" is false: the
> candidate's **commit** lives inside `buildAndStart`, not inside the harness
> (§2). Three of the corrections below are the same mistake in different
> clothes — a BOS-specific wrapper was removed along with a step that was never
> BOS-specific at all.

---

## 2. Context (container level)

```
 Build Studio (src/apps/build-studio/index.tsx)        Assistant chat
 ┌──────────────────────────────┐          ┌───────────────────────────┐
 │ tree: group rows per store   │          │ FeatureBranchSelector     │
 │  right-click → openStoreMenu │          │  (AgentSelector.tsx)      │
 │  withFeatureBranch(...)      │          │ ElicitationCards          │
 └──────────┬───────────────────┘          └────────────┬──────────────┘
            │  createAndActivateFeatureBranch (ONE pair) │
            └─────────────────┬──────────────────────────┘
                              ▼
            POST/GET /api/assistant/feature-branches
                              │  setBranchScope → data/system/branch-scopes.json
                              ▼
 ┌─────────────────────────────────────────────────────────────────────┐
 │ delegate-common.ts → claude-runner.ts                               │
 │   contentOnly ──► scratch dir                                       │
 │   else         ──► supervisorBegin(branch) ──► BOS worktree  ◄── the│
 │                                                 only arm today      │
 └─────────────────────────────────────────────────────────────────────┘
                              ▼  HTTP /__supervisor/*
 ┌─────────────────────────────────────────────────────────────────────┐
 │ Supervisor (separate process, no imports from src/)                 │
 │  control.mjs  → requireFeatureBranch (rejects anything not bos/)    │
 │  preview.mjs  → _provisionPreview: BOS branch, BOS worktree, data   │
 │                 clone, coupled mounts at <wt>/specs/<id>, port      │
 │  coupled-repos.mjs → listSpecStores(), coupledReposFor(),           │
 │                      mountCoupled(), promoteCoupled()               │
 │  worktree.mjs → addWorktreeForBranch(), reconcileFeatureBranches()  │
 └─────────────────────────────────────────────────────────────────────┘

 VFS:  /Specs/<id> → SpecFS(repoRoot, id, worktrees, writable, repoOffset)
       registerMount / unregisterMount (src/os/vfs.ts)
```

Key facts this design relies on (all confirmed in source):

- **`FEATURE_BRANCH_PREFIX = "bos/"`** (`config.mjs:51`) gates `isFeatureBranch` (`gitutil.mjs:58`), which gates `restorePreviews` (`preview.mjs:79`), `listBranches` → the toolbar, `reconcileFeatureBranches` (`worktree.mjs:754`), and `requireFeatureBranch` on every BOS control path (`pin`, and via `provisionPreview` on `begin`/`build`/`activate`/`stop`/`resume`/`discard`/`promote`).
- **`listSpecStores()` (supervisor) already returns the repo root** per store (`{ id, root, storeRoot, offset, kind, owner, writable }`), symlinks resolved (`coupled-repos.mjs:124`). The Supervisor needs no new discovery to find a project's repository.
- **`gitState(repoRoot)` in `/api/repositories` already enumerates `refs/heads/bos/` per repository** (`route.ts:92`) and the route already cross-references `getBranchScope` to flag stray branches. Per-repository branch enumeration exists.
- **`unregisterMount` does not fail closed.** `findMount` returns null and the path falls through to plain VFS (`src/os/vfs.ts:83,92`) — `050` FR-003c's decoy write.
- **Nothing clears `activeFeatureBranch`** on promote or discard; `listDeclaredFeatureBranches` (`conversations-server.ts:117`) scans every conversation file and merges every name it finds, forever.
- **`reconcileViaApi` already exists** (`reconcile-client.mjs`) — the Supervisor's sanctioned way to ask the Next.js process to do something it must not do itself.

And eight facts that **contradict an earlier draft of this design**. Each one is
a place where a mechanism looked reusable and is not; every §3 section below is
written against these rather than against the resemblance:

- **A candidate's COMMIT lives in the build, not the harness.** `buildAndStart`
  does `git add -A` + `git commit` in the worktree (`build.mjs:82-96`), and
  aborts the build if either fails. `claude-runner.ts:403`'s `stageAll(cwd)`
  only *stages*. Drop the build from a delegation and the work is never
  committed — §3.2/§3.7.
- **`addWorktreeForBranch` reuses a worktree only if it has `node_modules`.**
  `isHealthyWorktree` (`worktree.mjs:468`) ends `return !!(await fs.stat(path.join(wt, "node_modules")))`.
  An un-hydrated worktree is judged unhealthy and gets `worktree remove --force`
  + `fs.rm` (`:481-482`) on the next call. For a project that is data loss, not
  a cache miss — §3.2.
- **`promoteCoupled` is *most* of project-merge, not all of it.** It merges into
  `coupledMergeTarget`'s base, deletes the branch and calls `pushCoupledBase`
  (`coupled-repos.mjs:526-568`) — but (a) it returns `void` and reports through a
  `warnings[]` array, (b) it returns silently when the branch ref is absent
  (`:527`), (c) `coupledMergeTarget` (`:82`) returns **whatever the primary
  checkout has checked out**, falling back to `defaultBranchOf` only on a
  detached HEAD, and (d) `workContextFor` (`:419`) hardcodes the conflict
  agent's context to `user-apps`/`user-specs` — §3.3.
- **`runAutoPush` pushes only `autoPush: true` remotes** (`push.mjs:131`), and
  `registerRepository` records a project's origin with `autoPush: false`
  (`repositories.ts`, the `addRemoteConfig` call). As-is, a project merge pushes
  nothing and reports no warning — §3.3.
- **The Supervisor's credential path cannot refresh.** The merge-time push
  resolves through `resolveRemoteToken` (`secrets.mjs:163`), which reads the
  provider OAuth token but never refreshes it; the refresh-aware wrapper
  (`git-auth.mjs`) is wired only to `pullBaseBeforeNewBranch`. GitLab tokens live
  ~2 h — §3.10.
- **The two "spec-fs implementations" are not the same shape.**
  `src/os/fs/spec-fs.ts` is a `SpecFS` *class* (an `FSBackend`); `src/lib/dev/spec-fs.ts`
  is a *function module* whose `resolveInStore` (`:230`) calls `branchSpecsRoot`
  (`:105`) → `supervisorBeginOrThrow` → `POST /__supervisor/begin`, plus a
  store-less `branchSpecsRoot(branch)` for cross-store search. Build Studio reads
  through the second — §3.5.
- **`reconcileFeatureBranches` only ever sees BOS-repo refs** (every `git()` call
  in it uses the default cwd `REPO`). A legacy `bos/*` branch scoped to a project
  has its *commits* in the project repo, where no reclamation runs — §3.4.
- **`normalizeFeatureBranch` strips only `^bos/`** (`feature-branch.ts:34`), so
  `PATCH /api/assistant/feature-branches` silently rewrites `prj/playable-core`
  into `bos/prj-playable-core` (`route.ts:145`); `setConversationActiveFeatureBranch`
  rejects a `prj/` name outright (`conversations-server.ts:18`) and
  `listDeclaredFeatureBranches` filters it out (`:125`) — §3.1, R1.

---

## 3. Component design

### 3.1 The prefix, and where it is decided (FR-002, FR-002a)

`src/lib/agent/feature-branch.ts` becomes prefix-aware without gaining a second copy of the naming rule:

```ts
export const BRANCH_PREFIXES = { bos: "bos/", project: "prj/" } as const;
export type BranchPrefix = (typeof BRANCH_PREFIXES)[keyof typeof BRANCH_PREFIXES];

const SLUG = /^[a-z0-9]+(?:-[a-z0-9]+){0,3}$/;

export function parseFeatureBranch(b: string): { prefix: BranchPrefix; slug: string } | null;
export function isValidFeatureBranch(b: string, prefix?: BranchPrefix): boolean;
export function normalizeFeatureBranch(name: string, prefix: BranchPrefix = "bos/"): string | null;
export function suggestFeatureBranchName(task: string, prefix?: BranchPrefix): string;
```

**The prefix is derived from the scope, never passed alongside it.** `POST /api/assistant/feature-branches` already receives `{ name, scope, scopeId }` and already resolves a `BranchScope` (route.ts:47-94). It computes the prefix from the resolved scope:

```ts
const prefix = scope?.kind === "repository" ? BRANCH_PREFIXES.project : BRANCH_PREFIXES.bos;
const branch = normalizeFeatureBranch(name, prefix);
```

The route's two steps are in the **wrong order today** — `normalizeFeatureBranch(name)` runs at `route.ts:36`, before the scope block at `:47-94`. Normalisation moves below scope resolution; nothing else about the route's shape changes.

This is why `dev_branch_request` needs no change: it already declares the scope, and the scope is the only input the prefix has. A caller cannot produce a `prj/` branch scoped to `bos-core`, or the reverse, because there is no parameter with which to say it.

**The three callers that must learn the new prefix** — because the default-to-`bos/` story does *not* cover them, and one of them fails silently:

| Site | Today | Required |
|---|---|---|
| `route.ts:145` (`PATCH`) | `normalizeFeatureBranch(raw)` → `bos/prj-playable-core` | parse the prefix off the name and preserve it; reject what parses to neither |
| `conversations-server.ts:18` | `isValidFeatureBranch` throws on `prj/*` | accept any known prefix |
| `conversations-server.ts:125` | same predicate filters `prj/*` out of the declared list | accept any known prefix (FR-007b) |

`DELETE`'s `?name=bos/<kebab>` contract and `DELETABLE_BRANCH_RE` (`src/lib/system/git.ts:213`) stay `bos/`-only: deleting a project branch is `/workspace/discard`'s job (§3.3), in the project's own repository, and a second deletion path into a user's repo is precisely what this feature exists to prevent.

The Supervisor's `FEATURE_BRANCH_PREFIX` and `isFeatureBranch` are **unchanged**. That is the mechanism, not an oversight.

### 3.2 Workspace, separate from preview (FR-001, FR-001a, FR-002b/c/d)

New module `tools/supervisor/lib/workspace.mjs`. It owns one sentence: *a worktree of repository R, on branch B, at a deterministic path — provisioned, committed, merged or discarded.*

```js
// workspaces: Map<branch, { branch, repoId, root, worktree }>
export async function provisionWorkspace(branch, repoId)   // idempotent, de-duped like provisionPreview
export async function commitWorkspace(branch)              // add -A + commit in the worktree; the ONLY durability step
export async function mergeWorkspace(branch)               // commit, then merge into the project's default branch, push, tear down
export async function discardWorkspace(branch)             // remove worktree + branch
export function workspacePath(repoId, branch)              // WORKTREES/projects/<repoId>/<slug>
```

- The repo root comes from `listSpecStores().find(s => s.id === repoId)` — existing discovery.
- `workspacePath` encodes repo and slug separately, so `prj/x` in two projects cannot collide (FR-002c).
- **No data clone, no port, no proc, no build state** exist on a workspace record. They are absent from the type, not set to null — a field that can hold a port is a field something will eventually try to serve (FR-002d).

#### The worktree primitive, and the reuse predicate that cannot come with it

`addWorktree(repoRoot, wtPath, branch)` is extracted from `addWorktreeForBranch` — the `worktree prune`, the stale-registration clearing, the `worktree add` itself. What is **not** extracted is its reuse check. `isHealthyWorktree` (`worktree.mjs:468`) ends on `fs.stat(<wt>/node_modules)`, so a project worktree — which is never hydrated — is judged unhealthy on every call, and the next line is `worktree remove --force` + `fs.rm`. `/workspace/begin` is reached on every spec file op in the branch, so that is "destroy the agent's uncommitted work, repeatedly".

So the predicate is **parameterised, not inherited**:

```js
isHealthyWorktree(wt, branch, { requireNodeModules })   // true for the BOS preview path, false for a workspace
```

A workspace is healthy when it is a git worktree, on the right branch, at the branch tip. `hydrateWorktree` (node_modules linking) is not applied, and now nothing silently requires its output either.

> This is the one place where sharing a primitive would have been actively worse than copying it. The shared part — `prune`, teardown, `add` — is the part that has been got wrong before. The unshared part is the part that encodes "a BOS worktree needs node_modules", which is not a fact about worktrees.

#### Commit is a workspace concern, not a build concern (FR-003b)

Today a candidate's edits become commits inside `buildAndStart` (`build.mjs:82-96`), which also *aborts* when `git add` or `git commit` fails — because an uncommitted worktree reads downstream as "no changes". A project has no build, so `commitWorkspace` carries that step verbatim, with the same two failure modes treated the same way:

```js
export async function commitWorkspace(branch) {
  const w = workspaces.get(branch);                       // or re-resolve from disk
  await git(["add", "-A"], w.worktree);                   // throws → caller fails, never "nothing to commit"
  const dirty = await git(["status", "--porcelain"], w.worktree);
  if (!dirty) return { committed: false, head: await git(["rev-parse", "HEAD"], w.worktree) };
  await git([...GIT_IDENTITY, "commit", "-m", `project work (${branch})`], w.worktree);
  return { committed: true, head: await git(["rev-parse", "HEAD"], w.worktree) };
}
```

Called in exactly two places: at the end of a project delegation (§3.7, where `buildCandidateWithRetry` sits for BOS), and at the top of `mergeWorkspace`, before anything is torn down.

**Restart safety.** `reconcileWorktrees` safety-commits every dirty BOS worktree before removing it at boot (`worktree.mjs:594-615`) — a project worktree is registered in a *different* repo, so that pass never sees it. `reconcileWorkspaces()` is its twin: for each `WORKTREES/projects/<repoId>/<slug>` directory, resolve the repo, `commitWorkspace` if dirty, and reclaim only when the branch is gone from that repo. It runs at boot alongside the existing reconcilers, and it is what makes SC-006 true for the work as well as the ref.

`preview.mjs` is unchanged in behaviour. It keeps every BOS-candidate concern and continues to call `addWorktreeForBranch` + `provisionClone` + `coupledReposFor` + `allocPreviewPort` exactly as today. The two modules share `addWorktree` and nothing else.

> **Why not extend `provisionPreview`?** It would have to branch on scope at its top, leaving a function whose name promises a preview and whose first act is to decide whether to make one. That shape — one entry point silently meaning two things — is what produced the original defect.

### 3.3 The Supervisor control surface (FR-002b)

`requireFeatureBranch` continues to reject `prj/*` on `/pin` and — via `provisionPreview` — on `/begin`, `/build`, `/activate`, `/stop`, `/resume`, `/discard` and `/promote`. Project work gets its own routes in `control.mjs`:

| Route | Body | Returns |
|---|---|---|
| `POST /__supervisor/workspace/begin` | `{ branch, repoId }` | `{ ok, worktree }` |
| `POST /__supervisor/workspace/commit` | `{ branch }` | `{ ok, committed, head }` |
| `POST /__supervisor/workspace/merge` | `{ branch }` | `{ ok, merged, target, push: {…}, warnings[] }` |
| `POST /__supervisor/workspace/discard` | `{ branch }` | `{ ok, warnings[] }` |

Each validates with a `requireProjectBranch` that is the mirror image of `requireFeatureBranch` — `prj/` prefix, same slug rule — so neither guard can be loosened to admit the other's branches.

`listBranches()` (the toolbar's source) is untouched and therefore cannot return a `prj/*` branch (FR-007d). This requires no code and is asserted by a test rather than implemented.

#### What `/workspace/merge` actually does (FR-005, FR-005a, FR-005b)

`promoteCoupled` is reused for the **merge mechanics** — the working-tree merge, the plumbing merge for a busy checkout, the branch delete, the conflict escalation. It is not reused as the whole operation, because four of its properties are wrong here and three of them are silent:

| `promoteCoupled` as-is | What `mergeWorkspace` does instead |
|---|---|
| Returns `void`; outcomes land in `warnings[]` | Returns `{ merged, target, push }` — the route cannot otherwise tell "merged" from "there was nothing to merge" |
| Returns silently when the branch ref is absent (`:527`) | Absent ref is a **named refusal**, not an `ok: true` |
| `coupledMergeTarget` merges into whatever the primary checkout has checked out (`:82`) | Merge target is the project's **default** branch — `origin/HEAD`, else `main`/`master`, else the single local branch — i.e. `defaultBranchOf` is the rule, not the fallback. A project left on a topic branch must not absorb the feature |
| `pushCoupledBase` → `runAutoPush` skips any remote without `autoPush: true` (`push.mjs:131`), which is **every** 050-registered project (`autoPush: false` at registration) | Push is explicit: the project's configured remote is pushed on merge regardless of the auto-push preference, which governs *unattended* pushes and not this one |

So the push result is a three-state answer, carried to the caller and never collapsed:

```js
push: { status: "pushed", remote: "origin" }
    | { status: "none",   reason: "no remote is configured for this project" }
    | { status: "failed", remote: "origin", error: "<cause>" }
```

`status: "none"` and `status: "pushed"` being the same value is exactly FR-005b's failure, and it is what `runAutoPush`'s empty array means today.

Conflict escalation is kept, but not inherited blind: `workContextFor` (`coupled-repos.mjs:419`) hardcodes `repoKind` to `"user-apps"`/`"user-specs"` and labels the repo "`<id>` store". It gains a `project` case, so a conflict in a user's game repository is not handed to an agent told it is resolving a spec store.

`discardWorkspace` removes that project's worktree and branch and nothing else. It does **not** go near `discardPreview`, whose no-record path iterates *every* spec store unscoped and deletes the branch in each (`preview.mjs:243`) — out of scope here, noted in §7 (R7).

### 3.4 Reclamation, including the legacy case (FR-002e)

`reconcileFeatureBranches` gains one condition, not a prefix check: a branch whose recorded scope is `{ kind: "repository" }` is never reclaimed. `branchScope(dataDir, branch)` exists in `coupled-repos.mjs:252` but is **module-private** — it is exported for this.

**Be precise about which ref this protects.** Every `git()` call in `reconcileFeatureBranches` uses the default cwd `REPO`, so the pass only ever sees BOS-repo refs. For a legacy `bos/<x>` scoped to a project, the BOS-repo ref is an empty placeholder — the commits are in the *project* repo, created by `mountCoupled`, where no reclamation runs at all. So:

- The guard's real value is **listing stability** and not losing the scope record mid-session (a reclaimed branch also has its `previews` record dropped, `worktree.mjs:759`), not rescuing commits.
- The spec's §"Why the prefix is load-bearing" overstates this, and has been corrected: the `fpsoffice` branch was reclaimed, but what reclamation deleted was an empty BOS-side ref. The argument for the prefix stands on the other three call sites.
- FR-002e's second half — "MUST be listed under its project" — is satisfied by §3.9's per-repository enumeration reading the *project's* refs, not by this guard.

New `prj/*` branches are invisible to this enumeration anyway. Nothing is renamed.

### 3.5 Branch → git root becomes injectable (FR-004, C2)

Both filesystems compute `path.join(worktree, "specs", storeId)` (`src/os/fs/spec-fs.ts:152,199`; `src/lib/dev/spec-fs.ts:119`). That expression encodes *how the Supervisor lays out coupled mounts*, which is not a fact a filesystem should know. It is replaced by an injected resolver:

```ts
export interface BranchRootResolver {
  /** The git root a read/write on `branch` resolves to, or null when nothing is mounted yet. */
  rootFor(branch: string): Promise<{ gitRoot: string; mountError?: string } | null>;
}
```

Two implementations:

- `coupledMountResolver(storeId)` — `supervisorBeginOrThrow(branch)` then `<worktree>/specs/<storeId>`. Today's behaviour, moved.
- `projectWorkspaceResolver(repoId)` — `supervisorWorkspaceBegin(branch, repoId)` then the workspace root itself. Memoised per branch exactly as `supervisorBegin` is (`supervisor.ts:85`); that memo exists because `/begin` was being called 48 times per Build Studio tree render, and the workspace route is reached from the same fan-out.

**The two files are not two copies of one class.** `src/os/fs/spec-fs.ts` is the `SpecFS` *class* (an `FSBackend` behind the VFS mount table). `src/lib/dev/spec-fs.ts` is a *function module* — `resolveInStore` (`:230`) → `branchSpecsRoot(branch, storeId)` (`:105`) → `supervisorBeginOrThrow` — and it is what **Build Studio and `/api/specs` read through**. `AGENTS.md` §1.3 (L) requires their *routing policy* to stay interchangeable; it does not make them the same object, and treating them as such is how the first draft of this design left the feature's own entry point broken:

> Right-click a project → **New feature branch…** → the tree reloads → `resolveInStore` POSTs `/__supervisor/begin` with a `prj/*` branch → `requireFeatureBranch` refuses it → every read in Build Studio throws. The first thing US1 asks a user to do.

So both take the resolver, by different seams:

| | `src/os/fs/spec-fs.ts` | `src/lib/dev/spec-fs.ts` |
|---|---|---|
| Seam | constructor parameter | a `branchRootFor(branch, store)` function, replacing `branchSpecsRoot`'s body |
| Store-less case | n/a | `branchSpecsRoot(branch)` (cross-store search) resolves the BOS worktree's `specs/` dir **only for a `bos/*` branch**; on a project branch it returns that project's workspace root, so search narrows to the one repo the branch belongs to |
| Shared | the routing decision itself lives in one function both import | |

A test instantiates the class and calls the module function against the same branch and store, and asserts both land on the same absolute path — for a `bos/` branch *and* a `prj/` branch. That is the interchangeability the L rule is actually about.

#### `/Projects/<id>` is a ProjectFS that COMPOSES SpecFS, not a SpecFS with `repoOffset: ""`

The tempting line is `new SpecFS(repoRoot, id, worktrees, writable, "", projectWorkspaceResolver(id))`. It is wrong in two concrete ways, both because `repoOffset: ""` is what switches off behaviour written to protect a user's repository:

- **The startup sweep would commit the user's uncommitted source.** `maybeStartupSweep` (`spec-fs.ts:289-309`) walks `repoRoot` plus every worktree and runs `commitOnSave(root, "chore: recover uncommitted spec edits (startup sweep)")`, scoped by `repoOffset`. Its own comment says the scoping is what stops it "recovering the USER'S PROJECT's uncommitted source into a BOS chore commit". `ensureSystemMounts` fires `runStartupSweep()` on every mount at boot (`spec-mount.ts:142`). Mounting a project at its repo root with an empty offset removes the guard and keeps the caller.
- **`list()` hides every dotfile** (`spec-fs.ts:317`): at a repo root that is `.gitignore`, `.github/`, `.eslintrc`, `.env.example` — project source FR-004 requires to be reachable. The filter is right for a spec store and wrong for a code tree.

`ProjectFS` therefore holds a `SpecFS` and delegates the parts that are genuinely shared policy — the writable gate, the active-branch requirement, `prepareWrite`, atomic write, debounced `commitOnSave`, the self-worktree hand-off — while owning the two that are store policy:

```ts
class ProjectFS implements FSBackend {
  constructor(private readonly inner: SpecFS) {}
  list(rel)  { return this.inner.list(rel, { hideDotfiles: false, hideManifests: false }); }
  runStartupSweep() { /* no-op: a user's working tree is not BOS's to commit */ }
  // everything else delegates unchanged
}
```

Both carve-outs are expressed as options on `SpecFS` rather than as overrides, so there is still exactly one implementation of the gate and the commit (D4's reason for rejecting a parallel class stands) — what differs is the two pieces of *policy* that were never about the gate.

`repoOffset` is still passed through for *reads of the specs within the project* (§3.6), and is `""` for the mount root itself.

### 3.6 Mounts, and refusing the retired prefix (FR-004a, FR-004b, FR-004c)

`remountSpecStores()` partitions discovered stores:

```ts
for (const store of stores) {
  if (isProject(store)) {
    registerMount(`/Projects/${store.id}`, projectFs(store));
    registerMount(`/Specs/${store.id}`, new RetiredMountFS(`/Specs/${store.id}`, `/Projects/${store.id}`));
  } else {
    registerMount(`/Specs/${store.id}`, specFs(store));
  }
}
```

`RetiredMountFS` is a backend whose every method throws a `RetiredMountError` naming the replacement path. **It is registered, not unregistered** — unregistering is what makes the path fall through to plain VFS and write a silent decoy (`050` FR-003c). The mount is the refusal.

**The bookkeeping set has to grow with it.** `remountSpecStores` tracks the prefixes it owns in `specStoreMounts` and unregisters everything no longer wanted (`spec-mount.ts:112-120`) — that is how `deregisterRepository` drops a store's mount, and its comment says why: "A mount left behind is not inert — it keeps the path resolvable, which reads as 'still registered'." A project now contributes **two** prefixes, so both go in the set and both come out on deregistration. Otherwise forgetting a repository leaves `/Projects/<id>` live, which is the same bug in a new prefix.

**`repoOffset` may legitimately be `""` for a project.** `registerRepository` writes `spec-store.json` at `storeRoot`, which *is* `repoRoot` whenever no workflow was chosen or the method declares `storeRoot: "."`. Then the project's specs are at the repo root and FR-004a's `/Projects/<id>/<repoOffset>` is `/Projects/<id>` itself. Nothing breaks — `/Specs/<id>` still refuses, so there is still one path — but `storeContentPath` must return the mount unchanged rather than appending a trailing segment, and the agent-facing reference (FR-009) must say "the project's specs are at the offset the method declared, which may be the repository root".

`storeMountPath(store)` becomes the single resolver (FR-004c):

```ts
export function storeMountPath(store: SpecStore): string;      // "/Projects/fpsoffice" | "/Specs/user-specs"
export function storeContentPath(store: SpecStore): string;    // + repoOffset, where the SPECS are
```

The set of **code** sites composing `/Specs/${id}` by hand is small and closed — `spec-mount.ts:114,124` and `gitlab-feature-branch.ts:107` (which composes it for `user-specs` specifically and is unaffected). A grep in the task list covers it; no lint rule is proposed.

The larger set is **prose**: roughly two dozen files under `seed/` (skills, the spec-kit pack's agents and driver references) and `docs/dev/` tell an agent that specs live at `/Specs/<id>`. `storeMountPath` cannot reach those, and a stale instruction is how an agent arrives at a retired path in the first place. They are rewritten in the same change, and D5 is what makes a missed one a loud refusal instead of a decoy (R3).

### 3.7 Delegation (FR-003, FR-003a/b/c)

`claude-runner.ts` gains a third arm, selected from the scope the branch already carries:

```
contentOnly              → scratch dir under dataDir()          (unchanged)
scope.kind === repository → supervisorWorkspaceBegin(branch, repoId) → project worktree
otherwise                → supervisorBegin(branch)              (unchanged)
```

The project arm:
- cwd = the workspace root,
- a workspace note that states this is a standalone repository, that BOS's source is not reachable from it, and where the specs sit within it,
- **no** `buildCandidateWithRetry` — and in its place, `supervisorWorkspaceCommit(branch)`,
- a completion note reporting branch and commits only — reusing the pure-function shape of `candidateNote` (which exists precisely because the wrong variant of it was load-bearing in a production incident) as a sibling `projectNote`.

**The commit is not optional and it is not the build.** `buildCandidateWithRetry` is where a BOS delegation's edits actually become commits (`build.mjs:82-96` runs inside it); `stageAll(cwd)` at `claude-runner.ts:403` only stages. Removing the build from the project arm without putting `commitWorkspace` in its place means the agent's work is staged, uncommitted, and then destroyed by the first `worktree remove --force` that `mergeWorkspace` or a re-provision performs. So the arm reads:

```
run the harness
  ↓
supervisorWorkspaceCommit(branch)      // retried like buildCandidateWithRetry, for the same reason:
  ↓                                     // the Supervisor being briefly unreachable must not silently skip it
projectNote({ branch, committed, head, failed })
```

`projectNote` inherits `candidateNote`'s one hard rule — **a run that did not complete never gets confident wording** — and adds its own: it names commits and branch, and makes no claim about a build, a preview, a promote, or whether the project works (FR-003b, FR-009a). The unreachable-Supervisor case is explicit, because "your work may not be committed" is the single most important thing this note can say:

| condition | what it says |
|---|---|
| run failed / cancelled | the branch may be incomplete; any partial work was still committed; do not act on this message |
| commit unreachable after retries | **your changes may not be committed** — re-run before assuming the work is safe |
| committed | `<n>` commit(s) on `prj/<slug>` in `<project>`; review and merge when ready |
| nothing to commit | the run produced no changes in the repository |

`bosSourceTrigger`'s refusal text gains a third sentence naming the project path, so an agent steered off BOS source is not steered into `contentOnly` by elimination.

`dev_delegate`'s **tool description** is updated in the same change. It currently reads "edits BOS source on a feature branch" (`dev-delegate.ts:23`), which is the model's actual decision surface — FR-009's reference is read later, if at all, and a description that contradicts it is how an orchestrator improvises.

A test seam mirrors the existing `_setAgentLayerForTests` convention so the chosen cwd is assertable without a Supervisor.

### 3.8 Build Studio: starting work (FR-006, FR-006a/b/c)

`openStoreMenu` gains **New feature branch…** on a project group row. It calls the *same* `createAndActivateFeatureBranch` pair the two existing entry points use (FR-006c), with `{ scope: "repository", scopeId: group.name }` fixed — the row answered the scope, so no picker is shown (FR-006a). The prefix follows from the scope (§3.1). (The menu already opens for an `arbitrary` store: `projectsAre` returns `"folders"` for it, so the `projectUnit === "none"` early return at `index.tsx:1154` does not fire.)

#### The branch has to actually exist somewhere (FR-006, FR-007)

`POST /api/assistant/feature-branches` does **not** create a branch under the Supervisor. It calls `createFeatureBranch`, which throws `SUPERVISOR_OWNS_GIT`, and the route deliberately swallows that (`route.ts:99-107`) because for a BOS branch the Supervisor creates the ref later, at delegate time. A project has no such later: US1.2 says the branch is created *in that project's repository*, SC-001 asserts it, and §3.9 builds the branch list from refs — so a project branch that exists only as a name is in no repository and in no group.

For a `repository`-scoped branch the route therefore calls `/workspace/begin` directly:

```ts
if (scope?.kind === "repository") await supervisorWorkspaceBegin(branch, scope.repoId);
```

This is cheap and idempotent (a worktree `add`, no clone, no port, no build — FR-001), it is the same call the first delegation would make anyway, and it is what makes "I want to work on this project" a fact on disk rather than a note on one conversation. A failure is reported, not swallowed: unlike the BOS case there is no later step to make it true.

The declared-name mechanism (`listDeclaredFeatureBranches`) stays for BOS branches, where the pending window is real (§3.9, FR-007b).

`withFeatureBranch` at the point of a write is untouched.

The active project + branch is surfaced in the Build Studio header (FR-006b), reading the conversation's `activeFeatureBranch` and its recorded scope — no new state.

### 3.9 The branch list (FR-007 … FR-007d, C3)

**Grouping reuses the enumeration that already exists.** `gitState(repoRoot)` in `/api/repositories` already globs `refs/heads/bos/` per repository; it extends to both prefixes and is lifted into one shared helper so `/api/repositories` and the branch list cannot disagree about what exists.

`GET /api/assistant/feature-branches` returns groups:

```ts
{ groups: [ { repoId: "bos-src", label: "BrowserOS", branches: ["bos/x"] },
            { repoId: "fpsoffice", label: "FPSOffice", branches: ["prj/playable-core"] } ] }
```

The flat `featureBranches: string[]` is **removed**, not kept alongside — two shapes for one answer is how a consumer ends up reading the stale one. The full set, which is larger than "two consumers":

| Site | Change |
|---|---|
| `AgentSelector.tsx:64` | reads groups |
| `ElicitationCards.tsx:92` | reads groups |
| `route.ts:122` (`POST` response) | returns groups |
| `route.ts:186` (`DELETE` response) | returns groups |
| `e2e/031-self-healing.spec.ts:159` | its cleanup loop reads `featureBranches` from the GET |
| `e2e/build-studio-own-live-branch.spec.ts:55`, `e2e/037-project-layer.spec.ts:41` | both **stub** the route with `{ featureBranches: [...] }`; a stub left in the old shape makes the dropdown silently empty, which reads as a product bug |

Leaving the flat field on `POST`/`DELETE` while removing it from `GET` would be the two-shapes problem with extra steps.

**Truthfulness is fixed at the source, not in the reader** (FR-007a). On promote and discard, the Supervisor already calls `clearCoupledBranchScope`; it additionally notifies BOS over the existing `reconcileViaApi` channel, and BOS clears `activeFeatureBranch` on every conversation pointing at that branch — through `setConversationActiveFeatureBranch`, which is queued per conversation and therefore serialises against the agent loop's own saves. The Supervisor must **not** write the conversation files directly: it would be a second writer to a file with an owner.

`listDeclaredFeatureBranches` survives, unchanged in purpose (FR-007b): a BOS branch activated but not yet materialised still has no ref, and a second conversation must still be able to select it. With the pointer cleared at merge/discard it stops being a source of resurrection. Declared names join the BrowserOS group — which is correct **only because §3.8 makes a project branch a real ref the moment it is created**. Its predicate also has to stop filtering out non-`bos/` names (`conversations-server.ts:125`, §3.1): a declared project branch should be a contradiction, and if one ever appears it must be groupable by its recorded scope rather than silently dropped or silently filed under BrowserOS.

### 3.10 Credentials (FR-008, FR-008a/b, C4)

One helper replaces the hand-copied `buildAuth` functions:

```ts
// src/lib/gitops/auth.ts
export async function resolveAuthForRepo(repoPath: string, remoteName: string): Promise<GitAuth | null>;
```

It reads the remote's URL from the repository, then delegates to the existing, correct `resolveAuthForUrl` — which matches by host and handles provider-wide OAuth. The inline `authType` + `token` override is retained for the genuine case (a credential BOS does not hold) but is no longer the only path that works.

**There are four copies, not two**, and all four carry the same defect — `resolveAuth(remoteName, at)` with no `provider`, which makes the OAuth arm return `null` unconditionally (`auth.ts:53`):

| | |
|---|---|
| `tools/server/git-push.ts:14` | the observed failure — the agent pasted a PAT into a tool call |
| `tools/server/git-mounts.ts:23` | |
| `tools/server/git-merge.ts:20` | |
| `tools/server/git-fetch.ts:15` | |

All four are rewritten. `src/lib/gitops/auto-push.ts:38` already passes the provider and is left alone; it is the one that was right.

#### The policy cannot exist once across the process boundary — say so

FR-008b's "exist once" is achievable within BOS and not beyond it. The push that FR-005 and SC-010 actually exercise runs **in the Supervisor**, through `push.mjs` → `resolveRemoteToken` (`secrets.mjs:163`), which is a deliberate standalone re-implementation (the Supervisor cannot import `src/lib`). So:

- `resolveAuthForRepo` is the one implementation **in BOS**, for every agent-facing git tool.
- The Supervisor keeps its own, and that copy has a real gap this feature now depends on: `resolveRemoteToken` reads the provider OAuth token but **never refreshes it**, while the refresh-aware wrapper (`git-auth.mjs`'s `fetchOriginWithAuth`) is wired only to `pullBaseBeforeNewBranch`. A GitLab token lives ~2 h, so an unrefreshed merge-time push fails intermittently in exactly the deployment SC-010 describes.
- **Resolution:** `mergeWorkspace`'s push goes through the refresh-aware path — `git-auth.mjs` gains a `pushWithAuth` sibling to `fetchOriginWithAuth`, reusing the same `refreshOAuthToken` + retry-once policy that already exists there. This is not a new mechanism; it is the existing one reaching a second call site.

The duplication between BOS's and the Supervisor's credential readers is pre-existing, load-bearing and documented in both files. It is named here so that "one implementation" is not read as a promise this design does not keep.

---

## 4. File / module plan

**New**

| Path | What |
|---|---|
| `tools/supervisor/lib/workspace.mjs` | project workspace provision / **commit** / merge / discard / **reconcile** |
| `src/lib/specs/project-fs.ts` | `ProjectFS`, `projectWorkspaceResolver`, `RetiredMountFS` |
| `seed/skills/bos-domain/references/target-project.md` | the fourth shape |
| `tests/supervisor/project-workspace.test.mjs` | FS-level assertions (§6) |
| `tests/supervisor/project-merge-push.test.mjs` | merge target, the three push states, absent-ref refusal |
| `tests/specs/project-mount.test.ts`, `tests/specs/branch-prefix.test.ts`, `tests/gitops/repo-auth.test.ts` | |
| `tests/specs/spec-fs-routing-parity.test.ts` | the class and the dev module resolve identically, `bos/` and `prj/` |

**Modified**

| Path | Change |
|---|---|
| `src/lib/agent/feature-branch.ts` | prefix-aware helpers |
| `src/app/api/assistant/feature-branches/route.ts` | normalise AFTER scope; prefix from scope; `/workspace/begin` for a repository scope; grouped GET/POST/DELETE; prefix-preserving PATCH |
| `src/lib/agent/conversations-server.ts` | accept + record a `prj/*` active branch; stop filtering it out of the declared list |
| `src/lib/agent/subagents/claude-runner.ts` | third arm, workspace note, **workspace commit**, `projectNote`, refusal text |
| `src/lib/assistant/tools/server/dev-delegate.ts` | tool description: no longer "edits BOS source" |
| `src/os/fs/spec-fs.ts` | injected `BranchRootResolver`; `list`/startup-sweep behaviour as options |
| `src/lib/dev/spec-fs.ts` | the same routing decision, through its own function seam (Build Studio's read path) |
| `src/lib/specs/spec-mount.ts` | `/Projects` mounts + retired-prefix refusal + **both prefixes in the owned-mount set** |
| `src/lib/specs/stores.ts` | `storeMountPath` / `storeContentPath` (handles `repoOffset === ""`) |
| `src/apps/build-studio/index.tsx` | New feature branch… ; active project/branch header |
| `src/components/apps/assistant/AgentSelector.tsx`, `ElicitationCards.tsx` | grouped list |
| `src/app/api/repositories/route.ts` | shared ref enumeration, both prefixes |
| `src/lib/gitops/auth.ts` + `tools/server/git-push.ts`, `git-mounts.ts`, `git-merge.ts`, `git-fetch.ts` | `resolveAuthForRepo` (all four copies) |
| `tools/supervisor/lib/control.mjs` | `/workspace/*` routes + `requireProjectBranch` |
| `tools/supervisor/lib/worktree.mjs` | extract `addWorktree`; parameterise `isHealthyWorktree`; scope check in reclamation |
| `tools/supervisor/lib/coupled-repos.mjs` | export `branchScope`; `project` case in `workContextFor` |
| `tools/supervisor/lib/git-auth.mjs` | `pushWithAuth` (refresh-aware push, §3.10) |
| `tools/supervisor/lib/reconcile-client.mjs` | notify-on-branch-gone |
| `e2e/031-self-healing.spec.ts`, `e2e/build-studio-own-live-branch.spec.ts`, `e2e/037-project-layer.spec.ts` | grouped branch-list shape |
| `tests/specs/test-branch-naming.test.ts` | the fixture rule covers `prj/` too (R8) |
| `seed/skills/bos-domain/SKILL.md` + ~24 `seed/`/`docs/dev/` pages naming `/Specs/<id>` | four shapes; `/Projects` |
| `AGENTS.md`, `docs/dev/**` | `prj/`, `/Projects`, project workflow |

---

## 5. Integration points (existing mechanisms this merely calls into)

- `promoteCoupled` / `coupledMergeTarget` / `pushCoupledBase` — project merge is these, unchanged.
- `listSpecStores()` (supervisor) — project repo root resolution.
- `createAndActivateFeatureBranch` — the one create-and-record pair; a third caller, not a third mechanism.
- `setBranchScope` / `getBranchScope` — already the discriminator.
- `reconcileViaApi` — the sanctioned Supervisor→BOS callback.
- `registerMount` / `unregisterMount` — the mount table.
- `resolveAuthForUrl` — already correct; now the only implementation.

---

## 6. Decisions (ADRs)

**D1 — The prefix is derived from the scope, not supplied beside it.**
*Why:* the scope is already declared, already validated, and already decides coupling. A separate `prefix` argument makes `{scope: "bos-core", prefix: "prj/"}` expressible, and anything expressible eventually arrives. *Rejected:* an explicit prefix parameter on `dev_branch_request` — it would also mean changing a tool description that is currently the only correct part of the stack.

**D2 — `prj/` excludes by inability, not by allow-list.**
*Why:* `isFeatureBranch` cannot match it, so four separate call sites skip project branches with no code. The `fpsoffice` branch was *correctly scoped* and still reclaimed twice, because scope was consulted in one place and not the other. *Rejected:* an `isProjectBranch()` exemption at each site — four places that must all be remembered, which is the failure being fixed.

**D3 — A new `workspace.mjs`, sharing the worktree primitive with `preview.mjs`.**
*Why:* "isolated checkout" and "running BOS candidate" are different responsibilities with different lifecycles. *Rejected:* branching inside `provisionPreview` (§3.2); and a fully generic "repo worktree manager" absorbing preview — a refactor far larger than the defect, touching ports, procs and pin cookies for no gain.

**D4 — Branch→git-root resolution is injected into both filesystems, by different seams; `ProjectFS` composes `SpecFS` rather than reimplementing or reconfiguring it.**
*Why:* `path.join(worktree, "specs", storeId)` is knowledge of the Supervisor's coupled-mount layout living in a filesystem. Injecting it is what lets one gate serve both. But the two files are not two copies of one class — `src/lib/dev/spec-fs.ts` is a function module and is Build Studio's read path, so "the same constructor parameter" does not reach it (§3.5). And `repoOffset: ""` is not a neutral configuration: it switches off the startup-sweep scoping and leaves the dotfile filter, so a plain `SpecFS` at a repo root would commit a user's uncommitted source and hide `.gitignore`.
*Rejected:* a separate `ProjectFS` **class reimplementing the gate** — a second implementation of the branch gate and commit-on-save, i.e. the divergence that is already called a security hole when it happens between the existing pair. Composition keeps one gate and overrides only the two pieces that were store policy all along.
*Also rejected:* `new SpecFS(..., repoOffset: "")` as the whole answer — the first draft of this design, and the source of both defects above.

**D4a — The commit step belongs to the workspace, not to the build.**
*Why:* it only *looks* like a build concern because `buildAndStart` is where it happens to live (`build.mjs:82-96`). Nothing about `git add -A && git commit` in a worktree is BOS-specific, and a project needs it for the same reason BOS does: a `worktree remove --force` is coming, and an uncommitted worktree reads downstream as "no changes". *Consequence:* `/workspace/commit` is a route, not an implementation detail of merge — the delegation calls it while the work is fresh, and `mergeWorkspace` calls it again before touching anything. *Rejected:* committing only at merge time — a run that is never merged loses everything, and that is most runs.

**D5 — The retired `/Specs/<project-id>` is a registered refusing mount.**
*Why:* `unregisterMount` does not fail closed; the path falls through and writes a decoy that reads back perfectly (`050` FR-003c). Only a registered backend can refuse. *Rejected:* leaving it mounted as an alias — two paths to one byte, explicitly rejected in the spec.

**D6 — The stale branch list is fixed by clearing the pointer at merge/discard.**
*Why:* the list is a symptom; the cause is a conversation that still claims a branch which no longer exists. Filtering at read time leaves every other reader of `activeFeatureBranch` wrong. *Rejected:* dropping `listDeclaredFeatureBranches` — it exists for the pending-BOS-branch window, which is real.

**D7 — BOS clears the pointer, not the Supervisor.**
*Why:* conversation files have an owner — the per-conversation queue in `conversation-store.ts` that serialises against the agent loop's saves. A second process writing them directly races it. `reconcileViaApi` already exists for exactly this shape. *Rejected:* direct file writes from the Supervisor.

**D8 — Project merge reuses `promoteCoupled`'s merge MECHANICS, and owns its target, its push and its result.**
*Why:* the working-tree merge, the plumbing merge for a busy checkout, the branch delete and the conflict escalation are real, hard-won, and worth reusing. Four other properties of that function are wrong here and three fail silently: it returns `void` through a `warnings[]` array, it returns silently on an absent ref (`:527`), `coupledMergeTarget` merges into **whatever the primary checkout has checked out** rather than the default branch (`:82`), and `runAutoPush` pushes nothing for a project because registration records `autoPush: false` (§2, §3.3).
*Consequence:* conflict escalation does come along, but not for free — `workContextFor` must learn a `project` case or it hands a user's game repo to an agent told it is resolving a spec store.
*Rejected:* "promote is this call and nothing else" (the first draft) — it produces a merge that targets the wrong branch, pushes nothing, and answers `ok: true` to all three.
*Rejected:* a project-specific merge written from scratch — the plumbing-merge path alone is worth more than the divergence risk.

**D8a — Merge pushes regardless of the `autoPush` preference.**
*Why:* `autoPush` governs *unattended* pushes on BOS's own promote. A user choosing **Merge** on a project branch has asked for the work to be delivered, and `050` FR-001a recorded a credential precisely so BOS could deliver it. *Consequence:* "no remote configured" becomes a distinct, reported outcome rather than an empty array that reads as success (FR-005b).

**D9 — The grouped branch list reuses `/api/repositories`' per-repo enumeration.**
*Why:* that route already globs `refs/heads/bos/` per repository and already cross-references branch scope. A second enumeration would be a second answer to "which branches exist". *Consequence:* the glob and the shared helper move to one place both routes import.

**D10 — The flat `featureBranches: string[]` is removed, not deprecated.**
*Why:* two shapes for one answer, with one of them stale, is the precise shape of this feature's own bug. *Correction to the first draft:* there are six sites, not two — GET's two consumers, the POST and DELETE responses, a self-heal e2e cleanup loop, and two e2e route **stubs** that return the old shape (§3.9). A stub left behind makes the dropdown silently empty, which reads as a product bug and not as a stale test.

**D11 — A project branch is created in the project's repository at creation time, not at first delegation.**
*Why:* the BOS lazy-creation story exists because the Supervisor owns BOS's git and the ref appears when the worktree does. A project's ref costs a `worktree add` with no clone, no port and no build, and FR-007 derives the branch list from refs — so lazily-created means invisible, in the list *and* on the Repositories page. *Rejected:* widening `listDeclaredFeatureBranches` to carry project names and group them by recorded scope — that is a second source of truth about which project branches exist, which is the thing `050` deliberately does not have.

---

## 7. Risks & open questions

**R1 — `prj/` splits the branch vocabulary, and "default to `bos/`" does not make every caller safe.** `FEATURE_BRANCH_RE` is referenced by self-heal (`bos/self-heal-<id>`), by `gitlab-feature-branch.ts:182`, and asserted by `tests/specs/test-branch-naming.test.ts`. The first draft claimed a `bos/`-defaulted parameter leaves "every existing caller unchanged by construction" — that is true of those three and **false of the three that matter**, one of which fails silently (§3.1): `PATCH` rewrites `prj/x` into `bos/prj-x`, `setConversationActiveFeatureBranch` throws on it, and `listDeclaredFeatureBranches` filters it out. A prefix-aware default is the mechanism for the callers that keep meaning `bos/`; it is not a substitute for changing the three that must now mean both. → *Mitigation:* those three are named in the file plan and each gets a test asserting a `prj/` name survives the round trip byte-for-byte; the naming test gains a case asserting `prj/` is rejected where `bos/` is required and vice versa.

**R2 — A `prj/*` branch reaching a BOS control path fails with a validation error rather than a diagnosis.** → *Mitigation:* `requireFeatureBranch`'s message names the project route when the branch carries the project prefix. The guard stays strict; only the message gets smarter.

**R3 — The `/Specs`→`/Projects` cutover is a scattered edit, and a missed site writes a decoy.** → *Mitigation:* D5 turns every missed site into a loud refusal instead of a silent write, which converts the risk from data loss into a visible error. The `storeMountPath` rewrite is then cleanup, not correctness.

**R4 — The two spec-fs modules drift while gaining a resolver.** Sharper than first stated: they are not the same shape (§3.5), so "the same constructor parameter in both" is not available and the symmetry a reviewer assumes is not there. → *Mitigation:* the routing *decision* lands in one function both import; `tests/specs/spec-fs-routing-parity.test.ts` drives the class and the module function against the same branch + store and asserts one absolute path, for `bos/` and `prj/` alike. The parity test is the sync requirement made executable, which is what `AGENTS.md` §1.3 (L) actually asks for.

**R5 — Clearing `activeFeatureBranch` on merge could clear a branch a conversation is mid-run on.** → *Closed, in favour of refusing.* `/workspace/merge` refuses while a run is active on that branch and says so; `runManager()` already knows which runs are live, so this costs one lookup. The first draft deferred it as "a change to BOS promote semantics" — it is not: BOS's promote is untouched, and this is a new route deciding its own precondition. Merging a branch out from under a running delegation produces exactly the half-delivered state FR-005b exists to prevent, and a refusal a user can retry is strictly better than a pointer silently cleared.

**R6 — A project whose checkout is dirty when a workspace is provisioned.** → *Mitigation:* worktrees do not touch the primary checkout, so this is safe for `add`. It matters at merge, where the working-tree merge would carry the user's unrelated dirt into the merge commit — so `mergeWorkspace` uses the plumbing path (already written, for the detached-HEAD case) whenever the primary checkout is dirty, not only when it is busy. BOS is a contributor to a repository it does not own (`050` FR-003b); committing a user's unstaged edits as a side effect of a merge is the same class of violation as §3.5's startup sweep.

**R7 — `discardPreview`'s unscoped fallback still reaches a user's repositories.** With no preview record it iterates **every** spec store and deletes the branch in each (`preview.mjs:243-244`) — the pre-scoping shape that `coupledReposFor` was written to end, surviving in one path. Out of scope for this feature (it is a `bos/*`-only path and does not block anything here), but it is a live instance of the exact defect class, and it belongs in the backlog with this design as its citation.

**R8 — A leaked `prj/` test fixture lands in a user's own repository.** `tests/specs/test-branch-naming.test.ts` scans for `bos/` literals only; the production box once carried 18 fixture-named `bos/*` branches. The same leak under `prj/` is worse, because the branch is created in a repository that is not BOS's and that no BOS cleanup path reaps. → *Mitigation:* the naming rule covers both prefixes; `tests/_no-live-deployment.cjs` already blocks the channel (it clears `BOS_SUPERVISOR_URL` and the `/__supervisor/` routes the new ones live under), and that remains the first line of defence.

**R9 — Nothing reaps an orphaned project worktree directory.** `reconcileWorktreeDirs` and `reconcileDataClones` both scan `<root>/bos` only (`worktree.mjs:647,797`), and `workspaces` is in-memory. → *Mitigation:* `reconcileWorkspaces()` (§3.2), which safety-commits before reclaiming and reclaims only when the branch is gone from that repo — the same conservative rule its two siblings use.

**Open question — where the project's label comes from in the branch group header.** → *Closed:* `SpecStore.label`. `/api/repositories` already carries it, Build Studio already groups by it, and `PROJECT_MANIFEST` (`project.json`) is `033`'s marker for a Project *folder inside* a store — a registered repository has none, so there was no alternative to weigh.
