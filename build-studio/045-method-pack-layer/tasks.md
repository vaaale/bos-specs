# Tasks — 045 The method-pack layer

Spec: [`spec.md`](./spec.md) · Design: [`design.md`](./design.md) · Plan: [`plan.md`](./plan.md)

Phases map to the plan's A/B/C stages plus a prerequisite phase and a verification phase. Each phase ends at an independently revertible point with a green `npm run test:unit` (SC-009).

**Two ordering rules that are not negotiable:**
1. **T001 runs before any `pipeline.ts` edit.** The parity baseline cannot be captured retroactively — miss it and SC-001 is unverifiable for the life of the feature (R1/R2).
2. **Zero phase-state changes.** Spec-kit declares `requires: []` on all nine phases (T006), so `blocked` is **unreachable** under it and output is byte-identical. `blocked` appearing at all means an edge was introduced that design.md §3.4 forbids. Any change — `na → blocked` included — is a parity failure. (An earlier draft of this rule permitted `na → blocked`; pass 3 tightened SC-001 after a truth table showed the edges that would cause it also flip 120 `converge` and 114 `test` phases.)

---

## PHASE 0 — Prerequisites (blocks everything)

### T001 — Parity harness + baseline snapshot
- **Files:** `tests/specs/method-parity.test.ts` (new)
- **Story:** SC-001 gate / foundation
- **Work:**
  - Snapshot every `Specification` `listSpecifications()` returns — both directory-scanned stores **and the 8 item-owned stores** (`data/user-apps/items/*/spec/`). The item stores are not optional: `constitutionRoot` (FR-006b) exists *only* because per-store constitution resolution flips them `done → pending`, and a corpus without them cannot observe the regression T008 would introduce.
  - **Include a live `bos/*` branch** so the draft-branch overlay is covered (`listSpecifications:316-342`, `specTree:421-479`, `touchedFeaturesOnBranch:288-304`, `insertAtPath:258-282`).
  - Field-level comparison per SC-001: identical `id`, `store`, `title`, `path`, `artifacts[]`, `taskProgress`, `branch`, and identical `{id, state}` phase pairs in order. `label` is the **only** permitted addition; **no state substitutions at all**.
  - Fail loudly on any `blocked` under spec-kit — it means a `requires` edge was introduced.
- **Acceptance:** harness green against unmodified `main`; baseline committed. **Nothing in Phase 1 starts until this passes.**

### T002 — `unregisterMount`
- **Files:** `src/os/vfs.ts`, `tests/specs/vfs-unmount.test.ts` (new)
- **Story:** FR-012, FR-016 / foundation
- **Work:**
  - `registerMount` (`:60-65`) replaces in place, but `mounts` (`:57`) is only ever `findIndex`/`push`/`find` — never spliced. There is no removal path today.
  - Add `unregisterMount(vfsPrefix)`: normalize the prefix identically to `registerMount`, splice the entry, no-op if absent.
  - Do **not** broaden scope — no mount lifecycle events, no refcounting.
- **Acceptance:** unit-tested independently of the method layer; removing a mount makes its paths fall through to default VFS behaviour; no dangling backend. Core-OS change, so reviewed as such (R4).

### T002a — E2E fixture pack
- **Files:** `e2e/_fixtures/method-pack/` (new)
- **Story:** FR-017a
- **Work:**
  - A minimal local `user-apps` item with a `method/` facet, installable through the API at test time, no network.
  - **`spec.md` FR-017a places this in Stage A** and the first task draft wrongly put it in PHASE 4 — while T018 (preflight, needs a v2 descriptor changing `leafMarker`) and T024 (origin gate, needs the same pack offered from two origins) both depend on it. That inverted the graph.
  - Prerequisite for `046` FR-015, `047` FR-013 and `048` FR-013 — until it exists those specs can satisfy their e2e criteria only by skipping.
- **Acceptance:** installable and removable within a single test run; consumed by T018 and T024.

---

## PHASE 1 — Stage A: descriptor & pipeline

### T003 — Descriptor and rule types
- **Files:** `src/lib/specs/method/types.ts` (new)
- **Story:** FR-003
- **Work:**
  - `MethodDescriptor`: `schemaVersion`, `id`, `label`, `version`, `builtin?`, `sections[]` (`rel`, `kind: "active"|"truth"|"archive"`, `leafMarker`, `numbering: "nnn-slug"|"none"`, `terminal?`), `constitution`, `constitutionRoot`, `discrepancies`, `testFile?`, `artifacts[]` (`{id, generates, requires, scope: "store"|"unit"}`), `artifactOrder[]`, `phases[]` (`{id, label, requires, rules[], else}`), `stateLabels`, `templates`, `agents[]`, `roles{}`, `modules[]?`.
  - **Seven** fields exist because a sibling or a review round required them, and FR-003 originally omitted them: `discrepancies` (FR-006a), `stateLabels` (`047` FR-005), `sections[].terminal` (`047` FR-006b), `artifacts[].scope` (`047` FR-006a — `specs` means two different things at two roots), `testFile` (`046` FR-014), **`constitutionRoot`** (FR-006b — without it the 8 item stores flip `done → pending`), **`artifactOrder[]`** (separate from `artifacts[]`: `ARTIFACT_FILES` excludes `design.md`/`test-results.md`, which fall to `byArtifactOrder`'s `99 → localeCompare` tail, and 23 live features carry `design.md`). T006 and T008 both consume the last two — omit them and neither compiles.
  - `discrepancies` is required by FR-006a's prose though absent from FR-003's list — include it (design.md §3.3); `analyze` reconciles the spec.
  - Rule types for the DSL predicates.
  - **Framework-free** — no `server-only`, no Node imports. The Build Studio client imports this (Principle II).
- **Acceptance:** `tsc` clean; importable from a client component without pulling server code.

### T004 — Rule evaluator
- **Files:** `src/lib/specs/method/evaluate.ts` (new), `tests/specs/method-evaluate.test.ts` (new)
- **Story:** FR-004, FR-004a
- **Work:**
  - Predicates: `manual` (FR-004's list; yields `na` unconditionally), `exists` (directory-listing presence), **`nonEmpty`** (file has content — `test` keys on content, not listing: `pipeline.ts:111-113` reads then tests truthiness, so an empty `test-results.md` is `na` today while `exists` would give `pending`), `contains`, `notContains`, `checklist`, `containsUnitId`, `dependsOn`.
  - **`checklist` MUST guard `total > 0`.** `pipeline.ts:108-109` gives `na` when `tasks.length === 0`; a vacuously-true "all done" yields `done` — an unsanctioned `na → done` on the 9 live features whose `tasks.md` has no parseable `- [ ]` items, **this feature's own included**.
  - Edge satisfaction: a `requires` target is satisfied when its resolved state is `done`. A phase that can never be `done` MUST NOT be named as an edge target.
  - Set predicate `{glob, quantifier: "any"|"all"|"none", of}` plus `count` — required for artifacts of unknown cardinality (BMAD's sharded stories; `048` SC-012 asserts no BMAD-specific evaluation code exists in the pipeline).
  - Pure over an **injected file-reader** — no direct `fs` — so it is unit-testable without a store.
  - **Phase state resolution is an ordered clause list, not a boolean** (design.md §3.4). Evaluate `rules[]` in order, first match wins; if none matches, the state is `blocked` when a `requires` edge is unsatisfied, else the phase's `else` (default `pending`).
  - This is not a style choice. A naive *done-else-pending-else-blocked* rule flips three phases from `na` to `pending` across every existing feature: `implement` with `tasks.md` present and 0 done (`pipeline.ts:108-109`), `test` with no results file (`:111-113`), and `converge` with no discrepancy entry (`:123`, which is never `pending` today). That is an unsanctioned `na → pending` at scale and would have surfaced at T011, after T008's refactor.
- **Acceptance:** SC-007 — a phase depending on N dynamically generated artifacts is expressible, verified with a fixture glob whose cardinality is unknown at authoring time, not merely with spec-kit's fixed nine.

### T005 — Method registry
- **Files:** `src/lib/specs/method/registry.ts` (new), `tests/specs/method-registry.test.ts` (new)
- **Story:** FR-003a
- **Work:**
  - `registerMethod` / `unregisterMethod` / `getMethod` / `listMethods`.
  - `schemaVersion` gate: refuse registration of an unsupported version and **name the mismatch** (expected vs. found). Never default a missing or unrecognised field and continue.
  - `unregisterMethod` exists now because T024 needs it; keep it trivial here.
- **Acceptance:** SC-015 — a pack with an unsupported `schemaVersion` is refused with a message naming both versions; no store silently renders empty.

### T006 — Built-in spec-kit descriptor
- **Files:** `src/lib/specs/method/spec-kit.ts` (new)
- **Story:** ADR-5 / FR-003
- **Work:**
  - Express all nine phases as clause lists reproducing `derivePhases` (`pipeline.ts:106-124`) branch for branch. The three that need an explicit `else: "na"` are `implement`, `test` and `converge` — see T004.
  - `analyze` is a single `else: "na"`; no `manual` predicate is needed.
  - `stateLabels: {done:"Done", pending:"Pending", blocked:"Blocked", na:"N/A"}` — `047` supplies `pending: "Available"` (design.md §3.5).
  - **`requires: []` on all nine phases.** Gating lives in the clauses, not the edges. Under the linear edges `PHASE_ORDER` suggests, `converge requires implement` flips 120 features `na → blocked` and `test requires implement` 114 — `implement` is `done` on only 12 of 132. With no edges, output is byte-identical and `blocked` is unreachable under spec-kit.
  - `constitutionRoot: "system"` — preserves `hasConstitution()`'s cross-store read (design.md §3.6).
  - `artifactOrder[]` reproducing `ARTIFACT_FILES` verbatim, including the `99 → localeCompare` tail for `design.md`/`test-results.md`. 23 live features carry `design.md`; re-ranking it changes `Specification.artifacts[]`, which SC-001 requires be identical.
  - One section: `rel: ""`, `leafMarker: "spec.md"`, `numbering: "nnn-slug"`. `constitution: ".specify/memory/constitution.md"`.
  - `builtin: true`.
  - Expressed **in the same format a pack uses** — no privileged shape. `046` relocates this file.
- **Acceptance:** SC-002 — every phase resolves through T004's evaluator; zero `builtin`-special-case branches in the pipeline, verified by grep.

### T006a — Register the built-in descriptor at boot
- **Files:** `src/instrumentation.ts`
- **Story:** FR-003 / §3.1
- **Work:**
  - Register T006's descriptor via T005's registry **before the first spec-store read**. `design.md` §3.1 named this actor; no task owned it.
  - `ensureSystemMounts`/`ensureStoresOnce` run lazily (`src/os/vfs.ts:95-106`), so ordering is a live hazard. Assert registration rather than relying on import side-effects.
  - `046` FR-008 assumes this sequence already exists.
- **Acceptance:** a cold start followed immediately by `/api/specs` resolves a descriptor; no `builtin` shortcut needed (SC-002).

### T007 — Spec model types
- **Files:** `src/lib/specs/types.ts`
- **Story:** FR-005
- **Work:**
  - `PhaseId` union (`:4-13`) → `string`.
  - `PipelinePhase` gains `label`.
  - `PhaseState` gains `blocked`. **`pending` is retained** — see design.md §3.4 and the ordering rule above.
  - Delete `ARTIFACT_FILES` (`:87-94`); artifact ordering comes from the descriptor.
- **Acceptance:** no compile-time phase union remains (SC-003); `tsc` clean.

### T008 — `pipeline.ts` becomes descriptor-driven
- **Files:** `src/lib/specs/pipeline.ts`
- **Story:** FR-006, FR-006a
- **Work:**
  - `derivePhases` (`:81-126`) → T004's evaluator.
  - `CONSTITUTION_REL` (`:36`) → descriptor, resolved against `constitutionRoot` (FR-006b). `hasConstitution()` (`:72-79`) currently reads through `systemStoreId()` (`:74`) for every store; `constitutionReady` (`:71`) is a module-global memo and must become per-(descriptor, root).
  - `byArtifactOrder` (`:187-193`) sorts by `artifactOrder[]`, not by `artifacts[]`.
  - **Collapse four copies of the leaf rule into one predicate**: `:202`, `:215`, `:291` (the `endsWith("/spec.md")` variant — the non-obvious one) and `projects.ts:85`.
  - `walkFeatureLeaves` / `buildFolderNode` / `specTree` iterate the descriptor's `sections[]` rather than assuming one tree.
  - `nextFeatureId` (`:496-522`) honours `numbering`; `"none"` skips `NNN-` allocation entirely.
  - FR-006a: replace the hardcoded cross-store `discrepancies.md` read (`:94-104`, which reads a literal `user-specs/discrepancies.md`) with the descriptor's `discrepancies` path.
  - **Update `tests/specs/project-layer.test.ts:199-213`** — it pins the cross-store `discrepancies.md` read this task changes; leaving it red violates SC-009's per-stage green suite.
  - **FR-006a's explicit exception:** if the cross-store read is retained for `bos-system-specs`, that MUST be stated explicitly in the descriptor and in code, not left implicit.
  - **Do not touch the draft-branch overlay's semantics** — same-path replacement at `:270-273` exists to prevent duplicate React keys.
- **Acceptance:** SC-004 — exactly one code location decides whether a directory is a feature leaf. T001 parity still green.

### T009 — `isFeatureLeaf` delegates
- **Files:** `src/lib/specs/projects.ts`
- **Story:** FR-006
- **Work:** `:85` delegates to T008's shared predicate. (The `ProjectManifest` change is T016 — different stage.)
- **Acceptance:** no second leaf rule remains.

### T010 — PhaseStrip renders server-supplied phases
- **Files:** `src/apps/build-studio/index.tsx`
- **Story:** FR-007
- **Work:**
  - Delete `PHASE_ORDER` and `PHASE_LABEL` (`:66-88`).
  - `PhaseStrip` (`:204-219`) renders `{id, label, state}` from the server, with a `blocked` style distinct from `na`, and shows `stateLabels[state]` wherever the state is named (tooltip, legend) — the chip text stays the phase `label` (§3.5).
- **Acceptance:** no phase vocabulary remains client-side.

### T011 — Parity gate
- **Files:** — (verification)
- **Story:** SC-001
- **Work:**
  - Run T001. **Zero state changes expected.** Any `blocked`, any `pending` change, any `label`-unrelated field change: stop, fix, re-run. Not a refinement.
- **Acceptance:** SC-001 green. **Stage A revert point.**

---

## PHASE 2 — Stage B: discovery & binding

### T012 — Agent roots module
- **Files:** `src/lib/agent/subagents/roots.ts` (new)
- **Story:** FR-001, FR-001a
- **Work:**
  - `AgentRoot = { kind: "dir"; path; visibility: "picker" | "delegate-only" }`. `kind: "provider"` is `048`'s, not here.
  - Precedence, highest first: (1) `data/agents/`, (2) pack roots sorted by pack id (deterministic, not install order), (3) `seed/agents/`.
  - Two packs contributing the same agent id → **report a collision**, do not silently resolve by order.
- **Acceptance:** one precedence rule exists in the codebase; `048` FR-002a extends it rather than adding a second.

### T013 — Multi-root `listSubAgents`
- **Files:** `src/lib/agent/subagents/store.ts`, `tests/agent/agent-roots.test.ts` (new)
- **Story:** FR-001, FR-001b
- **Work:**
  - `agentsDir()` (`:17`) → roots list across **24 call sites** — all in this file; zero elsewhere in `src/`.
  - `listSubAgents()` (`:321`) merges roots by precedence.
  - **Pin seed writes to `data/agents/` only**: `applySeedAgent` (`:53`) and `archiveDroppedSeedAgents` (`:102`) must never write into a pack root. The `.seed-stamp` three-way contract (`:53-77`) is otherwise unchanged.
  - Registration and delegability are unconditional; only *discovery* is scoped by `visibility`.
- **Acceptance:** SC-006 — a `delegate-only` root is absent from the picker and reachable by `agent_delegate`. SC-008 — seed writes only to `data/agents/`.

### T014 — Picker grouping, escape hatch, collision report
- **Files:** `src/lib/agent/subagents/types.ts`, `src/app/api/assistant/agent/route.ts`, `src/components/apps/assistant/AgentSelector.tsx`, `src/components/apps/settings/AssistantTab.tsx`
- **Story:** FR-002, FR-001, FR-001a
- **Work:**
  - `Agent` (`types.ts:3-36`) gains its contributing root — it has no such field today, and FR-002's grouping cannot be rendered without it. **The picker's route is `/api/assistant/agent`, not `/api/subagents`** — and it is a hand-written field whitelist (`route.ts:53`), so the new field must be added explicitly or it is silently dropped, exactly like `readManifest`/`readProjectManifest`. `AgentsTab.tsx` is a 48-line shell that fetches nothing and is not the picker.
  - Group picker entries by contributing root, so similarly-named agents from different packs are distinguishable.
  - Escape hatch revealing `delegate-only` agents belongs in `AgentSelector.tsx` / `AssistantTab.tsx` — the two real pickers.
  - Surface T012's collision report.
- **Acceptance:** FR-002 satisfied without a per-agent visibility flag.

### T015 — Store manifest carries `method`
- **Files:** `src/lib/specs/stores.ts`, `src/lib/specs/seed.ts`
- **Story:** FR-008
- **Work:**
  - `StoreManifest` + `readManifest` (`:62-78`) accept `method`. The reader reconstructs a fixed object — adding the type field is not enough, it must carry the value.
  - `seed.ts:89`/`:111` — **merge** the manifest instead of rewriting from `SYSTEM_MANIFEST`/`USER_MANIFEST` constants. `:111` preserves only `label` today, so `method` is clobbered on every boot without this.
- **Acceptance:** a `method` set on `user-specs` survives a server restart (SC via US3 scenario 3).

### T016 — Project manifest carries `method`
- **Files:** `src/lib/specs/projects.ts`
- **Story:** FR-008
- **Work:** `ProjectManifest` (`:15-18`) gains `method?`; `readProjectManifest` (`:38-48`) carries it. Today it returns a freshly constructed `{label, description}` and drops every other key, so the override would never fire.
- **Acceptance:** a per-Project override resolves.

### T017 — Binding resolution
- **Files:** `src/lib/specs/method/resolve.ts` (new)
- **Story:** FR-008, FR-008a
- **Work:**
  - `spec-store.json.method` → `project.json.method` override → global default.
  - Absent everywhere ⇒ `spec-kit`, so nothing changes for existing stores.
  - Item-owned stores have no `spec-store.json` (synthesised at `item-stores.ts:57-93`) — fall back to the global default unless the item supplies one.
  - **The global default is per-user**, not per-deployment (FR-008a) — `data/` is a per-user volume under Bastion. Wording in Settings must say so.
- **Acceptance:** all four binding sources resolve in the stated precedence.

### T018 — Preflight (assignment **and** upgrade)
- **Files:** `src/lib/specs/method/preflight.ts` (new), `tests/specs/method-preflight.test.ts` (new)
- **Story:** FR-010, FR-010a, FR-010c
- **Work:**
  - Run discovery **twice** — effective descriptor vs. candidate — and report the set difference by feature path, listing paths, not just a count.
  - Include draft-branch-only features: those are what a user is most likely working on and least likely to notice vanishing.
  - **One implementation, two triggers.** FR-010c: the same check runs on a pack *upgrade* that changes the effective descriptor, not only on user-initiated assignment. On an upgrade that would orphan content, hold and report rather than apply.
  - Preflight is the **gate**; T019 is the **remedy**. Preflight evaluates first and may stop the change.
- **Acceptance:** SC-005, SC-010 — exercised with a fixture pack whose v2 descriptor changes `leafMarker`; one implementation verified by there being one.

### T019 — Constitution relocate-or-report
- **Files:** `src/lib/specs/method/constitution.ts` (new)
- **Story:** FR-009a
- **Work:** On an accepted method change, relocate the store's constitution to the new descriptor's path or report it as now-unreferenced; surface when the new path has no constitution at all. Single owner for this moment — `046` and `047` deliberately do not specify it.
- **Acceptance:** SC-012 — no store left with an unreferenced constitution and no silent absence.

### T020 — MethodPicker + write path
- **Files:** `src/apps/build-studio/MethodPicker.tsx` (new), `src/app/api/specs/route.ts`
- **Story:** FR-009
- **Work:**
  - Dropdown on the store-group header beside the existing owner/writable badges, listing installed packs. **Not rendered for `bos-system-specs`** (read-only, permanently spec-kit).
  - Writes `spec-store.json` on the active `bos/*` branch and rides the normal promote — `GET` is not enough, the route needs a write path.
  - Wire T018's preflight into the confirm step.
- **Acceptance:** US3 scenarios 1–3.

### T021 — Per-pack template mounts
- **Files:** `src/lib/specs/spec-mount.ts`, `src/lib/dev/spec-fs.ts`
- **Story:** FR-011
- **Work:**
  - `/Templates` (`:63`) → `/Methods/<packId>/templates`; drop the one-shot `mounted` latch (`:43`+`:46-47`) so mounts change on install/uninstall.
  - Keep `/Templates` as an alias to the active store's pack for one release (ADR-6).
  - **Stage B interim state:** only the built-in descriptor exists, so `/Methods/spec-kit/templates` resolves to `.specify/templates` in source until `046` moves the files. Deliberate, not a bug.
  - Delete `TEMPLATES_ROOT` (`spec-fs.ts:410`) and `readTemplate`/`listTemplates` (`:421-431`) — zero callers repo-wide.
- **Acceptance:** existing prompts referencing `/Templates/commands/<step>.md` still resolve.

### T022 — Section-kind badges
- **Files:** `src/apps/build-studio/index.tsx`
- **Story:** FR-010b
- **Work:**
  - Render each section's `kind` as a badge on the group, reusing the existing owner-badge pattern.
  - **A section with `terminal: true` renders a terminal state instead of evaluating the DAG** (`047` FR-006b/SC-004): an archived change must not show `tasks: available` for work finished months ago. Adding the field to the type without this mechanism is the same defect shape as the second review's MF-1.
  - Generic, not framework-specific — this lives here so `047` stays code-free.
- **Acceptance:** three section kinds distinguishable (supports SC-013's e2e). **Stage B revert point.**

---

### T028 — `/api/specs` returns the descriptor
- **Files:** `src/app/api/specs/route.ts`
- **Story:** FR-014
- **Work:** `GET` returns the active descriptor alongside `tree`/`specs`, so the client never re-derives phase metadata.
- **Acceptance:** T010's PhaseStrip has everything it needs from one call.

### T029 — Global default setting
- **Files:** `src/lib/config/registry.ts`, `src/components/apps/settings/BuildStudioTab.tsx`
- **Story:** FR-008
- **Work:** build-studio namespace (`:96-127`) gains `defaultMethod` beside `agent`/`conflictAgent`. **`fields` is required even with `customComponent`** — `/api/config`'s PATCH coerces against it and silently drops anything undeclared. Wording states the default is per-user.
- **Acceptance:** the default persists and applies to new stores.

---

## PHASE 3 — Stage C: install path

### T023 — `method` facet
- **Files:** `src/system/items/installed.ts`, `src/lib/marketplace/schema.ts`
- **Story:** FR-012
- **Work:** `ItemFacets` (`:66-75`) + `readFacets` (`:99-111`) gain `method`; `MarketplaceItem` gains the facet. Discovery through the existing depth-2 scan — nothing is copied (035).
- **Acceptance:** an item with a `method/` facet is discovered as one.

### T024 — Install/uninstall registration + origin gate
- **Files:** `src/lib/specs/method/install.ts` (new), `src/lib/marketplace/client.ts`
- **Story:** FR-012, FR-013a
- **Work:**
  - **`installMarketplaceItem` (`client.ts:880-881`) throws `has nothing installable` for any item without app/services/integration/voiceEngine** — a method-only pack is refused server-side even if the UI issues the op. `uninstallMarketplaceItem` (`:1032-1051`) dispatches on `facets.plugin/service/app` with no `method` branch. Both need one.
  - Install registers descriptor (T005), agent roots (T012) and template mount (T021). Uninstall unregisters all three — this is why T002 exists.
  - **Origin gate:** a pack with a `plugin/` facet installs freely from `user-apps` (origin derived via `deriveOrigin`/`itemProvenanceKey`, `installed.ts:118-128`/`:56-64`). From any other marketplace it requires an explicit **per-pack** opt-in, recorded against that pack — not a global setting, not a dismissible warning.
- **Acceptance:** SC-016 — same pack installs without opt-in from `user-apps`, refused until opt-in from elsewhere.

### T025 — Marketplace UI
- **Files:** `src/apps/marketplace/index.tsx`
- **Story:** FR-012a, FR-013
- **Work:**
  - `:383` — `hasInstallableFacet` must include `method`, or **no Install button renders at all**.
  - `:173` — the install-op condition must fire for a `method` facet, or no install is issued.
  - `:169-187` — `installedKinds` gains `"method"` for the post-install notice.
  - FR-013's `plugin/`-facet warning renders at install.
  - **Note:** FR-012a's "kind list / source-sidebar filter" describes machinery that does not exist — the sidebar (`:262-284`) groups by *source repo*, and per-source sections already list every item regardless of facet. Routed to `analyze`; implement the two real blockers.
- **Acceptance:** a method-only pack is installable and uninstallable from the UI.

### T026 — "Method not installed" state
- **Files:** `src/apps/build-studio/index.tsx`, `src/lib/assistant/tools/server/subagents.ts` (`:115`, `:123`), `src/lib/assistant/tools/server/delegate-common.ts` (`:30`), `src/lib/marketplace/client.ts` (uninstall warning)
- **Story:** FR-016
- **Work:** A store referencing a missing method renders a clear state and **never silently falls back to spec-kit** and reinterprets content. Delegation to an agent from an uninstalled pack fails naming the pack. Uninstall warns when any store references the pack.
- **Acceptance:** SC-011 — uninstall warns while a store still references the pack; afterwards the store shows "method not installed" and a delegation to a removed agent fails naming it.

### T027 — Item-store artifact naming
- **Files:** `src/lib/specs/create.ts`, `src/lib/specs/item-stores.ts`
- **Story:** FR-015
- **Work:** `create.ts:26` (`fs.access(… "spec","spec.md")`), `create.ts:123` (`installItem({ files: { "spec/spec.md": … } })`) and `item-stores.ts:67`/`:80` resolve to the descriptor's first artifact rather than the literal.
- **Acceptance:** an item store under a non-spec-kit method uses that method's first artifact. **Stage C revert point.**

## PHASE 4 — Verification, docs, gates

### T031 — E2E coverage
- **Files:** `e2e/045-method-packs.spec.ts` (new); update `e2e/013-artifact-reload.spec.ts`, `e2e/013-spec-anchor.spec.ts`; convert `e2e/build-studio.spec.ts:49-56`
- **Story:** FR-017, SC-013
- **Work:**
  - New: phase strip incl. `blocked`, Method dropdown, preflight, section badges.
  - Update the two existing specs covering this surface rather than letting them rot.
  - Convert `build-studio.spec.ts`'s `test.skip(true, "no marketplace item installed in this environment")` onto T002a's fixture — a test that passes by skipping is the pre-seeded-data dependency BOS's testing rule prohibits.
  - Every new test bundles its own fixtures.
- **Acceptance:** SC-013; no test in the touched set skips for want of environment state.

### T032 — Documentation *(lands per-stage, not at the end)*
- **Files:** `docs/dev/build-studio.md`, `docs/dev/architecture-overview.md` §9, `docs/dev/method-packs.md` (new)
- **Story:** FR-018, Constitution VI (NON-NEGOTIABLE)
- **Sequencing:** `design.md` §2 rules Principle VI compliant **conditional on FR-018 landing with the stages**. Listing it in PHASE 4 contradicted that. Each stage updates the docs it invalidates: Stage A the phase/descriptor prose, Stage B binding and mounts, Stage C install and marketplace. This entry is the checklist, not a deferred task.
- **Work:**
  - `build-studio.md` describes `PHASE_ORDER`, the single `/Templates` mount and the engine's location in source — all wrong after Stage A.
  - `architecture-overview.md` §9 likewise.
  - New pack-authoring reference: descriptor shape, `schemaVersion`, sections, DAG phases, role attachment.
  - **FR-010's documentation clause:** state that the recommended adoption path is a new store or new Project, **not** flipping an existing one.
- **Acceptance:** SC-014 — no statement in either existing doc contradicts shipped behaviour.

### T033 — Gates
- **Files:** —
- **Story:** SC-017
- **Work:** `npx tsc --noEmit`; `npm run lint`; `npm run test:unit` (**never** bare `npx playwright test -c playwright.unit.config.ts`). Re-run T001 parity a final time.
- **Acceptance:** all green; each of Phases 1–3 independently revertible with a green unit suite (SC-009).

---

## Dependency summary

Corrected after the second review, which found the first graph had a missing `T008 → T010` edge, drew T007/T008 as separable when they are not, and stated a "critical path" that was not a path in its own graph.

```
T001 ──────────────────────────────────────────► gates T008, T011, T033
T002 ──────────────────────────────────────────► gates T021, T024
T002a ─────────────────────────────────────────► gates T018, T024, T031

T003 ─┬─► T004 ──► T006 ──► T006a ─┐
      └─► T005 ──────────────┘     │
                                   ├─► T008 ─┬─► T009
      T007 ─────────────────────────┘        └─► T010 ─► T011   [Stage A revert]

T011 ─┬─► T012 ─► T013 ─► T014
      ├─► T015 ─┐
      ├─► T016 ─┼─► T017 ─┬─► T018 ─► T020 ─► T022                [Stage B revert]
      ├─► T029 ─┘         └─► T019 ─┘
      └─► T021
      └─► T028 ─────────────────────► T020, T022

T022 ─► T023 ─┐
T002/T002a ───┼─► T024 ─┬─► T025 ─► T026
T012, T021 ───┘         └─► T027
T013 ──────────────────────────► T026
                                                                  [Stage C revert]
T026 ─► T031 ─► T033        (T032 lands per-stage, not here)
```

**Four corrections from the review:**

1. **`T008 → T010` added.** PhaseStrip can only render `{id, label, state}` once `derivePhases` emits `label`. The first graph gated T010 on T007 alone.
2. **T007 and T008 are no longer drawn as parallel.** They are not separable: T007 deletes `ARTIFACT_FILES`, whose only consumer is `byArtifactOrder` (`pipeline.ts:188`), and `PhaseId` is used only by `types.ts`/`pipeline.ts`. Splitting them leaves `tsc` red.
3. **`T028 → T020, T022` added, and T028/T029 pulled forward into Stage B** (SI-2, SI-3). T020 lists installed packs and T022 renders section kinds, but the descriptor only reaches the client via FR-014 (T028); and T017's resolution chain ends at "global default", which has no backing store until T029. Both were stranded in Phase 3 behind consumers in Phase 2. `spec.md`'s staging table has the same inversion and is corrected with them.
4. **`T002a` is a Phase 0 prerequisite**, not Phase 4 — T018 and T024 consume it (MF-7).

**Stage A descriptor acquisition (SI-4):** with `resolve.ts` in Stage B, Stage A obtains its descriptor from T005's registry directly — the single registered built-in — via T006a's boot registration. No `builtin` shortcut, which SC-002 forbids and T006's acceptance greps for.

**Parallelisable within a phase:** T004/T005 (after T003); T012/T015/T016/T021 (different files, no shared state); T025/T027 (unrelated surfaces).

**Critical path** — verified to be an actual path through the graph above:

`T001 → T008` (gate) and `T003 → T004 → T006 → T006a → T008 → T010 → T011 → T015 → T017 → T018 → T020 → T022 → T023 → T024 → T025 → T026 → T031 → T033`

---

## Phase 6 — durability guards (T034 … T041)

Not optional hardening. Without these the layer passes its own test suite while
several of its requirements are unreachable in the product — each guards a seam
where the obvious implementation fails silently (`design.md` §9).

- **T034 — Manifest round-trip guard (FR-019).**
  For each manifest BOS rewrites (`spec-store.json`, `project.json`,
  `marketplace.json`, a pack's `method.json`): write a key the parser does not
  model, trigger a BOS-initiated rewrite, assert the key survives.
  *Acceptance:* the test fails when any reader is changed to reconstruct a fixed
  object. Audit every reader for reconstruct-then-return and convert to
  spread-then-narrow — there are four.

- **T035 — Restart-boundary test for the binding (FR-020, SC-018).**
  Assign a method, re-run the seed as a fresh boot would, assert the binding is
  still there. MUST NOT assert in the same process state that made the write —
  that is what passed while the seed was overwriting the file on every start.

- **T036 — Descriptor-memo lifetime (FR-021, SC-021).**
  Enumerate every public entry point in the pipeline; each MUST reset the memo.
  Add a test that rebinds a store to a method with different numbering IN THE
  SAME PROCESS and asserts the next allocated id follows the new method.
  Write the invariant at the cache declaration.
  *Note:* the absence of this surfaces as an unrelated test "flaking". Treat a
  numbering or discovery test that fails only in a full run as this bug until
  proven otherwise.

- **T037 — Write paths for every binding level (FR-022).**
  `set-method` accepts a Project and writes `project.json`; item stores read and
  write a binding in the item's own `spec/spec-store.json`. Scope the preflight
  to the level being bound, and skip the constitution reconcile at Project scope.
  *Acceptance:* an unknown Project id is REFUSED, not written.

- **T038 — Per-node method in the tree (FR-023).**
  Each group/project node carries its own resolved method id and whether the
  binding is declared there or inherited. The picker reads the node, never an
  app-level "active method". A group node MUST agree with its own children —
  including item stores, whose row is a synthetic child.

- **T039 — Section-aware discovery (FR-025).**
  `listSpecifications` and `specTree` walk sections, not Projects only. One
  function decides section ownership by longest prefix. Units in a `terminal`
  section return a terminal state without evaluating the DAG.
  *Acceptance:* a fixture store with `changes/`, `specs/` and `changes/archive/`
  discovers each unit exactly once, and archived units carry no live state.

- **T040 — Method-aware Project migration (FR-026).**
  The `033` migration consults the store's binding, read off disk, before moving
  anything. A declared-but-unregistered method blocks it too.
  *Acceptance:* a store bound to a root-owning method keeps its layout across a
  boot that runs the migration.

- **T041 — A live smoke check that crosses the boundaries unit tests cannot.**
  A unit suite cannot catch any of the above, however thorough. Each lives at a
  boundary a single-process test cannot cross —
  install → restart, one module instance vs another, parse → rewrite, seed
  memoisation, descriptor → user-facing message. Add a script that exercises the
  layer against a RUNNING BOS and exits non-zero, and give it a staleness guard
  so "the server predates this check" is never reported as a product defect.

- **T042 — Glob provider (FR-027, SC-022).** The pipeline supplies `glob` to the
  evaluator, over the unit's files INCLUDING nested ones. Share one
  glob-matching function between provider and fallback.
  *Acceptance:* a phase gated on `dir/*.md` reaches `done` when those files
  exist. A test that only asserts the empty case is NOT acceptance — it passes
  whether the glob works or cannot match at all.
- **T043 — Frozen sections are read-only (FR-028, SC-023).** `assertEditablePath`
  at the API's write/delete/rename and the spec-writing tool; rename checks BOTH
  ends. `terminal` resolves for every node, not just section roots.
  *Acceptance:* active work in the same store stays editable — the guard is per
  section, never a store-wide read-only flag.
