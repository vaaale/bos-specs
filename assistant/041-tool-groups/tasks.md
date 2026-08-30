# Tasks — 041 Tool Groups, Better Tool Discovery, Group-Aware Settings

**Input**: [spec.md](spec.md), [design.md](design.md) (architecture of record), [plan.md](plan.md)

**Branch**: `bos/tool-groups` (BOS source + this spec store). Item repos carry their own `bos/tool-groups` branches.

**Format**: `[P]` = parallelizable (different files, no dependency on a sibling in the same phase).

**Ordering note**: US2 (P1, discovery) lands before US1 (P1, block) — the block renders `find_tools(group: …)` instructions that must already work. See plan.md.

---

## PHASE 1 — Foundational (blocks everything)

### T001 — Create the tool-group registry
- **Files:** `src/lib/agent/tool-groups.ts` (new)
- **Story:** foundation (US1/US2/US3/US4)
- **Work:**
  - Export `ToolGroup { id, name, description, aliases, origin: "builtin" | "service" }`.
  - Move the ~20 built-in groups out of `capabilities-registry.ts`'s `GROUP_DEFINITIONS` into a declared table here, each gaining a slug `id` (e.g. `Google Drive` → `google-drive`). **The table's order is the canonical display/prompt order (FR-012).**
  - `globalThis`-backed dynamic layer — `registerToolGroups()` / `unregisterToolGroups()` / `listToolGroups()` — mirroring `registerAdditionalCapabilities()`'s `__bos_dynamic_capabilities__` pattern so it survives HMR.
  - `resolveGroup(query)` with **fixed precedence**: exact id → case/whitespace-normalized display name → alias. First match wins; never guess between candidates (FR-030).
  - Framework-free. Must NOT import `capabilities-registry.ts` (cycle).
- **Acceptance:** compiles; no import cycle; `resolveGroup` precedence is total and documented in-file.

### T002 — Migrate `Capability.group` to group ids
- **Files:** `src/lib/agent/capabilities-registry.ts`
- **Story:** foundation
- **Work:**
  - Rewrite every `group:` value to the slug id from T001.
  - Add optional `aliases?: string[]` to `Capability` (FR-020).
  - **Delete** `GROUP_DEFINITIONS` and `groupDescription()` — including its synthesized `Capabilities in the "<name>" group.` fallback, which FR-041 forbids and which must not be reimplemented against the new table.
  - `integrationCapabilities()` passes group ids.
- **Acceptance:** `tsc` clean; no fallback description remains anywhere in the module.

### T003 — Group override store
- **Files:** `src/lib/agent/tool-group-overrides.ts` (new)
- **Story:** US4 (needed early by the effective view)
- **Work:**
  - `dataDir()/tool-group-overrides.json`, atomic write, shaped after `tool-metadata-overrides.ts`.
  - `readGroupOverrides()`, `setGroupOverride(id, {description?, aliases?})`, `getEffectiveGroups()`.
  - Unlike `setMetadataOverride`, validate **shape, not membership** — a group present only while a service runs must be editable, and an override is never deleted because its group left the catalog (FR-049).
  - Resolve per invocation. No process-lifetime cache (FR-046 — see design §3.3(b)).
- **Acceptance:** override survives a group leaving and re-entering the catalog.

### T004 — `/api/tool-groups` route
- **Files:** `src/app/api/tool-groups/route.ts` (new)
- **Story:** US4
- **Work:** `GET` → effective groups (id, name, description, aliases, origin, sourceDescription). `PATCH {groupId, description?, aliases?}`. Leave `/api/tool-descriptions`'s existing `{id, description}` contract untouched.
- **Acceptance:** round-trips an override; unknown group id returns a 400 naming valid ids (no silent accept).

### T005 — [P] Unit tests: group model
- **Files:** `tests/agent/tool-groups.test.ts` (new)
- **Work:** built-in table integrity; dynamic register/unregister lifecycle; `resolveGroup` precedence incl. an alias colliding with another group's id; override persistence across absence; **the ADR-1/R3 invariant that every `Capability.group` in `listCapabilities()` resolves to a live group**; and that no code path yields a fallback group.
- **Acceptance:** green.

---

## PHASE 2 — US2 (P1): discovery that finds the right tool and says so honestly

### T010 — Delete the two dead discovery implementations
- **Files:** `src/lib/agent/subagents/tools.ts` (delete), `src/app/api/assistant/discovery/route.ts` (delete)
- **Story:** US2 (spec D4)
- **Work:** Remove both. Verified dead: `makeDiscoveryTools` has zero callers; `toolsFor`/`SUBAGENT_TOOLS`/`SPEC_TOOLS`/`makeSpecTools`/`DEV_DELEGATE_SCHEMA`/`RUN_COMMAND_SCHEMA`/`pickDeferredIds` have zero external references; `DEV_TOOLS`/`DELEGATE_TO_DEVELOPER` survive only in comments; the module's one importer is the route, which has zero callers repo-wide. Update the comments in `dev-source.ts`, `dev-delegate.ts`, `adapt.ts`, `api/dev/source/route.ts` that reference the deleted module.
- **Acceptance:** `npx tsc --noEmit` clean — this is the safety net for a missed transitive importer (R4). `npm run test:unit` still green.

### T011 — Ranking benchmark query set (BEFORE tuning — R5)
- **Files:** `tests/agent/fixtures/discovery-benchmark.ts` (new)
- **Story:** US2
- **Work:** Commit ≥25 `{query, expectedToolId}` pairs phrased as a user would ask, each sharing **no literal id word** with its target (e.g. "send a note to my colleague" → `gmail_messages_send`; "what's in this PDF" → `file_to_markdown`). Cover integration groups, file ops, scheduler (no common prefix), and service tools.
- **Acceptance:** committed and reviewed before T012 begins. Ranking is tuned to the set, never the set to the ranking.

### T012 — Ranking module
- **Files:** `src/lib/agent/discovery-search.ts` (new)
- **Story:** US2
- **Work:** `tokenize()` (lowercase, split, stopwords, `s|es|ing|ed` folding — FR-018/FR-019); `buildIndex(caps, groups)` (per-term document frequency over `{id, description, aliases, groupName, groupDescription}`, memoized on a catalog fingerprint — FR-003/FR-017); `score()` (field-weighted per-term accumulation + distinct-term coverage bonus, returning `{score, reasons[]}` — FR-016/FR-021/FR-023); `search()` (relevance threshold, sort by `score desc, groupOrder, id` — FR-022). Pure, framework-free, no I/O (FR-028).
  - **Do not boost group-level fields.** IDF already discounts terms repeated across a group's members; that is correct behaviour, not dilution to be "fixed" (design §3.3(c)).
- **Acceptance:** deterministic across runs; benchmark from T011 puts the target in the top three.

### T013 — [P] Unit tests: ranking
- **Files:** `tests/agent/discovery-search.test.ts` (new)
- **Work:** per-term matching against descriptions; IDF ordering; stopword rejection (a query containing "to" must not match `file_to_markdown` on that basis — FR-018/SC-005); morphology; alias hits; coverage bonus; threshold; tie-break determinism; the T011 benchmark (SC-004).

### T014 — Rewrite `find_tools`
- **Files:** `src/lib/assistant/tools/server/discovery.ts`
- **Story:** US2
- **Work:**
  - Add optional `group` parameter; response envelope `{results, totalMatches, withheld, alreadyVisible?, groups?}`.
  - **Results carry `{id, description, group, reasons}` and NO schema** (FR-024a, ADR-7) — the existing reveal path un-gates the tool so the provider delivers its schema natively on the next step.
  - Six modes per design §3.3(d): free-text; group-only (**uncapped** — FR-024b); both; unresolved group → error naming available groups; resolved-but-empty → explicit; no match → `groups` index (FR-025); unsearchable query → explanatory message (FR-027).
  - `alreadyVisible` = above-threshold hits that are granted but not deferred (FR-026).
  - `getMaxFindResults()` applies to **free-text only**; disclose `withheld` when it bites (FR-024).
  - Update the tool's own description to document `group` and that groups are indexed in the system prompt.
- **Acceptance:** all six modes covered by T016; a 35-tool group returns all 35.

### T015 — Dual-shape reveal derivation
- **Files:** `src/lib/assistant/messages.ts`, `src/lib/agent/tool-gate.ts`
- **Story:** US2 (FR-036, ADR-4, R1)
- **Work:** Both parsers accept the legacy bare array **and** the envelope's `results`, permanently — old transcripts are replayed from disk forever. `results` is the **only** reveal source: `alreadyVisible` and `groups` must never be read as grants.
  - **Do not change `deriveRevealedIds(messages)` in `agent-loop.ts:294` to read `contextMessages`** (R9) — it must stay on the canonical transcript or reveals silently vanish under compaction.
- **Acceptance:** T016 green.

### T016 — [P] Unit tests: discovery modes + reveal shapes
- **Files:** `tests/assistant/discovery-modes.test.ts` (new), `tests/assistant/revealed-ids-shapes.test.ts` (new)
- **Work:** all six modes incl. uncapped group mode and truncation disclosure; allowlist never widened; no schema in any payload. Separately: reveal derivation over a transcript containing legacy-array results, envelope results, both mixed, and a **compacted view whose find_tools result was cleared** while the canonical transcript retains it (R9).

### T017 — Retire `scoreCapability`
- **Files:** `src/lib/agent/discovery-score.ts`
- **Work:** Remove `scoreCapability` (superseded by T012). Keep `scoreAgent` — `find_agent` is out of scope — either in place or relocated alongside the new module; pick one and leave no duplicate.

---

## PHASE 3 — US1 (P1): the tool-group block

### T020 — `buildToolGroupsBlock`
- **Files:** `src/lib/agent/instructions.ts`
- **Story:** US1
- **Work:** New builder beside `buildSkillsIndexBlock`/`buildMcpIndexBlock`/`buildKbIndexBlock`, same "index + how to expand it" shape and same `return ""` when empty (FR-014).
  - **Preamble (FR-013):** fixed BOS-authored text — when to reach for `find_tools`, and explicitly that MCP tools are reachable only via the MCP gateway tools, not `find_tools` (FR-050).
  - **Membership:** `gate.allow` ∩ `listCapabilities()`; group emitted iff ≥1 such id (FR-007); discovery instruction iff ≥1 is in `gate.deferred` (FR-008). Nothing about `revealed` enters the block (FR-011).
  - **Rendering (D1/FR-015):** name each granted *visible* tool; **count** hidden ones; never name a deferred tool.
  - **Exclusions (FR-005):** non-registry tools (discovery, consent/elicitation, surface Tier-2) get no group and no bucket.
  - **Order (FR-012):** T001's table order, then dynamic groups by id.
- **Acceptance:** T023 green.

### T021 — Thread the gate into composition (primary run)
- **Files:** `src/lib/assistant/start-run.ts`, `src/lib/agent/instructions.ts`
- **Work:** `composeInstructions(agentId, { gate, tools })`; `start-run.ts` already builds both — pass them. Block omitted, not guessed, when the argument is absent.

### T022 — Same for delegated agents
- **Files:** `src/lib/assistant/delegation-gate.ts`
- **Story:** US1 (FR-010, SC-011)
- **Work:** `namedComposeSystem` / ephemeral / surface compose functions each close over their own gate so the block matches the tools that kind can actually call (ephemeral and surface both have empty `deferred`, so they get groups with no discovery instructions).

### T023 — [P] Unit tests: the block
- **Files:** `tests/assistant/tool-groups-block.test.ts` (new)
- **Work:** US1's four acceptance scenarios; preamble present for named/ephemeral/surface (SC-011); D1 rendering (visible named, hidden counted, no deferred names); ordering stable when a dynamic group appears/disappears; empty gate ⇒ no block.

### T024 — Trim the seeded prompts
- **Files:** `seed/agents/default_agent/AGENT.md`, `seed/agents/assistant/AGENT.md`
- **Story:** US1 (FR-052)
- **Work:** Remove the `# Tools` inventory/discovery prose from both (it now comes from T020's preamble). Move the assistant's "The Agent tool is a core execution tool…" line into its delegation-triggers section — it is delegation policy, not discovery. **Keep** genuinely agent-specific guidance in `default_agent` (web_search citation habit, scratchpad, memory). Remove the "a connected MCP service is likely to provide the capability" bullet (FR-050 — `find_tools` structurally cannot return MCP tools).
- **Note:** `seed-sync` only refreshes copies BOS wrote that nobody edited, so a user with an edited `AGENT.md` keeps the old prose alongside the new block. The generated text must not contradict it.

---

## PHASE 4 — US3 (P2): service-declared groups

### T030 — Declaration types
- **Files:** `src/core/service/serviceToolTypes.ts`, `src/core/service/types.ts`
- **Work:** Add `ToolGroupDeclaration { id, name, description, aliases? }`; add `ToolDeclaration.group?: string`; add `ServiceManifest.toolGroups?: ToolGroupDeclaration[]`; **remove the unused `ToolDeclaration.categories`** so there is only one grouping concept.
  - These are compile-time types. An item's worker runs unbundled and cannot import from `src/` — it posts a plain object literal over `tool_declare`. Correct the module header comment accordingly.

### T031 — Manifest validation
- **Files:** `src/core/service/manifestValidator.ts`
- **Work:** When `deploymentMode === "tools"`, require a non-empty `toolGroups` with well-formed, unique, slug-shaped ids and non-empty name/description (FR-039). Reject at install with a message naming the offending field.

### T032 — Group resolution at declare time
- **Files:** `src/core/service/ServiceManager.ts`
- **Work:** In the `tool_declare` case, resolve `declaration.group` against the manifest's `toolGroups` before calling the bridge — default to the sole declared group; **error** if several are declared and none is named. Register the manifest's groups via `registerToolGroups()` on first successful use.

### T033 — Bridge: group-aware registration + loud rejection
- **Files:** `src/lib/agent/service-tool-bridge.ts`
- **Work:** `registerTool` takes the resolved group id and registers the capability under it — **the hardcoded `group: "Service Tools"` is removed with no fallback (FR-041)**. Extend `unregisterServiceTools`/`syncCapabilityAfterRemoval` to drop a group when its last member goes (FR-004). On rejection, set the service's `lastError` in addition to the existing `tool:register-rejected` warn (FR-040).

### T034 — Surface the failure in Settings
- **Files:** `src/components/apps/settings/` services panel (verify the existing `lastError` rendering path)
- **Work:** Confirm a rejected declaration is visible to the user, not log-only. If `lastError` is already rendered, this is verification plus a test; if not, wire it.
- **Acceptance:** a fixture service declaring an unknown group shows an error in the UI.

### T035 — [P] Unit tests: service groups
- **Files:** `tests/agent/service-tool-groups.test.ts` (new)
- **Work:** manifest validation accept/reject; group resolution incl. the multi-group-no-name error; capability registered under the declared group; group removed when its last tool unregisters while its **override survives**; a pre-feature manifest (no `toolGroups`) is rejected with `lastError` set — US3 scenario 4, using a worker fixture in the style of `tests/agent/service-tool-bridge.test.ts`.

### T036 — [P] Migrate `workflows` (item repo — do this FIRST, see plan §Rollout)
- **Files:** `bos-marketplace/items/workflows/services/service.json`, `spec/**`, `docs/**`
- **Work:** Declare group `workflows` / "Workflows" with a description; assign the 12 `workflow_*` tools. Update the item's own spec + docs tool-surface text.

### T037 — [P] Migrate `okf-knowledge-base` (item repo)
- **Files:** `bos-marketplace/items/okf-knowledge-base/services/service.json`, `spec/tool-surface.md`, `spec/design.md`
- **Work:** Declare its group(s) for the ~35 `okf_*` tools — one group unless the tool families genuinely differ. Manifest `description` currently says "Exposes `okf_*` tools natively"; update to name the group.

### T038 — [P] Item-bundled agents/skills prompt text
- **Files:** `bos-marketplace/items/okf-knowledge-base/` bundled agents/skills
- **Work:** Update prose that names tools or describes how to discover them, consistent with the new block and FR-050.

---

## PHASE 5 — US4 (P2): Settings

### T040 — Shared collapsible group component
- **Files:** `src/components/apps/settings/tools/ToolGroupList.tsx` (new)
- **Story:** US4 (FR-043/FR-044/FR-048)
- **Work:** One grouping helper + one collapsible shell. Collapsed on load; header shows display name and tool count; independently expandable.
  - **No fallback bucket (FR-041):** neither this component nor its helper may coerce an unresolved group into `"General"` — an unresolvable id is surfaced as an error.

### T041 — `ToolsTab` adopts it, plus the group editor and filter
- **Files:** `src/components/apps/settings/ToolsTab.tsx`
- **Work:** Replace its local `groupByCategory` (and its `|| "General"`) with T040. Add the per-group description/alias editor using `useAutoSave` + `AutoSaveStatus`, with reset-to-default, mirroring `ToolRow` (FR-045). Add the filter box that matches ids, descriptions and aliases and auto-expands matching groups (FR-047).

### T042 — [P] `ToolAccordions` adopts it
- **Files:** `src/components/apps/settings/assistant/ToolAccordions.tsx`
- **Work:** Replace its duplicate `groupByCategory` and `|| "General"` with T040; keep its select-all/clear-group behaviour.

### T043 — [P] Tools panel shows service tools
- **Files:** `src/lib/agent/tool-manifest.ts`, `src/components/agent/v2/InfoPanelV2.tsx`
- **Work:** `ASSISTANT_TOOLS` switches from `actionCapabilities()` (static `CAPABILITIES`) to `listCapabilities()`, so service-declared tools appear at all; resolve group ids to display names for rendering.

### T044 — [P] Catalog payload carries group ids
- **Files:** `src/app/api/tool-descriptions/route.ts`
- **Work:** Effective catalog exposes the group id; group editing stays in `/api/tool-groups` (T004).

### T045 — E2E smoke
- **Files:** `e2e/tool-groups.spec.ts` (new)
- **Work:** Settings → Tools loads with every group collapsed; expanding one and editing its description persists; the edited text appears in the next run's composed prompt (SC-008/SC-009).

---

## PHASE 6 — Corrections, docs, gates

### T050 — Fix prompt text that names non-existent tools
- **Files:** `src/lib/agent/instructions.ts`, `src/lib/assistant/tools/server/mcp.ts`
- **Story:** FR-051
- **Work:** The MCP index block names `searchMcpTools`, `getMcpToolSchema`, `callMcpTool`, `listMcpServerTools`; the real ids are `mcp_tool_search`, `mcp_tool_schema`, `mcp_tool_call`, `mcp_server_tools`. The same stale names recur inside `mcp.ts`'s own tool descriptions. **Also verify the Knowledge Bases block**: it names `kbs_tool_search`/`kbs_tool_retrieve` while the `knowledge-base` item advertises `kb_search`/`kb_retrieve` — check the installed item and correct whichever is wrong.

### T051 — [P] SC-010 enforcement test
- **Files:** `tests/assistant/prompt-tool-names.test.ts` (new)
- **Work:** Extract every tool-name-shaped token from all generated prompt text (block, preamble, MCP index, KB index, skills index) and assert each resolves against `listCapabilities()` or a known non-registry tool. This is what keeps FR-050/FR-051 from regressing.

### T052 — Docs (Constitution VI — not optional)
- **Files:** `docs/dev/apps/services.md` §15; `docs/dev/assistant/actions-and-tools.md`; `docs/usage/settings/tools.md` (new); `docs/usage/settings/overview.md`; `docs/dev/architecture-overview.md` §8.2/§8.3
- **Work:** §15 gains the `toolGroups` contract. `actions-and-tools.md` is **rewritten, not patched** — it opens with a "Stale (v1 CopilotKit path)" banner and its tool tables describe the retired registration path. New user page for Settings → Tools (that directory has no Tools page today), linked from `overview.md`. `architecture-overview.md` §8.2's group list and "80+ capabilities, 20+ groups" figures updated; §8.3's reference to the module deleted in T010 removed.

### T053 — [P] Supersede 039's group statement
- **Files:** `bos-system-specs/app-infrastructure/039-service-tool-exposure/design.md`
- **Work:** Note that the `"Service Tools"` group in its file plan (`:175`) is superseded by 041 ADR-5, and that its open question (`:252`) about naming service-tool capability descriptors is closed.

### T054 — Gates
- **Work:** `npx tsc --noEmit`, `npm run lint`, `npm run test:unit`, `npm run test:e2e` for the new smoke. Do not run `npm run build` while `next dev` is running.
- **Acceptance:** all green; then follow plan.md §Rollout — items released first, BOS second, verified against the Dokploy deployment rather than this checkout.

---

## Dependency summary

- T001 → T002 → everything.
- T003/T004 gate T041 and T014's group-index responses.
- T010 before T014/T017 (delete before rewriting neighbours).
- T011 strictly before T012.
- Phase 2 before Phase 3 (the block's instruction must work).
- T030–T033 before T036–T038 make sense to test, but the item edits (T036–T038) **ship first**.
- T050/T051 can start any time after T020.
