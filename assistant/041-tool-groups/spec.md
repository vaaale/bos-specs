# Feature Specification: Tool Groups, Better Tool Discovery, and Group-Aware Settings

**Feature Branch**: `bos/tool-groups`

**Created**: 2026-08-30

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Organize tools into groups in the system prompt. Only include groups where at least one tool is granted. Under each group name, a short instruction telling the agent how to search for tools in that group with `find_tools` — shown only when at least one tool in the group is deferred. Group descriptions must be editable in Settings → Tools. Marketplace items must declare their own group name and description instead of everything landing in SERVICE TOOLS. Settings → Tools groups should be collapsible, collapsed by default. Avoid the gotcha where the documented search pattern does not match the tools' actual name prefix. `find_tools` must keep working for non-exact searches, and its search logic should be improved."

## Context: what exists today

Four facts about the current implementation shape this spec.

1. **No tool inventory exists in the system prompt.** `composeInstructions()`
   (`src/lib/agent/instructions.ts`) emits date/time, default prompt, personality,
   memory, `## Skills`, `## MCP servers`, `## Knowledge bases`. Tools reach the model
   only through the provider's native tool-calling field (`src/lib/assistant/model-turn.ts`),
   which is a flat list of `{name, description, schema}` carrying no group metadata in
   either the Anthropic or the OpenAI wire format. Grouping therefore cannot live in
   the tool definitions; it can only be a prompt-side index. This feature adds a
   fourth index block alongside Skills / MCP / Knowledge bases.

2. **Groups already exist as data, but only to widen search.** `Capability.group`
   (`src/lib/agent/capabilities-registry.ts`) is a free-text display name, and
   `GROUP_DEFINITIONS` maps that name to a description used solely to expand
   `find_tools` scoring. Neither is user-editable, neither has a stable identifier, and
   service-declared tools are hardcoded into a single `"Service Tools"` group by
   `service-tool-bridge.ts`.

3. **Many groups have no common name prefix**, so a prefix-based search instruction
   cannot be correct in general. `Scheduler` holds `list_scheduled_tasks`,
   `create_scheduled_task`, `run_task_now` — no shared prefix at all. `Files` holds
   `file_*` plus `view_image` and `video_keyframes`. `Dev` holds `dev_*`, `bos_source_*`
   and `run_command`. `Skills` holds `skill_*` plus `self_improve`. Conversely one
   prefix spans several groups: `app_install`/`app_list` are *Apps* while `app_spec_*`
   are *Specs*; `bos_app_launch` is *OS* while `bos_source_read` is *Dev*.

4. **Guidance about tools currently lives in agent prose, twice.** Both
   `seed/agents/default_agent/AGENT.md` (the shared default prompt) and
   `seed/agents/assistant/AGENT.md` carry a hand-written `# Tools` section; for the
   assistant `composeInstructions()` concatenates both, so the model receives two
   overlapping and partly incorrect sets of instructions. Delegated ephemeral and
   surface agents — composed by `src/lib/assistant/delegation-gate.ts` — read neither,
   so they receive no discovery guidance at all.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - The agent can see what capability families it has (Priority: P1)

An agent whose tools are mostly deferred sees a short, arbitrary-looking list of
visible tools and has no way to know which capability families exist behind
`find_tools`. With this feature its system prompt carries a group index: every group
containing at least one granted tool, each with a one-line description, and — for
groups that still hold hidden tools — an explicit instruction for revealing them.

**Why this priority**: This is the whole point. Deferred tools are invisible by
design; without an index the agent must already suspect a capability exists before it
can search for it, which is exactly the failure mode deferred visibility introduces.

**Independent Test**: Configure an agent whose entire Web group is deferred, start a
run, and confirm the composed system prompt names the Web group with its description
and a discovery instruction, and that the agent reveals and calls `web_search`
without the user naming the tool.

**Acceptance Scenarios**:

1. **Given** an agent granted `web_search` and `web_fetch`, both deferred, **When** a
   run starts, **Then** the system prompt contains a Web group entry with its
   description and a discovery instruction, and no group the agent lacks.
2. **Given** an agent granted `file_read` (visible) and no deferred Files tools,
   **When** a run starts, **Then** the Files group appears with its description but
   **without** a discovery instruction.
3. **Given** an agent granted no tools in the Memory group, **When** a run starts,
   **Then** the Memory group does not appear at all.
4. **Given** an agent with zero deferred tools, **When** a run starts, **Then** no
   discovery instruction appears anywhere in the block.

---

### User Story 2 - Discovery finds the right tool, and says so honestly (Priority: P1)

`find_tools` is the only route to a deferred tool, so its search quality is the
ceiling on what an agent can do. Today it matches the **entire query string** as a
substring against descriptions and group names, so any natural-language query ("send
an email to my colleague") can only ever match on the id-word rule — every
description and group signal is dead weight for exactly the queries they were meant
to serve. It has no term weighting, so "list" counts as much as "workflow"; no
stopword handling, so "to" in a query matches `file_to_markdown`; no morphology, so
"files" misses `file_read`; no synonyms, so "email" does not reach `gmail_*`. It then
silently truncates to a configured cap (default 10), which no Workflows (12 tools) or
OKF (~35 tools) group can survive intact. Both free-text search and a new
group-scoped mode must be substantially better, and must never drop matches silently.

**Why this priority**: A discovery instruction that under-delivers without saying so
is worse than no instruction — the agent concludes the capability does not exist and
tells the user it cannot help.

**Independent Test**: Ask an agent with everything deferred to "send an email
summarizing the meeting", and confirm `find_tools` surfaces the Gmail send tool from
a natural-language query that shares no literal word with the tool's id. Separately,
invoke group mode on a group larger than the result cap and confirm either all
members return or the response states how many were withheld.

**Acceptance Scenarios**:

1. **Given** a natural-language query sharing no id word with any tool, **When**
   `find_tools` runs, **Then** relevant tools are still returned via description,
   group, and alias signals.
2. **Given** a query containing common words ("list the files in my folder"), **When**
   `find_tools` runs, **Then** discriminative terms dominate the ranking and tools
   matched only on a stopword do not appear.
3. **Given** a group with more granted deferred tools than the result cap, **When**
   group mode is invoked, **Then** every member is returned, or the response states
   the exact number withheld.
4. **Given** a group whose members share no name prefix, **When** the agent follows
   the prompt's instruction for that group, **Then** every granted deferred member is
   returned.
5. **Given** a query matching nothing, **When** `find_tools` runs, **Then** the
   response lists the groups available to that agent instead of an empty array.
6. **Given** a query whose best match is a tool the agent already has visible,
   **When** `find_tools` runs, **Then** the response says so rather than appearing
   to find nothing.
7. **Given** an unrecognized group in group mode, **When** it is invoked, **Then** an
   explicit error names the groups available to that agent.
8. **Given** a tool the agent is not granted, **When** any search mode runs, **Then**
   it is never returned — search never widens the allowlist.

---

### User Story 3 - A marketplace item's tools appear under its own name (Priority: P2)

Every tool a marketplace item contributes lands in one undifferentiated `Service
Tools` group, so the Workflow Manager's 12 `workflow_*` tools and the OKF Knowledge
Base's ~35 `okf_*` tools are indistinguishable in both the prompt and Settings. An
item declares its own group name and description, and its tools appear under that
heading.

**Why this priority**: Depends on Stories 1–2 having somewhere to render, but is the
difference between the index being useful and being noise once more than one
tool-declaring item is installed.

**Independent Test**: Install both tool-declaring items, start a run granting their
tools, and confirm the prompt shows two distinct named groups carrying the items' own
descriptions.

**Acceptance Scenarios**:

1. **Given** an item declaring group "Workflows", **When** its service starts and an
   agent is granted its tools, **Then** the prompt shows a `WORKFLOWS` group with the
   item's description, not `SERVICE TOOLS`.
2. **Given** an item that declares more than one group, **When** its service starts,
   **Then** each declared group appears separately with its own description.
3. **Given** an item whose manifest declares no tool group, **When** it is installed,
   **Then** installation reports a validation error identifying the missing field.
4. **Given** an installed item that predates this feature, **When** its service
   starts, **Then** its tool registrations are rejected with a service-level error
   visible in Settings → Services — not silently dropped, not silently defaulted.

---

### User Story 4 - A user can curate groups and browse tools comfortably (Priority: P2)

Group descriptions steer both the prompt block and search ranking, so a user must be
able to edit them where they already edit tool descriptions. Settings → Tools becomes
group-first: collapsed groups, each expandable to reveal its editable group
description and its tool rows.

**Why this priority**: The prompt block and the search are only as good as their
descriptions, and the catalog is already ~25 groups and ~150 rows — unusable flat.

**Independent Test**: Open Settings → Tools, confirm all groups are collapsed, expand
one, edit its description, start a run, confirm the edited text appears in the prompt
and influences search.

**Acceptance Scenarios**:

1. **Given** Settings → Tools finishes loading, **Then** every group is collapsed and
   shows its name and tool count.
2. **Given** an expanded group, **When** the user edits the group description and
   blurs, **Then** it auto-saves with the same status affordance tool descriptions
   use, and a Reset control restores the built-in text.
3. **Given** an edited group description, **When** a run starts, **Then** the prompt
   block and `find_tools` ranking both use the edited text.
4. **Given** the user types in the filter box, **When** a group contains a matching
   tool, **Then** that group auto-expands and non-matching groups stay collapsed.

---

### Edge Cases

- **Group vanishes mid-conversation.** A service stops after its group was written
  into the prompt. The prompt is composed once per run, so the block goes stale; a
  call to a now-unregistered tool must fail in-band with a clear message.
- **Granted but unregistered.** An agent's allowlist names ids belonging to a stopped
  service. Those ids do not resolve against the live registry; the group is omitted
  and the existing "unresolved tool ids" warning covers it.
- **Tools with no group.** Discovery tools, consent/elicitation tools, and
  window-scoped surface tools bypass the allowlist and have no capability entry. They
  must not synthesize an empty or "General" group.
- **Agent with an empty allowlist.** Produces no groups; the block is omitted whole.
- **Two services declaring the same group name.** Their tools merge under one heading;
  the description used must be deterministic and the collision surfaced to the user.
- **Group renamed between releases.** User overrides must not silently detach.
- **Query is a single character, or only stopwords.** Must be rejected with a usable
  message, not scored into noise.
- **Query names a group that exists but holds no granted tools for this agent.** Must
  be reported as such, not returned empty.
- **Duplicate guidance.** A user who edited their `AGENT.md` keeps the old
  hand-written `# Tools` prose alongside the generated block; the generated text must
  not contradict it.

## Requirements *(mandatory)*

### Functional Requirements — group model

- **FR-001**: Every capability MUST belong to exactly one group, identified by a
  stable group identifier independent of the group's display name, so a rename does
  not orphan its description or user overrides.
- **FR-002**: A group MUST carry a display name and a description. That description is
  the single source used by both the prompt block and search ranking — there MUST NOT
  be two independently-authored descriptions for one group.
- **FR-003**: The group catalog MUST be resolved live at use time (built-in groups
  plus groups registered by currently-running services), never from a frozen snapshot.
- **FR-004**: When a group's last member capability is unregistered, the group MUST
  leave the live catalog; persisted user overrides for it MUST survive.
- **FR-005**: Capabilities outside the capability registry (discovery, consent and
  elicitation, window-scoped surface tools) MUST NOT be assigned a group and MUST NOT
  appear in the group index.

### Functional Requirements — system prompt block

- **FR-006**: BOS MUST compose a tool-group block into the system instructions,
  listing each group with its display name and description.
- **FR-007**: A group MUST be included if and only if at least one of its tools is
  granted to the agent for that run — counting both visible and deferred tools.
- **FR-008**: A group's entry MUST include a discovery instruction if and only if at
  least one of its granted tools is deferred for that run.
- **FR-009**: The discovery instruction MUST be generated by BOS from the group's own
  identity, MUST be invocable verbatim, and MUST NOT depend on the group's tools
  sharing a common name prefix.
- **FR-010**: The block MUST be derived from the run's effective tool gate, not from
  an agent record, so named, ephemeral, and surface-delegated agents each get a block
  matching the tools they can actually call.
- **FR-011**: The block MUST be stable for a run's duration — it MUST NOT encode which
  deferred tools have already been revealed — so prompt caching is not invalidated on
  every step.
- **FR-012**: Group ordering MUST be deterministic, with dynamically-registered groups
  after built-in groups, so service lifecycle changes do not perturb the cacheable
  prefix of the prompt.
- **FR-013**: The block MUST carry a short preamble stating when to reach for
  `find_tools`, replacing the per-agent prose so every agent — including delegated
  ephemeral and surface agents — receives identical guidance.
- **FR-014**: If the agent has no granted registry tools, the block MUST be omitted
  entirely rather than emitted empty.
- **FR-015**: Under each group, the block MUST list the **bare names** of that group's
  granted *visible* tools, and MUST NOT restate their descriptions or schemas, which
  the provider already delivers natively. Deferred tools MUST NOT be named — a group
  reports only how many of its tools are hidden, alongside its discovery instruction.
  Naming them would defeat deferral; counting them does not.

### Functional Requirements — search quality

Free-text search remains the primary mode; group scoping is an addition, not a
replacement.

- **FR-016**: Ranking MUST score the query **term by term** against each searchable
  field, replacing whole-query substring matching, so multi-word natural-language
  queries can match descriptions, group names, and group descriptions.
- **FR-017**: Terms MUST be weighted by how discriminative they are across the
  capability corpus, so a rare term outranks a term shared by most tools.
- **FR-018**: Common words carrying no retrieval value MUST NOT contribute to
  ranking, so a query containing "to" does not match `file_to_markdown` on that basis.
- **FR-019**: Matching MUST tolerate ordinary morphological variation (singular/plural
  and common verb endings), so "files"/"file" and "listing"/"list" match.
- **FR-020**: A capability MUST be able to carry curated search aliases, and a group
  MUST be able to carry curated aliases, both authored alongside their descriptions
  and editable by the user, so vocabulary the descriptions do not use ("email",
  "spreadsheet", "browse") can be made to reach the right tool.
- **FR-021**: A tool matching more distinct query terms MUST outrank a tool matching
  one term repeatedly.
- **FR-022**: Results MUST be filtered by a relevance threshold rather than "any
  non-zero score", and ordering MUST be fully deterministic including tie-breaks.
- **FR-023**: Every result MUST state why it matched (which field or alias), so the
  model can judge relevance rather than trusting an opaque number.
- **FR-024**: Results MUST NOT be silently truncated in any mode. When a cap applies,
  the response MUST state how many matches were withheld and how to retrieve them.
- **FR-024a**: A discovery response MUST NOT carry tool input schemas. A returned tool
  is un-gated for subsequent steps and its schema is delivered by the provider's
  native tool field; duplicating it into the transcript is redundant data that is
  never read back. Responses carry identity and relevance only: id, description,
  group, and why it matched.
- **FR-024b**: Group-scoped results MUST NOT be capped. The free-text result cap
  (`tools.maxFindResults`) applies to free-text search only. A group's size is bounded
  by its own declaration, and FR-024a makes a whole group cheap to return, so
  SC-001 ("the group index plus one group-scoped call reaches any granted tool") holds
  without pagination.
- **FR-025**: A search matching nothing MUST return the groups available to the
  calling agent, with their descriptions, instead of an empty result.
- **FR-026**: When a query's best matches include tools the agent already has visible,
  the response MUST report them as already available rather than appearing to find
  nothing.
- **FR-027**: A query that cannot be searched (too short, only stopwords) MUST return
  an explanatory message, not an empty result.
- **FR-028**: Ranking MUST remain deterministic and computable without a model call or
  network access, so it stays unit-testable.

### Functional Requirements — group-scoped discovery

- **FR-029**: `find_tools` MUST support a group-scoped mode returning every deferred
  tool of a named group that the calling agent is granted.
- **FR-030**: Group matching MUST be tolerant — case- and whitespace-insensitive, and
  accepting the group's identifier, display name, or aliases — and MUST NOT require
  the model to reproduce an exact string.
- **FR-031**: Group-scoped mode MUST respect the agent's allowlist and deferred set
  exactly as free-text mode does.
- **FR-032**: An unrecognized group MUST return an explicit error naming the groups
  available to that agent, never an empty result.
- **FR-033**: A recognized group holding no granted deferred tools for this agent MUST
  say so explicitly.
- **FR-034**: Free-text queries that name a group MUST also surface that group's
  members, so the two modes do not diverge in what they can reach.
- **FR-035**: There MUST be exactly one `find_tools` implementation. Reachability
  analysis (see Resolved Decisions, D4) found the other two are already dead code:
  they MUST be deleted, not brought to parity.
- **FR-036**: The mechanism that derives revealed tool ids from the transcript MUST
  keep working across any change to the discovery response shape, including for
  conversations that already contain responses in the old shape.

### Functional Requirements — service-declared groups

- **FR-037**: A tool-exposing marketplace item MUST declare, in its manifest, the tool
  group(s) it contributes — each with an identifier, display name, and description.
- **FR-038**: An item MUST be able to declare more than one group, and each declared
  tool MUST resolve to exactly one of them.
- **FR-039**: Manifest tool-group declarations MUST be validated at install time, so a
  malformed or missing declaration fails at install rather than at first service start.
- **FR-040**: A service tool that does not resolve to a declared group MUST be rejected
  at registration with a reason, surfaced as a service-level error in the Settings UI,
  not only in logs.
- **FR-041**: There MUST be no generic, default, or fallback group anywhere in the
  system — not in the registry, not in the prompt block, not in the Settings UI, and
  not in search. A capability whose group id does not resolve is an error, and that
  error MUST be surfaced to the user (Settings → Services for a service-declared tool,
  Settings → Tools for a built-in), never absorbed by a placeholder bucket or an
  invented description. This retires three existing fallbacks: the `"General"` bucket
  in `ToolsTab` and in `ToolAccordions`, and `groupDescription()`'s synthesized
  `Capabilities in the "<name>" group.` text for an unknown group.
- **FR-042**: Every marketplace item, built-in app, and bundled agent affected by this
  change MUST be migrated within this feature — including their manifests, their
  bundled specs and docs, and any prompt text of theirs that names tools or describes
  how to discover them.

### Functional Requirements — settings

- **FR-043**: Settings → Tools MUST present tools grouped, with every group collapsed
  on load and independently expandable.
- **FR-044**: Each group header MUST show its display name and its tool count.
- **FR-045**: An expanded group MUST expose an editable group description and group
  aliases, with auto-save and reset-to-default, matching the existing per-tool editor.
- **FR-046**: Edited group text MUST take effect in both the prompt block and search
  ranking without a restart.
- **FR-047**: Settings → Tools MUST offer a filter matching tool ids, descriptions and
  aliases, auto-expanding groups that contain a match.
- **FR-048**: The grouping and collapsible-group presentation MUST be implemented once
  and shared by Settings → Tools and the per-agent tool picker, replacing the two
  existing duplicate implementations.
- **FR-049**: A group override MUST persist across restarts and MUST NOT be discarded
  while the group's owning service is stopped.

### Functional Requirements — corrections carried by this feature

- **FR-050**: Prompt text MUST NOT state that `find_tools` can discover MCP tools; MCP
  tools are reachable only through the MCP gateway tools.
- **FR-051**: Generated prompt text and tool descriptions MUST name only tools that
  exist. The MCP index block and MCP tool descriptions currently name four tools by
  non-existent legacy names, and the Knowledge Bases block names knowledge-base tools
  whose real names must be verified against the installed item.
- **FR-052**: The hand-written `# Tools` sections in the seeded default-agent and
  assistant prompts MUST be reduced to genuinely agent-specific guidance; inventory
  and discovery policy move to the generated block, and delegation guidance parked
  there moves to the delegation section.

### Key Entities

- **Tool group**: A named family of capabilities — stable identifier, display name,
  description, aliases, origin (built-in or service-declared). Owns no tools directly;
  tools reference it.
- **Group override**: A user-authored replacement for a group's description and
  aliases, persisted by group identifier, surviving the group's temporary absence.
- **Tool group block**: The generated system-prompt section listing the groups
  relevant to one run, with per-group discovery instructions.
- **Group declaration**: The manifest statement by which a marketplace item
  contributes one or more groups.
- **Search result**: A ranked capability with its schema, its score, and the reason it
  matched; delivered in an envelope that also reports total matches and any shortfall.

## Success Criteria *(mandatory)*

- **SC-001**: For an agent with every tool deferred, the group index plus one
  group-scoped call is sufficient to reach any granted tool — no guessing.
- **SC-002**: Any discovery response returns 100% of the calling agent's matching
  granted tools, or reports the exact number withheld. Zero silent omissions.
- **SC-003**: Every group shown in an agent's prompt contains at least one tool that
  agent can call; no group the agent lacks appears.
- **SC-004**: A benchmark set of natural-language queries — phrased the way a user
  would ask, sharing no literal id word with the target — resolves to the correct tool
  in the top three results, and that benchmark runs as a test.
- **SC-005**: No query returns a tool matched solely on a common word.
- **SC-006**: A search that finds nothing still tells the agent what families exist.
- **SC-007**: Two installed tool-declaring items appear as two distinctly named groups
  in both the prompt and Settings, with no occurrence of a generic fallback group.
- **SC-008**: Group text edited in Settings is reflected in the next run's prompt and
  ranking with no restart.
- **SC-009**: Settings → Tools renders the full catalog collapsed, and any single tool
  is reachable in at most two interactions.
- **SC-010**: No generated prompt text names a tool absent from the live registry —
  verified by a test that cross-checks every tool name mentioned in generated text.
- **SC-011**: Delegated ephemeral and surface agents receive the same discovery
  guidance as named agents.
- **SC-012**: The block adds no more than a small constant per group and restates no
  tool description already delivered natively.

## Assumptions

- Grouping is expressible only prompt-side; the provider tool field stays flat and
  untouched, and visible tools continue to be delivered natively.
- The system prompt continues to be composed once per run and cached, so the block is
  static for the run's duration.
- Deferral remains per-agent; there is no registry-wide default deferred set.
- Group descriptions and aliases are user-editable; group display names are not, since
  they are the join key between capabilities, overrides, and the prompt.
- Existing per-tool description overrides are unaffected and keep their storage.
- One group per tool is sufficient; no tool needs to appear in two groups.
- Ranking stays a deterministic pure function over the live catalog.

## Out of Scope

- Embedding- or model-based semantic search. Ranking must stay deterministic,
  offline, and unit-testable; curated aliases are the intended substitute.
- Ranking by usage history or per-conversation learning.
- Reordering or renaming built-in groups from the UI.
- Per-agent group descriptions — per-agent variation is already expressible through
  the allowlist.
- Bringing MCP server tools into the capability registry or into groups; MCP keeps its
  separate gateway and index block.
- Changing which tools are deferred by default for any existing agent.
- Any change to how tools execute, are gated, or are timed out.

## Resolved Decisions

Settled before `plan`; each replaces an earlier open question.

- **D1 — The block names visible tools, counts hidden ones** (was NC-001; see FR-015).
  The provider's tool field is flat and unordered, so a tool's group is otherwise
  inferrable only from its name prefix — and prefixes are demonstrably unreliable
  (Scheduler shares none; `app_` spans two groups). Bare names are cheap, live in the
  cached system block, and make the grouping real rather than implied. The asymmetry
  is the point: **visible → named, hidden → counted**, so the block never leaks what
  deferral exists to hide.

- **D2 — A group whose owning service is stopped is omitted** (was NC-002). Not a
  preference but a consequence of FR-004: groups are scoped to their tools' lifecycle,
  so when a service stops, its capabilities unregister and its group leaves the
  catalog — at which point BOS no longer knows the group those granted-but-unresolved
  ids belonged to. Showing "Workflows (unavailable)" would require registering groups
  at **install** time from the manifest rather than at tool-registration time, which
  contradicts FR-004. The loss is bounded: the existing unresolved-tool-ids warning
  already fires. Recorded as a deliberate v1 limitation; the upgrade path is
  manifest-sourced group registration, and it should be taken only if stopped-service
  confusion shows up in practice.

- **D3 — No suppression toggle** (was NC-003). The block is already gate-derived, so
  it is correct per agent without configuration, and a toggle would add a config
  namespace plus a second code path through prompt composition. Anyone wanting custom
  guidance can still write it in `AGENT.md`, which remains additive.

- **D4 — The two other `find_tools` implementations are deleted, not updated**
  (see FR-035). Reachability analysis over `src/`, `e2e/` and `tools/` found:
  `makeDiscoveryTools` (`src/lib/agent/subagents/tools.ts`) has **zero** callers;
  `toolsFor`, `SUBAGENT_TOOLS`, `SPEC_TOOLS`, `makeSpecTools`, `DEV_DELEGATE_SCHEMA`,
  `RUN_COMMAND_SCHEMA` and `pickDeferredIds` in that same file have zero external
  references; every remaining mention of the module elsewhere is a **code comment**.
  Its one real importer is `getToolSchema` in `src/app/api/assistant/discovery/route.ts`
  — and that route has zero callers repo-wide. `DEV_TOOLS` and
  `DELEGATE_TO_DEVELOPER` likewise survive only in comments. So both are dead code:
  `docs/dev/architecture-overview.md` §8.3 was **right** that `subagents/tools.ts`
  "was retired"; the file simply was never deleted. This means there is exactly one
  live `find_tools` (`src/lib/assistant/tools/server/discovery.ts`), and FR-016–FR-034
  have exactly one implementation site.
