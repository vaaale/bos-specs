# Feature Specification: The spec-kit pack — extract BOS's own method onto the pack path

**Feature Branch**: `046-spec-kit-pack`

**Created**: 2026-09-10

**Status**: Draft

**App Target**: bos-core

**Input**: Third of five features from the swappable-spec-framework analysis. Turns spec-kit — currently BOS's hardcoded default — into a real pack, so the mechanism hosting OpenSpec and BMAD is the same mechanism BOS itself runs on.

> Depends on `assistant/044-skill-agent-hygiene` and `045-method-pack-layer`. Blocks `047-openspec-pack` and `048-bmad-pack`. `App Target` is `bos-core` because the work is a source-to-pack *migration* — it removes assets from `src/`, `.specify/` and `seed/` — even though the destination is pack content.

## Why this exists (context)

`045` leaves spec-kit's descriptor expressed in BOS source as the built-in default. That is correct for proving the refactor, but it leaves the pack *path* untested: the assets spec-kit needs — templates, driver skill, role agents — still live in `src/`, `.specify/` and `seed/`.

Left that way you get **two code paths**, built-in and installed, and only the built-in one is exercised by BOS's own specs. Every bug in the install path then waits for `047` to find it.

### The asymmetry to design in, not discover

`bos-system-specs` is read-only, ships with BOS, and every feature in it is `NNN-slug` + `spec.md`. **Spec-kit can therefore never be absent.** Its pack is non-removable in a way OpenSpec's and BMAD's are not. That is a property to declare, not an emergent surprise the first time someone clicks uninstall.

### A live prompt bug this feature fixes

Eight of spec-kit's vendored command prompts — `.specify/templates/commands/{plan,tasks,analyze,implement,clarify,checklist,converge,taskstoissues}.md` — reference `.specify/scripts/`. Those prompts are exactly what BOS mounts at `/Templates/commands/` and what the `build-studio` skill instructs agents to read before every step.

So agents are currently told to run `.specify/scripts/bash/check-prerequisites.sh`, which they cannot do: `/Templates` is a VFS mount and `run_command`'s shell does not share that filesystem. That is precisely the trap `build-studio/SKILL.md:62` was written to warn about, arriving from BOS's own vendored prompts. BOS already reimplemented what those scripts do — `nextFeatureId` replaces `create-new-feature.sh`, phase derivation replaces `check-prerequisites.sh`.

## Clarifications

### Session 2026-09-12

- Q: Where do method packs come from — how are the OpenSpec and BMAD packs distributed and installed? → A: spec-kit ships in-tree; OpenSpec and BMAD are items in the user's own private `user-apps` marketplace. Public publishing is a separate later decision.
- Q: How does a pack declare compatibility with BOS's descriptor format, which will evolve? → A: The descriptor carries a `schemaVersion`; BOS refuses to register a pack whose schema version it does not support, naming the mismatch. No silent defaulting.

- Q: Is `048` (BMAD) a real commitment, or a validation target that may never be built? → A: All five features are committed and will be built in sequence, `048` included.


## User Scenarios & Testing *(mandatory)*

### User Story 1 - Spec-kit runs as a pack, through the pack path (Priority: P1)

BOS's own specs are governed by a registered pack resolved through the ordinary registry, not by a special-cased built-in descriptor.

**Why this priority**: this is the feature's entire purpose. Until it holds, the install path is unexercised.

**Independent Test**: assert the pack reaches registration through the same call sequence a marketplace pack does. **Not a grep for `builtin`** — FR-001 makes it a legitimate descriptor field, so the type, `method.json` and the registry all contain the string; the string's presence proves nothing either way.

**Acceptance Scenarios**:

1. **Given** a cold start, **When** the first spec-store read happens, **Then** the spec-kit pack is already registered and the read succeeds.
2. **Given** the pack registry, **When** spec-kit is resolved, **Then** it resolves by the same code path a marketplace-installed pack would.

### User Story 2 - Spec-kit cannot be uninstalled (Priority: P1)

Attempting to remove the spec-kit pack is refused with a reason naming why.

**Why this priority**: `bos-system-specs` depends on it unconditionally. A generic error here would read as a bug; silent success would break every system spec.

**Independent Test**: Attempt uninstall via every path that can reach it. Note the pack is in-tree with no marketplace entry (FR-008), so there is no Marketplace row to click — the reachable surfaces are the registry API and any uninstall op that enumerates packs.

**Acceptance Scenarios**:

1. **Given** the spec-kit pack, **When** the user attempts uninstall, **Then** it is refused with a message naming `bos-system-specs` as the dependant.
2. **Given** the same pack, **When** an upgrade is available, **Then** upgrade still proceeds — non-removable is not non-upgradable.

### User Story 3 - Templates and process agents live in the pack (Priority: P1)

`.specify/templates` and the four process agents are no longer read from BOS source at runtime.

**Why this priority**: these are the assets that must travel per-pack. Leaving them in source means the next pack has no template mechanism to use.

**Independent Test**: Assert no runtime code path reads `.specify/templates`; assert `seed/agents/` contains no process personas.

**Acceptance Scenarios**:

1. **Given** the pack is registered, **When** an agent reads a command prompt, **Then** it resolves through `/Methods/spec-kit/templates`.
2. **Given** `seed/agents/`, **When** it is listed, **Then** it holds only capability agents — verified against the real tree: `assistant`, `build-studio`, `conversation-reviewer`, `default_agent`, `developer`, `devops`, `docs-cataloger`, `docs-explorer`, `researcher`, `scheduler-architect`. (An earlier draft listed `writer` and `file-organizer`, which do not exist in `seed/agents/` — they are `data/agents/` entries — and omitted four that do.)
3. **Given** the `architect` agent, **When** Build Studio delegates the `design` step, **Then** it resolves through a pack agent root and runs.

### User Story 4 - Command prompts no longer reference unreachable scripts (Priority: P2)

An agent following a command prompt is never instructed to run a script it cannot execute.

**Why this priority**: a real source of agent confusion, but it degrades gracefully today (the agent fails, notices, and works around it) so it ranks below the structural moves.

**Independent Test**: Grep the relocated command prompts for `scripts/`; there must be no reference.

**Acceptance Scenarios**:

1. **Given** any relocated command prompt, **When** an agent reads it, **Then** it contains no instruction to run a shell script.
2. **Given** the `.specify/scripts/` directory, **When** the repo is searched, **Then** it no longer exists and nothing references it.

### Edge Cases

- **Boot ordering.** The pack must be registered before the first spec-store read. `ensureStoresOnce()` is called from several entry points (spec-fs, `/api/specs`, the VFS mount); if pack registration is lazier than any of them, the first request after a cold start sees no descriptor. Register at `instrumentation.ts` time and assert it rather than relying on import side-effects.
- **A locally edited agent now shadows the pack's copy permanently.** A user who has edited `architect` has a `.seed-rev` mismatch protecting it; after the move, their copy in `data/agents/architect` wins by precedence and silently stops receiving pack updates. That is the correct outcome but must be surfaced, not silent.
- **The constitution does not move.** It is spec *content* and already lives in the store at `bos-system-specs/.specify/memory/constitution.md` (`pipeline.ts:36`). Only the engine relocates.
- **`WRITE_ALLOW_PREFIXES` would grant a path that no longer exists.** `src/lib/dev/repo-fs.ts:28` lists `.specify/`; leaving it is a quiet permission grant to nothing rather than an error.
- **The relocation is a large diff with no functional change** — exactly the shape reviewers skim. Keep it a pure move in one commit; any rewording goes in a separate one.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The descriptor MUST support `builtin: true`. A `builtin` pack MUST refuse uninstall with a reason naming its dependant, and MUST remain upgradable.

- **FR-002**: `.specify/templates` MUST relocate into the pack and mount at `/Methods/spec-kit/templates` (per `045` FR-011). No runtime code path may read it from BOS source afterwards.

- **FR-003**: `.specify/` MUST be removed from `WRITE_ALLOW_PREFIXES` (`src/lib/dev/repo-fs.ts:28`).

- **FR-004**: `.specify/scripts/` MUST be deleted, and the relocated prompts' `.specify/` references MUST be **discriminatingly rewritten, not blanket-replaced**: `.specify/templates/*` → the pack mount (four templates: `plan`, `spec`, `tasks`, `constitution`); `.specify/extensions.yml` (exactly 35 occurrences across ten command files — `plan`, `specify`, `implement`, `converge`, `taskstoissues`, `analyze`, `clarify`, `checklist`, `constitution`, `tasks`), `init-options.json` and `feature.json` → deleted as unreachable from a read-only VFS mount; `.specify/memory/constitution.md` → **preserved verbatim**, since that path stays valid (FR-011). Every script reference MUST be scrubbed from the relocated command prompts. **Note the literal string `.specify/scripts` appears zero times**: the references are unprefixed relative paths in a YAML `scripts:` frontmatter block (`scripts/bash/…`, `scripts/powershell/…`) across the eight named files, so a literal-substring check passes while every block survives (`commands/{plan,tasks,analyze,implement,clarify,checklist,converge,taskstoissues}.md`). BOS's native equivalents (`nextFeatureId`, phase derivation) already cover their function.

- **FR-005**: `architect`, `architect-reviewer`, `ui-designer` and `devil-s-advocate` MUST relocate from `seed/agents/` into the pack's agent root, all with `picker` visibility. They are process, not domain — spec-kit itself defines no roles, so these are BOS's own opinion and belong with the method that invokes them.

- **FR-006**: `seed/skills/build-studio/` MUST relocate into the pack as its driver skill: `SKILL.md` plus the per-step references (`constitution`, `specify`, `clarify`, `design`, `plan`, `tasks`, `analyze`, `implement`, `converge`). The `target-*.md` files MUST NOT relocate — `044` already moved them to `bos-domain`.

- **FR-007**: The BOS-owned intent skill, ui-designer craft skill and `bos-domain` (all from `044`) MUST remain BOS-owned. Pack content MUST reach them by skill id via `skill_load`/`skill_read_file`, never by path into their internals.

- **FR-008**: The pack MUST ship in-tree at `seed/method-packs/spec-kit/` and be **pre-seeded through the same install path a marketplace pack uses** (`045` FR-012's register-descriptor / register-roots / register-mount sequence), invoked at startup from `instrumentation.ts` rather than fetched. What differs is only the *source* (in-tree, not a marketplace clone) and the *trigger* (boot, not a user action) — **not the registration code**.

  > This is the spec's whole point and the easiest thing to get wrong. Registering spec-kit through a bespoke boot path would recreate exactly the two-code-paths problem this feature exists to remove, and would leave the install path unexercised until `047`. If the implementation ends up with a spec-kit-only registration function, the feature has failed regardless of what else passes.

- **FR-008a**: Upgrades MUST flow through the normal path, but **agents and the skill reconcile through different contracts** and the requirement splits accordingly:
  - **Agents** — the `.seed-rev` two-hash contract (`subagents/store.ts:53-77`, `seed-sync.ts:107-123`). The seed MUST write only to `data/agents/` (`045` FR-001b), never into a pack root.
  - **The driver skill** — the bundled-asset provenance contract (`.installed-from.json`, `bundledAssets.ts:54`, `:170-181`): replace when untouched, surface a conflict when diverged. It CANNOT use `.seed-rev` — not because of `installSkill` (ADR-3 rejects that mechanism outright as uninvokable for an in-tree pack) but because the **chosen** mechanism — `seedOneKind` (`bundledAssets.ts:139-183`), whose copy step writes `.installed-from.json` (`:54`) and no `.seed-rev` — records provenance in that file alone. (Bare `copyAsset` at `:118-123` is *not* the chosen mechanism; FR-008b forbids it.) For the record `installSkill` would also fail, writing no stamp and setting `createdBy: "user"` (`client.ts:859`), leaving the copy `local` forever.

- **FR-008b**: The relocation hand-off MUST be specified and tested, because it can silently destroy user-visible function on the first boot after upgrade. Moving an id out of `seed/` makes `decideSeedAction` see a deletion (`seed-sync.ts:121`). The required behaviour, as three separate obligations:
  1. **The archive fires** for both agents and the skill — it is the hand-off, not a hazard. For agents the pack root then supplies them. For the skill it requires a **dedicated seeding step**, not the shared item scan: `reconcileInstalledItemAssets()` is memoized per data root (`bundledAssets.ts:244`) and the **agent** store consumes that single pass first (`subagents/store.ts:187`), while the skill incumbent is still present, so the skill store's later call at `:200` is a no-op and the driver skill is lost for the life of the process. the pack's skill MUST be seeded by its own step inside `skills/store.ts`'s `ensureSeed()`, ordered after the archive, over `seedOneKind` (`bundledAssets.ts:139-183`) and `recordPendingConflicts` (`:301`) — **not bare `copyAsset`** (`:118-123`), which is an unconditional `rm -rf` + `cp` and would clobber the incumbent this requirement says must surface as a conflict.
  2. An **edited** incumbent is left alone. For agents that is FR-010's shadow report. For the skill FR-010 does not apply (it is scoped to `data/agents/<id>`), and the edited skill lands in the same terminal state as the unstamped one, having no `.installed-from.json` either.
  3. An **unstamped** incumbent — the state of every `data/skills/` directory observed — MUST surface as a `BundledAssetConflicts` entry rather than silently leaving the pack's copy uninstalled.

- **FR-009**: Any code path special-casing the built-in descriptor **in registration or resolution** MUST be removed once the pack registers through the ordinary registry. Scoped deliberately: FR-001's uninstall refusal *is* a `builtin` special-case and is required, so an unscoped version of this requirement would contradict it.

- **FR-010**: A pack agent shadowed by a local `data/agents/<id>` copy MUST be reported somewhere visible, so silently-frozen customisations are discoverable.

- **FR-011**: The spec-kit pack MUST declare `constitution: ".specify/memory/constitution.md"` — the existing path — so nothing moves for existing stores. The path is a fossil (named after a framework whose engine has just left the tree) but relocating it is out of scope here: **what happens to a constitution when a store's method changes is owned by `045` FR-009a**, so that one moment has one owner. This requirement is only the pack's declaration.

- **FR-012**: Verification MUST cover prompt-reachable paths, not only pipeline output. `045`'s `Specification` parity check (SC-001) captures derived objects, not the text agents read — so FR-002 can change every `/Templates/...` path an agent resolves while parity still passes. Assert explicitly that every template path referenced by relocated prompts resolves through `/Methods/spec-kit/templates`.

- **FR-013**: Remaining spec-kit-specific prompt and constant references MUST be de-coupled or explicitly declared spec-kit-only: `src/lib/self-heal/brief.ts:39-40`, `:58-59`, `:172`; `src/lib/agent/config.ts:15` (CORE_POLICY); `src/lib/agent/tool-groups.ts:128` (search alias) **and `:56`** (the `file` tool-group description, which names `/Templates` in agent-facing prompt text); `src/app/api/specs/run-tests/route.ts:127` (`test-results.md`); **and `seed/agents/build-studio/AGENT.md` — the most spec-kit-coupled BOS-owned asset in the repo**, which hardcodes the nine-step sequence and `skill_load("build-studio")` (`:31`), `/Templates` (`:42`,`:44`) and the constitution path (`:47`). It stays BOS-owned while the driver skill it loads becomes pack content, so under `047` it would name the wrong skill and the wrong phase list; it MUST reach the driver via the active pack, not by a literal id. *(Relocated here from `045` during review: this is spec-kit de-coupling, which is this feature's subject, and `045` had grown to 32 requirements.)*

- **FR-014**: `buildstudio_run_tests` has **two** method-blind assumptions, and FR-013 addresses only the output filename. `src/app/api/specs/run-tests/route.ts:65-70` derives `featureId` from the unit path's last segment and looks for `e2e/<featureId>.spec.ts` **inside BOS's own repository**. Under OpenSpec a unit is named (`add-dark-mode`), under BMAD it is a sharded story — and for an item-owned store the tests do not live in BOS's repo at all, which is already wrong today. The test-file naming convention MUST become descriptor-declared, and the tool MUST refuse with a clear reason when the active method or store kind provides no test location, rather than silently looking in the wrong repository.

- **FR-015**: This feature MUST carry e2e coverage for pack registration and template resolution — built on `045` FR-017a's fixture-pack harness, which is a hard prerequisite: that the pack is registered before the first spec-store read on a cold start, that uninstall is refused, and that a relocated command prompt's template path resolves. Per BOS's testing rule, fixtures MUST be bundled by the test rather than depending on pre-seeded store content.

- **FR-016**: Documentation MUST be updated. `docs/usage/apps/build-studio.md:62` ("only ever edits files under `specs/` and `.specify/`") is also falsified. `docs/dev/build-studio.md`'s "Conventions" (`:193-195`) states that "spec-kit's ENGINE is vendored under `.specify/templates` … and stays in BOS source" — the exact claim this feature falsifies. `docs/dev/repository-and-data-layout.md:7` makes the same claim and MUST be **corrected**, not merely extended. `docs/dev/assistant/sub-agents-and-delegation.md:37` describes the build-studio agent as authoring "with spec-kit". **`CLAUDE.md` (repo root) asserts the engine "stays in source at `.specify/templates`"** and MUST be corrected — Principle VI makes its accuracy binding. A new `docs/dev/method-packs.md` MUST record the agents-discovered / skill-copied asymmetry as a known shape.

- **FR-017**: This feature MUST retire the `/Templates` alias that `045` FR-011 keeps for one release. `045`'s ADR-6 justified the alias as time-boxed on the assumption that FR-002 retired it — FR-002 relocates `.specify/templates` and forbids reading it from source, but says nothing about the alias, leaving retirement with no owner. Once templates resolve through `/Methods/spec-kit/templates`, the alias MUST be removed and every prompt referencing `/Templates/...` (e.g. the relocated `SKILL.md:66`) updated to the pack path.

- **FR-018**: The pack's **source** root MUST resolve relative to the running tree — which `process.cwd()` already gives, so the substantive half is the second: the seeded skill's provenance MUST record that tree's rev. Pack roots resolve from `process.cwd()` (`subagents/store.ts:33`, `skills/store.ts:19`) while `dataDir()` may be a COW-isolated candidate clone, so without this a promote or discard leaves provenance describing a tree that no longer exists. This carries Principle IV's (NON-NEGOTIABLE) conditional pass, which otherwise has no owner. The mechanisms are `itemPath` via `sourceRootFor` (`bundledAssets.ts:63-64`) for the pack **source**, and `Provenance.version` (`:56-60`) to record the running tree's rev. **Not `root?`** — that is the *destination* knob (`destRootFor`, `:62`), documented as "a FEATURE BRANCH's data clone to seed **into**" (`:198-199`).

### Non-requirements

- No behavioural change to the spec-kit pipeline. This is a relocation.
- The constitution's location is unchanged (it is store content, not engine).
- No OpenSpec or BMAD work (`047`, `048`).

### Key Entities

- **spec-kit pack** — `builtin: true`, non-removable, in-tree, registered at boot. Holds templates, command prompts, the driver skill, and four process agents.
- **Capability agent** — what remains in `seed/agents/`: a tool-bearing capability (`developer` has repo access) rather than a method persona. A pack's Dev persona *binds to* `developer` rather than replacing it.

### Mechanism requirements

> Each of these governs a seam where the pack's two kinds of content, or its
> visibility rules, behave differently than they look.

- **FR-019**: **Agents are DISCOVERED in place; skills are COPIED.** The two are not symmetric and the asymmetry MUST be stated, because it is invisible from the pack layout — both look like folders of markdown inside the pack. An agent is read from the pack root through a registered agent root, so the pack's copy is the live one. A skill is COPIED into `data/skills/` by a seeding step, so a pack whose skills are never seeded contributes nothing — and its agents' skill declarations are dropped **silently** by the allowlist filter, which cannot distinguish "not permitted" from "not present". Installing a pack MUST seed its bundled skills, and a skill named by an agent but absent MUST be reported, not filtered away.

- **FR-020**: **`getAgent` MUST resolve through the delegatable listing, never the picker-filtered one.** Visibility scopes DISCOVERY, not reachability: a `delegate-only` agent is meant to be absent from the picker and still callable by `agent_delegate`. Routing the lookup through the picker listing makes every delegate-only agent undeliverable while both listing functions behave correctly in isolation — so a test asserting the listings directly passes while the feature is broken. The test MUST drive `getAgent`, i.e. the path `agent_delegate` actually takes.

- **FR-021**: **A pack's `roles` MUST name resolvable agent ids, and a dangling role MUST be reported at install.** Roles are written by hand against a framework's own vocabulary, which rarely matches the pack's directory names: the likely error is naming the framework's skill ids rather than its agent directories. A role pointing at nothing degrades to "the role silently does not exist", which is indistinguishable from the role having no opinion.

- **FR-022**: **Seed reconciliation MUST distinguish a local edit from a dropped seed, and MUST never write inside a pack root.** The relocation hand-off (`.seed-rev` two-hash stamp) and install provenance (`.installed-from.json`) are DIFFERENT contracts and MUST NOT be conflated. An unstamped local copy MUST be treated as a user edit (guarded before the archive path), or reconciliation archives work the user did by hand. Writes MUST resolve through the seed's own write root so a pack's files are never rewritten in place — that would revert the pack's content and lose it on the item's next upgrade.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: `seed/agents/` contains zero process personas.
- **SC-002**: Zero runtime code paths read `.specify/templates` from BOS source.
- **SC-003**: `WRITE_ALLOW_PREFIXES` grants no path that does not exist.
- **SC-004**: Uninstalling the spec-kit pack is refused with a message naming `bos-system-specs`.
- **SC-005**: **Zero spec-kit-only registration functions** — asserted by test, by following the call sequence, not by searching for the word `builtin` (which FR-001 makes a legitimate field).
- **SC-005a**: Simulating an upgrade from a real pre-relocation `data/`, the four relocated agents resolve from the pack root and `build-studio` resolves to the pack's copy. The fixture MUST cover **all three** skill branches — stamped-unedited, unstamped, and edited — and the **stamped** one is mandatory: `seedFromDiskPath` writes a stamp (`skills/store.ts:139`), so a deployment seeded after that code landed has a stamped `build-studio` — that is the branch whose archive fires, and therefore the branch the memo defect destroys. A fixture using only an unstamped incumbent (an artifact of older installs) passes straight over the defect.
- **SC-006**: Zero **unreachable** script/config references anywhere in the repo or pack — the `scripts:` frontmatter blocks (matched as `scripts/bash/` and `scripts/powershell/`, **not** as `.specify/scripts`, which never appears literally), `extensions.yml`, `init-options.json`, `feature.json`. `.specify/memory/constitution.md` references are preserved and MUST NOT be counted as violations.
- **SC-007**: `045`'s `Specification` parity check (its SC-001) still passes.
- **SC-007a**: Every template path referenced by a relocated command prompt resolves through `/Methods/spec-kit/templates` — verified by resolving each path, not by grepping for the string.
- **SC-008**: A full `specify → design → plan → tasks` run works, including `architect` and `ui-designer` delegations resolving through a pack root.
- **SC-009**: `buildstudio_run_tests` refuses with a reason under a method or store kind that declares no test location, instead of looking in BOS's own repository.
- **SC-010**: No spec-kit-specific string remains in `self-heal/brief.ts`, `agent/config.ts`, `tool-groups.ts` or `seed/agents/build-studio/AGENT.md` except where explicitly declared spec-kit-only. The AGENT.md check is the load-bearing one: it must name no literal driver-skill id and no literal phase list.
- **SC-011**: Neither `docs/dev/build-studio.md`'s "Conventions", nor `docs/dev/repository-and-data-layout.md:7`, nor `CLAUDE.md` still claims the spec-kit engine stays in BOS source.
- **SC-011a**: FR-017's `/Templates` retirement is verified by resolving every reference in the enumerated surface, not by grep — a missed one is a silently broken pipeline step, not a compile error.
- **SC-012**: `npm run test:unit` green; `npx tsc --noEmit` and `npm run lint` clean.

- **SC-013**: Installing a pack seeds its bundled skills; an agent's declared skill that is missing is reported rather than silently dropped.
- **SC-014**: A `delegate-only` agent is absent from the picker and resolvable through `getAgent` — asserted through that function, not through the listings.
- **SC-015**: A pack whose `roles` name an unresolvable agent reports it at install time.

## Assumptions

- **This feature's value is almost entirely enabling, and that is deliberate.** FR-013's straggler de-coupling (self-heal, CORE_POLICY, tool-groups) does stand alone; everything else is relocation, so if the series stops here BOS has gained one prompt cleanup and otherwise indirection. Its justification is that the install path must be exercised by BOS's own method before a third-party pack depends on it — **a precondition now satisfied**, since `047` and `048` are both committed (Q4). The enabling-only nature stays worth stating: a reviewer seeing a large diff with no user-visible change should know that is the intent, not an oversight.
- Shipping the pack in-tree rather than fetching it is acceptable and necessary: `bos-system-specs` depends on it at first read, so a marketplace fetch on first boot would deadlock.
- Treating spec-kit's vendored scripts as dead is safe because BOS never executed them — they were only ever reachable as text through a read-only VFS mount.
