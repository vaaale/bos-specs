# Feature Specification: Project development — a registered repository is developed on its own terms

**Feature Branch**: `053-project-development`

**Created**: 2026-10-01

**Status**: Draft

**App Target**: bos-core

**Input**: A user registered their game repository with BOS and asked the assistant to build it. BOS created a feature branch in its OWN source repo, checked out a full BrowserOS worktree, copied the entire data directory, built BrowserOS twice, and handed the developer sub-agent a BrowserOS checkout to work in. The sub-agent, having nowhere legal to put game code, wrote it into the VFS; a later delegation spent its whole budget moving it into the repository. The user ended up running the pre-fix copy and reported the game was unplayable.

> Implements `050-user-repositories` **FR-003a** and **SC-003**, which already specify that BOS develops in the whole of a registered repository and that development writes are deliberately not confined to the spec store. Only the SPEC half of `050` shipped. This is the development half.

## Why this exists (context)

`050` registered a repository as a **spec store** and stopped there — deliberately, to avoid a second source of truth about which repositories exist. That decision was right and is not reopened here. Its consequence was not noticed: the repository is visible to BOS *only* as its spec subdirectory, and the project around it has no representation in any execution path.

So `050` FR-003a — "the ordinary development tools operate on the whole repo" — describes something that has never been true. `kind: "arbitrary"` is read in exactly three places, and all three are UI listing. No provisioning, delegation, mount, build, promote or reclamation path consults it.

### What that produced, in one observed run

Conversation `c-munuuem3c515.json` (2026-09-30), branch `bos/playable-core`, scope correctly recorded as `{ kind: "repository", repoId: "fpsoffice" }`:

- The Supervisor logged the right decision — `branch "bos/playable-core" [repository:fpsoffice] couples: fpsoffice` — and then provisioned a BrowserOS feature branch, a full BrowserOS source worktree, and a clone of `data/` that degraded to a **full copy** when hardlinking failed. It allocated a preview port and ran `next build` twice. None of this has any relationship to a three.js game.
- The sub-agent's working directory was that BrowserOS checkout, and the runtime note it was given described the mount as "spec stores … they hold the authoritative, up-to-date **specs**". The project it was there to build was that mount. Told it was looking at specs, the agent believed the orchestrator's guess that `/Specs/fpsoffice/` was the repository root — it is `<repo>/openspec` — and wrote the whole game into the VFS at `/workspace/fpsoffice` instead, outside any repository. A third delegation existed only to relocate the result, and a fourth to fix what the relocation disturbed.
- The reconciler deleted the feature branch twice **mid-session**. A project branch carrying the `bos/` prefix has no BrowserOS commits by construction, so it is permanently "fully merged into base" and permanently eligible for reclamation.
- `git_push` could not resolve the credential `050` FR-001a had already recorded for the repository, so the agent pasted a GitLab PAT inline into a tool call — where it is now persisted in the conversation file.
- The user pushed by hand, to a different branch than the one holding the fixes. `main` and `bos/playable-core` diverged by 24 files, and the version the user ran was missing every fix.

Not one of these is a bug in the project, and not one was caught by anything. Each is a place that could have asked what the branch was for and did not.

### The single cause

`BranchScope` is declared when the branch is created, is persisted where both processes can read it, and is already authoritative for which repositories get coupled. Every defect above is a place that provisions, mounts, delegates, builds, or reclaims **without consulting it**.

This feature does not add a mechanism. It makes the one existing discriminator reach the places that ignore it — and gives a project branch a **name** that makes most of those places exclude it without having to ask.

### The fourth shape

BOS's agent-facing model names exactly three implementation shapes — `bos-core`, `builtin-app`, `marketplace-item` — and asserts that every feature is one of them. A registered project is none, so an orchestrator classifying one has no correct answer available and must improvise. The discriminator is simple and belongs in the model:

| Shape | Runs inside BOS? | Lives in | Branch | Finished by |
|---|---|---|---|---|
| `bos-core` | yes | BOS source | `bos/` | build → preview → promote |
| `builtin-app` | yes | BOS source | `bos/` | build → preview → promote |
| `marketplace-item` | yes | a marketplace repo | `bos/` | build → install → promote |
| **`project`** | **no** | **its own repository** | **`prj/`** | **commit → merge → push** |

A project is a repository BOS **drives** but does not **host**. Nothing about it is installed, bundled, served or previewed by BOS. Every requirement below follows from that one sentence.

Three names exist today for this thing — the manifest says `arbitrary`, the Repositories page says Projects, the branch scope says `repository`. One vocabulary is **project**; the on-disk `kind: "arbitrary"` value stays as it is, because `data/` must stay readable by older code.

### Why the prefix is load-bearing

`FEATURE_BRANCH_PREFIX` is `"bos/"`, and it gates `isFeatureBranch`, which in turn gates preview restoration, the top-bar branch list, feature-branch reclamation, and the Supervisor's own `requireFeatureBranch` guard. A branch named `prj/<slug>` is therefore invisible to **all four without a single exception being written**.

That is the difference between a rule and an exemption. A project branch excluded by an allow-list is one refactor away from being eligible again — which is exactly how a branch scoped correctly to `fpsoffice` still ended up reclaimed twice. A project branch that BOS's own machinery cannot name is excluded structurally.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Starting work on a project (Priority: P1)

A user has registered a project in Settings → Repositories. They open Build Studio, right-click the project, and start a feature branch on it.

**Why this priority**: it is the first thing a user does and today there is no way to say it. A branch can only be elicited as a side effect of a write, so "I want to work on this project" has no direct expression — and the scope, which decides everything downstream, is answered by a dialog that appears for a different reason.

**Acceptance Scenarios**:

1. **Given** a registered project in the Build Studio tree, **When** the user right-clicks it, **Then** a **New feature branch…** action is offered.
2. **Given** that action, **When** the user names the branch, **Then** `prj/<name>` is created **in that project's repository**, scoped to it, and made the active branch — with no scope question asked, because the row that was right-clicked already answered it.
3. **Given** an active project branch, **When** the user looks at Build Studio, **Then** which project and which branch are active is visible without opening anything.
4. **Given** a write action in a project with no active branch, **When** it is taken, **Then** a branch is still elicited at that point, as today. The explicit path is an addition, not a replacement.

### User Story 2 - Work delegated on a project lands in that project (Priority: P1)

The user asks the assistant to implement something. The developer sub-agent works in that project's repository and nowhere else.

**Why this priority**: everything else is a consequence of getting the working directory right.

**Acceptance Scenarios**:

1. **Given** an active project branch, **When** a developer delegation runs, **Then** its working directory is a worktree of **that project's** repository, checked out on that branch.
2. **Given** the same delegation, **When** it starts, **Then** BrowserOS has **no** branch, **no** worktree and **no** data clone created for it, and no BrowserOS build runs.
3. **Given** the same delegation, **When** the agent is told where it is, **Then** it is told it is in a standalone repository that is not BrowserOS's source — not that it is looking at spec stores.
4. **Given** a delegation on a `bos/` branch, **When** it runs, **Then** nothing about its behaviour changes.

### User Story 3 - A project's code is reachable from the file tools (Priority: P1)

The assistant reads and writes a project's source with the ordinary `file_*` tools.

**Why this priority**: without it the orchestrator can only reach a project's specs, so it either guesses that the spec folder is the repository root — the observed failure — or writes code into the VFS, outside git.

**Acceptance Scenarios**:

1. **Given** a registered project, **When** the assistant lists `/Projects/<id>`, **Then** it sees the repository root, with the project's spec directory as one entry among the source.
2. **Given** a write to `/Projects/<id>/src/...` on an active project branch, **When** it lands, **Then** it is committed to that branch in that repository.
3. **Given** a write to `/Projects/<id>/...` with no active branch, **When** it is attempted, **Then** it is refused — the branch-coupled contract `050` FR-003b already requires.
4. **Given** a path under the retired `/Specs/<project-id>` prefix, **When** anything addresses it, **Then** it is **refused by name**, pointing at the new path.

### User Story 4 - Finishing work means finishing it in the project (Priority: P1)

The user reviews the work and merges it, in the project's own repository.

**Why this priority**: the observed run ended with the user pushing the wrong branch by hand because nothing in BOS could complete the operation. The work being correct is worth nothing if it cannot be delivered.

**Acceptance Scenarios**:

1. **Given** completed work on a project branch, **When** the agent reports, **Then** it states commits and branch, and makes no claim about a build, a preview or a promote.
2. **Given** a project branch, **When** it is merged, **Then** it merges into **that project's** default branch and pushes if a remote is configured — and BrowserOS's base branch is neither advanced nor rebuilt.
3. **Given** a project branch, **When** it is discarded, **Then** that project's branch and worktree are removed and nothing of BrowserOS's is touched.
4. **Given** a push to a project remote registered by `050`, **When** it runs, **Then** it authenticates with the credential recorded at registration, with no credential supplied by the agent.

### User Story 5 - The branch list tells the truth (Priority: P1)

A user opens the feature-branch list and sees the branches that exist, organised by what they belong to.

**Why this priority**: the list currently accumulates every branch any conversation ever targeted, including ones merged and deleted long ago, in one flat alphabetical run. A picker whose entries mostly do not exist is one a user stops reading — and the branch is what decides where every subsequent write lands.

**Acceptance Scenarios**:

1. **Given** a branch that has been merged and deleted, **When** the list is opened, **Then** it is absent — including when an old conversation still records it as its active branch.
2. **Given** branches across BrowserOS, a marketplace and two projects, **When** the list is opened, **Then** they are **grouped by the repository they belong to**, each group named.
3. **Given** a BrowserOS branch just activated but not yet materialised by a delegation, **When** the list is opened, **Then** it is still offered — the pending case the declared-name merge exists for must survive.
4. **Given** any project branch, **When** the top toolbar's version control is opened, **Then** it is **not** listed there. The toolbar selects which BrowserOS the user is running; a project branch is not a BrowserOS.

### Edge Cases

- **A project whose default branch is not `main`.** The feature branch is cut from whatever that repository's default actually is, and merge goes back to the same. `050` FR-003b makes BOS a contributor to a repo it does not own; guessing `master` is how that contract gets broken.
- **A project with no remote.** Must work fully locally. Merge happens; push is skipped with that said plainly, not reported as success.
- **A project with an unborn HEAD.** `050` FR-005 guarantees an initial commit at registration; a repository that has lost it must fail with that named, not with a git error from three layers down.
- **The spec store and the source live in one repository.** A single branch carries both. Specs stay jailed to the method-declared subdirectory (`050` FR-003); source is not (`050` FR-003a). One branch, two write paths, not to be conflated.
- **A `bos/`-prefixed branch already scoped to a project**, created before this feature. It is not migrated automatically — renaming a branch under a user's work is not BOS's call — but it MUST be listed under its project and MUST NOT be reclaimed. The reclamation guard cannot rely on the prefix alone for these.
- **A project and BrowserOS with the same branch slug.** `prj/x` and `bos/x` are unrelated branches in unrelated repositories and must never be conflated by a listing, a picker or a worktree path.
- **Two projects with the same branch slug.** Each project's branches are its own; the worktree path must keep them apart.
- **A project branch while BrowserOS is mid-promote.** Independent operations on different repositories; they must not serialise against each other.
- **A stale reference to the retired `/Specs/<project-id>` prefix**, from any source. `050` FR-003c records that an unmounted `/Specs/<id>` does not fail closed — it writes a real decoy file under `data/vfs/Specs/<id>/` that reads back perfectly and never reaches the repository. The prefix must therefore be refused, not merely unregistered.

## Requirements *(mandatory)*

### Functional Requirements

#### Workspace and naming

- **FR-001**: Branch scope MUST decide which workspace a feature branch gets. A project-scoped branch MUST be provisioned as a worktree of **that project's repository only**. BrowserOS MUST NOT get a branch, a worktree, a data clone, a preview port, or a build for it.

- **FR-001a**: The isolated-checkout concern and the BrowserOS-candidate concern MUST be separated, so a project branch can have the first without the second. Today they are one object and one provisioning sequence, which is why a project branch acquires a BrowserOS build it has no use for. A project branch MUST NOT acquire a serving or build capability by default simply because one exists for BrowserOS.

- **FR-002**: A project feature branch MUST be named **`prj/<kebab-slug>`**, and MUST exist **only in that project's repository**. `bos/` remains BrowserOS's own prefix and is unchanged.

- **FR-002a**: The branch-name helpers MUST become prefix-aware rather than being duplicated per prefix. One validator, one normaliser, one suggester, parameterised by prefix — two near-copies of a naming rule is how the two spellings of one constant have repeatedly drifted in this subsystem.

- **FR-002b**: The Supervisor's feature-branch guard MUST continue to reject `prj/*` on the BrowserOS paths it protects (preview provisioning, build, promote, restoration, reclamation), and project workspace provisioning MUST have its **own** entry point that accepts it. The exclusion is the point: it is what makes FR-001 structural rather than conditional.

- **FR-002c**: A project worktree path MUST reflect the branch's prefix and its project, and MUST keep two projects' identically-named branches apart.

- **FR-002d**: No data clone MUST be created for a project branch. Nothing about a standalone repository reads or writes BOS's `data/`, and the clone is the step that degraded to a full copy of an 8.5 GB data directory in the observed run.

- **FR-002e**: A `bos/`-prefixed branch whose recorded scope is a project — created before this feature — MUST NOT be reclaimed and MUST be listed under its project. The reclamation guard MUST consult the recorded scope as well as the prefix, so correctness does not depend on every such branch having been renamed.

#### Delegation

- **FR-003**: A developer delegation MUST take its working directory from the active branch's scope. A project branch MUST place the agent at the **root of the project's worktree**. No new tool, argument or flag is introduced — the branch the user already confirmed carries the answer.

- **FR-003a**: The runtime workspace note given to a sub-agent MUST describe what the directory actually is. On a project branch it MUST state that this is a standalone repository, that it is **not** BrowserOS's source, that BrowserOS's source is not reachable from it, and where the project's specs sit within it. The note that described the mount as "spec stores … they hold the specs" is the sentence that produced the observed failure.

- **FR-003b**: No BrowserOS build or health gate MUST run for a project delegation, and the completion report MUST make **no claim about build state, preview or promote**. It reports commits and branch. A report inviting the user to "Promote to make it the base" for a change BrowserOS has no stake in is a false statement about the system's state.

- **FR-003c**: The refusal text that steers an agent away from BrowserOS source MUST name the project path as a third outcome. It currently offers two — BOS source, or standalone content generation in a scratch directory — and a project is neither.

#### The VFS surface

- **FR-004**: A registered project MUST be reachable through the ordinary `file_*` tools at **`/Projects/<id>`**, rooted at the **repository root**, branch-coupled and writable on a feature branch exactly as every other branch-coupled store is.

- **FR-004a**: `/Projects/<id>` MUST be the **only** mount for a project. `/Specs/<id>` MUST be retired for projects, and `/Specs` MUST mean a spec store BrowserOS owns (`bos-system-specs`, `user-specs`, `item-*`). A project's specs are addressed at `/Projects/<id>/<repoOffset>`. Two VFS paths resolving to the same bytes is the failure this subsystem has been burned by most, and an alias is that shape.

- **FR-004b**: The retired `/Specs/<project-id>` prefix MUST be **explicitly refused**, with a message naming the replacement path. Unregistering the mount is not sufficient and is actively harmful — see `050` FR-003c and the matching edge case.

- **FR-004c**: Every consumer that addresses a store by VFS path MUST resolve it through **one** function from the store to its mount prefix and offset. Composing `/Specs/${id}` by hand is what makes a mount change into a scattered edit, and a missed site into a silent decoy write.

#### Finishing

- **FR-005**: Merging a project branch MUST merge it into **that project's own default branch**, push when a remote is configured, and then remove the branch and its worktree. BrowserOS's base branch MUST NOT be advanced and BrowserOS MUST NOT be rebuilt.

- **FR-005a**: Discarding a project branch MUST remove that project's branch and worktree, and nothing of BrowserOS's.

- **FR-005b**: A merge that succeeds but cannot push MUST report that, naming the cause. Reporting success for a half-finished operation is the specific failure the Supervisor's promote path already has history with.

#### Starting work

- **FR-006**: Build Studio MUST offer **New feature branch…** on a project row, creating a `prj/*` branch in that project and activating it. This is the direct expression of "I want to work on this project", which today exists only as a side effect of attempting a write.

- **FR-006a**: A branch started from a project row MUST NOT ask which scope it is for. The row answers it. The scope picker exists for the case where the target is genuinely in question, and offering it here invites a wrong answer to a question already settled.

- **FR-006b**: Build Studio MUST show which project and branch are currently active without the user opening anything.

- **FR-006c**: Eliciting a branch at the point of a write MUST continue to work. FR-006 is an additional entry point, not a replacement, and MUST route to the same create-and-activate pair — a second branch-creation path is the duplication this product has been burned by most.

#### The branch list

- **FR-007**: The feature-branch list MUST contain only branches that **actually exist in a repository**. A branch merged or discarded MUST disappear from it.

- **FR-007a**: A conversation's recorded active branch MUST be **cleared when that branch is merged or discarded**. This is the root cause of the stale list: the declared-name merge scans every conversation file and resurrects every branch any conversation ever targeted, forever. Filtering the list at read time would leave the stale pointer in place and the next reader to consult it wrong.

- **FR-007b**: The declared-but-not-yet-materialised case MUST keep working. Under the Supervisor a BrowserOS branch has no git ref until a delegation provisions it, and dropping declared names entirely would make a just-activated branch unselectable from a second conversation.

- **FR-007c**: The list MUST be **grouped by the repository each branch belongs to** — BrowserOS, each marketplace, each project — with each group named. A flat list cannot distinguish `prj/x` in one project from `prj/x` in another, and that distinction decides where a write lands.

- **FR-007d**: Project branches MUST NOT appear in the top toolbar's version control. That control selects which BrowserOS build is being served; a project branch is not a BrowserOS and cannot be served.

#### Credentials

- **FR-008**: Git operations against a project MUST resolve credentials **from the remote's URL**, through the same single mechanism `050` FR-001a already requires for cloning. Resolving by remote NAME cannot work: `origin` names every remote BOS knows, and provider-wide OAuth — the credential `050` records at registration — is keyed by provider and is unreachable without it.

- **FR-008a**: An agent MUST NOT need to supply a credential to push to a repository BOS registered. The observed workaround put a live GitLab PAT into a tool-call argument, where it is persisted in the conversation file.

- **FR-008b**: The credential-resolution policy MUST exist once. It is currently hand-copied across git tool modules, and the copies already disagree with the one correct implementation.

#### The model and the documentation

- **FR-009**: The agent-facing model MUST carry **project** as a fourth implementation shape, with its own reference covering: that code goes at the repository root, that specs go in the method-declared subdirectory, that a project branch is `prj/*`, that the project is built and run with its **own** toolchain, that BOS never installs or serves it, and that finishing means merging into its own default branch.

- **FR-009a**: That reference MUST state that BOS provides **no** green light for a project: there is no build gate, no health check and no preview, so a claim that a project works MUST come from the agent having run the project's own checks and MUST be reported honestly. Three consecutive delegations in the observed run reported verified work on a game that died on load.

- **FR-010**: Documentation MUST cover working on a project end to end — register, start a branch, delegate, verify, merge — and MUST state plainly that BOS does not build, run or serve projects.

### Non-requirements

- **Running, building, serving or previewing a project.** Explicitly rejected, not deferred. How a project is run is a property of the project, not of BOS: a game, a Python service and a data pipeline share no answer. BOS provides an isolated checkout, the tools to work in it, and the means to deliver the result. The agent uses the project's own toolchain through the ordinary command tool, exactly as a developer would. BOS declares no build command, supervises no project process, and allocates no port.
- **Changing BrowserOS's own `bos/` prefix**, or migrating existing branches onto `prj/`. FR-002e covers the ones already scoped to a project without renaming anything under the user's feet.
- **A second registry of projects.** `050` deliberately has none; registration writes what discovery reads. Unchanged.
- **A second branch-creation path.** FR-006 adds an entry point, not a mechanism.
- **Changing where a project's specs live.** `050` FR-003 stands.
- **Cross-project work on one branch.** One feature branch, one project.
- **Renaming the on-disk `kind: "arbitrary"` value.** The vocabulary is unified in what the user and the agent see; `data/` stays backward-compatible.

### Key Entities

- **Workspace** — an isolated checkout on a feature branch. Every scope has one. For a project it is one worktree of that project's repository; for BrowserOS it is the source worktree plus its coupled repositories.
- **Preview** — a workspace that additionally carries a *running candidate build of BrowserOS*: its data clone, port, build state and process. Only BrowserOS-scoped branches have one.
- **Project** — a registered repository BOS drives but does not host. Fourth implementation shape, branch prefix `prj/`.
- **Branch prefix** — which product a feature branch belongs to, and therefore which machinery may see it at all.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can start work on a registered project from Build Studio in one action, and the branch created is `prj/*` in that project's repository.
- **SC-002**: A delegation on a project branch runs with its working directory inside that project's repository — asserted against the filesystem handed to the harness, not against intent.
- **SC-003**: After provisioning a project branch, the BrowserOS repository has no branch for it, no worktree, and no data clone, and no BrowserOS build has run.
- **SC-004**: A project's source is readable and writable through the `file_*` tools at `/Projects/<id>`, and a write lands as a commit on the feature branch in that repository — asserted by diffing the repository.
- **SC-005**: A path under the retired `/Specs/<project-id>` prefix is refused by name, and no decoy is created under `data/vfs/`.
- **SC-006**: A project branch still exists after a Supervisor restart and reconciliation, including one carrying the legacy `bos/` prefix with a project scope recorded.
- **SC-007**: The feature-branch list contains no branch that does not exist in a repository, is grouped by repository, and still offers a BrowserOS branch that has been activated but not yet materialised.
- **SC-008**: No project branch appears in the top toolbar's version control.
- **SC-009**: Merging a project branch merges into that project's default branch and leaves BrowserOS's base commit unchanged.
- **SC-010**: A push to a project registered by `050` succeeds with no credential passed by the agent, for a remote recorded with provider-wide OAuth.
- **SC-011**: The whole of the observed run is reproducible end to end — register a project, start a branch from Build Studio, delegate, write code, verify with the project's own commands, merge — without any step writing outside that project's repository.
- **SC-012**: `npm run test:unit` green; `npm run test:supervisor` green; `npx tsc --noEmit` and `npm run lint` clean.

## Assumptions

- **The scope is already right.** `BranchScope` is declared at branch creation, persisted where the Supervisor can read it, and was recorded correctly in the observed failure. Nothing here re-derives or infers it; the work is to consult it where it is currently ignored.
- **`050` is the contract, not a starting point.** FR-003a and SC-003 already say BOS develops in the whole repository and that development writes are not confined. This feature implements them; it does not renegotiate them.
- **Every registered project is reached by symlink.** Registration always links the specs root at the store root inside the repository, so `repoOffset` (`050` FR-003d) is the only safe basis for any path — not an unusual configuration to be handled separately.
- **A project's toolchain is the project's business.** Every project knows how to build and test itself, and the agent can read that from the project. BOS adding its own opinion would be a second mechanism for something each project already has exactly one of.
- **Separating workspace from preview is a factoring, not a new path.** BrowserOS-scoped behaviour is unchanged in every respect; what is new is that the BrowserOS-specific half can be absent.
- **The prefix does most of the exclusion work.** `prj/*` is invisible to BOS's preview restoration, toolbar listing, reclamation and feature-branch guard because none of them can name it. Requirements that would otherwise be exemptions become consequences — with FR-002e as the one deliberate exception, for branches that pre-date the prefix.
