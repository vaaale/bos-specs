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
  - Trigger on **Issues** (and **Merge requests**).
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

### 7. Validate the switches are independent
- **Event Viewer → Configuration** → toggle `gitlab-issue-triage` **off** → open another issue → the `gitlab.issue.opened` event still appears, but **no agent runs** (the issue is untouched). This is FR-018 / SC-004.
- Change the **agent picker** (Settings → Integrations → GitLab → Workflow) to another agent → subsequent issues dispatch to it (SC-008).

### 8. Validate graceful failure (SC-007)
- Temporarily remove your GitLab server from the agent's MCP allowlist (or point the panel at an agent with no GitLab MCP) → open a `Bug` issue → the `gitlab.issue.opened` event is still recorded, **no crash**, and a `com.bos.gitlab.triage.failed` (`no_gitlab_mcp`) event surfaces. The issue is left unmodified.

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
- **`webhook.receive`** — issue+open → 1 `issue-opened` event with correct projection; MR+open → 1 `mr-opened`; push / issue-update / note → `[]`; malformed JSON → `[]`; description truncated; labels normalized.
- **`dispatch` failure detection** — agent missing → `agent_not_found`; `run.error` → `run_error`; clean run with **no successful `mcp_tool_call`** → `no_gitlab_mcp`; clean run with **one successful `mcp_tool_call`** → success (no failure event). (This is the v5 blocker — the test must fail on the pre-fix `try/catch`-only implementation.)
- **`labels`** — the three classification labels + eight status labels are the exact expected sets; `Needs Review` is terminal.

### E2E (Playwright) — `e2e/002-gitlab-integration.spec.ts`
Run with the `run_e2e_tests` tool / `buildstudio_run_tests`. Cover, against a running preview:
1. Panel loads at Settings → Integrations → GitLab (dedicated view, **no** OAuth panel, **no** MCP status).
2. Enable → URL + "Generate token" visible; generate → one-time reveal + rotate available.
3. (With a mock/recorded GitLab POST + a stubbed agent) a new `Bug` issue emits `gitlab.issue.opened` (Event Viewer), the dispatcher hands to the agent, and the failure/success paths record the right events.
4. Disabling the `gitlab-issue-triage` handler still emits the event but dispatches no agent (FR-018).
5. Loop prevention: a follow-up `note`/`update` POST for the same issue does **not** emit a new `gitlab.issue.opened` (object_kind/action filter).

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
| `com.bos.gitlab.triage.failed` `no_gitlab_mcp` | No GitLab MCP reachable by the agent | Configure a GitLab MCP server + include it in the agent's `mcp` allowlist (Settings → Agents). |
| `agent_not_found` | Panel points at a deleted/renamed agent | Re-select a valid agent in the Workflow card. |
| Issue not re-processed on its own updates | — | Correct by design (loop prevention, ADR-9). Only a **new** issue (`action=open`) triggers. |
