# Quickstart — GitLab Integration (Webhook → Triage)

How to set up and validate this feature end-to-end. Two audiences: the **manual setup + validation path** (SC-001) and the **automated checks** (unit + e2e + gates).

> **No BOS-owned GitLab credential.** BOS stores nothing for GitLab except the webhook verification token. All GitLab access is via a **GitLab MCP server** you configure in **Settings → MCP Servers** and assign to the handling agent.

## Part 1 — Manual setup + validation (SC-001, < 10 min)

### 1. Configure a GitLab MCP server
- **Settings → MCP Servers** → add a GitLab MCP server pointing at your GitLab (gitlab.com or self-hosted), with its own auth (a personal access token or OAuth).
- Confirm it's reachable (the MCP server list shows it healthy).

### 2. Ensure the Gitlab Issue Agent can use it
- **Settings → Agents → Gitlab Issue Agent** (seeded by this feature) → confirm its MCP allowlist includes your GitLab server (default is "all servers," so this is usually already fine).
- The agent loads the `gitlab-issue-triage` skill (seeded). Leave it as-is unless you want to customize the pipeline.

### 3. Enable the webhook
- **Settings → Integrations → GitLab** → **Enable** the webhook.
- **Generate token** → copy the one-time-revealed token.
- Copy the **Receiver URL**.

> **Generating the token is also what makes deliveries reachable.** A delivery
> carries no cookie and no `Authorization` header, so on a Bastion-fronted
> (multi-user) deployment the gateway routes it by resolving
> `sha256(X-Gitlab-Token)` in `data/system/credentials-index.json`. The
> handler registers the token there on **enable**, on **rotate** (both the new
> token and the demoted grace token), and revokes on **disable/delete** — so a
> token that was never generated through the panel is rejected by the gateway
> before the receiver's own `verify()` runs. See
> `docs/dev/integrations/gitlab-triage.md` §4.

### 4. Register in GitLab
- **GitLab → Project → Settings → Webhooks** → **Add webhook**:
  - **URL** = the Receiver URL.
  - **Secret token** = the generated token.
  - Trigger on **Issues**, **Comments**, and (optionally) **Merge requests**.

> **Tick "Comments".** Without it, step 6b below does nothing: a reply you type on the issue never reaches BOS, and the agent can only ever be started by a brand-new issue. This is the single most common reason "commenting doesn't do anything."
- (Optional) Click **Test** in the panel after registering — it should POST a sample and return `{ ok:true }`.

### 5. Create labels in your project (one-time)
Ensure these exist so classification + status work: `Bug`, `Feature`, `Improvement`, and the status labels `Triage`, `Analyzing`, `Verifying`, `Fix Proposed`, `Spec Drafted`, `Needs Review`, `Needs Input`, `Failed`. (The agent adds them to issues; having them pre-created keeps the UI tidy.)

### 6. Open a labeled issue → watch it work
Create an issue in that project labeled **`Bug`** (with a clear title + repro). Then:
- **Event Viewer** → `gitlab.issue.opened` appears (the webhook emitted it).
- The Gitlab Issue Agent runs (fire-and-forget). In GitLab, the issue should:
  1. get a classification comment + `Triage` status,
  2. advance through `Analyzing` → `Verifying` (Bug path),
  3. get the proposed fix posted + `Fix Proposed`,
  4. end at **`Needs Review`** (terminal — you verify + close).
- Repeat with a `Feature` and an `Improvement` issue → expect `Spec Drafted` → `Needs Review` with a draft spec posted.

### 6b. Drive the process by commenting (R7)

The pipeline is not one-shot. **A new comment on the issue re-dispatches the agent**, which resumes from whatever status label the issue is wearing. Two things to try:

**Answer a `Needs Input` question.** Open a deliberately vague Bug ("it crashes sometimes") so the agent stops at `Needs Input` and asks for the missing piece. Type the answer as a normal GitLab comment. Within a moment:
- **Event Viewer** → a `gitlab.issue.noted` event appears, summarised `comment on issue #<iid>`;
- the agent resumes *from where it stopped* — it does not re-classify or re-post its earlier comments — and works on to `Needs Review`.

**Give feedback at `Needs Review`.** On an issue the agent has finished, comment what you want changed ("the proposed fix misses the batch path"). The agent revises its deliverable, posts the revision as a **new comment**, and leaves `Needs Review` on so you can re-verify. Reply again and it revises again. When you are happy, **you** close the issue — the agent never does, no matter what you ask it in a comment.

**What you should NOT see: a comment war.** The agent's own comments come back to BOS as `gitlab.issue.noted` too — GitLab sends them in exactly the same shape as yours, and BOS deliberately does not try to tell them apart (it holds no GitLab credential, so it does not know who your MCP token is). The **agent** does: it asks the MCP server which user the token acts as, sees that the newest note is its own, and stops without posting or relabelling.

So each comment the agent posts costs **one extra, cheap, read-only run** that does nothing. In the Event Viewer that looks like a `gitlab.issue.noted` event with no visible effect on the issue — which is correct, and is recorded as a **success**, not a failure. If you ever see a `com.bos.gitlab.triage.failed` for one of those, the agent could not reach GitLab at all (see step 8), not "the agent had nothing to say."

### 7. Validate the switches are independent
- **Event Viewer → Configuration** → toggle `gitlab-issue-triage` **off** → open another issue → the `gitlab.issue.opened` event still appears, but **no agent runs** (the issue is untouched). This is FR-018 / SC-004.
- Change the **agent picker** (Settings → Integrations → GitLab → Workflow) to another agent → subsequent issues dispatch to it (SC-008).

### 8. Validate graceful failure (SC-007)
- Temporarily remove your GitLab server from the agent's MCP allowlist (or point the panel at an agent with no GitLab MCP) → open a `Bug` issue → the `gitlab.issue.opened` event is still recorded, **no crash**, and a `com.bos.gitlab.triage.failed` (`mcp_rejected`, or `mcp_not_attempted` if the agent never reached for MCP at all) event surfaces. The issue is left unmodified.

### 9. Validate security
- POST to the URL with a **wrong** `X-Gitlab-Token` → `401`. On a Bastion-fronted
  deployment, tell the two 401s apart by their body: an **empty** body with a
  `WWW-Authenticate: Basic` header is the gateway refusing to route (a routing
  problem); `{"error":{"code":"unauthorized"}}` is the receiver's `verify()`
  rejecting the token (a token mismatch).
- **Disable** the webhook → POST (even with a valid token) → `404`.
- POST a **push** event (or a non-`open` issue action) → `200`, `emitted: 0` (ignored, no event).

## Part 2 — Automated checks

### Unit tests (Vitest) — `src/lib/integrations/services/gitlab/__tests__/`
- **`webhook.verify`** — correct token → true; wrong/missing → false; **no candidates → false** (never accept-all); rotation grace (previous accepted).
- **`webhook.routing-index`** — enable/rotate/disable/delete keep the Bastion credentials index in step with the tokens `verify` accepts: generating the first token makes exactly it routable, rotation keeps the grace token routable and drops the one before it, and disable/delete leave nothing behind. (Paired with `tests/bastion/webhook-secret-header.test.ts`, which pins the gateway's half.)
- **`webhook.receive`** — issue+open → 1 `issue-opened` event with correct projection; MR+open → 1 `mr-opened`; push / issue-update → `[]`; malformed JSON → `[]`; description truncated; labels normalized. **Note hooks (R7):** a comment on an Issue → 1 `issue-noted` with the note projection, including the assertion that `iid` comes from the top-level `issue` and **not** from `noteable_id` (the fixture's two ids differ on purpose); a comment whose author is the agent still emits (the receiver cannot tell); a comment on a `MergeRequest` / `Snippet` / `Commit` → `[]`; note truncated.
- **`handler`** — the registration's `eventType` is the prefix `gitlab.issue.*`, headless, `declaredBy:"core"`; the prefix matches `gitlab.issue.opened` **and** `gitlab.issue.noted` but not `gitlab.mr.opened`; a user's disable survives re-registration; and the registered core executor builds the **fresh** task for `opened`, the **resume** task for `noted`, and **no run at all** for an unhandled `gitlab.issue.*` type.
- **`dispatch` failure detection** — agent missing → `agent_not_found`; `run.error` → `run_error`; clean run with **no tool results at all** → `no_output`; clean run with tool results but **no `mcp_tool_call`** → `mcp_not_attempted`; clean run where **every `mcp_tool_call` failed** → `mcp_rejected` carrying the first failure's own error text; clean run with **one successful `mcp_tool_call`** → success (no failure event). (This is the v5 blocker — the test must fail on the pre-fix `try/catch`-only implementation.) **R7:** the resume task carries the note, its author and the identity/self-terminate instructions; a **no-op resume** (successful reads, nothing posted) is a **success**, not `mcp_rejected`; a resume with a broken MCP is still `mcp_rejected`.
- **`gitlab-triage-skill-references`** (`tests/agent/`) — every shipped reference is listed by `skill_load` and readable through `skill_read_file`, the router links them all with no dangling entries, and `references/notes.md` carries the identity step, the "post nothing, change no label, stop" branch and the unwidened boundary.
- **`labels`** — the three classification labels + eight status labels are the exact expected sets; `Needs Review` is terminal.

### E2E (Playwright) — `e2e/002-gitlab-integration.spec.ts`
Run with the `run_e2e_tests` tool / `buildstudio_run_tests`. Cover, against a running preview:
1. Panel loads at Settings → Integrations → GitLab (dedicated view, **no** OAuth panel, **no** MCP status).
2. Enable → URL + "Generate token" visible; generate → one-time reveal + rotate available.
3. (With a mock/recorded GitLab POST + a stubbed agent) a new `Bug` issue emits `gitlab.issue.opened` (Event Viewer), the dispatcher hands to the agent, and the failure/success paths record the right events.
4. Disabling the `gitlab-issue-triage` handler still emits the event but dispatches no agent (FR-018).
5. Loop prevention (structural half): a follow-up `action:"update"` POST for the same issue, and a comment on a merge request, emit **nothing**.
6. R7 re-entry: a comment on an issue emits `gitlab.issue.noted` with the project-scoped iid, is dispatched to the **same** handler, and drives a resume run that establishes its identity, reads the thread and posts **exactly one** reply. Then the agent's own note comes back and the run reads but writes nothing — and records **no** `com.bos.gitlab.triage.failed`.

> **Every note the e2e delivers must carry an `@@e2e` script on its own line**, for the same reason every issue does: an unscripted note now dispatches a real run, and `llm.ts` sends it with the api key `"MISSING"` — a live billable call. The directive must start a line because the dispatcher embeds the note mid-paragraph in the resume task.

> A full live GitLab round-trip needs real credentials + a GitLab MCP server, so e2e **stubs the agent + GitLab MCP** and drives the real webhook receiver with recorded payloads. The manual path (Part 1) is the real-credentials validation.

### Gates
```bash
npx tsc --noEmit        # typecheck (strict, erasableSyntaxOnly)
npm run lint            # lint
# do NOT run `npm run build` while `next dev` is running
```
Preview via the Supervisor, then **Promote** (user action). Promote is **code-only** — the seeded agent/skill land in `data/` on boot (`seed-sync.ts`); the webhook token is per-environment (re-generate on a fresh preview if needed).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Webhook 401, body `{"error":{"code":"unauthorized"}}` | Token not generated, or mismatched | Generate a token in the panel; paste the **same** one into GitLab's "Secret token." |
| Webhook 401, **empty** body + `WWW-Authenticate: Basic` | The Bastion gateway could not route the delivery: the token is not in this user's credentials index | **Rotate secret** once (that is what registers it) and re-paste. If it persists, check `bastion/src/webhook-secret-header.ts` lists `x-gitlab-token`. |
| Webhook 404 | Webhook disabled (or unknown) | Enable the webhook in the panel. |
| Event emitted but issue untouched | `gitlab-issue-triage` handler disabled | Enable it in Event Viewer → Configuration. |
| `com.bos.gitlab.triage.failed` `no_output` | The run produced no tool calls at all | Check the agent has usable tools — an empty `mcp` allowlist leaves it nothing to call (Settings → Agents). |
| `com.bos.gitlab.triage.failed` `mcp_not_attempted` | The agent acted but never tried GitLab MCP | Include a GitLab MCP server in the agent's `mcp` allowlist and check the `gitlab-issue-triage` skill is available to it. |
| `com.bos.gitlab.triage.failed` `mcp_rejected` | Every GitLab MCP call failed | Read the event's `cause` — it quotes the server verbatim (e.g. `401 Unauthorized`). |
| `agent_not_found` | Panel points at a deleted/renamed agent | Re-select a valid agent in the Workflow card. |
| Issue not re-processed after a **label** change | — | Correct by design (loop prevention, ADR-9): label edits arrive as `action:"update"` and are ignored. |
| **Commenting does nothing** — no `gitlab.issue.noted` event at all | The GitLab webhook is not triggering on **Comments** | GitLab → Project → Settings → Webhooks → edit the hook → tick **Comments**. |
| A `gitlab.issue.noted` event appears but the issue is untouched | Usually correct: the newest note was the agent's own, so it self-terminated (R7). | Check the run transcript. If the newest note really was a human's and the agent still did nothing, its MCP server may expose no "current user" tool and the stage-prefix fallback misread the thread — see `references/notes.md`. |
| The agent keeps commenting on itself | The skill's self-termination was edited or overridden | `references/notes.md` is the file that decides. The receiver deliberately does **not** filter notes (it cannot tell whose they are), so do not "fix" this in `adapters/webhook.ts`. |
| Comments on a **merge request** do nothing | — | Correct by design: re-entry is issues-only (Q1). |
