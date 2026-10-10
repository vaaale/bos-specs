# Tasks: User repositories (050)

Ordered so the discovery change lands before anything depends on it, and the
largest, least urgent part (multiple marketplaces) comes last.

## Phase 1 — a store root may be inside its repo (T001 … T003)

- **T001 — `repoRoot` is the nearest ancestor with a `.git`.**
  `listStores()` today requires the store directory itself to be a git repo and
  sets `repoRoot: dir`, so a subdirectory store root is skipped entirely.
  *Acceptance:* a store whose root is `<repo>/openspec` is discovered, with
  `root` = the subdir and `repoRoot` = the repo. `user-specs` and item stores
  are unchanged — one rule, three cases.
  *Guard:* resolve once per scan (`design.md` R1), not per call.

- **T002 — `kind` in the store manifest, read not inferred.**
  `store-kind.ts` currently guesses `user-specs` for every directory-scanned
  store. A manifest `kind` survives a restart and travels with the repository.
  *Acceptance:* an absent `kind` keeps today's behaviour exactly — no migration,
  and no existing store changes meaning.

- **T003 — `arbitrary` as a kind.** Binds at store level, projects are folders,
  writable on a branch (`049` FR-011's table).

## Phase 2 — registration (T004 … T008)

- **T004 — `repositories.ts`: register by clone, or by `git init`.**
  Ensure the store root exists, write `spec-store.json`, symlink
  `specsRoot/<id>` → the store root. No registry file: a second source of truth
  next to the existing scan is how "my repo vanished" happens (`design.md` §3.1).
  *Acceptance:* registering is inspectable entirely on disk; removing the
  symlink fully deregisters.

- **T005 — An initialised repo gets a commit before it is registered.**
  An unborn HEAD is not a usable store and surfaces later as unrelated git
  errors.

- **T006 — A failed add leaves NOTHING.** No symlink, no half-written manifest,
  no directory a retry would trip over. Partial registration is worse than
  failure because it looks like success.

- **T007 — Refuse a duplicate or nested registration.**
  Duplicate by URL *or* resolved path; nested inside an already-registered repo.
  Both put two jails over one worktree. REFUSED, not warned — a warning beside a
  completed registration is not a gate.

- **T008 — Detect an existing framework layout (FR-004).**
  An `openspec/` directory means OpenSpec. Offer it **preselected**, never
  applied silently (`design.md` R4).

## Phase 3 — the Repositories page (T009 … T013)

- **T009 — Rename `Versions` → `Repositories`** in the config registry, and move
  the components. Every remote/branch/sync capability must survive the rename —
  SC-007 is about not losing function in a redesign.

- **T010 — Reorganise by repository** (`mockup.html`, approved): kind and
  dirty/unpushed state on the row; remotes demoted to a detail inside it.
  Grouped BrowserOS / Marketplaces / Projects.

- **T011 — Add-repository flow.** Clone vs create, kind, workflow — with the
  detected method preselected and the copy that says where BOS writes.

- **T012 — Remove: forget vs delete.** Named as two operations, quoting the
  unpushed commits that would be lost. `bos-system-specs` and `user-specs` are
  not removable.
  *Acceptance:* neither is offered a remove action at all — an action that
  always refuses is worse than no action.

- **T013 — A dangling symlink shows as BROKEN, not hidden** (`design.md` R2).
  Discovery skips it; the page must not, or the disappearance has no
  explanation.

## Phase 4 — many marketplaces (T014, T015) — DEFERRED

> **Deferred, with the reason.** This is the only part of `050` that unblocks
> nothing else: arbitrary repos give you non-BOS work, and the page gives you
> visibility, without it. One writable marketplace is not currently painful.
>
> The cost is not a constant becoming an array. `user-apps` is BRANCH-COUPLED
> (`038`): `spec-fs` resolves `<previewDataDir>/user-apps` so an item's spec
> travels with its app's code onto a feature branch. A second writable
> marketplace needs the same coupling, which is the Supervisor's worktree
> machinery — not a loop over a list. `LOCAL_MARKETPLACE_ID` has 18 call sites
> across two files on top of that.
>
> **Revisit when SHARING is the need** — handing a colleague a marketplace you
> both write to. Organisation alone (splitting a large collection) does not
> justify touching branch coupling.

- **T014 — More than one writable marketplace.** `LOCAL_MARKETPLACE_ID` is
  singular throughout `marketplace/client.ts`; this is the bulk of the work and
  the least urgent.
- **T015 — The Marketplace app lists every registered marketplace**, attributed,
  with an item-id collision REPORTED rather than resolved by registration order
  (`045` FR-001a's rule).

## Phase 5 — docs and verification (T016, T017)

- **T016 — Docs**: adding each kind, and **where BOS writes inside an arbitrary
  repo**. A user cannot infer that, and it is their source tree.
- **T017 — Live run. DONE.** Add a real repository, bind it, create a project in
  it, confirm by DIFFING the repo that spec writes landed only in the method's
  subdirectory and that nothing touched its default branch.

  Run against a real GitLab clone (`police-mcp`, spec-kit, store root `specs/`).
  Seven assertions, all passing on the second attempt:

  | | |
  |---|---|
  | the default branch is untouched | PASS |
  | the write is on the feature branch | PASS |
  | every change is inside the method's folder | PASS *(failed first)* |
  | no source file outside it was touched | PASS |
  | the user's checkout was not switched | PASS |
  | the user's working tree stayed clean | PASS |

  **It earned its place.** The first attempt wrote `t017-check/project.json` at
  the repository ROOT rather than inside `specs/`: the offset from repo to store
  was derived as `path.relative(repoRoot, root)`, which walks out of the data dir
  and back for a SYMLINKED store, then cancelled out exactly against a mount at
  the same depth. Right content, wrong directory, no error. No unit test could
  see it — it needs a real repository reached the way real deployments reach one.
  Fixed by recording `repoOffset` at discovery.

## Dependencies

```
T001 → T004 → T007, T008 → T011
T002 → T003 → T010
T009 → T010 → T011, T012, T013
T004 → T017
```

## Critical path

`T001 → T004 → T009 → T010 → T017`

Phase 4 is independent of everything above and can be deferred without blocking
the rest.
