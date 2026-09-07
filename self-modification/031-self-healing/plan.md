# Implementation Plan: Self-Healing Mechanism

**Branch**: `031-self-healing` | **Date**: 2026-09-07 | **Spec**: `spec.md` | **Design**: `design.md`

**Input**: Feature specification from `spec.md`; architectural decisions from `design.md` (9 ADRs, verified by architect-reviewer against source).

## Summary

Add a self-healing loop to BOS: detect agent/tool failures (5 triggers, all toggleable), diagnose them with an extended `conversation-reviewer` **Diagnostician** (markdown report + scope-class verdict), and — for genuine gaps — drive the **Build Studio pipeline autonomously** (no step-boundary stops) to a TDD'd, ≥95%-coverage fix on a feature-branch preview (class e) or `app_build` result (class d-bis), then notify via a `fix_ready` event. Promotion stays human-gated; the mechanism never touches the Supervisor/base.

**Technical approach** (from `design.md`): the "fast spine" is **not** a 002 workflow — it is a deterministic **034 core headless handler** (`registerCoreExecutor`) fronting a server-only **case store**, with a **hook plugin** for error capture and `runSubAgent` for the Diagnostician and the slow-path BS conversation. The slow path is the existing `build-studio` local agent run autonomously on a pre-branch-seeded conversation (the ref is created lazily by the first `dev_delegate`). The full mechanism lives in `src/lib/self-heal/` (server-only) + two agent surfaces (Diagnostician extension, BS Self-Heal pane) + a Settings tab.

This plan does **not** restate the architecture — see `design.md` §1–§8. It fixes the implementation structure, phases, test strategy, and the open implement-step decisions (R6/R7/R8).

## Technical Context

**Language/Version**: TypeScript (BOS `src/`), Node.js (server), React (Next.js App Router) for the BS pane + Settings tab.

**Primary Dependencies**: none new. SHA-256 via `node:crypto` (already used by the event store). Builds entirely on existing BOS systems: 034 events, agent/subagent, config registry, plugin system, scheduler, Supervisor.

**Storage**:
- Case store — `data/self-heal/index.json` (warm index: in-flight slot, dedupe map, cost ledger, slow-path + cost queues) and `data/self-heal/cases/<caseId>.json` (full record + timeline). Atomic writes via `@/os/atomic-write`. Gitignored runtime dir, created at boot.
- Diagnostics reports — VFS `/Documents/BOS Improvements/<caseId>.md` (user-authored content, FR-028).
- 034 events — `data/events/` (immutable bodies + projections; the audit/notification channel, ADR-6).

**Testing**: Vitest (unit — the existing pattern, see `tests/events/`) + Playwright e2e (`e2e/031-self-healing.spec.ts`). All tests self-cleaning (spec 034/028 precedent). Coverage gate: **≥95% on `src/lib/self-heal/**`** (the feature's own subsystem), matching the FR-014 TDD mandate.

**Target Platform**: BOS server process (base) + browser (BS pane, Settings). No new network port, no worker-thread service.

**Project Type**: bos-core (mechanism spanning several subsystems) + one `builtin-app`-shaped leaf (the BS Self-Heal pane, part of the same change).

**Performance Goals**: spine intake <50ms p99 (pure in-memory store read + dedupe hash; Diagnostician launched fire-and-forget). Case-store list <200ms. No per-emit disk write beyond the atomic case update (mirrors 034's O(1) write discipline).

**Constraints**:
- The Diagnostician must be launched **fire-and-forget** from the 034 core handler or the 30s ack window times out (ADR-1, R4).
- The in-flight slot, dedupe map, and cost ledger live in the **durable store** (not in-memory) so they survive restarts and are correct under the Supervisor's multiple processes (ADR-9).
- Headless self-heal runs do **not** fire plugin hooks (`runLocalHeadless` passes no `hooks`) — re-entrancy is by construction (ADR-4).
- The cost cap is enforced **between** cases, not within one (FR-020 mid-pipeline semantics).

**Scale/Scope**: single-user desktop. One in-flight slow-path case; bounded cost queue (≤100, 7-day TTL) and slow-path FIFO. A handful of active cases at any time.

## Constitution Check

*Gated in `design.md` §2; re-confirmed here against `bos-system-specs/.specify/memory/constitution.md`.*

| Principle | Status |
|---|---|
| I. Spec-Driven | ✅ This plan is the plan artifact; tasks/implement continue the pipeline. |
| II. Server Authority & SSR Boundary | ✅ Spine, store, dedupe, cost ledger, and trigger intake are `server-only` under `src/lib/self-heal/`; the BS pane + Settings tab talk to `/api/self-heal/**` over `fetch`. No secrets to the client. |
| III. Always Delegate; Claude Codes | ✅ All *fix implementation* is delegated to the `developer` (Claude) inside the autonomous BS pipeline. The Diagnostician is `type:local` (analysis, not code). The spine is deterministic code except the one Diagnostician call. |
| IV. Minimize Blast Radius | ✅ Every fix lands on a feature-branch preview or `app_build`; promotion always explicit (FR-024); never touches Supervisor/base (FR-023). |
| V. VFS Is Not the Source | ✅ Reports are VFS user content; code changes go through the Developer on a branch. |
| VI. Specs & Docs Stay in Sync | ✅ Deliverables include `docs/dev/self-healing/self-healing.md` + `docs/usage/self-healing.md`. |
| VII. Respect Boundaries | ✅ No `package.json`/lockfile/build-config change; no new dependencies. |

**No violations. Complexity Tracking not required.**

One *scoped* relaxation, documented in `design.md` §2 + ADR-1: FR-015's "run the full BS pipeline without stopping" overrides the `build-studio` agent's default stop-at-each-step contract **for the self-heal conversation only**, via the pre-authorization brief. The `build-studio` agent definition is unchanged globally.

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/self-modification/031-self-healing/
├── spec.md              # specify (done)
├── design.md            # design (done, architect + reviewer + 1 revision)
├── plan.md              # THIS file
├── mockup.html          # UI reference (binding)
└── tasks.md             # tasks (next step)
```

Delivered runtime docs (Constitution VI):
```text
docs/dev/self-healing/self-healing.md    # how it works, for maintainers
docs/usage/self-healing.md               # user-facing: triggers, the BS page, Settings, promoting fixes
```

### Source Code (repository root)

Structure from `design.md` §4. Decision: **single bos-core change** — a new server-only `src/lib/self-heal/` library, a hook plugin, three server tools, one agent extension, a Settings tab, a BS pane, and API routes. No new service, no new port, no new dependency.

```text
src/lib/self-heal/                    # NEW — deterministic spine (server-only)
├── types.ts                          # ScopeClass, Ownership, CaseStatus, HealingCase, FailureSignature, CostLedgerEntry
├── signature.ts                      # deterministic dedupe key (FR-019, ADR-7)
├── allowlist.ts                      # env allowlist (FR-002, C2) + BOS-owned log components (R8)
├── store.ts                          # case store: data/self-heal/index.json + cases/<id>.json
├── intake.ts                         # selfHealIntake() front door + Phase-B resolver + escalation
├── diagnostician.ts                  # runSubAgent(headless) → parse report → write VFS
├── cost.ts                           # per-day ledger + capExhaustedForToday() + midnight reset
├── queue.ts                          # bounded cost queue + slow-path FIFO + suspended-timeout sweep
├── reentrancy.ts                     # selfHeal conversation marker + event-payload filter (ADR-4)
└── spine-handler.ts                  # registerCoreExecutor for self_heal.* at boot

src/plugins/self-heal/                # NEW — tool-error capture plugin (FR-002/003)
├── index.ts
└── init.ts                           # afterToolCall / onError; 300s/3-count rolling window

src/lib/assistant/tools/server/
├── self-heal.ts                      # NEW — self_heal.request, self_heal.request_decision, self_heal.complete_fix
└── diagnostics.ts                    # NEW — submit_diagnostics_report (the Diagnostician's one write)

src/components/apps/settings/
└── SelfImprovementTab.tsx            # NEW — Settings → Self Improvement (FR-027)

src/apps/build-studio/selfheal/       # NEW — BS Self-Heal pane (FR-022), mirrors build-studio/conflict/
├── SelfHealPane.tsx
├── CaseList.tsx
├── CaseDetail.tsx
├── ConsentCard.tsx
├── SuspendedCard.tsx
├── PreviewStatus.tsx
└── useSelfHealCases.ts

src/app/api/self-heal/
└── route.ts                          # NEW — GET list / GET ?caseId / POST ?op=consent|answer|dismiss|report

data/self-heal/                       # NEW — runtime store (gitignored, created at boot)

seed/agents/conversation-reviewer/
└── AGENT.md                          # MODIFY — +Mode 2, unified tools (app_list, query_events, get_event), submit_diagnostics_report

src/lib/assistant/
├── model-turn.ts                     # MODIFY — read message_delta.usage (Anthropic) / final-chunk usage (OpenAI) [ADR-5]
└── agent-loop.ts                     # MODIFY — +optional usage on TurnResult [ADR-5]

src/lib/agent/subagents/
├── types.ts                          # MODIFY — +optional usage on AgentRunResult [ADR-5]
├── runner.ts                         # MODIFY — runLocalHeadless accumulates per-turn usage [ADR-5]
└── claude-runner.ts                  # MODIFY — parse usage from result stream-json / OpenCode final event [ADR-5]

src/instrumentation.ts                # MODIFY — register spine handler + self-heal plugin + scheduled Diagnostician job at boot
src/lib/config/registry.ts            # MODIFY — +selfHeal ConfigRegistration
src/apps/settings/index.tsx           # MODIFY — map "self-improvement" → SelfImprovementTab in CUSTOM_TABS
src/apps/build-studio/index.tsx       # MODIFY — +self-heal nav pane (peer of conflict pane)
src/apps/build-studio/manifest.ts     # MODIFY — +eventHandler(s) for self_heal.fix_ready / decision_needed
src/lib/agent/capabilities-registry.ts# MODIFY — +self_heal.* / submit_diagnostics_report capabilities
src/lib/agent/tool-manifest.ts        # MODIFY — mirror the new server tools

tests/self-heal/                      # NEW — unit tests (Vitest, self-cleaning)
└── *.test.ts                         # signature, allowlist, store, intake (dedupe/cap/slot/reentrancy), cost, queue
e2e/031-self-healing.spec.ts          # NEW — Playwright (self-cleaning)
```

**Structure Decision**: a cohesive `src/lib/self-heal/` server library (single owner for the spine) + thin surface layers (API route, BS pane, Settings tab, plugin, tools). The case store is the single writer for state; 034 events are the derived audit/notification channel (ADR-6). This mirrors the established `src/lib/events/` + `src/apps/event-viewer/` split from spec 034.

## Design Notes & Open Decisions Resolved

These close out the implement-step open items from `design.md` §7:

- **R5 (cost cap default) — RESOLVED by user (2026-09-07): 1,000,000 tokens/day.** `ConfigRegistration` default + Settings tab default.
- **R6 (scheduled-Diagnostician idle detection)**: define *idle* as "last message age ≥ `idleThresholdSec`" and source the conversation list from the existing conversation store (the memory fast-loop already enumerates conversations; reuse its enumeration, not a raw `/Documents/Chats` glob). The scheduled pass is a recurring `internal`/`prompt` scheduler job (`owner: "self-heal"`, registered at boot) that, per FR-021, runs **both** Mode 1 (idle conversations) and Mode 2 (pending failure queue) in one pass. Implement in `diagnostician.ts`.
- **R7 (workflow-timeout trigger)**: the spine's handler is **tolerant of a missing 002 timeout event** — in v1, if 002 does not emit one, the `workflowTimeout` trigger simply has no source and stays inert (it's off by default anyway, per FR-006). Emitting the event is a **002-side addition tracked as a cross-spec dependency** (noted in `docs/dev/self-healing/self-healing.md`), out of scope for this bos-core change. The handler reads only `workflow id`, `node`, configured-vs-actual duration, so it works once 002 emits.
- **R8 (BOS-owned log components)**: `allowlist.ts` holds a static v1 list of owned namespaces to **include** (the spine/assistant/devharness/events/scheduler/gitfs/self-heal component prefixes) — an allowlist, not a blocklist, so unknown/third-party components are excluded by default. This is the safer default for a mechanism that can drive code changes.
- **R9 (cold-restart recovery)**: the boot-reconcile is **single-owner** via the scheduler daemon lock (the spine reuses the scheduler's `daemon-election` so exactly one process reconciles), preventing double `fix_ready`. Runs after the event kernel and Supervisor are up (boot ordering in `instrumentation.ts`).
- **R10 (pre-authorization scope)**: confirmed a per-conversation brief override — acceptable, no constitution concern.

## Test Strategy

**Unit (Vitest, `tests/self-heal/`, self-cleaning temp roots like `tests/events/`):**
- `signature.test.ts` — error_category mapping, normalization (UUID/timestamp/quoted/numeric stripping, path-prefix strip on/off), SHA-256 determinism, the relaxed explicit key.
- `allowlist.test.ts` — env allowlist (401/429/timeout/OOM/DNS in, `permission_denied` NOT in), BOS-owned log component list.
- `store.test.ts` — atomic create/update, timeline append, dedupe-map read/write, in-flight slot set/clear.
- `intake.test.ts` — the front-door decision tree: disabled→no-op, allowlist→no-op, reentrancy (selfHeal marker + event filter)→no-op, dedupe hit→suppressed event, cost-cap→queued, else→case created. Phase-B routing for each scope class (a/b/c/d/d-bis/e). Mutual exclusion: second escalation while in-flight → queued-slow.
- `cost.test.ts` — ledger append, capExhaustedForToday, midnight-UTC reset, undefined-usage estimate fallback.
- `queue.test.ts` — bounded FIFO (max 100, FIFO eviction), 7-day TTL eviction + `cost_cap_evicted`, slow-path dequeue-on-slot-free (re-entry, no dedupe re-check), suspended-timeout sweep → abandoned.

**Integration (server-only):**
- `usage-surface.test.ts` — a stubbed model turn returns usage; `TurnResult`/`AgentRunResult` carry it; the claude-runner parses it from a fixture stream-json result line. (Proves ADR-5 M1's "surface usage" actually propagates.)
- `spine-handoff.test.ts` — seed a BS conversation + set `activeFeatureBranch` → assert the field is set **before** any agent turn (the FR-015b precondition); `self_heal.complete_fix` reads a `ready` Supervisor preview state and emits `fix_ready`.

**E2E (Playwright, `e2e/031-self-healing.spec.ts`, self-cleaning, `test.describe.configure({mode:'serial'})` per spec 034/039 precedent):**
- **The acceptance test (SC-001)**: fire `self_heal.request` with the `bos_app_launch`-missing-params signature → assert the Diagnostician writes a markdown report classifying `e` with a source citation → the case escalates and a BS conversation is seeded with `activeFeatureBranch` = `bos/self-heal-<id>` → (mock the developer step to a committed `params` fix on the branch) → preview health passes → `fix_ready` event emitted with the branch name → the Event Viewer shows it. **Regression test that fails on base**: a test asserting `bos_app_launch` accepts a `file` param must fail before the fix and pass after (the user's standing requirement for bug fixes).
- **Env filter (SC-003)**: simulate a network-timeout tool error → assert zero cases, zero `case_created`.
- **Dedupe (SC-007)**: fire the same non-env signature twice within the window → one case, one `dedupe_suppressed`.
- **Kill switch (SC-006)**: `selfHeal.enabled=false` → no trigger fires.
- **Ownership (SC-004/005)**: a failure signature pointing at an item present in `data/user-apps/items/` → `d-bis` (app_build path); the same signature with no local item → `d` (notify only).
- **Consent/decision (SC-009)**: a suspended case shows the decision card on the BS page; submitting an answer emits `decision_resolved` and resumes.

**Coverage gate**: CI asserts ≥95% line+branch on `src/lib/self-heal/**`. (This is also the FR-014 mandate applied to the feature's own code — dogfooding the rule.)

## Phases (for tasks.md)

1. **P0 — Usage surfacing (ADR-5 M1)**: extend `TurnResult`/`AgentRunResult`/`model-turn.ts`/`runner.ts`/`claude-runner.ts`. Independent, unblocks the cost ledger. TDD: `usage-surface.test.ts` first.
2. **P1 — Spine core**: `types`, `signature`, `allowlist`, `store` (+ tests). The deterministic foundation.
3. **P2 — Intake + routing**: `intake` front door, Phase-B resolver, scope-class routing, mutual exclusion, reentrancy (+ tests). The decision tree.
4. **P3 — Diagnostician**: `diagnostician.ts`, `submit_diagnostics_report` tool, `conversation-reviewer/AGENT.md` Mode 2 + unified tools, skill Mode-2 method. (Read the existing agent + skill first — this is additive.)
5. **P4 — Cost + queues**: `cost.ts`, `queue.ts` (+ tests).
6. **P5 — Slow-path escalation + completion**: branch pre-creation (FR-015b), autonomous brief, `self_heal.request_decision` / `self_heal.complete_fix` tools, boot-reconcile (R9).
7. **P6 — Triggers**: `self_heal.request` tool + the `self-heal` hook plugin (hard-error + repeated-failure) + 034 core handler registration + workflow-timeout/log-event tolerance (R7/R8).
8. **P7 — Scheduled Diagnostician (FR-021, R6)**: scheduler job, idle detection, both-modes pass.
9. **P8 — Config + Settings tab**: `selfHeal` namespace + `SelfImprovementTab`.
10. **P9 — BS Self-Heal pane + API route**: `/api/self-heal`, `SelfHealPane` + components, manifest event handlers. (Reference `mockup.html` + the existing `build-studio/conflict/` pane.)
11. **P10 — Docs + e2e**: `docs/dev/self-healing/self-healing.md`, `docs/usage/self-healing.md`, the full e2e suite (incl. the `bos_app_launch` acceptance test + its failing-on-base regression test). Coverage gate.

*Dependencies*: P1 before P2; P0 before P4; P3 before P2's Phase-B (routing needs the report); P5 before P6's escalation path; P9 after P5 (the pane reads cases the spine writes). P8 is independent. P10 last.

## Assumptions

See `spec.md` → Assumptions (already reconciled for the "not a 002 workflow" finding) and `design.md` §1–§8 for the verified architectural premises. This plan adds: the 1M/day cost cap default (user-confirmed), and the R6/R7/R8 implement-step resolutions above.
