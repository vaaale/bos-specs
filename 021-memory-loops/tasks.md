# Tasks: Memory Loops (Episodic Fast Loop & Consolidating Slow Loop)

**Feature**: 021-memory-loops  
**Status**: Ready for Implementation  
**Branch**: `bos/memory-loops`

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

### Task 2.1: Create Fast Loop Scheduler Job
- [ ] **File**: `src/lib/integrations/scheduler/jobs/memory-fast-loop.ts`
- [ ] Register job with scheduler daemon
- [ ] Default interval: 2 minutes (configurable via `memoryLoops.fastLoop.tickInterval`)
- [ ] Job handler calls `runFastLoop()` from `fast-loop.ts`
- [ ] Respect existing failure isolation (no blocking other jobs)
- [ ] Log job start/complete/failure to central logging

**Acceptance**: Fast loop runs every 2 min automatically; no manual trigger needed.

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

### Task 3.4: Create Slow Loop Scheduler Job
- [ ] **File**: `src/lib/integrations/scheduler/jobs/memory-slow-loop.ts`
- [ ] Register job with scheduler daemon
- [ ] Default interval: 1 hour (configurable via `memoryLoops.slowLoop.interval`)
- [ ] Job handler calls `runSlowLoop()` from `consolidate.ts`
- [ ] Exit immediately if no pending episodes (zero LLM cost when idle)
- [ ] Respect overlap lock; log lock contention
- [ ] Log job start/complete/failure to central logging

**Acceptance**: Slow loop runs hourly; zero cost when no pending episodes; lock prevents overlap.

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

1. **Phase 1** (Tasks 1.1–1.5): Episode store + fast-loop refactor → Manual trigger works
2. **Phase 2** (Tasks 2.1–2.3): Scheduler job + watermarks → Fast loop runs automatically (**MVP shippable**)
3. **Phase 3** (Tasks 3.1–3.5): Slow loop + topics + skill gate → Full consolidation pipeline
4. **Phase 4** (Tasks 4.1–4.6): Search + docs + API → Feature complete

Each phase is independently testable and shippable.

---

## Notes for Developer

- **Atomic writes**: Use temp-file + rename pattern from `curated.ts`; all episode/topic ops must follow this
- **Injection safety**: Every write to episodes/topics/MEMORY.md scanned via `looksLikeInjection()`; refused content logged and dropped
- **No npm dependencies**: Search uses substring/word match; ranking logic isolated for future BM25 swap
- **Feature branch**: `bos/memory-loops` (confirm active before starting)
- **TypeScript**: `npx tsc --noEmit` clean required; do not run `npm run build` while `next dev` is live
- **Prompts are normative**: Embed system prompts verbatim from bundled files; any wording change requires spec update first
