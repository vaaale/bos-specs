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
- The reconciler deleted the feature branch twice **mid-session**. A project-scoped branch has no BrowserOS commits by construction, so it is permanently "fully merged into base" and permanently eligible for reclamation.
- `git_push` could not resolve the credential `050` FR-001a had already recorded for the repository, so the agent pasted a GitLab PAT inline into a tool call — where it is now persisted in the conversation file.
- The user pushed by hand, to a different branch than the one holding the fixes. `main` and `bos/playable-core` diverged by 24 files, and the version the user ran was missing every fix.

Not one of these is a bug in the project, and not one was caught by anything. Each is a place that could have asked what the branch was for and did not.

### The single cause

`BranchScope` is declared when the branch is created, is persisted where both processes can read it, and is already authoritative for which repositories get coupled. Every defect above is a place that provisions, mounts, delegates, builds, or reclaims **without consulting it**.

This feature does not add a mechanism. It makes the one existing discriminator reach the places that ignore it.

### The fourth shape

BOS's agent-facing model names exactly three implementation shapes — `bos-core`, `builtin-app`, `marketplace-item` — and asserts that every feature is one of them. A registered project is none, so an orchestrator classifying one has no correct answer available and must improvise. The discriminator is simple and belongs in the model:

| Shape | Runs inside BOS? | Lives in | Finished by |
|---|---|---|---|
| `bos-core` | yes | BOS source | build → preview → promote |
| `builtin-app` | yes | BOS source | build → preview → promote |
| `marketplace-item` | yes | a marketplace repo | build → install → promote |
| **`project`** | **no** | **its own repository** | **commit → merge → push** |

A project is a repository BOS **drives** but does not **host**. Nothing about it is installed, bundled, served or previewed by BOS. Every requirement below follows from that one sentence.

Three names exist today for this thing — the manifest says `arbitrary`, the Repositories page says Projects, the branch scope says `repository`. One vocabulary is **project**; the on-disk `kind: "arbitrary"` value stays as it is, because `data/` must stay readable by older code.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Work delegated on a project lands in that project (Priority: P1)

A user with a registered project asks the assistant to implement something. The developer sub-agent works in that project's repository and nowhere else.

**Why this priority**: this is the feature. Everything else is a consequence of getting the working directory right.

**Acceptance Scenarios**:

1. **Given** an active feature branch scoped to a project, **When** a developer delegation runs, **Then** its working directory is a worktree of **that project's** repository, checked out on the feature branch.
2. **Given** the same delegation, **When** it starts, **Then** BrowserOS has **no** branch, **no** worktree and **no** data clone created for it, and no BrowserOS build runs.
3. **Given** the same delegation, **When** the agent is told where it is, **Then** it is told it is in a standalone repository that is not BrowserOS's source — not that it is looking at spec stores.
4. **Given** a delegation on a `bos-core` or `marketplace-item` branch, **When** it runs, **Then** nothing about its behaviour changes.

### User Story 2 - A project's code is reachable from the file tools (Priority: P1)

The assistant reads and writes a project's source with the ordinary `file_*` tools, the same way it reaches everything else.

**Why this priority**: without it the orchestrator can only reach a project's specs, so it either guesses that the spec folder is the repository root — the observed failure — or writes code into the VFS, outside git.

**Acceptance Scenarios**:

1. **Given** a registered project, **When** the assistant lists `/Projects/<id>`, **Then** it sees the repository root, with the project's spec directory as one entry among the source.
2. **Given** a write to `/Projects/<id>/src/...` on an active feature branch, **When** it lands, **Then** it is committed to that branch in that repository.
3. **Given** a write to `/Projects/<id>/...` with no active feature branch, **When** it is attempted, **Then** it is refused — the same branch-coupled contract `050` FR-003b already requires.
4. **Given** a path under the retired `/Specs/<project-id>` prefix, **When** anything addresses it, **Then** it is **refused by name**, pointing at the new path.

### User Story 3 - Finishing work means finishing it in the project (Priority: P1)

The user reviews the work and merges it, in the project's own repository.

**Why this priority**: the observed run ended with the user pushing the wrong branch by hand because nothing in BOS could complete the operation. The work being correct is worth nothing if it cannot be delivered.

**Acceptance Scenarios**:

1. **Given** completed work on a project branch, **When** the agent reports, **Then** it states commits and branch, and makes no claim about a build, a preview or a promote.
2. **Given** a project branch, **When** it is promoted, **Then** the branch is merged into **that project's** default branch and pushed if a remote is configured — and BrowserOS's base branch is neither advanced nor rebuilt.
3. **Given** a project branch, **When** it is discarded, **Then** that project's branch and worktree are removed and nothing of BrowserOS's is touched.
4. **Given** a push to a project remote registered by `050`, **When** it runs, **Then** it authenticates with the credential recorded at registration, with no credential supplied by the agent.

### User Story 4 - The branch survives being left alone (Priority: P2)

A user starts work on a project, leaves it overnight, and comes back to it.

**Why this priority**: lower than the above because the work was recoverable each time — but it was reclaimed twice during a single session, which is how a branch silently becomes someone's lost afternoon.

**Acceptance Scenarios**:

1. **Given** a project branch with no BrowserOS commits, **When** the Supervisor restarts and reconciles, **Then** the branch is still there.
2. **Given** a project branch, **When** the feature-branch picker is opened, **Then** it is listed alongside BrowserOS's own.

### Edge Cases

- **A project whose default branch is not `main`.** The feature branch is cut from whatever that repository's default actually is, and promote merges back into the same. `050` FR-003b makes BOS a contributor to a repo it does not own; guessing `master` is how that contract gets broken.
- **A project with no remote.** Must work fully locally. Promote merges; push is skipped with that said plainly, not reported as success.
- **A project with an unborn HEAD or an empty repository.** `050` FR-005 guarantees an initial commit at registration; a repository that has lost it must fail with that named, not with a git error from three layers down.
- **The spec store and the source live in one repository.** A single branch carries both. Specs are still jailed to the method-declared subdirectory (`050` FR-003); source is not (`050` FR-003a). One branch, two write paths, and they must not be conflated.
- **A project registered by symlink, outside the data dir.** Every path must derive from the recorded `repoOffset` (`050` FR-003d), never from `path.relative(repoRoot, root)`.
- **A stale `/Specs/<project-id>` reference** in an old conversation, a memory entry or a skill. `050` FR-003c documents that an unmounted `/Specs/<id>` does not fail closed — it falls through to the plain VFS and writes a real decoy file that reads back perfectly. Retiring the prefix without refusing it recreates exactly that bug.
- **Two projects, two branches, same name.** Each project's branches are its own; a `bos/x` in one project says nothing about a `bos/x` in another, or in BrowserOS.
- **A project branch while BrowserOS is mid-promote.** The two are independent operations on different repositories and must not serialise against each other.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Branch scope MUST decide which workspace a feature branch gets. A `repository`-scoped branch MUST be provisioned as a worktree of **that project's repository only**. BrowserOS MUST NOT get a branch, a worktree, a data clone, a preview port, or a build for it. The scope is already declared and already persisted; no new input is introduced.

- **FR-001a**: The isolated-checkout concern and the BrowserOS-candidate concern MUST be separated, so that a project branch can have the first without the second. Today they are one object and one provisioning sequence, which is why a project branch acquires a BrowserOS build it has no use for. A project branch MUST NOT acquire a serving or build capability by default simply because one exists for BrowserOS.

- **FR-001b**: No BrowserOS branch MUST be created for a project-scoped feature branch. This is the requirement, not an exemption from reclamation: a branch that is never created cannot be reclaimed, whereas a branch protected by an exception remains one refactor away from being eligible again.

- **FR-002**: A developer delegation MUST take its working directory from the active branch's scope. A project-scoped branch MUST place the agent at the **root of the project's worktree**. No new tool, argument or flag is introduced — the branch the user already confirmed carries the answer.

- **FR-002a**: The runtime workspace note given to a sub-agent MUST describe what the directory actually is. On a project branch it MUST state that this is a standalone repository, that it is **not** BrowserOS's source, that BrowserOS's source is not reachable from it, and where the project's specs sit within it. The note that described the mount as "spec stores … they hold the specs" is the sentence that produced the observed failure.

- **FR-002b**: No BrowserOS build or health gate MUST run for a project delegation, and the completion report MUST make **no claim about build state, preview or promote**. It reports commits and branch. A report that invites the user to "Promote to make it the base" for a change BrowserOS has no stake in is a false statement about the system's state.

- **FR-002c**: The refusal text that steers an agent away from BrowserOS source MUST name the project path as a third outcome. It currently offers two — BOS source, or standalone content generation in a scratch directory — and a project is neither.

- **FR-003**: A registered project MUST be reachable through the ordinary `file_*` tools at **`/Projects/<id>`**, rooted at the **repository root**, branch-coupled and writable on a feature branch exactly as every other branch-coupled store is.

- **FR-003a**: `/Projects/<id>` MUST be the **only** mount for a project. `/Specs/<id>` MUST be retired for projects, and `/Specs` MUST mean a spec store BrowserOS owns (`bos-system-specs`, `user-specs`, `item-*`). A project's specs are addressed at `/Projects/<id>/<repoOffset>`. Two VFS paths resolving to the same bytes is the failure this subsystem has been burned by most, and an alias is that shape.

- **FR-003b**: The retired `/Specs/<project-id>` prefix MUST be **explicitly refused**, with a message naming the replacement path. Unregistering the mount is not sufficient and is actively harmful: `050` FR-003c records that such a path writes a silent decoy under `data/vfs/`, which reads back correctly and never reaches the repository.

- **FR-003c**: Every consumer that addresses a store by VFS path MUST resolve it through **one** function from the store to its mount prefix and offset. Composing `/Specs/${id}` by hand is what makes a mount change into a scattered edit, and a missed site into a silent decoy write.

- **FR-004**: Promote on a project-scoped branch MUST merge that branch into **that project's own default branch**, push when a remote is configured, and then remove the branch and its worktree. BrowserOS's base branch MUST NOT be advanced and BrowserOS MUST NOT be rebuilt.

- **FR-004a**: Discard on a project-scoped branch MUST remove that project's branch and worktree, and nothing of BrowserOS's.

- **FR-004b**: A promote that merges but cannot push MUST report that, naming the cause. Reporting success for a half-finished operation is the specific failure the Supervisor's promote path already has history with.

- **FR-005**: The feature-branch picker MUST list project branches alongside BrowserOS's. With no BrowserOS ref backing them (FR-001b), a listing that reads only BrowserOS's refs would hide every project branch and make a branch in progress unselectable from another conversation.

- **FR-006**: Git operations against a project MUST resolve credentials **from the remote's URL**, through the same single mechanism `050` FR-001a already requires for cloning. Resolving by remote NAME cannot work: `origin` names every remote BOS knows, and provider-wide OAuth — the credential `050` records at registration — is keyed by provider and is unreachable without it.

- **FR-006a**: An agent MUST NOT need to supply a credential to push to a repository BOS registered. The observed workaround put a live GitLab PAT into a tool-call argument, where it is persisted in the conversation file.

- **FR-006b**: The credential-resolution policy MUST exist once. It is currently hand-copied across git tool modules, and the copies already disagree with the one correct implementation.

- **FR-007**: The agent-facing model MUST carry **project** as a fourth implementation shape, with its own reference covering: that code goes at the repository root, that specs go in the method-declared subdirectory, that the project is built and run with its **own** toolchain, that BOS never installs or serves it, and that finishing means merging into its own default branch.

- **FR-007a**: That reference MUST state that BOS provides **no** green light for a project: there is no build gate, no health check and no preview, so a claim that a project works MUST come from the agent having run the project's own checks and MUST be reported honestly. Three consecutive delegations in the observed run reported verified work on a game that died on load.

- **FR-008**: Documentation MUST cover working on a project end to end — register, branch, delegate, verify, merge — and MUST state plainly that BOS does not build, run or serve projects.

### Non-requirements

- **Running, building, serving or previewing a project.** Explicitly rejected, not deferred. How a project is run is a property of the project, not of BOS: a game, a Python service and a data pipeline share no answer. BOS provides an isolated checkout, the tools to work in it, and the means to deliver the result. The agent uses the project's own toolchain through the ordinary command tool, exactly as a developer would. BOS declares no build command, supervises no project process, and allocates no port.
- **A second registry of projects.** `050` deliberately has none; registration writes what discovery reads. Unchanged.
- **Changing where a project's specs live.** `050` FR-003 stands.
- **Cross-project work on one branch.** One feature branch, one project.
- **Renaming the on-disk `kind: "arbitrary"` value.** The vocabulary is unified in what the user and the agent see; `data/` stays backward-compatible.

### Key Entities

- **Workspace** — an isolated checkout on a feature branch. Every scope has one. For a project it is one worktree of that project's repository; for BrowserOS it is the source worktree plus its coupled repositories.
- **Preview** — a workspace that additionally carries a *running candidate build of BrowserOS*: its data clone, port, build state and process. Only BrowserOS-scoped branches have one.
- **Project** — a registered repository BOS drives but does not host. Fourth implementation shape.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A delegation on a project-scoped branch runs with its working directory inside that project's repository — asserted against the filesystem handed to the harness, not against intent.
- **SC-002**: After provisioning a project-scoped branch, the BrowserOS repository has no branch for it, no worktree, and no data clone, and no BrowserOS build has run.
- **SC-003**: A project's source is readable and writable through the `file_*` tools at `/Projects/<id>`, and a write lands as a commit on the feature branch in that repository — asserted by diffing the repository.
- **SC-004**: A path under the retired `/Specs/<project-id>` prefix is refused by name. No decoy is created under `data/vfs/`.
- **SC-005**: A project-scoped branch still exists after a Supervisor restart and reconciliation, and is listed in the feature-branch picker.
- **SC-006**: Promoting a project-scoped branch merges into that project's default branch and leaves BrowserOS's base commit unchanged.
- **SC-007**: A push to a project registered by `050` succeeds with no credential passed by the agent, for a remote recorded with provider-wide OAuth.
- **SC-008**: The whole of the observed run is reproducible end to end — register a project, branch, delegate, write code, verify with the project's own commands, merge — without any step writing outside that project's repository.
- **SC-009**: `npm run test:unit` green; `npm run test:supervisor` green; `npx tsc --noEmit` and `npm run lint` clean.

## Assumptions

- **The scope is already right.** `BranchScope` is declared at branch creation, persisted where the Supervisor can read it, and was recorded correctly in the observed failure. Nothing here re-derives or infers it; the work is to consult it where it is currently ignored.
- **`050` is the contract, not a starting point.** FR-003a and SC-003 already say BOS develops in the whole repository and that development writes are not confined. This feature implements them; it does not renegotiate them.
- **A project's toolchain is the project's business.** Every project knows how to build and test itself, and the agent can read that from the project. BOS adding its own opinion about it would be a second mechanism for something each project already has exactly one of.
- **Separating workspace from preview is a factoring, not a new path.** The BrowserOS-scoped behaviour is unchanged in every respect; what is new is that the BrowserOS-specific half can be absent.
