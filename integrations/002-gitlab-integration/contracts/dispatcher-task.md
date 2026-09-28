# Contract: Handler → Agent Dispatch (task + failure detection)

The contract of the **thin dispatcher** — how the `gitlab-issue-triage` handler hands a new issue to the configured agent, and how it detects failure **post-run** (the v5 blocker fix). Source: [`src/lib/integrations/services/gitlab/triage/dispatch.ts`](../../../src/lib/integrations/services/gitlab/triage/dispatch.ts) (new).

## Entry

`dispatchToAgent(record)` is launched **fire-and-forget** from the core handler executor (`void dispatchToAgent(record)`). The executor returns immediately (prompt-ack, ADR-3). `record` is the dispatched event record — one of the two `gitlab.issue.*` types the handler's prefix subscription matches.

| `record.type` | Means | Task shape |
|---|---|---|
| `gitlab.issue.opened` | a new issue | **fresh** (§2a) — start the pipeline |
| `gitlab.issue.noted` | a human (or the agent) commented | **resume** (§2b) — pick up from the issue's current status, or self-terminate |
| any other `gitlab.issue.*` | nothing this dispatcher handles | **none** — no run, **no failure event** |

The last row is not defensive padding: a prefix subscription means future `gitlab.issue.*` types arrive here, and running the fresh pipeline on, say, a `gitlab.issue.closed` would restart finished work and re-comment on it. `taskForRecord(record)` returning `undefined` is the single signal, and the handler's executor consults the same predicate (`isTriageEventType`) so its ack says `dispatched: false` rather than claiming a run.

## Step 1 — Resolve the agent

```
agentId = record/... no — from persisted config:
agentId = (await readState("gitlab")).services["webhooks"].config.triageAgentId ?? "gitlab-issue-agent"
agent   = getAgent(agentId)
```

- **`agent` is undefined** → emit `com.bos.gitlab.triage.failed` with `reason: "agent_not_found"`, `agentId`, and the issue id. **Stop — no run.** (The original `gitlab.issue.opened` event is already durably recorded; nothing is lost.)

## Step 2a — Build the FRESH task (`gitlab.issue.opened`)

A short, **context-bearing** instruction that names the skill and supplies the entry context. The skill (loaded by the agent) carries the pipeline logic; the task only says *what to start from*. Shape:

```
A new GitLab issue was opened. Use the `gitlab-issue-triage` skill to work it
through the pipeline.

Context:
- project: #<projectId> (<path_with_namespace, if available>)
- issue:   #<iid> "<title>"
- url:     <url>
- labels:  <labels joined>
- author:  <author.name> (@<author.username>)
- description:
<description, truncated>

Read the current issue via the GitLab MCP server, classify it, run the matching
workflow, and update it with comments and status labels as you go. Never close
the issue or open a merge request — end at the `Needs Review` status label.
```

The `projectId`, `iid`, `title`, `labels`, `author`, `url`, and `description` come from the event `payload` (see [bos-event.md](./bos-event.md)). The trailing line restates the Q2 autonomy boundary (the skill also enforces it — defense in depth).

## Step 2b — Build the RESUME task (`gitlab.issue.noted`, R7)

```
A human commented on GitLab issue #<iid> in project #<projectId> (<url>).
The note (by @<author.username>): <note text>

Resume the `gitlab-issue-triage` pipeline for this issue. Read the issue and its
notes via the GitLab MCP server. Determine your OWN GitLab identity from the MCP
server (the token's user). Then:
- If the newest note is from a HUMAN (not you): resume from the issue's CURRENT
  status label and address their comment — if you were at `Needs Input`, they
  answered your question, so continue from where you stopped; if at `Needs Review`,
  revise your deliverable per their feedback and re-post it (keep `Needs Review`).
  Update the status label as you go. Post exactly one reply.
- If the newest note is YOUR OWN, or there is no new human input: do NOTHING —
  post no comment, change no label, and stop. (This is how the loop self-terminates.)
Never close the issue or open a merge request — the boundary still applies.
```

`projectId`, `iid`, `note`, `author` and `url` come from the note payload ([bos-event.md](./bos-event.md)). The note text is already truncated by the receiver.

Three lines here are load-bearing, and none of them can become dispatcher code:

- **"Determine your OWN GitLab identity from the MCP server."** The agent is re-dispatched for its own comments as well as a human's, because the receiver cannot tell them apart and BOS holds no GitLab credential with which to try (R1/R4). The only party that can answer is the one holding the MCP token. The dispatcher supplies no identity and learns none.
- **"do NOTHING — post no comment, change no label, and stop."** This is the whole of the self-termination. Without it the feature oscillates.
- **"Post exactly one reply."** A run that posts two comments buys two more dispatches. One keeps the cost at one no-op per reply.

The entry context is a **pointer**, not truth — notes may have arrived since the hook fired, so the skill re-reads the live thread.

## The self-terminating cost

| Situation | Dispatches |
|---|---|
| A new issue, worked end to end with N agent comments | 1 fresh run + **N** no-op resume runs |
| A human replies once at `Needs Review` | 1 resume run (revises + re-posts) + **1** no-op for that reply |

A no-op resume is a few read-only MCP calls and no model-visible work beyond one decision. It writes nothing, so it generates no further note, and the chain ends there.

## Step 3 — Run

```
const events: ToolResult[] = []
const run = await runSubAgent(agent, task, {
  contentOnly: true,            // inert for a local agent (Claude-harness-only); kept for precedent consistency
  onEvent: (e) => { if (e.type === "tool_result") events.push(e) },   // collect outcomes
})
```

- **No `conversationId`** (genuinely headless — the deliverable is a GitLab comment, not a branch-gated spec write).
- `runSubAgent` **resolves the stored agent by id and composes its instructions** (system prompt + skills index including `gitlab-issue-triage` + MCP servers index scoped to its `mcp` allowlist).
- `run.endedReason` ∈ `{ completed, aborted, error }`; `run.error?: string`.

## Step 4 — Post-run, outcome-based failure detection

> **The v5 blocker:** a `try/catch` around `runSubAgent` does **not** work for the no-MCP case. The MCP gateway is in-band (a missing/unreachable server yields an `Error: …` *tool-result string*, not a throw) and the agent loop "never dies of tools," so `runSubAgent` **returns a clean `completed` with `error: undefined`** even when the agent had no GitLab MCP access. Failure must be detected from the **run's outcomes**, not from a throw.

| Condition | Detection | Action |
|---|---|---|
| Agent missing | Step 1: `getAgent` → undefined | emit `com.bos.gitlab.triage.failed` `reason:"agent_not_found"`; no run |
| Model/provider failure | `run.error` set, **or** `run.endedReason === "error"` | emit `com.bos.gitlab.triage.failed` `reason:"run_error"`, with `run.error` |
| **Empty run** | the run finished but **zero `tool_result` events** were collected | emit `com.bos.gitlab.triage.failed` `reason:"no_output"`, `cause:"The agent run produced no tool calls — the run ended with no recorded activity"` |
| **GitLab never attempted** | tool results exist, but **none has `name === "mcp_tool_call"`** | emit `com.bos.gitlab.triage.failed` `reason:"mcp_not_attempted"`, `cause` naming the call count and the agent's `mcp` allowlist |
| **GitLab refused** | `mcp_tool_call` results exist and **none has `ok:true`** | emit `com.bos.gitlab.triage.failed` `reason:"mcp_rejected"`, `cause` = the **first failed call's own error text** (capped at 300 chars) |
| Agent reached GitLab | **at least one `mcp_tool_call` `ok:true`** | success path — any mid-pipeline problem is the **skill's** responsibility (it posts a `Failed` status label). The dispatcher does not second-guess a run that demonstrably touched GitLab. |

**A no-op resume is a SUCCESS** (R7), and no rule above had to change for that. The commonest resume run writes nothing — the agent sees the newest note is its own and stops. It still READS GitLab (identity, the issue, its notes), so it produces successful `mcp_tool_call`s and lands on the last row. Treating "posted nothing" as a failure would raise one spurious `com.bos.gitlab.triage.failed` per comment the agent has ever posted. Zero successful MCP calls still means what it always meant: the run could not reach GitLab at all, not even to read.

The corollary is that the dispatcher **cannot** distinguish a deliberate no-op from a run that read the issue and gave up — and must not try. Once a run has touched GitLab, what it did there is the skill's to report.

The three no-GitLab reasons are one partition of the single collected `tool_result` list (total / `mcp_tool_call` / `ok`), checked shortest-run-first. They are separate because **a reason must state what was observed, not what it implies**: a single `no_gitlab_mcp` reported "the agent has no usable GitLab MCP server" for runs that never called one (issue #31), sending the operator to debug a healthy server.

`ok` is derived from the `tool_result` event's result: an `Error:`-prefixed / error payload ⇒ `ok:false`, otherwise `ok:true`. (The runner's `onEvent` already emits `{ type:"tool_result", name, result, ok }` per ADR-13 — collect `name === "mcp_tool_call"`.)

**Failure event** `com.bos.gitlab.triage.failed`:
```
{
  reason: "agent_not_found" | "run_error" | "no_output" | "mcp_not_attempted" | "mcp_rejected",
  projectId, iid, url,          // from the issue context
  cause?: string                // run.error / short detail
}
```
This is the deterministic surface for FR-016 / SC-007 (no MCP ⇒ graceful, recorded, non-crashing, issue unmodified). The original `gitlab.issue.*` event is **always** already recorded (before dispatch), so neither the issue's arrival nor the comment is ever lost.

## Guarantees & non-guarantees

- **Guaranteed by the dispatcher:** the receiver never crashes; the event is never lost; a failure (agent missing / model error / no MCP) is always recorded as `com.bos.gitlab.triage.failed`.
- **NOT guaranteed by the dispatcher (the skill's responsibility):** the single-active-status invariant (FR-014), the resume/idempotency branch, the exact label transitions, the Q2 "never close/commit" boundary, and — since R7 — **self-termination on the agent's own note** and "one reply per dispatch". These are **model-followed guidance** in the skill, not mechanism-enforced (design §3.8, ADR-12/ADR-13). SC-003's "≥90%" absorbs this residual. The R7 additions are bounded rather than open-ended: a run that ignores them posts one extra comment and buys one more dispatch, which is visible in the issue.
- **Trade-off (ADR-3):** if the process dies *between* the prompt-ack and the run finishing, that one triage is lost (the event stays acked and visible). Mitigated by the skill's resume/idempotency on a re-dispatch; acceptable for v1.
