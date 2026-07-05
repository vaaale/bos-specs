# Implementation Plan: Memory Loops (Episodic Fast Loop & Consolidating Slow Loop)

**Feature**: 021-memory-loops  
**Spec Status**: Draft → Ready for Planning  
**Target Branch**: `bos/memory-loops`

---

## Technical Context

This feature implements two automated scheduler jobs that replace the voluntary `skill_reflect` trigger model from spec 003:

1. **Fast Loop** (every ~2 min): Scans idle conversations, reviews new turns, writes/updates episode files
2. **Slow Loop** (hourly): Consolidates pending episodes into long-term memory topics and patches/creates skills

### Existing Touchpoints

| Component | Path | Role in This Feature |
|-----------|------|---------------------|
| Memory store | `src/lib/agent/memory/curated.ts` | Atomic writes, injection scanning, budget enforcement |
| Review loop | `src/lib/agent/review.ts` | Refactored: fast-loop logic extracted, toolset restricted |
| Scheduler | `src/lib/integrations/scheduler/jobs.ts` | Two new internal jobs registered here |
| Conversations | `src/lib/agent/conversations-server.ts` | VFS access to `/Documents/Chats/<id>.json` |
| Skills store | `src/lib/agent/skills/store.ts` | SEED list extended with `recall-long-term-memory` |
| Config registry | `src/lib/config/registry.ts` | New `memoryLoops` namespace |
| Logging | Central logging facility (spec 017) | Both loops log runs, ops, refusals |

### New Modules to Create

```
src/lib/agent/memory/
├── episodes.ts           # Episode CRUD, watermark persistence, atomic writes
├── fast-loop.ts          # Fast loop logic (scan, review, write episode)
├── consolidate.ts        # Slow loop logic (merge episodes → topics/skills)
├── topics.ts             # Topic file management (create, add/replace/remove entry)
└── search.ts             # memory_search implementation (substring match + ranking)

src/lib/integrations/scheduler/jobs/
├── memory-fast-loop.ts   # Scheduler job wrapper for fast loop
└── memory-slow-loop.ts   # Scheduler job wrapper for slow loop

prompts/
├── fast-loop-system.md   # Normative system prompt (FR-021)
└── slow-loop-system.md   # Normative system prompt (FR-021)

skills/
├── implement-memory-loops/
│   ├── SKILL.md          # Developer build procedure (FR-023)
│   └── references/
│       └── code-touchpoints.md  # Per-file design detail
└── recall-long-term-memory/
    └── SKILL.md          # Runtime skill, seeded (FR-022)

data/memory/
├── episodes/             # Episode files: <yyyy-mm-dd>-<conversationId>.md
├── topics/               # Topic shards: <slug>.md
├── .watermarks.json      # Per-conversation review pointers
└── .archive/             # Consolidated episodes >14 days old
```

---

## Constitution Check

Against `.specify/memory/constitution.md` (principles from spec 001):

| Principle | Compliance |
|-----------|------------|
| **Specs before code** | ✅ This plan derives from the approved spec; implementation delegates to Developer |
| **No npm dependencies** | ✅ Search uses substring/ranking; no vector/embedding libs |
| **Atomic writes** | ✅ Episode/topic ops use temp-file + rename (pattern from `curated.ts`) |
| **Injection safety** | ✅ All writes scanned via existing `looksLikeInjection` |
| **Incremental ops only** | ✅ ACE anti-collapse: no full rewrites of topics/memory files |
| **User-profile protection** | ✅ Automated loops never write `USER.md`; profile suggestions recorded in episodes |
| **Negative claims rejected** | ✅ FR-004 anti-patterns bind both loops (transient failures, one-offs) |

---

## Project Structure (Real Paths)

### Phase 1: Episode Store + Fast Loop Refactor
```
src/lib/agent/memory/episodes.ts          # NEW
src/lib/agent/memory/fast-loop.ts         # NEW (extracts from review.ts)
src/lib/agent/review.ts                   # MODIFY (delegates to fast-loop for automated path)
prompts/fast-loop-system.md               # NEW
```

**Deliverable**: Manual trigger via `skill_reflect` works; episodes written correctly.

### Phase 2: Fast Loop Scheduler + Watermarks
```
src/lib/agent/memory/fast-loop.ts         # ADD watermark logic
src/lib/agent/memory/watermarks.ts        # NEW (sidecar persistence)
src/lib/integrations/scheduler/jobs/memory-fast-loop.ts  # NEW
src/lib/config/registry.ts                # MODIFY (add memoryLoops namespace)
```

**Deliverable**: Fast loop runs every 2 min automatically; watermarks survive restarts.

### Phase 3: Slow Loop + Topics
```
src/lib/agent/memory/consolidate.ts       # NEW
src/lib/agent/memory/topics.ts            # NEW
src/lib/integrations/scheduler/jobs/memory-slow-loop.ts  # NEW
prompts/slow-loop-system.md               # NEW
```

**Deliverable**: Episodes consolidated into topics; skills patched/created per gate.

### Phase 4: Search + Config + Docs
```
src/lib/agent/memory/search.ts            # NEW
docs/dev/memory/memory.md                 # UPDATE (document loops)
docs/dev/self-improvement/self-improvement.md  # UPDATE (note trigger-model change)
skills/recall-long-term-memory/SKILL.md   # NEW → SEED list
skills/implement-memory-loops/            # NEW (dev-time only, not seeded)
specs/discrepancies.md                    # UPDATE (003 vs 021 divergence)
```

**Deliverable**: Full feature operational; documentation updated.

---

## Design Notes

### Watermark Strategy (FR-006)
Watermarks MUST live in a sidecar file (`data/memory/.watermarks.json`) rather than the conversation JSON, because:
- The client owns `/Documents/Chats/<id>.json` and may overwrite it
- A write race could lose review progress
- Sidecar is agent-owned; concurrent fast-loop ticks serialize via watermark read-modify-write

### Episode File Naming (FR-001)
Format: `<yyyy-mm-dd>-<conversationId>.md`  
Rationale: One episode per conversation per day. If a conversation spans multiple days, the next day's review updates the same file (watermark advances), not a new file. Archive move happens at consolidation time.

### Topic Sharding (FR-012)
Per-topic budget: 4000 chars (configurable via `memoryLoops.topicBudget`)  
Index in `MEMORY.md`: One line per topic (`- <slug>: <one-line digest>`)  
Entry format: Identical to existing `MEMORY.md` entries (timestamped, bullet-listed)

### Skill Creation Gate (FR-014)
Three conditions ALL required:
1. **No existing skill** covers the task class (`skill_list` search first)
2. **Complexity threshold**: Multi-step, non-obvious ordering, or discovered pitfalls
3. **Recurrence evidence**: Same `skill-candidate` tag in ≥ 2 episodes

First occurrence → episode records `skillCandidates` tag  
Second occurrence → slow loop creates class-level skill (if conditions 1 & 2 also met)

### Overlap Lock (FR-011)
Slow-loop lock file: `data/memory/.consolidate.lock`  
Staleness expiry: 30 min (prevents wedging if daemon crashes)  
Lock content: `{ pid, startedAt, batchId }`

### Memory Search (FR-017)
Initial implementation: Case-insensitive word match + match-count ranking  
No new dependencies; ranking logic isolated for future BM25 swap  
Provenance returned: `source: "topics/<slug>.md#entry-3"` or `source: "episodes/2026-07-05-abc123.md#lessons"`

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Fast loop fires too often, causing LLM cost | Debounce: skip if < 4 new turns (FR-005); configurable idle threshold |
| Slow loop processes corrupted episode | Validate episode schema before consolidation; skip malformed with log entry |
| Topic file exceeds budget | Rejection + fallback to creating new topic shard (e.g., `gmail-workflows-2.md`) |
| Skill patch corrupts existing skill | Atomic write + rollback on validation failure; patch diff logged |
| Watermark desyncs from actual messages | On startup, scan for watermarks > max message index; reset to last valid |

---

## Testing Strategy

### Unit Tests
- `episodes.test.ts`: Atomic writes, injection scanning, watermark persistence
- `fast-loop.test.ts`: Eligibility logic, watermark advancement, episode update idempotency
- `consolidate.test.ts`: Topic entry ops, skill patch/creation gate, deduplication
- `search.test.ts`: Substring match, ranking, provenance format

### Integration Tests
- Fast loop runs on scheduler tick; produces episode within 2×tick interval
- Slow loop processes pending episodes; marks them consolidated; topics updated
- Crash recovery: stale lock expires; half-consolidated episodes reprocessed correctly

### Acceptance Scenarios (from spec)
All User Stories 1–4 and Edge Cases validated via end-to-end test suite in `tests/memory-loops/`

---

## Dependencies & Ordering

| Step | Depends On | Blocks |
|------|------------|--------|
| Phase 1 (episodes + fast-loop refactor) | None | Phase 2 |
| Phase 2 (scheduler job + watermarks) | Phase 1 | None (shippable MVP) |
| Phase 3 (slow loop + topics) | Phase 1 | Phase 4 |
| Phase 4 (search + config + docs) | Phase 3 | Feature complete |

**MVP Shippable After Phase 2**: Fast loop runs automatically, writes episodes. Slow loop can be manually triggered via API for initial testing.

---

## Open Questions

1. **Idle threshold default**: Spec says 5 min; confirm this is appropriate for typical conversation cadence?
2. **Episode archive age**: Spec says 14 days; should this be configurable per user workflow volume?
3. **Topic sharding strategy**: When does a topic split into `_2`? Fixed count (e.g., 5 shards) or budget-based?

These are answered in the `clarify` step if needed; otherwise defaults apply as written in spec.
