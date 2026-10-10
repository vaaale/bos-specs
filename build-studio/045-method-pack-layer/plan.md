# Implementation Plan: The method-pack layer (045)

**Branch**: `bos/spec-method-packs` | **Date**: 2026-09-12 | **Spec**: [`spec.md`](./spec.md) | **Design**: [`design.md`](./design.md)

## Summary

Make BOS's spec pipeline framework-agnostic by extracting spec-kit's hardcoded vocabulary into a **method descriptor** that packs supply, with zero behavioural change for existing specs.

Three things land together because they cannot land apart: the descriptor and its evaluator, the consumers that read it (`pipeline.ts`, Build Studio, `/api/specs`), and the install path that registers a pack. The feature is staged A/B/C with independent revert points (Principle IV), and gated on a parity harness that does not exist yet.

Full architecture, six ADRs and the file/module plan are in [`design.md`](./design.md) — not repeated here. This document covers only what task breakdown needs.

## Technical Context

**Language/Version**: TypeScript 5.x, React 19, Next.js App Router (existing stack — no additions)

**Primary Dependencies**: none new. Principle VII forbids dependency changes, and nothing here needs one — the rule evaluator is hand-written, globs use the existing matching already in `pipeline.ts`'s walk.

**Storage**: git-backed spec stores under `BOS_SPECS_ROOT` (`spec-store.json`, `project.json`); `data/` for installed packs and config namespaces. No database.

**Testing**: `npm run test:unit` (Playwright unit config — **never** bare `npx playwright test -c playwright.unit.config.ts`, which drops the `NODE_OPTIONS` the suite needs); `e2e/` for the UI surface. Convention `tests/<area>/*.test.ts`.

**Target Platform**: Next.js server (Node runtime) + browser client. Per-user containers under Bastion.

**Project Type**: `bos-core` — modification of BOS's own source across `src/lib/specs/`, `src/lib/agent/subagents/`, `src/os/`, `src/apps/`, `src/app/api/`, `src/system/`.

**Performance Goals**: none stated. Performance and context-usage measurement are handled elsewhere in BOS and are deliberately out of scope for this spec.

**Constraints**: zero behavioural change for existing specs (SC-001); no marketplace item may execute code in the pipeline path (ADR-1); no new dependencies.

**Scale/Scope**: 31 FRs, 17 SCs, ~10 new modules, ~20 modified files, 24 call sites in one file. Two live spec stores and one live `bos/*` branch form the parity corpus.

## Constitution Check

*GATE: passed before implementation. Re-check after Stage A.*

All seven principles assessed in [`design.md` §2](./design.md). Summary: compliant throughout, with two conditions carried into implementation:

- **VI (NON-NEGOTIABLE)** — FR-018's doc updates must land **with** their stage, not after. `docs/dev/build-studio.md` describes the hardcoded model and becomes wrong at Stage A.
- **IV (NON-NEGOTIABLE)** — each of A/B/C must be independently revertible with a green unit suite (SC-009). If a stage cannot revert cleanly, it is too big and splits.

**No violations. Complexity Tracking table intentionally omitted** — nothing here needs justifying against a simpler alternative that was rejected for expedience.

## Project Structure

### Documentation (this feature)

```text
user-specs/build-studio/045-method-pack-layer/
├── spec.md          # what and why (31 FR, 17 SC, 5 clarifications)
├── design.md        # architecture, 6 ADRs, file plan, review resolution
├── plan.md          # this file
└── tasks.md         # ordered, dependency-annotated task list
```

No `research.md`, `data-model.md`, `quickstart.md` or `contracts/`: there is nothing to research (every mechanism was verified against source during design), the data model is the descriptor shape already specified in design.md §3.3, and there is no external API contract — `/api/specs` is internal and additive.

### Source (repository root)

```text
src/lib/specs/method/     # NEW — types, registry, evaluate, resolve,
                          #       preflight, constitution, install, spec-kit
src/lib/specs/            # pipeline.ts, types.ts, stores.ts, seed.ts,
                          #       projects.ts, create.ts, item-stores.ts,
                          #       spec-mount.ts
src/lib/agent/subagents/  # roots.ts (NEW), store.ts
src/os/                   # vfs.ts — unregisterMount (NEW export)
src/apps/build-studio/    # index.tsx, MethodPicker.tsx (NEW)
src/apps/marketplace/     # index.tsx
src/system/               # items/installed.ts, marketplace/install/
src/app/api/specs/        # route.ts
src/lib/config/           # registry.ts
src/components/apps/settings/  # BuildStudioTab.tsx
tests/specs/, tests/agent/, e2e/   # see Test Strategy
```

**Structure Decision**: a new `src/lib/specs/method/` module holds everything descriptor-related, so the layer has one home and `pipeline.ts` becomes its consumer rather than its owner. Agent-root discovery deliberately lives under `src/lib/agent/subagents/` instead — it is not part of the method layer (design.md §3.3: a descriptor *names* agents but does not resolve them).

## Design notes folded from `design.md`

Only the decisions that change task shape:

1. **`pending` is retained, not renamed** (§3.4, ADR-2). The first draft renamed it `available` and that breaks SC-001 by construction. Tasks must treat **any** state change as a parity failure. Spec-kit declares `requires: []` throughout, so `blocked` is unreachable under it and the criterion is zero substitutions — not "enumerate the `na → blocked` ones", which an earlier draft said.
2. **`unregisterMount` does not exist and must be written** (review M2). `src/os/vfs.ts` is Core-OS, stability HIGH. It is sequenced early and independently so its blast radius is testable on its own.
3. **The parity harness gates the refactor** (R1, R2). The baseline snapshot must be captured *before* any `pipeline.ts` edit, including a live `bos/*` branch. Miss that ordering and the gate is unrecoverable.
4. **Two manifest readers silently drop unknown keys** — `readManifest` (`stores.ts:62-78`) and `readProjectManifest` (`projects.ts:38-48`). Both reconstruct a fixed object. Adding `method` to the type is not enough; the reader must carry it.
5. **The marketplace blockers are `:173` and `:383`**, not `installedKinds`. A `method`-only pack renders no Install button today.
6. **Phase state is an ordered clause list, not a boolean** (§3.4, ADR-2). Three phases — `implement` with 0 done, `test` with no results, `converge` with no discrepancy — are `na` today *while reachable*. A boolean rule flips all three to `pending` across both live stores. Tasks must treat any `na → pending` as a parity failure, exactly as they treat `pending → anything`.
7. **`stateLabels` is how OpenSpec renders "available"** (§3.5, ADR-2a). The phase `label` cannot carry it — that field replaces `PHASE_LABEL`.
8. **`roles{}` needs no `045`-side machinery.** `design.md` §7 routed this question here. It is pack-side metadata, carried in the descriptor type and exposed via FR-014, consumed by `046` (`agents[]`) and `048` (its FR-011b, persona-to-capability binding). Nothing in `045` resolves or validates it — which is what keeps `047`'s "carries no code" claim true while its SC-005 (all five roles reachable) still holds.
9. **Descriptor carries five fields FR-003 originally omitted** (§3.3) — `discrepancies`, `stateLabels`, `sections[].terminal`, `artifacts[].scope`, `testFile`. Each exists because a sibling spec needs it. All have since been added to FR-003; nothing is left to reconcile.

## Test Strategy (mandatory, first-class)

Testing is not a trailing phase here — the parity harness is the feature's acceptance gate and must exist first.

| Layer | File | Covers |
|---|---|---|
| **Parity gate** | `tests/specs/method-parity.test.ts` | SC-001. Snapshots every `Specification` `listSpecifications()` returns — both directory-scanned stores **and the 8 item-owned stores** — plus a live `bos/*` branch; field-level comparison; **zero state changes**. |
| Unit | `tests/specs/method-evaluate.test.ts` | FR-004/004a, SC-007 — every predicate, incl. the set predicate over a glob of unknown cardinality. |
| Unit | `tests/specs/method-registry.test.ts` | FR-003a, SC-015 — `schemaVersion` refusal names expected vs. found; no silent default. |
| Unit | `tests/specs/method-preflight.test.ts` | FR-010a/010c, SC-010 — assignment and upgrade share one implementation; orphan diff includes draft-branch-only features. |
| Unit | `tests/agent/agent-roots.test.ts` | FR-001/001a/001b, SC-006/008 — precedence, `delegate-only` invisible yet delegatable, same-id collision reported, seed writes only to `data/agents/`. |
| Unit | `tests/specs/vfs-unmount.test.ts` | `unregisterMount` — removes by prefix, no-op if absent, no dangling backend. |
| E2E | `e2e/045-method-packs.spec.ts` | FR-017, SC-013 — phase strip incl. `blocked`, Method dropdown, preflight, section badges. |
| E2E fixture | `e2e/_fixtures/method-pack/` | FR-017a — a minimal local `user-apps` item with a `method/` facet, installable through the API at test time. **Phase 0**, not a trailing task: T018 and T024 consume it. Prerequisite for `046`/`047`/`048`. |
| E2E (update) | `e2e/013-artifact-reload.spec.ts`, `e2e/013-spec-anchor.spec.ts` | FR-017 — existing coverage of this surface. |
| E2E (convert) | `e2e/build-studio.spec.ts:49-56` | Retire `test.skip(true, "no marketplace item installed…")` onto the fixture pack. |

Every new test bundles its own fixtures, written through the API at test time. None may depend on pre-seeded spec-store content.

## Phase breakdown

| Phase | Stage | Content | Revert point |
|---|---|---|---|
| **0** | prereq | Parity harness + baseline snapshot; `unregisterMount`; **e2e fixture pack** (T002a — consumed by T018/T024, so it cannot be a trailing task) | Nothing else has moved |
| **1** | A | Descriptor, evaluator, registry, spec-kit descriptor, `pipeline.ts`, `types.ts`, PhaseStrip | Spec-kit runs through the descriptor; no packs, no binding |
| **2** | B | Agent roots, binding, preflight, constitution remedy, MethodPicker, per-pack mounts, section badges, **`/api/specs` descriptor + `defaultMethod`** (their consumers live here) | Method assignable; still no installable packs |
| **3** | C | `method` facet, install/uninstall registration, marketplace UI, FR-015/016 | Packs installable |
| **4** | — | e2e coverage, gates | Feature complete |

Phase 0 is not optional sequencing preference — capturing the parity baseline after Stage A begins makes SC-001 unverifiable for the rest of the feature's life.

## Rollout

Single feature branch `bos/spec-method-packs`, promoted once. No feature flag: the descriptor path replaces the hardcoded path rather than running beside it, and a flag would create exactly the two-code-paths problem `046` exists to remove.

`/Templates` remains an alias through this feature and into `046`, which retires it (its FR-017, added to give ADR-6's time-box a real owner).

## Risks carried into implementation

Carried verbatim from [`design.md` §7](./design.md); repeated here only as the ones that change task ordering.

- **R1 / R2 — parity baseline must be captured first.** Phase 0, before any `pipeline.ts` edit.
- **R3 — the e2e fixture harness gates three downstream specs.** It is Phase 0 here (T002a, consumed by T018/T024), and `046`/`047`/`048` cannot satisfy their e2e criteria until it exists.
- **R4 — `unregisterMount` touches Core-OS.** Isolated into Phase 0 with its own unit test so it is revertible independently of the method layer.
- **R5 — 31 FRs is large.** If Stage B or C sprawls during implementation, split the stage rather than widening it.
- **R6 — Stage B's interim template mapping is a deliberate half-state.** `/Methods/spec-kit/templates` resolves to `.specify/templates` in source until `046` moves the files. An implementer reading only the FR list would read this as a bug.

## Risks at the layer's seams

R1-R6 above concern sequencing. These concern the places where this layer meets
a long-lived server, a user's repositories and a restart — where the obvious
implementation fails silently (`design.md` §9).

- **R7 — Readers of round-tripped files are deleters.** A reader that narrows a
  manifest to modelled fields, followed by a BOS rewrite, erases the rest from
  the user's repo and commits it. Budget an audit of every manifest reader, not
  just the ones this feature adds.
- **R8 — A module-global memo in a long-lived server.** A descriptor memo
  outlives rebinding and serves stale descriptors from any entry point that does
  not clear it. It presents as test flakiness, so expect it to be misdiagnosed.
- **R9 — "Resolves" without "can be set".** A level with no write path passes
  every resolver test while being unreachable. Plan the write path and the UI
  surface for EVERY level in the same task as the resolver.
- **R10 — A descriptor field with one consumer.** `sections` and `terminal` are
  easy to parse and leave half-honoured. When adding a field, list its consumers
  explicitly and test each.
- **R11 — Migrations that predate the feature.** The `033` Project migration
  destructively rearranges any store whose root it does not understand, on every
  boot, before anything renders. Audit existing migrations whenever a feature
  makes a previously-universal layout assumption configurable.
- **R12 — Test topology, not test count.** No unit suite can reach these seams,
  however large. Plan for a check that crosses a restart and a second module
  instance (T041); a single-process suite structurally cannot.
