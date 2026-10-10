# Tasks — 046 The spec-kit pack

Spec: [`spec.md`](./spec.md) · Design: [`design.md`](./design.md) · Plan: [`plan.md`](./plan.md)

**Three rules that are not negotiable:**

1. **T001 runs before any relocation.** Once assets move, the pre-relocation `data/` state the fixture must simulate no longer exists in the tree to copy from.
2. **Never suppress an archive.** `archiveDroppedSeedAgents`/`archiveDroppedSeedSkills` firing *is* the hand-off (ADR-2). Suppressing it pins the stale copy forever.
3. **Never call bare `copyAsset`.** It is `rm -rf` + `cp` with no existence or provenance check (`bundledAssets.ts:118-123`).

---

## PHASE 0 — Prerequisites (blocks all relocation)

### T001 — Relocation hand-off fixture
- **Files:** `tests/specs/relocation-handoff.test.ts` (new)
- **Story:** FR-008b, SC-005a / the gate
- **Work:**
  - Simulate an upgrade from a real pre-relocation `data/`: stamped agents, and a `build-studio` skill in each of three states.
  - **All three branches**, since deployments differ — this machine has 10 of 14 `data/agents/` stamped and **0** of 16 `data/skills/`:
    - *stamped-unedited* → `decideSeedAction` returns `archive` → the dedicated step copies the pack's skill. **This is the branch the memo defect destroyed**, so it is mandatory.
    - *unstamped* → `local` → no archive → the step finds the dest present, no `.installed-from.json` → `BundledAssetConflicts`.
    - *edited* → `local` → same terminal state as unstamped.
  - Assert the four relocated agents resolve from the pack root after their archive.
- **Acceptance:** all three branches asserted; fixture green against the pre-relocation tree. **Nothing in Phase 2 starts until this passes.**

### T002 — Skill-kind-scoped export in `bundledAssets.ts`
- **Files:** `src/system/marketplace/install/bundledAssets.ts`
- **Story:** FR-008b / ADR-3
- **Work:**
  - `copyAsset` (`:118`), `seedOneKind` (`:139`) and `recordPendingConflicts` (`:301`) are **module-private**. Add an exported entry point over `seedOneKind` + `recordPendingConflicts`, scoped to `kind: "skill"` and to a caller-supplied source root.
  - **Do not reuse `seedItemBundledAssets` (`:194`)** — it seeds both kinds (`:204-205`), so it would copy the pack's `agents/` into `data/agents/`, resurrecting the four copies T007 archives and violating `045` FR-001b.
  - Accept `version` so the caller can record the running tree's rev in `Provenance.version` (`:56-60`) — FR-018.
- **Acceptance:** the new export seeds only skills; `seedItemBundledAssets`'s behaviour is unchanged for real marketplace items.

---

## PHASE 1 — Pack skeleton and registration

### T003 — Pack scaffold + `method.json`
- **Files:** `seed/method-packs/spec-kit/method.json` (new)
- **Story:** FR-008, FR-011 / ADR-1
- **Work:**
  - Relocate `045`'s descriptor content from `src/lib/specs/method/spec-kit.ts` into `method.json`, `builtin: true`.
  - `constitution: ".specify/memory/constitution.md"` verbatim (FR-011) with `constitutionRoot: "system"` (`045` FR-006b) — nothing moves for existing stores.
  - Directory layout per design §3.3. **`skills/` is plural** — see T008.
- **Acceptance:** descriptor parses; identical in content to the deleted TS module.

### T004 — Register through the install sequence
- **Files:** `src/lib/specs/method/builtin-pack.ts` (new), `src/instrumentation.ts`
- **Story:** FR-008, FR-009 / ADR-1
- **Work:**
  - Resolve the in-tree pack root **relative to the running tree** (`process.cwd()`) and hand it to `045` FR-012's registration sequence. FR-018: pass the tree's rev through to provenance.
  - Invoke from `instrumentation.ts`, extending `045` T006a rather than adding a second hook.
  - **`045` FR-013a's origin gate never triggers for this pack** — it applies only to packs carrying a `plugin/` facet, and this one has none. It is not skipped; it does not apply (ADR-1).
  - **FR-018 negative constraint:** pass the pack **source** via `itemPath`/`sourceRootFor` and the rev via `Provenance.version`. **Not `root?`** — that is the destination knob (`destRootFor`, `bundledAssets.ts:62`).
  - `builtin-pack.ts` is a **thin source resolver**. SC-005 means "no second registration *sequence*", not "no new file".
- **Acceptance:** SC-005 — the pack reaches registration through the same call sequence a marketplace pack does, asserted by following the sequence in a test. **Not a grep for `builtin`** (FR-001 makes it a legitimate field).

### T005 — `builtin` uninstall refusal
- **Files:** `src/lib/specs/method/registry.ts`, `tests/specs/builtin-pack.test.ts` (new)
- **Story:** FR-001, FR-009
- **Work:**
  - Refuse uninstall of a `builtin` pack with a message naming `bos-system-specs` as the dependant. Upgrade still permitted.
  - FR-009 is scoped to **registration and resolution** — this refusal is itself a `builtin` special-case and is required.
  - Test also covers registration-before-first-store-read (the Edge Case: `ensureStoresOnce()` has several entry points).
- **Acceptance:** SC-004; refusal reachable from the registry API and any op that enumerates packs. No Marketplace row exists to click (FR-008).

### T006 — Delete the built-in TS descriptor
- **Files:** `src/lib/specs/method/spec-kit.ts` (delete)
- **Story:** FR-009
- **Work:** Remove it once T003/T004 register the pack. If it survives as a fallback, the two-code-paths problem has moved rather than gone.
- **Acceptance:** SC-005 — zero spec-kit-only registration functions. **Phase 1 revert point.**

---

## PHASE 2 — The relocations

### T007 — Relocate the four process agents
- **Files:** `seed/agents/{architect,architect-reviewer,ui-designer,devil-s-advocate}/` → `seed/method-packs/spec-kit/agents/`
- **Story:** FR-005, FR-008a
- **Work:**
  - Pure `git mv`, all four `picker`-visible (`045` FR-001 makes visibility a per-root property; the pack root declares it).
  - **Let `archiveDroppedSeedAgents` fire.** It is the hand-off: the stale `data/agents/<id>` copy archives and the pack root supplies the agent (`045` FR-001a precedence). Suppressing it pins the stale copy forever, because `applySeedAgent` (`subagents/store.ts:53`) never runs for an id absent from `seed/`.
  - `subagents/store.ts` is **unmodified** by this task.
  - **FR-008a (agent half):** the four relocated agents keep reconciling through the `.seed-rev` two-hash contract (`subagents/store.ts:53-77`, `seed-sync.ts:107-123`), and the seed MUST continue writing only to `data/agents/` (`045` FR-001b) — never into the pack root.
- **Acceptance:** SC-001 — `seed/agents/` holds only the ten capability agents (`assistant`, `build-studio`, `conversation-reviewer`, `default_agent`, `developer`, `devops`, `docs-cataloger`, `docs-explorer`, `researcher`, `scheduler-architect`). T001's agent branch green.

### T008 — Relocate the driver skill + the seeding step
- **Files:** `seed/skills/build-studio/` → `seed/method-packs/spec-kit/skills/build-studio/`; `src/lib/agent/skills/store.ts`
- **Story:** FR-006, FR-007, FR-008a, FR-008b / ADR-3
- **Work:**
  - **The destination is `skills/build-studio/` — plural.** `sourceRootFor` pluralises the kind (`bundledAssets.ts:63-64`); a `skill/` directory makes the step a **silent** no-op (`readdir` catches to `[]`).
  - Add a dedicated pack-skill seeding step to `ensureSeed()`, **after** the archive at `:195`, calling T002's export.
  - It must **not** ride `reconcileInstalledItemAssets()` (`:200`): that pass is memoized per data root (`:244`) and the agent store consumes it first (`subagents/store.ts:187`), so a skill archived afterwards can never be re-seeded in that process.
  - `archiveDroppedSeedSkills` (`:147`) is **unmodified**.
  - **FR-007:** the relocated `SKILL.md` and its references MUST reach `044`'s BOS-owned skills — `bos-domain`, the intent skill, the ui-designer craft skill — **by skill id** via `skill_load`/`skill_read_file`, never by path into their internals. Grep the relocated pack content for any path reaching into `seed/skills/` or `data/skills/`; there must be none.
  - **FR-008a (skill half):** updates reconcile through the bundled-asset provenance contract (`.installed-from.json`, `bundledAssets.ts:54`, `:170-181`) — replace when untouched, conflict when diverged. Not `.seed-rev`: T002's export writes provenance and no stamp.
- **Acceptance:** all three T001 branches green; zero path-based reaches from pack content into BOS-owned skills. This is the feature's load-bearing code change.

### T009 — Relocate the templates
- **Files:** `.specify/templates/` → `seed/method-packs/spec-kit/templates/`
- **Story:** FR-002
- **Work:** Pure `git mv`; mount at `/Methods/spec-kit/templates` (`045` FR-011 built the mount).
- **Acceptance:** SC-002 — no runtime code path reads `.specify/templates` from BOS source. **Phase 2 revert point.**

### T010 — Report shadowed pack agents
- **Files:** `src/components/apps/settings/BuildStudioTab.tsx`
- **Story:** FR-010
- **Work:** A pack agent shadowed by a `data/agents/<id>` copy is reported. Distinct from `045` FR-001a's *collision* report (two packs, same id) though they share a home — this one's trigger is precedence.
- **Acceptance:** an edited `architect` is reported as shadowing the pack's copy.

---

## PHASE 3 — De-coupling

### T011 — Discriminating `.specify/` rewrite in the prompts
- **Files:** `seed/method-packs/spec-kit/templates/commands/*.md`, `plan-template.md`
- **Story:** FR-004
- **Work:** Three dispositions, not a blanket replace:
  - **Rewrite to the pack mount:** `.specify/templates/{plan,spec,tasks,constitution}-template.md` and `commands/*.md`.
  - **Delete as unreachable:** 35 `.specify/extensions.yml` probes across **ten** files (`plan` 3, `specify` 3, `implement` 3, `converge` 4, `taskstoissues` 4, `analyze` 4, `clarify` 3, `checklist` 4, `constitution` 4, `tasks` 3), plus `.specify/init-options.json` and the `.specify/feature.json` write (`specify.md:87`, `:99`).
  - **Preserve verbatim:** the 5 `.specify/memory/constitution.md` references in `commands/constitution.md` — that path stays valid (FR-011).
  - The `scripts:` frontmatter blocks in the eight named files: **match on `scripts/bash/` and `scripts/powershell/`**. The literal `.specify/scripts` appears **zero** times, so a literal check passes while every block survives.
- **Acceptance:** SC-006 by the corrected predicate.

### T012 — Delete `.specify/` and its write grant
- **Files:** `.specify/` (delete), `src/lib/dev/repo-fs.ts`
- **Story:** FR-003, FR-004
- **Work:** Remove the directory (25 files today; `scripts/` dropped, `templates/` already moved by T009). Drop `.specify/` from `WRITE_ALLOW_PREFIXES` (`repo-fs.ts:28`) — leaving it grants write access to a path that no longer exists.
- **Acceptance:** SC-003.

### T013 — Retire the `/Templates` alias
- **Files:** `src/lib/specs/spec-mount.ts`, plus every site in the enumerated surface
- **Story:** FR-017 / R5
- **Work:** Enumeration **is** the deliverable — a miss is a silently broken pipeline step, not a compile error. The surface (design §4):
  - **BOS-owned, staying:** `seed/agents/build-studio/AGENT.md:42`,`:44`; `seed/agents/ui-designer/AGENT.md:74`; `references/target-bos-core.md:19` and `target-marketplace-item.md:29` (moved to `bos-domain` by `044` FR-005); `seed/skills/bos-app/SKILL.md:26` (phase 0, moved to the intent skill by `044` FR-004, carrying a dead `/Templates` read and two dead `.specify/templates/...` paths).
  - **Pack content:** `SKILL.md:9` (describes `/Templates` as "the spec-kit ENGINE" — false post-retirement), `:62`, `:66`; `references/{plan,tasks,constitution,specify}.md`.
  - **Agent-facing prompt text:** `src/lib/agent/tool-groups.ts:56` (the `files` group description).
  - **Docs/comments:** `docs/dev/build-studio.md:82`,`:104`,`:110`; `docs/dev/os-shell/virtual-file-system.md:26`; `src/os/vfs.ts:91`,`:103`,`:217`; `src/os/fs-types.ts:5`; `src/os/fs/readonly-fs.ts:8`.
- **Acceptance:** SC-011a — verified by **resolving** every reference, not by grep.

### T014 — De-couple the stragglers
- **Files:** `src/lib/self-heal/brief.ts`, `src/lib/agent/config.ts`, `src/lib/agent/tool-groups.ts`, `seed/agents/build-studio/AGENT.md`
- **Story:** FR-013 (incl. `run-tests/route.ts:127`'s `test-results.md`, the artifact `045`'s `testFile?` was added for)
- **Work:**
  - `brief.ts:39-40`, `:58-59`, `:172` — the spine hardcodes the step sequence and artifact names.
  - `config.ts:15` — CORE_POLICY names "the spec-kit pipeline".
  - `tool-groups.ts:128` (search alias) **only** — `:56` is T013's (the `/Templates` surface). Two tasks must not edit the same line; the parallelisation note below assumes they do not.
  - **`seed/agents/build-studio/AGENT.md` is the load-bearing one.** `:3` "using spec-kit"; `:6` `skills: [build-studio, …]`; `:31` the nine-step sequence + `skill_load("build-studio")`; `:42`/`:44` `/Templates`; `:47` the constitution path. It stays BOS-owned while the driver skill becomes pack content, so under `047` it would name the wrong skill and the wrong phase list. It must reach the driver **via the active pack**, not by a literal id.
- **Acceptance:** SC-010 — the AGENT.md check is the one that matters: no literal driver-skill id, no literal phase list.

### T015 — `testRoot` on `SpecStore`
- **Files:** `src/lib/specs/stores.ts`, `src/app/api/specs/run-tests/route.ts`
- **Story:** FR-014
- **Work:**
  - Two axes: the test-file *name* comes from `045`'s `testFile?`; the *location* is a **store-kind** property and belongs on `SpecStore`, not the descriptor.
  - `testRoot: "repo"` for `system`/`user`/`marketplace`; `undefined` for `owner: "item"` (content lives in `data/user-apps/items/<id>/`, tests are not BOS's to run).
  - `undefined` ⇒ refuse with that reason. Today `:65` derives `featureId` from the last path segment, `:70` builds `e2e/<featureId>.spec.ts`, and `:72`/`:79` probe and execute inside `process.cwd()` regardless of store kind.
- **Acceptance:** SC-009. **This is the feature's one behavioural change** — `045`'s parity harness will not catch a regression here. **Phase 3 revert point.**

---

## PHASE 4 — Docs, e2e, gates

### T016 — Documentation
- **Files:** `CLAUDE.md`, `docs/dev/build-studio.md`, `docs/dev/repository-and-data-layout.md`, `docs/dev/assistant/sub-agents-and-delegation.md`, `docs/usage/apps/build-studio.md`, `docs/dev/method-packs.md` (created by `045` T032; **extended** here, not created)
- **Story:** FR-016, Constitution VI (NON-NEGOTIABLE)
- **Work:**
  - `CLAUDE.md` asserts the engine "stays in source at `.specify/templates`" — **corrected**, not extended; Principle VI makes its accuracy binding.
  - `build-studio.md:193-195` (Conventions) makes the same claim; `repository-and-data-layout.md:7` likewise; `sub-agents-and-delegation.md:37` calls build-studio a spec-kit author; `docs/usage/apps/build-studio.md:62` says Build Studio "only ever edits files under `specs/` and `.specify/`".
  - `method-packs.md` — created by `045` T032, **extended** here — records the **agents-discovered / skill-copied asymmetry** as a known shape, with multi-root skills noted as the eventual convergence (ADR-3).
  - (`AGENTS.md:5` carries the identical falsified sentence but is untracked — out of scope by that fact, not by choice.)
- **Acceptance:** SC-011.

### T017 — E2E
- **Files:** `e2e/046-spec-kit-pack.spec.ts` (new)
- **Story:** FR-015
- **Work:** On `045` FR-017a's fixture harness: registration before first store read, uninstall refusal, and **every** relocated prompt's template path resolving (FR-012/SC-007a — at least five: `SKILL.md:66` plus four `references/*.md`), by resolution rather than grep.
- **Acceptance:** SC-007a.

### T018 — Gates
- **Files:** —
- **Story:** SC-007, SC-008, SC-012
- **Work:** `npx tsc --noEmit`; `npm run lint`; `npm run test:unit`. Re-run `045`'s parity harness (SC-007). Manual: a full `specify → design → plan → tasks` run with `architect` and `ui-designer` resolving through a pack agent root (SC-008).
- **Acceptance:** all green; each phase independently revertible.

---

## Dependency summary

```
T001 ──────────────────────────────► gates T007, T008 (and all of PHASE 2)
T002 ──────────────────────────────► gates T008

T003 ─► T004 ─► T005 ─► T006                         [Phase 1 revert]

T006 ─┬─► T007 ─► T010
      ├─► T008        (needs T001, T002)
      └─► T009 ───────────────────────────────────── [Phase 2 revert]

T009 ─► T011 ─► T012
T008, T009 ─► T013     (T013's surface includes pack content — `SKILL.md`,
                        `references/*.md` — which exists only after T008)
T006, T008 ─► T014     (AGENT.md must reach the driver via the active pack,
                        meaningless until the driver IS pack content)
T006 ─► T015 ──────────────────────────────────────── [Phase 3 revert]

T012, T013, T014, T015 ─► T016 ─► T017 ─► T018
```

**T001 gates every Phase 2 task** — T007, T008 and T009 alike. **Why, and not just T008:** T007's agent branch is equally a first-boot-after-upgrade behaviour, and once *any* asset moves the pre-relocation state is no longer reproducible from the tree.

**Parallelisable:** T007/T009 (different trees, no shared state); T011/T013 (different files); T014/T015 (unrelated surfaces).

**Critical path** — verified as an actual path through the graph above (an earlier draft's began with `T001 → T002`, which is not an edge: both are independent Phase 0 gates):

`T003 → T004 → T005 → T006 → T009 → T011 → T012 → T016 → T017 → T018`

---

## Mechanism guards

- **T019 — Install seeds the pack's skills (FR-019).** With an assertion that an
  agent's declared-but-missing skill is REPORTED. The filter path cannot raise
  it, so the check belongs at install.
- **T020 — `getAgent` covers delegate-only (FR-020).** Assert through `getAgent`,
  and keep a control asserting an unknown id is refused — otherwise a lookup that
  returns everything also passes.
- **T021 — Dangling-role report (FR-021).** A pack whose `roles` name a
  non-existent agent reports it at install; test with the realistic error: a role
  naming a *skill* id rather than an agent directory.
- **T022 — Seed-reconciliation matrix (FR-022).** Cases: unstamped local copy
  (kept, never archived), stamped and unmodified (updated), stamped and modified
  (kept), dropped from seed (archived). Plus: no reconciliation write ever lands
  inside a pack root.
