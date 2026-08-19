# Feature Specification: Projects — a grouping layer for spec stores, and a git workflow for working on them

**Feature Branch**: `037-project-layer`

**Created**: 2026-08-18

**Status**: Draft — User Stories 1-2 (the Project layer itself, and the coarse migration) are implemented; User Stories 3-8 (the per-project git workflow) are not yet built.

**App Target**: bos-core

**Input**: "Introduce the concept of a 'Project' as a folder inside a spec store, so specs that belong together (e.g. everything that's part of one app or module) can be grouped, and so specs for things that have nothing to do with BOS itself don't get conflated with BOS's own specs. On top of that, add a full git workflow for working on a project: activating it (creating or resuming a feature branch + worktree, pulling from a configured remote first), requiring all edits to happen on that branch, discarding it, renaming/deleting files, browsing and restoring file history, and pushing a project's branch to a configured remote."

> Inserts one new layer between a store (`018-external-spec-store`) and its features. `020-branch-coupled-specs`'s per-store worktree-mount, draft-branch, and promote mechanics are UNCHANGED and continue to apply at the store level for the `bos-system-specs` store specifically (see FR-010). Supersedes `018-external-spec-store` **FR-002**'s implicit assumption that a store's top-level directories ARE its features, and **FR-009**'s "reusing the existing spec-tree UI" (that UI was flat; it is now a recursive tree). Everything else in `018` (FR-001, FR-003..FR-008, FR-010) and all of `020` stand unchanged. Item-owned spec stores (`034-user-apps-marketplace-parity`, `035-install-by-symlink` — a marketplace item's own `spec/` folder) are explicitly OUT OF SCOPE for every part of this spec; see Edge Cases.

## Why this exists (context)

A store (`bos-system-specs`, `user-specs`, or a marketplace clone) held features directly at its top level (`<store>/<NNN-feature>/spec.md`). With one flat list per store, there was no structural way to tell which specs belonged together — e.g. "everything that's part of the Assistant App" — without reading every spec's content. This also meant a store like `user-specs`, whose whole point is to hold whatever the user chooses to build, conflated a user's unrelated projects into one indistinguishable list.

Separately, BOS's founding promise for specs is that "at any point in time, the spec should reflect the functionality currently implemented." That promise is hard to keep once specs are organized into meaningful groups a user actively works on, unless working on a group has a real editing discipline — a feature branch, so review and promotion of a change is a first-class, visible unit of work, not just an ambient commit-on-save.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - A store's features are grouped into Projects (Priority: P1) — Implemented

A directory-scanned store's top level holds **Projects** (folders with their own `project.json` — a name and optional description), not features directly. Below a Project, arbitrary plain sub-folders may nest to any depth purely for organization — they carry no metadata of their own. A directory (at any depth under a Project) is a **feature leaf** if and only if it directly contains `spec.md`; that rule, not depth, is what makes something a feature. Feature numbering (`NNN-slug`) resets per Project, so the same number can legitimately recur in different Projects within one store.

**Why this priority**: Everything else in this spec — the migration, and every git-workflow story below — operates on a Project, so the layer itself has to exist first.

**Independent Test**: Create two Projects in a store, each with a feature numbered `001-*`; confirm both are discovered with distinct full paths and neither is treated as a duplicate/collision.

**Acceptance Scenarios**:

1. **Given** a store with two Projects, **When** the spec tree is built, **Then** it shows store → Project → (any plain sub-folders) → feature → files, not a flat list.
2. **Given** a Project with a feature nested two plain sub-folders deep, **When** specs are listed, **Then** the feature is discovered by the presence of `spec.md`, regardless of its depth under the Project.
3. **Given** two different Projects in the same store each requesting the next feature number, **When** both allocate a new feature, **Then** each gets `001-<slug>` independently — numbering never collides across Projects, because the two features' full paths (not their bare `NNN-slug`) are what's unique.
4. **Given** a `project.json` file, **When** a store's contents are listed, **Then** the manifest itself is hidden from the listing, the same way a store's own manifest is hidden.

### User Story 2 - Existing content is migrated into one default Project per store (Priority: P1) — Implemented

Because this is a structural change to stores that may already have real content, existing content is not silently orphaned: on boot, any store content sitting outside a Project (the pre-this-feature flat layout) is moved into one coarse default Project — `bos-system-specs` gets a Project named **BOS**, `user-specs` gets a Project named **User**. This is a one-time, idempotent step; a more fine-grained reorganization is explicitly deferred to later work.

**Why this priority**: Without this, shipping User Story 1 would make every existing spec disappear from discovery the moment a store is scanned by the new Project-aware logic.

**Independent Test**: Boot BOS against a store with existing flat `NNN-feature` content; confirm it's discoverable afterward under the store's default Project, and that store-root siblings (the store's own manifest, `overview.md`, `discrepancies.md`, the constitution) are left exactly where they were.

**Acceptance Scenarios**:

1. **Given** a store with pre-existing flat feature folders and no Projects yet, **When** BOS boots, **Then** every one of those folders is moved under one new default Project and a `project.json` is written for it.
2. **Given** a store that has already been migrated, **When** BOS boots again, **Then** nothing moves and no duplicate Project is created (idempotent).
3. **Given** a store's root-level metadata (its own manifest, `overview.md`, `discrepancies.md`, `.specify/`), **When** migration runs, **Then** none of it is moved into the default Project — only feature content is.

### User Story 3 - Activating a Project provisions a feature branch (Priority: P1) — Pending

A user can **activate** a Project — from Build Studio (right-click a Project → "Activate") or via an assistant tool — which creates a feature branch for it if none exists yet (eliciting a branch name from the user or agent when needed) or resumes the existing one, and checks out a worktree for it. Before a **new** branch is created, if the store has a remote configured, changes are pulled from it first; resuming an existing branch does not re-pull. Once a Project is active, that is the branch every subsequent edit to it lands on.

Two provisioning mechanisms exist, chosen by which Project is being activated: the **BOS** Project (the store `bos-system-specs`) reuses BOS's existing Supervisor-coupled branch + preview mechanism unchanged (per `020-branch-coupled-specs` — its branch is the same branch as a BOS source-code change, and review stays coupled to a code promote). Every **other** Project gets an independent, lightweight branch + worktree scoped only to that Project's own store repo, with no BOS build or preview involved — spinning up a full BOS build to edit specs for something unrelated to BOS itself would be disproportionate, and mechanically the two cases need different machinery (see FR-002).

**Why this priority**: This is the mechanism that makes User Story 4 possible — there is nothing to "require a feature branch" for until activation exists.

**Independent Test**: Activate a Project with no existing branch and a configured remote with new upstream commits; confirm the remote is pulled before the branch is created, and that the resulting worktree is checked out on the new branch. Reactivate the same Project later; confirm no pull happens and the existing branch/worktree is reused.

**Acceptance Scenarios**:

1. **Given** a Project with no branch yet, **When** it is activated, **Then** a branch name is elicited (if not supplied) and a new branch + worktree are created for it.
2. **Given** a Project with an already-active branch, **When** it is activated again, **Then** the existing branch/worktree is reused with no new branch and no elicitation.
3. **Given** a store with a configured remote, **When** a NEW branch is about to be created for one of its Projects, **Then** the remote is fetched and fast-forward-merged into the store's default branch first.
4. **Given** the "BOS" Project, **When** it is activated, **Then** the existing Supervisor preview/branch mechanism runs, unchanged.
5. **Given** any other Project, **When** it is activated, **Then** no BOS build or preview is started — only a branch and worktree for that Project's own store.
6. **Given** an agent conversation, **When** the agent calls the activate tool for a Project, **Then** the same activation happens as the equivalent right-click action.

### User Story 4 - All work on an active Project happens on its feature branch (Priority: P1) — Pending

Once Project support exists, editing a Project's specs is not allowed outside an active branch: a write, rename, or delete attempted against a Project with no active session is refused with a clear message, instead of silently landing on the store's base checkout.

**Why this priority**: This is the actual guarantee the user asked for ("if work is to be done on a project it must be done through a feature branch") — without it, activation is optional advice, not a rule.

**Independent Test**: Attempt to edit a file in a Project that has never been activated; confirm the edit is refused with a message naming the missing activation, both via the UI and via the equivalent agent tool call.

**Acceptance Scenarios**:

1. **Given** a Project with no active session, **When** a write/rename/delete is attempted against any file under it, **Then** it is refused with an error naming the Project and telling the caller to activate it first.
2. **Given** a Project with an active session, **When** a write/rename/delete is attempted, **Then** it succeeds and lands on that Project's branch/worktree.
3. **Given** an item-owned store, **When** it is written to, **Then** this rule does not apply — it keeps its existing, separate commit-on-save behavior (see Edge Cases).

### User Story 5 - Rename and delete files (Priority: P2) — Pending

A file inside an active Project can be renamed or deleted by right-clicking it in Build Studio, and the assistant has the equivalent capability. Both auto-commit the same way an ordinary edit does — no separate commit step.

**Why this priority**: A real editing workflow needs file-management, not just content edits; this was the one gap called out explicitly against the existing tool set.

**Independent Test**: Rename a file and delete another file in an active Project, once via the UI and once via the agent tool; confirm both are committed automatically and neither is possible against an inactive Project.

**Acceptance Scenarios**:

1. **Given** an active Project, **When** a file is renamed or deleted (via UI or tool), **Then** the change is committed automatically, same as a content edit.
2. **Given** an inactive Project, **When** a rename or delete is attempted, **Then** it is refused the same way User Story 4 refuses a write.

### User Story 6 - Discarding a Project (Priority: P2) — Pending

An active Project can be discarded by right-clicking it and confirming in a dialog. Discarding deletes its worktree and its feature branch. For the "BOS" Project this is the existing Supervisor discard; for any other Project it is the equivalent lightweight operation against that Project's own store.

**Why this priority**: The existing merge/discard path (the store's Candidate mechanism) isn't reachable from a Project's own context menu today; this closes that gap.

**Independent Test**: Activate a Project, make an edit, discard it; confirm a confirmation dialog is shown, and afterward the branch and worktree no longer exist while the store's base content is untouched.

**Acceptance Scenarios**:

1. **Given** an active Project, **When** "Discard" is chosen, **Then** a confirmation dialog is shown before anything happens.
2. **Given** confirmation, **When** discard proceeds, **Then** the Project's worktree and feature branch are deleted and its base-branch content is unaffected.

### User Story 7 - Browsing and restoring a file's history (Priority: P3) — Pending

Right-clicking a file offers "View history," opening a master/detail dialog: the master list shows every version (commit) that touched the file, and selecting one shows its content in the detail pane. A "Restore" action replaces the file's current content with a selected historical version.

**Why this priority**: Specs are stored in git specifically so history exists; there was no way to browse it. Lower priority than the activate/discard/edit path since it's a read-mostly convenience feature.

**Independent Test**: Edit a file twice, open its history, confirm both versions are listed with their content; restore the older one and confirm the file's current content matches it (as a new commit, not a rewritten one).

**Acceptance Scenarios**:

1. **Given** a file with multiple commits (on any branch of its store, not only the currently active one), **When** its history is opened, **Then** every version is listed with enough detail to identify it (date, message).
2. **Given** a selected historical version, **When** it is viewed, **Then** its content is shown in the detail pane without altering the current file.
3. **Given** a historical version, **When** "Restore" is chosen, **Then** the file's current content becomes that version's content, committed as a new change.

### User Story 8 - Pushing a Project's feature branch to a remote (Priority: P3) — Pending

Right-clicking an active Project offers "Push feature branch," listing one menu item per remote configured for that Project's store. Separately, the existing Settings → Versions "auto-push on promote" mechanism (push to a configured remote automatically when a version is promoted) is verified to work end-to-end, since it was never confirmed as part of this effort.

**Why this priority**: Lowest priority — a manual convenience on top of git operations that already exist (`git push`) once a branch exists at all.

**Independent Test**: Configure two remotes for a store, activate one of its Projects, push to each remote via the menu; confirm both remotes receive the branch. Separately, promote a version with an `autoPush` remote configured and confirm it receives the push.

**Acceptance Scenarios**:

1. **Given** a store with N configured remotes, **When** "Push feature branch" is opened on one of its active Projects, **Then** N menu items are shown, one per remote.
2. **Given** a remote is chosen, **When** push runs, **Then** the Project's current branch is pushed to that remote.
3. **Given** a remote configured with `autoPush` enabled, **When** a version is promoted, **Then** that remote receives the push as part of the promote.

### Edge Cases

- Item-owned spec stores (a marketplace item's own `spec/` folder) are excluded from EVERY part of this spec — the Project layer, the migration, and the entire git workflow (User Stories 3-8) never apply to them. An item store is a bare subdirectory of a repo shared with sibling items (the whole `user-apps` tree, or a whole marketplace clone), not its own repo — plain git has no directory-scoped branch or worktree mechanism, so "activate a feature branch for one item" would necessarily branch/expose every other item sharing that repo. Item stores keep their existing, separate, already-working model (flat, one implicit feature, commit-on-save, no branch/worktree/remote of their own).
- A directory directly containing `spec.md` is always a feature leaf, even if it also has sub-folders of its own (e.g. a feature with a nested `skills/` folder) — recursion stops there; a feature's own sub-content is never itself walked looking for further feature leaves.
- Because numbering is per-Project, the same `NNN-slug` can legitimately exist in two different Projects in one store — nothing in the system may treat a bare `NNN-slug` as a unique key; the full path from the store root is the only safe identity.
- A store that still has legacy top-level content with no `NNN-` prefix at all (an older, un-numbered feature folder) is migrated the same way as a numbered one — the migration test is "not already inside a Project and not a known store-root sibling," not a pattern match on the name.
- The "BOS" Project's activation touches the separate Supervisor codebase (which deliberately does not depend on the rest of BOS's server code); its pull-before-create step has to be implemented natively there rather than reused from elsewhere.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A directory-scanned store's top level MUST hold **Projects** — sub-folders each carrying a `project.json` manifest (`{ label, description? }`) — instead of features directly. A Project's manifest is hidden from directory listings, the same way a store's own manifest is. This does not apply to item-owned stores (out of scope, per Edge Cases).
- **FR-002**: Below a Project, arbitrary plain sub-folders MUST be allowed to any depth, carrying no manifest of their own. A directory (the Project root, or any depth below it) MUST be treated as a feature leaf if and only if it directly contains `spec.md`; feature discovery MUST use this rule, not a fixed depth.
- **FR-003**: Feature numbering (`NNN-slug`) MUST be scoped to the Project it is created in (scanning that Project's whole subtree for the existing maximum), not the store as a whole. The same `NNN-slug` MAY exist in two different Projects of the same store; every place that identifies a feature MUST use its full path from the store root, never a bare `NNN-slug`.
- **FR-004**: On boot, any store content found outside a Project (pre-existing flat content) MUST be moved into one default Project for that store — `bos-system-specs` → a Project named "BOS", `user-specs` → a Project named "User" — without touching store-root siblings (the store's own manifest, `overview.md`, `discrepancies.md`, `.specify/`). This migration MUST be idempotent: re-running it against an already-migrated store MUST be a no-op.
- **FR-005**: A Project MUST be **activatable**: given no existing branch, a branch name MUST be elicited (from a user via a right-click flow, or from an agent via an equivalent tool call) and a new branch + worktree created for it; given an existing branch, activation MUST resume it without re-eliciting or re-creating anything. Activation MUST be reachable both as a Build Studio UI action (right-click a Project → "Activate") and as an assistant tool, and both MUST perform the same underlying operation.
- **FR-006**: Before a NEW branch is created for a Project (not when resuming an existing one), IF the Project's store has a remote configured, that remote MUST be fetched and fast-forward-merged into the store's default branch first.
- **FR-007**: The "BOS" Project's activation and discard MUST use the existing Supervisor-coupled preview/branch mechanism (`020-branch-coupled-specs`) unchanged — its branch remains the same branch as a BOS source-code feature branch. Every OTHER Project's activation and discard MUST use an independent mechanism scoped only to that Project's own store repo, involving no BOS build or preview.
- **FR-008**: Once ANY Project has been given the ability to be activated, a write, rename, or delete against a file in a Project with no active session MUST be refused with an error identifying the Project and stating that it must be activated first — it MUST NOT silently fall back to editing the store's base checkout. This requirement does not apply to item-owned stores.
- **FR-009**: A file inside an active Project MUST be renameable and deletable, both via a Build Studio right-click action and via an equivalent assistant tool capability; both operations MUST auto-commit, the same way a content write does, with no separate commit step required.
- **FR-010**: An active Project MUST be discardable via a right-click action that shows a confirmation dialog before proceeding; confirming MUST delete the Project's worktree and feature branch (via the Supervisor mechanism for "BOS", the independent mechanism for every other Project) and MUST leave the store's base-branch content untouched.
- **FR-011**: A file's history MUST be browsable via a master/detail view: a list of every version (commit) that touched the file — spanning the whole store's history, not only the currently active branch — and, for a selected version, its content. A "restore" action MUST be available that replaces the file's current content with a selected historical version's content, recorded as a new commit (not a destructive rewrite of history).
- **FR-012**: An active Project's feature branch MUST be pushable to any remote configured for its store, offered as one menu item per configured remote. The existing "push on promote" mechanism for remotes configured with `autoPush` MUST be verified to actually push on a real promote.

### Key Entities *(include if feature involves data)*

- **Project** — a store's top-level, `project.json`-bearing grouping folder; the unit a user activates, works on, and discards. Carries a label and optional description.
- **Feature leaf** — a directory (the Project root or any depth below it) that directly contains `spec.md`; the unit that was previously called "a feature," now identified by content rather than fixed depth.
- **Project session** — the record of a Project's currently active branch and worktree (if any); its absence is what FR-008 checks before allowing an edit.
- **Store-scoped remote** — a remote configured against a store's own git identity (unchanged from existing git-remotes configuration); the set a Project's "push" menu is built from.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Every existing spec in a real store remains discoverable immediately after this feature ships — none silently disappear because they weren't yet inside a Project.
- **SC-002**: Two Projects in the same store can each hold a feature numbered `001-*` with no error, no silent overwrite, and no UI confusion between them.
- **SC-003**: A user can go from "no branch exists for this Project" to "editing a file on its branch" in one guided action (activate), with zero manual git commands.
- **SC-004**: An edit attempted against an unactivated Project fails with a message a non-technical user can act on (what to do next), not a raw error.
- **SC-005**: Discarding a Project leaves no orphaned worktree or branch behind, verifiable by inspecting the store's repo afterward.
- **SC-006**: A file's full version history is viewable and a prior version restorable without the user ever running a git command directly.

## Assumptions

- Single-user, local BOS instance for the lightweight (non-"BOS") Project git workflow, consistent with the rest of BOS's spec-store design — no concurrent multi-user editing of one Project's branch.
- A Project has at most one active branch/session at a time; re-activating always means "resume the one branch this Project already has," not "start a second parallel branch."
- The finer-grained reorganization of the coarse "BOS"/"User" default Projects (splitting them into more meaningful groupings) is out of scope for this spec and left for later work, per the user's own explicit sequencing.
- Item-owned spec stores are a fully separate, already-specified mechanism (`034-user-apps-marketplace-parity`, `035-install-by-symlink`) and are not modified, extended, or referenced by any requirement in this spec beyond the exclusion in Edge Cases.
