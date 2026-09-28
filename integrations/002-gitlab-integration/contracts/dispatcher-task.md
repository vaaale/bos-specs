# Contract: Handler → Agent Dispatch (task + failure detection)

The contract of the **thin dispatcher** — how the `gitlab-issue-triage` handler hands a new issue to the configured agent, and how it detects failure **post-run** (the v5 blocker fix). Source: [`src/lib/integrations/services/gitlab/triage/dispatch.ts`](../../../src/lib/integrations/services/gitlab/triage/dispatch.ts) (new).

## Entry

`dispatchToAgent(record)` is launched **fire-and-forget** from the core handler executor (`void dispatchToAgent(record)`). The executor returns immediately (prompt-ack, ADR-3). `record` is the dispatched `gitlab.issue.opened` event record.

## Step 1 — Resolve the agent

```
agentId = record/... no — from persisted config:
agentId = (await readState("gitlab")).services["webhooks"].config.triageAgentId ?? "gitlab-issue-agent"
agent   = getAgent(agentId)
```

- **`agent` is undefined** → emit `com.bos.gitlab.triage.failed` with `reason: "agent_not_found"`, `agentId`, and the issue id. **Stop — no run.** (The original `gitlab.issue.opened` event is already durably recorded; nothing is lost.)

## Step 2 — Build the task

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
| **No usable GitLab MCP** | the run finished (`completed`/`aborted`) but **no collected `tool_result` for `mcp_tool_call` has `ok:true`** | emit `com.bos.gitlab.triage.failed` `reason:"no_gitlab_mcp"` |
| Agent reached GitLab | **at least one `mcp_tool_call` `ok:true`** | success path — any mid-pipeline problem is the **skill's** responsibility (it posts a `Failed` status label). The dispatcher does not second-guess a run that demonstrably touched GitLab. |

`ok` is derived from the `tool_result` event's result: an `Error:`-prefixed / error payload ⇒ `ok:false`, otherwise `ok:true`. (The runner's `onEvent` already emits `{ type:"tool_result", name, result, ok }` per ADR-13 — collect `name === "mcp_tool_call"`.)

**Failure event** `com.bos.gitlab.triage.failed`:
```
{
  reason: "agent_not_found" | "run_error" | "no_gitlab_mcp",
  projectId, iid, url,          // from the issue context
  cause?: string                // run.error / short detail
}
```
This is the deterministic surface for FR-016 / SC-007 (no MCP ⇒ graceful, recorded, non-crashing, issue unmodified). The original `gitlab.issue.opened` event is **always** already recorded (before dispatch), so the issue's arrival is never lost.

## Guarantees & non-guarantees

- **Guaranteed by the dispatcher:** the receiver never crashes; the event is never lost; a failure (agent missing / model error / no MCP) is always recorded as `com.bos.gitlab.triage.failed`.
- **NOT guaranteed by the dispatcher (the skill's responsibility):** the single-active-status invariant (FR-014), the resume/idempotency branch, the exact label transitions, and the Q2 "never close/commit" boundary. These are **model-followed guidance** in the skill, not mechanism-enforced (design §3.8, ADR-12). SC-003's "≥90%" absorbs this residual.
- **Trade-off (ADR-3):** if the process dies *between* the prompt-ack and the run finishing, that one triage is lost (the event stays acked and visible). Mitigated by the skill's resume/idempotency on a re-dispatch; acceptable for v1.
