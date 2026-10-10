# Tasks — 044 Skill and agent hygiene

Spec: [`spec.md`](./spec.md) · Design: [`design.md`](./design.md) · Plan: [`plan.md`](./plan.md)

**Three rules that are not negotiable:**

1. **Salvage before delete.** Every retirement extracts content first. The seed directories are removed in **one consolidated task near the end** (T009) — deleting one early destroys the source being salvaged from.
2. **Repoint before delete.** All consumers are rewired in Phase 2, before T009. The reverse order leaves an allowlist naming a skill that no longer exists, which `filterAllowed` drops **silently** — no error, no log.
3. **Relocations are pure moves** (FR-010). No rewording while relocating, so a diff review can confirm only location changed. FR-006 is a genuine *split* and cannot satisfy this literally — it lands as two commits instead.

---

## PHASE 0 — Baseline

### T001 — Delegation-target guard
- **Files:** `tests/agent/delegation-targets.test.ts` (new)
- **Story:** FR-001, SC-001
- **Work:**
  - Enumerate every `agent_delegate( agent: "…" )` target across `seed/skills/**` and `seed/agents/**`; assert each resolves to a directory in `seed/agents/`.
  - **Baseline is zero failures.** All five targets resolve today — `architect`, `architect-reviewer`, `developer`, `devil-s-advocate`, `ui-designer`. An earlier draft of this spec claimed `devil-s-advocate` was missing and built a requirement on it; that was wrong and was withdrawn. This task guards the retirements, it does not fix a bug.
- **Acceptance:** green before any other task; re-run after T009.

---

## PHASE 1 — Create the destinations

### T002 — `bos-domain`
- **Files:** `seed/skills/bos-domain/SKILL.md` + `references/` (new)
- **Story:** FR-005
- **Work:**
  - Move in `build-studio`'s three `target-*.md` (~40KB) — `target-bos-core.md`, `target-builtin-app.md`, `target-marketplace-item.md`.
  - Move in `senior-solution-architect`'s **Phase 1** (classify — the three implementation shapes) and **Phase 2**'s real-container vocabulary (Next.js process, Supervisor + preview worktrees, worker-thread services, marketplace item repos, VFS/GitFS stores, Bastion).
  - Every sentence must pass the seam test: *would this still be true if we swapped frameworks?* If not, it belongs to the pack, not here.
  - **Pure move** — FR-010.
- **Acceptance:** SC-003 — contains no sentence that changes when the framework changes.

### T003 — `intent`
- **Files:** `seed/skills/intent/SKILL.md`, `references/design-interview-script.md` (new)
- **Story:** FR-004, FR-004b
- **Work:**
  - Move in `bos-app` **phases 0–2** (orient, categorize, interview, functional design) plus `design-interview-script.md`.
  - **FR-004b — split phase 0 at the seam.** *Eliciting and confirming what the user wants* stays here. *The knowledge that BOS has exactly three implementation shapes and how to tell them apart* is loaded from `bos-domain` (T002), not restated.
  - Method-neutral by construction: capturing what a user wants does not change with the framework, which is why this is not pack content.
- **Acceptance:** classification reasoning appears once, in `bos-domain`, and is referenced here.

### T004 — `ui-designer-craft`
- **Files:** `seed/skills/ui-designer-craft/SKILL.md`, `references/{a2ui-catalog,ui-conventions}.md` (new)
- **Story:** FR-004
- **Work:**
  - Move in `bos-app` **phase 3** (live A2UI UI design) plus `a2ui-catalog.md` (136 lines) and `ui-conventions.md` (102 lines).
  - `ui-designer` is the only agent with `skills: []` today, so all its craft currently lives in its AGENT.md — this is where it goes.
  - **`bos-app` phases 4–7 are deleted, not moved** — they re-implement specify/plan/tasks/implement/converge.
- **Acceptance:** SC-002's precondition — the duplicate pipeline is gone, the UI craft is not.

### T005 — Split `senior-solution-architect`
- **Files:** `seed/skills/senior-solution-architect/SKILL.md`
- **Story:** FR-006
- **Work:**
  - BOS facts → `bos-domain` (T002). C4/ADR discipline and the discovery method stay.
  - **Two commits**: first the verbatim extraction, then any rewording. FR-010's pure-move rule cannot apply literally to a split, and this is how the extraction stays reviewable as a move.
  - Today its Phase 1 — which it calls "the load-bearing step" — instructs the architect to `skill_read_file` on **build-studio's** `references/target-*.md`. That cross-skill reach is what breaks when `046` makes `build-studio` pack-owned, and removing it is the point of this task.
- **Acceptance:** SC-004 — no skill reaches into another skill's `references/` for domain facts.

---

## PHASE 2 — Rewire every consumer

### T006 — Agent allowlists
- **Files:** `seed/agents/{build-studio,architect,architect-reviewer,ui-designer}/AGENT.md`
- **Story:** FR-006, FR-006a
- **Work:**
  - **Seed allowlists name only seed skills (FR-006a).** Remove every entry that resolves outside `seed/skills/`: `assistant`'s eight (`delegate-to-sub-agent`, `docx`, `mcp-tool-invocation-strategy`, `pdf`, `pptx`, `preview-html-file`, `skill-creator`, `xlsx`) — all dead on a fresh install — and `conversation-reviewer`'s `workflow-manager`, which is item-bundled rather than seeded. The latter is live on deployments running the workflows item; they re-add it locally, and `.seed-rev` protects that edit.
  - `build-studio:6` — add `bos-domain` and `intent`; drop `bos-app` (dissolved). **Also drop `tidy-documents` and `workflow-manager-design`**: they exist in neither `seed/skills/` nor `data/skills/` and no registered marketplace offers them, so `filterAllowed` already drops them silently. After FR-004 this list would otherwise hold one live entry of four.
  - `architect:6`, `architect-reviewer:6` — drop `build-studio`, add `bos-domain`. They depend on the *process* skill today, which breaks when `046` makes it pack-owned.
  - `ui-designer` — `skills: []` → `[ui-designer-craft, bos-domain]`.
- **Acceptance:** SC-003a — every `skills:` entry in every seeded AGENT.md resolves to a `seed/skills/` directory, asserted by the T001 test rather than by inspection.

### T007 — Category routing
- **Files:** `seed/agents/build-studio/AGENT.md`
- **Story:** FR-003, FR-004a
- **Work:**
  - `:33` — category 3 ("a UI needs to be designed") routes to the **intent** skill, which then delegates UI design to `ui-designer`. It names `bos-app` today.
  - `:34` — category 4 ("end-to-end feature wizardry") folds into category 2; `feature-wizard` is retired.
- **Acceptance:** no request category routes to a retired skill.

### T008 — Ordinal cross-references
- **Files:** `seed/skills/build-studio/SKILL.md`, `references/target-builtin-app.md`
- **Story:** FR-007
- **Work:**
  - `SKILL.md:25` cites "bos-app's Phase 2"; `target-builtin-app.md:7` cites its "Functional Design" phase. Both must **name the content**, not the ordinal — T003/T004 renumber those phases out of existence.
  - `SKILL.md`'s `target-*.md` references become `bos-domain` loads (by skill id, never by path).
- **Acceptance:** no cross-skill reference addresses a phase by number.

---

## PHASE 3 — Delete, default, document

### T009 — Delete the retired seed directories
- **Files:** `seed/skills/develop-in-browseros/`, `seed/skills/feature-wizard/`, `seed/skills/bos-app/` (delete)
- **Story:** FR-002, FR-003, FR-004
- **Work:**
  - Remove all three, **after** T002–T005 have salvaged from them and T006–T008 have repointed every consumer.
  - Salvage already taken: `feature-wizard`'s `branch-setup.md`, `testing.md` and its promote/discard phase; `bos-app` phases 0–3 and its three references; `develop-in-browseros` — **nothing**, it is superseded by `bos-domain`'s `target-bos-core.md`/`target-builtin-app.md` and referenced by no code or prompt.
  - `bos-app` phases 4–7 and `feature-wizard` phases 3–5 are discarded, not salvaged.
- **Acceptance:** `seed/skills/` holds none of the three; T001 re-run green; nothing in `seed/` references a deleted id.
- **Also SC-002a:** assert no e2e or unit test references a retired skill. Zero do today — this guards against one being added between now and merge, not against an existing failure. Any new fixture MUST bundle its own spec content rather than depending on pre-seeded store data.

### T010 — Operator clean-up note
- **Files:** `docs/dev/self-improvement/self-improvement.md` (or the release notes)
- **Story:** FR-002 / rollout
- **Work:**
  - **T009 is necessary and not sufficient.** Deleting from `seed/` does not remove `data/skills/` copies: `archiveDroppedSeedSkills` calls `decideSeedAction` with `inSeed: false`, but the `local` guard (`seed-sync.ts:120`) fires first and catches **unstamped** copies as well as edited ones. All 16 of this machine's `data/skills/` are unstamped.
  - Document the clean-up **as a list, not a principle**. The intuitive rule — "delete whatever has a counterpart in `seed/`" — spares exactly the three that must go, since after T009 they have no counterpart.
    - **Must delete:** `develop-in-browseros`, `feature-wizard`, `bos-app`.
    - **Delete to force a stamped re-seed** (recommended): the nine still in `seed/`.
    - **Indifferent:** `okf-bundle-schema`, `okf-ingest-workflow`, `workflow-manager` — `reconcileInstalledItemAssets` restores them.
    - **Must keep:** `summarize-a-web-page` — `created_by: seed`, absent from today's `seed/`, offered by no marketplace, carries its own `scripts/` and `references/`.
  - Simplest correct form: **delete everything except `summarize-a-web-page`**.
  - Note the side effect: re-seeded copies **are** stamped (`skills/store.ts:139`), so this also repairs the unstamped state and every future retirement archives correctly with no mechanism.
- **Acceptance:** SC-002 is assertable after the step; the note names the four groups explicitly.

### T011 — Compaction default
- **Files:** `src/lib/agent/compaction/config.ts`
- **Story:** FR-008, FR-008a
- **Work:**
  - `:48` — `unrecoverableTools: []` → `["skill_load"]`. `summarize.ts:58` asks the summarizer to preserve "app/agent/skill **ids**", not bodies, so without this an agent retains "I loaded skill X" and loses X's instructions mid-session.
  - **`skill_read_file` MUST NOT be added** (FR-008a) — reference reads are large and numerous (`target-marketplace-item.md` alone is 31KB) and would pin most of a long session.
  - This is the feature's **only** line of code. `src/lib/agent/skills/store.ts` is not touched (ADR-5).
- **Acceptance:** a skill loaded before `summarizeThreshold` is still verbatim after it.

### T012 — Documentation
- **Files:** `docs/dev/self-improvement/self-improvement.md`, `docs/dev/build-studio.md`, `src/app/api/specs/run-tests/route.ts`
- **Story:** FR-002, FR-009, FR-003
- **Work:**
  - `self-improvement.md:27` calls `develop-in-browseros` load-bearing.
  - `build-studio.md:111-113` claims seeded agents are "never overwritten, so a stale local copy needs a manual fix" — the code does three-way reconciliation via `.seed-rev` (`subagents/store.ts:53-77`).
  - `run-tests/route.ts:11` cites "feature-wizard Phase 6" as `buildstudio_run_tests`' origin. The tool stays; only the provenance comment changes.
- **Acceptance:** no doc asserts something this feature falsified.

---

## PHASE 4 — Gates

### T013 — Gates
- **Files:** —
- **Story:** SC-005, SC-006
- **Work:**
  - `npx tsc --noEmit`; `npm run lint`; `npm run test:unit`. Re-run T001.
  - **Manual, and load-bearing:** a full `specify → design → plan → tasks` run in Build Studio, including the `architect` and UI-design delegations. The suite cannot detect prose regression — it will stay green through a reorganisation that makes the architect materially worse (R1).
  - **US3 AS2 is deferred**, not passed: *"Given `build-studio` is later relocated into a pack (`046`), nothing breaks"* is a forward-compatibility claim about a feature that does not exist yet.
- **Acceptance:** SC-005 green; SC-006 performed and recorded; AS2 marked deferred to `046`.

---

## Dependency summary

```
T001 ──────────────────────────────────► gates T009 (re-run), T013

T002 ─┬─► T003        (intent loads bos-domain for classification, FR-004b)
      ├─► T005        (senior-solution-architect's facts land in bos-domain)
      └─► T006        (allowlists name bos-domain)
T004 ───► T006        (allowlists name ui-designer-craft)
T003 ───► T007        (category 3 routes to intent)

T006, T007, T008 ─► T009 ─► T010
                          └─► T012
T011 (independent)
T009, T010, T011, T012 ─► T013
```

**Why T009 is late and consolidated:** it is the only irreversible step. Every task before it either creates a destination or repoints a consumer; deleting earlier would destroy salvage sources (T002–T005) or leave allowlists naming absent skills (T006–T008), which `filterAllowed` swallows silently.

**Parallelisable:** T002/T004 (different destinations, no shared source); T011 (touches nothing else in the feature); T012's three files.

**Critical path** — verified as an actual path through the graph above:

`T002 → T003 → T007 → T009 → T010 → T013`
