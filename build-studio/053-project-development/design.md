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
| **C1** | The Supervisor gains a *workspace* concern separate from its *preview* concern |
| **C2** | Branch→git-root resolution becomes injectable, because both spec-fs implementations hardcode the BOS coupled-mount layout |
| **C3** | The branch list is derived from refs that already exist per repository, and the stale pointer that resurrects dead branches is cleared at source |
| **C4** | Credential resolution collapses to one URL-keyed implementation |

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

- **`FEATURE_BRANCH_PREFIX = "bos/"`** (`config.mjs:51`) gates `isFeatureBranch` (`gitutil.mjs:58`), which gates `restorePreviews` (`preview.mjs:79`), `listBranches` → the toolbar, `reconcileFeatureBranches` (`worktree.mjs:753`), and `requireFeatureBranch` on every BOS control path.
- **`promoteCoupled` already is project-promote.** It merges `branch` into the repo's own `coupledMergeTarget` base, deletes the branch, and calls `pushCoupledBase` → `runAutoPush(repo.root, base, repo.id)`. For a project, promote is this call and nothing else.
- **`listSpecStores()` (supervisor) already returns the repo root** per store (`{ id, root, storeRoot, offset, kind, owner, writable }`), symlinks resolved. The Supervisor needs no new discovery to find a project's repository.
- **`gitState(repoRoot)` in `/api/repositories` already enumerates `refs/heads/bos/` per repository** (`route.ts:92`) and the route already cross-references `getBranchScope` to flag stray branches. Per-repository branch enumeration exists.
- **Both spec-fs implementations hardcode the coupled-mount layout**: `src/os/fs/spec-fs.ts:152` and `src/lib/dev/spec-fs.ts:119` both compute `path.join(worktree, "specs", storeId)`. Neither can address a worktree that *is* the repo.
- **`unregisterMount` does not fail closed.** `findMount` returns null and the path falls through to plain VFS — `050` FR-003c's decoy write.
- **Nothing clears `activeFeatureBranch`** on promote or discard; `listDeclaredFeatureBranches` (`conversations-server.ts:117`) scans every conversation file and merges every name it finds, forever.
- **`reconcileViaApi` already exists** (`reconcile-client.mjs`) — the Supervisor's sanctioned way to ask the Next.js process to do something it must not do itself.

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

**The prefix is derived from the scope, never passed alongside it.** `POST /api/assistant/feature-branches` already receives `{ name, scope, scopeId }` and already resolves a `BranchScope` *before* creating the branch (route.ts:47-94). It computes the prefix from the resolved scope:

```ts
const prefix = scope?.kind === "repository" ? BRANCH_PREFIXES.project : BRANCH_PREFIXES.bos;
const branch = normalizeFeatureBranch(name, prefix);
```

This is why `dev_branch_request` needs no change: it already declares the scope, and the scope is the only input the prefix has. A caller cannot produce a `prj/` branch scoped to `bos-core`, or the reverse, because there is no parameter with which to say it.

The Supervisor's `FEATURE_BRANCH_PREFIX` and `isFeatureBranch` are **unchanged**. That is the mechanism, not an oversight.

### 3.2 Workspace, separate from preview (FR-001, FR-001a, FR-002b/c/d)

New module `tools/supervisor/lib/workspace.mjs`. It owns one sentence: *a worktree of repository R, on branch B, at a deterministic path.*

```js
// workspaces: Map<branch, { branch, repoId, root, worktree }>
export async function provisionWorkspace(branch, repoId)   // idempotent, de-duped like provisionPreview
export async function discardWorkspace(branch)             // remove worktree + branch
export async function mergeWorkspace(branch)               // promoteCoupled against the project repo
export function workspacePath(repoId, branch)              // WORKTREES/projects/<repoId>/<slug>
```

- The repo root comes from `listSpecStores().find(s => s.id === repoId)` — existing discovery.
- The worktree is created with the **same** `addWorktree` primitive the preview path uses, extracted from `addWorktreeForBranch` so the stale-worktree teardown and `worktree prune` handling is not re-derived. `hydrateWorktree` (a BOS-specific step: node_modules linking) is **not** applied.
- `workspacePath` encodes repo and slug separately, so `prj/x` in two projects cannot collide (FR-002c).
- **No data clone, no port, no proc, no build state** exist on a workspace record. They are absent from the type, not set to null — a field that can hold a port is a field something will eventually try to serve (FR-002d).

`preview.mjs` is unchanged in behaviour. It keeps every BOS-candidate concern and continues to call `addWorktreeForBranch` + `provisionClone` + `coupledReposFor` + `allocPreviewPort` exactly as today. The two modules share the worktree primitive and nothing else.

> **Why not extend `provisionPreview`?** It would have to branch on scope at its top, leaving a function whose name promises a preview and whose first act is to decide whether to make one. That shape — one entry point silently meaning two things — is what produced the original defect.

### 3.3 The Supervisor control surface (FR-002b)

`requireFeatureBranch` continues to reject `prj/*` on `/begin`, `/build`, `/promote`, `/pin`, `/discard`. Project work gets its own routes in `control.mjs`:

| Route | Body | Returns |
|---|---|---|
| `POST /__supervisor/workspace/begin` | `{ branch, repoId }` | `{ ok, worktree }` |
| `POST /__supervisor/workspace/merge` | `{ branch }` | `{ ok, merged, pushed, warnings[] }` |
| `POST /__supervisor/workspace/discard` | `{ branch }` | `{ ok, warnings[] }` |

Each validates with a `requireProjectBranch` that is the mirror image of `requireFeatureBranch` — `prj/` prefix, same slug rule — so neither guard can be loosened to admit the other's branches.

`listBranches()` (the toolbar's source) is untouched and therefore cannot return a `prj/*` branch (FR-007d). This requires no code and is asserted by a test rather than implemented.

### 3.4 Reclamation, including the legacy case (FR-002e)

`reconcileFeatureBranches` gains one condition, not a prefix check: a branch whose recorded scope is `{ kind: "repository" }` is never reclaimed. `branchScope(dataDir, branch)` already exists in `coupled-repos.mjs` and reads the canonical scope file.

This covers both populations: new `prj/*` branches are invisible to the enumeration anyway, and pre-existing `bos/*` branches scoped to a project — `bos/playable-core`, `bos/game-fpsoffice` — are protected by the scope they already carry. Nothing is renamed.

### 3.5 Branch → git root becomes injectable (FR-004, C2)

Both spec-fs implementations compute `path.join(worktree, "specs", storeId)`. That expression encodes *how the Supervisor lays out coupled mounts*, which is not a fact a filesystem should know. It is replaced by an injected resolver:

```ts
export interface BranchRootResolver {
  /** The git root a read/write on `branch` resolves to, or null when nothing is mounted yet. */
  rootFor(branch: string): Promise<{ gitRoot: string; mountError?: string } | null>;
}
```

Two implementations:

- `coupledMountResolver(storeId)` — `supervisorBeginOrThrow(branch)` then `<worktree>/specs/<storeId>`. Today's behaviour, moved.
- `projectWorkspaceResolver(repoId)` — `supervisorWorkspaceBegin(branch, repoId)` then the workspace root itself.

Everything else in `SpecFS` — the writable gate, `prepareWrite`, atomic write, `commitOnSave`, `repoOffset` scoping, the self-worktree hand-off for standalone dev — is unchanged and now serves both. `src/lib/dev/spec-fs.ts` takes the same resolver, because `AGENTS.md` §1.3 (L) requires the two to stay interchangeable and this is exactly the kind of divergence that has bitten before.

`/Projects/<id>` is then `SpecFS(repoRoot, id, worktrees, writable, /* repoOffset */ "", projectWorkspaceResolver(id))`.

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

`storeMountPath(store)` becomes the single resolver (FR-004c):

```ts
export function storeMountPath(store: SpecStore): string;      // "/Projects/fpsoffice" | "/Specs/user-specs"
export function storeContentPath(store: SpecStore): string;    // + repoOffset, where the SPECS are
```

Every site composing `/Specs/${id}` by hand is rewritten to call it. A lint rule is not proposed; a grep in the task list is, because the set is small and closed.

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
- **no** `buildCandidateWithRetry`,
- a completion note reporting branch and commit count only — reusing the pure-function shape of `candidateNote` (which exists precisely because the wrong variant of it was load-bearing in a production incident) as a sibling `projectNote`.

`bosSourceTrigger`'s refusal text gains a third sentence naming the project path, so an agent steered off BOS source is not steered into `contentOnly` by elimination.

A test seam mirrors the existing `_setAgentLayerForTests` convention so the chosen cwd is assertable without a Supervisor.

### 3.8 Build Studio: starting work (FR-006, FR-006a/b/c)

`openStoreMenu` gains **New feature branch…** on a project group row. It calls the *same* `createAndActivateFeatureBranch` pair the two existing entry points use (FR-006c), with `{ scope: "repository", scopeId: group.name }` fixed — the row answered the scope, so no picker is shown (FR-006a). The prefix follows from the scope (§3.1).

`withFeatureBranch` at the point of a write is untouched.

The active project + branch is surfaced in the Build Studio header (FR-006b), reading the conversation's `activeFeatureBranch` and its recorded scope — no new state.

### 3.9 The branch list (FR-007 … FR-007d, C3)

**Grouping reuses the enumeration that already exists.** `gitState(repoRoot)` in `/api/repositories` already globs `refs/heads/bos/` per repository; it extends to both prefixes and is lifted into one shared helper so `/api/repositories` and the branch list cannot disagree about what exists.

`GET /api/assistant/feature-branches` returns groups:

```ts
{ groups: [ { repoId: "bos-src", label: "BrowserOS", branches: ["bos/x"] },
            { repoId: "fpsoffice", label: "FPSOffice", branches: ["prj/playable-core"] } ] }
```

The flat `featureBranches: string[]` is **removed**, not kept alongside — two shapes for one answer is how a consumer ends up reading the stale one. Both consumers (`AgentSelector.tsx`, `ElicitationCards.tsx`) are updated in the same change.

**Truthfulness is fixed at the source, not in the reader** (FR-007a). On promote and discard, the Supervisor already calls `clearCoupledBranchScope`; it additionally notifies BOS over the existing `reconcileViaApi` channel, and BOS clears `activeFeatureBranch` on every conversation pointing at that branch — through `setConversationActiveFeatureBranch`, which is queued per conversation and therefore serialises against the agent loop's own saves. The Supervisor must **not** write the conversation files directly: it would be a second writer to a file with an owner.

`listDeclaredFeatureBranches` survives, unchanged in purpose (FR-007b): a BOS branch activated but not yet materialised still has no ref, and a second conversation must still be able to select it. With the pointer cleared at merge/discard it stops being a source of resurrection. Declared names join the BrowserOS group.

### 3.10 Credentials (FR-008, FR-008a/b, C4)

One helper replaces the two hand-copied `buildAuth` functions:

```ts
// src/lib/gitops/auth.ts
export async function resolveAuthForRepo(repoPath: string, remoteName: string): Promise<GitAuth | null>;
```

It reads the remote's URL from the repository, then delegates to the existing, correct `resolveAuthForUrl` — which matches by host and handles provider-wide OAuth. `git-push.ts` and `git-mounts.ts` both call it. The inline `authType` + `token` override is retained for the genuine case (a credential BOS does not hold) but is no longer the only path that works.

---

## 4. File / module plan

**New**

| Path | What |
|---|---|
| `tools/supervisor/lib/workspace.mjs` | project workspace provision / merge / discard |
| `src/lib/specs/project-fs.ts` | `projectWorkspaceResolver`, `RetiredMountFS` |
| `seed/skills/bos-domain/references/target-project.md` | the fourth shape |
| `tests/supervisor/project-workspace.test.mjs` | FS-level assertions (§6) |
| `tests/specs/project-mount.test.ts`, `tests/specs/branch-prefix.test.ts`, `tests/gitops/repo-auth.test.ts` | |

**Modified**

| Path | Change |
|---|---|
| `src/lib/agent/feature-branch.ts` | prefix-aware helpers |
| `src/app/api/assistant/feature-branches/route.ts` | prefix from scope; grouped GET |
| `src/lib/agent/subagents/claude-runner.ts` | third arm, workspace note, `projectNote`, refusal text |
| `src/os/fs/spec-fs.ts` + `src/lib/dev/spec-fs.ts` | injected `BranchRootResolver` (kept in sync) |
| `src/lib/specs/spec-mount.ts` | `/Projects` mounts + retired-prefix refusal |
| `src/lib/specs/stores.ts` | `storeMountPath` / `storeContentPath` |
| `src/apps/build-studio/index.tsx` | New feature branch… ; active project/branch header |
| `src/components/apps/assistant/AgentSelector.tsx`, `ElicitationCards.tsx` | grouped list |
| `src/app/api/repositories/route.ts` | shared ref enumeration, both prefixes |
| `src/lib/gitops/auth.ts`, `tools/server/git-push.ts`, `git-mounts.ts` | `resolveAuthForRepo` |
| `tools/supervisor/lib/control.mjs` | `/workspace/*` routes + `requireProjectBranch` |
| `tools/supervisor/lib/worktree.mjs` | extract `addWorktree`; scope check in reclamation |
| `tools/supervisor/lib/reconcile-client.mjs` | notify-on-branch-gone |
| `seed/skills/bos-domain/SKILL.md` | four shapes |
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

**D4 — Branch→git-root resolution is injected into spec-fs.**
*Why:* `path.join(worktree, "specs", storeId)` is knowledge of the Supervisor's coupled-mount layout living in a filesystem. Injecting it is what lets one implementation serve both without a copy — and the two spec-fs files must stay interchangeable (`AGENTS.md` §1.3 L). *Rejected:* a separate `ProjectFS` class — a second implementation of the branch gate and commit-on-save, i.e. the divergence that is already called a security hole when it happens between the existing pair.

**D5 — The retired `/Specs/<project-id>` is a registered refusing mount.**
*Why:* `unregisterMount` does not fail closed; the path falls through and writes a decoy that reads back perfectly (`050` FR-003c). Only a registered backend can refuse. *Rejected:* leaving it mounted as an alias — two paths to one byte, explicitly rejected in the spec.

**D6 — The stale branch list is fixed by clearing the pointer at merge/discard.**
*Why:* the list is a symptom; the cause is a conversation that still claims a branch which no longer exists. Filtering at read time leaves every other reader of `activeFeatureBranch` wrong. *Rejected:* dropping `listDeclaredFeatureBranches` — it exists for the pending-BOS-branch window, which is real.

**D7 — BOS clears the pointer, not the Supervisor.**
*Why:* conversation files have an owner — the per-conversation queue in `conversation-store.ts` that serialises against the agent loop's saves. A second process writing them directly races it. `reconcileViaApi` already exists for exactly this shape. *Rejected:* direct file writes from the Supervisor.

**D8 — Project merge is `promoteCoupled` and nothing else.**
*Why:* it already merges into the repo's own default branch, handles a busy checkout via plumbing, deletes the branch, pushes, and escalates conflicts. A project-specific merge would be a second implementation of all of that. *Consequence:* conflict escalation comes for free and must be tested, not re-designed.

**D9 — The grouped branch list reuses `/api/repositories`' per-repo enumeration.**
*Why:* that route already globs `refs/heads/bos/` per repository and already cross-references branch scope. A second enumeration would be a second answer to "which branches exist". *Consequence:* the glob and the shared helper move to one place both routes import.

**D10 — The flat `featureBranches: string[]` is removed, not deprecated.**
*Why:* two shapes for one answer, with one of them stale, is the precise shape of this feature's own bug. Two consumers; both change in the same commit.

---

## 7. Risks & open questions

**R1 — `prj/` splits the branch vocabulary across two processes.** `FEATURE_BRANCH_RE` is referenced by self-heal (`bos/self-heal-<id>`) and asserted by `tests/specs/test-branch-naming.test.ts` (fixtures must be `bos/testfixture-*`). → *Mitigation:* the regex becomes prefix-parameterised with `bos/` as the default, so every existing caller is unchanged by construction; the naming test gains a case asserting `prj/` is rejected where `bos/` is required and vice versa.

**R2 — A `prj/*` branch reaching a BOS control path fails with a validation error rather than a diagnosis.** → *Mitigation:* `requireFeatureBranch`'s message names the project route when the branch carries the project prefix. The guard stays strict; only the message gets smarter.

**R3 — The `/Specs`→`/Projects` cutover is a scattered edit, and a missed site writes a decoy.** → *Mitigation:* D5 turns every missed site into a loud refusal instead of a silent write, which converts the risk from data loss into a visible error. The `storeMountPath` rewrite is then cleanup, not correctness.

**R4 — The two spec-fs implementations drift while gaining a constructor parameter.** → *Mitigation:* the existing sync requirement is explicit in `AGENTS.md`; the resolver lands in both in one commit, and a test instantiates both against the same resolver and asserts identical routing.

**R5 — Clearing `activeFeatureBranch` on merge could clear a branch a conversation is mid-run on.** A promote while a delegation is in flight is already undefined; this makes the conversation lose its branch pointer as well. → *Mitigation:* clear only when no run is active for that conversation, and log when one is. **Open:** whether to refuse the merge outright in that case — arguably the better answer, but it is a change to BOS promote semantics and is out of this feature's scope unless we decide otherwise.

**R6 — A project whose checkout is dirty when a workspace is provisioned.** `mountCoupled` handles the mounted-on-wrong-branch case; a project repo's *primary* checkout being dirty is new territory for the worktree path. → *Mitigation:* worktrees do not touch the primary checkout, so this is safe for `add`; it matters only at merge, where `coupledMergeTarget`'s `busy` branch already routes around a checkout it cannot touch.

**Open question — where the project's label comes from in the branch group header.** `SpecStore.label` is the registration label; `/api/repositories` already carries it. Assumed sufficient; flagged in case Build Studio wants `project.json`'s label instead.
