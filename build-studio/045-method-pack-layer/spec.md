# Feature Specification: The method-pack layer — make the spec framework swappable

**Feature Branch**: `045-method-pack-layer`

**Created**: 2026-09-10

**Status**: Draft

**App Target**: bos-core

**Input**: "BOS currently uses GitHub spec-kit as the driver of the development pipeline. Other frameworks such as OpenSpec and BMAD exist. It would be nice if these frameworks could be plugged into BOS as some kind of marketplace item. Analyze BOS and figure out what needs to be done to make these frameworks swappable."

> Depends on `assistant/044-skill-agent-hygiene` for its agent-relocation step only; the descriptor and discovery work is concurrent with it. Blocks `046-spec-kit-pack`, `047-openspec-pack`, `048-bmad-pack`. Extends `018-external-spec-store` (whose self-describing store manifest gains a `method` field) and `037-project-layer` (whose `project.json` gains an optional override). Does not change `020-branch-coupled-specs`' promote mechanics.

## Why this exists (context)

Spec-kit is not *integrated* into BOS — its vocabulary is **inlined at four independent layers**, only one of which is data:

| Layer | Where | Coupling |
|---|---|---|
| Model / derivation | `src/lib/specs/types.ts`, `pipeline.ts` | Phase ids are a TypeScript union; artifact names, leaf marker, numbering and state rules are string literals |
| Engine assets | `.specify/templates`, mounted at `/Templates` | One global path, one global mount behind a one-shot boot latch |
| UI / API | `src/apps/build-studio/index.tsx`, `/api/specs` | Phase order and labels duplicated client-side |
| Driver prompts | `seed/skills/build-studio/**` | Handled by `044` and `046` |

This feature makes layers 1–3 descriptor-driven with **zero behavioural change**.

It is worth doing even if no second framework ever ships: it collapses **four** independent copies of the "a directory is a feature iff it directly contains `spec.md`" rule (`pipeline.ts:202`, `:215`, `:291`, `projects.ts:85`) into one, and removes a compile-time phase union that makes the pipeline unextendable by construction.

### What is already right — do not rebuild

The store layer (`018`/`037`) is already framework-agnostic. A store is any git repo with a `spec-store.json`, discovered by listing a container root with no central registry (`stores.ts:85`), ranked system → user → marketplace → item (`:104-105`). That ranking is the template for FR-001's agent-root precedence.

### Shape decisions forced by real frameworks, not chosen

- **`sections` is a list, not a single leaf marker.** OpenSpec has two parallel trees with distinct semantics (`specs/` = current truth, `changes/` = proposals) plus an archive. A single-marker abstraction looks sufficient and fails on the first real pack.
- **Phases form a DAG, not a sequence.** OpenSpec's schema declares `requires: [proposal]` and states outright: *"Dependencies are enablers, not gates. They show what's possible to create, not what you must create next."* Today's linear `PHASE_ORDER` cannot express that.
- **`generates` is a glob, not a filename.** OpenSpec has `generates: specs/**/*.md`; BMAD shards a PRD into N story files. `ARTIFACT_FILES` as a fixed list (`types.ts:87-94`) expresses neither.
- **Roles are BOS-owned; attachment is pack-declared.** Spec-kit and OpenSpec define no agents at all — `architect`, `architect-reviewer` and `ui-designer` are BOS's own opinion. BMAD brings its own cast. So a pack declares which of its artifacts each role attaches to, and packs may ship their own personas.

Aligning `{id, generates, requires}` with OpenSpec's own schema vocabulary means `047` reads as a translation rather than a hand-built mapping.

## Clarifications

### Session 2026-09-12

- Q: Where do method packs come from — how are the OpenSpec and BMAD packs distributed and installed? → A: spec-kit ships in-tree; OpenSpec and BMAD are items in the user's own private `user-apps` marketplace. Public publishing is a separate later decision.
- Q: How does a pack declare compatibility with BOS's descriptor format, which will evolve? → A: The descriptor carries a `schemaVersion`; BOS refuses to register a pack whose schema version it does not support, naming the mismatch. No silent defaulting.

- Q: What is the trust policy for a method pack that carries executable code (a `plugin/` facet)? → A: Warning plus an origin gate: packs with a `plugin/` facet install freely from the user's own `user-apps`; from any other marketplace they require an explicit per-pack opt-in.

- Q: Are method packs per-user or deployment-wide under Bastion? → A: Per-user, implicitly — packs install into each user's own `data/`; a store's method travels in its version-controlled `spec-store.json`, so users never disagree about a store, they can only lack the pack it asks for.


## User Scenarios & Testing *(mandatory)*

### User Story 1 - Nothing changes for existing specs (Priority: P1)

Every feature in `bos-system-specs` and `user-specs` renders, derives phase state, and behaves exactly as it did before this feature.

**Why this priority**: this is the acceptance test for the whole refactor. `pipeline.ts` is 522 lines of subtle branch/draft handling, and the draft-branch overlay (`listSpecifications`, `specTree`, `insertAtPath`) is the least obvious part and the easiest to regress.

**Independent Test**: Snapshot every `Specification` produced by both live stores — including a live `bos/*` branch so the draft overlay is covered — before starting; assert **field-level parity** after, per SC-001 (identical everything, plus identical `{id, state}` phase pairs; `label` additive, `blocked` only where justified).

**Acceptance Scenarios**:

1. **Given** the pre-refactor snapshot, **When** the descriptor-driven pipeline runs, **Then** every `Specification` matches per SC-001's parity definition, with every `na` → `blocked` substitution individually enumerated.
2. **Given** a feature that exists only on a draft `bos/*` branch, **When** the tree is built, **Then** it is grafted at the same nesting with the same branch badge as before.

### User Story 2 - Spec-kit is expressed as a descriptor, not as code (Priority: P1)

Spec-kit's nine phases, six artifacts, leaf marker, numbering and constitution path are all data, evaluated by the same code path a pack uses.

**Why this priority**: without this, the layer is unvalidated and there are two code paths — built-in and installed — of which only the built-in one is exercised by BOS's own specs.

**Independent Test**: `grep` for any `if (builtin)`-style branch in the pipeline; there must be none.

**Acceptance Scenarios**:

1. **Given** the spec-kit descriptor, **When** phase state is derived, **Then** all nine phases resolve through the declarative rule evaluator.
2. **Given** a phase whose rule cannot be satisfied, **When** it is evaluated, **Then** it reports `blocked` rather than being silently bucketed.

### User Story 3 - A user assigns a method to a store (Priority: P1)

A user picks which framework governs a store, from the Build Studio store-group header, and the choice survives a restart.

**Why this priority**: the binding is the feature's whole user-facing surface.

**Independent Test**: Set `method` on `user-specs` on a feature branch, restart the server, confirm it persists.

**Acceptance Scenarios**:

1. **Given** a writable store on an active `bos/*` branch, **When** the user selects a method, **Then** `spec-store.json` is written on that branch and rides the normal promote.
2. **Given** the read-only `bos-system-specs`, **When** its group header renders, **Then** no method dropdown appears.
3. **Given** a server restart, **When** the seed reconciles manifests, **Then** the chosen `method` is preserved, not overwritten.

### User Story 4 - Existing specs are never silently hidden (Priority: P1)

Assigning a method whose layout doesn't match a store's existing content warns before doing it, and the recommended path is not to do it at all.

**Why this priority**: this is the single most dangerous operation the feature introduces. Flipping `user-specs` from spec-kit to OpenSpec leaves every feature as `NNN-slug/spec.md`, which OpenSpec's sections do not match. The features are not damaged — they **vanish from the tree**, which reads as data loss.

**Independent Test**: Attempt the flip on a store with existing spec-kit features; assert a preflight blocks it with a count.

**Acceptance Scenarios**:

1. **Given** a store with 41 spec-kit features, **When** the user selects OpenSpec, **Then** a preflight reports how many features would become undiscoverable and requires explicit confirmation.
2. **Given** the user confirms anyway, **When** the tree renders, **Then** the store is visibly marked as containing content the active method cannot see.

### User Story 5 - A pack's agents are discoverable without cluttering the picker (Priority: P2)

Agents contributed by a pack are delegatable by that pack's driver, and only those meant for direct conversation appear in the Assistant's agent picker.

**Why this priority**: BMAD contributes ~8 (BMM) + 5 (CIS) + 1 (BMB) agents. Three extras in the picker is fine; fourteen per pack is not. But scoping must not break delegation — if pack agents are absent from the registry, the pack's own driver cannot reach them.

**Independent Test**: Register a `delegate-only` root; assert absent from the picker and reachable by `agent_delegate`.

**Acceptance Scenarios**:

1. **Given** a `delegate-only` agent root, **When** the picker renders, **Then** its agents are hidden.
2. **Given** the same root, **When** a driver calls `agent_delegate`, **Then** the agent resolves and runs.
3. **Given** an agent id present in both a pack root and `data/agents/`, **When** it is resolved, **Then** the `data/agents/` copy wins by precedence.
4. **Given** several roots contribute agents, **When** the picker renders, **Then** entries are grouped by source so similarly-named agents from different packs are distinguishable.

### Edge Cases

- **Item-owned stores have no `spec-store.json`.** They are synthesised (`item-stores.ts:78-90`). They MAY optionally contain one in their `spec/` folder; absent, they inherit the global default. For an item store the method governs artifacts and phases only — layout is already degenerate (no Project, no `NNN-`, the whole store is one spec).
- **`bos-system-specs` is read-only and permanently spec-kit-shaped.** Its method is implicit and unchangeable, which is why spec-kit remains the default forever.
- **A pack may be uninstalled while a store still references it.** The store must render with a clear "method not installed" state, never silently fall back to spec-kit and reinterpret content.
- **`/Templates` is referenced by existing prompts.** Keep it as an alias to the active store's pack for one release so prompts don't break mid-migration.
- **The `seed.ts` manifest rewrite** (`:89`, `:111`) currently overwrites `spec-store.json` from a constant on every boot. Without `044` FR-equivalent behaviour or a fix here, the `method` field is clobbered every restart.

## Implementation stages

This feature spans eight subsystems — the pipeline, agent discovery, VFS mounts, the Build Studio UI, `/api/specs`, the marketplace schema, the config registry and self-heal. That is too much to review or revert as one unit, so it MUST land as three stages, each with an independent revert point and its own green test run. (Same staging discipline `037-project-layer` uses for its user stories.)

| Stage | Requirements | Independent revert point | Blast radius |
|---|---|---|---|
| **A — Descriptor & pipeline** | FR-003, FR-003a, FR-004, FR-004a, FR-005, FR-006, FR-006a, FR-007, **FR-017a** | Spec-kit descriptor drives the existing pipeline; UI renders server-supplied phases. No packs, no binding. | `src/lib/specs/`, Build Studio's phase strip |
| **B — Discovery & binding** | FR-001, FR-001a, FR-001b, FR-002, FR-008, FR-008a, FR-009, FR-009a, FR-010, FR-010a, FR-010b, FR-010c, FR-011, **FR-014** | Multi-root agents, method assignment, per-pack mounts. Still no installable packs. | `subagents/store.ts`, `spec-mount.ts`, config, tree UI |
| **C — Install path** | FR-012, FR-012a, FR-013, FR-013a, FR-015, FR-016 | The `method` facet, Marketplace surfacing, uninstall transition. | `system/items/`, `marketplace/schema.ts`, `apps/marketplace/`, `/api/specs` |

**FR-017a (the e2e fixture-pack harness) belongs to Stage A**, before any e2e is written against it — it is a prerequisite for FR-017 here and for `046` FR-015, `047` FR-013 and `048` FR-013. FR-017's own coverage then accompanies whichever stage introduces the surface it tests, not a separate pass at the end.

**Stage B's interim template mapping.** FR-011 introduces `/Methods/<packId>/templates` in Stage B, but the `method` facet arrives in Stage C and `046` is what physically relocates the template files. So for the duration of Stage B there is exactly one pack — the built-in spec-kit descriptor — and `/Methods/spec-kit/templates` MUST resolve to `.specify/templates` in BOS source. That is a deliberate interim state, not an oversight: the mount indirection lands before the content moves, so `046` changes only a path and not a mechanism. The `/Templates` alias (FR-011) covers existing prompts throughout.

Stage A alone satisfies SC-001 through SC-004 and is worth shipping on its own merits. `046` depends on all three.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `agentsDir()` MUST become a root list with per-root `visibility: "picker" | "delegate-only"`, defaulting to `picker`. Call sites in `src/lib/agent/subagents/store.ts` include `:107`, `:291`, `:323`, `:346`, `:393` (24 in total, all in that one file; zero elsewhere in `src/`). Registration and delegability MUST be unconditional; only *discovery* is scoped. A Settings escape hatch MUST allow revealing `delegate-only` agents for debugging. `kind: "provider"` roots are out of scope (`048`).

- **FR-001a**: Root precedence MUST be **explicitly ordered**, highest wins: (1) `data/agents/` — the user's own copies and overrides; (2) pack roots, in pack-registration order, which MUST be deterministic (sorted by pack id, not install order); (3) `seed/agents/` — BOS capability agents. `listStores()` is the structural template (a ranked merge of independently discovered roots) but **not** the ordering itself: its rank is by store *owner*, which has no agent analogue. Two packs contributing the same agent id MUST be reported as a collision, not silently resolved by ordering.

- **FR-001b**: Seed reconciliation MUST target exactly one root. `applySeedAgent` (`subagents/store.ts:53`) and the `.seed-stamp` three-way contract assume a single writable location; once roots are a list, the seed MUST continue to write only to `data/agents/`, and MUST NOT write into a pack root under any circumstance. Pack roots are read-only to the seed.

- **FR-002**: The agent picker MUST group entries by contributing root.

- **FR-003**: A new `src/lib/specs/method/` module MUST provide descriptor types, a registry, and a declarative rule evaluator. The descriptor MUST carry: `schemaVersion`, `id`, `label`, `version`, `builtin?`, `sections[]` (`rel`, `kind`, `leafMarker`, `numbering`, `terminal?`), `constitution`, `constitutionRoot`, `discrepancies`, `testFile?`, `artifacts[]` (`{id, generates, requires, scope}` with glob `generates`), `artifactOrder[]`, `phases[]` (`{id, label, requires, rules[], else}`), `stateLabels`, `templates`, `agents[]`, `roles{}`, `modules[]?`.

- **FR-003a**: `schemaVersion` MUST declare which version of BOS's **descriptor format** the pack targets — distinct from `version`, which is the pack's own release. BOS MUST refuse to register a pack whose `schemaVersion` it does not support and MUST name the mismatch (expected vs. found). It MUST NOT default a missing or unrecognised field and continue.

  > Silent tolerance is wrong here specifically because the descriptor decides **what exists**. `readManifest` already tolerates unknown fields in `spec-store.json` and that is correct for a four-field manifest — but defaulting a missing `sections` would render a store full of specs as empty, the same data-loss shape FR-010 exists to prevent. Fail loudly at registration, matching the existing precedent that an unresolvable group id is surfaced as an error rather than bucketed.

- **FR-004**: The rule DSL MUST support `exists`, `contains`, `notContains`, `checklist`, `containsUnitId`, `dependsOn`, `manual` — sufficient to express all nine current spec-kit phases including the `"## Clarifications"` check, the `"[PROJECT_NAME]"` placeholder test, `tasks.md` checklist progress, the `discrepancies.md` feature-id lookup, and `test-results.md`'s `"**Status**: PASSED"`. No pack MUST be able to execute code in the pipeline path.

- **FR-004a**: The DSL MUST additionally support a **set predicate** over a glob of unknown cardinality: `{ glob, quantifier: "any" | "all" | "none", of: <predicate> }`, plus `count` for progress reporting. Without it, phase state for **dynamically generated artifacts** is inexpressible — BMAD (`048`) shards a PRD into N story files (1.1, 1.2, 2.1…) and needs "all stories complete", which `checklist` cannot answer because it operates on one known file. The seven predicates above were verified against spec-kit's nine phases only; that verification says nothing about dynamic-artifact frameworks, and discovering the gap during `048` would mean reopening a frozen layer. Build it here.

- **FR-005**: `PhaseId` (`types.ts:4-13`) MUST become `string`; `PipelinePhase` MUST carry `label`; phase state MUST include `blocked` distinct from `na`.

- **FR-006**: `ARTIFACT_FILES` (`types.ts:87-94`), `CONSTITUTION_REL` (`pipeline.ts:36`) and `derivePhases` (`:81-126`) MUST become descriptor-driven. The four leaf-rule copies MUST collapse to one predicate. `nextFeatureId` (`:496-522`) MUST honour the section's `numbering`, skipping `NNN-` allocation entirely for `"none"`.

- **FR-006a**: The **converge signal is currently hardcoded cross-store** and MUST be resolved explicitly. `pipeline.ts:94-104` reads the system store's `discrepancies.md` *and* `user-specs/discrepancies.md` **by literal store id**, so a marketplace-cloned store's converge phase today consults BOS's own discrepancy file. The descriptor MUST declare where a store's drift record lives (a `discrepancies` path, defaulting to the store's own root). If the cross-store read is retained for `bos-system-specs` — whose read-only status is the reason the fallback exists — that MUST be stated as an explicit exception rather than left implicit in the code.

- **FR-006b**: The **constitution read is cross-store in exactly the same way** and MUST be handled identically. `hasConstitution()` (`pipeline.ts:72-79`) resolves through `systemStoreId()` (`:74`) for every feature in every store, and `constitutionReady` (`:71`) memoizes one answer module-globally. The descriptor MUST declare `constitutionRoot` (`"system"` | `"store"`); spec-kit declares `"system"`. Resolving per-store instead gives `done → pending` on the 8 live item stores, which SC-001 does not sanction in any form.

- **FR-007**: `PHASE_ORDER` and `PHASE_LABEL` (`src/apps/build-studio/index.tsx:66-88`) MUST be deleted; `PhaseStrip` MUST render server-supplied `{id, label, state}`.

- **FR-008**: A store MUST resolve its method through the binding chain `project → store → global default → spec-kit`, where **the levels that apply depend on the repository kind** (`049` FR-011): a marketplace item binds individually, while `user-specs` and an arbitrary repo bind once for the whole store. A binding written at a level that does not apply MUST be read and REPORTED, never silently honoured (which would make one unit disagree with the store it lives in) and never silently dropped (which leaves a binding the user can neither see nor act on).
- **FR-008a**: Method packs are **per-user**. They install into `data/`, which under Bastion is a per-user volume in a per-user container, so FR-008's global default is global *to one user's BOS*, not to the deployment — the wording MUST say so. A store's method lives in `spec-store.json`, which is version-controlled store content, so a shared store carries one agreed method that travels with the repo: two users can never disagree about a store's method, only differ in whether they have the pack it names. That case is already FR-016's "method not installed" state and needs no separate mechanism.

- **FR-009**: A **Method** dropdown MUST appear on the Build Studio store-group header for writable stores, writing on the active `bos/*` branch. It MUST NOT render for `bos-system-specs`.

- **FR-009a**: Assignment owns **constitution continuity**, in one place. A store's constitution path is descriptor-declared (FR-003), so changing method changes where BOS looks — spec-kit uses `.specify/memory/constitution.md`, OpenSpec uses `project.md`. On assignment, BOS MUST either relocate the existing constitution to the new descriptor's path or report it as now-unreferenced, and MUST surface when the new path has no constitution at all. This lives here, not in `046` or `047`: both packs otherwise specify assignment-time constitution behaviour in different terms, and two owners of one moment will diverge.

- **FR-010**: Assignment MUST run a preflight reporting how many existing features would become undiscoverable under the new method, requiring explicit confirmation. Documentation MUST state that the recommended adoption path is a new store or new Project, not flipping an existing one.

- **FR-010a**: The preflight MUST compute that count by running discovery **twice** — once under the effective descriptor and once under the candidate — and reporting the set difference by feature path. It MUST include draft-branch-only features (`listDraftBranches` overlay), since those are the ones a user is most likely to be actively working on and least likely to notice disappearing. The report MUST list the affected paths, not only a count.

- **FR-010b**: Section kinds MUST be renderable as visually distinct groups in the Build Studio tree, reusing the existing owner-badge pattern. This is generic section rendering, not framework-specific: it lives here rather than in `047`, so that pack can remain genuinely code-free.

- **FR-010c**: FR-010c is the **gate**; FR-009a is the **remedy**. On any effective descriptor change these run in that order: the preflight evaluates first and may hold the change entirely; only if the change proceeds does constitution relocation-or-reporting apply. They MUST NOT both independently decide whether a change happens. The preflight MUST run on any **effective descriptor change for a store**, not only on user-initiated assignment. A pack **upgrade** can change `sections`, `leafMarker`, `numbering` or `phases`, silently reinterpreting existing content — the identical data-loss shape FR-010 guards, arriving on a path with no deliberate user action behind it. On an upgrade that would orphan content, the upgrade MUST be held and reported rather than applied silently. Assignment and upgrade MUST share one implementation; two copies of this check will diverge.

- **FR-011**: The single `/Templates` mount (`spec-mount.ts:63`) MUST be replaced by `/Methods/<packId>/templates`, and the one-shot `mounted` latch (`:43`+`:46-47`) MUST be removed so mounts change on install/uninstall. The dead second copy — `TEMPLATES_ROOT`, `readTemplate`, `listTemplates` (`src/lib/dev/spec-fs.ts:410`, `:421-431`, no callers) — MUST be deleted.

- **FR-012**: `ItemFacets` and `readFacets` (`src/system/items/installed.ts:99-111`) and `src/lib/marketplace/schema.ts` MUST gain a `method` facet. Install MUST register the descriptor, agent roots and template mount; uninstall MUST unregister them. Nothing MUST be copied (035).

- **FR-012a**: A method pack MUST be installable and uninstallable from the Marketplace app. Three gates refuse it today and all three MUST admit a `method` facet: `src/lib/marketplace/client.ts:880-881` throws `has nothing installable` server-side; `src/apps/marketplace/index.tsx:383` (`hasInstallableFacet`) renders no Install button; `:173` issues no install op. `:169-187`'s `installedKinds` SHOULD gain `"method"` for the post-install notice. There is **no** per-kind list or facet filter to extend — the sidebar (`:262-284`) groups by source repo and per-source sections already list every item regardless of facet.

- **FR-013**: The Marketplace UI MUST warn at install time when a pack carries a `plugin/` facet — installing "a spec framework" reads far lighter to a user than installing server code.

- **FR-013a**: A pack carrying a `plugin/` facet MUST additionally pass an **origin gate**. Sourced from the user's own `user-apps` marketplace it installs freely (it is the user's own content, per `035`'s provenance derivation). Sourced from any other marketplace it MUST require an explicit, per-pack opt-in recorded against that pack — not a global setting, and not a dismissible warning.

  > A warning the user clicks through once is a speed bump, not a control, and it decays into noise. A *global* "allow server code" switch is worse: flipped once to install one pack, it stays on for every pack thereafter, so it protects nothing while adding a setting. Origin is nearly free to gate on here because Q1 put these packs in `user-apps` — the gate costs nothing today and bites exactly when a third-party marketplace appears, which is when it should.

- **FR-014**: `GET /api/specs` MUST return the active descriptor alongside `tree` and `specs`.

- **FR-015**: The hardwired `spec/spec.md` in `create.ts:26`, `:123` and `item-stores.ts:67`/`:80` MUST resolve to the descriptor's first artifact.

- **FR-016**: Uninstalling a pack MUST have a defined transition for in-flight work. A pack contributes agent roots as well as a descriptor, so uninstalling it removes the `architect` (or Sally) that a conversation is mid-feature with. Stores referencing the missing method MUST render a "method not installed" state (never silently fall back to spec-kit and reinterpret content), delegations to a now-absent agent MUST fail with a message naming the uninstalled pack, and uninstall MUST warn when any store currently references the pack.

- **FR-017**: This feature MUST carry e2e coverage for the surfaces it changes: the phase strip rendering server-supplied phases including `blocked`, the Method dropdown, the FR-010 preflight, and FR-010b's section badges. Existing coverage of this surface (`e2e/013-artifact-reload.spec.ts`, `e2e/013-spec-anchor.spec.ts`) MUST be updated rather than left to rot. Per BOS's testing rule, any new test MUST bundle its own fixtures — written through the API at test time — and MUST NOT depend on pre-seeded spec-store content.

- **FR-017a**: **The e2e harness cannot currently install a method pack at test time, and that capability is a prerequisite for FR-017 and for `046`/`047`/`048`'s e2e requirements.** Evidence: `e2e/build-studio.spec.ts:49-56` comments that "whether any marketplace item is installed depends on the environment" and calls `test.skip(true, "no marketplace item installed in this environment")` — a test that silently passes by skipping, and precisely the pre-seeded-data dependency BOS's testing rule prohibits. This feature MUST add the ability to install a **fixture pack** — a minimal local `user-apps` item with a `method/` facet, requiring no network — through the API at test time, and SHOULD convert that existing self-skipping test to use it. Without this, four downstream specs carry e2e criteria that can only be satisfied by skipping.

- **FR-018**: Documentation MUST be updated as part of this feature, per BOS's standing rule that `docs/` tracks feature changes. `docs/dev/build-studio.md` describes the spec subsystem in detail — the `PHASE_ORDER` strip, the single `/Templates` mount, the spec-kit engine's location in source, the agents seeded from `seed/` — and is substantially wrong after Stage A alone. `docs/dev/architecture-overview.md` §9 (Specs System) likewise. Both MUST describe the descriptor, sections, the DAG, and multi-root agent discovery. A new `docs/dev/method-packs.md` SHOULD carry the pack-authoring reference.

### Durability and reachability

> These govern the seams where this layer meets a long-lived server, a user's
> git repositories and a restart. Each is stated because the obvious
> implementation is wrong in a way that fails silently.

- **FR-019**: **Every manifest reader MUST preserve keys it does not model.** BOS does not merely *read* `spec-store.json`, `project.json`, `marketplace.json` and a pack's `method.json` — it **rewrites** them. A reader that reconstructs a fixed object from named fields silently drops the rest, and the next write **deletes them from disk and commits the deletion**. Four readers in this subsystem have this shape (`readManifest`, `readProjectManifest`, the agent-route field whitelist, `parseManifest`), and each one can destroy a pack's declaration in the user's own repo. Readers MUST spread-then-narrow, or explicitly round-trip unknown keys, and every writer MUST merge into the parsed original rather than emit a fresh object. A test MUST write an unmodelled key, round-trip it through a BOS-initiated rewrite, and assert it survives.

- **FR-020**: **A binding MUST survive a restart, and the seed MUST NOT rewrite it.** A seed that normalises `spec-store.json` from a fixed template on every boot makes a method assigned through the picker last exactly until the next start. Seeding MUST preserve an existing store's binding. SC-018 covers it, and the check MUST cross a real restart boundary rather than asserting the write in the same process that made it.

- **FR-021**: **Descriptor resolution MUST NOT be cached beyond one call into the pipeline.** The per-store descriptor memo is a module global in a long-lived server: a store rebound to another method keeps resolving through the OLD descriptor in every entry point that does not clear the memo. A store rebound to a `numbering: "none"` method otherwise keeps allocating `NNN-` prefixes from the cached descriptor. EVERY public entry point MUST reset it first, and that invariant MUST be written where the cache is declared, because a new entry point that forgets reintroduces the bug invisibly.

- **FR-022**: **All three levels of the binding chain MUST be settable, not merely resolvable.** A resolver that honours every level while only one has a write path leaves the others unusable, and its unit tests pass throughout. Specifically: an **item-owned store** MUST be able to carry a binding (it is synthesised and has no manifest, so the binding lives in the item's own `spec/spec-store.json`, travelling with the item when it is published); a **Project** MUST have an API write path to its `project.json`; and both MUST be reachable from the UI. A requirement that something *resolves* MUST be paired with one that it can be *set*.

- **FR-023**: **The UI MUST render each store's OWN method, resolved server-side.** A single app-level "active method" rendered as every store's dropdown value makes any store bound to something else display a binding it does not have, so the control disagrees with the binding BOS would apply. Each tree node MUST carry its own resolved method id AND whether the binding is declared at that node or inherited, so an inherited value is never presented as a decision made there. A group node MUST agree with its own children.

- **FR-024**: **Preflight MUST be scoped to the level being bound, and MUST resolve `from` through the whole chain.** An unscoped report lists the whole store's units as orphaned; because preflight is the GATE, that forces the user through an "override and hide N units" path to do something that hides nothing. Scoping MUST match the binding level. Separately, the "from" descriptor MUST be resolved including the global default — omitting it reported every change as starting from spec-kit whenever the user's default was something else. A constitution belongs to a STORE, so a Project-scoped change MUST NOT reconcile one.

- **FR-025**: **Discovery MUST walk SECTIONS, and `terminal` MUST be honoured.** Discovery that walks BOS Projects only, never consulting `sections`, renders a store empty when its method's sections live at the store root — and disagrees with `preflight`, which walks sections correctly. A `rel: ""` section means "this root is organised into BOS Projects"; a section with a `rel` owns that subtree directly. Section ownership of a path MUST be decided by **longest prefix** in exactly one place (nested sections such as `changes/archive` under `changes` are otherwise discovered twice, once as live work). A unit in a `terminal: true` section MUST render a terminal state instead of being evaluated against the phase DAG — `terminal` was declared in the descriptor from the first draft and read by nothing.

- **FR-026**: **BOS MUST NOT restructure a store whose method owns the store root.** The `033` Project migration renames every top-level directory of a store into a default Project and commits it, on every boot, unless one already owns a `project.json`. For a method declaring sections at the store root that **moves the user's content where the descriptor cannot find it**: the store empties, and the cause is a committed rename that ran before anything rendered. The migration MUST consult the store's method first, reading the binding off disk (resolving it through the pipeline re-enters the seed). A method that is **declared but not registered** MUST also block the migration: not knowing a store's layout is the strongest possible reason not to rearrange it, and refusing costs nothing a later boot cannot do once the pack is installed.


- **FR-027**: **The pipeline MUST supply a glob provider to the rule evaluator.** `set`/`count` exist for artifacts whose cardinality is unknown when the descriptor is written — BMAD's sharded `stories/*.md`, OpenSpec's delta `specs/**/*.md`. Every such glob is NESTED inside a unit, and the evaluator's fallback matches only the unit's TOP-LEVEL artifact names — so without a provider both predicates evaluate to "no files matched" for every pack that uses them, silently. A test that supplies its OWN globber exercises the evaluator rather than the pipeline that feeds it, and cannot detect this. The glob-matching function MUST be shared between provider and fallback so the two cannot disagree about what `**` means, and the acceptance test MUST assert the POSITIVE case (files present ⇒ `done`); asserting only the empty case passes whether the glob works or is structurally unable to match.

- **FR-028**: **Content in a `terminal` section MUST be read-only to a user edit.** `terminal` suppressing phase evaluation is a DISPLAY concern; this is the correctness one. An archived unit is the record of what was decided and other artifacts were merged from it, so editing it rewrites history with nothing to report the change. Store-level editability alone (writable store + a live branch) leaves frozen content fully editable without warning. The refusal MUST apply at every USER-FACING write seam — the API's write, delete and rename, and the spec-writing agent tool, which matters more than the UI because an agent cannot see a disabled control. A rename MUST check BOTH ends: moving into a frozen section is un-reviewed archiving and moving out is un-archiving, each a semantic operation, not a file move. It MUST NOT be enforced inside the spec filesystem: BOS itself retains write access there (seeding writes store roots, and a future archive operation must be able to MOVE a unit into the archive), and a blanket jail would make that unimplementable and be discovered only by whoever tried to build it. `terminal` MUST therefore resolve for every node, not only a section root, so the client can refuse before the user retypes a paragraph.

- **FR-029**: **A method-only marketplace entry MUST survive BOS's manifest reconcile.** The local marketplace manifest is reconciled on every catalog read: declared entries are repaired, an entry with no remaining facet is PRUNED, and the disk scan appends anything it finds that is not already known. `method` MUST count as a facet in that liveness test. It did not, so the first METHOD-ONLY pack was judged to describe nothing, pruned from the user's own git repo, and re-added as a scan-derived stub — losing its name, description, tags and the entire `method` block, and the deletion was COMMITTED. The visible symptom was three steps downstream: the Marketplace offered "Adopt spec" and no Install, because the Install button is gated on `item.method` (FR-012a). The scanner does not emit `method`, so its silence MUST NOT be read as evidence that a declared entry is stale. A test MUST drive the catalog-read path the UI calls and assert the FILE ON DISK, not only the returned value — the reconcile writes and commits, so an in-memory assertion passes while the repo is being rewritten.

- **FR-030**: **Installing a method facet MUST create the item link, and MUST NOT fail silently.** Under `035` the item symlink IS the install. Only the iframe-app branch, the service copy and the plugin path created one, so a METHOD-ONLY pack was never linked — and the method branch was guarded by `if (installed)`, so the missing link skipped registration and the install returned SUCCESS. The user saw a confirmation toast for a pack that was not installed, and the Marketplace correctly refused to mark it installed, which reads as a UI bug rather than a failed install. The method facet MUST link the item itself, and a facet that cannot be installed MUST raise an error naming the item. Separately, a descriptor missing a required key MUST be refused with a message naming the pack and the key: reading `.length` off an absent field produced a bare `TypeError`, which is the opposite of FR-016. Absent and empty are one failure to a pack author.

- **FR-030a**: **A post-install message MUST name a surface that can actually show the thing.** The generic confirmation directed every install to the desktop or to Settings → Plugins → Services. A method pack appears in neither; it appears in Build Studio's Method dropdown. Sending a user to a screen that cannot show what they just installed is indistinguishable from the install having failed.

### Key Entities

- **Method descriptor** — the pack's declaration of sections, artifacts, phases, templates, cast and role attachment.
- **Agent root** — a discovery source for agents, carrying visibility. `dir` kind only in this feature.
- **Section** — one typed tree within a store (`active` / `truth` / `archive`), with its own leaf marker and numbering.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For every feature `listSpecifications()` returns — both directory-scanned stores **and the 8 item-owned stores**, including a live `bos/*` branch so the draft overlay is covered — the pre- and post-refactor `Specification` objects have **identical `id`, `store`, `title`, `path`, `artifacts[]`, `taskProgress`, `branch`, and identical `{id, state}` pairs for every phase, in the same order**. `label` (new in FR-005) is a permitted additive field. **No state substitutions are permitted at all**: spec-kit declares `requires: []` for all nine phases, so `blocked` is unreachable under it and the phase-state output is byte-identical. (An earlier draft sanctioned `na → blocked`; a truth table over all 132 live feature leaves showed the edges that would have caused it also flip 120 `converge` and 114 `test` phases, so the edges were removed instead.)

  > Byte-identity is deliberately **not** the criterion: FR-005 adds `label` and introduces `blocked`, so a byte-identical snapshot is unsatisfiable by construction. Stating it that way would force an implementer to either weaken the test silently or get stuck.
- **SC-002**: Zero `builtin`-special-case branches in the pipeline.
- **SC-003**: `PhaseId` is `string`; no compile-time phase union remains.
- **SC-004**: Exactly one code location decides whether a directory is a feature leaf (was 4).
- **SC-005**: Flipping a store's method to a non-matching descriptor triggers the preflight, which lists the affected feature paths (including draft-branch-only ones) and cannot silently hide content.
- **SC-006**: A `delegate-only` root is absent from the picker and reachable by `agent_delegate`. Two packs contributing the same agent id produce a reported collision, not a silent winner.
- **SC-007**: A phase whose state depends on N dynamically generated artifacts is expressible with the set predicate — verified with a fixture of a glob whose cardinality is unknown at descriptor-authoring time, not merely with spec-kit's fixed nine.
- **SC-008**: The seed writes only to `data/agents/`; no pack root is ever written by reconciliation.
- **SC-009**: Each of stages A, B and C is independently revertible with a green `npm run test:unit`.
- **SC-010**: A pack upgrade that would orphan existing content is held and reported, not applied — exercised by a fixture pack whose v2 descriptor changes `leafMarker`. Assignment and upgrade run the same check, verified by there being one implementation.
- **SC-011**: Uninstalling a pack referenced by a store warns first; afterwards the store renders "method not installed" and a delegation to a removed agent fails naming the pack.
- **SC-012**: Changing a store's method relocates or reports its constitution; no store is left with an unreferenced constitution and no silent absence.
- **SC-013**: e2e covers the phase strip (including `blocked`), the Method dropdown, the preflight and section badges; every new test bundles its own fixtures.
- **SC-014**: `docs/dev/build-studio.md` and `docs/dev/architecture-overview.md` §9 describe the descriptor, sections, the DAG and multi-root discovery — no statement in either contradicts the shipped behaviour.
- **SC-015**: A pack declaring an unsupported `schemaVersion` is refused at registration with a message naming expected vs. found; no store silently renders empty.
- **SC-016**: A `plugin/`-bearing pack from `user-apps` installs without an opt-in; the same pack offered from another marketplace is refused until a per-pack opt-in is recorded.
- **SC-017**: `npm run test:unit` green; `npx tsc --noEmit` and `npm run lint` clean.
- **SC-018**: A method assigned through the picker is still assigned after a server restart, and an unmodelled key written into any manifest BOS rewrites survives that rewrite.
- **SC-019**: Binding a store, an item store, and a single Project each works end to end through the API and the UI; each picker displays that node's own binding, marked inherited where it is inherited.
- **SC-020**: A store bound to a method whose sections live at the store root renders its content, keeps its on-disk layout across a restart, and reports archived units as terminal rather than live.
- **SC-021**: A store rebound to a method with different numbering allocates the next id under the NEW method without a restart.
- **SC-022**: A phase gated on a glob over NESTED files reaches `done` when those files exist — asserted through the pipeline, never with a caller-supplied globber.
- **SC-023**: A unit in a `terminal` section is refused by every user-facing write seam, while active work in the same store is unaffected.
- **SC-024**: A marketplace entry declaring ONLY a method survives a catalog read with its declaration intact, on disk as well as in memory.
- **SC-025**: Installing a method-only pack creates its item link, registers the descriptor, and is reported as installed by the same state the Marketplace reads — or raises an error naming the item. It never reports success having done nothing.

## Assumptions

- **This feature must not ship with zero packs.** An abstraction with one implementation is unvalidated, and every wrong descriptor assumption is one only a second framework exposes. The spec-kit descriptor (FR-003/004/006) is therefore in scope here even though its *assets* relocate in `046`. Build then extract, not build then hope.
- **Spec-kit stays the default forever.** `bos-system-specs` is read-only and spec-kit-shaped, so "switch BOS wholesale to another framework" is not an available outcome; packs are a per-store or per-Project choice.
- **The descriptor stops BOS contradicting a framework; it does not make the framework work.** Roughly 80% of what enforces a pipeline is prose. Each pack remains a real authoring effort.
- Running a framework's own CLI (`openspec validate`, the BMAD installer) is out of scope: there is no execution path rooted at a spec store, and `run_command` is sandboxed to the source worktree. A pack's `service` facet (`039` `deploymentMode: "tools"`) is the eventual answer, deliberately deferred.
