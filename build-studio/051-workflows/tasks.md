# Tasks: Workflows (051)

Ordered so the cheapest answer to the biggest open question comes first, and
nothing depends on that answer before it exists.

> Every state in the approved `mockup.html` has a task here. An earlier draft
> covered the mechanism thoroughly and the surface barely — four of six mockup
> states had nothing that would build them, and a design's UI does not exist
> until a task says so.

## Phase 1 — see what the packs actually declare (T001 … T006)

> **This phase ends in a decision (R0, agreed).** Deliberately cheap and
> self-contained: derivation, one read endpoint, a read-only canvas. No storage,
> no editing, no agent tooling. Nothing here is wasted whichever way the decision
> goes — `workflow_list` needs the derivation regardless (FR-013).

- **T001 — DONE: `graph.ts`: derive the three edge kinds from a descriptor (FR-016).**
  Gate (`phases[].requires`), artifact dependency (`artifacts[].requires` joined
  through `generates`), data flow (a phase's `rules` naming a file another phase
  generates).
  *Acceptance:* pure, no I/O — a descriptor in, typed edges out.
  *Guard (R6):* match on `(rel, scope)`, never `rel` alone. spec-kit reads a
  store-scoped constitution and writes a store-scoped `discrepancies.md`, and
  name-only matching invents an arrow between the two. A file shape it does not
  recognise yields NO edge and is reported, not silently dropped.

- **T002 — DONE: pin the derivation against the REAL packs (SC-010).**
  | pack | phases | gates | links (distinct pairs) | edge records | connected |
  |---|---|---|---|---|---|
  | spec-kit | 12 | 0 | 6 | 8 | 7 |
  | BMAD | 8 | 0 | 4 | 6 | 4 |
  | OpenSpec | 7 | 6 | 6 | 8 | 6 |

  *Acceptance:* those numbers, against the descriptors as shipped. The design's
  first draft was wrong because it reasoned about descriptors instead of measuring
  them; this is the test that would have caught it.
  *Guard:* BMAD and OpenSpec ship in the marketplace repo — read them through the
  installed-pack path, and SKIP WITH A MESSAGE when absent rather than passing
  vacuously.

- **T003 — DONE: the isolated-phase report.** A phase in no edge, with the REASON:
  declares no rules (`analyze`), reads a file no phase writes (`constitution`),
  writes a file nothing reads (`converge`, `test`).
  *Acceptance:* spec-kit reports exactly those four, each with its reason. This
  may turn out to be the feature's real value.

- **T004 — DONE: `GET /api/workflows` and `GET /api/workflows/<id>`.** The list, and one
  workflow's derived graph + isolated-phase report + `rev`.
  *Why its own task:* the canvas is a client component and cannot reach `graph.ts`
  directly. Without this, Phase 1 has no way to run end to end — the omission that
  prompted this rewrite.
  *Acceptance:* the server computes edges and validity; the client receives them
  and computes nothing (constitution check §2).

- **T005 — the read-only canvas. DONE.** Laid out by DECLARED ORDER with
  dependencies as arcs over it (§3.1c) — not by dependency depth, which scattered
  a pipeline whose order is nine-tenths convention and put `ui-design` adrift
  though the pack lists it fourth. Three edge renderings (§3.1a); a phase with no
  dependency is marked dashed and KEEPS ITS PLACE; the inspector shows only
  declared values (FR-020).
  *Guard:* opaque node fills — a translucent fill lets the edge layer show through
  the node, the bug the Workflow app's `Graph.tsx` was fixed for.

- **T006 — render all three packs and LOOK (R0). ANSWERED: continue.**
  The canvas was mounted and all three packs rendered live. The judgement was made
  against the real thing, and two findings came out of it that no amount of design
  had produced:

  - the pipeline shown was immediately wrong for the user, who redefined it on the
    spot (spec-kit is now 12 phases, `Specify -> [Clarify] -> [UI Design] ->
    Design -> Review -> Plan -> Tasks -> Implement`, keeping the other four);
  - the picture exposed that BOS stores order in two places that disagree, which
    became §3.1c.

  It also produced the requirement in Phase 1b: a phase's INSTRUCTIONS matter more
  than its wiring, and three of the new phases had none.

## Phase 1b — what a phase TELLS THE AGENT TO DO (T029 … T032) — DONE

> Unplanned, and it arrived from T006: with the pipeline on screen the first
> question was not "what depends on what" but "what does this step actually say".
> Three of the twelve phases said nothing at all.
>
> It needed NO fork and NO override patch, which is the finding worth carrying
> into Phase 2: `data/method-packs/<id>/` (048's pack overlay) already exists as
> "a write destination for user modifications to already-installed content", and
> extending it from agents to templates is the same precedence rule.

- **T029 — `PhaseSpec.instructions`, DECLARED not inferred (FR-021). DONE.**
  spec-kit ships `templates/commands/<phase-id>.md` and matching that convention
  would be BOS guessing at pack internals. It declares the 8 it has; the 4 it does
  not stay honestly empty.
- **T030 — resolve overlay → pack, and report which won (FR-023). DONE.**
- **T031 — write ONLY to the overlay; revert deletes the user's copy (FR-022).
  DONE.** Asserted against the pack directory, which is untouched.
- **T032 — the inspector edits it, and the agent gets the same two operations
  through the same functions (FR-019). DONE.**

## Phase 2 — a workflow you own (T007 … T013)

> **Reframed by Phase 1b.** The overlay turned out to serve "change this pack's
> content for me" completely, so Phase 2 is no longer the only route to a
> customization — and the two are not competitors:
>
> | | overlay (`data/method-packs/<id>/`) | fork (`data/workflows/<id>/`) |
> |---|---|---|
> | scope | changes the pack **everywhere** it is used | a **new named** workflow |
> | binding | none — it IS that pack | bound per store or project |
> | use | "our `review` prompt says this" | "BMAD-without-the-PRD, for these two repos only" |
>
> Fork remains necessary for the second column and only that. Anything that is
> really "I want this pack to behave differently" should reach for the overlay,
> which is cheaper and keeps upstream improvements.



- **T007 — `data/workflows/<id>/workflow.json`. DONE.** One file, one
  discriminant (`kind`), plus `rev` (§3.6b) and `baseline` — the source exactly as
  taken, which is what makes an upgrade's delta computable. The union has ONE arm:
  `"override"` went with Phase 3, because a type variant nothing produces is
  exactly the half-finished thing that misleads whoever reads it next.
- **T008 — fork (FR-002).** Copy the resolved descriptor; record `from` /
  `fromVersion` for reporting only, never for resolution.
  *Acceptance:* unchanged by upgrading its source, and still working when the
  source pack is UNINSTALLED (SC-001).
- **T009 — register user workflows (FR-009)** through the same `registerMethod`
  gate a pack goes through, AFTER packs.
  *Guard (R1):* a malformed user descriptor names itself and does not abort the
  rest of registration — user data reaching a gate built for pack data.
- **T010 — `workflow_list` reports user workflows (FR-013)** with provenance and
  customization state.
- **T011 — no customization is written inside a pack (SC-004).**
  *Acceptance:* diff the pack directory after every authoring operation. "We only
  write to `data/workflows/`" is exactly the claim that fails quietly.
- **T012 — the Fork-vs-OVERLAY dialog (FR-004). DONE.** Framed by CONSEQUENCE,
  not mechanism: what happens when the pack updates.
  *As built:* the choice is not fork-vs-override but **overlay-vs-fork**, which is
  the one a user actually faces — "I want this pack to say something different"
  versus "I want a separate variant". The cheaper route is offered FIRST and by
  name, because the expensive mistake is forking to change one prompt and thereby
  never receiving the pack's improvements again.
- **T013 — the workflow list view (FR-011).** Yours versus from-packs, with
  provenance, customization state, and per-pack counts — the mockup's `list`
  state. Binding picker labelled as workflows, grouped by method.

## Phase 3 — override — DROPPED

> **Not deferred. Dropped, with the reason.**
>
> Two findings killed it, and neither was visible when it was planned:
>
> 1. **No pack declares an override surface IN BOS'S OWN `overrides` FIELD** —
>    not spec-kit, not BMAD, not OpenSpec. The rule "a pack with no surface can
>    only be forked" made the feature inert on everything shipped.
>
>    *Corrected later:* BMAD DOES ship an override surface — `customize.toml` per
>    skill, three-layer merge, documented upstream. Nobody fills BOS's field. The
>    correction strengthens the drop: layering a second merge under a pack that
>    already has one gives "which version am I running" two answers.
> 2. **The overlay already serves content.** A prompt, a template, an agent —
>    edited at file granularity, keeping every other file's upstream improvements,
>    with no patch format at all (Phase 1b).
>
> What remained was structural override, and it pays for itself only if
> re-applying a change onto a moved base can be done mechanically. It cannot: a
> base that renames a phase needs judgement about whether the user's change still
> means the same thing. A merge engine answers that silently; an agent answers it
> and says what it did.
>
> So an upgrade is **"fork the new version and re-apply my changes"**, performed
> by an agent against a diff BOS computes. That needs one thing, not five:
>
> - **T014 — a fork records its BASELINE. DONE.** The source exactly as taken, so
>   `diff(baseline, yours)` is precisely the user's own edits — the delta, DERIVED
>   rather than authored. Without it a user's changes cannot be told apart from
>   what the base already said.
> - **T015 — `forkStatus` + `workflow_fork_status`. DONE.** What it was forked at,
>   what is available now, which phases the user added or removed, and an explicit
>   instruction to re-apply with judgement rather than mapping silently.
>
> Deleted with it: the patch format, the `a.b` / `a[*].b` selector language, the
> merge-rule table, the generated override editor, and the problems view's
> override cases. `kind: "override"` is gone from `workflow.json` — a type variant
> nothing produces is exactly the half-finished thing that misleads whoever reads
> it next.

## Phase 4 — authoring (T021 … T025) — DONE

- **T021 — `applyWorkflowEdit(id, op, rev)` (FR-019). DONE.** One function behind
  both the canvas's `PATCH /api/workflows` and the agent's `workflow_edit`.
  Validates before writing; refuses a `rev` mismatch naming both revisions.
  *Beyond the design's op set:* `movePhase` (§3.1c made `phases[]` the pipeline,
  and without a reorder a fork can change everything except the thing most people
  fork to change) and `setOptional` (`optional` became a declared field in this
  feature and nothing could set it). `setOverride` is gone with Phase 3.

- **T022 — the op set branches on fork vs PACK, not fork vs override. DONE.**
  Rewritten: override no longer exists, so the branch is "yours" versus "a
  pack's". A pack's workflow refuses every structural op and names forking as the
  route; its PROMPTS stay editable, which needs no fork at all.
  *Acceptance met:* a refusal is a normal outcome carrying its reason and its
  code — the route returns 409 with `code`, so the UI can offer Fork for
  `not-yours` and Reload for `stale` rather than printing a string.
  *Also refused with a reason:* `removePhase` when a rule `dependsOn` the phase
  (dropping the clause would change what the surrounding condition means), and
  `setArtifacts` naming an artifact the workflow never declared (BOS would have
  to guess a scope).

- **T023 — structural validation (FR-017). DONE.** Dangling phase and artifact
  references, duplicate ids, an empty phase list, and cycles. **Not** "an
  artifact no phase generates" (spec-kit ships three) and **not** "a phase
  nothing can reach" (all twelve, since spec-kit declares no gates).
  *Corrected from the task:* cycles are checked over gates **and** `dependsOn`.
  `evaluate.ts` recurses through both and throws on either, so a gates-only check
  passes a descriptor that fails for every unit the moment it is bound.
  *What the guard caught:* "every shipped pack validates clean" failed on its
  first run — OpenSpec's `artifacts[].requires` named PHASE ids where the schema
  says artifact ids, so four of its five artifact dependencies had been silently
  dropped from the derived graph. Fixed in the pack.

- **T024 — the `workflow_edit` server tool. DONE.** *Acceptance met literally:*
  `workflow-edit-one-path.test.ts` runs BOTH transports against the same fork in
  every case and asserts they agree — same refusal codes, same revisions named,
  same gate warning, and an edit made through one is visible to the other.

- **T025 — editing a gate is warned and counted (R5). DONE.** Each bound unit is
  evaluated twice, under what is bound today and under the proposed descriptor.
  *Corrected from the task:* 045 FR-010's preflight compares DISCOVERY, and a
  gate changes no leaf marker, so it reports nothing — it cannot answer this. The
  two now share the unit walk and nothing else.
  *And the derivation shortcut is wrong too:* gates are consulted ONLY when no
  rule clause matched, so current phase states do not tell you what a gate
  changes. Two evaluations, not one and some arithmetic.
  *`examined` is reported with the count* — "0 blocked" and "nothing is bound, so
  nothing was checked" are the same number and opposite facts.
  *Computed inside `applyWorkflowEdit`*, not by the canvas: putting it in front
  of the Apply button would mean an agent never saw it.

## Phase 5 — the centre column (T026)

- **T026 — DONE: one resolved mode, replacing the ternary chain (R4).** Today:
  `showSelfHeal ? … : showConflictPane ? … : viewer`, with `showConflictPane`
  carrying `&& !showSelfHeal` to break the tie by hand. A fourth arm is four
  booleans and six pairwise interactions, failing silently.
  *Acceptance:* one discriminated value, one place, stated precedence;
  behaviour-preserving for the three existing modes.

## Phase 6 — docs and verification (T027, T028) — DONE

- **T027 — Docs (FR-014). DONE.** `docs/dev/method-packs.md` gains "Changing a
  workflow": overlay versus fork as a table of CONSEQUENCES, the closed op set,
  why every edit carries a `rev`, the one warned operation, and upgrading as
  "re-fork and re-apply" performed by an agent. It also records why there is no
  override, so the next reader does not rediscover the idea and rebuild it.
  *Corrected while writing it:* the binding paragraph described `method` alone,
  and 049 made `workflow` supersede it at every level — which is how a fork is
  bound at all. `CLAUDE.md` carries the same two-mechanism summary.

- **T028 — Live run. DONE, 13 of 13.** Against the REAL data dir: the installed
  BMAD pack, the real `user-specs` store, the real registry. Forked
  `bmad @ 6.13.0-next`, removed the `prd` phase through `applyWorkflowEdit`,
  bound `user-specs` to the fork, then upgraded the pack to a new version.

  | # | Assertion | |
  |---|---|---|
  | 1 | the real pack registers | PASS |
  | 2 | fork copies all 8 phases | PASS |
  | 3 | `prd` is gone; 7 left | PASS |
  | 3b | and it SAID prd.md lost its producer | PASS |
  | 3c | BMAD itself still has `prd` | PASS |
  | 4 | a real store resolves to the fork | PASS |
  | 5 | the pack upgrades to 99.0.0 | PASS |
  | 6 | the fork's descriptor did not move | PASS |
  | 6b | it still reports the version it was taken at | PASS |
  | 7 | the binding still resolves after the upgrade | PASS |
  | 8 | drift is REPORTED, not merged | PASS |
  | 8b | and names what the user changed | PASS |
  | 9 | SC-004 — nothing written inside the pack | PASS |

  The run mutated real state and restored it: the manifest written back
  byte-for-byte and git-checked-out, the fork deleted, both repos verified clean.

  *Assertion 7 is the one that earns the task.* It is exactly what the seed bug
  broke — `ensureUserStore` rewrote the store manifest from an allowlist that
  049's `workflow` field was never added to, so every fork binding reverted on
  the next boot, silently. That was found while writing T025's test rather than
  here, but this is the assertion that would have caught it in production.

## Coverage

| Mockup state | Built by |
|---|---|
| canvas — spec-kit / BMAD / OpenSpec | T005 |
| fork vs override | T012 — as **fork vs overlay**, which is the choice that survived |
| adding a gate | T025 + the inspector's preview-then-confirm |
| override editor | **dropped with Phase 3** — no pack declares a surface, and the overlay serves content edits already |
| drift & problems | T015 (`forkStatus`, `workflow_fork_status`) + the canvas's "source is now @ X" banner |
| workflow list | T013 |

Two rows changed meaning rather than being quietly deleted. The override editor
has no owner because the thing it edited does not exist; saying so is the point,
since a coverage table naming absent tasks is worse than no table. "Drift &
problems" narrowed to drift, which is the only half that still has a subject —
the "problems" cases in the mockup were override conflicts.

## Dependencies

```
T001 → T002, T003 → T004 → T005 → T006 ──(decision)──> T007 → T008 → T009 → T010, T011
                                                        T008 → T012, T013
                                                        T009 → T014 → T015
                                                        T008 → T021 → T022, T023 → T024 → T025
T026 independent of everything above
T029 … T032 arrived from T006 and depend only on T004
```

T016 … T020 are absent because Phase 3 was dropped, not because they are pending.

## Critical path

`T001 → T004 → T005 → T006 → T008 → T021 → T028`

## Status

Every task is done. Phase 3 was dropped with its reason recorded above; T016 …
T020 never existed.

Deferred by the design, not by omission:
- **drag-to-connect** (§3.7, P2) — a second input method for `setRequires`, the
  rarest of the three edge kinds and the one that needs a warning before it
  applies. Revisit if users reach for the canvas to build rather than to review.
