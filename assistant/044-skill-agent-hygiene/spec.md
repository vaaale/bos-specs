# Feature Specification: Skill and agent hygiene — separate BOS domain knowledge from framework process

**Feature Branch**: `044-skill-agent-hygiene`

**Created**: 2026-09-10

**Status**: Draft

**App Target**: bos-core

**Input**: "BOS currently uses GitHub spec-kit as the driver of the development pipeline. Other frameworks such as OpenSpec and BMAD exist. It would be nice if these frameworks could be plugged into BOS as some kind of marketplace item." — first of five features derived from that analysis.

> Prerequisite for `build-studio/045-method-pack-layer` (whose agent-relocation step depends on it) but **valuable standalone**: it removes three overlapping spec-authoring pipelines and a documented prompt-reliability gap, none of which need packs to be worth fixing. Does not touch `pipeline.ts`, `types.ts`, or the Build Studio UI — those are 045's scope.

## Why this exists (context)

BOS has **three** overlapping spec-authoring pipelines, and nothing arbitrates between them beyond request-category triage in `seed/agents/build-studio/AGENT.md`:

| `build-studio` (9 steps) | `bos-app` (8 phases) | `feature-wizard` (8 phases) |
|---|---|---|
| constitution | 0 orient & categorize | 0 requirements |
| specify | 1 interview | 1 UI design |
| clarify | 2 functional design | 2 feature branch |
| design | 3 UI design (A2UI) | 3 specification |
| plan | 4 spec finalization | 4 plan & tasks |
| tasks | 5 plan & tasks | 5 implementation |
| analyze | 6 delegate | 6 testing |
| implement | 7 verify & converge | 7 promote / discard |
| converge | | |

`bos-app` phases 4–7 and `feature-wizard` phases 3–5 are re-implementations of specify/plan/tasks/implement/converge. A fourth skill, `develop-in-browseros`, is referenced by nothing at all.

Separately, BOS's **domain knowledge** (what BOS is made of, where things live, how installs work) is tangled together with **framework process knowledge** (what artifacts exist, in what order, who produces them). The seam is already being violated: `seed/skills/senior-solution-architect/SKILL.md`'s Phase 1 — which it calls "the load-bearing step" — instructs the architect to `skill_read_file` on **build-studio's** `references/target-*.md`. A craft skill reaching into the process skill for BOS facts.

That tangle is why this must land before packs. `build-studio` becomes pack-owned in `046`; anything left inside it duplicates once per framework — roughly 60KB of references instead of 8KB, including 40KB of BOS install mechanics that no framework has an opinion about.

### The organising test

> **Would this sentence still be true if we swapped frameworks?** True → BOS owns it. Changes → the pack owns it.

Applied to what BOS ships today:

| Content | Owner |
|---|---|
| `target-*.md` install/delegation mechanics (~40KB) | BOS — domain |
| A2UI catalog, style guide, UI conventions | BOS — domain |
| "Classify as bos-core / builtin-app / marketplace-item" | BOS — domain (a fact about BOS) |
| "BOS's real containers: Next.js process, Supervisor, worker-thread services, GitFS stores" | BOS — domain |
| "Write ADRs: Context / Options / Decision / Consequences" | Pack — process |
| "C4 Context / Container / Component" | Pack — process |
| `architect`, `architect-reviewer`, `ui-designer`, stress-testing | Pack — process |

The last row is the finding: **BOS ships process agents it shouldn't own.** That is also why the CIS-overlap question needs no curation decision — once stress-testing is pack-owned, BMAD's CIS Brainstorming Coach competes with the *spec-kit pack's* stress-tester, and only one pack is active per store.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Every delegation target still resolves after the reorganisation (Priority: P1)

An agent following a skill's instructions never hits a delegation to an agent that isn't installed — **including after this feature moves skills and agents around**.

**Why this priority**: this is a regression guard, not a bug fix. An earlier draft of this spec asserted that `devil-s-advocate` was missing and that six `agent_delegate` references were dead. **That was wrong** — it exists at `seed/agents/devil-s-advocate/` (added in `e9fc989`), and every delegation target in seeded content resolves today: `architect`, `architect-reviewer`, `developer`, `devil-s-advocate`, `ui-designer`. The claim came from a stale directory listing that was never re-verified. What remains true is that this feature retires three skills and moves content between them, which is exactly the kind of change that *can* break a delegation.

**Independent Test**: Enumerate every `agent_delegate( agent: "…" )` target across all seeded skills and agents; assert each resolves to an agent in `seed/agents/`. Run before and after.

**Acceptance Scenarios**:

1. **Given** the pre-change tree, **When** every delegation target is enumerated, **Then** all five resolve (the baseline).
2. **Given** the post-change tree, **When** the same enumeration runs, **Then** all targets still resolve and none was orphaned by a retired skill.

### User Story 2 - One pipeline driver, not three (Priority: P1)

A user asking BOS to build a feature gets one authoring flow, chosen by the active method, not one of three overlapping flows chosen by triage heuristics.

**Why this priority**: the ambiguity is the root cause of a documented real-world failure — `seed/skills/build-studio/SKILL.md:31` records a session where an agent classified early, went straight to `agent_delegate`, never called `skill_load` at all, and produced a marketplace item silently missing its manifest entry and app facet, discovered two bug reports later.

**Independent Test**: Grep the seeded skill set for pipeline-shaped phase sequences; exactly one should remain.

**Acceptance Scenarios**:

1. **Given** a fresh install, **When** the skills index is composed, **Then** `bos-app`, `feature-wizard` and `develop-in-browseros` are absent.
2. **Given** the user asks to build an app with a UI, **When** the agent routes the request, **Then** it follows `build-studio`'s pipeline and its UI-design step, not a separate competing pipeline.

### User Story 3 - Domain knowledge lives in one place (Priority: P1)

BOS facts — install mechanisms, real containers, the three implementation shapes, the A2UI catalog, the style guide — live in exactly one skill that any agent under any framework can load.

**Why this priority**: this is the structural change everything downstream depends on. Without it, each pack duplicates BOS's domain knowledge and the copies drift.

**Independent Test**: Assert no skill reaches into another skill's `references/` for domain facts; assert `bos-domain` contains no sentence that changes when the framework changes.

**Acceptance Scenarios**:

1. **Given** the `architect` agent runs its classification step, **When** it needs the three implementation shapes, **Then** it loads `bos-domain`, not `build-studio`'s references.
2. **Given** `build-studio` is later relocated into a pack (`046`), **When** the architect classifies, **Then** nothing breaks.

### User Story 4 - A loaded skill survives a long session (Priority: P2)

An agent that loaded its craft skill early in a long conversation still has its instructions late in that conversation.

**Why this priority**: `src/lib/agent/compaction/config.ts:48` defaults `unrecoverableTools` to `[]`, and `summarize.ts:58` asks the summarizer to preserve "app/agent/skill **ids**" — not bodies. So an agent retains "I loaded skill X" and loses X's instructions mid-flight. Real, but slower-burning than the P1s.

**Independent Test**: Drive a conversation past the summarize threshold with a `skill_load` early in history; assert the loaded body is still present verbatim.

**Acceptance Scenarios**:

1. **Given** a conversation that has crossed `summarizeThreshold`, **When** the turn containing `skill_load` would be summarized, **Then** it is preserved verbatim.
2. **Given** the hard limit is hit, **When** `forceClearAll` runs, **Then** the `skill_load` turn is still protected.

### Edge Cases

- **`data/skills/` and `data/agents/` hold live copies.** Deleting a seed skill must archive rather than orphan its live copy. `archiveDroppedSeedAgents` is the precedent for agents; the skills store needs equivalent treatment.
- **A user may have locally edited a skill being retired.** The `.seed-stamp` three-way contract (`subagents/store.ts:53-77`) protects local edits by design; a retired-but-locally-edited skill must be reported, not silently archived or silently kept.
- **`buildstudio_run_tests` originated in `feature-wizard`** (`src/app/api/specs/run-tests/route.ts:11`). The tool stays; only the comment's provenance reference changes.
- **`assistant` is `DEFAULT_AGENT_ID`** (`src/lib/agent/agent-ids.ts:11`) and its allowlist already excludes the retired skills, so the main chat surface is unaffected by their removal.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Every `agent_delegate` target in seeded content MUST still resolve after this feature's retirements and moves. No agent needs creating: `architect`, `architect-reviewer`, `developer`, `devil-s-advocate` and `ui-designer` all exist in `seed/agents/` today. (An earlier draft required shipping `devil-s-advocate` on the basis that it was missing; it is not, and that requirement has been withdrawn rather than quietly reworded.)
- **FR-002**: The `develop-in-browseros` skill MUST be removed. Its content is superseded by `build-studio`'s `references/target-bos-core.md` and `target-builtin-app.md`. `docs/dev/self-improvement/self-improvement.md:27`, which still calls it load-bearing, MUST be corrected.
- **FR-003**: The `feature-wizard` skill MUST be removed. `references/branch-setup.md`, `references/testing.md` and its promote/discard phase MUST be salvaged into BOS-owned neutral references; the duplicate pipeline MUST NOT be. Request category 4 in `seed/agents/build-studio/AGENT.md:34` folds into category 2.
- **FR-004**: The `bos-app` skill MUST be dissolved: phases 0–2 (orient, categorize, interview, functional design) plus `references/design-interview-script.md` become a BOS-owned **intent** skill; phase 3 plus `references/a2ui-catalog.md` and `references/ui-conventions.md` become the **ui-designer craft** skill; phases 4–7 MUST be deleted. This content MUST NOT be folded into `build-studio`.
- **FR-004a**: Request category 3 in `seed/agents/build-studio/AGENT.md:33` currently routes UI work to `bos-app`. It MUST be rewritten to route to the intent skill, which then delegates UI design to `ui-designer`. (Category 4 folds into category 2 per FR-003; category 3 must not be left pointing at a deleted skill.)
- **FR-004b**: `bos-app` phase 0's classification step straddles the intent/domain seam and MUST be split at it: *eliciting and confirming what the user wants to build* stays in the intent skill; *the knowledge that BOS has exactly three implementation shapes and how to tell them apart* belongs to `bos-domain` (FR-005) and MUST be loaded from there, not restated in the intent skill.
- **FR-005**: A new **`bos-domain`** skill MUST hold the three `target-*.md` files, the style guide, the A2UI catalog, and the classify step plus real-container vocabulary lifted out of `senior-solution-architect`.
- **FR-006**: `senior-solution-architect` MUST be split — BOS facts to `bos-domain`, C4/ADR discipline and discovery method retained as process content. `build-studio` MUST be removed from the `skills:` lists of `seed/agents/architect/AGENT.md:6` and `seed/agents/architect-reviewer/AGENT.md:6`.
- **FR-006a**: A seeded agent's `skills:` allowlist MUST name only skills that `seed/skills/` ships. The seed is the template BOS installs with; an entry pointing at a marketplace or item-bundled skill is dead on a fresh install, and `filterAllowed` drops it silently. Deployments that install such skills add them to their own copy, which the `.seed-rev` contract then protects as a local edit. Current violations: `assistant` names eight skills that resolve **nowhere** (`delegate-to-sub-agent`, `docx`, `mcp-tool-invocation-strategy`, `pdf`, `pptx`, `preview-html-file`, `skill-creator`, `xlsx`); `conversation-reviewer` names `workflow-manager`, which is live here but bundled by the workflows item, not seeded; `build-studio` names `tidy-documents` and `workflow-manager-design`, which exist nowhere.
- **FR-007**: Cross-references by phase ordinal MUST be rewritten to name content instead: `seed/skills/build-studio/SKILL.md:25` ("bos-app's Phase 2") and `references/target-builtin-app.md:7` ("Functional Design phase").
- **FR-008**: `unrecoverableTools` MUST default to `["skill_load"]` (`compaction/config.ts:48`), so a skill loaded early in a session is still present late in it. `summarize.ts:58` asks the summarizer to preserve "app/agent/skill **ids**" — not bodies — so without this an agent retains "I loaded skill X" and loses X's instructions mid-flight.

  Note the mechanism's shape: `protectedTurnMessageIds` (`turns.ts:34-46`) protects the **whole turn** containing the call, not the skill body alone, and applies at every layer — block summarization (`blocks.ts:32-36`), view rendering (`render.ts:83`) and the hard-limit `forceClearAll` (`truncate.ts:27-30`). That is a characteristic of the existing mechanism, not something this feature changes.

- **FR-008a**: `skill_read_file` MUST NOT be added to `unrecoverableTools`. Reference reads are large and numerous (`target-marketplace-item.md` alone is 31KB) and would pin most of a long session — a design decision, independent of any measurement.

- **FR-009**: The stale claim at `docs/dev/build-studio.md:111-113` (seeded agents "never overwritten, so a stale local copy needs a manual fix") MUST be corrected; the code performs three-way reconciliation via a per-agent `.seed-stamp` (`subagents/store.ts:53-77`, `:170-172`).
- **FR-010**: Relocations in **FR-004 and FR-005** MUST be pure moves — no rewording while relocating — so a diff review can confirm only location changed. **FR-006 is a genuine split and cannot satisfy that constraint**; it MUST instead land as two commits — first the verbatim extraction of the domain half, then any rewording — so the extraction itself remains reviewable as a move.

### Non-requirements

- **The `hooks/` facet is NOT investigated here.** It is already documented: `docs/dev/apps/services.md:329-333` describes it as an inert placeholder for future *service lifecycle* hooks, with `symlinkManager.ts` creating `dataDir()/system/hooks/<id>` and nothing consuming it. It is not an agent-format adapter home; leave it untouched.
- **No `preload` frontmatter mechanism.** It was considered and dropped: FR-008 covers the same failure more surgically, and the only agents that would have needed it are pack-authored ones bridging to BOS-owned skills, which `048` handles BMAD-natively.

### Key Entities

- **`bos-domain` skill** — BOS facts, framework-independent, loaded by any agent under any pack.
- **Intent skill** — requirement capture and target classification; framework-independent.
- **UI-designer craft skill** — live A2UI mockup design; framework-independent.
- **`build-studio` skill** — the spec-kit pipeline driver only. Becomes pack content in `046`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Zero `agent_delegate` targets in seeded content resolve to a nonexistent agent — measured **before and after**, so the number is unchanged rather than improved. The baseline is already zero.
- **SC-002**: Exactly one skill remains that authors spec artifacts (`spec.md`/`plan.md`/`tasks.md` or their equivalents) — was 3. Measured by that criterion specifically, not by "has phases": the intent skill retains gated phases for requirement capture and is not a counter-example, because it writes no spec artifact.
- **SC-002a**: No e2e or unit test references a retired skill. (Verified zero references at authoring time — this criterion guards against one being added between now and merge, not against an existing failure.) Any new test fixture MUST bundle its own spec content rather than depending on pre-seeded store data.
- **SC-003**: Zero skills reach into another skill's `references/` for domain facts (was: 1, the load-bearing case).
- **SC-003a**: Every `skills:` entry in every `seed/agents/*/AGENT.md` resolves to a directory in `seed/skills/` — asserted by test, so the rot cannot silently return.
- **SC-004**: A skill loaded before the summarize threshold is still verbatim in context after it.
- **SC-005**: `npm run test:unit` green; `npx tsc --noEmit` and `npm run lint` clean.
- **SC-006**: A full `specify → design → plan → tasks` run in Build Studio behaves as before, including the architect and UI-design delegations.

## Assumptions

- Prompt regression is invisible to the test suite — the suite cannot detect that the architect got worse. FR-010's pure-move constraint is the mitigation, and manual verification (SC-006) is required, not optional.
- Retiring a skill that a user has locally edited is rare enough to handle by reporting rather than by building a migration path.
