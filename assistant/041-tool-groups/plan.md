# Implementation Plan: Tool Groups, Better Tool Discovery, and Group-Aware Settings (041)

**Branch**: `bos/tool-groups` | **Date**: 2026-08-30 | **Spec**: [spec.md](spec.md) | **Design**: [design.md](design.md)

**Input**: Feature specification from `spec.md` (Draft, App Target: `bos-core`), structural design in `design.md` (7 ADRs, 9 risks, two review rounds, no open questions). This is a **kernel** feature: it changes the capability registry, instruction composition, the discovery tool, the service-tool bridge, and two Settings surfaces. Two marketplace items are **consumers** whose manifests must migrate (FR-042). Implementation is delegated to the Developer (Claude) sub-agent on `bos/tool-groups` per Constitution III/IV.

## Summary

Give tools a real group model and use it in three places: a generated `## Tool groups` block in the system prompt (groups the agent actually has, their descriptions, and — where tools are hidden — an invocable discovery instruction); a substantially better `find_tools` (term-level IDF ranking, stopwords, stemming, curated aliases, match reasons, plus an uncapped group-scoped mode); and a group-first Settings → Tools with collapsible groups and editable group text. Marketplace items declare their own groups in `service.json` instead of everything landing in `Service Tools`. Two dead `find_tools` implementations are deleted rather than maintained.

`design.md` is the architecture of record; this plan does not restate it. The decisions that drive task breakdown are folded in below.

## Technical Context

**Language/Version**: TypeScript, Next.js App Router (SSR), React, Node.js backend.

**Primary Dependencies**: No new packages. Ranking is hand-written and deterministic (ADR-3) precisely to avoid one. Reuses Ajv (already in `manifestValidator.ts`) for manifest validation, and the existing `useAutoSave`/`AutoSaveStatus` Settings affordances.

**Storage**: One new JSON file, `dataDir()/tool-group-overrides.json`, written atomically via `writeFileAtomic` following `tool-metadata-overrides.ts`. Group definitions themselves are code (built-ins) or manifest-declared (service groups) — never persisted state.

**Testing**: Unit tests under `tests/<area>/*.test.ts` (`playwright.unit.config.ts`, `testDir: "./tests"`, run with `npm run test:unit`, which sets `NODE_OPTIONS=--conditions=react-server` — required for any test importing a `server-only` module). E2E under `e2e/*.spec.ts` (`npm run test:e2e`). New suites land in `tests/agent/` and `tests/assistant/`, both of which already exist. **Automated testing is a first-class deliverable** (§ Test Strategy) — notably because `discovery-score.ts` has never had a test (ADR-3).

**Target Platform**: BOS kernel (`src/`), plus `seed/` prompts, `docs/`, and two marketplace item repos.

**Performance Goals**: No latency target. Two cost properties are load-bearing and asserted instead: the prompt block adds no more than a small constant per group (SC-012), and a discovery response carries no schemas (FR-024a), which is what makes uncapped group mode affordable.

**Constraints**: The ranking module must stay framework-free and pure (no I/O, no model call) so one implementation serves the tool and the tests. The prompt block must be static for a run (FR-011) because the system block is cached. Reveal derivation must read the canonical transcript, never the compacted view (R9).

**Scale/Scope**: ~150 capabilities across ~25 groups today; the largest single group is OKF's ~35 `okf_*` tools. Single-user and multi-user (Bastion) deployments both affected, since service groups are per-installed-item.

## Constitution Check

*GATE: re-checked after design.*

| Principle | Verdict | Notes |
|---|---|---|
| **I. Spec-Driven** | PASS | `spec.md` + `design.md` precede implementation; both reviewed twice. |
| **II. Server Authority & SSR Boundary** | PASS | Overrides read/written only behind `src/app/api/**`; `tool-groups.ts` and `discovery-search.ts` are framework-free; no secrets. |
| **III. Always Delegate; Claude Codes** | PASS | All coding delegated to the Developer (Claude) sub-agent. |
| **IV. Minimize Blast Radius** | PASS | `bos/tool-groups` feature branch; item repos carry their own branches; promote is code-only. |
| **V. The VFS Is Not the Source** | PASS | Edits in `src/`, `seed/`, `docs/`, `tests/`; runtime overrides under `data/`. |
| **VI. Specs & Docs Stay in Sync** | PASS *with obligations* | Docs tasks are in Phase 6 and are not optional: `services.md` §15, `actions-and-tools.md` (rewrite, not patch), a new `docs/usage/settings/tools.md`, `architecture-overview.md` §8.2/§8.3, and 039's superseded group statement. |
| **VII. Respect Boundaries** | PASS | No dependency, `package.json`, or build-config changes. `npx tsc --noEmit` + `npm run lint` + `npm run test:unit` gate the change. Do not run `npm run build` while `next dev` is running. |

## Project Structure

### Documentation (this feature)

```text
bos-system-specs/assistant/041-tool-groups/
├── spec.md      # requirements (D1–D4 resolved decisions)
├── design.md    # architecture of record — 7 ADRs, 9 risks
├── plan.md      # this file
└── tasks.md     # phase/task breakdown
```

### Source Code

See `design.md` §4 for the authoritative created/modified file plan. It is not duplicated here; tasks reference it path by path.

## Design notes folded from `design.md`

The decisions that shape sequencing, rather than a summary of the design:

1. **ADR-1** makes `Capability.group` a stable id. This is the foundational, everything-else-depends-on-it change, and it touches four render sites (`ToolsTab:327`, `ToolAccordions:181`, `InfoPanelV2:56/62`, and `discovery-score.ts:44`).
2. **ADR-7 + D4** shrink the discovery work substantially: results carry no schemas (the existing reveal already un-gates the tool so the provider delivers them), and two of the three `find_tools` implementations are dead code to be deleted, not ported.
3. **FR-041** forbids any fallback group. Three existing fallbacks must die with the change, not be carried forward: `"General"` in `ToolsTab` and `ToolAccordions`, and `groupDescription()`'s synthesized text.
4. **R8** fixes the ship order: **item manifests first, BOS second**, because FR-040 makes pre-feature items fail loudly and `manifestValidator.ts` performs no unknown-key rejection (so `toolGroups` is inert on today's BOS).
5. **R9** pins reveal derivation to the canonical transcript.

### Sequencing note: US2 lands before US1, despite equal priority

Both are P1. US1 (the block) renders an instruction of the form `find_tools(group: "web")`; shipping that before US2 (group-scoped discovery) would put a non-working instruction in every agent's system prompt. Discovery therefore lands first, and the block second.

## Test Strategy (mandatory, first-class)

### FR → test mapping

| FR / SC | Test | Suite |
|---|---|---|
| FR-001, FR-003, FR-004, FR-030, FR-041 | Group registry, dynamic lifecycle, `resolveGroup` precedence, no fallback bucket, every `Capability.group` resolves | `tests/agent/tool-groups.test.ts` |
| FR-049, FR-046 | Override persists across a group's absence; effective view resolved per call | `tests/agent/tool-groups.test.ts` |
| FR-016 – FR-023, FR-028, SC-004, SC-005 | Tokenizer, stopwords, stemming, IDF, coverage bonus, threshold, determinism, match reasons, and the natural-language benchmark | `tests/agent/discovery-search.test.ts` |
| FR-024, FR-024a, FR-024b, FR-025 – FR-027, FR-029, FR-031 – FR-034 | All six response modes, no schemas in payload, uncapped group mode, allowlist respected | `tests/assistant/discovery-modes.test.ts` |
| FR-036, ADR-4, R1, R9 | Reveal derivation over legacy array + envelope + compacted view; `alreadyVisible` is not a reveal source | `tests/assistant/revealed-ids-shapes.test.ts` |
| FR-005 – FR-015, SC-003, SC-011, SC-012, D1 | Block membership, discovery-instruction rule, preamble for all three gate kinds, visible-named/hidden-counted rendering, ordering | `tests/assistant/tool-groups-block.test.ts` |
| FR-037 – FR-041, SC-007 | Manifest validation, group resolution at `tool_declare`, loud rejection + `lastError` | `tests/agent/service-tool-groups.test.ts` |
| FR-050, FR-051, SC-010 | Every tool name in generated prompt text resolves against the live registry | `tests/assistant/prompt-tool-names.test.ts` |
| FR-042 – FR-048, SC-009 | Settings surfaces | Manual + `e2e/` smoke (see below) |

### User story → acceptance mapping

- **US1** → `tests/assistant/tool-groups-block.test.ts` covers all four acceptance scenarios directly (granted+deferred, granted+visible-only, not-granted, zero-deferred).
- **US2** → `tests/agent/discovery-search.test.ts` (scenarios 1–2) + `tests/assistant/discovery-modes.test.ts` (scenarios 3–8).
- **US3** → `tests/agent/service-tool-groups.test.ts` (scenarios 1–4), with scenario 4 (pre-feature item) using a worker fixture in the style of `tests/agent/service-tool-bridge.test.ts`.
- **US4** → an `e2e/` smoke asserting collapsed-on-load and that an edited group description reaches the next run's prompt; the rest is manual.

### SC-004 benchmark — write the queries before tuning the ranking

R5: the benchmark query set is committed **first**, as its own task, before `discovery-search.ts` is tuned. Otherwise the ranking gets fitted to whatever queries were convenient. Each entry is `{query, expectedToolId}` phrased the way a user would ask, sharing **no literal id word** with the target (e.g. "send a note to my colleague" → `gmail_messages_send`). Target: correct tool in the top three.

### Gates

`npx tsc --noEmit`, `npm run lint`, `npm run test:unit` must pass before promote. `tsc` is also the safety net for D4's deletions (R4) — a missed transitive importer is a compile error, not a silent one.

## Phase breakdown

| Phase | Content | Gate to exit |
|---|---|---|
| 1 | Foundational: group model, ids, overrides store, API | `tool-groups.test.ts` green; `tsc` clean |
| 2 | US2: delete dead paths, ranking module, rewritten `find_tools`, reveal parsers | discovery suites green |
| 3 | US1: prompt block, composition wiring, seed prompt trim | block suite green |
| 4 | US3: service group declarations + item migration | service suite green; items branch ready |
| 5 | US4: Settings surfaces | e2e smoke green |
| 6 | Corrections, docs, gates | all gates green |

## Rollout

1. Merge and release the **item** manifest changes (`workflows`, `okf-knowledge-base`) first — inert on current BOS.
2. Then promote BOS.
3. Verify against the Dokploy deployment, not this checkout (R6): locally `data/user-apps/items/` is empty and `data/system/okf-knowledge-base` is a dangling symlink.

## Risks carried into implementation

All nine live in `design.md` §7. The three that most affect execution: **R1/R9** (reveal derivation — silent failure, test it), **R3** (group-id migration — assert every `Capability.group` resolves), **R8** (ship order). R4 is resolved (D4) and R2/R7 are accepted limitations, not work.
