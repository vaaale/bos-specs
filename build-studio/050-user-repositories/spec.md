# Feature Specification: User repositories — four kinds, one Repositories page

**Feature Branch**: `050-user-repositories`

**Created**: 2026-09-13

**Status**: Draft

**App Target**: bos-core

**Input**: BOS can only ever see the repositories it shipped with. A user cannot add a marketplace, cannot point BOS at an existing project, and cannot start a new one — so "use BOS to build my Python app" has no answer, and OpenSpec, which organises by STORE rather than by project, has nowhere to live.

> Depends on `045-method-pack-layer`. Independent of `049-project-lifecycle` — either may ship first. Supplies the repository kinds `049` binds against, and is a prerequisite for using OpenSpec on anything but an existing store.

## Why this exists (context)

Stores are discovered by scanning `BOS_SPECS_ROOT` for directories — symlinks included — that carry a `spec-store.json`. The mechanism for "more stores" already exists and is exercised (a developer points BOS at stores outside the data dir exactly this way). **There is simply no way to add one from inside BOS.**

That single gap produces three unrelated-looking limitations:

1. **One marketplace.** `user-apps` is the only writable one. Another user-owned marketplace cannot be added, so items cannot be organised into more than one repo.
2. **No non-BOS work.** BOS can drive a spec pipeline, but only over BOS's own specs. A Python application, a data pipeline, anything else — BOS cannot see it.
3. **OpenSpec has nowhere natural to live.** It declares its sections at the STORE root (`changes/`, `specs/`), so its unit of organisation is the store. With a fixed set of stores, adopting OpenSpec means taking over `user-specs` wholesale.

`Settings → Versions` is where repositories are already surfaced — remotes, branches, sync — so it is the page this belongs on, and the name no longer describes it.

### The four kinds

Binding scope is a property of the repository KIND. One rule cannot be right for all four: a marketplace repo holds many independent products, while `user-specs` holds refinements to ONE product and therefore has one pipeline for all of it.

**Kind and ownership are different axes.** Kind decides binding scope and what a Project means inside the repo. Ownership decides writability. Only `bos-system-specs` is BOS-owned and read-only; **every other kind, arbitrary repos included, is USER-OWNED and therefore writable on a branch** — the same contract `user-specs` and `user-apps` already have.

| Kind | What a Project is | Bind at | Owned by | Writable | How many |
|---|---|---|---|---|---|
| **Marketplace store** | each **item** — one repo, many projects | each item | user | yes, on a branch | many |
| **BOS system specs** | — | nowhere | **BOS** | **no** | exactly one |
| **BOS user specs** | the store **is** one project | the store | user | yes, on a branch | exactly one |
| **Arbitrary repo** | the repo **is** one project | the store | **user** | yes, on a branch | many |

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Add an existing repository (Priority: P1)

A user clones a repository they already have and BOS starts driving a spec pipeline over it.

**Why this priority**: the common case. Most work BOS would be useful for already exists somewhere.

**Acceptance Scenarios**:

1. **Given** a reachable git URL, **When** the user adds it as an arbitrary repository, **Then** it is cloned, registered, and appears as a store bound to the default workflow.
2. **Given** a clone that fails (bad URL, auth, network), **When** it fails, **Then** the failure names the cause and leaves NO half-registered repository behind.
3. **Given** a repository already containing a framework's layout (e.g. an `openspec/` directory), **When** it is added, **Then** BOS detects it and offers that method rather than imposing the default.

### User Story 2 - Start a new repository (Priority: P1)

**Acceptance Scenarios**:

1. **Given** a name and a kind, **When** the user creates a new repository, **Then** it is `git init`-ed, scaffolded for the chosen workflow, and has an initial commit — an unborn HEAD is not a usable store.
2. **Given** no remote is configured, **When** it is created, **Then** it works fully locally. A remote is an optional later step, never a precondition.

### User Story 3 - Repositories is a page a user can understand (Priority: P1)

`Settings → Versions` becomes **Repositories**, redesigned around "what repositories does BOS know about, and what state are they in".

**Why this priority**: the page already carries remotes, branches and sync, and will now carry add/remove. Bolting onto the current layout is how it becomes unusable.

**Acceptance Scenarios**:

1. **Given** the page, **When** it opens, **Then** every known repository is listed with its KIND, its branch, and whether it has uncommitted or unpushed work.
2. **Given** a repository with local changes, **When** the user looks at the list, **Then** that is visible without opening anything.
3. **Given** `bos-system-specs`, **When** shown, **Then** it is clearly read-only and offers no mutating action.

### User Story 4 - More than one marketplace (Priority: P2)

A user adds a second marketplace repository; the Marketplace app lists items from every registered one.

**Why this priority**: real, but `user-apps` already works for a single-marketplace user. It ranks below the kinds that today have no answer at all.

**Acceptance Scenarios**:

1. **Given** two registered marketplace stores, **When** the Marketplace app opens, **Then** items from both are listed, each attributed to its source.
2. **Given** two marketplaces offering the same item id, **When** both are listed, **Then** the collision is REPORTED, not resolved by registration order.
3. **Given** a user-owned marketplace, **When** an item in it is edited, **Then** the write lands on that repo's branch — not on `user-apps`.

### Edge Cases

- **Removing a repository.** Must distinguish *forget* (stop tracking, leave the files) from *delete* (remove them). Defaulting to delete loses work; defaulting to forget silently leaks disk. Both must be offered, named exactly.
- **A repository whose method is not installed** still lists, carrying the reason (`045` FR-016) — a store that vanishes reads as data loss.
- **Adding a repository that is already registered**, by URL or by path, must be refused rather than producing two stores over one checkout with divergent branch state.
- **`bos-system-specs` and `user-specs` are not removable.** They are BOS's own. The page must not offer an action that would break the product.
- **An arbitrary repo is the USER'S**, and BOS develops in it — source included. What is confined is the SPEC store (FR-003), so a framework's own CLI keeps working; what is not confined is development, which is the point. Both ride a branch (FR-003b).
- **A repository BOS did not create** may already have its own branch conventions and protected branches. Writing to a feature branch rather than the default one is what keeps BOS a well-behaved contributor to a repo it does not own outright.
- **Nested or overlapping repositories** (adding a repo inside another registered one) must be refused: two stores writing one worktree is a corruption path.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST let a user register a repository by **cloning an existing one** or by **creating a new one** (`git init`), choosing its KIND at registration.

- **FR-001a**: A clone MUST authenticate through the **platform's one credential mechanism** — the shared git runner and its credential helper — with the credential resolved from the URL's host (the provider connected in Settings → Integrations → Git Providers, or a token already stored for a remote on that same host). A bare `git clone` shell-out MUST NOT be used: it carries no credential and no terminal, so every private repository failed with git's `could not read Username for '<host>'`. When no credential is configured for the host, the failure MUST say **that**, and where to configure it — a state distinct from a credential the host rejects.

- **FR-002**: BOS MUST recognise the four kinds of FR-002a and apply each one's binding scope. Kind MUST be recorded in the store's own manifest, so it survives a restart and travels with the repository.

- **FR-002a**: The kinds and their scopes are: **marketplace store** (projects are its items; bind per item; many allowed), **BOS system specs** (read-only, binds nowhere, exactly one), **BOS user specs** (the store is one project; bind at store level; exactly one — there is no reason to mount a second), **arbitrary repo** (the repo is one project; bind at store level; many allowed).

- **FR-003**: An arbitrary repository's **spec store root** MUST be a **method-declared subdirectory** of the repo, not the repo root. OpenSpec's own convention is `openspec/`; that framework's CLI must still work on the same checkout. This is where the SPEC-FS jail applies — the same path containment item stores already have.

- **FR-003a**: **The repository as a whole is USER-OWNED, and BOS develops in it.** The subdirectory of FR-003 bounds the spec store, NOT BOS. Driving a pipeline over a Python application is pointless if BOS cannot write that application's source, so the ordinary development tools operate on the whole repo. FR-003 and FR-003a are two different write paths and MUST NOT be conflated: specs are jailed to the store root so a framework's CLI still finds them; code is not.

- **FR-003b**: **Every write to an arbitrary repository MUST ride a feature branch**, never land directly on its default branch — the same branch-coupled contract `user-specs` (`020`) and `user-apps` (`038`) already have, for the same reason. BOS is not the only thing touching a user's repository, and work that arrives unbidden on `main` is not reviewable. The default naming follows BOS's existing `bos/*` convention; the branch is chosen the same way it is for every other store.

- **FR-003c**: A registered repository's spec store MUST be reachable through the ordinary `file_*` tools at `/Specs/<store-id>`, like every other spec store. The VFS mount table MUST be derived from the discovered stores, not from a hardcoded list — a store with no mount does not fail closed. `/Specs/<unmounted-id>/…` falls through to the plain VFS and writes a real decoy file under `data/vfs/Specs/<id>/`, which reads back perfectly and never reaches the repository; Build Studio then reports `Could not load "<id>/…"` for content the agent has just confirmed it wrote. Registering or deregistering a repository MUST update the mounts without a restart.

- **FR-003d**: Every path a store's content is addressed by MUST be derived from its recorded `repoOffset` (FR-003's method-declared subdirectory), never re-derived as `path.relative(repoRoot, root)` — a store is reached by SYMLINK, so `root` is the link and that subtraction walks out of the data dir and back. This binds BOTH spec-fs implementations, the git surfaces that address a path by repo (`git show <ref>:<path>`, `git log -- <path>`), and the draft-branch listings, which MUST speak ONE path vocabulary: store-relative.

- **FR-004**: Adding a repository that already carries a recognisable framework layout MUST detect it and offer that method, rather than imposing the default over a repo that is already using one.

- **FR-005**: A newly created repository MUST have an initial commit before it is registered. An unborn HEAD is not a usable store, and the failure surfaces later as unrelated git errors.

- **FR-006**: A failed add MUST leave nothing registered and nothing on disk that a later add would trip over. Partial registration is worse than failure because it looks like success.

- **FR-007**: Removing a repository MUST distinguish **forget** from **delete**, and MUST name which is which. `bos-system-specs` and `user-specs` MUST NOT be removable.

- **FR-008**: Registering a repository already known — by URL or by resolved path — MUST be refused. Two stores over one worktree diverge silently.

- **FR-009**: Registering a repository nested inside an already-registered one MUST be refused.

- **FR-010**: `Settings → Versions` MUST be renamed **Repositories** and redesigned around the list of known repositories: KIND, branch, dirty/unpushed state, and the add/remove actions. Existing remote, branch and sync functionality MUST be preserved, not dropped in the redesign — a redesign is the easiest place to quietly lose a capability nobody mentions until they need it.

- **FR-010a**: The page MUST group repositories by kind (BrowserOS / Marketplaces / Projects), and each row MUST show its KIND and whether it has uncommitted or unpushed work **without opening anything**. Those are the two questions the current layout requires a click to answer; remotes become a detail inside a repository rather than the page's organising principle. See `mockup.html` (reviewed and approved) for the reference layout and for the four states it must cover: populated, empty, add, remove.

- **FR-010b**: A registered repository whose symlink no longer resolves MUST be shown as BROKEN, not omitted. Discovery skips it by design; a page that also hides it leaves a disappearance with no explanation.

- **FR-011**: The Marketplace app MUST list items from **every** registered marketplace store, each attributed to its source. An item-id collision between marketplaces MUST be REPORTED, never resolved by registration order — the same rule `045` FR-001a applies to agent ids.

- **FR-012**: A write to an item in a user-owned marketplace MUST land on THAT repository's branch. `user-apps` must not become an implicit destination for every marketplace.

- **FR-013**: Every repository BOS knows about MUST appear on the Repositories page, including the ones it ships with. A page that shows only what the user added hides exactly the repositories whose state most often explains a problem.

- **FR-014**: Documentation MUST cover adding a repository of each kind, and MUST state where BOS writes inside an arbitrary repo.

### Non-requirements

- **Publishing a marketplace.** Registering and using one is in scope; publishing is a separate decision.
- **Authoring workflows** — `051`.
- **Project lifecycle inside a repository** — `049`.
- **A second `user-specs`.** Explicitly rejected: user specs are refinements to one product, so a second store has no meaning.

### Key Entities

- **Repository kind** — what a repo IS to BOS, and therefore where binding applies and what a Project means inside it.
- **Store root offset** — the method-declared subdirectory of an arbitrary repo that BOS may write in.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A user can clone an existing repository and a new one, of each writable kind, entirely from the Repositories page.
- **SC-002**: Each kind binds at its declared scope and nowhere else.
- **SC-003**: An arbitrary repository's SPEC writes land only inside its method-declared subdirectory — asserted by diffing the repo, not by inspecting intent. Development writes are not so confined, by design.
- **SC-003a**: No write to an arbitrary repository lands on its default branch. Every one rides a feature branch, exactly as `user-specs` and `user-apps` writes do.
- **SC-004**: A failed add leaves no registered store and no residue that blocks a retry.
- **SC-005**: The Marketplace app lists items from every registered marketplace, attributed; a duplicate item id is reported.
- **SC-006**: Repositories lists every repository BOS knows about with kind, branch and dirty/unpushed state; `bos-system-specs` offers no mutating action; neither BOS store can be removed.
- **SC-007**: Every remote, branch and sync capability present on the old Versions page still works after the rename and redesign.
- **SC-008**: A store bound to OpenSpec can be created as its own repository, with `changes/` and `specs/` where that framework's CLI expects them.
- **SC-009**: `npm run test:unit` green; `npx tsc --noEmit` and `npm run lint` clean.

## Assumptions

- **The discovery mechanism already works.** Stores are found by scanning for `spec-store.json`, symlinks included. This feature adds registration and a UI, not a new discovery path — and must not introduce a second one.
- **An arbitrary repo is the user's, and BOS is a contributor to it.** That cuts both ways: BOS may develop in it freely (it was added precisely so BOS could), and BOS works on a branch so the user reviews before anything reaches their default branch. The restraint that matters is not "write less" but "write where it is expected and where it can be reviewed" — specs where the framework's CLI looks for them, everything on a branch.
- **Kind is durable.** It lives in the store manifest so it survives a restart and is visible to anything that reads the store, rather than being inferred from where a directory happens to sit.
