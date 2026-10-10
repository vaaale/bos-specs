# Feature Specification: Workflows — a pipeline you can name, fork and own

**Feature Branch**: `051-workflows`

**Created**: 2026-09-13

**App Target**: bos-core

**Input**: A method's pipeline is fixed by whoever wrote the pack. A user who wants BMAD-but-without-the-PRD, or spec-kit with an extra review step, has no move except editing pack files that the next upgrade overwrites. Upstream frameworks already solve this — BMAD with a declared override surface, OpenSpec with `schema fork` — and BOS exposes neither.

> Depends on `045` (descriptor, binding) and `049` (workflows as the binding unit, `driverSkill`). Independent of `050`. `049` deliberately binds by WORKFLOW already, so this feature adds authoring and does not rework that signature.

## Why this exists (context)

`049` made a **workflow** — a named pipeline within a method — the thing a store or project binds to. Every method already has the concept, and every method already lets its own users change it. BOS lets them choose one and nothing more.

### What the three methods actually do

| Method | Mechanism | Shape |
|---|---|---|
| **BMAD** | `customize.toml` per workflow | Declares the override SURFACE and its merge rules: scalars override, arrays append, keyed tables replace-or-append. The file itself says `DO NOT EDIT -- overwritten on every update`; overrides live OUTSIDE the pack (`_bmad/custom/<workflow>.toml`, plus a `.user.toml` for personal ones). |
| **OpenSpec** | `openspec schema fork` | Copies a whole schema directory (`schema.yaml` + templates) into an independent one, dereferencing links so the fork owes nothing to its source. |
| **spec-kit** | none | Its pipeline is fixed. |

These look like two idioms — **override** keeping a live relationship with the base, **fork** severing it — and an earlier draft of this spec set out to build both.

**BOS builds one, deliberately.** Two things settled it. First, the pack OVERLAY (048) already handles content — a prompt, a template, an agent — at file granularity, so editing one keeps every other file's upstream improvements without any patch machinery at all. Second, for STRUCTURE the merge idiom pays for itself only if re-applying a change to a moved base can be done mechanically, and it cannot: a base that renames a phase needs judgement about whether the user's change still means the same thing. An agent has that judgement and can say what it did; a merge engine answers silently.

So: content is overlaid, structure is forked, and an upgrade is "fork the new version and re-apply my changes" — performed by an agent, with the diff BOS computes for it. No patch format, no path selectors, no merge-rule table, and no declared override surface — a field no pack ever filled. (BMAD does ship an override surface, in its OWN format: a `customize.toml` per skill with a three-layer merge. That makes the case stronger, not weaker — a second merge underneath a pack's own is how "which version am I running" gets two answers.)

### The rule this feature must not break

A customization **must not live inside the pack**. BMAD states why in every `customize.toml`: pack content is replaced wholesale on upgrade, so an edit there is lost without warning. The same is true of BOS's packs — they are marketplace items, and an upgrade is a re-symlink.

## The split: what BOS provides, what a pack must

The question this feature turns on. Get it wrong in the generous direction and every new method needs BOS code before it can be authored; get it wrong in the stingy direction and BOS's editor can only offer a text box over JSON.

**BOS owns the GRAPH. The pack owns the VOCABULARY.**

A workflow already IS a graph in every descriptor: `phases[]` are nodes and `requires` are edges. That structure is method-neutral — it is the same shape for spec-kit's nine phases, BMAD's modules and OpenSpec's proposal/specs/design fan-out — so everything that operates on the *shape* is BOS's, once, for every method that will ever exist:

| BOS provides, for every pack | Why it can |
|---|---|
| Rendering the DAG — boxes, arrows, layered layout | `phases[].requires` is already there |
| Structural editing — add/remove/rename a phase, draw or cut an edge, reorder artifacts | Operations on nodes and edges, not on meaning |
| Structural validation — cycles, dangling references, duplicate ids | Properties of a graph, provable without knowing what a phase *is* |
| Fork, provenance, and the diff an upgrade re-applies | Mechanism (FR-002, FR-008) |
| Binding, preflight, listing, deletion | Already 045/049 |
| Overlaying a pack's CONTENT — prompts, templates, agents | 048's overlay; file granularity, no declaration needed |

What BOS must never learn is what any of it MEANS. It cannot know that BMAD's product brief belongs before the PRD, that an OpenSpec change is archived rather than deleted, or what a spec-kit `clarify` phase is for. That knowledge belongs to the pack, exactly as 049 settled for driver skills:

| The pack provides | Form |
|---|---|
| Which phases exist, their labels and rules | `method.json` — already |
| Templates a phase generates | pack files — already |
| How to RUN the pipeline | the driver skill — 049, already |
| A phase's INSTRUCTIONS — the prompt for that step | a declared path (FR-021) |
| Semantic constraints — "this phase cannot be removed", "this one must precede that" | **data** in the descriptor (FR-015) |
| Authoring richer than a declaration can express | a `plugin` facet, gated by 045 FR-013a (FR-012) |

**The line is drawn at validity.** BOS can always say a graph is *malformed*. Only the pack can say it is *wrong for this method*. So structural checks are BOS's and semantic ones are declared by the pack — and where a pack declares nothing, BOS still gives a working editor, just one that will let you build a pipeline its own method thinks is odd.

> **AS BUILT — two candidate structural rules were struck, measured first.**
> "An unreachable phase" and "an artifact no phase generates" both look
> structural and are neither: spec-kit ships THREE ungenerated artifacts
> (`research.md`, `data-model.md`, `quickstart.md`), and declares zero gates, so
> under a reachability rule all twelve of its phases are unreachable. Both would
> have flagged BOS's own default method as malformed. Isolation is reported as a
> DIAGNOSTIC with its reason instead.
>
> One rule was also added: cycles are checked over gates **and** `dependsOn`, not
> gates alone. `evaluate.ts` recurses through both and throws on either.
>
> **The semantic column was never built** (see FR-015). No pack declares a
> constraint, so it would have been a mechanism with no subject — the same
> finding that killed override.

**The chat is the third thing, and it is why this works.** A user who wants "BMAD without the PRD" does not want to reason about nodes and edges; they want to say that sentence. The Build Studio agent already carries the pack's vocabulary (its driver skill) and can already drive BOS's mechanism (its tools). So the agent is the bridge between the two columns above — the meaning lives with it, not in BOS — and the canvas is what makes its edits reviewable rather than a wall of diff. Neither surface is sufficient alone: the canvas without chat cannot express intent, and chat without the canvas gives nothing to check.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fork a workflow and change it (Priority: P1)

A user takes an installed workflow, forks it under a new name, edits the copy, and binds a store to it.

**Why this priority**: the universal mechanism, and now the ONLY one for structure. It works for every method, including all three shipped today, none of which declares anything customizable.

**Acceptance Scenarios**:

1. **Given** an installed workflow, **When** the user forks it, **Then** a user-owned workflow exists under a new id, appears in `workflow_list`, and is bindable exactly like a pack's own.
2. **Given** a fork, **When** the source pack is upgraded, **Then** the fork is UNCHANGED — that is what forking is for.
3. **Given** a fork, **When** the source pack is UNINSTALLED, **Then** the fork keeps working; it owes nothing to its source.

### User Story 2 - Move a fork onto a newer version of its source (Priority: P1)

A user forked BMAD months ago. BMAD has released twice since. They ask the agent
to bring their workflow onto the current version.

**Why this priority**: without it a fork is a SILENT FREEZE. The user believes
they are current; they are two versions behind and nothing said so.

**Why an agent and not a merge**: re-applying a change onto a moved base needs
judgement BOS does not have. If the base renamed `prd` to `requirements`, does the
user's change to `prd` still mean the same thing? A merge engine answers that
mechanically and silently. An agent can answer it and SAY what it did — including
"I could not map this one, here it is".

**Acceptance Scenarios**:

1. **Given** a fork whose source has released a newer version, **When** the user
   or the agent asks, **Then** BOS reports the version it was forked at, the
   version available now, and exactly which phases the user added or removed.
2. **Given** that report, **When** the agent upgrades, **Then** it forks the
   CURRENT version and re-applies those changes, saying what it re-applied and
   what it could not.
3. **Given** a fork whose source pack is no longer installed, **When** status is
   asked, **Then** that is reported and is NOT an error — the fork still works,
   there is simply no newer base to move onto.

### User Story 3 - Binding selects a workflow, and says so (Priority: P1)

Build Studio's picker offers workflows, not methods.

**Why this priority**: the user-visible half of `049`'s model. Until the UI says "workflow", the fact that a method may provide several is invisible.

**Acceptance Scenarios**:

1. **Given** a method providing several workflows, **When** the user opens the picker, **Then** all of them are offered, grouped by their method.
2. **Given** an existing `method`-keyed binding, **When** it is read, **Then** it still resolves — as that method's default workflow.

### User Story 4 - See the pipeline as a graph, and edit it there (Priority: P1)

A user opens a workflow and sees its phases as boxes and arrows — the same visual language the Workflow app already uses for step graphs — and changes it by direct manipulation.

**Why this priority**: a pipeline IS a graph, and it is the only artifact in BOS that has been readable only as JSON. "BMAD without the PRD" is one deletion on a canvas and a paragraph of prose in a file; the canvas is also what makes an agent's edit reviewable at a glance rather than as a diff.

**Acceptance Scenarios**:

1. **Given** any installed workflow, **When** the user opens it, **Then** its phases render as a DAG laid out by dependency depth, with each phase's artifacts and current rules readable without leaving the canvas.
2. **Given** a workflow the user owns, **When** they remove a phase, add one, or change an edge, **Then** the change is applied to their own copy and the canvas re-renders.
3. **Given** an edit that would create a CYCLE or orphan a phase, **When** it is attempted, **Then** it is refused on the canvas with the offending path named — structural validity is BOS's to enforce.
4. **Given** a PACK's workflow (not the user's), **When** it is opened, **Then** its STRUCTURE is read-only and says why, offering Fork — while its phase prompts stay editable through the overlay — rather than silently accepting edits that FR-005 forbids storing.
5. **Given** a phase whose meaning BOS cannot know, **When** it is selected, **Then** the inspector shows what the PACK declared about it, never BOS's guess.

### User Story 5 - Change a workflow by describing the change (Priority: P1)

A user tells the Build Studio agent "drop the PRD step and let architecture start from the brief" and watches the graph change.

**Why this priority**: the canvas expresses structure; only prose expresses intent. The agent holds the pack's vocabulary (its driver skill), so it can translate a sentence about BMAD into operations on a graph that BOS itself does not understand.

**Acceptance Scenarios**:

1. **Given** a user-owned workflow open on the canvas, **When** the agent edits it through its tools, **Then** the canvas reflects each change as it lands, so the user reviews a picture rather than a diff.
2. **Given** an agent edit that would break structural validity, **When** it is attempted, **Then** it is refused by the SAME check a direct edit hits — one rule, not a second implementation for the agent path.
3. **Given** a request against a PACK's workflow, **When** the agent acts, **Then** it either edits the pack's content through the overlay or forks first, and says WHICH — never an edit that FR-005 forbids storing, and never a fork where an overlay edit would have done.

### User Story 6 - Edit what a phase TELLS THE AGENT TO DO (Priority: P1)

A user opens a phase and edits its instructions — the prompt the agent executes
for that step — either directly in the inspector or by asking the agent to change
it.

**Why this priority**: the instructions are the phase. A pipeline's structure
says which steps exist and in what order; the prompt says what actually happens
when one runs. Adding `ui-design`, `design` and `review` to spec-kit created three
steps with NO instructions at all — they exist, they are in the strip, and they
tell the agent nothing. Structure without prompts is a table of contents for a
book nobody wrote.

**Acceptance Scenarios**:

1. **Given** a phase whose pack ships instructions, **When** the user opens it,
   **Then** the prompt is shown, with where it came from (the pack, or their own
   edit).
2. **Given** a phase the user edits, **When** they save, **Then** the change is
   written OUTSIDE the pack and survives upgrade, uninstall and reinstall
   (FR-005) — no fork required for a prompt change.
3. **Given** an edited phase, **When** the user reverts, **Then** the pack's own
   instructions are in force again and their edit is gone.
4. **Given** a phase the pack ships NO instructions for, **When** it is opened,
   **Then** that is stated plainly and the user can write them.
5. **Given** the user asks the agent to change a phase's instructions, **When** it
   does, **Then** it goes through the SAME write path as the inspector — one
   implementation, per FR-019.

### User Story 7 - Authoring tools ship with the pack (Priority: P2)

The knowledge of what a workflow's parts MEAN belongs to the method that defines them.

**Why this priority**: real, but a generic editor over the declared surface serves every pack from day one; per-pack authoring is a refinement.

**Acceptance Scenarios**:

1. **Given** any pack, including one that declares nothing customizable, **When** the user edits its workflow, **Then** BOS's generic editor serves it — no per-method code.
2. **Given** a pack wanting richer authoring than a declaration expresses, **When** it ships that as a `plugin` facet, **Then** it is subject to `045` FR-013a's per-origin opt-in like any other marketplace code.

### Edge Cases

- **Deleting or renaming a workflow that stores are bound to.** Must run `045` FR-010's preflight: report which units would be affected and refuse silently hiding them.
- **A fork of a fork.** Allowed; provenance records the immediate source, not a chain that invites cycles.
- **A fork whose source pack is gone** (uninstalled). The fork KEEPS WORKING — it owes nothing to its source — and its status reports that there is no newer base to move onto. Not an error.
- **Name collisions** between a user workflow and a pack's. `049` FR-004's rule applies: reported, never resolved by order; `<method>:<workflow>` disambiguates and a user workflow needs an equivalent qualified form.
- **A pack upgrade that changes a phase the user had changed.** Reported as part of the fork's status, with the phases the user added or removed, so the agent performing the upgrade can say what it could and could not map. Never merged blindly.
- **Editing a pack file directly.** BOS must not offer it, and should say why when asked: the next upgrade discards it.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A pack MUST be able to declare **more than one** workflow, each with its own phases, artifacts and driver behaviour. `049` established the addressing (`<method>:<workflow>`, bare when unambiguous, a bare method id meaning its default); this feature makes multiple workflows real rather than the degenerate single case.

- **FR-002**: BOS MUST support **forking** a workflow into a user-owned copy under a new id. The fork MUST be independent: a later pack upgrade or uninstall does not change or remove it.

- **FR-004**: Changing a pack's CONTENT and forking it are DIFFERENT operations with different consequences, and MUST be presented as such. Editing a prompt keeps every other upstream improvement; forking stops receiving them. Offering only the second, or blurring the two, pushes a user into a fork for a change that never needed one.

- **FR-005**: A customization MUST NOT be stored inside the pack. Pack content is replaced wholesale on upgrade, so an edit there is lost silently — which is why every BMAD `customize.toml` opens with `DO NOT EDIT`. User workflows and overrides live in user-owned storage and survive upgrade, uninstall and reinstall.

- **FR-007**: A fork MUST record the source EXACTLY as it was when taken, so "what did the user change" is COMPUTABLE rather than guessed. Without it a user's edits cannot be told apart from what the base already said, and an upgrade has nothing to re-apply.

- **FR-008**: A fork MUST record what it derives from and at what version, and a NEWER base MUST be reported. Unreported, a fork is a silent freeze: the user believes they are current while two versions behind. BOS reports; it never merges — re-applying a change onto a moved base needs judgement it does not have (US2).

- **FR-009**: A user workflow MUST be addressable, bindable, listable and deletable exactly like a pack's own. If it is second-class anywhere, users keep editing pack files.

- **FR-010**: Deleting or renaming a workflow that stores or projects are bound to MUST run `045` FR-010's preflight and refuse to silently orphan content.

- **FR-011**: The Build Studio picker MUST offer **workflows**, grouped by providing method, and MUST be labelled as workflows. Existing `method`-keyed bindings MUST keep resolving as that method's default (`049` FR-009).

- **FR-012**: Workflow authoring MUST work for a pack that declares nothing about being customizable — all three shipped packs are in that state. A pack needing richer authoring than the generic editor MUST ship it as a `plugin` facet, subject to `045` FR-013a's per-origin opt-in.

- **FR-013**: `workflow_list` MUST report user workflows alongside pack ones, marking which are customized and what they derive from — it is the agent's only view of what is available.

- **FR-014**: Documentation MUST explain when to edit a pack's content and when to fork, and where each is stored. A user who does not know the difference will fork to change one prompt and thereby stop receiving that pack's improvements entirely.

- **FR-021**: A phase MAY declare its **instructions** — the prompt an agent runs
  for that step — as a pack-relative path. DECLARED, never inferred from a
  filename: spec-kit happens to ship `templates/commands/<phase-id>.md`, and
  matching on that convention would be BOS guessing at pack internals, which is
  the thing this feature's split exists to prevent. A pack that organises
  differently must work.

- **FR-022**: Instructions MUST be editable, from the inspector and by the agent,
  through ONE write path (FR-019). The user's version MUST be stored outside the
  pack and MUST survive upgrade, uninstall and reinstall — the existing pack
  OVERLAY (`data/method-packs/<id>/`, 048) is that destination, extended from
  agents to templates. A prompt change MUST NOT require forking the workflow.

- **FR-023**: A phase's resolved instructions MUST report their SOURCE — the pack
  or the user's overlay — and a user's version MUST be revertible to the pack's.
  Without that, an edit is indistinguishable from what shipped.

- **FR-024**: A phase whose pack declares no instructions MUST say so, and MUST
  still be writable. Three of spec-kit's twelve phases are in exactly this state.

- **FR-015** *(NOT BUILT — deferred, with the reason)*: A pack MAY declare **semantic constraints** on its own workflows as DATA — a phase that cannot be removed, an ordering that must hold. BOS enforces what is declared and nothing more. A pack declaring none still gets a working editor; it simply cannot object to an arrangement it dislikes. Data rather than code for the same reason as FR-006 and 049 FR-007: a constraint that arrives as executable content needs 045 FR-013a's origin gate, and most constraints do not need to be code.
  > **Deferred.** No shipped pack declares one, so building the enforcement would
  > have produced an inert mechanism — the same reason override was dropped. The
  > honest state today: structural validation ships, and a pack that wants to
  > object to an arrangement it dislikes still cannot. Revisit when a pack has
  > something to say.

- **FR-016**: A workflow MUST render as a **directed graph** — phases as nodes, dependencies as edges — laid out by the pack's DECLARED ORDER, with dependencies drawn over it as arcs. This is the pipeline's actual shape and it is method-neutral, so ONE renderer serves every method, present and future. No per-method view code.
  > **AS BUILT — two corrections, both from rendering the real packs.**
  > **1. `requires` is not the edge set.** Two of the three installed packs —
  > including the default — declare ZERO `requires` edges, because an unsatisfied
  > one makes a phase `blocked`. The structure is declared in three places, and
  > the graph is the union tagged by KIND: `gate` (enforced), `artifact` (declared
  > order), `flow` (derived from a rule reading a file another phase writes).
  > **2. "Laid out by dependency depth" was wrong.** `phases[]` declares a TOTAL
  > ORDER and is what the phase strip renders; only 2 of spec-kit's 11 consecutive
  > steps are backed by a dependency. Depth-layout scattered a pipeline whose order
  > is nine-tenths convention. Order is POSITION, not an edge — emitting the array
  > as "follows" edges would make OpenSpec's parallel `specs`/`design`/`ui` look
  > serial.

- **FR-017**: The canvas MUST support structural editing of a user-owned workflow — add, remove, rename and REORDER a phase; create and delete a gate; mark a phase optional; set which artifacts it produces — and MUST refuse an edit that breaks structural validity (a cycle, a dangling reference, a duplicate id, an empty phase list), naming what is wrong. Structural validity is BOS's to enforce because it is a property of the graph, not of the method.
  > **AS BUILT.** "Reorder a phase" is new here and matters more than it sounds:
  > FR-016's correction makes declared order THE pipeline, so a fork that cannot
  > reorder cannot change the thing most people fork to change. "Mark optional" is
  > new because `optional` became a declared field during this feature.
  > An op that does MORE than asked reports it rather than doing it silently:
  > removing a phase drops the gates pointing at it and orphans the artifacts it
  > produced, and both come back as warnings.

- **FR-018**: A pack's own workflow MUST be read-only on the canvas as to its STRUCTURE, offering Fork. Its phase PROMPTS stay editable, through the overlay, with no fork. An editable-looking canvas over content FR-005 forbids storing is a trap; so is sending a user to the expensive mechanism for the cheap need.
  > **AS BUILT — "Override" is gone.** No pack declares a surface, so the option
  > would never have been offered. The read-only surface narrowed to *structure*
  > once Phase 1b showed the overlay handles prompts completely.

- **FR-019**: The Build Studio agent MUST be able to perform every canvas operation through tools, and its edits MUST go through the SAME validation as a direct edit. Two paths to one outcome is how the two diverge; this subsystem has produced that defect repeatedly.

- **FR-020**: The inspector for a selected phase MUST show what the PACK declared — label, rules, artifacts, templates — and MUST NOT synthesise an explanation BOS cannot know to be true. Where a pack declares nothing for a field, that absence is shown as absence.

- **FR-025** *(added during implementation)*: Every structural edit MUST carry the **revision** it was made against, and a mismatch MUST be refused naming both revisions. Two surfaces edit the same workflow — the canvas and the agent — and without this "the user and the agent both edited" resolves by whoever wrote last, silently. This is why edits are OPERATIONS rather than a whole-descriptor save: a save cannot tell "the user removed a phase" from "the user is working off a stale copy".

- **FR-026** *(added during implementation)*: Adding a gate MUST report, **before it is applied**, how many live units it would newly block and which — with the number EXAMINED alongside. A gate is the only enforced edge kind and the measured hazard is severe: one plausible edge flips 120 of 132 features. "0 would be blocked" and "nothing is bound to this workflow, so nothing was checked" are the same number and opposite facts, so the denominator is part of the answer. The report MUST reach the agent's path too, not only the canvas's — a warning one route can skip is not a warning.

- **FR-027** *(added during implementation)*: Where an operation does MORE than it was asked to, it MUST say so; where it cannot do so without guessing, it MUST refuse and name why. Removing a phase drops the gates pointing at it and orphans the artifacts it produced — reported as warnings. Removing a phase some rule `dependsOn` is refused, because dropping a clause changes what the surrounding condition means. Setting an artifact the workflow never declared is refused, because BOS would have to invent a scope.

### Non-requirements

- **Editing a pack in place.** Explicitly refused (FR-005).
- **Publishing a user workflow** as a pack. A later decision.
- **Migrating an existing binding** to a different workflow — `045` FR-010's preflight already covers changing one.
- **Round-tripping to upstream formats.** BOS does not write `customize.toml` or run `openspec schema fork`; it implements the same two idioms over its own descriptor.

### Key Entities

- **Workflow** — a named pipeline. Provided by a pack, or user-owned.
- **Baseline** — the source EXACTLY as a fork took it, so `diff(baseline, yours)` is the user's own edits. Replaced the designed "override surface", which no pack declares.
- **Revision** — the `rev` an edit was made against; a mismatch is refused rather than merged.
- **Provenance** — what a user workflow derives from, and at what version.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A workflow can be forked, edited and bound, and the fork is unaffected by upgrading or uninstalling its source pack.
- **SC-002**: A prompt edited through the overlay survives a pack upgrade, and the pack's OTHER files still update — verified by upgrading a fixture pack, not by inspecting intent.
- **SC-003**: A fork whose source has moved on reports the newer version and exactly which phases the user changed.
- **SC-004**: No customization is stored inside a pack — asserted by diffing the pack directory after every authoring operation.
- **SC-005**: A pack declaring nothing customizable is fully editable anyway — its content through the overlay, its structure through a fork.
- **SC-006**: The picker offers workflows grouped by method; every pre-`051` `method`-keyed binding still resolves.
- **SC-007**: Deleting a bound workflow triggers the preflight and hides nothing silently.
- **SC-008**: `workflow_list` shows user workflows with their provenance and customization state.
- **SC-009**: `npm run test:unit` green; `npx tsc --noEmit` and `npm run lint` clean.
- **SC-010**: All three installed methods — spec-kit, BMAD, OpenSpec — render on the canvas with NO method-specific view code. Verified by rendering each, not by reading the renderer.
- **SC-011**: A structurally invalid edit is refused identically whether made on the canvas or by the agent — asserted by driving both paths at the same validator.
- **SC-012**: A pack's own workflow cannot have its STRUCTURE edited in place from the canvas; the attempt is refused with a `not-yours` code and offers Fork. Its prompts remain editable through the overlay. Asserted by diffing the pack directory, like SC-004.
- **SC-014**: Every method pack BOS ships **validates clean** — asserted against the real descriptors, not a fixture. This is the guard on the validator being too strict, and it earned itself: it failed on its first run and exposed a real defect in the OpenSpec pack (`artifacts[].requires` naming phase ids), which had been silently dropping four of five artifact dependencies from the derived graph.
- **SC-015**: An edit made against a stale revision is refused, naming both revisions, and writes nothing (FR-025).
- **SC-016**: Adding a gate reports what it would block and out of how many, identically on both paths, and a preview writes nothing (FR-026).
- **SC-017**: A fork survives its source pack being upgraded AND uninstalled with its binding still resolving — asserted against a REAL installed pack and a REAL store, not a sandbox. The binding half is not redundant with SC-001: it exercises manifest persistence, which is where the defect actually was.
- **SC-013**: A phase inspector shows only declared values. Asserted against a pack that deliberately declares a sparse phase — the absence must render as absence, not as a plausible default.

## Assumptions

- **One idiom, because the second does not pay for itself.** BMAD overrides and OpenSpec forks, and an earlier draft built both. The overlay turned out to serve content at file granularity with no patch machinery, and for structure a mechanical merge cannot handle the case that matters — a base that renamed what you changed. An agent re-applying a computed diff does that better and can say what it did. So: content overlaid, structure forked, upgrades agent-driven.
- **The pack owns the meaning; BOS owns the mechanism.** BOS renders an editor from a declaration and merges by stated rules. It never learns what a BMAD "activation step" is — that knowledge lives in the pack, as `049` established for driver skills.
- **Customizations outlive packs.** Upgrade, uninstall and reinstall are routine. Anything that does not survive all three will be lost by someone, silently.

## What implementation changed about this spec

Recorded so the next reader sees the corrections rather than rediscovering them.

- **A mechanism with no subject is not worth building.** Override died of it
  (no pack declares a surface) and FR-015's semantic constraints are deferred for
  the same reason. The test that catches this early is "run it against the packs
  that actually ship" — which is now SC-014 and which found a real bug on its
  first execution.
- **Measure the descriptors before reasoning about them.** The original FR-016
  said `requires` is the edge set and layout follows dependency depth. Both were
  false for two of the three installed packs, including the default, and no
  amount of design review caught it — rendering the real thing did (design R0).
- **`optional` and reorder were missing.** Neither was in the original
  requirements; both turned out to be load-bearing once declared order became the
  pipeline.
- **Two surfaces editing one thing needs a revision, and a warning neither can
  skip.** FR-025 and FR-026 were both absent, and both are properties of having
  two callers rather than of workflows.
- **The binding was assumed to work.** It did not: a store bound by `workflow`
  reverted to spec-kit on every boot, because the seed rewrote the store manifest
  from an allowlist that `049`'s field had never been added to. Nothing in this
  spec would have caught that; SC-017 exists so something does.
