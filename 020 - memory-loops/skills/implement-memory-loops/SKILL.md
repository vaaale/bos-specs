---
name: Implement memory loops (spec 020)
description: Step-by-step procedure for implementing the episodic fast loop and consolidating slow loop from spec 020-memory-loops.
when_to_use: When delegated to implement, continue, review, or fix the memory-loops feature (spec 020) in the BOS source tree.
created_by: user
pinned: true
---

# Implement memory loops (spec 020)

Development-time skill for the developer sub-agent. The requirements live in the spec
(`specs/020 - memory-loops/020-memory-loops-spec.md`); this skill is the build procedure.
Where this skill and the spec disagree, the spec wins.

## Before writing code

1. Read the spec in full, then `docs/dev/memory/memory.md` and
   `docs/dev/self-improvement/self-improvement.md` (you are changing both subsystems).
2. Read the two normative prompts in `specs/020 - memory-loops/prompts/` —
   `fast-loop-system.md` and `slow-loop-system.md`. Embed their body text **verbatim**
   (strip the leading HTML comment). Do not paraphrase, "improve", or reflow them.
3. Read `references/code-touchpoints.md` (bundled with this skill) for per-file
   design detail: module APIs, data shapes, and the patterns to copy from existing code.
4. Work on branch `bos/memory-loops`. No new npm dependencies; never touch
   `package.json`, lockfiles, or build config.

## Milestones (each independently shippable — finish and verify one before the next)

**M1 — Episode store + fast-loop refactor (manual trigger only).**
`src/lib/agent/memory/episodes.ts` (store, FR-001..003), `src/lib/agent/memory/fast-loop.ts`
(refactor of `review.ts`; toolset = `episode_write` + `skill_patch` only, FR-007), rewire
`/api/assistant/reflect` and the `skill_reflect` action to it (FR-009). `review.ts`'s
skill tool definitions move to a shared module — the slow loop reuses them in M3.

**M2 — Hard-wiring the fast loop.**
Watermark sidecar (FR-006), eligibility scan of `/Documents/Chats` (FR-005), scheduler
job registration (FR-004). Mechanical `skillsUsed` capture (FR-008).

**M3 — Slow loop + topic-sharded LTM.**
Topic files + incremental ops in `curated.ts` (FR-012), `consolidate.ts` with the
restricted toolset (FR-013..016), hourly gated + locked scheduler job (FR-010..011),
`POST /api/memory/consolidate` for manual runs.

**M4 — Retrieval, config, seeding, docs.**
`memory/search.ts` + `memory_search` tool (FR-017) and `memory_recall` extension (FR-018);
`memoryLoops` config namespace (FR-019); logging + Memory-app surfacing (FR-020); add the
bundled `recall-long-term-memory` skill to the SEED list in `skills/store.ts` (FR-022);
update `docs/dev/**` and `discrepancies.md`.

## Verify after every milestone

- `npx tsc --noEmit` and `npm run lint` — clean.
- Do NOT run `npm run build` while `next dev` is running.
- Manual check per milestone: M1 — trigger reflect on a real conversation, inspect the
  episode file; M2 — watch the scheduler log produce an episode with no manual action,
  confirm an unchanged conversation causes no LLM call (SC-002); M3 — run consolidation
  on ≥ 2 episodes, confirm topic entries and `consolidated` status, confirm no `USER.md`
  write; M4 — `memory_search` returns entries with provenance.
- Walk the spec's Success Criteria (SC-001..006) before declaring the feature done.

## Pitfalls (from the codebase, not general advice)

- Conversation JSONs under `/Documents/Chats` are owned by the **client** store — never
  write to them from the loops; all loop state goes in the sidecar/episodes (FR-006).
- Every episode/topic write must pass `looksLikeInjection` — this text re-enters prompts.
- Both loops must no-op when `hasCredentials()` is false, like today's review.
- Scheduler jobs must not throw across the tick boundary — copy the failure-isolation
  style in `src/lib/integrations/scheduler/`.
- Keep `memorySnapshot()` frozen-per-conversation semantics untouched.
