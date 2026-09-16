# Design: The spec-kit pack

> **Authoring note.** BOS's `design` step delegates to `architect`, verified by `architect-reviewer`. Neither agent exists in a Claude Code session, so this was authored directly. `045`'s design needed **three** independent review rounds, and in each one roughly a third of the previous round's self-asserted fixes turned out to be wrong — every failure being a claim reasoned about rather than looked up. This document therefore cites source for every mechanical claim. It has **not** yet been reviewed.

Spec: [`spec.md`](./spec.md) · Series: `044` → `045` → **`046`** → `047` → `048`

---

## 1. Classification

**`bos-core`.** Agrees with spec.md's `App Target`.

The classification is worth one sentence of justification because the feature's *output* is pack content: this is a **migration**, and every file it touches is BOS source being removed from BOS source. `target-marketplace-item.md`'s guard-rail does not apply — nothing here binds a port, runs continuously, or handles a raw protocol. There is no service facet, and therefore no worker-thread import question anywhere in this design.

The pack it produces is not a marketplace item either: it ships in-tree (FR-008) because `bos-system-specs` depends on it at first read, so a marketplace fetch on first boot would deadlock.

---

## 2. Constitution check

| Principle | Assessment |
|---|---|
| **I. Spec-Driven (NON-NEGOTIABLE)** | Relocates the engine, not the governance. `bos-system-specs` stays read-only and spec-kit-shaped; the constitution keeps its current path (FR-011). **Compliant.** |
| **II. Server Authority & SSR Boundary** | Pack registration, template mounting and install are server-side. No client reads pack content directly. **Compliant.** |
| **III. Always Delegate; Claude Codes** | `bos-core` source change → `dev_delegate`. **Compliant.** |
| **IV. Minimize Blast Radius (NON-NEGOTIABLE)** | The principle's subject is feature branches, isolated worktrees and COW-isolated candidate data — not diff size (that is spec.md's Edge Case). The IV-relevant question here: pack roots resolve from `process.cwd()` (`subagents/store.ts:33`, `skills/store.ts:19`), so under a preview candidate the pack comes from the **candidate worktree** while `data/` may be a COW clone of base. The pack and the data it seeds into can therefore be from different versions. **Compliant only if** registration resolves the pack relative to the running tree and the seeded copy records that tree's rev — otherwise a promote/discard leaves provenance describing a tree that is gone. A conditional pass on a NON-NEGOTIABLE principle needs an owner, so this is **FR-018**, not a footnote. The mechanisms are `itemPath` via `sourceRootFor` (`bundledAssets.ts:63-64`) for the pack **source**, and `Provenance.version` (`:56-60`) to record the tree's rev. **Not `root?`** — that is the *destination* knob (`destRootFor`, `:62`), documented as "a FEATURE BRANCH's data clone to seed **into**" (`:198-199`), and an earlier draft cited it for a purpose it cannot serve. |
| **V. The VFS Is Not the Source** | `/Methods/spec-kit/templates` exposes engine assets read-only; no mount exposes `src/`. FR-003 *removes* a write grant (`.specify/` from `WRITE_ALLOW_PREFIXES`), tightening rather than loosening. **Compliant.** |
| **VI. Specs & Docs Stay in Sync (NON-NEGOTIABLE)** | FR-016. `docs/dev/build-studio.md`'s Conventions section asserts the exact claim this feature falsifies ("spec-kit's ENGINE is vendored under `.specify/templates` … and stays in BOS source"). **Compliant, conditional on FR-016 landing with the move, not after it.** |
| **VII. Respect Boundaries** | No dependency changes. Quality gates are SC-012. **Compliant.** |

---

## 3. Architecture

### 3.1 Context

The design goal is that **nobody notices**. Every actor's surface is unchanged:

- **A user in Build Studio** — same tree, same phase strip, same artifacts. If anything differs, this feature failed.
- **The Build Studio agent** — reads command prompts at `/Methods/spec-kit/templates/commands/<step>.md` instead of `/Templates/commands/<step>.md`. FR-017 retires the alias here, so this is the release where prompts actually change.
- **A user who edited a seeded agent or skill** — their copy keeps winning, but now shadows a *pack* copy rather than a *seed* copy, and must be told (FR-010).
- **BOS at boot** — registers the pack through the same path a marketplace pack uses (FR-008).

### 3.2 Container

Unchanged from `045`: the Next.js app process owns all of it. No daemon, no port, no Bastion change. The one new physical location is `seed/method-packs/spec-kit/`, which is BOS source shipped by the Dockerfile's `COPY . .` (`:82`) exactly as `seed/agents/` and `seed/skills/` already are.

### 3.3 Component — the pack layout

```
seed/method-packs/spec-kit/
├── method.json          # the descriptor (045 FR-003), builtin: true
├── templates/           # ← .specify/templates  (FR-002)
│   ├── commands/        #   scripts/ references scrubbed (FR-004)
│   └── *-template.md
├── agents/              # ← seed/agents/{architect,architect-reviewer,
│                        #                ui-designer,devil-s-advocate} (FR-005)
└── skills/              # ← seed/skills/build-studio/ (FR-006)
    └── build-studio/    #   **plural** — `sourceRootFor(itemPath,"skill")` resolves
                         #   `<itemPath>/skills` (`bundledAssets.ts:63-64`). A
                         #   `skill/` dir makes `seedOneKind`'s readdir return
                         #   empty and the step a silent no-op.
```

Registration reuses `045` FR-012's sequence — descriptor, agent roots, template mount — invoked from `instrumentation.ts` rather than from a marketplace fetch. **The code path is the same; only the source and the trigger differ.** If the implementation ends up with a spec-kit-only registration function, this feature has failed regardless of what else passes (FR-009, SC-005). `builtin-pack.ts` is a thin source resolver, not a second registration path — SC-005 means "no second registration *sequence*", not "no new file". **The check cannot be a grep for `builtin`** — FR-001 makes it a legitimate descriptor field, so the type, `method.json` and the registry all contain the string. The checkable predicate is: the pack reaches registration through the same call sequence a marketplace pack does, asserted by test rather than by search.

### 3.4 The relocation hazard: a move looks like a deletion

This is the finding that most shapes the feature, and it is not in spec.md.

Both stores share one reconciliation module, `src/lib/agent/seed-sync.ts` (`skills/store.ts:8` and `subagents/store.ts` both import `decideSeedAction`/`archiveSeededDir`/`readSeedStamp`/`seedRev`). Its decision table (`seed-sync.ts:107-123`):

```ts
if (liveRev === undefined) return inSeed ? "seed" : "current";
if (!stamp || stamp.live !== liveRev) return "local";   // edited → never touched
if (!inSeed) return "archive";                          // ← the hazard
return stamp.seed === currentSeedRev ? "current" : "update";
```

`archiveDroppedSeedAgents` (`subagents/store.ts:102`) and `archiveDroppedSeedSkills` (`skills/store.ts:147`) both enumerate `data/` and pass `inSeed: false` for anything the seed no longer ships. So on the first boot after this feature lands, the four relocated agents and the relocated skill are **no longer in `seed/`**, and every deployment's *unmodified* copy is archived to `.archive/`.

**For agents, archiving is the correct outcome and must be allowed.** `045` FR-001a orders roots `data/agents/` → pack roots → `seed/agents/`, **highest wins**. So if the stale `data/agents/architect` copy is *not* archived, it wins forever — and nothing updates it, because the id is no longer in `seed/`, `applySeedAgent` (`subagents/store.ts:53`) never runs for it, and `045` FR-001b forbids the seed writing into a pack root. Every upgraded deployment would be pinned to a pre-relocation agent while a fresh install got the pack's, and FR-010's shadow report would fire for all four agents everywhere, turning a diagnostic into noise. US3 AS3 and SC-008 ("resolves through a pack agent root") would be false except on fresh installs.

Letting the archive fire is safe precisely because of the `local` guard ahead of it (`seed-sync.ts:120`): only a **stamped, provably unedited** copy is archived. An edited one returns `local` and stays — which is the shadow case FR-010 exists to report, and the only case where it should fire.

**The skill needs the same disposition, and a dedicated seeding step.** An earlier draft said the existing archive-then-reconcile order at `skills/store.ts:195`/`:200` was "already correct". **It is not**, and the reason is the memo:

- `reconcileInstalledItemAssets()` is memoized **per data root** (`bundledAssets.ts:244`, `:259-268`) — the first caller runs the pass, everyone else awaits the same promise.
- It is called from **two** independent `ensureSeed()`s: `subagents/store.ts:187` and `skills/store.ts:200`. Nothing orders them, and the **agent store runs first** on the ordinary paths (`listSubAgents()` from `start-run.ts:49`, `api/subagents/route.ts:21`, `api/scheduler/agents/route.ts:9`).

So on first boot after relocation: the agent store archives its four agents and then runs *the* reconcile pass — at which point the pass runs and does nothing for this skill — `reconcileInstalledItemAssets` iterates `listInstalledItems()` and an in-tree pack is not an installed item, so the pack's skill is never a candidate. The skill store then archives the incumbent at `:195`, and its own `:200` call awaits the spent promise. **Net: the driver skill is gone for the life of the process**, recoverable only by restart — a workaround, not a fix.

**Resolution (FR-008b): do not put the pack's skill on the memoized item scan at all.** `reconcileInstalledItemAssets` iterates `listInstalledItems()`, which cannot see an in-tree pack anyway (ADR-3). Give the pack a dedicated seeding step invoked from `skills/store.ts`'s own `ensureSeed()` **after** its archive, reusing `seedOneKind` (`bundledAssets.ts:139-183`) and `recordPendingConflicts` (`:301`) — **not** bare `copyAsset` (`:118-123`), which is an unconditional `rm -rf` + `cp` with no existence or provenance check and would clobber the incumbent this requirement says must surface as a conflict. Ordered by construction, no memo involved, and no change to the item scan.

**The unstamped-incumbent case is not a corner case.** On this machine **zero** of the 16 directories in `data/skills/` carries a `.seed-rev`, while 10 of `data/agents/` do. Without a stamp, `decideSeedAction` returns `local` (`seed-sync.ts:120`), the archive does **not** fire, the incumbent stays, and `seedOneKind` reports `unknown-provenance` forever — the pack's skill never installs and the driver is frozen at its pre-relocation version, with a `SKILL.md:66` pointing at a `/Templates` mount FR-017 has retired. The honest resolution is a `BundledAssetConflicts` prompt: this is the skill-side analogue of FR-010's shadow report, and it needs an owner.

### 3.5 Skills are not agents — FR-006 has an unspecified mechanism

`045` FR-001 gave multi-root discovery to **agents only**. `skillsDir()` (`skills/store.ts:12`) is a single root, and `SEED_DIR` (`:19`) is a single seed path. Nothing anywhere reads skills from a pack root.

So FR-006 as written — "`seed/skills/build-studio/` MUST relocate into the pack" — has no mechanism behind it. Left as-is, the driver skill is archived from `data/skills/` and never reappears, and Build Studio loses the skill that drives the entire pipeline.

The skill therefore needs a mechanism, and `installSkill` — the obvious candidate — **cannot be it**. Read at `client.ts:834-864`:

- **It is defined over a marketplace clone.** `installSkill(marketplaceId, itemId)` calls `readManifest(marketplaceId)` / `findItem` / `cloneDir`. An in-tree pack (FR-008) has no marketplace, no manifest entry and no clone dir — the op cannot be invoked for it at all.
- **It would change the skill's id.** It passes `name: item.name` to `saveSkill`, which derives `id = slugify(input.name)` (`skills/store.ts:428`). The driver skill's id must stay `build-studio`; it is pinned in `seed/agents/build-studio/AGENT.md:6` and `:31` (`skill_load("build-studio")`).
- **It writes no stamp and marks the skill `createdBy: "user"`** (`client.ts:859`). `decideSeedAction` returns `local` for any unstamped copy (`seed-sync.ts:120`), so a pack-installed skill would be **never updated and never archived** — falsifying FR-008a's "upgrades flow through the existing reconciliation".

The mechanism with the right properties is `seedOneKind` (`system/marketplace/install/bundledAssets.ts:139-183`) — the **primitive**, not its driver: it copies `<itemPath>/skills/<id>/`, records `.installed-from.json`, replaces when untouched and reports a conflict when diverged — the stamp contract FR-008a asks for, under a preserved id. Its *driver*, `reconcileInstalledItemAssets` (`:259`), is not usable here — it cannot see an in-tree pack and is a memoized per-root pass the agent store consumes first (§3.4). ADR-3 therefore takes the primitives and leaves the driver alone.

### 3.6 What stays behind

Three things deliberately do **not** move, and each is a decision rather than an omission:

- **The constitution.** It is spec *content*, already at `bos-system-specs/.specify/memory/constitution.md`. FR-011 has the pack declare that path verbatim. `045` FR-006b's `constitutionRoot: "system"` keeps the cross-store read intact.
- **`bos-domain`, the intent skill, the ui-designer craft skill** (all from `044`). FR-007. These are BOS domain/craft knowledge, not method process — the whole point of `044`'s split. Pack content reaches them by skill id, never by path.
- **`.specify/scripts/`** — deleted outright (FR-004), not moved. Nothing executes them: they are reachable only as text through a read-only VFS mount, and BOS reimplemented their function natively (`nextFeatureId` for `create-new-feature.sh`, phase derivation for `check-prerequisites.sh`).

---

## 4. Concrete file/module plan

### New

| Path | Responsibility |
|---|---|
| `seed/method-packs/spec-kit/method.json` | The descriptor, `builtin: true`. Relocated from `045`'s `src/lib/specs/method/spec-kit.ts`. |
| `seed/method-packs/spec-kit/templates/**` | ← `.specify/templates/**` (FR-002), with `scripts/` references scrubbed (FR-004). |
| `seed/method-packs/spec-kit/agents/**` | ← the four process agents (FR-005). |
| `seed/method-packs/spec-kit/skills/build-studio/**` | ← `seed/skills/build-studio/**` (FR-006). **Plural `skills/`** — `sourceRootFor` (`bundledAssets.ts:63-64`) pluralises the kind, so a `skill/` directory would silently seed nothing. |
| `src/lib/specs/method/builtin-pack.ts` | Resolves the in-tree pack root and hands it to `045`'s registration sequence. Deliberately thin — it must not become a second registration path (FR-009). |
| `tests/specs/builtin-pack.test.ts` | Registration before first store read; uninstall refusal; no `builtin` special-case. |
| `tests/specs/relocation-handoff.test.ts` | **The hand-off fixture SC-005a and R6 call mandatory.** Simulates an upgrade from a real pre-relocation `data/`, covering all three branches: stamped-unedited → archive → dedicated step copies; unstamped → no archive → `BundledAssetConflicts`; edited → `local` → conflict. This is the one defect class that can silently destroy the driver skill on first boot, and it had no artifact. |
| `e2e/046-spec-kit-pack.spec.ts` | FR-015, on `045` FR-017a's fixture harness. FR-012/SC-007a require **every** relocated prompt's template path resolve — at least five (`SKILL.md:66` plus four `references/*.md`) — by resolution, not by grep. |

### Modified

| Path | Change |
|---|---|
| `src/lib/specs/method/spec-kit.ts` | **Deleted** — its content becomes `method.json`. This is FR-009's test: if it survives as a fallback, the two-code-paths problem has merely moved. |
| `src/instrumentation.ts` | Registers the in-tree pack via `builtin-pack.ts` (extends `045` T006a rather than adding a second hook). |
| `src/lib/specs/method/registry.ts` | `builtin: true` refuses uninstall with a reason naming `bos-system-specs` (FR-001); upgrade still permitted. |
| `src/lib/dev/repo-fs.ts` | `:28` — drop `.specify/` from `WRITE_ALLOW_PREFIXES`. Leaving it grants write access to a path that no longer exists. |
| `src/lib/specs/spec-mount.ts` | Retire the `/Templates` alias `045` FR-011 kept (FR-017). |
| **`/Templates` references — the full surface (FR-017, R5)** | Enumeration *is* the deliverable, since a miss is a silently broken pipeline step. Post-`044` survivors: **BOS-owned and staying** — `seed/agents/build-studio/AGENT.md:42`,`:44`; `seed/agents/ui-designer/AGENT.md:74`; `references/target-bos-core.md:19` and `target-marketplace-item.md:29` (which `044` FR-005 moves into `bos-domain`, so they become BOS-owned and still say `/Templates`); **`seed/skills/bos-app/SKILL.md:26`** — phase 0, which `044` FR-004 moves into the BOS-owned intent skill, carrying one dead `/Templates` read and two dead `.specify/templates/...` paths. (`044` FR-010 forbids rewording while relocating, so it arrives in this feature's tree uncorrected.) The same 044-forward reasoning was applied to the `target-*.md` files and simply missed here. **Relocating into the pack** — `seed/skills/build-studio/SKILL.md:66` and `references/{plan,tasks,constitution,specify}.md`. **Pack content, also missing from the first enumeration** — `seed/skills/build-studio/SKILL.md:9` (describes `/Templates` as "the spec-kit ENGINE" — false post-FR-017) and `:62` (lists it among the VFS mounts). **Agent-facing prompt text** — `src/lib/agent/tool-groups.ts:56`, the `files` group description naming `/Templates`; the design previously cited this file for `:128` only. **Docs/comments** — `docs/dev/build-studio.md:82`,`:104`,`:110`; `docs/dev/os-shell/virtual-file-system.md:26`; `src/os/vfs.ts:91`,`:103`,`:217`; `src/os/fs-types.ts:5`; `src/os/fs/readonly-fs.ts:8`. |
| `seed/agents/build-studio/AGENT.md` | **The most spec-kit-coupled BOS-owned asset, and FR-013's category.** `:3` "using spec-kit"; `:6` `skills: [build-studio, …]`; `:31` hardcodes the nine-step sequence and `skill_load("build-studio")`; `:42`/`:44` `/Templates`; `:47` `.specify/memory/constitution.md`. The agent stays BOS-owned while the driver skill it loads becomes pack content — so under `047` it names the wrong skill and the wrong phase list. It must reach the driver via the active pack, not by a literal id. |
| `src/components/apps/settings/BuildStudioTab.tsx` | **FR-010's surface.** A pack agent shadowed by a `data/agents/<id>` copy is reported here. Distinct from `045` FR-001a's *collision* report (two packs, same id) though they share a home; this one's trigger is precedence, not ambiguity. |
| `src/lib/agent/subagents/store.ts` | **No change.** `archiveDroppedSeedAgents` (`:102`) is the hand-off, not a hazard (ADR-2) — suppressing it would pin the stale copy forever. An earlier draft's row said the opposite of its own ADR. |
| `src/system/marketplace/install/bundledAssets.ts` | **Must be modified, not merely called into.** `copyAsset` (`:118`), `seedOneKind` (`:139`) and `recordPendingConflicts` (`:301`) are all module-private; the only exported wrapper, `seedItemBundledAssets` (`:194`), seeds **both** kinds (`:204-205`) and would therefore copy the pack's `agents/` into `data/agents/` — resurrecting the four copies ADR-2 requires be archived and violating `045` FR-001b. This feature adds a **skill-kind-scoped exported entry point** over `seedOneKind` + `recordPendingConflicts`, and passes `version` to carry the running tree's rev (FR-018). |
| `src/lib/agent/skills/store.ts` | **The feature's load-bearing code change**, not "no change" as an earlier draft's row claimed. `archiveDroppedSeedSkills` (`:147`) is itself unmodified, but `ensureSeed()` gains a **dedicated pack-skill seeding step after the archive at `:195`** (§3.4) — ordered by construction, off the memoized item scan. |
| `src/lib/marketplace/schema.ts` | The pack's `method` facet coexists with its `skill` facet — both already exist after `045`; this confirms the combination is legal. |
| `src/lib/self-heal/brief.ts` | `:39-40`, `:58-59`, `:172` hardcode the step sequence and artifact names in the autonomous spine's brief (FR-013). Either descriptor-derived or explicitly declared spec-kit-only. |
| `src/lib/agent/config.ts` | `:15` CORE_POLICY names "the spec-kit pipeline" (FR-013). |
| `src/lib/agent/tool-groups.ts` | `:128` `aliases: ["specification", "requirements", "spec-kit", "feature spec"]` (FR-013). |
| `src/app/api/specs/run-tests/route.ts` | `:65-70` derives `featureId` from the unit path's last segment and looks for `e2e/<featureId>.spec.ts` **inside BOS's own repo** (`:70-72`); `:127` writes `test-results.md`. Both become descriptor-declared, and the tool refuses with a reason when no test location is declared (FR-014). **Two axes, not one:** the *name* comes from `045`'s `testFile?`; the *location* is a *store-kind* question `045` does not answer — an item-owned store's tests do not live in BOS's repo at all, and `runPlaywright(specFileRel)` (`:79`) executes inside `process.cwd()` regardless. The refusal predicate is "this store kind declares no test root". **The field is `testRoot`, and it belongs on `SpecStore` (`src/lib/specs/stores.ts`), not on the descriptor** — the location is a property of where the store's content lives, not of which method governs it. Values: `"repo"` for `system`/`user`/`marketplace` stores (tests live in BOS's `e2e/`, today's behaviour); `undefined` for `owner: "item"` stores, whose content lives in `data/user-apps/items/<id>/` and whose tests are not BOS's to run. `undefined` ⇒ the tool refuses with that reason instead of probing `process.cwd()`. |
| `seed/skills/build-studio/**`, `seed/agents/{architect,architect-reviewer,ui-designer,devil-s-advocate}/**` | Removed from `seed/` (they become pack content). |
| `.specify/` | Deleted entirely: `templates/` moves, `scripts/` is dropped. 25 files today. |
| **`.specify/` paths *inside* the relocated prompts (FR-004, widened)** | A larger surface than `scripts/`, and the scrub **must discriminate** rather than blanket-replace. **Rewrite to the pack mount:** `.specify/templates/{plan,spec,tasks,constitution}-template.md` (four, not three — `commands/constitution.md:56` references the constitution template) and `commands/*.md` (`commands/constitution.md:81-84`, `plan-template.md:7`). **Delete as unreachable:** **exactly 35** `.specify/extensions.yml` probes across **ten** command files — `plan`(3), `specify`(3), `implement`(3), `converge`(4), `taskstoissues`(4), `analyze`(4), `clarify`(3), `checklist`(4), `constitution`(4), `tasks`(3). An earlier draft named six of the ten, omitting 14 of the 35 occurrences, plus `.specify/init-options.json` and the `.specify/feature.json` write (`commands/specify.md:87`,`:99`). **Preserve verbatim:** the 5 `.specify/memory/constitution.md` references in `commands/constitution.md` — that path stays valid under ADR-4. This is ADR-5's defect class at larger scale, arriving from the same vendored prompts. |
| `docs/dev/method-packs.md` (new) | Required by ADR-3 and ADR-5's consequences — records the agents-discovered / skill-copied asymmetry as a known shape, with multi-root skills noted as the eventual convergence. Was required by two ADRs and absent from this table. |
| `CLAUDE.md` (repo root) | FR-016, and a governance obligation: Principle VI makes CLAUDE.md's accuracy binding (constitution `:76-77` requires CLAUDE.md point here; VI is what makes its claims binding), and it asserts "The spec-kit **engine** (templates + command prompts) stays in source at `.specify/templates`" — falsified by this feature. |
| `docs/dev/build-studio.md`, `docs/dev/repository-and-data-layout.md` (`:7` makes the same claim and must be **corrected**, not extended), `docs/dev/assistant/sub-agents-and-delegation.md:37`, `docs/usage/apps/build-studio.md:62` ("only ever edits files under `specs/` and `.specify/`" — falsified) | FR-016. `AGENTS.md:5` carries the identical falsified sentence but is untracked, so it is out of scope by that fact rather than by choice. |

---

## 5. Integration points (called into, not modified)

- **`045` FR-012's registration sequence** — this feature supplies a different source and trigger, not a different path.
- **`seed-sync.ts`'s stamp contract** (`:107-123`) — the `local` branch is what protects a user's edited agent or `skill_improve`-rewritten skill. This design changes *when* `inSeed` is false, never the table itself. **FR-008a** rests on this for *agents*. For the skill it does **not** hold through `installSkill`, which writes no stamp at all (ADR-3) — FR-008a is satisfied there only via the bundled-asset contract's `.installed-from.json`. An earlier draft claimed the stamp contract governed both; it does not.
- **`045` FR-012 says "nothing MUST be copied (035)"** — the bundled-asset path is the codebase's existing, deliberate exception to that rule (`bundledAssets.ts:14-22` explains why a mutable asset cannot be served from a read-only item root). `046` inherits the exception rather than breaking the rule.
- **The `.installed-from.json` contract** (`bundledAssets.ts:54`, `:170-181`) — the semantics ADR-3 reuses. Note `seedOneKind` (`:139`) and `recordPendingConflicts` (`:301`) are **module-private**, so reusing them is a *modification* of that file (see §4), not an integration point. **`reconcileInstalledItemAssets` (`:259`) is deliberately NOT an integration point**: memoized per data root, consumed by the agent store first (§3.4), and blind to an in-tree pack. `installSkill` (`client.ts:834`) is not one either — it cannot be invoked for an in-tree pack at all.
- **`045` FR-001a's precedence** — `data/agents/` → pack roots → `seed/agents/`. The relocation relies on it and adds nothing.
- **Dockerfile `COPY . .`** (`:82`) — `seed/method-packs/` ships with the image exactly as `seed/agents/` does. No Dockerfile change.

---

## 6. ADRs

### ADR-1 — In-tree, but registered through the marketplace install path

**Context.** `bos-system-specs` needs the pack at first store read, so it cannot be fetched.

**Options.** (a) A bespoke boot registration for the built-in pack. (b) Ship in-tree, register through `045` FR-012's ordinary sequence.

**Decision.** (b).

**Consequences.** `045`'s `method/install.ts` also owns FR-013a's origin gate and per-pack opt-in; an in-tree pack has no marketplace origin, and `045` FR-013a's gate applies only to packs carrying a `plugin/` facet, which this pack does not have — so the gate never triggers here rather than being skipped — stated here rather than left as an implicit branch. One registration path, exercised by BOS's own method — which is this feature's entire justification. (a) would recreate the two-code-paths problem it exists to remove and leave the install path untested until `047`. Cost: `builtin-pack.ts` must stay thin, and FR-009/SC-005 exist to enforce that — **by following the call sequence in a test, never by grepping for `builtin`**, which FR-001 makes a legitimate descriptor field (§3.3). An earlier draft of this ADR said "by grep", contradicting its own body and the criterion it cites.

### ADR-2 — Let the archive fire for both; it is the hand-off, not the hazard

**Context.** §3.4. Moving an id out of `seed/` is indistinguishable, to `decideSeedAction`, from deleting it.

**Options.** (a) Let the archive fire for both; agents are then supplied by the pack root, and the skill by a **dedicated post-archive seeding step**. (b) Teach both archive paths that a pack-supplied id is not "dropped". (c) Keep stub entries in `seed/`. (d) Let the archive fire for agents, copy-before-archive for the skill.

**Decision.** (a). **(b) and (d) were both earlier drafts and are both wrong.**

**Consequences.** For **agents**, archiving *is* the hand-off: the stale `data/` copy moves to `.archive/` and the pack root supplies the agent, which is what US3 AS3 and SC-008 assert. (b) would let the stale copy win forever under `045` FR-001a's precedence, with `applySeedAgent` (`subagents/store.ts:53`) unable to update an id no longer in `seed/`.

For the **skill**, (d)'s copy-first ordering is backwards against `seedOneKind` (`bundledAssets.ts:156`), which copies only into an absent destination — so copying first copies nothing and the subsequent archive destroys the skill. But the existing order is **not** sufficient either: `:200`'s reconcile is a memoized pass the agent store consumes first (§3.4), so the skill needs its own step after `:195`'s archive.

The `local` guard (`seed-sync.ts:120`) protects edited copies in both cases. (c) is a workaround that exists only to defeat a cleanup mechanism.

**Residual, and it is the feature's sharpest edge:** an *unstamped* incumbent returns `local`, so nothing archives and the reconcile reports `unknown-provenance`. That is the state of every `data/skills/` directory on this machine. It must surface as a conflict the user can act on, not as silence.

### ADR-3 — The skill is seeded through the bundled-asset contract, not `installSkill`

**Context.** §3.5 — skills have a single root, so the relocated driver skill needs a copy mechanism with a stamp.

**Options.** (a) `installSkill` (`client.ts:834`). (b) Extend multi-root discovery to skills. (c) Seed it with the bundled-asset **primitives** — `seedOneKind` (`:139-183`), `copyAsset` (`:118-123`), the `.installed-from.json` contract (`:54`) and `recordPendingConflicts` (`:212`) — from a dedicated step, **explicitly excluding** `reconcileInstalledItemAssets` (`:259`), which is the memoized item scan.

**Decision.** (c).

**Consequences.** (a) is **impossible**, not merely unattractive: it is defined over a marketplace clone an in-tree pack does not have, it would rename the skill to `slugify(item.name)` when the id must stay `build-studio`, and it writes no stamp and `createdBy: "user"`, so the copy would be `local` forever — never updated, contradicting FR-008a. (b) is the consistent-with-agents answer and remains the better long-term shape, but it widens `045` (already 32 requirements) and, decisively, **skills are mutated in place** — `saveSkill`→`writeSkill` (`skills/store.ts:312-321`) is reached by `patchSkill` (`:453`), `improve.ts:19,40` and the Curator — so a read-only pack root would break `skill_improve` in a way it does not break agents.

(c) reuses the primitives whose semantics match FR-008a (`.installed-from.json`, replace-when-untouched, conflict-when-diverged) under a preserved id, while **not** joining the memoized item scan — `reconcileInstalledItemAssets` (`:259`) is driven by `listInstalledItems()`, cannot see an in-tree pack, and is consumed by the agent store first (§3.4). **It is still new machinery**; saying "no new machinery", as an earlier draft did, was false whichever option was chosen. Note `copyAsset` alone is an unconditional `rm -rf` + `cp` (`:118-123`), so the step must go through `seedOneKind`'s exists→provenance→hash path or it will clobber the very incumbent FR-008b says must surface as a conflict.

Cost: agents are discovered in place while the skill is copied. That asymmetry is real and should be recorded in `docs/dev/method-packs.md` as a known shape, with (b) noted as the eventual convergence.

### ADR-4 — The constitution's path stays a fossil

**Context.** Stores hold the constitution at `.specify/memory/constitution.md` — a path named after a framework whose engine has just left the tree.

**Options.** (a) Rename to something method-neutral and migrate stores. (b) Declare the existing path verbatim.

**Decision.** (b).

**Consequences.** Nothing moves for existing stores, and `045` FR-009a already owns what happens when a store's method *changes*. Renaming would be a store-content migration inside a feature whose defining property is that it changes no behaviour. Cost: a path that reads oddly to anyone who doesn't know the history, mitigated by saying so in FR-011 and in the docs.

### ADR-5 — `.specify/scripts/` is deleted, not relocated

**Context.** Eight command prompts reference it, and BOS never executes it.

**Options.** (a) Move with the templates. (b) Delete, and scrub the references.

**Decision.** (b).

**Consequences.** Removes a live prompt defect: agents are currently told to run `check-prerequisites.sh`, which they cannot — `/Templates` is a VFS mount and `run_command`'s shell does not share that filesystem, which is the exact trap `build-studio/SKILL.md:62` warns about, arriving from BOS's own vendored prompts. (a) would preserve unreachable instructions in a new location. Cost: the scrubbed prompts diverge from upstream spec-kit, so a future upstream sync is a merge rather than a copy — acceptable, since they had already diverged by being mounted read-only.

---

## 7. Risks / open questions

**R1 — The skill's seeding must not ride the memoized item scan.** `reconcileInstalledItemAssets()` is one pass per data root (`bundledAssets.ts:244`) shared by both stores, and the agent store consumes it first — so a skill archived afterwards can never be re-seeded within that process. The pack's skill gets its own step, ordered after the archive inside `skills/store.ts`'s `ensureSeed()`. An earlier draft scoped this risk to "pack registration vs. the reconcile" and missed the memo entirely. `listSubAgents()` (`subagents/store.ts:321`) calls `ensureSeed()` itself, and `skills/store.ts` does the same, so the first call from any entry point triggers it. Registration must happen at `instrumentation.ts` time and be asserted, not assumed from import side-effects — the same instruction `045` T006a gives, now load-bearing for data safety rather than just correctness.

**R2 — This is a large diff with no functional change.** Exactly the shape a reviewer skims and a bug hides in. Keep each relocation a pure `git mv` in its own commit; put the FR-004 scrub and the FR-013 de-coupling in separate commits so they are readable as edits rather than lost in a move.

**R3 — `seed-sync.ts` is not valid UTF-8.** `file` reports it as `data`, and `grep` silently skips it — I hit this while verifying §3.4 and had to read it through Python. Anyone auditing this feature with grep will get false negatives on the module it most depends on. Worth fixing opportunistically; worth *knowing* regardless.

**R4 — FR-014 changes behaviour, unlike everything else here.** `buildstudio_run_tests` currently looks in BOS's own `e2e/` for any store, which is already wrong for item-owned stores. Making it refuse rather than silently look in the wrong repo is a correctness fix, but it is the one place this feature is not behaviour-preserving, and the `045` parity harness will not catch it (it tests `Specification` objects, not this route).

**R5 — FR-017 is the first release where prompts actually change.** The `/Templates` alias retires here, so every prompt referencing `/Templates/commands/<step>.md` must be updated in the same change — including the relocated `SKILL.md:66`. A missed reference is a silently broken pipeline step, not a compile error.

**R6 — Both dispositions depend on deployment state that differs per install.** Agents relocate cleanly *only if* the incumbent is stamped — 10 of this machine's `data/agents/` are. The skill's incumbent is **not** stamped anywhere here, so its archive will not fire and the reconcile will report a conflict instead. Neither outcome is observable in development without simulating an upgrade from a real pre-relocation `data/`, which makes the fixture a requirement rather than a nicety.

**Open question (for `plan`):** whether `046` should also extend multi-root discovery to skills (ADR-3 option (a)) rather than deferring it. It would remove the agents-discovered / skills-copied inconsistency, but widens a feature whose value is that it changes nothing. My recommendation is to defer, and to record the inconsistency in `docs/dev/method-packs.md` so it is a known shape rather than an accident.

---

## 8. Two seams that behave differently than they look

### 8.1 The pack contains two kinds of content with two different lifecycles

A pack folder holds agents and skills side by side, as directories of markdown.
They are handled in opposite ways, and nothing in the layout says so:

| | Agents | Skills |
|---|---|---|
| Mechanism | **discovered in place** via a registered root | **copied** into `data/skills/` by a seeding step |
| Live copy | the pack's own file | BOS's copy |
| If the step is missing | agent still appears | skill is simply absent |

The consequence is asymmetric too. A pack installed without its skills being
seeded still shows its agents — which then declare skills that do not exist, and
the allowlist filter drops them **silently**, because a filter cannot tell
"not permitted" from "not present". The pack looks installed and is quietly
degraded.

**Rule: when two similar-looking things have different lifecycles, the failure
mode of the one with more steps must be loud.** A declared-but-absent skill is a
report, not a filter result.

### 8.2 Visibility is about discovery, not reachability

`delegate-only` means "not in the picker", not "not callable". Implementing
`getAgent` on top of the picker-filtered listing collapsed the two, and every
delegate-only agent became undeliverable.

Both listing functions were individually correct, and the test asserted them
directly — so it passed while the feature was broken. The test was checking the
helpers rather than the path `agent_delegate` takes.

**Rule: test the entry point the product calls.** A test that reaches past the
product's own call path validates a function, not a feature. Same failure as
`045` §9.3, from the other direction.

### 8.3 Two reconciliation contracts that must not be conflated

`.seed-rev` (a two-hash stamp: what we seeded, what it looked like) answers "has
the user edited this?". `.installed-from.json` answers "where did this come
from?". They look interchangeable and are not.

An UNSTAMPED copy is the dangerous case: it means a user put it there by hand.
The `local` guard therefore has to be evaluated before the `archive` path, or
reconciliation files the user's own work into `.archive/` on the next boot.

And reconciliation must resolve its write root explicitly, never through the
list of discovery roots — writing through a discovery root reaches INSIDE an
installed pack, reverting the pack's content and losing the change entirely on
the item's next upgrade.
