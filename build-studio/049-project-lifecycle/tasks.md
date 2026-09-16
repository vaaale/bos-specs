# Tasks: Project lifecycle (049)

## Phase 1 — the model ✅

- **T001 — `WorkflowSpec` + `ProjectScaffold` in the descriptor.** ✅
  Scaffold is DATA, never a pack-registered handler: `045`'s premise is that a
  framework is a marketplace item rather than a code change, and FR-013a already
  gates marketplace code behind a per-origin opt-in.
- **T002 — `workflows.ts`: enumeration and resolution.** ✅
  *Acceptance:* a method declaring none has exactly one, named for itself — the
  compatible case is the default, not a shim. Unknown → refused with the list.
  Ambiguous → reported with both providers. A bare method id beats another
  method's workflow of the same name. 7 tests.
- **T003 — `store-kind.ts`: `kindOf` / `bindingScopeOf` / `projectsAre`.** ✅
  Kept as separate functions on purpose (`design.md` §3.2): every time binding
  scope and lifecycle unit were one thing, one of the two answers came out wrong.
- **T004 — rename / delete / unit count in `projects.ts`.** ✅
  Rename moves the directory AND the label — two halves of one identity.
- **T005 — `createProject` takes a branch ctx (FR-013).** ✅
- **T006 — FR-012: `createItemSpec` resolves the FULL chain.** ✅
  *Acceptance:* asserted by the artifact's NAME on disk. "The binding was
  recorded" was true the whole time the wrong file was being written.

### Guards the rename and create paths need

- **T007 — `readProjectManifest` must SPREAD, not reconstruct.** ✅
  BOS rewrites `project.json` on rename, so an unmodelled key is not merely
  ignored on read — it is deleted from the user's repo and committed
  (`045` §9.1).
- **T008 — `assertEditablePath` must reset the descriptor memo.** ✅
  Every public pipeline entry point must (`045` §9.2). Missing it surfaces as an
  unrelated test failing intermittently.

## Phase 2 — reachable ✅

- **T009 — `lifecycle.ts`: kind dispatch.** ✅ Item / folder / refused.
  A workflow passed to a store-scoped kind is refused naming the store's own
  binding. Item rename and delete are refused with their reason (§3.4).
- **T010 — API ops + the three tools.** ✅
  Lifecycle ops handled BEFORE the method validation in the route — they carry
  no `method` and below it would have failed with `Method "" is not installed`.
  `delete_project` elicits in the TOOL, so an agent cannot skip the confirmation
  by not asking; the route refuses an unconfirmed POST independently.

## Phase 3 — the tree ✅

- **T011 — Enumerate the tree's existing context menus FIRST.** ✅
  Two findings: only FILE nodes carry a menu today, and `listStores()` yields no
  store for the marketplace REPO — only `item-<id>` stores. So the headline case
  has no id to address and no row to right-click. Synthesised in ONE place;
  `050` re-models it. They already
  vary by node type; this adds a third axis (store kind). `plan.md` R1 — the
  thinnest part of the spec.
- **T012 — Store row → "New project".** ✅ Offers a workflow choice ONLY where
  binding scope is `project`. Absent on the read-only system store.
  *Acceptance:* calls `lifecycle.ts` — the same path the tools take, not a
  parallel implementation (SC-005).
- **T013 — Project row → Rename / Delete.** ✅ Delete shows the unit count from
  `describeProjectDeletion` before asking.
  *Acceptance:* a folder renames and deletes; an ITEM row offers neither, since
  both are refused server-side (§3.4) and an action that always errors is worse
  than no action.

## Phase 4 — stop offering what cannot work ✅

- **T014 — Gate the per-Project picker on `bindingScopeOf(store) === "project"`.** ✅
  The tree now carries `bindingScope` / `projectUnit`, server-computed.
  It currently renders in `user-specs`, where a per-project binding is not
  honoured. Zero projects carry one today, so nothing regresses.
- **T015 — Report a stray per-Project binding (FR-011b).** ✅
  `045` FR-008's chain applies per kind, so a binding at a level that does not
  apply is read and reported rather than honoured or dropped. Present but not
  honoured must be REPORTED — silently honouring or silently ignoring both leave
  a binding the user can neither see nor act on.

## Phase 5 — docs ✅

- **T016 — How to create a project with a chosen workflow** ✅, per kind, and what
  `Settings → Default method` now affects. State plainly that item rename and
  delete are not supported and why.

## Phase 6 — verification (REMAINING)

- **T017 — Run it against a live BOS.** Create a folder in `user-specs`; create
  an item in `user-apps` under a non-default workflow and confirm the artifact's
  NAME; rename and delete a folder.
  *Why:* every real defect in this feature family was found by running it.

## Dependencies

```
T001 → T002 → T009 → T010 → T012, T013
T003 → T009, T014
T006 → T017 (the fix worth verifying first)
T011 → T012, T013
```

## Critical path

`T011 → T012 → T013 → T017`

T014/T015 are independent of Phase 3 and can land either side of it.
