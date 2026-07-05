# Tasks: Memory Loops (Episodic Fast Loop & Consolidating Slow Loop)

**Feature**: 021-memory-loops  
**Status**: Ready for Implementation  
**Branch**: `bos/memory-loops`

---

## Phase 0: Unified Scheduler Engine (prerequisite)

Memory Loops runs both loops as jobs on the **Unified Scheduler Engine** (spec `scheduler`, see `bos-system-specs/scheduler/plan.md` and `tasks.md`). Fast/slow loops must NOT introduce their own timer, daemon, or persistence — they register `internal` handlers on the engine and seed two `system` category jobs.

Phase 0 tasks below are the alignment points between this spec and the Scheduler spec. **All of Phase 0 depends on Scheduler Phase 1 being complete.** Phase 0 itself blocks Memory Loops Phase 2 (fast-loop scheduling) and Phase 3 (slow-loop scheduling).

### Task 0.1: Verify Scheduler Phase 1 landed
- Prerequisite check: `src/lib/scheduler/engine.ts`, `src/lib/scheduler/handlers/registry.ts`, `src/lib/scheduler/store.ts`, `src/lib/scheduler/history.ts`, `src/lib/scheduler/schedule.ts`, `src/lib/scheduler/acl.ts`, and `src/lib/scheduler/types.ts` exist and export the surface described in `scheduler/tasks.md` Tasks 1.1–1.10.
- If not, stop and complete `scheduler/tasks.md` Phase 1 first.
- **Acceptance**: engine boots with zero jobs; `resolve('internal', 'memory-loops.fast')` returns `null` (registered later in Task 0.3).

### Task 0.2: Verify Scheduler Phase 2 (Migration) landed
- Prerequisite check: `src/lib/scheduler/migrate.ts` exists and runs at boot before `engine.start()`; legacy `state.services[svcId].poll` records for existing integrations have been migrated into `/Documents/System/scheduler-jobs.json`; `schema.migratedFromLegacy` marker present.
- If not, stop and complete `scheduler/tasks.md` Phase 2 (Tasks 2.1–2.4) first.
- **Acceptance**: existing integrations continue to poll via the engine; migration is idempotent on subsequent boots.

### Task 0.3: Register internal handlers for both loops
- **File**: `src/lib/agent/memory/fast-loop.ts` (module-load side effect) and `src/lib/agent/memory/consolidate.ts` (module-load side effect).
- At module load call:
  ```ts
  import { register } from '@/lib/scheduler/handlers/registry';
  register('internal', 'memory-loops.fast', fastLoopHandler);
  register('internal', 'memory-loops.slow', slowLoopHandler);
  ```
- `fastLoopHandler` signature: `(job, ctx) => Promise<RunResult>` — calls `runFastLoop()` (defined in Task 1.3), returns `{ ok, durationMs, output: summary, error? }`.
- `slowLoopHandler` similar — calls `runSlowLoop()` (Task 3.2). It MUST short-circuit and return `{ ok: true, output: 'no pending episodes' }` when no pending episodes exist, without acquiring the LLM.
- Both handlers respect `ctx.abortSignal`.
- **Acceptance**: after boot, `resolve('internal', 'memory-loops.fast')` and `resolve('internal', 'memory-loops.slow')` both return the registered handlers.

### Task 0.4: Seed two system jobs on first boot
- **File**: `src/lib/agent/memory/seed-jobs.ts` (new).
- Exports `async function seedMemoryLoopJobs(): Promise<void>` — invoked from the memory-loops boot path AFTER handler registration.
- Idempotent: check `/Documents/System/scheduler-jobs.json` for existing jobs with `owner: 'memory-loops'` and skip creation if present.
- Seeds:
  ```
  { id: <ulid>, category: 'system', name: 'Memory: fast loop',
    handler: { kind: 'internal', id: 'memory-loops.fast' },
    inputs: {}, schedule: { type: 'recurring', interval: 2, unit: 'minute' },
    status: 'active', owner: 'memory-loops', createdBy: 'seed', ... }

  { id: <ulid>, category: 'system', name: 'Memory: slow loop',
    handler: { kind: 'internal', id: 'memory-loops.slow' },
    inputs: {}, schedule: { type: 'recurring', interval: 1, unit: 'hour' },
    status: 'active', owner: 'memory-loops', createdBy: 'seed', ... }
  ```
- Interval values must read from the `memoryLoops` config namespace (Task 2.2) if it exists; fall back to defaults above.
- **Acceptance**: after first boot, `GET /api/scheduler/jobs?category=system` returns both jobs; they are non-deletable (ACL, per `scheduler/plan.md` §Category-based ACL).

### Task 0.5: One-time migration of any prior memory-loops timers
- **File**: `src/lib/agent/memory/migrate.ts` (new).
- Exports `async function migrateLegacyMemoryLoopsState(): Promise<{ migrated: boolean; note: string }>` — called once from the memory-loops boot path.
- If the branch introduced any temporary `data/memory/.timers/` or ad-hoc persisted timer state before the unified engine landed, delete it and log the removal. This is a safety net; for a clean install the function is a no-op.
- MUST NOT touch or migrate `state.services[svcId].poll` — that migration is owned exclusively by `src/lib/scheduler/migrate.ts` (see `scheduler/tasks.md` Task 2.1).
- Records completion via a marker file (`data/memory/.migrated`) so re-runs are no-ops.
- **Acceptance**: fresh install → migration is a no-op; if any legacy timer state exists, it is removed with a log entry.

### Task 0.6: Scheduler UI shows memory-loops jobs correctly
- Not a code task in this repo — a verification checkpoint.
- Open the Scheduler app (`src/apps/scheduler/` — see `scheduler/tasks.md` Task 3.2). Confirm:
  - Both memory-loops jobs appear in the list with the `system` badge.
  - Delete button is hidden / disabled for both.
  - Name and handler fields are locked (per `getEditableFields('system')` in `src/lib/scheduler/acl.ts`).
  - Schedule field is editable; pause/resume works.
  - `POST /api/scheduler/jobs/:id/run` triggers an immediate run and appends to `/Documents/System/scheduler-history/<jobId>.jsonl`.
- **Acceptance**: manual test passes; screenshots or a walkthrough note recorded on the implementation PR.

### Task 0.7: Wire memory-loops boot sequence
- **File**: `src/lib/agent/memory/boot.ts` (new or existing memory-loops bootstrap point).
- Boot order:
  1. `migrateLegacyMemoryLoopsState()` (Task 0.5)
  2. Handler-registration side effects fire on module import (Task 0.3)
  3. `seedMemoryLoopJobs()` (Task 0.4)
- Called from the same app-boot path that invokes `SchedulerEngine.start()`, but BEFORE `start()` (so the seeded jobs are present when the engine loads them).
- **Acceptance**: cold boot on a fresh install produces both seeded jobs; engine picks them up on first tick.

**Phase 0 Deliverable**: Memory Loops has zero timers of its own. Both fast and slow loops run as scheduler jobs, appear in the Scheduler UI as `system` category, and are subject to the engine's failure isolation, history logging, and central-logging integration.

---

## Phase 1: Episode Store + Fast Loop Refactor

### Task 1.1: Create Episode Module
- [ ] **File**: `src/lib/agent/memory/episodes.ts`
- [ ] Implement `Episode` type with frontmatter fields:
  - `conversationId`, `createdAt`, `updatedAt`, `watermark`
  - `skillsUsed: string[]`, `status: 'pending' | 'consolidated'`
  - `skillCandidates: string[]` (optional)
- [ ] Implement `EpisodeBody` sections:
  - Task & outcome, What worked / what failed, Corrections received
  - Durable lesson candidates, Profile suggestions
- [ ] Implement `createEpisode(conversationId: string): Promise<Episode>`
  - Atomic write (temp file + rename) to `data/memory/episodes/<yyyy-mm-dd>-<conversationId>.md`
  - Injection scan via `looksLikeInjection()`; reject if suspicious
- [ ] Implement `updateEpisode(conversationId: string, updates: Partial<EpisodeBody>): Promise<Episode>`
  - Idempotent: one file per conversation per day
  - Preserve existing sections, merge new content
- [ ] Implement `getEpisode(conversationId: string): Promise<Episode | null>`
- [ ] Implement `markEpisodeConsolidated(conversationId: string): Promise<void>`
- [ ] Implement `archiveOldEpisodes(olderThanDays: number = 14): Promise<number>`
  - Move to `data/memory/episodes/.archive/` (never delete)
- [ ] Write unit tests: `tests/memory/episodes.test.ts`

**Acceptance**: Episodes created/updated atomically; injection-scanned; idempotent per conversation.

---

### Task 1.2: Create Watermark Persistence
- [ ] **File**: `src/lib/agent/memory/watermarks.ts`
- [ ] Implement watermark data structure: `{ [conversationId: string]: { messageId: string, reviewedAt: string } }`
- [ ] Persist to `data/memory/.watermarks.json` (atomic writes)
- [ ] Implement `getWatermark(conversationId: string): Promise<string | null>`
- [ ] Implement `setWatermark(conversationId: string, messageId: string): Promise<void>`
- [ ] Implement `resetWatermark(conversationId: string): Promise<void>`
- [ ] Add startup validation: scan for watermarks > max message index; reset to last valid

**Acceptance**: Watermarks survive restarts; no write races with client-owned conversation files.

---

### Task 1.3: Extract Fast Loop Logic
- [ ] **File**: `src/lib/agent/memory/fast-loop.ts`
- [ ] Define `FastLoopConfig` interface (tickInterval, idleThreshold, turnCap)
- [ ] Implement `scanEligibleConversations(): Promise<ConversationRef[]>`
  - Scan `/Documents/Chats/*.json` via VFS
  - Filter: messages beyond watermark AND (idle ≥ threshold OR unreviewed turns ≥ cap OR conversation closed)
  - Skip if < 4 new turns (debounce trivial exchanges)
- [ ] Implement `reviewConversation(convRef: ConversationRef): Promise<EpisodeUpdate>`
  - Extract transcript slice after watermark
  - Call LLM with restricted toolset (`episode_write`, `skill_patch` only)
  - System prompt from bundled `prompts/fast-loop-system.md` (embed verbatim as constant)
  - Capture `skillsUsed` mechanically from telemetry/transcript tool calls
- [ ] Implement `runFastLoop(): Promise<RunSummary>`
  - Process eligible conversations sequentially
  - Create/update episodes; advance watermarks
  - Log run (start, processed count, episodes created/updated, refusals)
- [ ] Remove `skill_create` from fast-loop toolset; keep only `episode_write`, `skill_patch`
- [ ] Write unit tests: `tests/memory/fast-loop.test.ts`

**Acceptance**: Fast loop reviews only new turns; writes episodes correctly; no skill creation.

---

### Task 1.4: Create Fast Loop System Prompt
- [ ] **File**: `prompts/fast-loop-system.md`
- [ ] Write system prompt with these constraints:
  - Role: "You are the fast-loop reviewer, analyzing recent conversation turns"
  - Scope: Only turns after the watermark; do not re-review old content
  - Output: Update episode sections (Task/outcome, lessons, corrections)
  - Restrictions: NO skill creation, NO writes to USER.md/MEMORY.md/topics
  - Anti-patterns: Ignore transient failures, negative tool claims, one-off narratives
  - Tool usage: `episode_write` for updates; `skill_patch` only if skill explicitly corrected
- [ ] Embed as constant in `fast-loop.ts`: `export const FAST_LOOP_SYSTEM_PROMPT = /* verbatim file content */`

**Acceptance**: Prompt embedded verbatim; LLM respects restrictions.

---

### Task 1.5: Refactor review.ts to Delegate
- [ ] **File**: `src/lib/agent/review.ts`
- [ ] Modify `runReview()` to detect automated vs manual invocation
- [ ] For automated path (called by scheduler): delegate to `fast-loop.ts` logic
- [ ] Keep `skill_reflect` as manual trigger: "run fast loop now for this conversation"
- [ ] Update documentation comments

**Acceptance**: `skill_reflect` still works manually; automated reviews use fast-loop module.

---

## Phase 2: Fast Loop Scheduler + Watermarks

### Task 2.1: Wire Fast Loop into the Unified Scheduler
- [ ] **Depends on**: Phase 0 complete (`scheduler/tasks.md` Phase 1 landed).
- [ ] Handler registration happens in Task 0.3 — this task adds the runtime pieces:
  - [ ] Ensure `fastLoopHandler` (registered via `register('internal', 'memory-loops.fast', ...)`) invokes `runFastLoop()` from `src/lib/agent/memory/fast-loop.ts` and returns a `RunResult` (`{ ok, durationMs, output, error? }`).
  - [ ] Default schedule (2 min) is seeded by Task 0.4; interval is read from `memoryLoops.fastLoop.tickInterval` at seed time (Task 2.2).
- [ ] Failure isolation is provided by the engine — this handler MUST NOT swallow errors; it re-throws (or returns `{ ok: false, error }`) so the engine can log to history and central logging.
- [ ] History for this job is at `/Documents/System/scheduler-history/<fastLoopJobId>.jsonl` (managed by the engine, not this module).

**Acceptance**: Fast loop runs every 2 min via the scheduler engine's tick loop; entries appear in the job's history JSONL; central log shows one `component: 'scheduler.handler.memory-loops.fast'` entry per run.

---

### Task 2.2: Add memoryLoops Config Namespace
- [ ] **File**: `src/lib/config/registry.ts`
- [ ] Register new namespace: `memoryLoops`
- [ ] Define fields:
  - `fastLoop.enabled` (boolean, default true)
  - `fastLoop.tickInterval` (number, default 120 seconds)
  - `fastLoop.idleThreshold` (number, default 300 seconds / 5 min)
  - `fastLoop.turnCap` (number, default 40 unreviewed turns)
  - `slowLoop.enabled` (boolean, default true)
  - `slowLoop.interval` (number, default 3600 seconds / 1 hour)
  - `slowLoop.batchSize` (number, default 10 episodes per run)
  - `modelOverride` (string, optional; per-loop model override)
  - `episodeArchiveAge` (number, default 14 days)
- [ ] Expose in Settings UI under "Memory Loops" tab
- [ ] Add agent tools for reading/updating config

**Acceptance**: Config visible in Settings; agent can read/update via tools.

---

### Task 2.3: Update Discrepancies Documentation
- [ ] **File**: `specs/discrepancies.md`
- [ ] Document divergence from spec 003:
  - 003 used voluntary `skill_reflect` trigger; 021 uses automated scheduler jobs
  - 021 introduces episodic store (episodes/) as buffer before consolidation
  - 021 adds topic sharding for long-term memory growth beyond MEMORY.md budget
- [ ] Note that GEPA (Pass 2) and Curator (Pass 3) remain unchanged

**Acceptance**: Spec drift documented; developers understand the model change.

---

## Phase 3: Slow Loop + Topics

### Task 3.1: Create Topics Module
- [ ] **File**: `src/lib/agent/memory/topics.ts`
- [ ] Implement `TopicEntry` type (timestamped bullet-listed content)
- [ ] Implement `getOrCreateTopic(slug: string): Promise<Topic>`
  - Create `data/memory/topics/<slug>.md` if not exists
  - Parse existing entries from file
- [ ] Implement `addTopicEntry(topicSlug: string, entry: TopicEntry): Promise<void>`
  - Append to topic file; enforce budget (default 4000 chars)
  - Rejection + fallback to new shard if budget exceeded
- [ ] Implement `replaceTopicEntry(topicSlug: string, entryId: string, newContent: string): Promise<void>`
  - Supersession semantics: mark old entry as superseded, add new
- [ ] Implement `removeTopicEntry(topicSlug: string, entryId: string): Promise<void>`
- [ ] Implement `updateMemoryIndex(slug: string, digest: string): Promise<void>`
  - Add/update one-line entry in `MEMORY.md` (budget enforced)
- [ ] Write unit tests: `tests/memory/topics.test.ts`

**Acceptance**: Topic files grow incrementally; no full rewrites; budget enforced.

---

### Task 3.2: Create Consolidate Module
- [ ] **File**: `src/lib/agent/memory/consolidate.ts`
- [ ] Define `ConsolidateConfig` interface (interval, batchSize)
- [ ] Implement `acquireLock(): Promise<Lock | null>`
  - Create `data/memory/.consolidate.lock` with pid/start time
  - Check for stale locks (>30 min); expire if needed
  - Return null if lock held by another process
- [ ] Implement `releaseLock(lock: Lock): Promise<void>`
- [ ] Implement `loadPendingEpisodes(batchSize: number): Promise<Episode[]>`
  - Oldest-first ordering; batch-limited
- [ ] Implement `consolidateEpisode(episode: Episode): Promise<ConsolidationResult>`
  - Call LLM with restricted toolset (memory_add_entry, memory_replace_entry, topic_create, skill_patch, skill_create gated)
  - System prompt from bundled `prompts/slow-loop-system.md` (embed verbatim)
  - Process ops incrementally; mark episode consolidated only after success
- [ ] Implement `runSlowLoop(): Promise<RunSummary>`
  - Acquire lock; exit if none available
  - Process pending episodes in batch; log each op
  - Release lock; archive old episodes
- [ ] Write unit tests: `tests/memory/consolidate.test.ts`

**Acceptance**: Slow loop runs hourly; processes episodes oldest-first; lock prevents overlap.

---

### Task 3.3: Create Slow Loop System Prompt
- [ ] **File**: `prompts/slow-loop-system.md`
- [ ] Write system prompt with these constraints:
  - Role: "You are the consolidation engine, merging episodic memories into long-term knowledge"
  - Input: Pending episode(s) with task/outcome/lessons
  - Output: Incremental ops only (`memory_add_entry`, `topic_create`, `skill_patch`, gated `skill_create`)
  - Skill creation gate (FR-014): Require recurrence evidence (≥ 2 episodes) + complexity threshold
  - Anti-patterns: Never harden transient failures, negative claims, one-off narratives
  - Deduplication: Supersede contradictory entries; do not append duplicates
- [ ] Embed as constant in `consolidate.ts`: `export const SLOW_LOOP_SYSTEM_PROMPT = /* verbatim file content */`

**Acceptance**: Prompt embedded verbatim; LLM respects incremental ops and skill gate.

---

### Task 3.4: Wire Slow Loop into the Unified Scheduler
- [ ] **Depends on**: Phase 0 complete (`scheduler/tasks.md` Phase 1 landed).
- [ ] Handler registration happens in Task 0.3 — this task adds the runtime pieces:
  - [ ] Ensure `slowLoopHandler` (registered via `register('internal', 'memory-loops.slow', ...)`) invokes `runSlowLoop()` from `src/lib/agent/memory/consolidate.ts` and returns a `RunResult`.
  - [ ] Handler MUST short-circuit and return `{ ok: true, output: 'no pending episodes' }` when no `pending` episodes exist — zero LLM cost when idle (SC-003).
  - [ ] Default schedule (1 hour) is seeded by Task 0.4; interval is read from `memoryLoops.slowLoop.interval` at seed time (Task 2.2).
- [ ] Overlap lock (`data/memory/.consolidate.lock`, 30 min staleness) remains internal to `consolidate.ts` (Task 3.2) — the engine's own dispatch does not serialize handlers across ticks, so this lock is still required to prevent a manual `runNow` colliding with a scheduled tick.
- [ ] Failure isolation is provided by the engine — this handler MUST NOT swallow errors; return `{ ok: false, error }` so the engine records history + central log.
- [ ] History for this job is at `/Documents/System/scheduler-history/<slowLoopJobId>.jsonl` (managed by the engine).

**Acceptance**: Slow loop runs hourly via the engine; zero LLM cost when no pending episodes; overlap lock prevents manual + scheduled collision; entries appear in the job's history JSONL.

---

### Task 3.5: Implement Skill Creation Gate
- [ ] **File**: `src/lib/agent/memory/consolidate.ts` (extend)
- [ ] Before allowing `skill_create`, validate all three conditions (FR-014):
  1. **No existing skill**: Call `skill_list()` and search for matching task class
  2. **Complexity threshold**: Check episode for multi-step, non-obvious ordering, or discovered pitfalls
  3. **Recurrence evidence**: Search episode files for matching `skill-candidate` tags (≥ 2 occurrences)
- [ ] If any condition fails: reject skill creation; log reason; record `skill-candidate` tag on episode instead
- [ ] Add helper: `searchSkillCandidates(taskClass: string): Promise<number>` (count matching episodes)

**Acceptance**: No skill created from single occurrence; recurrence evidence required.

---

## Phase 4: Search + Config + Docs

### Task 4.1: Create Memory Search Module
- [ ] **File**: `src/lib/agent/memory/search.ts`
- [ ] Implement `memory_search(query: string, maxResults: number = 10): Promise<SearchResult[]>`
  - Scan `data/memory/topics/**/*.md` and `data/memory/episodes/**/*.md`
  - Case-insensitive word match; rank by match count
  - Return provenance: `{ source: "topics/<slug>.md#entry-3", content: "...", score: number }`
- [ ] Isolate ranking logic for future BM25 swap (no interface change)
- [ ] No new dependencies; substring/word match only
- [ ] Write unit tests: `tests/memory/search.test.ts`

**Acceptance**: Search returns matching entries with provenance; ranked by relevance.

---

### Task 4.2: Extend memory_recall for Topics
- [ ] **File**: `src/lib/agent/memory/curated.ts` (modify)
- [ ] Extend `memory_recall(slug?: string)` to handle topic slugs
- [ ] If slug provided: return entries from `data/memory/topics/<slug>.md`
- [ ] If no slug: existing behavior (global memory)

**Acceptance**: Topic retrieval via `memory_recall("gmail-workflows")` works.

---

### Task 4.3: Create Recall Long-Term Memory Skill
- [ ] **File**: `skills/recall-long-term-memory/SKILL.md`
- [ ] Write skill teaching assistant to use `memory_search` and `memory_recall`
- [ ] Include examples: "Search for lessons about Gmail workflows", "Recall the gmail-workflows topic"
- [ ] Add to SEED list in `src/lib/agent/skills/store.ts` (`created_by: seed`)

**Acceptance**: Skill seeded; fresh installs include it; existing installs can add via Build Studio.

---

### Task 4.4: Create Implement Memory Loops Skill (Dev-Time Only)
- [ ] **File**: `skills/implement-memory-loops/SKILL.md`
- [ ] Write build procedure for developer sub-agent
- [ ] Include references to code touchpoints (`references/code-touchpoints.md`)
- [ ] List per-file design details (atomic writes, watermark strategy, lock file format)
- [ ] Install into `data/skills/` during implementation; NOT seeded at runtime

**Acceptance**: Developer has detailed build procedure; spec is source of truth if conflicts.

---

### Task 4.5: Update Documentation
- [ ] **File**: `docs/dev/memory/memory.md`
  - Document episodic store (episodes/, watermarks)
  - Explain fast loop vs slow loop roles
  - Show topic sharding strategy and budget enforcement
- [ ] **File**: `docs/dev/self-improvement/self-improvement.md`
  - Note trigger-model change: voluntary → automated scheduler jobs
  - Document skill creation gate (FR-014)
- [ ] **File**: `docs/usage/memory.md` (user-facing, if exists)
  - Explain automatic reflection; no user action required
  - Describe what gets saved (lessons, corrections) vs what doesn't (transient failures)

**Acceptance**: Docs reflect new architecture; users understand automation.

---

### Task 4.6: Create API Endpoints
- [ ] **File**: `src/api/memory.ts` (or extend existing)
- [ ] `POST /api/memory/consolidate`: Manual slow-loop trigger (for debugging/testing)
- [ ] Extend `POST /api/assistant/reflect`: Manual fast-loop trigger for specific conversation
- [ ] Both endpoints log to central logging; return run summary

**Acceptance**: Manual triggers available for testing; mirror Curator on-demand pattern.

---

## Testing & Verification

### Integration Tests
- [ ] **File**: `tests/memory-loops/fast-loop-integration.test.ts`
  - Fast loop runs on scheduler tick; produces episode within 2×tick interval
  - Watermark advances correctly; re-review is idempotent
- [ ] **File**: `tests/memory-loops/slow-loop-integration.test.ts`
  - Slow loop processes pending episodes; marks them consolidated
  - Topic files updated incrementally; skills patched/created per gate
- [ ] **File**: `tests/memory-loops/crash-recovery.test.ts`
  - Stale lock expires; half-consolidated episodes reprocessed correctly

### Acceptance Scenario Validation
Validate all User Stories from spec:
- [ ] Story 1: Reflection happens without user action (fast loop → episode → consolidation)
- [ ] Story 2: Long conversations reviewed in bounded chunks (turn cap)
- [ ] Story 3: Skills evolve conservatively (gate enforcement)
- [ ] Story 4: LTM grows beyond injected budget (topic sharding)

### Success Criteria Verification
- [ ] SC-001: 100% of conversations with ≥ 4 turns have episode within threshold + 2×tick
- [ ] SC-002: Unchanged conversations trigger zero LLM calls; idempotent reviews
- [ ] SC-003: Pending episodes consolidated within interval + run duration; idle = zero cost
- [ ] SC-004: No skill from single occurrence; skills used are re-examined
- [ ] SC-005: Topics never shrink by more than one entry; no full rewrites
- [ ] SC-006: 003 anti-patterns enforced across both loops

---

## Implementation Order (Recommended)

**Phase 0 (Tasks 0.1–0.7): Scheduler prerequisite** — MUST complete `scheduler/tasks.md` Phase 1 first, then land Phase 0 here (handler registration + system-job seeding). Blocks all subsequent phases.

1. **Phase 1** (Tasks 1.1–1.5): Episode store + fast-loop refactor → Manual trigger works
2. **Phase 2** (Task 2.1–2.3): Fast loop wired to engine + config + docs update → Fast loop runs automatically as a scheduler `system` job (**MVP shippable**)
3. **Phase 3** (Tasks 3.1–3.5): Slow loop wired to engine + topics + skill gate → Full consolidation pipeline
4. **Phase 4** (Tasks 4.1–4.6): Search + docs + API → Feature complete

Each phase is independently testable and shippable.

### Dependency Chain (explicit)

```
scheduler Phase 1 (Engine Core)  ──►  021 Phase 0 (register + seed)
                                       │
                                       ├──►  021 Phase 1 (episodes) — independent
                                       │
                                       └──►  021 Phase 2 (fast loop via engine)
                                             └──►  021 Phase 3 (slow loop via engine)
                                                   └──►  021 Phase 4 (search + docs)
```

The Scheduler UI showing Memory Loops jobs correctly (Task 0.6) depends on `scheduler/tasks.md` Phase 3 (UI + API) — but that dependency only affects the *verification* checkpoint, not the runtime behavior. The loops will run as jobs as soon as Scheduler Phase 1 + 021 Phase 0 are complete.

---

## Notes for Developer

- **Atomic writes**: Use temp-file + rename pattern from `curated.ts`; all episode/topic ops must follow this
- **Injection safety**: Every write to episodes/topics/MEMORY.md scanned via `looksLikeInjection()`; refused content logged and dropped
- **No npm dependencies**: Search uses substring/word match; ranking logic isolated for future BM25 swap
- **Feature branch**: `bos/memory-loops` (confirm active before starting)
- **TypeScript**: `npx tsc --noEmit` clean required; do not run `npm run build` while `next dev` is live
- **Prompts are normative**: Embed system prompts verbatim from bundled files; any wording change requires spec update first
