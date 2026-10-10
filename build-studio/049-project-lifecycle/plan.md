# Implementation Plan: Project lifecycle (049)

## Summary

Make a project creatable, renameable and deletable — from the assistant and from
the tree — with its workflow chosen at creation rather than after. Three tools,
one implementation, dispatched on repository kind.

## Technical context

| | |
|---|---|
| Depends on | `045` (descriptor, binding chain, install) |
| Anticipates | `051` — binds by WORKFLOW from the start so the signature is not reworked |
| Independent of | `050` |
| Touches | the method layer (additively), `create.ts`, the specs route, the assistant registry, Build Studio's tree |

## Constitution check

- **Server decides, client displays** — dispatch and refusals are server-side.
- **No workarounds** — item rename/delete are refused with their reason
  (`design.md` §3.4), not approximated.
- **Docs track features** — FR-014. The current answer to "how do I create a
  project" is "hand-write `project.json`", which is not discoverable.

## Test strategy

Three rules carried from `045`–`048`, each earning its place here:

1. **Drive the product's call path.** The lifecycle tests call `lifecycle.ts`,
   which is what both the tools and (next) the menus call — not the helpers
   underneath it.
2. **Assert the round trip, not the write.** The rename test writes an
   unmodelled manifest key and requires it to survive. That is what caught
   `readProjectManifest` still reconstructing — the fifth occurrence of that
   trap in this subsystem.
3. **Assert the artifact, not the record.** FR-012's test checks the file NAME
   on disk: "the binding was recorded" can be true while the wrong file is
   written.

## Phase breakdown

| Phase | What | State |
|---|---|---|
| 1 | Workflows, store kind, lifecycle service, FR-012 | **done** — `085389c` |
| 2 | API ops + the three agent tools | **done** — `a3dc6a7` |
| 3 | Build Studio context menus, on the same implementation | remaining |
| 4 | Gate the per-Project picker by binding scope; report a stray binding | remaining |
| 5 | Docs | remaining |
| 6 | Live verification | remaining |

## Rollout

Additive. No existing binding changes meaning, no manifest is rewritten on
upgrade, and `method`-keyed bindings keep resolving — verified by SC-007 rather
than assumed.

Phase 4 REMOVES a control (the per-Project picker in `user-specs`) that could
never take effect there. Zero projects carry such a binding today, so nothing
regresses; the removal is what stops the UI advertising a choice the product
does not honour.

## Risks

- **R1 — the tree's context menus already vary by node type.** Phase 3 adds a
  third axis (store kind). This is the thinnest part of the spec and the most
  likely to reveal an assumption; enumerate the node types before editing.
- **R2 — `ProjectScaffold` has no real consumer.** A descriptor field with no
  consumer is exactly how `sections` and `terminal` came to be half-honoured
  (`045` D-4). `051` is its first; until then it is fixture-only, and that
  should be stated rather than assumed working.
- **R3 — arbitrary repos.** The verbs may not make sense there. Deferred to
  `050` deliberately, not forgotten.
- **R4 — nothing is verified live.** Every real defect in this feature family
  was found by running it, not by testing it.
