# Design: The method-pack layer

> **Authoring note.** BOS's `design` step delegates to the `architect` agent, verified once by `architect-reviewer`. This document was written directly in a Claude Code session where neither agent exists. It **has** since been through one independent review pass (a separate agent context, read-only, verifying claims against source) and this is the post-revision version — see §8 for what that review found and how each finding was resolved. Revision quota per the method is one round; no second review was run.

Spec: [`spec.md`](./spec.md) · Series: `044` → **`045`** → `046` → `047` → `048`

---

## 1. Classification

**`bos-core`.** Agrees with spec.md's `App Target` field — no reconciliation needed (design.md step 4).

This is not a close call, and the reasoning matters because the *series* spans two shapes:

| Feature | Shape | Why |
|---|---|---|
| `045` (this) | `bos-core` | Modifies `src/lib/specs/`, agent discovery, VFS mounts, the Build Studio app, `/api/specs`, marketplace install/UI, config. No self-contained installable artefact. |
| `046` | `bos-core` | A source-to-pack *migration* — removes assets from `src/`, `.specify/`, `seed/`. |
| `047`, `048` | `marketplace-item` | Self-contained items in `data/user-apps/items/<id>/` with a `method/` facet (Q1). |

The trap this classification avoids: "method packs are marketplace items, therefore this is a `marketplace-item` feature." The *packs* are items; the *layer that hosts them* is BOS itself. Building `045` as an item would require the item to modify `pipeline.ts`, which is impossible by construction.

Per `target-marketplace-item.md`'s guard-rail — before concluding `bos-core`, check whether a service facet sidesteps the need — the answer here is no: this feature's work is inside the Next.js request path (spec discovery, tree building, phase derivation). Nothing binds a port, runs continuously, or handles a raw protocol. **There is no service facet in this design**, and therefore no worker-thread/`src/lib` import question anywhere in it.

---

## 2. Constitution check

All seven principles, not a subset.

| Principle | Assessment |
|---|---|
| **I. Spec-Driven (NON-NEGOTIABLE)** | This feature *is* the spec pipeline. The risk is self-referential: a regression degrades BOS's ability to author specs, including its own. Mitigated by SC-001's field-level parity gate. **Compliant.** |
| **II. Server Authority & SSR Boundary** | Descriptor loading, rule evaluation and store discovery are server-only (`import "server-only"` in `src/lib/specs/method/*`). The client receives the evaluated result plus descriptor via `GET /api/specs` (FR-014) and never reads a descriptor from disk. `method/types.ts` stays framework-free so the Build Studio client can import the shape. **Compliant.** |
| **III. Always Delegate; Claude Codes** | Implementation is a `bos-core` source change → `dev_delegate` to the Developer at `implement`. **Compliant** (see the Authoring note for where this design itself deviated). |
| **IV. Minimize Blast Radius (NON-NEGOTIABLE)** | Work on `bos/spec-method-packs`; A/B/C staging with independent revert points (SC-009). This principle is what drove staging: eight subsystems in one unrevertable commit is exactly the blast radius it prohibits. **Compliant.** |
| **V. The VFS Is Not the Source** | This feature adds VFS mounts (`/Methods/<packId>/templates`) and places packs in per-user `data/`. Both are correct under this principle: mounts expose *engine assets*, not BOS source, and the pack content is installed data, not a source edit. No mount exposes `src/`. **Compliant.** |
| **VI. Specs & Docs Stay in Sync (NON-NEGOTIABLE)** | Directly load-bearing here, in two ways. It owns FR-018 — `docs/dev/build-studio.md` and `architecture-overview.md` §9 both describe the hardcoded model and would be wrong after Stage A. It also governs `discrepancies.md`, the drift record FR-006a is changing from a hardcoded cross-store read to a descriptor-declared path; that relocation must not lose the principle's recording obligation. **Compliant, conditional on FR-018 landing with the stages rather than after them.** |
| **VII. Respect Boundaries** | No dependency changes, no `package.json`/lockfile edits. Quality gates (`npx tsc --noEmit`, `npm run lint`, `npm run test:unit`) are SC-017. **Compliant.** |

One tension worth naming rather than designing around: Principle I names the pipeline `constitution → specify → clarify → plan → tasks → implement`. This feature makes that pipeline configurable per store. It does not weaken the principle — `bos-system-specs` stays permanently spec-kit-shaped and read-only, so BOS's own governance keeps the constitution's own pipeline. A user store adopting OpenSpec changes which artefacts BOS looks for *there*, not whether specs precede code.

---

## 3. Architecture

### 3.1 Context

- **A user in Build Studio** — a Method dropdown on a store group header, a phase strip whose labels come from the server, section-kind badges, a preflight dialog on method change.
- **The assistant / Build Studio agent** — unchanged surface; reads `/Methods/<pack>/templates` instead of `/Templates`, with the alias surviving one release (FR-011, ADR-6).
- **A pack author** (`047`, `048`) — writes `method.json`; refused at registration if `schemaVersion` is unsupported (FR-003a).
- **A user in the Marketplace** — can see, install and uninstall a method pack (FR-012a), which today is impossible: `apps/marketplace/index.tsx:383` renders no Install button for an item with no app/skill/service/plugin facet.
- **BOS at boot** — registers the built-in spec-kit descriptor before the first spec-store read.

### 3.2 Container

| Container | Role |
|---|---|
| **Next.js app process** | Owns everything: registry, evaluation, store discovery, agent discovery, mounts, `/api/specs`, install wiring. |
| **Supervisor + preview worktrees** | Unchanged. Branch-coupled stores (`020`) keep working; descriptor resolution happens *after* root resolution. |
| **Spec stores (git repos)** | Gain optional `method` in `spec-store.json`, optional per-Project override in `project.json`. Store content, version-controlled, travels with the repo — this is what makes Q5's multi-user answer fall out. |
| **`data/` (per-user under Bastion)** | Holds installed packs. Per-user by construction (FR-008a). |
| **Worker-thread services** | **Not involved.** No daemon, no port. |
| **Bastion** | Not modified. Per-user isolation inherited, not built. |

### 3.3 Component

```
                        ┌─────────────────────────┐
  spec-store.json ─────▶│  method/resolve.ts      │  store → descriptor
  project.json          │  (binding + default)    │  (FR-008, FR-008a)
                        └───────────┬─────────────┘
                                    │
                        ┌───────────▼─────────────┐
  method.json ─────────▶│  method/registry.ts     │  schemaVersion gate (FR-003a)
  (pack or built-in)    │  register / unregister  │  refuse, never default
                        └───────────┬─────────────┘
                                    │ MethodDescriptor
                        ┌───────────▼─────────────┐
                        │  method/evaluate.ts     │  the rule DSL (FR-004/004a)
                        │  predicate → PhaseState │  pure; file-reader injected
                        └───────────┬─────────────┘
                                    │
  ┌─────────────────────────────────▼──────────────────────────────┐
  │  pipeline.ts  — descriptor-driven                               │
  │  · one leaf predicate (was 4 copies)                            │
  │  · derivePhases → evaluate()                                    │
  │  · walk per section, not one tree                               │
  │  · nextFeatureId honours numbering: "none"                      │
  └─────────────────────────────────┬──────────────────────────────┘
                                    │ Specification[] + descriptor
                        ┌───────────▼─────────────┐
                        │  /api/specs (FR-014)    │  GET descriptor; PUT method
                        └───────────┬─────────────┘
                                    │
                        ┌───────────▼─────────────┐
                        │  Build Studio           │  PhaseStrip renders what
                        │  (no phase vocabulary)  │  it is given (FR-007)
                        └─────────────────────────┘
```

**Descriptor shape** (`method/types.ts`) — the full field list, resolving a spec-level gap (see §8, S5):

`schemaVersion` · `id` · `label` · `version` · `builtin?` · `sections[]` (`rel`, `kind`, `leafMarker`, `numbering`, **`terminal?`**) · `constitution` · **`constitutionRoot`** · **`discrepancies`** · **`testFile?`** · **`artifactOrder[]`** · `artifacts[]` (`{id, generates, requires, `**`scope`**`}`, `generates` a glob) · `phases[]` (`{id, label, requires, `**`rules[]`**`, `**`else`**`}`) · **`stateLabels`** · `templates` · `agents[]` · `roles{}` · `modules[]?`

Five of these were added by the second review and are not in FR-003's list; `analyze` reconciles the spec:

- **`discrepancies`** — required by FR-006a's prose, omitted from FR-003.
- **`stateLabels`** — `{done, pending, blocked, na}` display map. See §3.4; without it `047` FR-005 is unsatisfiable.
- **`sections[].terminal`** — an `archive` section renders a terminal state rather than evaluating the DAG (`047` FR-006b).
- **`artifacts[].scope`** — `"store"` or `"unit"`. `047` FR-006a requires globs be explicitly typed as change-relative; `specs` means two different things at two roots in OpenSpec.
- **`testFile`** — the test-file naming convention, which `046` FR-014 requires become descriptor-declared.
- **`constitutionRoot`** — `"system"` or `"store"` (§3.6). Without it the 8 live item stores flip `done → pending`.
- **`artifactOrder[]`** — the **display** sort order, deliberately distinct from `artifacts[]` (the DAG). `ARTIFACT_FILES` (`types.ts:87-94`) lists six names and excludes `design.md`/`test-results.md`, which fall to `byArtifactOrder`'s `99 → localeCompare` tail (`pipeline.ts:187-193`). 23 live features carry `design.md`; any `artifacts[]`-derived ordering re-ranks it and changes `Specification.artifacts[]`, which SC-001 requires be identical. Spec-kit's `artifactOrder[]` reproduces `ARTIFACT_FILES` verbatim, tail behaviour included.

`discrepancies` is required by FR-006a's prose but absent from FR-003's field list; it belongs in the type, and `analyze` should reconcile the spec.

Separately, and deliberately **not** coupled to the above:

```
  data/agents/  ─┐
  pack roots    ─┼──▶  subagents/roots.ts  ──▶  listSubAgents()  (FR-001, FR-001a/b)
  seed/agents/  ─┘     (ordered merge)          store.ts:321
```

**Why agent discovery is a separate component.** A descriptor *names* agents (`agents[]`, `roles{}`) but does not resolve them. Resolution is a property of the install, not the method — the same `architect` serves spec-kit and OpenSpec. Coupling them would let a store's method change alter which agents exist, which is wrong: agents are per-user, stores are shared.

### 3.4 Phase states — parity by construction

`PhaseState` becomes `done | pending | blocked | na`. **`pending` and `na` both keep their current meanings**; only `blocked` is new.

A naive rule — *done if the predicate holds, else `pending` if reachable, else `blocked`* — is wrong, and was the second review's sharpest finding. Three of spec-kit's nine phases are `na` today **while fully reachable** (`pipeline.ts:108-113`, `:123`):

| Phase | Today | Under the naive rule |
|---|---|---|
| `implement` with `tasks.md` present, 0 of 10 done | `na` | `pending` ✗ |
| `test` with no `test-results.md` | `na` | `pending` ✗ |
| `converge` with no discrepancy entry (the normal case) | `na` — never `pending` | `pending` ✗ |

That is `na → pending` across every feature in both live stores — a violation of SC-001 in the direction nobody was guarding, and of tasks.md's own rule 2. It would have surfaced at T011, *after* T008's refactor.

**The rule model instead is an ordered clause list**, which reproduces today's branching exactly:

```jsonc
{ "id": "clarify", "label": "Clarify", "requires": ["specify"],
  "rules": [
    { "when": { "contains": ["spec.md", "## Clarifications"] }, "then": "done" },
    { "when": { "exists": "spec.md" },                          "then": "pending" }
  ],
  "else": "na" }
```

Resolution: evaluate `rules` in order, first match wins. If none matches, the state is `blocked` when any `requires` edge is unsatisfied, otherwise the phase's `else` (default `pending`).

**Spec-kit declares `requires: []` for all nine phases.** Its gating is expressed entirely in the clause list, never in edges — `clarify` is `pending` because `spec.md` exists, not because a `specify` edge is satisfied. A third review built the truth table across all 132 live feature leaves and showed why this matters: under the linear edges `PHASE_ORDER` suggests, `converge requires implement` would flip **120** features `na → blocked` and `test requires implement` **114**, because `implement` is `done` on only 12 of 132. An earlier draft of this section asserted those three phases were "preserved" — true only with no edges, which it never said.

With `requires: []` throughout, spec-kit's output is **byte-identical**, not merely parity-compliant:

| Phase | `requires` | Rules | Result |
|---|---|---|---|
| `constitution` | `[]` | store-global read (§3.6) → `done` : `pending` | unchanged |
| `specify` | `[]` | `exists spec.md` → `done`; else `pending` | unchanged |
| `clarify` | `[]` | `contains` → `done`; `exists spec.md` → `pending`; `else: "na"` | unchanged |
| `plan` | `[]` | `exists plan.md` → `done`; `exists spec.md` → `pending`; `else: "na"` | unchanged |
| `tasks` | `[]` | `exists tasks.md` → `done`; `exists plan.md` → `pending`; `else: "na"` | unchanged (46 stay `na`) |
| `analyze` | `[]` | `else: "na"` | unchanged |
| `implement` | `[]` | `checklist` all → `done`; some → `pending`; `else: "na"` | unchanged (107 stay `na`) |
| `converge` | `[]` | `containsUnitId` → `done`; `else: "na"` | unchanged (all 132 `na`) |
| `test` | `[]` | non-empty ∧ `contains PASSED` → `done`; non-empty → `pending`; `else: "na"` | unchanged |

`blocked` is therefore **unreachable under spec-kit** and exists for packs that declare real edges — `047`'s `tasks requires [specs, design]` is the first consumer. That strengthens SC-001: the criterion is **zero state changes**, with no substitutions to enumerate at all.

**Edge satisfaction** is defined as: a `requires` target is satisfied when its own resolved state is `done`. A phase that can never be `done` (spec-kit's `analyze`, whose only clause is `else: "na"`) must therefore never be named as an edge target — stated here because naming it would silently block everything downstream.

Two predicate subtleties the truth table exposed, both of which change `evaluate.ts`:

- **`exists` means directory-listing presence, but `test` keys on content.** `pipeline.ts:111-113` reads the file and tests truthiness, so an empty `test-results.md` is `na` today while `{exists}` would yield `pending`. The DSL needs both `exists` and `nonEmpty`.
- **`checklist` must guard `total > 0`.** `pipeline.ts:108-109` gives `na` when `tasks.length === 0`. A vacuously-true "all done" predicate would yield `done` — an unsanctioned `na → done`. This is not hypothetical: 9 of 132 features have a `tasks.md` with zero parseable `- [ ]` items (they use `### T001` headings), **including this feature's own `tasks.md`**.

### 3.5 State labels — how OpenSpec renders "available"

The first revision claimed OpenSpec's presentation rides the phase's `label`. **That was wrong**: `label` is the *phase's* display name and replaces `PHASE_LABEL` (`index.tsx:78-88`, phase id → `"Const"`, `"Spec"`…). It cannot carry state vocabulary, so `047` FR-005, SC-003 and US3's Independent Test were all unsatisfiable as written.

The descriptor therefore carries a separate `stateLabels` map:

```jsonc
"stateLabels": { "done": "Done", "pending": "Pending",  "blocked": "Blocked", "na": "N/A" }   // spec-kit
"stateLabels": { "done": "Done", "pending": "Available", "blocked": "Blocked", "na": "N/A" }  // openspec
```

`PhaseStrip` renders the phase `label` as chip text and `stateLabels[state]` wherever the state is shown (tooltip, legend). The state *id* stays `pending`, so parity holds; the *word the user sees* is the pack's. `047` gets "enablers, not gates" without widening this feature's acceptance gate.

### 3.6 The constitution is store-global, like `discrepancies` was

`hasConstitution()` (`pipeline.ts:72-79`) reads through `systemStoreId()` (`:74`) — the *system* store, for every feature in every store. FR-006a already caught the identical pattern for `discrepancies.md` (`:94-104`); the constitution read was missed.

The descriptor therefore carries `constitutionRoot: "system" | "store"` alongside `constitution`. Spec-kit declares `"system"`, preserving today's behaviour exactly. Without this, resolving the path per-store gives `done → pending` on the **8 live item-owned stores** (`data/user-apps/items/*/spec/`), none of which has a `.specify/memory/constitution.md` — and `done → pending` is not a substitution SC-001 sanctions at all.

`constitutionReady` (`:71`) is a single module-global memo shared by every store. It must become per-(descriptor, root) or it returns one store's answer for all of them.

## 4. Concrete file/module plan (only what this feature creates or modifies)

### New — source

| Path | Responsibility |
|---|---|
| `src/lib/specs/method/types.ts` | Framework-free descriptor + rule types (§3.3). Client-importable (Principle II). |
| `src/lib/specs/method/registry.ts` | `registerMethod` / `unregisterMethod` / `getMethod` / `listMethods`. Enforces FR-003a's `schemaVersion` gate at registration. |
| `src/lib/specs/method/evaluate.ts` | The rule DSL: `exists`, `contains`, `notContains`, `checklist`, `containsUnitId`, `dependsOn`, `manual`, plus FR-004a's set predicate `{glob, quantifier, of}` + `count`. Pure over an injected file-reader — unit-testable without a store. |
| `src/lib/specs/method/spec-kit.ts` | The built-in descriptor, in the same format a pack uses. Relocated to the pack by `046`. |
| `src/lib/specs/method/resolve.ts` | Store → effective descriptor: `spec-store.json.method` → `project.json.method` → global default. |
| `src/lib/specs/method/preflight.ts` | FR-010a/010c's double-discovery diff — one implementation shared by assignment **and** upgrade. |
| `src/lib/specs/method/constitution.ts` | FR-009a's relocate-or-report when a store's constitution path changes with its method. Separate from `preflight.ts` because FR-010c makes preflight the *gate* and this the *remedy*. |
| `src/lib/specs/method/install.ts` | FR-012's register-on-install / unregister-on-uninstall of descriptor, agent roots and template mount; FR-013a's origin gate and per-pack opt-in record. |
| `src/lib/agent/subagents/roots.ts` | Ordered agent-root list + precedence (FR-001a). `kind: "dir"` only; `048` adds `"provider"`. |
| `src/apps/build-studio/MethodPicker.tsx` | Method dropdown + preflight dialog (FR-009, FR-010). |

### New — tests

The two harnesses this design calls prerequisites need files, not just risk entries.

| Path | Responsibility |
|---|---|
| `tests/specs/method-parity.test.ts` | **SC-001's parity harness.** Snapshots every `Specification` from both live stores incl. a live `bos/*` branch; compares field-level per SC-001. Stage A's first task (R2). |
| `tests/specs/method-evaluate.test.ts` | The rule DSL incl. FR-004a's set predicate against a glob of unknown cardinality (SC-007). |
| `tests/specs/method-registry.test.ts` | `schemaVersion` refusal (SC-015); descriptor round-trip. |
| `tests/specs/method-preflight.test.ts` | Assignment and upgrade share one implementation (SC-010); orphan-count diff incl. draft-branch-only features. |
| `tests/agent/agent-roots.test.ts` | Precedence order, `delegate-only` invisibility with delegability retained, same-id collision reporting (SC-006, SC-008). |
| `tests/specs/vfs-unmount.test.ts` | `unregisterMount`. **`tests/specs/`, not `tests/os/`** — that directory does not exist, and the existing VFS tests (`vfs-core.test.ts`, `mount-table.test.ts`) live here. |
| `e2e/045-method-packs.spec.ts` | FR-017: phase strip incl. `blocked`, Method dropdown, preflight, section badges. Bundled fixtures. |
| `e2e/_fixtures/method-pack/` | **FR-017a's fixture pack** — a minimal local `user-apps` item with a `method/` facet, installable through the API at test time, no network. Prerequisite for `046` FR-015, `047` FR-013, `048` FR-013. |

Existing **unit** coverage that MUST be updated: `tests/specs/project-layer.test.ts:199-213` pins the cross-store `discrepancies.md` read that FR-006a changes. The system-store half of that read (`pipeline.ts:100-102`) has no test at all, so FR-006a's required explicit exception would otherwise ship unverified.

Existing e2e that MUST be updated (FR-017): `e2e/013-artifact-reload.spec.ts`, `e2e/013-spec-anchor.spec.ts`. `e2e/build-studio.spec.ts:49-56` should be converted to the fixture pack, retiring its `test.skip(true, …)`.

### Modified

| Path | Change |
|---|---|
| `src/lib/specs/types.ts` | `PhaseId` union (`:4-13`) → `string`; `PipelinePhase` gains `label`; `PhaseState` gains `blocked` (`pending` retained, §3.4); `ARTIFACT_FILES` (`:87-94`) deleted in favour of descriptor globs. |
| `src/lib/specs/pipeline.ts` | **FR-006/FR-006a.** The bulk. `CONSTITUTION_REL` (`:36`) → descriptor. `derivePhases` (`:81-126`) → `evaluate()`. Four leaf-rule copies (`:202`, `:215`, `:291`, + `projects.ts:85`) → one predicate. `walkFeatureLeaves`/`buildFolderNode`/`specTree` iterate sections. `nextFeatureId` (`:496-522`) honours `numbering`. `byArtifactOrder` (`:187`) sorts by descriptor order. FR-006a: the hardcoded cross-store `discrepancies.md` read (`:94-104`) becomes descriptor-declared. |
| `src/lib/specs/projects.ts` | `isFeatureLeaf` (`:85`) → shared predicate. **Also** `ProjectManifest` (`:15-18`) gains `method?`, and `readProjectManifest` (`:38-48`) must carry it — today it reconstructs `{label, description}` and silently drops every other key, so FR-008's override would never fire. |
| `src/lib/specs/stores.ts` | `StoreManifest` + `readManifest` (`:62-78`) accept `method` — same drop-unknown-fields shape as above. |
| `src/lib/specs/seed.ts` | `:89`, `:111` — merge the manifest instead of overwriting from a constant, or `method` is clobbered every boot. |
| `src/lib/specs/create.ts` | `:26` (`fs.access(… "spec","spec.md")`) and `:123` (`installItem({ files: { "spec/spec.md": … } })`) → descriptor's first artifact (FR-015). |
| `src/lib/specs/item-stores.ts` | `:67`/`:80` — the `spec/` facet probe and store root → descriptor-aware (FR-015). Also the module behind spec.md's item-store Edge Case: item stores have no `spec-store.json`, so `resolve.ts` falls back to the global default unless the item supplies one. |
| `src/lib/specs/spec-mount.ts` | `:63` `/Templates` → per-pack `/Methods/<packId>/templates`; drop the `mounted` latch (`:43`+`:46-47`); mount/unmount on pack install/uninstall. |
| `src/os/vfs.ts` | **Add `unregisterMount`.** `registerMount` (`:60-65`) replaces in place, but `mounts` (`:57`) is only ever `findIndex`/`push`/`find` — never spliced — so there is no removal path. FR-012's "uninstall MUST unregister" and FR-016's "method not installed" both require one; without it an uninstalled pack stays mounted at a deleted directory. This is an OS-primitive addition and the lowest-layer change in the feature. |
| `src/lib/dev/spec-fs.ts` | Delete `TEMPLATES_ROOT` (`:410`) and the callerless `readTemplate`/`listTemplates` (`:421-431`) — verified zero callers repo-wide. |
| `src/apps/build-studio/index.tsx` | Delete `PHASE_ORDER`/`PHASE_LABEL` (`:66-88`); `PhaseStrip` (`:204-219`) renders server-supplied phases incl. `blocked`; section badges (FR-010b); "method not installed" store state (FR-016); mount `MethodPicker`. |
| `src/app/api/specs/route.ts` | `GET` returns the active descriptor (FR-014). `PUT`/`PATCH` writes `method` into `spec-store.json` on the active `bos/*` branch (FR-009's write path). |
| `src/instrumentation.ts` | **Registers the built-in descriptor at boot**, before the first spec-store read (§3.1 named this actor; nothing owned it). `ensureSystemMounts`/`ensureStoresOnce` run lazily (`os/vfs.ts:95-106`), so this ordering is a live hazard, not a formality — and `046` FR-008 assumes the sequence already exists. |
| `src/lib/agent/subagents/types.ts` | `Agent` gains its contributing root (no such field today, `:3-36`) — FR-002's grouping cannot be rendered without it. |
| `src/app/api/subagents/route.ts` | `:21` spreads `...a`, so a new `Agent` field survives — but its consumers are `BuildStudioTab`, `TelegramBotAgentConfig`, `AgentDetails`, `NewAgentDialog`, `DangerZone`. **Not the picker.** |
| `src/app/api/assistant/agent/route.ts` | **The picker's actual route**, and a hand-written field whitelist (`:53` `id: a.id`, …) — the same drop-unknown-keys trap already documented twice here for `readManifest`/`readProjectManifest`. The contributing root must be added explicitly or FR-002 cannot render. |
| `src/components/apps/assistant/AgentSelector.tsx` (`:26`), `src/components/apps/settings/assistant/AssistantTab.tsx` (`:32`) | The two real pickers, both fed by that route. FR-002's grouping and FR-001's escape hatch belong here. |

| `src/lib/assistant/tools/server/subagents.ts`, `tools/server/delegate-common.ts` | FR-016's second clause: a delegation to an agent from an uninstalled pack fails naming the pack (`subagents.ts:115`/`:123`, `delegate-common.ts:30`). |
| `src/lib/agent/subagents/store.ts` | `agentsDir()` (`:17`) → roots list across **24 call sites** — all in this one file; zero elsewhere in `src/`. `listSubAgents()` (`:321`) merges roots by precedence. `applySeedAgent` (`:53`) and `archiveDroppedSeedAgents` (`:102`) pinned to `data/agents/` only (FR-001b). |
| `src/system/items/installed.ts` | `ItemFacets` (`:66-75`) + `readFacets` (`:99-111`) gain `method`. |
| `src/lib/marketplace/client.ts` | **The real install orchestrator** — `src/system/marketplace/install/` holds only `bundledAssets.ts`/`serviceInstaller.ts`/`symlinkManager.ts` and was cited in error. `installMarketplaceItem` (`:874`, guard `:880`, throw `:881`) throws `has nothing installable` for any item without app/services/integration/voiceEngine, so a method-only pack is refused server-side even if the UI issues the op. `uninstallMarketplaceItem` (`:1032-1051`) dispatches on `facets.plugin/service/app` with no `method` branch. Both need one, calling `method/install.ts`. |
| `src/lib/marketplace/schema.ts` | `MarketplaceItem` gains a `method` facet. |
| `src/apps/marketplace/index.tsx` | `:173` — `if (item.app \|\| item.services \|\| hasPluginFacet(item))` must also fire for a `method` facet, or no install op is issued. `:383` — `hasInstallableFacet` must include `method`, or **no Install button renders at all**. `:169-187`'s `installedKinds` gains `"method"` for the post-install notice. FR-013's `plugin/`-facet warning renders here at install. |
| `src/lib/config/registry.ts` | `:96-127` build-studio namespace gains `defaultMethod`. |
| `src/components/apps/settings/BuildStudioTab.tsx` | Renders `defaultMethod`; FR-001's escape hatch revealing `delegate-only` agents; FR-001a's same-id collision report. |
| `docs/dev/build-studio.md`, `docs/dev/architecture-overview.md` §9 | FR-018. Both describe the hardcoded model today. |

**Not modified here, though one might expect it:** `src/lib/self-heal/brief.ts`, `src/app/api/specs/run-tests/route.ts`, `src/lib/agent/config.ts`, `src/lib/agent/tool-groups.ts` — moved to `046` during review; they are spec-kit *de-coupling*, that feature's subject.

---

## 5. Integration points (existing mechanisms this design calls into, does not modify)

- **`.seed-stamp` three-way reconciliation (`subagents/store.ts:53-77`, `:170-172`)** — "refresh what BOS wrote and nobody edited; user edits always win." Multi-root inherits it unchanged; FR-001b only constrains *where* it writes.
- **`reconcileInstalledItemAssets` (`store.ts:181-185`)** — items already bundle agents (`040`); a pack shipping agents is that mechanism with a different payload.
- **`listStores` (`stores.ts:85-106`)** — store discovery and its owner ranking. The structural template for FR-001a, explicitly *not* its ordering (store rank is by owner; agents have no owner axis).
- **`itemProvenanceKey` / `deriveOrigin` (`system/items/installed.ts:56-64`, `:118-128`)** — how FR-013a's origin gate knows a pack came from `user-apps`. No new provenance tracking; the opt-in *record* is new (`method/install.ts`), the origin *derivation* is not.
- **`branchSpecsRoot` / `supervisorBegin` (`dev/spec-fs.ts`)** — branch-coupled routing (`020`). Descriptor resolution happens after root resolution, so branch behaviour is untouched.
- **`readNamespace` / `patchNamespace` (`lib/config/`)** — backing store for `defaultMethod`.
- **Symlink install (`035`)** — the `method` facet is found by the existing depth-2 scan; nothing is copied.

---

## 6. ADRs

### ADR-1 — Descriptor rules are declarative data, not pack-supplied code

**Context.** Phase state must be derivable from arbitrary frameworks' artefacts. A pack could supply an evaluator.

**Options.** (a) Declarative predicate DSL. (b) Pack-supplied JS evaluated in-process. (c) Evaluator in a worker-thread service facet.

**Decision.** (a).

**Consequences.** No marketplace item executes code in the Next.js pipeline path — the property that lets `047` be genuinely code-free and confines `048`'s code to its `plugin/` facet behind an origin gate. Cost: the DSL must be expressive enough, and it already needed extending once (FR-004a's set predicate, added when BMAD's sharded stories proved the original seven insufficient). That extension is the evidence for the risk: a third framework may force another. (b) also risks a malformed pack evaluator breaking spec discovery for every store.

### ADR-2 — Phases are a DAG, resolved by ordered clauses

**Context.** `PHASE_ORDER` is a linear array. OpenSpec declares `requires` edges: *"Dependencies are enablers, not gates."* But spec-kit's `derivePhases` (`pipeline.ts:106-124`) is not a simple done/not-done function — three phases are `na` while reachable.

**Options.** (a) Keep linear, approximate OpenSpec by ordering. (b) DAG with a boolean `done` predicate, renaming `pending` → `available`. (c) DAG with a boolean predicate, retaining `pending`. (d) DAG with an **ordered clause list** per phase plus an `else` state.

**Decision.** (d). **(b) and (c) were both earlier drafts and are both rejected** — see §3.4.

**Consequences.** Parity holds *by construction* rather than by inspection: every branch of today's `derivePhases` maps to a clause, so the only state change is `na → blocked` where a `requires` edge is genuinely unsatisfied. `specs` and `design` can be simultaneously reachable, inexpressible before. Cost: the descriptor is more verbose than a single predicate per phase, and a pack author must understand clause ordering. That is the right trade — (b) broke SC-001 by renaming, (c) broke it by collapsing three reachable-`na` phases into `pending`, and neither failure was visible without reading `derivePhases` line by line.

### ADR-2a — State vocabulary is pack-supplied, separately from phase labels

**Context.** `047` requires that where spec-kit shows "pending", OpenSpec shows "available" (its FR-005, SC-003, US3).

**Options.** (a) Rename the state globally. (b) Carry it on the phase's `label`. (c) A separate `stateLabels` map on the descriptor.

**Decision.** (c). **(b) was the first revision's claimed fix and does not work** — `label` is the phase's display name and replaces `PHASE_LABEL` (`index.tsx:78-88`); it has no room for state vocabulary.

**Consequences.** State ids stay stable (parity), presentation is per-pack, and `047` stays code-free. Cost: one more descriptor field and a second lookup in `PhaseStrip`. Without it `047` FR-005/SC-003 are unsatisfiable and would have to be struck from that spec instead.

### ADR-3 — The binding lives in `spec-store.json`, not in BOS config

**Context.** Something must record which method governs a store.

**Options.** (a) `spec-store.json` (store content, version-controlled). (b) A BOS-side map in `data/config/`. (c) Per-user preference.

**Decision.** (a), with an optional `project.json` override.

**Consequences.** Q5's multi-user answer falls out for free: a shared store carries one agreed method that travels with the repo, so two users cannot disagree about a store — they can only lack the pack it names (FR-016). Cost: changing method is a commit on a feature branch rather than a UI toggle — heavier but correct under Principle IV. Forces the `seed.ts` merge fix, and forces `readProjectManifest` to stop dropping unknown keys.

### ADR-4 — Multi-root agent discovery rather than copying pack agents in

**Context.** Packs contribute agents. `040` already copies item-bundled agents via `reconcileInstalledItemAssets`.

**Options.** (a) Copy into `data/agents/` (reuse `040`). (b) Discover in place from a registered root.

**Decision.** (b).

**Consequences.** Follows `035`'s copy-nothing rule and matches every other discovery path in BOS. Fork-on-edit becomes precedence: a user override is a same-id entry in `data/agents/` shadowing the pack copy — which also means their edits silently stop receiving pack updates, so `046` FR-010 requires shadowing be reported. Cost: 24 call sites, all confined to `subagents/store.ts` (zero elsewhere in `src/`), and one precedence rule stated once (`048` FR-002a extends it rather than adding a second).

### ADR-5 — Build then extract: spec-kit's descriptor lands in-source here, relocates in `046`

**Context.** `045` must not ship with zero packs, but relocating spec-kit's assets is a large independent change.

**Options.** (a) Relocate everything in `045`. (b) Descriptor in-source here; assets in `046`. (c) Ship with no descriptor; let `047` be first.

**Decision.** (b).

**Consequences.** `045` is provable against real content (SC-001) without a large asset move in the same change. Cost: `046` must delete the temporary in-source path; its FR-009/SC-005 enforce that. (c) rejected because it defers all validation to a pack that depends on the layer being right.

### ADR-6 — `/Templates` survives as an alias, retired by an owner that must be named

**Context.** FR-011 replaces a global mount with per-pack mounts. Existing prompts reference `/Templates/commands/<step>.md` (e.g. `seed/skills/build-studio/SKILL.md:66`).

**Options.** (a) Hard cutover. (b) Alias to the active store's pack, retired later.

**Decision.** (b).

**Consequences.** Prompts keep working across the `045`→`046` boundary where templates physically move. Cost: a temporary ambiguity — "the active store" is well-defined for Build Studio but not for an agent with no store in context, where it resolves to the global default. An earlier draft claimed `046` FR-002 retires the alias; it does not — FR-002 relocates `.specify/templates` and forbids reading it from source, saying nothing about the alias. **`046` FR-017 was added to own retirement explicitly**, so "time-boxed" is now a plan rather than an intention.

---

## 7. Risks / open questions

**R1 — `pipeline.ts`'s draft-branch overlay is the regression hotspot.** `listSpecifications` (`:316-342`), `specTree` (`:421-479`), `touchedFeaturesOnBranch` (`:288-304`) and `insertAtPath` (`:258-282`) implement subtle behaviour: branch-only features grafted into the base tree at their real nesting, with same-path replacement to avoid duplicate React keys (`:270-273`). Section-aware walking touches all of it. *Mitigation:* SC-001's snapshot must include a live `bos/*` branch and be captured **before** any refactoring. If it isn't captured first it cannot be captured at all.

**R2 — The parity harness does not exist and gates everything.** `tests/specs/method-parity.test.ts` is new work and must land before Stage A's refactor, not alongside it.

**R3 — The e2e fixture-pack harness is a prerequisite four specs depend on.** FR-017a. `e2e/build-studio.spec.ts:49-56` currently passes by skipping when no marketplace item is installed — the exact pre-seeded-data dependency BOS's testing rule prohibits. Until a fixture pack installs through the API at test time, `046` FR-015, `047` FR-013 and `048` FR-013 can only be satisfied the same way.

**R4 — `unregisterMount` is a change to the lowest layer in the system.** `src/os/vfs.ts` is Core-OS, stability HIGH, and every mount consumer shares it. A removal path that leaves a dangling backend or races an in-flight read is worse than the leak it fixes. Scope it narrowly (remove by prefix, no-op if absent) and unit-test it independently of the method layer.

**R5 — 31 requirements in one feature.** Raised four times in stress-testing, mitigated with A/B/C staging rather than a split, because the descriptor and its consumers cannot land independently without a temporary second code path. If Stage B or C sprawls, split rather than widen.

**R6 — Stage B's interim template mapping is a deliberate half-state.** `/Methods/spec-kit/templates` resolves to `.specify/templates` in source until `046` moves the files. Called out here because an implementer reading only the FR list would read it as a bug.

**Open question (for `plan`):** the descriptor carries both `agents[]` and `roles{}`. `046` consumes `agents[]`; `048` consumes `roles{}` (its FR-011b). Whether `roles{}` needs `045`-side machinery beyond being carried in the type, or is pack-side metadata until `048`, should be settled at `plan`.

---

## 8. Review findings and resolution

One independent review pass; one revision round (the method's cap). Verdict was `Needs significant rework`, driven by file-plan coverage gaps rather than by the architecture, which the review found sound and asked to carry forward unchanged.

| # | Finding | Resolution |
|---|---|---|
| M1 | `pending` renamed to `available`, breaking SC-001 by construction | **Fixed.** §3.4 retains `pending`; presentation moves to the descriptor `label`. ADR-2 rewritten with the rejected option recorded. |
| M2 | "no unmount API needed" — but none exists, and FR-012/FR-016 require one | **Fixed.** `unregisterMount` added to the plan as an OS-primitive change; R4 added for its blast radius. |
| M3 | ~7 FRs unowned in the file plan (all of Stage C's behaviour) | **Fixed.** Added `method/install.ts`, `method/constitution.ts`, the `/api/specs` write path, `system/marketplace/install/` (moved out of §5), FR-016's UI state, FR-013's warning. |
| M4 | FR-015's three call sites absent | **Fixed.** `create.ts` and `item-stores.ts` added; the item-store Edge Case now addressed. |
| M5 | Marketplace row cited a display-string region; the real blockers are `:173`/`:383` | **Fixed.** Row rewritten. FR-012a's "kind list / sidebar filter" describes machinery that does not exist — flagged for `analyze`. |
| M6 | `project.json` override has no mechanism — `readProjectManifest` drops unknown keys | **Fixed.** `ProjectManifest` + `readProjectManifest` added to the projects.ts row. |
| M7 | Constitution check covered 4 of 7 principles | **Fixed.** All seven; VI is load-bearing (owns FR-018 and the `discrepancies.md` obligation). |
| S1 | No test files in the plan, including the two prerequisite harnesses | **Fixed.** New *tests* subsection; existing e2e to update named. |
| S2 | ADR-6's time-box rests on a `046` FR that doesn't retire the alias | **Fixed** (pass 3): `046` FR-017 now owns retirement explicitly. |
| S3 | FR-001's escape hatch and FR-001a's collision report had no surface | **Fixed.** Both assigned to `BuildStudioTab.tsx`. |
| S4 | `agentsDir()` call sites understated (~15 vs 24) | **Fixed**, with the reviewer's useful addition: zero outside `subagents/store.ts`. |
| S5 | `discrepancies` required by FR-006a but missing from the descriptor field list | **Fixed.** Added to §3.3; spec reconciliation routed to `analyze`. |
| S6 | One plan row was a directory | **Fixed.** `BuildStudioTab.tsx`. |

### Second review pass

The user authorised iterating past the method's one-round cap. A second independent pass returned `Needs one revision round` — 7 must-fix, 9 should-improve. It confirmed M2, M5, M6, M7 and S4 genuinely landed, and found that two claimed fixes had not.

| # | Finding | Resolution |
|---|---|---|
| MF-1 | M1's fix was half-done: `label` is the *phase's* name, so nothing could render OpenSpec's "available" | **Fixed.** New §3.5 + ADR-2a: a separate `stateLabels` map. `047` FR-005/SC-003 are satisfiable again. |
| MF-2 | The state rule created `na → pending` at scale — `implement`/`test`/`converge` are `na` today *while reachable* | **Fixed.** §3.4 replaced the boolean predicate with an ordered clause list + `else`, so parity holds by construction. ADR-2 rewritten. **This was the most serious defect in either pass.** |
| MF-3 | `installMarketplaceItem` (`client.ts:880-881`) refuses a method-only item server-side; the cited install module was wrong | **Fixed.** §4 now names `src/lib/marketplace/client.ts` and both entry points. |
| MF-4 | No task registered the descriptor at boot, though §3.1 named the actor | **Fixed.** `src/instrumentation.ts` added. |
| MF-5 | FR-002 (picker grouping) had no real surface | **Fixed.** `subagents/types.ts` (`Agent` gains its root), `api/subagents/route.ts`, `AgentsTab.tsx`. |
| MF-6 | FR-016's delegation clause was unowned | **Fixed.** `tools/server/subagents.ts`, `delegate-common.ts`. |
| MF-7 | Three documents disagreed on staging (FR-017a, FR-018) | **Fixed** in `tasks.md`/`plan.md` — see the reconciliation commit. |
| SI-1 | Dependency-graph defects; critical path not a path | **Fixed** in `tasks.md`. |
| SI-2/3/4 | Stage B needs Stage C's API; FR-008 straddles; Stage A descriptor acquisition unspecified | **Fixed** in `tasks.md`/`plan.md`. |
| SI-5 | `plan.md` never answered the `roles{}` question routed to it | **Fixed** in `plan.md`. |
| SI-6 | Three more descriptor gaps breaking siblings' "no code" premise | **Fixed.** `artifacts[].scope`, `sections[].terminal`, `testFile` added to §3.3. |
| SI-7 | `tests/os/` is not a real convention | **Fixed.** `tests/specs/vfs-unmount.test.ts`. |
| SI-8 | Existing unit coverage of the `discrepancies` read unnamed | **Fixed.** `tests/specs/project-layer.test.ts:199-213`. |
| SI-9 | Citation slips (SC-014 vs SC-013, latch lines, 523 vs 522) | **Fixed** across documents. |

### Third review pass

A third independent pass built a truth table across all **132 live feature leaves** and returned `Needs one revision round` — 6 must-fix, 12 should-improve. It confirmed `stateLabels` genuinely satisfies `047` (its FR-005/SC-003/US3 are worded as *display*, and US3's one state-semantics scenario is delivered by the DAG), and confirmed MF-3/4/6 and SI-7/8 from pass 2 landed exactly.

| # | Finding | Resolution |
|---|---|---|
| MF-1 | `requires` edges were specified **nowhere**, and §3.4's "preserved" claim was false under the obvious ones — `converge requires implement` flips 120 features, `test` 114 | **Fixed.** Spec-kit declares `requires: []` throughout; gating lives in clauses. Output is now byte-identical and `blocked` is unreachable under spec-kit, so SC-001 tightened from "enumerate substitutions" to **zero state changes**. Edge satisfaction defined; permanently-`na` phases barred as edge targets. |
| MF-2 | The constitution read is cross-store like `discrepancies`, and FR-006a's treatment was never extended to it — per-store resolution gives `done → pending` on 8 live item stores | **Fixed.** New §3.6, `constitutionRoot` field, FR-006b. The module-global `constitutionReady` memo is called out. |
| MF-3 | `artifacts[]` ordering would change on ≥23 features; SC-001 requires it identical | **Fixed.** Separate `artifactOrder[]` reproducing `ARTIFACT_FILES` incl. its `localeCompare` tail. |
| MF-4 | Pass 2's MF-5 fix landed on the wrong files **again** — `AgentsTab.tsx` is a 48-line shell that fetches nothing; the picker is `AgentSelector.tsx`/`AssistantTab.tsx` over `/api/assistant/agent`, itself a field whitelist | **Fixed.** All three real files named, and the whitelist trap called out. |
| MF-5 | `sections[].terminal` / `artifacts[].scope` were type-only — same defect shape as pass 2's MF-1 | **Fixed.** T022 owns terminal-section rendering; T004 owns scope-relative globs. |
| MF-6 | ADR-6 and `plan.md` still said the alias had no owner after `046` FR-017 was added | **Fixed** in both. |
| SI-8/9 | `test` keys on content not listing; `checklist` vacuously true on empty lists — 9 live features, including this one | **Fixed.** `nonEmpty` predicate and a `total > 0` guard, both in T004. |
| SI-1..7, 10..12 | Staging layout, dangling T030 ref, SC-013/014 slip, critical-path non-edge, missing graph edges, FR-001 count, `modules[]`, legend, SC-001 corpus, `client.ts` lines | **Fixed** across the four documents. |

**Spec-level issues — all now resolved.** The first pass routed five to `analyze`; the second pass added three descriptor gaps. All eight have since been applied to `spec.md` (FR-003's field list, FR-012a's marketplace wording, the `nextFeatureId`/`item-stores.ts`/latch/line-count citations) or to `046` (its new FR-017, retiring the `/Templates` alias — the owner ADR-6 said was missing). Nothing is left deferred.

---

## 9. Seams where the obvious implementation is wrong

Each of these is a decision that looks like a detail and is not. They sit where
this layer meets a long-lived server, a user's git repositories, or a restart —
and in every case the natural implementation fails silently.

### 9.1 Manifests are REWRITTEN, so a reader is also a deleter

It is natural to treat a manifest reader as read-only and type it defensively:
parse, pick the fields we model, return a narrow object. That is correct for a
reader and catastrophic here, because BOS **writes these files back** — the store seed normalises `spec-store.json`, marketplace repair
rewrites `marketplace.json`. A field the parser did not name was therefore not
merely ignored; it was **erased from the user's own git repo and committed**.

Four readers in this subsystem have that shape (`readManifest`,
`readProjectManifest`, an agent-route field whitelist, `parseManifest`). Four
occurrences of one mistake in one layer is a sign the shape invites it:
reconstruct-and-return reads as good hygiene.

**Rule: in a system that round-trips a file, narrowing is a mutation.** Spread
the parsed object and narrow only what you validate; merge into the original on
write. The corresponding test is not "does the parser read field X" but "does an
unknown field survive a BOS-initiated rewrite".

### 9.2 Resolution memos and long-lived servers

`methodFor` is memoised per store because listing 143 features would otherwise
resolve the descriptor 143 times. It is tempting to treat that memo as
request-scoped; it is a module global, and in a long-lived server a module
global outlives the thing it caches.

If any entry point fails to clear it, a store rebound to another method keeps
resolving through the old descriptor indefinitely, until something unrelated
happens to clear it. This surfaces as a "flaky" test — the characteristic
disguise, since same-process staleness looks like nondeterminism because whether
it bites depends on what ran before.

**Rule: a cache whose key can be rebound at runtime needs an explicit lifetime
and a stated invariant at the declaration site.** "Clear it at the entry points"
is not enforceable by types, so it must be written where someone adding a sixth
entry point will read it.

### 9.3 "Resolves" and "can be set" are different requirements

The binding chain `project > store > global` was specified, implemented and
unit-tested — and two of its three levels had no write path. The resolver test
passed the whole time. It was testing a pure function whose inputs the product
could not produce.

The item-store case is the sharper version: those stores are *synthesised* from
a directory scan and have no manifest, so there was nowhere for a binding to
live. The feature was unimplementable at that level and nothing said so.

**Rule: every "X resolves from Y" requirement needs a paired "Y can be set"
requirement, and a test that drives the setting path rather than the resolver.**

Where the binding is *stored* is a real decision, not a detail: an item's
binding lives in the item's own `spec/` folder, because the method its specs are
written in is a property of that content and must travel when the item is
published. Storing it per-install would let the same specs be read through a
different framework on another machine.

### 9.4 A descriptor field is not shipped until every consumer honours it

`sections` and `terminal` are both easy to add to the schema and leave
half-honoured. If `preflight` walks sections and discovery does not, two parts
of one feature disagree about what a store contains: a store bound to a
root-owning method renders empty while its preflight correctly reports the units
it would orphan. If `terminal` is declared and rendered as a tooltip but
consulted by no evaluator, archived units report live phase state.

**Rule: a descriptor field is not shipped when it parses; it is shipped when
every consumer honours it.** A field added to the schema needs an explicit
inventory of its consumers, and a test per consumer. Where a field can nest
(sections within sections), the ownership rule — longest prefix wins — belongs in
exactly one function, or nested content is discovered twice.

### 9.5 The store root is not always BOS's to organise

The `033` Project migration assumes a store root contains feature folders, so
anything else must be pre-Project content to rescue — it renames it all into a
default Project and commits. That assumption is invisible and was true for every
method that existed when it was written.

A method that declares sections at the store root makes it false, and the
failure is destructive: content moves where the descriptor cannot find it,
before any UI renders, committed. The store appears to have emptied itself.

**Rule: when a feature makes a previously-universal assumption configurable,
audit what *already* depends on it — especially migrations, which run before
anything is displayed and write before anyone can object.** The safe default for
a destructive rearrangement is to refuse when the layout is unknown; an
unregistered method must block it exactly as a root-owning one does.
