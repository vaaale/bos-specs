# Design: Skill and agent hygiene

> **Authoring note.** BOS's `design` step delegates to `architect`, verified by `architect-reviewer`. Neither exists in a Claude Code session, so this was authored directly. Across `045` and `046` this author's designs needed three and five independent review rounds; every failure was a claim reasoned about rather than looked up, and one was a claim looked up once and never re-checked. Every mechanical assertion below cites source. **Not yet reviewed.**

Spec: [`spec.md`](./spec.md) · Series: **`044`** → `045` → `046` → `047` → `048`

---

## 1. Classification

**`bos-core`.** Agrees with spec.md's `App Target`.

Unusually for a `bos-core` feature, almost none of it is code. The source surface is:

| Area | Extent |
|---|---|
| `seed/skills/**` | Three skills retired, one split, one created |
| `seed/agents/**` | Three `skills:` frontmatter lists, two prose sections |
| `src/lib/agent/compaction/config.ts:48` | **One line** — `unrecoverableTools: []` → `["skill_load"]` |
| `docs/**` | Three stale claims |

No `pipeline.ts`, no `types.ts`, no UI, no API route, no schema. That is what makes it independent of `045`–`048`: every reference to them in spec.md is forward-looking (`044` is a prerequisite *for* them), never a dependency *on* them.

`target-marketplace-item.md`'s guard-rail does not apply — nothing binds a port, runs continuously, or handles a raw protocol. No service facet.

---

## 2. Constitution check

| Principle | Assessment |
|---|---|
| **I. Spec-Driven (NON-NEGOTIABLE)** | Self-referential in the same way `045` is, but in prose: this feature rewrites the instructions that *drive* the spec pipeline. A regression degrades BOS's ability to author specs, including the remaining four in this series. **Compliant**, but see R1 — the usual mitigation (a test) does not exist for prompts. |
| **II. Server Authority & SSR Boundary** | Seed content and one config default. No client/server boundary crossed. **Compliant.** |
| **III. Always Delegate; Claude Codes** | `bos-core` → `dev_delegate`. Note the feature *edits agent definitions*, including `skills:` allowlists that govern what a delegated agent can load — so the Developer is modifying the machinery it is itself subject to. Not a violation; worth stating. **Compliant.** |
| **IV. Minimize Blast Radius (NON-NEGOTIABLE)** | Feature branch. The blast radius is unusual: deleting a skill is instant and total for every agent whose index it appeared in, and there is no staged rollout for prose. FR-010's pure-move discipline is the mitigation. **Compliant.** |
| **V. The VFS Is Not the Source** | `seed/` is BOS source, not the VFS. No mount changes. **Compliant.** |
| **VI. Specs & Docs Stay in Sync (NON-NEGOTIABLE)** | FR-002 and FR-009 exist *because* this principle was violated — two docs assert things that are false today. **Compliant, and corrective.** |
| **VII. Respect Boundaries** | No dependency changes; `npm run test:unit`, `tsc`, `lint` are SC-005. **Compliant.** |

---

## 3. Architecture

### 3.1 The seam

One organising test decides where every sentence goes:

> **Would this sentence still be true if we swapped frameworks?** True → BOS owns it. Changes → the pack owns it.

Applied, it produces three destinations, and the whole feature is moving content into them:

```
                    BOS-owned, method-neutral          Pack-owned (046)
  ┌──────────────────────────────────────────┐   ┌──────────────────────┐
  │  bos-domain          (FACTS about BOS)   │   │  build-studio        │
  │   · target-*.md (~40KB)                  │   │   · SKILL.md         │
  │   · A2UI catalog, style guide            │   │   · per-step refs    │
  │   · the three implementation shapes      │   │   · C4/ADR discipline│
  │   · BOS's real containers                │   └──────────────────────┘
  ├──────────────────────────────────────────┤
  │  intent              (WHAT the user wants)│
  │   · ← bos-app phases 0–2                  │        DELETED
  │   · design-interview-script.md            │   ┌──────────────────────┐
  ├──────────────────────────────────────────┤   │ develop-in-browseros │
  │  ui-designer craft   (HOW a UI is made)  │   │ feature-wizard       │
  │   · ← bos-app phase 3                     │   │ bos-app phases 4–7   │
  │   · a2ui-catalog.md, ui-conventions.md   │   └──────────────────────┘
  └──────────────────────────────────────────┘
```

`senior-solution-architect` splits across the line: its Phase 1 (classify) and Phase 2's real-container vocabulary are **facts** → `bos-domain`; its C4/ADR discipline is **method** → stays as craft, and becomes pack-adjacent in `046`.

### 3.2 Container

None. There is no process, port, daemon or storage change. The only runtime behaviour this feature alters is which skills appear in an agent's system-prompt index (`instructions.ts:181`'s `buildSkillsIndexBlock`) and which turns compaction protects (`compaction/config.ts:48`).

### 3.3 Component — what actually changes at runtime

Two mechanisms, and only two:

1. **The skills index.** `buildSkillsIndexBlock(agent?.skills, skills)` filters the library to an agent's allowlist. Retiring a skill removes it from every index; adding one requires the allowlist to name it.
2. **Compaction protection.** `unrecoverableTools` (`config.ts:30`, default `[]` at `:48`) is consumed by `protectedTurnMessageIds` (`turns.ts:34-46`) at every layer — block summarization (`blocks.ts:32-36`), view rendering (`render.ts:83`) and the hard-limit `forceClearAll` (`truncate.ts:27-30`).

Everything else is text an agent reads.

### 3.4 The retirement problem: on this deployment, retiring a skill does nothing

**This is the finding that most shapes the feature, and it is not in spec.md.**

Retiring a skill from `seed/skills/` does not remove it from `data/skills/`. That is `archiveDroppedSeedSkills`' job (`skills/store.ts:147-161`), which calls `decideSeedAction` with `inSeed: false`. The table (`seed-sync.ts:107-123`):

```ts
if (liveRev === undefined) return inSeed ? "seed" : "current";
if (!stamp || stamp.live !== liveRev) return "local";   // ← unstamped lands here
if (!inSeed) return "archive";
```

The `local` guard fires **before** the archive branch, and it fires on an **unstamped** copy as well as an edited one.

**Every one of the 16 directories in this machine's `data/skills/` is unstamped** — zero carry a `.seed-rev` (contrast `data/agents/`, where 10 of 14 do). So on this deployment, retiring `develop-in-browseros`, `feature-wizard` and `bos-app` from `seed/` archives **none** of them. All three remain in `data/skills/`, remain in every agent's index, and remain loadable by `skill_load` — which is ungated (`tools/server/skills.ts:27-41`).

FR-002, FR-003 and FR-004's retirements would be **no-ops here**, and SC-002 ("`bos-app`, `feature-wizard` and `develop-in-browseros` are absent from the skills index") would fail — not because the code is wrong, but because the deployment state defeats it.

Two sub-cases survive even where stamps exist:

- **Edited copy** → `local` → stays forever, now orphaned with no seed backing and no path to update. This is correct for user work, but it means a retired skill can persist indefinitely on any deployment where `skill_improve` or the Curator touched it.
- **Unstamped copy** → identical outcome, for a reason that has nothing to do with user intent.

The feature needs an explicit disposition for this. See ADR-5.

### 3.5 `build-studio`'s allowlist is already partly dead

`seed/agents/build-studio/AGENT.md:6` reads `skills: [build-studio, bos-app, tidy-documents, workflow-manager-design]`.

`tidy-documents` and `workflow-manager-design` exist in **neither** `seed/skills/` nor `data/skills/`, and are offered by no registered marketplace. `filterAllowed` drops unmatched ids silently, so two of the four entries contribute nothing today.

This matters because FR-004 dissolves `bos-app`, leaving that list with **one** live entry out of four. The allowlist has to be rewritten regardless — to add `bos-domain`, the intent skill and (for delegation) nothing else — so the dead ids should go in the same edit rather than being left as archaeology.

---

## 4. Concrete file/module plan

### New

| Path | Responsibility |
|---|---|
| `seed/skills/bos-domain/SKILL.md` + `references/` | **FR-005.** BOS facts. Receives `build-studio`'s three `target-*.md` (~40KB), `bos-app`'s `a2ui-catalog.md` and `ui-conventions.md` **by reference from the craft skill**, and `senior-solution-architect`'s Phase 1 + container vocabulary. |
| `seed/skills/intent/SKILL.md` + `references/design-interview-script.md` | **FR-004b.** ← `bos-app` phases 0–2. Phase 0's classification step straddles the seam and splits at it: *eliciting and confirming what the user wants* stays here; *the knowledge that BOS has exactly three implementation shapes* is loaded from `bos-domain`, not restated. Method-neutral: capturing what a user wants does not change with the framework. |
| `seed/skills/ui-designer-craft/SKILL.md` + `references/{a2ui-catalog,ui-conventions}.md` | ← `bos-app` phase 3. `ui-designer` is the only agent with `skills: []` today, so all its craft is trapped in its AGENT.md. |
| `tests/agent/delegation-targets.test.ts` | FR-001's regression guard — enumerate every `agent_delegate( agent: "…" )` target in seeded content and assert each resolves. Baseline is **zero failures** (all five resolve today); this guards the retirements, it does not fix a bug. |

### Modified

| Path | Change |
|---|---|
| `seed/skills/senior-solution-architect/SKILL.md` | Split (FR-006). Phase 1 (classify) and Phase 2's real-container vocabulary → `bos-domain`; C4/ADR discipline stays. **Two commits** (verbatim extraction, then any rewording) so a diff review can confirm the extraction changed only location — FR-010. |
| `seed/agents/build-studio/AGENT.md` | `:6` allowlist rewritten — add `bos-domain` and `intent`, drop `bos-app` (dissolved) and the two dead ids (§3.5). `:33` category 3 routes to the intent skill, not `bos-app` (FR-004a). `:34` category 4 folds into category 2 (FR-003). |
| `seed/agents/architect/AGENT.md`, `architect-reviewer/AGENT.md` | `:6` — drop `build-studio`, add `bos-domain`. They depend on the *process* skill today, which breaks when `046` makes it pack-owned. |
| `seed/agents/ui-designer/AGENT.md` | `skills: []` → `[ui-designer-craft, bos-domain]`. |
| `seed/skills/build-studio/SKILL.md` | `:25` — "bos-app's Phase 2" named by content, not ordinal (FR-007). Its `target-*.md` references become `bos-domain` loads. |
| `seed/skills/build-studio/references/target-builtin-app.md` | `:7` — "Functional Design phase" named by content (FR-007). |
| `src/lib/agent/compaction/config.ts` | `:48` `unrecoverableTools: []` → `["skill_load"]` (FR-008). **Not `skill_read_file`** (FR-008a) — reference reads are large and numerous (`target-marketplace-item.md` alone is 31KB) and would pin most of a long session. |

| `docs/dev/self-improvement/self-improvement.md:27` | Calls `develop-in-browseros` load-bearing (FR-002). |
| `docs/dev/build-studio.md:111-113` | Claims seeded agents are "never overwritten, so a stale local copy needs a manual fix"; the code does three-way reconciliation via `.seed-rev` (FR-009). |
| `src/app/api/specs/run-tests/route.ts:11` | Comment cites "feature-wizard Phase 6" as `buildstudio_run_tests`' origin (FR-003). The tool stays; only the provenance note changes. |

### Deleted

`seed/skills/develop-in-browseros/` (SKILL.md + 2 references) · `seed/skills/feature-wizard/` (SKILL.md + 3 references, of which `branch-setup.md`, `testing.md` and the promote/discard phase are salvaged) · `seed/skills/bos-app/` (phases 4–7 discarded; 0–3 relocated).

---

## 5. Integration points (called into, not modified)

- **`buildSkillsIndexBlock` / `filterAllowed`** (`instructions.ts:181`, `capabilities.ts`) — an unset allowlist means *all* skills, so the five agents with no `skills:` line (`default_agent`, `developer`, `planner`, `researcher`, `writer`, `file-organizer`) see retirements immediately without any edit.
- **`skill_load` is ungated** (`tools/server/skills.ts:27-41`) — any agent can load any skill by id regardless of allowlist. So retirement must remove the skill, not merely de-list it; §3.4 is what makes that non-trivial.
- **`decideSeedAction` / `archiveDroppedSeedSkills`** (`seed-sync.ts:107-123`, `skills/store.ts:147`) — the retirement mechanism, unmodified under ADR-5's option (a).
- **`protectedTurnMessageIds`** (`turns.ts:34-46`) — consumes FR-008's default; no change to the mechanism.

---

## 6. ADRs

### ADR-1 — The seam is the swap-the-framework test, not "reduce skill count"

**Context.** BOS has three overlapping pipeline drivers and domain knowledge tangled with process knowledge. The obvious fix is "fewer skills".

**Options.** (a) Fold everything into `build-studio`. (b) Split by *method-coupling* — domain/craft to BOS, process to the pack.

**Decision.** (b).

**Consequences.** (a) optimises count today and pessimises duplication tomorrow: `build-studio` becomes pack-owned in `046`, so anything folded in duplicates once per framework — ~60KB of references including 40KB of BOS install mechanics no framework has an opinion about, versus ~8KB of genuine process prose. The count still drops (four skills to two in this area), but as a consequence rather than a goal. Cost: two destinations to reason about instead of one, and a test that must be applied sentence by sentence.

### ADR-2 — Dissolve `bos-app` rather than retire or fold it

**Context.** `bos-app` is a full parallel pipeline (8 phases) whose phases 4–7 re-implement specify/plan/tasks/implement/converge.

**Options.** (a) Retire wholesale. (b) Fold into `build-studio`. (c) Dissolve: 0–2 → intent, 3 → ui-designer craft, 4–7 deleted.

**Decision.** (c).

**Consequences.** Phases 0–3 are genuinely valuable and have real infrastructure behind them — UI Preview and the A2UI toolchain exist, by their own comments, to serve a "bos-app design session" (`src/apps/ui-preview/manifest.ts:12`, `index.tsx:21`, `frontend-declarations.ts:128`, `a2ui-render.ts:119`). Retiring them wholesale would orphan that. Folding them in duplicates per pack (ADR-1). Cost: ordinal cross-references break — `build-studio/SKILL.md:25` cites "bos-app's Phase 2" and `target-builtin-app.md:7` its "Functional Design" phase — which FR-007 fixes by naming content instead.

### ADR-3 — Retire `feature-wizard` outright

**Context.** A third pipeline (8 phases), whose 3–5 duplicate specify/plan/tasks.

**Options.** (a) Keep as a distinct entry point. (b) Retire, salvaging what is unique.

**Decision.** (b). Unique content — `branch-setup.md`, `testing.md`, the promote/discard phase — is salvaged into BOS-owned references.

**Consequences.** Its only inbound reference is `build-studio/AGENT.md:34` (category 4), which folds into category 2. Note it is **not** in `build-studio`'s own allowlist, so the agent is told to load a skill absent from its index — de-listed and referenced at once, which is its own small argument for retirement.

### ADR-4 — `bos-domain` is one skill, not a section in each craft skill

**Context.** Multiple roles need the same BOS facts: architect, ui-designer, the intent flow, and any future pack's cast.

**Options.** (a) Inline the facts into each craft skill. (b) One `bos-domain` skill, loaded by id.

**Decision.** (b).

**Consequences.** The facts have one home, so a BOS change updates one file. Critically, it is what lets `048`'s BMAD personas reach BOS knowledge without the pack embedding a copy — the property the whole series depends on. Cost: an extra `skill_load` per agent, which FR-008's compaction protection is what makes safe.

### ADR-5 — No special handling; retirement is a seed deletion plus a manual `data/` clean-up

**Context.** §3.4 — `archiveDroppedSeedSkills` returns `local` for unstamped copies, and all 16 of this machine's `data/skills/` are unstamped, so a seed deletion alone archives nothing.

**Options.** (a) Special-case unstamped copies of retired ids in `skills/store.ts`. (b) Delete the three ids from `data/` on upgrade regardless of stamp. (c) No code change: delete from `seed/`, and clean `data/` as a documented one-time operator step.

**Decision.** (c).

**Consequences.** `044` touches **zero** lines of `src/lib/agent/skills/store.ts`; its only code change stays the one-line `unrecoverableTools` default. This is deterministic rather than best-effort — fresh copies are written *and stamped* (`skills/store.ts:139`), so the clean-up also repairs the unstamped state and every future retirement archives correctly with no mechanism at all. (a) and (b) both build machinery to automate something done once, and (b) is the exact failure the `local` guard exists to prevent.

Cost: an operator step that must be stated precisely, because the obvious rule is wrong — see §3.6.

### 3.6 The clean-up rule, stated exactly

Deferred to an operator, so the rule has to be unambiguous. **"Delete whatever has a counterpart in `seed/`" is wrong** — after this feature the three retired skills have *no* seed counterpart, so that rule spares precisely the ones that must go, and being unstamped they are never archived either. They would survive in every agent's index indefinitely.

The 16 partition as:

| Group | Skills | Action |
|---|---|---|
| **Retired — no seed counterpart, no other source** | `develop-in-browseros`, `feature-wizard`, `bos-app` | **Must delete** |
| Still seed-backed | `agent-behavior-review`, `agent-skill-orchestrator`, `build-studio`, `deep-research`, `design-stress-test`, `devops-merge-conflict-resolution`, `executing-plans`, `recall-long-term-memory`, `senior-solution-architect` | Delete to force a re-seed **with stamps** — optional but recommended |
| Item-bundled | `okf-bundle-schema`, `okf-ingest-workflow`, `workflow-manager` | Indifferent — `reconcileInstalledItemAssets` restores them |
| **No source anywhere** | `summarize-a-web-page` | **Must keep** — `created_by: seed`, absent from today's `seed/`, offered by no marketplace |

Simplest correct form: **delete everything except `summarize-a-web-page`.**

Agents need no equivalent step in this feature — `044` retires none. (`046` relocates four, all currently seed-backed, where the ordinary archive path is correct and sufficient.)

## 7. Risks / open questions

**R1 — The unit suite cannot detect this feature's main failure mode.** Prompt regression is invisible: the suite stays green through a reorganisation that makes the architect materially worse. There is no parity harness for prose, and building one is out of scope. The mitigations are FR-010's pure-move discipline (so a diff review can confirm only location changed) and the manual check in SC-006. Treat both as load-bearing, not ceremony.

**R2 — The retirement completes only after the operator step** (§3.6). The seed deletion is necessary and not sufficient; SC-002 is asserted against a cleaned `data/`, not against the branch alone. The rule is counter-intuitive enough that stating it as a principle rather than a list is the likely failure.

**R3 — `data/skills/` copies of retired skills persist even where stamps exist**, if edited. That is correct behaviour, but it means "retired" is not the same as "gone", and any assertion that reads as the latter will be wrong on some deployment.

**R4 — Deleting a skill is instant and total** for the five agents with no `skills:` allowlist, who see the whole library. There is no staged rollout for prose; the change lands for every agent at once on the next `ensureSeed()`.

**R5 — One acceptance scenario cannot be tested in this feature.** US3 AS2 — *"Given `build-studio` is later relocated into a pack (`046`), then nothing breaks"* — is a forward-compatibility assertion about a feature that does not exist yet. It should be marked deferred to `046` rather than silently counted as passing.

**Resolved (was blocking `plan`):** ADR-5 — no special handling. `044` does not touch `skills/store.ts`.
