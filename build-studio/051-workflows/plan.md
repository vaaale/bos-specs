# Implementation Plan: Workflows — a pipeline you can name, fork and own (051)

## Summary

Make a workflow something a user can **see**, **fork** and **own** — rather than a
fixed pipeline chosen by whoever wrote the pack.

Three parts, in dependency order: a renderer that shows what a pack actually
declares, storage and resolution for user-owned workflows, and one authoring path
shared by the canvas and the agent.

The picture comes first, and it comes with a stop.

> **AS BUILT.** "Override" is out of the summary because the mechanism was
> dropped mid-flight (design §3.4): no pack declares a surface, and 048's overlay
> already handles content edits at file granularity while keeping upstream
> improvements. The pair that survived is **content overlaid, structure forked**,
> with upgrades performed by an agent against a diff BOS computes.

## Technical context

| | |
|---|---|
| Depends on | `045` (descriptor, registry, preflight), `049` (workflow as the binding unit, `driverSkill`) |
| Independent of | `050` |
| Reuses | `registerMethod`'s validation gate; `045` FR-010's preflight; the Workflow app's layout maths (copied, §3.8) |
| Touches | `types.ts`, `registry.ts` (callback only), Build Studio's centre column |
| Mockup | `mockup.html` — renders all three packs from their real descriptors |

## Constitution check

- **Server decides, client displays** — edges, validity, merge and drift are
  computed server-side; the canvas renders what it is given and posts operations.
- **No workarounds** — a gate edit that would block live features says so before
  applying, with the number examined alongside the number blocked; a fork whose
  source has moved is reported rather than silently frozen.
- **One implementation** — canvas and agent share `applyWorkflowEdit` and one
  validator (FR-019). Two paths to one outcome is this subsystem's recurring
  defect.
- **Docs track features** — FR-014. Overlay-vs-fork is the thing a user cannot
  infer and will get wrong by default: forking to change one prompt trades a pack
  you never maintain for one you now do.

## Test strategy

1. **Assert against the REAL packs, not fixtures alone** (SC-010). The design's
   first draft was wrong precisely because it reasoned about descriptors instead
   of measuring three of them. Every edge-derivation test runs over spec-kit,
   BMAD and OpenSpec as shipped, and pins the counts:

   | pack | phases | gates | links (distinct pairs) | edge records | connected |
   |---|---|---|---|---|---|
   | spec-kit | 12 | 0 | 6 | 8 | 7 |
   | BMAD | 8 | 0 | 4 | 6 | 4 |
   | OpenSpec | 7 | 6 | 6 | 12 | 6 |

   If a pack changes, the test fails and someone looks — which is the point.

   > **AS BUILT — the numbers moved twice, and both movements were the point.**
   > spec-kit went 9 → 12 phases because looking at the render made the user
   > redefine the pipeline on the spot (design R0). OpenSpec's edge RECORDS went
   > 8 → 12 because this feature's validator found a defect in that pack: its
   > `artifacts[].requires` named PHASE ids where the schema says artifact ids, so
   > four of its five artifact dependencies were being silently dropped.
   >
   > **The distinct-pair column could not see that bug** — every dropped edge
   > duplicated a gate pair already counted — which is why the record count is now
   > pinned too. A table with the wrong granularity is a test that passes.

6. **Validate the REAL packs, not just the editor** (SC-014). A validator for user
   edits is easy to make too strict, and the way that fails is invisible: the rule
   looks sensible, nobody runs it against the shipped descriptors, and the default
   method turns out to be "invalid". Two candidate rules were dropped this way and
   one real pack bug was found.
7. **Drive BOTH transports in the same test** (SC-011). Not "the route has tests
   and the tool has tests" — one test, both paths, same fork, asserting they
   agree. Separate suites are what let a drift pass twice.
2. **Drive the path the product calls.** The canvas and the agent both reach
   `applyWorkflowEdit`; tests call that, not the helpers under it. Four defects in
   this feature family survived tests that exercised a reachable function the
   product does not use.
3. **Prove the negative by diffing the pack directory** (SC-004, SC-012). "We only
   write to `data/workflows/`" is exactly the claim that fails quietly.
4. **Upgrade a fixture pack for real** (SC-002). An overlaid PROMPT surviving an
   upgrade — while the pack's other files still update — cannot be asserted by
   inspecting intent; the fixture pack has to change version and content
   underneath it.
5. **Declare absence deliberately** (SC-013). A fixture pack ships a phase with no
   description so the inspector's "shows absence as absence" is testable rather
   than aspirational.

## Phase breakdown

| Phase | What | Gate |
|---|---|---|
| **1** | Derive the three edge kinds; render all three packs read-only | **STOP — look at it (R0)** |
| **1b** | A phase's INSTRUCTIONS: resolve overlay→pack, edit, revert | *unplanned* — arrived from looking at Phase 1 |
| 2 | `data/workflows/` storage, fork, registration | a fork survives pack uninstall |
| ~~3~~ | ~~Override: declared surface, merge, drift~~ | **DROPPED** — replaced by `baseline` + agent-driven re-fork |
| 4 | `applyWorkflowEdit` + the agent tool; canvas editing | canvas and agent hit one validator |
| 5 | Centre-column mode consolidation (R4) | no two panes can claim the column |
| 6 | Docs + a live run | fork, edit, bind, upgrade the source, nothing lost |

> **AS BUILT — the phase plan changed in two places, both at Phase 1's gate.**
>
> **1b appeared.** With the pipeline on screen, the first question was not "what
> depends on what" but "what does this step actually SAY" — and three of the
> twelve phases said nothing at all. It needed no fork and no patch format:
> 048's overlay is already "a write destination for user modifications to
> already-installed content", and extending it from agents to templates is the
> same precedence rule. That finding is what later made Phase 3 unnecessary.
>
> **3 was dropped**, not deferred. Its two premises were both false once measured
> — no pack declares an override surface, and the overlay already serves content
> — and what remained needed a mechanical merge onto a moved base, which cannot
> be done without judgement. Five modules became one field (`baseline`) and two
> functions. Deleted with it: the patch format, the `a.b`/`a[*].b` selector
> language, the merge-rule table, the generated override editor, and
> `kind: "override"`.

### Phase 1 ends in a decision, not a handoff

`R0`, agreed: spec-kit yields **4 edges over 9 phases**, and whether that picture
earns the rest of the feature is a judgement no amount of design settles. Phase 1
is deliberately shaped to answer it cheaply — derivation plus a read-only
renderer, no storage, no editing, no agent tooling.

Three outcomes, all acceptable:

- **The picture is good** → continue to Phase 2.
- **The picture is thin but the isolated-phase reporting is valuable** → ship
  Phase 1 alone as a *diagnostic* view ("which steps can BOS actually observe?")
  and defer the rest.
- **The picture is not worth it** → stop. Phase 2 (fork) still has independent
  value and can proceed without a canvas at all; the spec's US1–US3 never needed
  one.

Nothing in Phase 1 is wasted in any of the three: the derivation is what
`workflow_list` needs anyway (FR-013).

## Rollout

Additive. No existing descriptor changes, no migration: a pack that declares no
constraints behaves exactly as today, and the canvas is a new mode rather than a
replacement for anything.

> **AS BUILT — additive held, with two in-place changes beyond R4.**
> `seed.ts` stopped rewriting the store manifest from an allowlist (it was
> dropping `049`'s `workflow` binding on every boot); and the OpenSpec pack's
> `artifacts[].requires` was corrected. Both are fixes to existing defects rather
> than migrations, and both are covered by regression tests.

The one in-place change is R4's centre-column consolidation — behaviour-preserving
by construction, and worth doing while there are three arms rather than four.

## Risks

- **R0 — the picture may be too sparse to be worth it.** Agreed and sequenced:
  Phase 1 answers it before anything depends on the answer.
- **R1 — a fork's descriptor is user data hitting `registerMethod`.** A bad one
  must name itself and not abort the rest of registration. Register user
  workflows after packs.
- **R2 — ~~an override resolves only once its base has~~. MOOT.** Override was
  dropped; the re-resolve callback was never added, and `registry.ts` stayed plain
  data with no fs access.
- **R3 — id collisions with a pack's workflow.** `049` FR-004's rule: reported,
  never resolved by order.
- **R4 — a fourth arm on the centre column's ternary.** Four booleans, six
  pairwise interactions, silent failure. Replace the chain rather than extend it.
- **R5 — the gate edit is the dangerous one.** Adding `phases[].requires` where a
  pack deliberately has none can flip live features to `blocked`. Warned, counted,
  and never the default gesture.
  > **OUTCOME: real, and the counting mechanism this plan assumed does not
  > exist.** `045` FR-010's preflight compares DISCOVERY and reports nothing for a
  > gate change. The count needs a before/after EVALUATION of each bound unit —
  > twice, because gates are consulted only when no rule clause matched, so
  > current phase states cannot be used to derive it.
- **R6 — data-flow derivation must honour SCOPE, not just filename.** Measured
  across the three packs: 9 distinct predicate kinds (`exists`, `nonEmpty`,
  `checklist`, `contains`, `notContains`, `containsUnitId`, `count`, `set`,
  `all`) but a UNIFORM file shape — every one of the 34 references is
  `{rel, scope?, roots?}`. So the walk needs no predicate knowledge, which
  removes most of this risk.

  What remains is scope. spec-kit's `constitution` reads a **store-scoped** file
  in the `system` root; `converge` writes a **store-scoped** `discrepancies.md`.
  Matching on `rel` alone would draw edges between a store-scoped read and a
  unit-scoped write of the same name — a plausible-looking arrow that is simply
  false. Compare `(rel, scope)`, and treat an unrecognised file shape as "no
  edge, and say so" rather than as an edge.

## What the plan got wrong, in retrospect

Kept because the pattern repeats, not to keep score.

1. **A phase gate that ends in "look at it" was the single most valuable line in
   this plan.** Phase 1's stop produced three things no review had: the pipeline
   was wrong and got redefined on the spot, order turned out to be stored in two
   disagreeing places (§3.1c), and Phase 1b was discovered — which then made
   Phase 3 unnecessary. The lesson is not "R0 was a good risk"; it is that a
   render-and-stop is cheap and design review is not a substitute for it.

2. **Two of the six phases were wrong about their own premise**, and in the same
   way: Phase 3 assumed packs declare override surfaces, and the test strategy
   assumed `requires` is the edge set. Both were checkable against three files on
   disk before either was written. "Measure the shipped artifacts first" would
   have saved most of the design.

3. **The plan reused a mechanism it had not read.** R5 and T025 both say "use
   `045` FR-010's preflight". That function compares discovery and cannot answer a
   question about phase state. Naming a component to reuse is a claim worth
   checking at plan time — the check is one file read.

4. **Nothing in the plan covered the boundary it actually broke.** The seed
   rewrites the store manifest on every boot from an allowlist, and `049`'s
   binding field had never been added to it, so every fork binding reverted
   silently. No phase, risk or test in this plan pointed at that path; it was
   found while writing an unrelated test. SC-017 now asserts it end to end.

5. **Two requirements were missing because they are properties of having two
   callers**, not of workflows: a revision on every edit (FR-025) and a warning
   neither transport can skip (FR-026). Worth a standing question for any feature
   with a canvas and an agent over the same state.
