# Feature Specification: Project lifecycle — standard tools, workflow-named, one implementation

**Feature Branch**: `049-project-lifecycle`

**Created**: 2026-09-13

**Status**: Draft

**App Target**: bos-core

**Input**: A usability question with no good answer — "how do I create a new project using BMAD?" Today: you cannot. `createProject()` exists in `projects.ts` and its only callers are tests; there is no UI, no API route and no agent tool. The only way to get a Project is to hand-write `project.json`.

> Depends on `045-method-pack-layer` (descriptor, binding chain, install). Independent of `050-user-repositories`. Anticipates `051-workflows`: this feature binds by WORKFLOW from the start, so `051` does not have to rework the signature it ships.

## Why this exists (context)

Three separate holes meet here, and each one alone reads as a small omission:

1. **A Project cannot be created.** `createProject()` is unreachable from the product.
2. **A method cannot be chosen at creation time.** Binding is only possible *after* authoring, so the first artifact is written under the wrong framework and then orphaned by the switch. Creating a document-processing app under BMAD today means: create it (gets `spec.md`), bind it to BMAD, watch the preflight warn that 1 unit would be hidden, then rename the file to `product-brief.md` by hand.
3. **`createItemSpec` ignores the configured default method.** It calls `resolveMethod({}, "user-apps")` — an empty binding, so no `globalDefault` — and always falls through to spec-kit. The comment directly above it says the function exists to prevent exactly this. The comment is right; the code does not do it.

The result is that **Settings → Build Studio → Default method changes nothing** about what gets created, and the method layer's finest-grained binding is reachable only by hand-editing JSON.

### Workflows, not methods

A framework's unit of pipeline is not the framework — it is a **workflow** within it, and every method already has the concept:

| Method | Workflow unit | More than one? |
|---|---|---|
| OpenSpec | `schema: spec-driven` in `config.yaml` | yes — `openspec schema fork` |
| BMAD | modules (`bmm`, `bmb`, `cis`) | yes |
| spec-kit | one pipeline | not yet |

So the user names a workflow — *"create Document Processing using the bmad-enterprise workflow"* — and BOS enumerates workflows across every installed method to find its owner. Binding a bare method id remains valid and means "that method's default workflow".

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Create a project under a chosen workflow, in one step (Priority: P1)

A user (or an agent) creates a Project and names the workflow it should use. The Project is created already bound, and its first artifact carries the name that workflow expects.

**Why this priority**: this is the whole feature. Every other story is a refinement of it.

**Independent Test**: create a project naming a non-default workflow; assert the binding is recorded and the scaffolded artifact uses that workflow's leaf marker.

**Acceptance Scenarios**:

1. **Given** BMAD is installed, **When** a project is created naming a BMAD workflow, **Then** the project exists, its manifest records the workflow, and its scaffold matches BMAD's declared shape — not spec-kit's.
2. **Given** no workflow is named, **When** a project is created, **Then** the configured default is used, and the default is the one from Settings — not a hardcoded fallback.
3. **Given** a workflow name that no installed method provides, **When** creation is attempted, **Then** it is REFUSED naming the unknown workflow and listing what is available — never created under a silent fallback.

### User Story 2 - Rename and delete, with deletion confirmed (Priority: P1)

**Why this priority**: a create-only lifecycle strands every mistake permanently.

**Acceptance Scenarios**:

1. **Given** a project, **When** it is renamed, **Then** its directory and its manifest label move together and its features are still discovered.
2. **Given** a project with content, **When** deletion is requested, **Then** an explicit confirmation naming the project AND its unit count is required first.
3. **Given** deletion is confirmed, **When** it runs, **Then** the removal lands on the active feature branch like any other spec write.

### User Story 3 - The same operations from the tree (Priority: P1)

Right-click a store → **New project**. Right-click a project → **Rename**, **Delete**.

**Why this priority**: the tools without the menus leaves the UI the only place a user cannot do this, which is where they will look first.

**Acceptance Scenarios**:

1. **Given** a writable MARKETPLACE store, **When** the user right-clicks it, **Then** "New project" appears, offering the available workflows.
1a. **Given** a writable store-scoped store (`user-specs`, an arbitrary repo), **When** the user right-clicks it, **Then** "New project" appears WITHOUT a workflow choice — that store binds one workflow for everything in it.
2. **Given** `bos-system-specs` (read-only), **When** the user right-clicks it, **Then** no mutating action is offered.
3. **Given** any of these actions, **When** it runs, **Then** it executes **the same code path** the agent tools use — not a parallel implementation.

### User Story 4 - A method declares what its projects look like (Priority: P2)

A pack says what a project IS for its workflows — the marker manifest, directories to create, templates to write — and BOS scaffolds accordingly.

**Why this priority**: without it every method gets spec-kit's shape, which is the bug this feature exists to fix. P2 only because a sensible default keeps the feature working for packs that declare nothing.

**Acceptance Scenarios**:

1. **Given** a method declaring a project scaffold, **When** a project is created under it, **Then** exactly the declared directories and files exist.
2. **Given** a method declaring none, **When** a project is created, **Then** a manifest-only project is created and nothing fails.

### Edge Cases

- **Two packs declaring the same workflow name.** Resolved the way `045` FR-001a resolves duplicate agent ids: REPORTED, never decided by install order. A bare name works when unambiguous; `<method>:<workflow>` disambiguates.
- **Deleting a project that is the only one in its store.** Allowed — an empty store is a valid state, and refusing would strand a store created by mistake.
- **Renaming a project whose id is referenced elsewhere** (a draft branch, an open editor tab). The rename must not silently orphan them.
- **A store whose method is not installed** (`045` FR-016) can carry no lifecycle action at all: BOS cannot say what a project in it even is.
- **A marketplace store's PROJECTS ARE ITS ITEMS.** One git repo, many projects. "New project" on a marketplace store therefore creates a new ITEM — which is the same flow as "create a new app", and is the answer to the question that started this feature.
- **`bos-system-specs` is read-only**, so it carries no lifecycle action at all.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST expose three project-lifecycle tools — `create_project`, `rename_project`, `delete_project` — available to the assistant like any other tool.

- **FR-002**: `create_project(name, workflow?)` MUST accept a **workflow** name resolved across ALL installed methods. A bare method id MUST remain valid and mean "that method's default workflow", so every binding that works today keeps working.

- **FR-002a**: The lifecycle tools MUST dispatch on the target store's KIND (FR-011), because "a project" is a different object in each:

  | Kind | What a project IS | `create_project` | `rename` / `delete` |
  |---|---|---|---|
  | Marketplace store | an **item** | a new item | that item |
  | BOS user specs | a **folder / module** (`037`) | a new folder, e.g. "Network Filesystem support" | that folder |
  | Arbitrary repo | a **folder / module** | a new folder | that folder |
  | BOS system specs | — read-only | refused | refused |

  ONE tool, three behaviours. A user asking for a project should not have to know which kind of repository they are pointing at.

- **FR-002b**: **"One project" describes BINDING SCOPE, not the lifecycle.** `user-specs` and an arbitrary repo each bind ONE workflow for everything in them (FR-011) — and still contain folders, which is what these tools create, rename and delete. The two are independent: how many things can be bound is not the same question as how the content is organised. Conflating them misleads in both directions: either every store gets folders, or none does.

- **FR-002c**: The `workflow` argument MUST be accepted only where binding scope is per-project — i.e. on a marketplace store. Passing one to a store-scoped kind MUST be REFUSED, naming the store's own binding as the place to change it. Silently ignoring it would record an intent the product does not honour, and silently applying it would make one folder disagree with the store it lives in.

- **FR-003**: Every method MUST expose at least one workflow, so a workflow name always exists to bind. A method that declares none has exactly one, named for the method itself.

- **FR-004**: An ambiguous workflow name MUST be REPORTED, never resolved by install order — same rule as `045` FR-001a for agent ids. `<method>:<workflow>` MUST always be accepted as an unambiguous form.

- **FR-005**: An unknown workflow MUST refuse creation, naming the unknown value and listing the available ones. It MUST NOT fall back to a default: a project silently created under the wrong framework is the failure this feature exists to remove.

- **FR-006**: `delete_project` MUST require an explicit confirmation that names the project AND how many units it contains. A count is what distinguishes "delete the empty folder I just made" from "delete 30 specs".

- **FR-007**: The descriptor MAY declare a **project scaffold** — a marker manifest, directories to create, and files to write from the pack's templates. It MUST be DATA, not code. Packs stay pure data (`045`'s premise, and FR-013a gates code from a marketplace behind a per-origin opt-in); nothing about creating a folder needs an executable handler.

- **FR-008**: A method declaring no scaffold MUST still work, producing a manifest-only project.

- **FR-009**: `project.json` MUST gain a `workflow` field. The existing `method` field MUST continue to resolve (as "that method's default workflow"), and MUST NOT be dropped when BOS rewrites the manifest — `045` FR-019's preservation rule applies to our own migration. Resolution order: `workflow` → `method` → store → global default.

- **FR-010**: Build Studio MUST offer **New project** on a writable store's context menu, and **Rename** / **Delete** on a project's. These MUST call the same implementation as FR-001's tools. A second implementation is how the two drift, and the UI is where the divergence would be found last.

- **FR-011**: **Binding scope is a property of the repository KIND, not a universal rule.** BOS MUST distinguish four kinds and apply different scope to each:

  | Kind | What a Project is | Bind at |
  |---|---|---|
  | Marketplace store | **each item** — one repo, many projects | each item |
  | BOS system specs | — (read-only) | nowhere |
  | BOS user specs | the store IS one project | the store |
  | Arbitrary repo | the repo IS one project | the store |

  A single rule cannot be right for all four: a marketplace repo holds many independent products, while `user-specs` holds refinements to ONE product (BOS) and so has one pipeline for all of it.

- **FR-011a**: `Project` KEEPS its `037` role inside directory-scanned stores — organizational grouping and feature-numbering scope. Only *binding* moves to the store for `user-specs`; the existing folders stay exactly as they are. No migration is required because no project-level binding exists on any of them today.

- **FR-011b**: A per-Project binding present in a store whose kind binds at store level MUST be **REPORTED**, not silently honoured and not silently ignored. Both silent options leave the user with a binding they cannot see and cannot act on.

- **FR-011c**: No mutating action may be offered for a read-only store, or for a store whose method is not installed (`045` FR-016) — BOS cannot say what a project in it even is.

- **FR-012**: `createItemSpec` MUST resolve the primary artifact name through the FULL binding chain including the configured default, fixing the empty-binding call that makes Settings → Default method inert. A test MUST create an item with a non-spec-kit default and assert the artifact's NAME.

- **FR-013**: Every lifecycle write MUST land on the active feature branch, like any other spec write.

- **FR-014**: Documentation MUST cover creating a project with a chosen workflow, since the current answer — hand-write `project.json` — is not discoverable.

### Non-requirements

- **Authoring or editing workflows.** `051` owns that. This feature only SELECTS one.
- **Creating stores.** `050` owns that. A project lives in an existing store.
- **Migrating existing projects** to a different workflow. `045` FR-010's preflight already covers changing a binding.

### Key Entities

- **Workflow** — a named pipeline provided by a method. The unit a project binds to.
- **Project scaffold** — a method's declaration of what a project's directory contains.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A project can be created, renamed and deleted from the assistant AND from the tree, with no hand-edited JSON at any point.
- **SC-002**: Creating a project under a non-default workflow produces that workflow's scaffold and artifact names — verified by file name, not by the recorded binding.
- **SC-003**: An unknown workflow refuses and lists the available ones; an ambiguous one reports the conflict. Neither silently falls back.
- **SC-004**: Deleting a project requires a confirmation naming it and its unit count.
- **SC-005**: The tree actions and the tools are the same implementation — demonstrated by there being one, not by both being tested.
- **SC-006**: With the default method set to something other than spec-kit, a newly created item's primary artifact carries THAT method's leaf marker.
- **SC-007**: Every binding that works before this feature still resolves after it, `method`-keyed manifests included.
- **SC-007a**: Each of the four repository kinds offers binding at its declared scope and nowhere else; a per-Project binding in a store-scoped kind is reported.
- **SC-007b**: "Create a document-processing app using the bmad-enterprise workflow" succeeds as ONE call against the marketplace store, producing an item whose primary artifact carries that workflow's leaf marker.
- **SC-008**: `npm run test:unit` green; `npx tsc --noEmit` and `npm run lint` clean.

## Assumptions

- **Workflow is the durable unit; method is the coarse one.** Binding by workflow now costs little and saves `051` from reworking a signature it did not choose.
- **Scaffolding is data.** If a pack ever needs behaviour data cannot express, add a hook then, with the evidence in hand — not speculatively, and not at the cost of packs shipping code.
- **The UI and the tools are one implementation.** Every divergence in this codebase so far has come from two paths to the same outcome.
