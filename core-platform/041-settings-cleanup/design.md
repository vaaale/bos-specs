# Design: Settings cleanup (041-settings-cleanup)

**Feature**: Remove dead System Tools page · remove "Knowledge bases" agent tab · sort Settings modules alphabetically
**App Target**: `bos-core`
**Spec**: `spec.md` (this directory) · **Mockup**: `mockup.html` (this directory)

This design is a *deletions + one re-sort* change. It touches no runtime data flow, no new subsystem, and no protocol/daemon — so there are no C4 container additions and no service concerns. The whole risk surface is: (1) not deleting something that is *not* actually orphaned, and (2) not breaking the one behavioural change (the sidebar sort) that is anything other than a pure removal.

## Classification

- **Deletions (A, B):** remove code whose sole consumer is removed. Safe iff the orphan set is closed (verified below). No API contract, data model, or runtime behaviour survives that should be deleted.
- **Re-order (C):** change the sort key in one server function that feeds the Settings sidebar. Safe iff nothing else depends on the current `order`-based sequence (verified: it doesn't).
- **Intentionally inert (A.4):** `data/config/system-tools.json` is left in place — a file with no reader. No migration hook is introduced.

## Verified orphan set (repo-wide, `src/`)

Confirmed against live source at spec time:

**A. System Tools**
- `system-tools` namespace: defined **only** in its own registration (`src/lib/config/registry.ts`, the `REGISTRATIONS` entry `order: 37`) and read **only** by `POST /api/system/bash` (`src/app/api/system/bash/route.ts` line 13 via `getConfigValue("system-tools","enabled")`). The registration's `load`/`save` reference `readNamespace`/`patchNamespace` — generic helpers used by every other namespace, so they are **not** orphaned. No Settings tab supplies a `customComponent` for `system-tools`, so it renders generically from its single `fields` entry → removing the registration removes the page.
- `@/lib/system/bash` (`runBash`, `DEFAULT_TIMEOUT_MS`): imported **only** by the bash route.
- `BashResult`, `MAX_TIMEOUT_MS`, private `truncate`: defined in `src/lib/system/bash.ts`, referenced **nowhere else**. The lib's own imports are stdlib (`node:child_process`, `node:path`), so deleting it orphans nothing downstream.
- `/api/system/bash`: no client references the endpoint path.
- **Closed.** The `system-tools` string appears only in: the registration, the route (its sole reader), and a self-referential comment inside the lib (removed with the file).

**B. Knowledge bases tab**
- `KnowledgeBasesGrid` component: imported **only** by `AgentDetails.tsx` and re-exported **only** by the barrel `src/components/apps/settings/assistant/index.ts` (two lines: the component + `KnowledgeBasesGridProps`).
- In `AgentDetails.tsx`, the tab is fully self-contained: the `Tab` union includes `"kb"`, one `<TabButton …>Knowledge bases</TabButton>`, and the `{tab === "kb" && (<KnowledgeBasesGrid …/>)}` block.
- **What is NOT orphaned (stays):** the `CatalogKb` / `Catalog.kbs` / `AgentMeta.kbs` data model in `types.ts`, the agent API's `catalog.kbs` payload, `src/lib/agent/kb-catalog.ts`, `buildKbIndexBlock` in `src/lib/agent/instructions.ts`, and the per-agent `kbs` allowlist semantics. These are the Knowledge Base *item's* runtime, not the Settings UI.

**C. Sidebar sort**
- `listConfigSchemas()` (`src/lib/config/registry.ts` line 942) is the single point where built-in `REGISTRATIONS.map(r => r.schema)` and `listSettingsPanels().map(pluginSchemaFor)` are merged, then sorted by `(a.order ?? 100) - (b.order ?? 100)`.
- The Settings sidebar renders the `schemas` array **in the order it arrives** from `GET /api/config`; no client Settings component re-sorts the module list. (The `sort` calls in `McpServersTab` / `RunCommandTab` / `LogsTab` are internal to their own content lists, not the sidebar.)
- Neither the assistant's `config_*` tools (`src/lib/assistant/tools/server/config.ts` iterates `listConfigSchemas()` for discovery, not for ordering) nor the `/api/config` PATCH path depend on module order.
- **Conclusion:** changing the sort key in `listConfigSchemas()` alone reorders the sidebar, for built-ins **and** plugin panels together.

## Component / change plan (only what this feature touches)

### A — System Tools
1. **Modify** `src/lib/config/registry.ts` — delete the entire `system-tools` `ConfigRegistration` block from the `REGISTRATIONS` array (the object with `namespace: "system-tools"`, `order: 37`).
2. **Delete** `src/app/api/system/bash/route.ts`.
3. **Delete** `src/lib/system/bash.ts`.
4. **No change** to `data/config/` — the persisted file is left inert.

### B — Knowledge bases tab
1. **Modify** `src/components/apps/settings/assistant/AgentDetails.tsx`:
   - remove `"kb"` from the `Tab` union (→ `"instructions" | "skills" | "tools" | "mcp"`);
   - delete `import { KnowledgeBasesGrid } from "./KnowledgeBasesGrid";`;
   - delete `<TabButton active={tab === "kb"} onClick={() => setTab("kb")}>Knowledge bases</TabButton>`;
   - delete the `{tab === "kb" && (<KnowledgeBasesGrid … />)}` render block;
   - update the doc comment that says "five sub-tabs (… / Knowledge bases)" to four.
2. **Delete** `src/components/apps/settings/assistant/KnowledgeBasesGrid.tsx`.
3. **Modify** `src/components/apps/settings/assistant/index.ts` — remove the two `KnowledgeBasesGrid` / `KnowledgeBasesGridProps` re-exports.

### C — Alphabetical modules
1. **Modify** `src/lib/config/registry.ts` → `listConfigSchemas()`: replace the `order`-based comparator with a title-based, then-namespace comparator (see ADR-1).

**Integration points (existing BOS mechanisms this merely calls into, not re-implements):** the generic config `load`/`save` (`readNamespace`/`patchNamespace`), `getRegistration` (falls through to `undefined` for `system-tools` after removal — no plugin path, so it simply returns `undefined`), the `GET /api/config` route, and the Settings sidebar renderer. None are modified.

## ADRs

### ADR-1 — Sort comparator for the Settings sidebar
**Context:** FR-011/FR-012/FR-013 require A→Z by title, stable for equal titles, plugin panels interleaved, with no client change.
**Decision:** In `listConfigSchemas()`, replace
```ts
.sort((a, b) => (a.order ?? 100) - (b.order ?? 100))
```
with
```ts
.sort((a, b) =>
  a.title.localeCompare(b.title, undefined, { sensitivity: "base" })
  || a.namespace.localeCompare(b.namespace, undefined, { sensitivity: "base" }))
```
- `sensitivity: "base"` gives case-insensitive, accent-insensitive ordering ("AI Provider" < "Agent Runs" < "Agents"… note `localeCompare`'s base sensitivity still orders by full character sequence, so "Agent Runs" sorts before "Agents" — both acceptable; the tie-break by `namespace` only ever fires on genuinely equal titles).
- Tie-break on `namespace` guarantees a deterministic order for identical titles.
- The `order` field is **retained as data** but no longer the sort key (it still exists on each `ConfigSchema`; nothing breaks by leaving it).
**Alternatives rejected:** (a) sorting client-side in the Settings component — spreads the logic to two places and would have to handle plugin panels separately; (b) reassigning the numeric `order` values to be alphabetical — brittle (a new module needs a renumber, and plugin panels' `order` comes from the plugin, not from us) and would not keep itself alphabetical; (c) a `useInsertionSort`/`Intl.Collator` cache — overkill for ~21 modules, and `Array.prototype.sort` stability (guaranteed in ES2019+) plus the namespace tie-break already makes it deterministic.
**Consequence:** one-line-of-intent change in a single server function; sidebar stays alphabetical automatically as modules/plugins are added (SC-005).

### ADR-2 — Leave `catalog.kbs` and the `kbs` allowlist model in place
**Context:** after B, the frontend `Catalog.kbs`, `CatalogKb`, `AgentMeta.kbs`, the agent API's `catalog.kbs` payload, and the Knowledge Base catalog source are no longer consumed by Settings.
**Decision:** Leave them all untouched (FR-009/FR-010). The only deletion is the Settings UI (component + tab + barrel exports).
**Rationale:** pruning them would edit the Knowledge Base item's own contract (`kb-catalog.ts`, the agent API route) for zero functional gain, and risks regressing `kb_search`/`kb_retrieve` and the runtime allowlist — out of scope and a real behaviour surface. This is the minimal-diff, lowest-risk boundary.
**Consequence:** a per-agent `kbs` allowlist, if set, still round-trips and applies at runtime (unset = all KBs); there is simply no Settings UI left to edit it. Re-adding a UI later is a separate, trivial change.

### ADR-3 — Do not delete the persisted `system-tools.json`
**Context:** FR-004 (per @alex) — leave the config file as-is.
**Decision:** No migration hook, no boot-time cleanup, no lazy deletion. The file becomes inert once the registration is gone (nothing calls `readNamespace("system-tools")`).
**Rationale:** an orphaned file with no reader is harmless; a cleanup path is more code and a more failure surface than the feature warrants.
**Consequence:** `data/config/system-tools.json` may persist on existing installs indefinitely; it has no effect.

## Risks & mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| A: a consumer of `system-tools` / the bash route / lib survives → runtime 404s or broken import. | Low (verified closed) | Repo-wide search for `system-tools`, `@/lib/system/bash`, `runBash`, `BashResult`, `api/system/bash` returns no hits in `src/` after the change (FR-015); build + typecheck clean (FR-016). |
| C: an equal-title module pair reorders unpredictably / flickers between renders. | Low | Namespace tie-break (ADR-1) makes the order deterministic; `Array.prototype.sort` is stable. |
| C: a plugin panel that relied on `order`-based positioning shifts. | By design (FR-013) | Interleaving alphabetically *is* the requirement; flagged to @alex as the default in the spec's open question. |
| B: deleting `KnowledgeBasesGrid` breaks the barrel or another import. | Low (verified) | Barrel is the only other reference; typecheck catches any stray import. |
| A: removing the registration breaks `getRegistration("system-tools")` callers. | Low | Only the deleted route called it; `getRegistration` falls through to `undefined` for a non-plugin namespace, which no surviving code requests. |

## Out of scope (reaffirmed)
- `run-command` namespace, Command Execution page, `run_command` tool — unchanged.
- Knowledge Base item, `kb_search`/`kb_retrieve`, `kb-catalog.ts`, `buildKbIndexBlock`, and the `kbs` allowlist model — unchanged (UI-only removal).
- No migration of `system-tools.json` (left inert, ADR-3).
- No new UI, no new API route, no new data model.
