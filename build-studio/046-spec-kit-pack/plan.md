# Implementation Plan: The spec-kit pack (046)

**Branch**: `bos/spec-method-packs` | **Date**: 2026-09-12 | **Spec**: [`spec.md`](./spec.md) | **Design**: [`design.md`](./design.md)

## Summary

Move spec-kit's engine — templates, four process agents, driver skill — out of BOS source and into a real pack, registered through the same install sequence a marketplace pack uses.

The feature's defining property is that **nothing changes for a user**. Its risk is entirely in the relocation: moving an id out of `seed/` is indistinguishable, to BOS's reconciliation, from deleting it, and the skill's hand-off has no working path today. That hazard — not the file moves — is what the phases are ordered around.

Full architecture, five ADRs and the file/module plan are in [`design.md`](./design.md). This document covers only what task breakdown needs.

## Technical Context

**Language/Version**: TypeScript 5.x, Next.js App Router (existing stack — no additions)

**Primary Dependencies**: none new (Principle VII).

**Storage**: `seed/method-packs/spec-kit/` (BOS source, shipped by `Dockerfile:82`'s `COPY . .`); `data/agents/`, `data/skills/` for reconciled copies.

**Testing**: `npm run test:unit` — **never** bare `npx playwright test -c playwright.unit.config.ts`. Convention `tests/<area>/*.test.ts`.

**Target Platform**: Next.js server (Node runtime). Per-user containers under Bastion.

**Project Type**: `bos-core` — a source-to-pack migration.

**Constraints**: no behavioural change except FR-014's refusal; no new dependencies; `045` must be complete (this feature consumes its registration sequence, its agent-root precedence, and its fixture harness).

**Scale/Scope**: 20 FRs, 15 SCs. ~10 files relocated, ~12 modified, `.specify/` (25 files) deleted.

## Constitution Check

*GATE: passed before implementation.*

All seven assessed in [`design.md` §2](./design.md). Two conditions carry into implementation:

- **IV (NON-NEGOTIABLE)** — the pack resolves from `process.cwd()` while `dataDir()` may be a COW clone, so a candidate's pack and its data can be from different versions. Carried by **FR-018**; its mechanisms are `itemPath` via `sourceRootFor` and `Provenance.version`, **not** `root?` (the destination knob).
- **VI (NON-NEGOTIABLE)** — FR-016's doc corrections land **with** the move. `CLAUDE.md`, `docs/dev/build-studio.md`, `docs/dev/repository-and-data-layout.md:7` and `docs/usage/apps/build-studio.md:62` all assert claims this feature falsifies.

**No violations.**

## Project Structure

```text
user-specs/build-studio/046-spec-kit-pack/
├── spec.md · design.md · plan.md · tasks.md
```

No `research.md`/`data-model.md`/`contracts/`: every mechanism was verified against source across five review rounds, and this feature adds no data model or external contract.

```text
seed/method-packs/spec-kit/   # NEW — method.json, templates/, agents/, skills/
src/lib/specs/method/         # builtin-pack.ts (NEW), spec-kit.ts (deleted)
src/system/marketplace/install/  # bundledAssets.ts — new exported entry point
src/lib/agent/{skills,subagents}/ # store.ts — the seeding step
.specify/                     # DELETED
tests/specs/, e2e/            # see Test Strategy
```

**Structure Decision**: the pack lives under `seed/`, beside `seed/agents/` and `seed/skills/`, because those already resolve from `process.cwd()` and ship with the image — no Dockerfile change, no new packaging concept.

## Design notes folded from `design.md`

Only what changes task shape. Each of these was found by review, not by reading the spec:

1. **The archive *is* the hand-off, for both kinds** (ADR-2). Moving an id out of `seed/` makes `decideSeedAction` return `archive` for a stamped-unedited copy. Suppressing that would pin the stale copy forever under `045` FR-001a's precedence. Do not "fix" the archive.
2. **The skill needs a dedicated seeding step** (FR-008b). `reconcileInstalledItemAssets` is memoized per data root (`bundledAssets.ts:244`) and the **agent** store consumes the single pass first (`subagents/store.ts:187`), so a skill archived afterwards can never be re-seeded in that process.
3. **The directory is `skills/`, plural.** `sourceRootFor` pluralises the kind (`:63-64`). A `skill/` directory makes the step a **silent** no-op — `readdir` catches to `[]`, nothing throws.
4. **The primitives are module-private.** `copyAsset` (`:118`), `seedOneKind` (`:139`), `recordPendingConflicts` (`:301`). The only exported wrapper seeds **both** kinds (`:204-205`) and would resurrect the four archived agents. A new skill-kind-scoped export is required.
5. **Never bare `copyAsset`** — unconditional `rm -rf` + `cp`, no existence or provenance check.
6. **`.specify/scripts` appears zero times literally.** The references are unprefixed `scripts/bash/…` in YAML frontmatter, so a literal-substring check passes while every block survives.
7. **Enumeration is the deliverable** for FR-017 and FR-004 — a missed `/Templates` or `.specify/` reference is a silently broken pipeline step, not a compile error.
8. **`testRoot` belongs on `SpecStore`**, not the descriptor: test *location* is a store-kind property, test *name* is the method's.

## Test Strategy (mandatory, first-class)

| Layer | File | Covers |
|---|---|---|
| **Hand-off fixture** | `tests/specs/relocation-handoff.test.ts` | SC-005a. Simulates an upgrade from a real pre-relocation `data/`, all **three** branches: stamped-unedited → archive → dedicated step copies; unstamped → no archive → `BundledAssetConflicts`; edited → `local` → conflict. **This is the gate.** |
| Unit | `tests/specs/builtin-pack.test.ts` | FR-008/009, SC-004/005 — registration before first store read, through the marketplace call sequence; uninstall refusal. |
| E2E | `e2e/046-spec-kit-pack.spec.ts` | FR-015, on `045` FR-017a's fixture harness. |
| Regression | `045`'s `tests/specs/method-parity.test.ts` | SC-007 — spec-kit's `Specification` output is unchanged by the relocation. |

The hand-off fixture must exist **before** any relocation. It is the only artifact that can observe a first-boot-after-upgrade failure, and that failure class is the one that silently destroys the driver skill.

## Phase breakdown

| Phase | Content | Revert point |
|---|---|---|
| **0** | Hand-off fixture; the skill-kind-scoped export in `bundledAssets.ts` | Nothing has moved |
| **1** | Pack skeleton, `method.json`, registration via the install sequence, `builtin` uninstall refusal | Pack registered; assets still in source |
| **2** | The relocations: templates, four agents, driver skill + the `ensureSeed()` seeding step | Assets in the pack |
| **3** | De-coupling: FR-004 scrub, FR-013 stragglers, FR-014 `testRoot`, FR-017 alias retirement, `.specify/` deletion | Source is clean |
| **4** | Docs, e2e, gates | Feature complete |

Phase 0 before Phase 2 is not preference: once the assets move, the pre-relocation `data/` state the fixture needs to simulate no longer exists in the tree to copy from.

## Rollout

Same feature branch as `045`, promoted together — `046` is meaningless without it, and splitting the promote would leave a descriptor with no pack.

The first boot after this lands is the risky moment on **every existing deployment**, not just in development: four agents archive, one skill hands off. Phase 0's fixture is what makes that observable before it happens.

## Risks carried into implementation

From [`design.md` §7](./design.md); repeated where they change ordering.

- **R1 — the skill's seeding must not ride the memoized pass.** Its own step, after the archive at `skills/store.ts:195`.
- **R2 — a large diff with no functional change.** Keep each relocation a pure `git mv` in its own commit; put the FR-004 scrub and FR-013 de-coupling in separate commits so they read as edits, not as noise inside a move.
- **R3 — `seed-sync.ts` is not valid UTF-8.** `file` reports `data`; `grep` silently skips it. Anyone auditing the module this feature most depends on will get false negatives.
- **R4 — FR-014 is the one behavioural change.** `045`'s parity harness tests `Specification` objects, not that route; it will not catch a regression there.
- **R5 — FR-017 is the release where prompts actually change.** A missed reference is silent.
- **R6 — both dispositions depend on per-deployment state.** 10 of this machine's `data/agents/` are stamped; **none** of its 16 `data/skills/` are. The unstamped branch is therefore the live one here and the stamped one is the live one elsewhere — the fixture must cover both.
