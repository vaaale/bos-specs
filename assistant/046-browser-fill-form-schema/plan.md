# Implementation Plan: Browser Fill Form Schema Fix

**Branch**: `bos/self-heal-0044` | **Date**: 2026-09-24 | **Spec**: `/Specs/user-specs/assistant/046-browser-fill-form-schema/spec.md`

**Input**: Feature specification from `/Specs/user-specs/assistant/046-browser-fill-form-schema/spec.md` (self-heal case EHS-0044, scope class e / bos-core).

## Summary

The `browser_fill_form` registry tool declares its per-field element key as `ref` and requires `["name","type","ref","value"]`, but it is a pure passthrough to the pinned `@playwright/mcp` server, whose real `browser_fill_form` contract validates the per-field key as `target`. An agent that follows BOS's own declared schema therefore sends `ref`, the proxied server rejects it (`expected string, received undefined at fields[*].target`), and the call can never succeed. The fix renames the declared per-field key `ref` → `target` (the property and the item `required` array) so the declaration matches the proxied implementation, matching the rest of the `browser_*` element-tool family that already names the key `target`. No executor change — the passthrough forwards the key as-is.

## Technical Context

**Language/Version**: TypeScript (BOS, Next.js App Router, server-only modules)
**Primary Dependencies**: pinned `@playwright/mcp` (`^0.0.76`, the proxied MCP server); `@playwright/test` for the unit suite
**Storage**: N/A (in-memory tool registry)
**Testing**: `@playwright/test` unit config — `npx playwright test -c playwright.unit.config.ts tests/assistant/browser-tools.test.ts`
**Target Platform**: BOS server (Node)
**Project Type**: server library module (the browser-tools declaration) + its unit test
**Performance Goals**: N/A (no runtime behavior change)
**Constraints**: one-file declared-schema change; TDD (failing test first); ≥60% line & branch coverage on the modified file; must not touch Supervisor or build config
**Scale/Scope**: 1 source file, 1 test file; the `browser_fill_form` `serverTool` declaration only

## Constitution Check

*GATE: Must pass before research. Re-checked post-design.*

| # | Principle | Status | Note |
|---|---|---|---|
| I | Spec-Driven (SAP) | PASS | `spec.md` authored and agreed before this plan; code is a derivative. |
| II | Server Authority / SSR Boundary | PASS | Change is in a `server-only` module (`src/lib/assistant/tools/server/browser.ts`); no client/secret impact. |
| III | Always Delegate; Claude Codes | PASS | `implement` is delegated to the Developer sub-agent via `dev_delegate`. |
| IV | Minimize Blast Radius | PASS | Change made on feature branch `bos/self-heal-0044`; isolated worktree. |
| V | VFS Is Not the Source | PASS | BOS source edited only via the developer worktree, never through VFS `file_*` tools. |
| VI | Specs & Docs Stay in Sync | PASS | Spec updated in-tree. Docs see "Documentation" below — no docs edit is *required* (the docs never stated the per-field parameter name), and this reasoning is recorded here rather than as a silent drift. |
| VII | Respect Boundaries | PASS | No changes to `package.json`, lockfiles, build config, or secrets; `npx tsc --noEmit` + `npm run lint` must pass. |

**Gate result**: PASS. No Complexity Tracking entries required (no principle violated).

## Documentation (Constitution VI)

The relevant developer page is `docs/dev/automation/browser-automation.md`. Its "Interaction model" section states element targeting uses `@playwright/mcp`'s snapshots with refs like `[ref=e12]` that "click/type target" — here `ref` is correctly used as the **snapshot display token**, not as a `browser_fill_form` parameter name. A `docs/` sweep confirms no page documents the `browser_fill_form` per-field keys at all (`browser_fill_form` appears only by name in the tool list at line 37).

**Conclusion**: the docs are not wrong as written; the defect is a latent schema bug the documentation never captured at parameter granularity. Therefore **no documentation change is required** for this fix, and no entry is needed in `specs/discrepancies.md` (code did not drift from spec — the spec, once corrected, matches the dependency's real contract). If the Developer surfaces any docs that *do* name `ref` as a parameter during implementation, that page must be corrected in the same change (FR-006 / Constitution VI).

## Project Structure

### Source Code (repository root) — files this change touches

```text
src/
└── lib/assistant/tools/server/
    └── browser.ts                       # MODIFY — browser_fill_form declared schema: ref → target (property + required)

tests/
└── assistant/
    └── browser-tools.test.ts            # MODIFY — add regression test(s) for the browser_fill_form per-field schema
```

**Structure Decision**: Minimal-surface, single-module change. The fix is confined to the `browser_fill_form` `serverTool` declaration in `browser.ts` (the one element tool that deviates from the shared `el`/`target` convention) plus the existing test file for this tool family. No new files, no new directories.

### Why the fix is only the rename (not a restructure)

The other element tools share a module-level `el` object `{ element, target }` and spread it. `browser_fill_form` does **not** spread `el` because its per-field human-readable key is `name` (a field name), whereas `el.element` is an *element description* — reusing `el` would mislabel the field's name as an element. So the per-field object is intentionally hand-written; the minimal correct fix is renaming its `ref` property to `target` and updating the item `required` array to `["name","type","target","value"]`. (Spec A-003.)

## Design Notes

- **Executor unchanged** (FR-005): `browser_fill_form`'s executor is `(input, ctx) => proxy("browser_fill_form", input, ctx)`; `proxy` → `callBrowserTool` forwards `input` to the MCP server verbatim (`session.client.callTool({ name, arguments })`). There is no `ref`→`target` translation anywhere in BOS, so the declared key reaches the server exactly as named. Once the declaration says `target`, the proxied `@playwright/mcp` server validates it — the same success path the `browser_type` fallback already exercised.
- **`target` description** (FR-004): align the per-field `target` description with the family's shared wording — "Exact element reference from the latest browser_snapshot (e.g. \"e12\"), or a unique selector" — so the model reads one consistent contract across the tool family.
- **Regression test** (FR-006, TDD): assert against the *declared* schema, which `serverTool` exposes at `.parameters` (a `schema()`-wrapped object). The assertion target is `browserTools().browser_fill_form.parameters.properties.fields.items` — its `properties` must contain `target` and not `ref`, and its `required` must be `["name","type","target","value"]`. This test fails on the pre-fix code (which declares `ref`) and passes post-fix — it proves the contract the model is shown, not the executor.
- **Coverage** (FR-006 / NFR): the modified file is `browser.ts`; the new test(s) exercise the `browser_fill_form` declaration. The file is largely declarative (data objects + passthrough executors), so the existing suite already carries line/branch coverage; the Developer must verify ≥60% line & branch on `browser.ts` after the change and add targeted assertions for any uncovered branch (e.g. the `FOLLOW_WITH_SNAPSHOT` follow-up for `browser_fill_form`, already covered by the composition test pattern in the file).

## Classification Verification (FR-015a)

Compared this plan's file list against the Diagnostician's `proposedSurface`
(`src/lib/assistant/tools/server/browser.ts` — the `browser_fill_form` declared
schema) before writing tasks:

- **Source change** = `src/lib/assistant/tools/server/browser.ts` — the *exact*
  proposed surface (same top-level `src/` dir, same file, same function).
- **Second file** = `tests/assistant/browser-tools.test.ts` — the test tree, not
  a subsystem. It is none of the rule's divergence triggers (a different
  top-level `src/` dir, a different `seed/agents/<id>`, a different
  `src/apps/<id>`). It is the TDD regression test + coverage mandated by the
  FR-014 brief, and it exercises the file in the proposed surface.

**Result: NO DIVERGENCE.** The plan touches only the proposed surface plus its
brief-mandated test companion. No re-diagnosis required; proceeding to tasks.

## Verification (run at implement)

- `npx tsc --noEmit` — clean
- `npm run lint` — clean
- `npx playwright test -c playwright.unit.config.ts tests/assistant/browser-tools.test.ts` — green, including the new `browser_fill_form` schema regression test(s)
- Regression test proven to fail on the unfixed base before the fix is applied (TDD)
