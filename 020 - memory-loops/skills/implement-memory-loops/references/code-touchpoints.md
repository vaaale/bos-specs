# Code touchpoints — memory loops (spec 020)

Per-file design detail for the implementing agent. Requirement numbers refer to the spec.
These are the intended shapes; adjust to local conventions where the code disagrees, and
say so in your report.

## New: `src/lib/agent/memory/episodes.ts` (server-only)

Store for `data/memory/episodes/`. Copy the atomic temp-file+rename write and the
`looksLikeInjection` gate from `memory/curated.ts`.

```ts
interface Episode {
  conversationId: string;
  createdAt: string;          // ISO
  updatedAt: string;
  watermark: string;          // last reviewed message id (or index as string)
  skillsUsed: string[];       // skill ids, set mechanically (FR-008)
  status: "pending" | "consolidated";
  skillCandidates: string[];  // task-class slugs (FR-014c)
  sections: Record<EpisodeSection, string[]>; // bullet lists
}
type EpisodeSection = "task-outcome" | "worked-failed" | "corrections" | "lessons" | "profile-suggestions";
```

API: `readEpisode(convId, date?)`, `upsertEpisode(partial)` (merges sections, bumps
`updatedAt`), `listPending(limit)` (oldest-first by `updatedAt`), `markConsolidated(file)`,
`tagCandidate(file, slug)`, `archiveOldConsolidated(maxAgeDays)` → moves to
`episodes/.archive/`. Filename: `<yyyy-mm-dd>-<conversationId>.md` (FR-001); frontmatter
uses the same `buildFrontmatter`/`parseFrontmatter` helpers as `skills/store.ts`.

Watermarks: `data/memory/episodes/.watermarks.json` — `Record<conversationId,
{ watermark: string; reviewedAt: string }>`. Single JSON file, atomic write (FR-006).

## Changed: `src/lib/agent/review.ts` → `src/lib/agent/memory/fast-loop.ts`

- Keep `runToolLoop` + `hasCredentials()` structure and `maxSteps` ~12.
- System prompt = body of `specs/020 - memory-loops/prompts/fast-loop-system.md`,
  verbatim, HTML comment stripped (store as a module constant).
- Toolset: `episode_write` (section-scoped upsert) and `skill_patch` only (FR-007).
  Move `SKILL_TOOLS` out of `review.ts` into a shared `src/lib/agent/skills/llm-tools.ts`
  so fast loop imports `skill_patch` and slow loop imports the full set. Delete
  `MEMORY_LLM_TOOL` from this pass.
- Entry point `runFastLoop(conversationId)`: load conversation via VFS
  (`/Documents/Chats/<id>.json`, see `conversations-server.ts`), slice messages after
  watermark, derive `skillsUsed` from tool-call records in the transcript slice
  (`skill_load` calls) + usage telemetry timestamps (FR-008), call the LLM pass, then
  advance the watermark only on success.
- `reflectAndLearn` export stays as an alias so `/api/assistant/reflect` keeps working
  (FR-009).

## New: fast-loop scheduler job

In `src/lib/integrations/scheduler/jobs.ts`, register an internal job (default every
2 min). Eligibility scan (FR-005): list `/Documents/Chats/*.json`, compare last message
id against `.watermarks.json`; eligible when new turns ≥ 4 AND (idle ≥ 5 min by last
message timestamp OR unreviewed turns ≥ 40). Process a bounded number of conversations
per tick (suggest 3). Wrap each conversation in try/catch — one failure must not stop
the scan (daemon failure-isolation style).

## New: `src/lib/agent/memory/consolidate.ts` (server-only)

- System prompt = body of `specs/020 - memory-loops/prompts/slow-loop-system.md`, verbatim.
- Gate: `listPending()` empty → return immediately, no LLM call (FR-010).
- Lock: `data/memory/.consolidate.lock` containing a timestamp; stale after 30 min
  (FR-011). Release in `finally`.
- User message: episodes oldest-first (≤ 10), full markdown; current `MEMORY.md`; skill
  index. Tools (FR-013): `memory_add_entry`, `memory_replace_entry`, `memory_remove_entry`,
  `topic_create`, `memory_search`, `skill_list`, `skill_view`, `skill_patch`,
  `skill_create`, `episode_tag_candidate`, `episode_mark_consolidated`. `skill_create`'s
  tool description must restate the FR-014 gate. No file-write tool.
- `maxSteps`: size to batch (suggest 6 + 4 × episodes in batch).
- Register hourly scheduler job + `POST /api/memory/consolidate` (mirror the curator
  route pattern) for manual runs.

## Changed: `src/lib/agent/memory/curated.ts`

Add topic support (FR-012): `data/memory/topics/<slug>.md`, entry-list format and
budget-rejection identical to `MEMORY.md`; per-topic budget 4000 chars.
`listTopics()`, `readTopic(slug)`, `createTopic(slug, digest)` (also inserts the index
line into `MEMORY.md`), and extend `addEntry`/`replaceEntry`/`removeEntry` to take a
topic target. `memorySnapshot()` is unchanged — topics are never injected. Automated
paths must have no code path to `USER.md` (edge case in spec; SC-005).

## New: `src/lib/agent/memory/search.ts`

`searchMemory(query, opts): { file: string; entry: string; score: number }[]` over
`topics/**` and `episodes/**`. v1 ranking: case-insensitive word/substring match count.
Keep ranking in one exported function so BM25 can replace it without interface change
(FR-017). Expose as `memory_search` LLM tool + client action; extend `memory_recall`
to return a topic's entries when given a slug (FR-018) — see `MemoryActions.tsx` and
`/api/memory`.

## Config, seeding, observability

- `memoryLoops` namespace in `src/lib/config/registry.ts` (FR-019): `fastLoop.enabled`,
  `fastLoop.tickMinutes=2`, `fastLoop.idleMinutes=5`, `fastLoop.turnCap=40`,
  `slowLoop.enabled`, `slowLoop.intervalMinutes=60`, `slowLoop.batchSize=10`,
  `model` per loop (empty = default provider), `episodeArchiveDays=14`.
- Seed skill (FR-022): add `recall-long-term-memory` (bundled beside this skill) to the
  `SEED` array in `skills/store.ts` so the existing top-up path installs it on running
  systems.
- Log both loops to central logging (FR-020); surface pending-episode count + last run
  summaries in the Memory app (`src/apps/memory/index.tsx` over `/api/memory`).

## Docs to update (working rules)

`docs/dev/memory/memory.md`, `docs/dev/self-improvement/self-improvement.md` (Pass 1
trigger model changed), `docs/usage/memory/how-memory-works.md`, and note the 003
supersession in the system store's `discrepancies.md` via Build Studio.
