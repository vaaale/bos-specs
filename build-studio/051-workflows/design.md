# Design: Workflows — a pipeline you can name, fork and own (051)

## 1. What is actually new

Less than the spec's length suggests. A workflow is **already** a graph in every
descriptor — `phases[]` are nodes, `requires` are edges — and `049` already made
a workflow the binding unit, with `<method>:<workflow>` addressing and a
`driverSkill` per workflow. Nothing about the model changes.

What is new is that a workflow can now come from somewhere other than a pack, and
that it has a surface.

| | |
|---|---|
| storage | user-owned workflows under `data/workflows/` (new) |
| registry | one more source feeding `registerMethod`, not a second registry |
| resolution | ~~an override merges over its base when the base registers~~ — **dropped (§3.4)**; a fork registers as-is and owes its source nothing |
| authoring | one `applyWorkflowEdit` behind both the canvas and the agent (new) |
| Build Studio | a canvas mode in the centre column (new) |

## 2. Constitution check

- **Server decides, client displays** — validity, merge and drift are decided
  server-side; the canvas renders a graph it is given and posts operations.
- **No workarounds** — a fork whose source has moved is REPORTED (FR-008),
  never dropped to keep the happy path quiet.
- **One implementation** — the canvas and the agent share one mutation path and
  one validator (FR-019). Every divergence in this subsystem so far has come from
  two paths to one outcome.
- **Docs track features** — FR-014; overlay-vs-fork is the thing a user cannot
  infer and will get wrong by default.

## 3. Decisions

### 3.1 The graph is DERIVED from three declared sources — not from `phases[].requires`

An earlier draft of this design said "a workflow already IS a graph: `phases[]`
are nodes, `requires` are edges." That is false for two of the three installed
packs, and measuring it is what corrected the rest of this document:

| pack | phases | `phases[].requires` |
|---|---|---|
| **spec-kit** (the default) | 9 | **0** |
| **BMAD** | 8 | **0** |
| OpenSpec | 7 | 6 |

`phases[].requires` is a HARD GATE — `evaluate.ts` makes a phase `blocked` when
one is unsatisfied — and two packs deliberately leave it empty. A test exists to
keep it that way: *"requires: [] on ALL NINE — gating lives in the clauses. Any
edge makes `blocked` reachable, and under the linear edges PHASE_ORDER suggests
that flips 120 live features na -> blocked."*

The structure is real; it is just declared in three places, and the graph is the
UNION of them:

| # | Edge | Declared in | Enforced? |
|---|---|---|---|
| 1 | **Gate** | `phases[].requires` | **Yes** — unsatisfied ⇒ `blocked` |
| 2 | **Artifact dependency** | `artifacts[].requires` + `generates` | No — an ordering the pack states |
| 3 | **Data flow** | a phase's `rules` naming a file another phase `generates` | No — entailed by what the pack declared |

Source 3 is *derived*, not invented: the rule literally names a file, and another
artifact literally names the phase that writes it. Joining two declared facts is
not the synthesis FR-020 forbids — but it is not a pack's own statement either,
so it renders differently (§3.1a) and is never editable.

What that yields, measured:

| pack | gates | artifact deps | data flow | links | phases with a dependency |
|---|---|---|---|---|---|
| spec-kit | 0 | 2 | 6 | **6** | 7 / 12 |
| BMAD | 0 | 3 | 3 | **4** | 4 / 8 |
| OpenSpec | 6 | 1 | 1 | **6** | 6 / 7 |

*(spec-kit's numbers are post-051: its pipeline gained `ui-design`, `design` and
`review`, taking it from 9 phases to 12. The shape of the finding is unchanged —
it still declares zero gates.)*

**A phase with no dependency is a real and correct result, not a rendering
failure.** spec-kit's `analyze` declares `rules: []` and `else: "na"` — it never
reports automatically, by design. `constitution` reads a store-scoped file no
phase produces. `converge`, `test` and `ui-design` write artifacts nothing else
consumes. Reporting that, with the reason, is more truthful than inventing a
spine, and it surfaces something currently invisible: which steps of your pipeline
BOS can actually observe.

They are MARKED, not moved — see §3.1c. An earlier build exiled them below the
graph, which put `ui-design` adrift even though the pack lists it fourth.

### 3.1a The canvas distinguishes what is ENFORCED from what is merely declared

Three edge kinds, three renderings — because the difference is exactly what a
user needs and cannot currently see anywhere:

- **Gate** — solid, arrowed. Adding one CAN block phases. Editing a gate is a
  deliberate, warned action (§3.6a).
- **Artifact dependency** — solid, lighter. The pack's stated ordering.
- **Data flow** — dashed, labelled with the file. Derived; **read-only** on the
  canvas. You change it by changing the rule or the artifact, not by dragging.

Without this distinction the canvas would show four indistinguishable arrows in
spec-kit, of which zero are enforced, and invite the user to add a fifth that is.

### 3.1b The canvas is a VIEW; the descriptor stays the only model

No derived graph is persisted. Edges are computed on read and thrown away; the
canvas posts operations against the descriptor's own fields. A cached graph would
need invalidating on every rule change, and the pipeline evaluator would keep
reading the descriptor regardless.

### 3.1c Order comes from `phases[]`, and it is a POSITION, not an edge

The fourth declared source, and the one the first draft missed entirely.

`phases[]` is itself a declaration — `PhaseStrip` renders it verbatim, and it is
what a user means by "the pipeline". Measured against spec-kit's declared
sequence, only **2 of its 11** consecutive steps are backed by a dependency:

```
     constitution -> specify        DEP  specify -> clarify
     clarify      -> ui-design           clarify -> ui-design
     ui-design    -> design         DEP  plan    -> tasks
     design       -> review              ...9 of 11 are true only because
     review       -> plan                they are adjacent in an array
```

It is deliberately NOT returned as edges. An array declares a TOTAL ORDER, not
pairwise relationships, and emitting it as edges over-claims: OpenSpec lists
`specs, design, ui` consecutively and all three gate on `proposal` — they are
PARALLEL. A "follows" edge between them would assert a sequence the pack never
declared and make parallel work look serial.

So **order is conveyed by position** (the route returns each phase's index; the
canvas lays out left-to-right by it) and **dependencies are drawn over it as
arcs**. The canvas and the strip then show one truth at two zoom levels rather
than two views that disagree about where a phase belongs.

A phase nothing depends on is MARKED (dashed), never MOVED. `ui-design` is the
fourth step because the pack lists it fourth, whether or not anything links to
it — and a layout derived from dependencies put it adrift, which is what exposed
this.

### 3.2 A user workflow is one directory with one file

```
data/workflows/<id>/workflow.json
```

Under `data/`, because that is where per-user mutable state lives (`data/agents`,
`data/skills`) and because FR-005 requires surviving pack upgrade, uninstall and
reinstall — all three of which rewrite or remove the pack.

One file with a **discriminant**:

```jsonc
{
  "kind": "fork",
  "from": "bmad", "fromVersion": "6.13.0-next",
  "descriptor": { … },   // complete and yours, including every edit since
  "baseline":   { … },   // the source EXACTLY as taken, never touched again
  "rev": 3               // bumped on every write (§3.6b)
}
```

> **AS BUILT.** The union has one arm. `kind: "override"` was designed here and
> is gone with §3.4 — a type variant nothing produces is exactly the
> half-finished thing that misleads whoever reads it next. The discriminant
> stays, because it is what lets a second kind be added later without every
> reader having to guess.
>
> `baseline` was not in this design and is the thing that made dropping override
> possible. It is the source as taken, so `diff(baseline, descriptor)` is
> **precisely the user's own edits**, separable from what the base already said.
> The delta is DERIVED rather than authored, which is why 051 needs no patch
> format, no path selectors and no merge rules.

### 3.3 A fork owes nothing to its source

> **AS BUILT — this section described fork AND override; override is gone (§3.4).**
> Everything below about forks held. Everything about re-merging on re-registration
> was never built, and the `onMethodRegistered` callback it required does not exist.

`descriptor` is complete and registered as-is. `from`/`fromVersion` exist only
for drift *reporting* (FR-008), never for resolution — which is what makes the
property testable: a fork keeps working when its source pack is **uninstalled**
(SC-001), verified live in T028 and in `workflows-api.test.ts`.

A fork goes through the same `registerMethod` gate a pack does, AFTER packs, and
a malformed one names itself without aborting the rest (R1).

### 3.4 ~~The merge rule follows the VALUE~~ — DROPPED, and why

> **AS BUILT.** This whole section is struck. Override does not exist: no
> `overrides` surface, no path selectors (`a.b`, `a[*].b`), no merge-rule table,
> no generated override editor, no `kind: "override"`.

Two findings killed it, and neither was visible when it was designed:

1. **No pack declares an override surface IN BOS'S `overrides` FIELD** — not
   spec-kit, not BMAD, not OpenSpec. The rule this section itself states ("a pack
   declaring no `overrides` supports forking only") made the feature **inert on
   everything shipped**.

   > **CORRECTION (found later, from BMAD's own docs).** The first draft of this
   > sentence said "no pack declares an override surface" flatly, and that is
   > FALSE. BMAD declares a real, documented one — a `customize.toml` per skill,
   > with a three-layer merge (`_bmad/custom/<skill>.user.toml` > `_bmad/custom/
   > <skill>.toml` > the shipped `customize.toml`) and merge rules by data shape.
   > Those files are sitting in the installed pack, headed `DO NOT EDIT`.
   >
   > What is true is the narrower claim now written above: nobody fills the field
   > BOS invented. And the correction STRENGTHENS the decision rather than
   > weakening it — a pack that already ships its own override mechanism is the
   > worst possible candidate for a second one layered underneath, which is
   > exactly what §3.4 would have built. Two competing merges over one file, and
   > "which version of this am I running" acquires two answers.
   >
   > See also the open gap this exposed: BOS runs none of BMAD's merge, so that
   > surface is currently inert inside BOS.
   It would have been a mechanism with no subject.
2. **The overlay already serves content.** Phase 1b (`data/method-packs/<id>/`,
   048) edits a prompt, a template or an agent at FILE granularity, keeps every
   other file's upstream improvements, and needs no patch format at all. That is
   the case this section was mostly for.

What remained was *structural* override, and it only pays for itself if
re-applying a change onto a moved base can be done mechanically. **It cannot.** A
base that renames a phase needs judgement about whether the user's change still
means the same thing. A merge engine answers that silently; an agent answers it
and says what it did.

So an upgrade is **"fork the new version and re-apply my changes"**, performed by
an agent against a diff BOS computes from `baseline` (§3.2). That needed one new
field and two functions instead of five modules.

This section's own closing paragraph turns out to have been the argument against
itself: it says an override cannot REMOVE anything, so *"BMAD without the PRD" —
this feature's own motivating example — is a FORK*. The motivating example never
needed the mechanism.

### 3.5 Validity splits at "malformed" versus "wrong"

| | Enforced by | Examples |
|---|---|---|
| **Structural** | BOS, always | a cycle in the GATE edges; `requires` naming a phase or artifact that does not exist |
| **Semantic** | the pack, as declared data | a phase that cannot be removed; an ordering that must hold |

**Two rules an earlier draft listed are wrong and are struck.** "An artifact no
phase generates" is normal — spec-kit ships `research.md`, `data-model.md` and
`quickstart.md` with no `generates` at all. "A phase nothing can reach" is normal
too: 4 of spec-kit's 9 phases are isolated by design (§3.1). Both would have
flagged the default method as malformed. Only CYCLES and DANGLING REFERENCES are
safe to enforce universally — everything else varies by pack, which is precisely
why it belongs in the pack's own declaration.

> **AS BUILT — the cycle rule here is WRONG, and was corrected in `validate.ts`.**
>
> This section said *"cycles are checked over gate edges only. A cycle in derived
> data-flow edges is possible and harmless."* The first half is a bug and the
> second half is right for the wrong reason.
>
> `evaluate.ts`'s `resolve()` recurses through `phases[].requires` **and** through
> every `dependsOn` predicate, and its `inFlight` guard throws on either. So a
> cycle made of `dependsOn` clauses — with no gate anywhere in it — fails for
> every unit in the store the moment the workflow is bound. A gates-only check
> passes that descriptor. Cycles are now checked over gates ∪ `dependsOn`.
>
> Data-flow edges are still excluded, and that part stands: they are DERIVED for
> display and nothing evaluates them.
>
> The semantic half of the table was never built. No pack declares `required` or
> `mustFollow`, so it would have been the same inert mechanism as §3.4. Structural
> validation alone shipped, and a pack that wants to object still cannot — the
> honest state, recorded rather than implied.

> **AS BUILT — the two struck rules were struck for the right reason, and the
> guard that proves it caught a real defect.** The test written for this
> paragraph is "every pack BOS ships validates clean", and it **failed on its
> first run**: OpenSpec's `artifacts[].requires` named PHASE ids where the schema
> says artifact ids, so `graph.ts` resolved each against a map keyed by artifact
> id, found nothing, and silently dropped four of that pack's five artifact
> dependencies. Fixed in the pack. The existing graph test could not see it,
> because it pinned distinct PAIRS and every dropped edge duplicated a gate pair
> already counted; it now pins edge records too.

### 3.6 One mutation path, for the canvas and the agent alike

```
canvas  ──PATCH /api/workflows─┐
                               ├──> applyWorkflowEdit(id, op) ──> validate ──> write
agent   ──workflow_edit tool───┘
```

`applyWorkflowEdit` takes a small closed set of operations — `addPhase`,
`removePhase`, `renamePhase`, `movePhase`, `setRequires`, `setOptional`,
`setArtifacts` — and every one runs the same validator before anything is
written.

> **AS BUILT — the op set moved in three ways.**
> - `setOverride` is **gone** with §3.4.
> - `movePhase` is **added**. §3.1c established that `phases[]` IS the pipeline —
>   position, not edges, is what the canvas lays out by and what a user means by
>   the order of steps. Without a reorder op a fork can change everything about a
>   workflow except the one thing most people fork to change.
> - `setOptional` is **added**. `optional` became a declared field during this
>   feature (`[Clarify]`, `[UI Design]`) and nothing could set it.
>
> Also as built, and not designed here: `applyWorkflowEdit` takes a **`dryRun`**
> flag rather than there being a separate preview endpoint. Same op, same
> validator, same refusals — so what a preview promises is literally what will
> happen, instead of a second code path guessing at it.
>
> And an op that does more than it was told **says so**: `removePhase` returns
> the gates it dropped and the artifacts it orphaned as warnings. Without that it
> would have to either refuse on the first dangling reference (too strict — you
> could not remove a phase anything referenced) or change things silently.

This is `049`'s `lifecycle.ts` lesson applied up front: that module exists
because the agent tools and the Build Studio context menus had drifted, and its
own header says two paths to one outcome is the shape every divergence in this
subsystem has taken. FR-019 is that rule stated as a requirement; this is the
structure that keeps it.

An **operation** rather than a whole-descriptor PUT, because a PUT cannot
distinguish "the user removed a phase" from "the user is working off a stale copy
and is about to clobber the agent's edit."

### 3.6a The same operation means different things on a fork and a PACK'S

> **AS BUILT — the branch is fork vs PACK, not fork vs override.** Override is
> gone (§3.4), so the table below has no second column. What replaced it:

| | Fork | A pack's workflow |
|---|---|---|
| every structural op | applies | **refused** (`not-yours`), naming forking as the route |
| the phase's PROMPT | editable | **editable too** — that is the overlay, and it needs no fork |

That second row is the part worth keeping from this section's spirit. The
read-only surface is the *structure*, not the workflow: telling a user "fork this
to change the review prompt" would send them to the expensive mechanism for the
cheap need.

A refusal is a normal outcome, not an error — and it carries a **code**, so the
route returns 409 with `not-yours` / `stale` / `invalid` / `no-such-phase` /
`unsupported` and the UI can offer *Fork this* or *Reload* rather than printing a
sentence. Two more refusals were needed once the ops were real, both for the same
reason — BOS would have to guess:

- `removePhase` when a rule `dependsOn` the phase. Dropping the clause changes
  what the surrounding condition means. The refusal names who depends on it.
- `setArtifacts` naming an artifact the workflow never declared. Inventing it
  would mean guessing a scope and a file name.

**Editing a GATE is warned, once.** Adding a `phases[].requires` edge can flip
existing features to `blocked` — 120 of them, in the case the test in §3.1 guards
against. The operation is allowed (it is the user's workflow) but the canvas says
what it will do, and to how many units, before applying it.

> **AS BUILT — the last sentence was wrong twice.**
>
> **1. `045` FR-010's preflight cannot answer this.** `preflightMethodChange`
> compares DISCOVERY — which units a descriptor can still find — and a gate
> changes no leaf marker, so it reports *nothing at all*. The question here is
> about phase STATE, which needs the evaluator. `gate-impact.ts` is a separate
> computation; the two share the unit walk (`walkUnits`) and nothing else.
>
> **2. Deriving it from current states is also wrong.** The tempting shortcut is
> "phase X is currently `pending`, so gating it on an undone phase blocks it".
> But gates are consulted ONLY when no rule clause matched, so a phase whose rules
> already answer is unaffected by any gate. Each bound unit is therefore evaluated
> **twice** — once under what is bound today, once under the proposed descriptor.
> A derivation would be right most of the time, which is worse than being right.
>
> **3. The count needs its denominator.** "0 units would be blocked" and "nothing
> is bound to this workflow, so nothing was checked" are the same number and
> opposite facts. `examined` is reported alongside, and an unbound fork says it
> was not checked rather than implying safety. Draft-branch units are counted as
> not-examined rather than quietly omitted.
>
> **4. It is computed inside `applyWorkflowEdit`, not by the canvas.** This
> section puts the warning in front of the Apply button — which would mean an
> agent calling `workflow_edit` never saw it. That is precisely the
> two-paths-one-outcome shape §3.6 exists to prevent, so both transports render
> the same field.

### 3.6b Every edit carries the revision it was made against

The stale-copy risk above is the reason for operations, but operations alone do
not detect it. `workflow.json` carries a `rev` that increments on every write;
`applyWorkflowEdit` takes the `rev` the caller last saw and refuses a mismatch,
naming what changed. Without it, "the user and the agent both edited" resolves by
whoever wrote last — silently, which is the failure this subsystem keeps
producing.

### 3.7 Edges are edited as a LIST first; dragging is a later refinement

**The open question from the mockup, resolved.**

Stronger now than when it was first argued. Of the three edge kinds (§3.1a),
**only gates are editable at all** — artifact dependencies are edited on the
artifact, and data-flow edges are derived and read-only. So drag-to-connect would
apply to the single rarest edge kind, which is also the one that needs a warning
before it applies (§3.6a) — a modal confirmation at the end of a drag is a poor
interaction in any case.

The inspector's "Requires" chips and a drag between two ports produce the
identical operation — `setRequires(phase, [...])`. Dragging is a second input
method for one operation, not additional capability. So:

- **Now**: the canvas renders and selects; the inspector edits. Every operation in
  §3.6 is reachable.
- **Later (P2)**: drag-to-connect as sugar over the same op.

> **AS BUILT — held, and the gate list earned its preview.** Every op is
> reachable from the inspector: rename (label and id), optional, position, gates,
> remove, plus Add phase in the header. Gates are the only control that previews
> first, and the confirmation appears **only when there is something to be asked
> about** — a confirmation that always appears is one nobody reads.
>
> One thing this section did not anticipate: after an applied edit the canvas
> **re-reads the server's graph** rather than patching its own copy. The edges are
> derived from three separate declarations (§3.1), and a client that recomputed
> them would be the second derivation this whole feature exists to avoid.

Sequenced this way because port geometry, hit targets, drag state, and
cancel/escape handling are real work that shares nothing with the rest of BOS —
the Workflow app's own `GraphEditor.tsx` is 654 lines against `Graph.tsx`'s 208,
and that ratio is the honest estimate. The canvas's primary job in this feature
is making an agent's edit **reviewable at a glance** (US5), and it does that fully
while read-only.

Revisit once the feature is in use: if users reach for the canvas to build rather
than to review, dragging earns its cost.

### 3.8 The renderer copies the Workflow app's visual language, not its code

`items/workflows/app/src/components/Graph.tsx` already solves exactly this
picture: layered layout by dependency depth, 140×52 nodes, bezier edges, and
**opaque node fills** — which exist because translucent ones let the SVG edge
layer show through the node, a bug that component was fixed for.

BOS cannot import it: it lives in a marketplace item, in a different repo, over a
different type (`WorkflowStep`, not `PhaseSpec`).

So this is a deliberate second implementation of ~80 lines of layout maths, and
it should be called that rather than dressed up. The alternative — extracting a
shared graph package across two repos — is heavier than the thing being shared,
and the shared thing is a *visual language* (which must match) rather than logic
(which must not diverge). The constants and the layering rule are carried over
verbatim, with a comment pointing at the original.

### 3.9 Drift is reported, never merged

A fork records `fromVersion`. When the pack's version moves, BOS reports it —
and does nothing else.

> **AS BUILT.** The override half of this section is gone with §3.4. For a fork
> drift is NOT "purely informational", which is what this section said: without
> the notice a fork is a **silent freeze**, and the user believes they are current
> when the source has moved on twice. So it is surfaced in three places — the
> canvas banner ("bmad is now @ 6.14"), `forkStatus`, and the
> `workflow_fork_status` tool, which also lists which phases the user added or
> removed and instructs re-applying **with judgement**.
>
> A fourth state exists and is reported rather than treated as an error: the
> source pack is **uninstalled**, so there is no version to compare against and
> the fork is not "behind" anything.

## 4. Risks — and what happened to each

- **R0 — the picture may still be too sparse to be worth it.** spec-kit yields 4
  edges over 9 phases. §3.1 argues that is the truth and worth showing; that is a
  judgement, and the cheap way to test it is to render all three packs before
  building any editing. Sequence it first.
  > **OUTCOME: answered YES, and the render paid for itself twice over.** Two
  > findings came out of looking at it that no amount of design had produced: the
  > pipeline shown was immediately wrong for the user, who redefined spec-kit on
  > the spot; and the picture exposed that BOS stored order in two places that
  > disagreed, which became §3.1c. It also produced Phase 1b — with the pipeline on
  > screen the first question was not "what depends on what" but "what does this
  > step actually SAY", and three of the twelve phases said nothing.

- **R1 — a fork's descriptor must pass `registerMethod`.** It is user-editable
  data reaching the same gate as a pack's. Good — but the failure must name the
  workflow and stay recoverable, not refuse at boot and hide the file. Register
  user workflows AFTER packs, and report a bad one without aborting the rest.
  > **OUTCOME: built as designed**, and `applyWorkflowEdit` re-registers through
  > the same gate after every write, so an edit that validates but fails
  > registration surfaces immediately rather than at the next boot.

- **R2 — re-resolution order.** An override registers only once its base has.
  > **OUTCOME: MOOT.** Override is gone (§3.4). The `onMethodRegistered` callback
  > this risk required was never added, and `registry.ts` stayed plain data with no
  > fs access — which was the property §3.3 was protecting in the first place.

- **R3 — id collisions** between a user workflow and a pack's.
  > **OUTCOME: built as `049` FR-004 says — REPORTED, never resolved by order.** A
  > fork whose id a pack later takes does not register, and says so by name rather
  > than silently winning or silently losing.

- **R4 — the canvas is a fourth mode in an already-crowded centre column.**
  Replace the nested-ternary chain with one resolved mode as part of this feature.
  > **OUTCOME: done (T026), `centre-mode.ts`.** One pure total function with a
  > stated precedence ("interruption beats intent"), behaviour-preserving for the
  > three existing modes.

- **R5 — deleting a bound workflow** (FR-010). `045` FR-010's preflight exists;
  this must call it rather than grow a second confirmation.
  > **OUTCOME: the premise was wrong, and the same wrong premise appears in T025.**
  > `045`'s preflight compares DISCOVERY, which answers "would this hide content",
  > not "would this block a phase". It is the right tool for a METHOD change and
  > the wrong one here. `gate-impact.ts` is the second computation this risk was
  > trying to avoid — justified, and sharing the unit walk so the leaf rule still
  > has one implementation.

- **R6 — a file reference the derivation cannot interpret** (added during
  implementation). A sparser graph presented as fact is the failure mode, so
  `workflowGraph` matches on `(rel, scope)` and never `rel` alone, and reports
  what it could not read instead of dropping it.
  > **OUTCOME: this is what caught the OpenSpec pack defect** — see §3.5.

## 5. Risks that materialised and were NOT in this design

Recorded because each was invisible from the design and cost real time.

- **A store bound by WORKFLOW reverted on every boot.** `ensureUserStore`
  rewrites the store manifest from a constant plus an ALLOWLIST of preserved
  fields. `045` FR-008 added `method` to that list because a binding otherwise
  "would survive exactly until the next restart and then silently revert to
  spec-kit". `049` then made `workflow` the field that SUPERSEDES `method` — which
  is how every 051 fork is bound — and never added it. The same bug, via the field
  that replaced the one it was fixed for. The allowlist is gone: the manifest is
  preserved by default and only the three identity fields are enforced. This
  design assumed binding worked and never questioned it.

- **OpenSpec's `artifacts[].requires` named phase ids** where the schema says
  artifact ids, silently dropping four of five artifact dependencies from the
  derived graph (§3.5). A pack defect, found by BOS's validator.

- **`.catch(() => [])` in `hashDirectory`** made unequal assets compare EQUAL, and
  surfaced only as an intermittent test failure. Unrelated to 051 in subject and
  entirely related in kind — it is the same "absence and failure are the same
  answer" defect the workflow code guards against. A lint rule now fails the build
  on the shape.

## 6. Files — as built

```
src/lib/specs/method/graph.ts             derive the 3 edge kinds from a descriptor
src/lib/specs/method/user-workflows.ts    data/workflows/, fork, register, forkStatus
src/lib/specs/method/authoring.ts         applyWorkflowEdit + the closed op set
src/lib/specs/method/validate.ts          structural validation (was graph-validate.ts)
src/lib/specs/method/gate-impact.ts       what a new gate would block          (NOT designed)
src/lib/specs/method/instructions.ts      read/write/revert a phase's prompt   (Phase 1b)
src/lib/specs/method/types.ts             `instructions`, `optional` on PhaseSpec
src/lib/specs/method/preflight.ts         `walkUnits` exported, so the leaf rule
                                          still has ONE implementation
src/lib/specs/pipeline.ts                 `evaluateUnitWith` — evaluate a unit under
                                          an ARBITRARY descriptor, the seam T025 needs
src/lib/specs/seed.ts                     manifest preserved by default (the bug above)
src/app/api/workflows/route.ts            GET list / detail / instructions,
                                          PUT prompt, PATCH structure, POST fork, DELETE
src/lib/assistant/tools/server/specs.ts   workflow_edit, workflow_fork,
                                          workflow_fork_status, workflow_phase_instructions[_set]
src/apps/build-studio/workflow/WorkflowCanvas.tsx   canvas, inspector, structure editor, fork dialog
src/apps/build-studio/centre-mode.ts      ONE resolved mode, replacing the ternary (R4)
```

**Not built, and why:**

```
src/lib/specs/method/overrides.ts         §3.4 dropped — no pack declares a surface
registry.ts onMethodRegistered()          only override needed it (R2)
src/lib/assistant/tools/server/workflow-edit.ts
                                          the tool lives with the other workflow_*
                                          tools in specs.ts rather than alone
```

Tests: `workflow-graph`, `workflow-validate`, `workflow-authoring`, `gate-impact`,
`workflow-edit-one-path`, `workflows-api`, `phase-instructions`, `user-workflows`,
plus the seed regression for the binding bug.
