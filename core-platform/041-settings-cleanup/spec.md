# Feature Specification: Settings cleanup — remove dead System Tools page, remove Knowledge bases agent tab, sort Settings modules alphabetically

**Feature Branch**: `bos/041-settings-cleanup`

**Created**: 2026-10-07

**Status**: Spec

**App Target**: `bos-core` — changes to BOS source only: the config registry (`src/lib/config/registry.ts`), a Next.js API route (`src/app/api/system/bash/route.ts`), a server lib (`src/lib/system/bash.ts`), and two Settings UI components (`src/components/apps/settings/assistant/AgentDetails.tsx` + the `KnowledgeBasesGrid` component/barrel). No app facet and no background daemon; every change is either deletions or a re-order of existing Settings data. There is no raw/protocol/continuous behavior, so this is not a `marketplace-item`.

**Input**: GitLab issue #58 — "Remove unused page in Settings" — scope expanded via comment thread to three cleanups: (A) remove the System Tools Settings page and its orphaned API route + lib, (B) remove the "Knowledge bases" tab in Settings → Agents, (C) sort the Settings sidebar "modules" alphabetically. Persisted `data/config/system-tools.json` is intentionally left in place (per @alex).

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Settings no longer shows a dead "System Tools" page (Priority: P1)

A user opens Settings. The **System Tools** page — a single "Command execution enabled" toggle that gated the legacy unsandboxed `bash -lc` endpoint — no longer appears, because nothing in the product reads that toggle or calls that endpoint. Real command execution is governed by the separate **Command Execution** page (the `run_command` tool), which is untouched.

**Why this priority**: This is the core of issue #58. The page misleads users (it looks like it configures command execution when it doesn't) and duplicates the Command Execution page; removing it is the headline fix.

**Independent Test**: Open Settings → the sidebar has no "System Tools" entry and every other entry is unchanged. `GET /api/config` returns no `system-tools` namespace. `POST /api/system/bash` returns `404`. The assistant's `run_command` and the Command Execution page behave exactly as before.

**Acceptance Scenarios**:

1. **Given** a clean or existing BOS install, **when** I open Settings, **then** there is no "System Tools" entry in the module list and all other modules render identically (including the Command Execution page).
2. **Given** `system-tools.enabled` was previously `true`, **when** the change is deployed, **then** the page is gone and the stored value is inert — nothing reads it anymore; command execution is governed only by the Command Execution page.
3. **Given** any code path, **when** `POST /api/system/bash` is called, **then** it returns `404` (route deleted).
4. **Given** the assistant runs a command, **when** it uses `run_command`, **then** its disabled-state message still points to "Settings → Command Execution" (unchanged).

---

### User Story 2 — Per-agent editor no longer has a "Knowledge bases" tab (Priority: P2)

A user opens Settings → Agents and selects an agent. The right-hand editor shows tabs **Instructions / Skills / Tools / MCP** only — the **Knowledge bases** tab is gone. The Knowledge Base item itself (and its `kb_search` / `kb_retrieve` MCP tools) still works; only the Settings UI for editing a per-agent KB allowlist is removed.

**Why this priority**: This is a follow-on cleanup @alex requested in the same pass ("since we are here"). It has no live-product behavior change, so it is secondary to the System Tools removal but still part of the agreed scope.

**Independent Test**: Settings → Agents → any agent shows exactly four tabs (Instructions / Skills / Tools / MCP) — no "Knowledge bases" tab. The Knowledge Base item still functions (`kb_search` / `kb_retrieve` reachable). Build/typecheck is clean with no dangling references to the removed component.

**Acceptance Scenarios**:

1. **Given** any agent in Settings → Agents, **when** I open the right-hand editor, **then** the tabs are Instructions / Skills / Tools / MCP and there is no "Knowledge bases" tab.
2. **Given** the Knowledge Base item installed, **when** the assistant calls `kb_search` / `kb_retrieve`, **then** it works as before (the item and its tools are unchanged).
3. **Given** a per-agent `kbs` allowlist that was set, **when** the tab is removed, **then** the allowlist data still applies at runtime (unset = all KBs) — only the Settings UI to edit it is gone.

---

### User Story 3 — Settings modules are listed alphabetically (Priority: P3)

A user opens Settings. The sidebar "modules" are listed A→Z by display title, instead of in the arbitrary numeric `order` they currently render in. This holds for built-in modules **and** plugin-contributed panels, and it stays correct automatically as modules are added later.

**Why this priority**: The least urgent of the three — a presentation change with zero behavior change. It is P3 because it is a pure reorder and the two removals (US1, US2) are the substance of the issue.

**Independent Test**: Open Settings → the sidebar is A→Z by title. Add a hypothetical module (or install a plugin that contributes one) → it slots into the alphabetical position without any further change to the sort logic.

**Acceptance Scenarios**:

1. **Given** the built-in Settings modules, **when** I open Settings, **then** the sidebar lists them A→Z by display title (case-insensitive).
2. **Given** a plugin-contributed Settings panel, **when** Settings renders, **then** it appears interleaved alphabetically with the built-ins by title.
3. **Given** a newly added Settings module, **when** Settings renders, **then** it appears in its alphabetical position with no additional code change.

---

### Edge Cases

- **Titles that sort identically** (two modules with the same display title): break the tie deterministically by namespace so the order is stable across renders.
- **A module with no `order` field (undefined)**: previously treated as `order ?? 100`; after the change it simply sorts by title/namespace like any other module.
- **Removing the `system-tools` registration while `data/config/system-tools.json` exists**: the file is left in place and inert — no reader, no crash, no migration.
- **Removing the KB tab while an agent has a non-empty `kbs` allowlist**: the allowlist still round-trips through the agent API and still applies at runtime; only the UI affordance is gone. The `catalog.kbs` payload from the agent API is left as-is (harmless, out of scope to prune).
- **A plugin panel whose title is a special-case string** (e.g. starts with a number or punctuation): sorting is a plain case-insensitive `localeCompare`, so these slot naturally; no special handling.

## Requirements *(mandatory)*

### Functional Requirements

**A. System Tools page removal**

- **FR-001**: System MUST remove the `system-tools` registration from the config registry so the "System Tools" page no longer appears in Settings; the remaining modules' rendering is otherwise unaffected.
- **FR-002**: System MUST delete the legacy endpoint `src/app/api/system/bash/route.ts` so `POST /api/system/bash` returns `404`.
- **FR-003**: System MUST delete the now-orphaned `src/lib/system/bash.ts` (its exported `runBash`, `BashResult`, `DEFAULT_TIMEOUT_MS`, `MAX_TIMEOUT_MS`, and private `truncate` helper) — it is imported only by the deleted route.
- **FR-004**: System MUST leave the persisted `data/config/system-tools.json` (if present) in place — no migration, no cleanup (per @alex). After FR-001 nothing reads it, so it is inert.
- **FR-005**: `GET /api/config` MUST NOT return a `system-tools` namespace after the change.
- **FR-006**: The `run-command` namespace, the Command Execution page, and the `run_command` tool MUST remain exactly as they are (no behavior change to real command execution).

**B. Knowledge bases tab removal**

- **FR-007**: System MUST remove the "Knowledge bases" tab from the per-agent editor in `AgentDetails.tsx`: drop `"kb"` from the `Tab` union, remove the `KnowledgeBasesGrid` import, the `<TabButton …>Knowledge bases</TabButton>`, and the `{tab === "kb" && (…)}` render block. The remaining tabs (Instructions / Skills / Tools / MCP) are unaffected.
- **FR-008**: System MUST delete `src/components/apps/settings/assistant/KnowledgeBasesGrid.tsx` (orphaned once the tab is gone) and remove its two re-exports (`KnowledgeBasesGrid`, `KnowledgeBasesGridProps`) from `src/components/apps/settings/assistant/index.ts`.
- **FR-009**: The Knowledge Base *item* and its MCP tools (`kb_search` / `kb_retrieve`), `src/lib/agent/kb-catalog.ts` (`listKnowledgeBases`), the per-agent index block in `src/lib/agent/instructions.ts` (`buildKbIndexBlock`), and the per-agent/subagent `kbs` allowlist **data model** MUST all remain. Only the Settings UI tab is removed.
- **FR-010**: A per-agent `kbs` allowlist, if set, MUST still round-trip through the agent API and apply at runtime (unset = all KBs); removing the tab MUST NOT change that runtime behavior.

**C. Alphabetical Settings modules**

- **FR-011**: `listConfigSchemas()` MUST return its schemas sorted by display title, case-insensitive, rather than by the numeric `order` field.
- **FR-012**: The sort MUST be stable/deterministic for equal titles — tie-broken by namespace — so the sidebar order does not flicker between renders.
- **FR-013**: Both built-in registrations **and** plugin-contributed panels (merged by `listConfigSchemas()` before sorting) MUST be interleaved alphabetically in a single A→Z list.
- **FR-014**: The Settings UI renders the `schemas` array in the order it arrives from `GET /api/config`, so the reorder MUST require no change to the client-side Settings rendering (it stays correct automatically as modules are added).

**Cross-cutting**

- **FR-015**: After all three changes, a repo-wide search over `src/` for `system-tools`, `@/lib/system/bash`, `runBash`, `BashResult`, `api/system/bash`, and `KnowledgeBasesGrid` MUST return no hits (in the `system-tools` case, the self-referential comment in the deleted lib is itself removed).
- **FR-016**: The build and typecheck MUST be clean with no dangling references.

### Key Entities *(include if feature involves data)*

- **Config schema / namespace**: a Settings module entry (`namespace`, `title`, `order`, `fields`). `order` continues to exist as data but no longer drives sidebar position after FR-011/FR-013.
- **`system-tools` registration**: the specific module removed by FR-001 (single `enabled` boolean, `order: 37`).
- **`CatalogKb` / `Catalog` / `AgentMeta.kbs`**: the per-agent KB allowlist data model that REMAINS (FR-009); only the `KnowledgeBasesGrid` UI that edits it is removed.
- **`BashResult` / `runBash`**: the deleted lib's types/functions (FR-003); no shared consumer survives.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Settings shows no "System Tools" page and no "Knowledge bases" agent tab; the sidebar is A→Z. Verified by opening Settings in a running build.
- **SC-002**: `GET /api/config` omits the `system-tools` namespace and its `schemas` array is sorted by title; `POST /api/system/bash` returns `404`.
- **SC-003**: A repo-wide search for `system-tools`, `@/lib/system/bash`, `runBash`, `BashResult`, `api/system/bash`, and `KnowledgeBasesGrid` returns no hits in `src/`.
- **SC-004**: Build and typecheck are clean; the `run_command` tool, the Command Execution page, and the Knowledge Base item (`kb_search` / `kb_retrieve`) all behave exactly as before (no regression).
- **SC-005**: Adding a hypothetical Settings module (or a plugin panel) slots it into the correct alphabetical position with no further code change.

## Assumptions

- The Settings sidebar is rendered directly from the `schemas` array order returned by `GET /api/config`; no client-side Settings component re-sorts the module list (the `sort` calls in `McpServersTab` / `RunCommandTab` / `LogsTab` are internal to their own lists, not the sidebar). So the reorder lives entirely in `listConfigSchemas()`.
- `listConfigSchemas()` is the single point where built-in registrations and plugin panels are merged and ordered — it is the one place to apply the alphabetical sort (confirmed in `src/lib/config/registry.ts`).
- Removing the `system-tools` registration alone removes its Settings page, because no Settings tab supplies a `customComponent` for it — it is rendered generically from its single `fields` entry.
- The `kbs` allowlist and the Knowledge Base item are deliberately **out of scope** for removal (FR-009); the change is UI-only for the per-agent KB allowlist.
- The persisted `system-tools.json` is left in place per @alex; an orphaned file with no reader is harmless and requires no migration hook.
- The alphabetical sort is a presentation choice with no behavioral dependency: neither the assistant's `config_*` tools nor the `/api/config` PATCH path depends on module ordering.

## Open Questions (non-blocking — resolve at design stage)

1. **(C) Sort comparator detail** — exact `localeCompare` arguments and the namespace tie-break (proposed in FR-011/FR-012: case-insensitive `localeCompare` on title, then namespace). Confirm no module title contains locale-sensitive characters that would sort unexpectedly.
2. **(B) `catalog.kbs` pruning** — after FR-008, the frontend `Catalog` type and the agent API's `kbs` payload are no longer consumed by Settings. Leave them (harmless, minimal diff) or prune them too? Default: **leave them** — removing them touches the agent API contract and the Knowledge Base item's catalog source for no functional gain.
3. **(A) Whether to delete `data/config/system-tools.json`** — settled: **no** (FR-004, per @alex). Recorded here only so the design/plan doesn't re-open it.
