# Implementation Plan: Skill and agent hygiene (044)

**Branch**: `bos/spec-method-packs` | **Date**: 2026-09-12 | **Spec**: [`spec.md`](./spec.md) | **Design**: [`design.md`](./design.md)

## Summary

Collapse three overlapping spec-authoring pipelines into one, and split BOS's domain knowledge apart from framework process knowledge, so the latter can become pack-owned in `046` without dragging 40KB of BOS install mechanics with it.

**Independent of the rest of the series.** Every reference to `045`–`048` in the spec is forward-looking; nothing here waits on them. It is also the smallest of the five: one line of code, everything else prose.

Architecture, five ADRs and the file plan are in [`design.md`](./design.md).

## Technical Context

**Language/Version**: TypeScript 5.x (one line) + Markdown (everything else).

**Primary Dependencies**: none new.

**Storage**: `seed/skills/`, `seed/agents/` — BOS source. `data/skills/` is touched only by the operator step, never by this feature's code.

**Testing**: `npm run test:unit` — **never** bare `npx playwright test -c playwright.unit.config.ts`. One new test, `tests/agent/delegation-targets.test.ts`.

**Target Platform**: Next.js server. No runtime/deployment change.

**Project Type**: `bos-core`.

**Constraints**: no dependency changes; FR-010's pure-move discipline on every relocation; **zero** lines of `src/lib/agent/skills/store.ts` (ADR-5).

**Scale/Scope**: 13 FRs, 8 SCs. Three skills retired, one split, three created, four agent definitions edited, three docs corrected, one config default.

## Constitution Check

All seven assessed in [`design.md` §2](./design.md). Two notes carry into implementation:

- **I (NON-NEGOTIABLE)** — this rewrites the instructions that drive the spec pipeline, including for the remaining four features in this series. Self-referential in the same way `045` is, but in prose, where no parity harness exists (R1).
- **IV (NON-NEGOTIABLE)** — deleting a skill is instant and total for the five agents with no `skills:` allowlist. There is no staged rollout for prose; FR-010's discipline is the whole mitigation.

**No violations.**

## Project Structure

```text
user-specs/assistant/044-skill-agent-hygiene/
├── spec.md · design.md · plan.md · tasks.md
```

No `research.md`/`data-model.md`/`contracts/` — nothing to research, no data model, no external contract.

```text
seed/skills/          # bos-domain (new), intent (new), ui-designer-craft (new)
                      # senior-solution-architect (split)
                      # develop-in-browseros, feature-wizard, bos-app (deleted)
seed/agents/          # build-studio, architect, architect-reviewer, ui-designer
src/lib/agent/compaction/config.ts   # :48 — the one line
docs/                 # three stale claims
tests/agent/          # delegation-targets.test.ts (new)
```

**Structure Decision**: three destination skills rather than one, because the seam is method-coupling, not count (ADR-1). Folding everything into `build-studio` would duplicate ~60KB per framework pack in `046`.

## Design notes folded from `design.md`

Only what changes task shape:

1. **Salvage before delete.** Each retirement extracts content first. Deleting a seed directory early destroys the source being salvaged from — which is why the deletions are one consolidated task near the end (T009), not part of each retirement.
2. **Retirement needs an operator step** (§3.6, ADR-5). The seed deletion is necessary, not sufficient: unstamped `data/` copies are never archived. `044` adds no code for this.
3. **The obvious clean-up rule is wrong.** "Delete whatever has a counterpart in `seed/`" spares exactly the three that must go. The rule must be stated as a list.
4. **`summarize-a-web-page` must survive** the clean-up — `created_by: seed`, absent from today's `seed/`, offered by no marketplace, and it carries `scripts/` and `references/`.
5. **`build-studio`'s allowlist is already partly dead** (§3.5) — `tidy-documents` and `workflow-manager-design` exist nowhere. It is being rewritten anyway, so they go in the same edit.
6. **FR-006 is a split, not a move**, so FR-010's "pure move" cannot apply to it literally — it lands as two commits (verbatim extraction, then rewording).
7. **`skill_load` is ungated** (`tools/server/skills.ts:27-41`). De-listing a skill from an allowlist does not make it unreachable; only deletion does.

## Test Strategy

Deliberately thin, because the feature's main failure mode is untestable.

| Layer | File | Covers |
|---|---|---|
| Unit | `tests/agent/delegation-targets.test.ts` | FR-001, SC-001. Enumerates every `agent_delegate( agent: "…" )` target in seeded content; asserts each resolves. **Baseline is zero failures** — all five resolve today. This guards the retirements; it does not fix a bug. |
| Unit | existing suite | SC-005 — `npm run test:unit`, `tsc`, `lint` stay green. |
| **Manual** | — | SC-006. A full `specify → design → plan → tasks` run in Build Studio, including the architect and UI-design delegations. **Load-bearing, not ceremony** (R1). |

There is no prose-parity harness and building one is out of scope. The suite will stay green through a reorganisation that makes the architect materially worse — so the diff review required by FR-010 is the real gate.

## Phase breakdown

| Phase | Content | Revert point |
|---|---|---|
| **0** | Delegation-target baseline | Nothing moved |
| **1** | Create the three destinations: `bos-domain`, `intent`, `ui-designer-craft` | New skills exist; nothing retired |
| **2** | Rewire: agent allowlists, category routing, ordinal cross-references | Everything points at the new destinations |
| **3** | Delete the retired seed directories; compaction default; docs | Source is clean |
| **4** | Gates + the operator step | Feature complete |

Deletions are Phase 3, after every consumer has been repointed in Phase 2. The reverse order leaves a window where an agent's allowlist names a skill that no longer exists — which `filterAllowed` drops silently, so it would not surface as an error.

## Rollout

Same feature branch as the rest of the series.

**This feature does not complete on merge.** After it lands, an operator must clean `data/skills/` — see T010 and `design.md` §3.6. Until then the three retired skills remain in every agent's index and remain loadable, because unstamped copies are never archived. SC-002 is asserted against a cleaned `data/`, not against the branch.

## Risks carried into implementation

- **R1 — the unit suite cannot detect this feature's main failure mode.** Prompt regression is invisible to it. FR-010's diff review and SC-006's manual run are the mitigations; treat both as required.
- **R2 — the retirement completes only after the operator step**, and the intuitive rule for that step is wrong (§3.6).
- **R3 — edited `data/` copies persist** even on stamped deployments. "Retired" is not "gone" everywhere; any assertion phrased as the latter will be false somewhere.
- **R4 — deletion is instant and total** for the five agents with no allowlist. No staged rollout exists for prose.
- **R5 — US3 AS2 cannot be tested here.** *"Given `build-studio` is later relocated into a pack (`046`), nothing breaks"* is a forward-compatibility claim about a feature that does not exist yet. Mark it deferred to `046`; do not count it as passing.
