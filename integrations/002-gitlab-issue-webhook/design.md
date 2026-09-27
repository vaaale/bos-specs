# Design: GitLab Issue Webhook Integration

**Feature Branch**: `bos/002-gitlab-issue-webhook`
**Spec**: `spec.md` (binding) · **UI contract**: `mockup.html` (binding)

---

## Classification

**bos-core.** The feature ships entirely inside the Next.js app process: a new integration service on the existing kernel webhook platform (`src/lib/integrations/`), a core headless executor on the 034 event bus, a thin per-issue state map in the integration state store, and one new section in the Settings → Integrations UI. No marketplace item, no new app, no new agent. The one architectural commitment that shapes everything: **the GitLab MCP server is the single holder of the GitLab credential and the sole I/O layer for all agent-side GitLab operations.** Consequently the kernel holds no GitLab credential, performs no GitLab I/O, registers no provider-side webhook (registration is manual by the user, FR-004), and the "issue-update glue" is not a kernel module at all — it is a **brief section** written into each resolving agent run (model change #4). The kernel's job is exactly: verify → parse → emit → classify → dispatch → track.

---

## Context — reused vs. new

Everything below is grounded in real source paths. "New" means files this feature creates; "edit" means a registry/wiring touch point.

| Concern | Reused (real path) | New / edited |
|---|---|---|
| Receiver route, enable/connect gate, raw body, idempotency, verify/ack, emission | the Next.js route file `src/app/api/integrations/webhooks/[integrationId]/[serviceId]/route.ts` (404 if no handler/not enabled/not connected; verify→401; receive→`emitIntegrationEvent` per event) | — |
| Handler contract (pure parsing: `verify`, `receive`, optional `onEnable`/`onDisable`) | `src/lib/integrations/webhooks/handler.ts` | `services/gitlab/adapters/issue-webhook.ts` (impl; **no** `onEnable`/`onDisable` — see §Component plan) |
| Idempotency ring (1000), body-hash fallback (GitLab sends no `x-goog-message-id`/`x-idempotency-key`) — note `verify.ts` exports only the **HMAC** `verifySignature`, not a token helper, so the constant-time token compare is local to the handler (see below) | `src/lib/integrations/webhooks/verify.ts` | — |
| Telegram-shaped token verify precedent (fixed header, constant-time compare against `[extras.token?, secrets.primary, secrets.previous]`, no HMAC) | `src/lib/integrations/services/telegram/adapters/bot-webhook.ts` | mirrored by `issue-webhook.ts` with header `X-Gitlab-Token` |
| Secret minting (32-byte hex), rotation grace (primary→previous), `webhookUrl(id,sid,origin)` = `<origin>/api/integrations/webhooks/<id>/<sid>` | `src/lib/integrations/webhooks/manager.ts`, `src/lib/integrations/webhooks/store.ts` (secrets key `webhook:<serviceId>`) | — |
| Integration manifest + service definition + registration | `src/lib/integrations/types.ts`, `src/lib/integrations/registry.ts`, `src/lib/integrations/index.ts` (side-effect barrel imports of `./services/gsuite`, `./services/telegram`) | `services/gitlab/manifest.ts` + `services/gitlab/index.ts`; **edit** `integrations/index.ts` (one import) |
| Webhook handler registry ("adding a handler = one entry + the impl") | `src/lib/integrations/webhooks/registry.ts` (static `HANDLERS`) | **edit**: static entry `gitlab: { issues: issueWebhookHandler }` |
| Event-bus emission + legacy type derivation | `src/lib/events/from-integration-event.ts` (`emitIntegrationEvent(ev)` → `emit({type: deriveLegacyType(ev.service, ev.type), source: legacySourceFor(ev.service), payload: {...ev.data, _integrationType, _integrationService, summary}})`); `src/lib/events/legacy-mapping.ts` has **no** gitlab entry ⇒ systematic fallback: `"gitlab/issues"` + `"issue_opened"` → `com.bos.gitlab.issues.issue-opened` | — |
| Non-sensitive integration state (config in `state.json`, per-integration mutex, atomic write) | `src/lib/integrations/state/store.ts` (`readState`/`mutateState`); config home `services[svcId].config.webhook` | `services/gitlab/resolution-state.ts` (writes through that store; serialization precedent: `src/lib/self-heal/store.ts` — one in-process promise chain, atomic) |
| Generic webhook UI (URL, token reveal/rotate, enable, test, delete, last-received plumbing) | `src/components/apps/settings/integrations/WebhookSection.tsx` | `src/components/apps/settings/integrations/GitLabIssuesSection.tsx` (+ **edit** `ServiceConfigView.tsx` wiring) |
| Self-heal trigger + autonomous slow path | `src/lib/self-heal/`: trigger = emit `com.bos.self-heal.trigger` (`SELF_HEAL_EVENTS.trigger`) handled by core executor `core:self-heal` (`spine-handler.ts` `registerSelfHealSpine`); `selfHealIntake({trigger:"explicit", description, toolName, extra, …})` → guards → reentrancy filter → dedupe → cost cap → `createCase` → Diagnostician; autonomous escalation: `escalateCase` → `buildAutonomousBrief` (**`brief.ts`** — the per-conversation pre-authorization precedent) sets `activeFeatureBranch` server-side **before the first token**, then `runSubAgent(agent, brief, {conversationId, contentOnly:false, featureBranch, onEvent})` fire-and-forget with a run observer; terminal `self_heal_complete_fix` → case "preview-ready"; lifecycle events `com.bos.self-heal.*` via `emitSelfHeal`; reentrancy (`reentrancy.ts`) filters its own events via a `selfHeal.role` payload marker | brief-augmenter seam (**new**, see §Headless pipeline-driving mechanism) so the GitLab-reporting protocol enters the brief without self-heal importing the gitlab service |
| Build Studio spec pipeline (spec-kit method: specify → design → plan → tasks → implement → preview) | spec-kit method + `create_project` (user-specs store), server-side `activeFeatureBranch` mechanism (self-heal precedent) | feature-intent brief builder (lives in `issue-router.ts`, §Component plan) |
| **Reconciliation — existing gitlab OAuth provider**: `src/lib/integrations/oauth/providers/gitlab.ts` exists, but it was built for **git remotes** (push/pull credentials), not API access. Under the MCP model the `gitlab` integration manifest **must not claim an OAuth flow**. Confirmed against `src/lib/integrations/types.ts`: `IntegrationManifest.oauthConfig` is a **required** field (not optional), so the manifest supplies a descriptor that advertises no usable flow — `supportedScopes: []` plus placeholder `authorizationUrl`/`tokenUrl` — and nothing in the integration's flow consumes it (no OAuth connect, no consent screen, no token exchange). The git-remote provider stays untouched and orthogonal. |

**What is NOT reused / is new by necessity:** the bug-path **delivery tail** — pushing the fix branch to the watched GitLab project and opening a draft MR. Self-heal today works in a local worktree on `bos/self-heal-<caseId>`, never pushes to external remotes, and never opens MRs. That tail is new, lives entirely inside the agent run's brief protocol, and is the feature's biggest unproven surface (Risk R1).

---

## Component / file plan

### New files

**`src/lib/integrations/services/gitlab/manifest.ts`**
Responsibility: the `IntegrationManifest` for id `gitlab` — `services: [{ id: "issues", name: "Issues", scopes: [], configSchema: { targetProject } }]`, icon, description, and the **required** `oauthConfig` as a no-flow descriptor (type constraint confirmed in `src/lib/integrations/types.ts`; see Context reconciliation). Composes: `types.ts` (`IntegrationManifest`, `ServiceDefinition`, `OAuthConfig`).

**`src/lib/integrations/services/gitlab/index.ts`** (barrel)
Responsibility: side-effect registration — `registerIntegration(manifest)` and `registerGitLabIssueSpine()` (core headless executor registration, mirroring `registerSelfHealSpine`), plus the brief-augmenter registration. This is the single import added to `src/lib/integrations/index.ts`. Composes: `manifest.ts`, `issue-router.ts`, `brief-section.ts`, the self-heal augmenter registry, `integrations/registry.ts`.

**`src/lib/integrations/services/gitlab/adapters/issue-webhook.ts`**
Responsibility: the `WebhookHandler` for `gitlab/issues` — pure parsing, zero I/O, zero GitLab knowledge beyond the payload shape.
- `verify({req, body, secrets, config})`: read `X-Gitlab-Token`; constant-time compare (via `webhooks/verify.ts` helpers) against `[config.extras?.token?, secrets.primary, secrets.previous]` — the Telegram `bot-webhook.ts` shape, no HMAC; a **local** `timingSafeStringEqual` helper (the Telegram precedent defines its own, `bot-webhook.ts:80` — `webhooks/verify.ts` has no token helper to import). **Stricter than the Telegram precedent:** Telegram returns `true` when no candidate secret exists (to avoid bricking, `bot-webhook.ts:110`); here no candidates ⇒ reject, and a missing header ⇒ reject. A conscious divergence — the platform always mints a primary secret on enable, so an empty-candidate state must not silently accept. (401-vs-404 is the framework's job, `route.ts`.)
- `receive({req, body, config})`: JSON parse; `object_kind !== "issue"` ⇒ return `[]` (FR-013). Otherwise return exactly one event (shape in §Data model) — `type` is `issue_opened` for `action: open`, `issue_updated` for `update`/`close` (the precise value rides in `data.action`). No other object_kinds, no notes, no MRs (A8).
- **No `onEnable`, no `onDisable`.** The handler has nothing provider-side to do: the webhook is registered manually by the user (FR-004), secret minting is the platform's (`manager.ts`), and disabling is local (`enabled:false` ⇒ receiver 404s; hook removal in GitLab is the user's action, FR-005). The old design's onEnable throw-vs-record problem does not exist under this model.
Composes: `handler.ts` contract, `verify.ts`, nothing else.

**`src/lib/integrations/services/gitlab/issue-router.ts`**
Responsibility: the core headless executor on `com.bos.gitlab.issues.issue-opened` / `com.bos.gitlab.issues.issue-updated` (the types derived by `from-integration-event.ts`), **I/O-free** — it performs no GitLab calls and no MCP calls; it reads/writes only the thin local state map and emits internal bus events / starts local runs. It also owns the **label vocabulary constants** (exactly the spec's 8 labels — see §Data model; no separate `labels.ts`), and the **feature-intent brief builder**. Arms:
1. **Track**: on any in-scope-shaped issue event, update `lastReceived` (FR-006) — including out-of-scope events (intake visibility is unconditional, US2). The *basic* "last received" timestamp is maintained for free by the receiver (`route.ts` writes `state.services["issues"].lastSync` on every accepted delivery); the router's `lastReceived` is the **additive** enrichment (issue id + title, the "issue #482 …" the mockup shows), stored under the `config.extras` / sibling key in §Data model — not a replacement for `lastSync`.
2. **Idempotency gate**: `issueKey = "<projectPath>:<issueIid>"`; if the map already holds a live entry for the key ⇒ no-op (FR-014, SC-003, FR-030(b)). Entries are created **only** by an in-scope dispatch (exactly one of `Bug`/`Feature`/`Enhancement`) or an ambiguity record — a **0-label (out-of-scope) event creates NO map entry** (it is a pure no-op, FR-020/SC-009), so a later classification label applied to that issue is **not** gated and is picked up on the label-change event per FR-014. The "never auto-cleared" rule applies only to entries that actually exist (dispatched / ambiguous): v1 does not automatically clear them, because re-processing a finished issue is out of scope (spec: "re-opened issue" edge case).
3. **Classify**: count of `{Bug, Feature, Enhancement}` present in `data.labels` — exact, case-sensitive match, never inferred (FR-015/016, A4).
   - **0** ⇒ no-op, **no write to GitLab at all, not even a note** (FR-020, SC-009).
   - **>1** ⇒ ambiguity (see M3 decision below).
   - **1** ⇒ dispatch.
4. **Dispatch — Bug**: emit `com.bos.self-heal.trigger` with
   `payload = { trigger:"explicit", description: "<title>\n\n<description>", toolName: "gitlab-issue:<projectPath>#<iid>", extra: { gitlab: { projectPath, issueIid, issueUrl, issueKey, classification:"Bug" } } }`.
   **No `selfHeal.role` marker** — this is a genuine external trigger, so self-heal's reentrancy filter does not swallow it. The `toolName` embedding the issue id keeps self-heal's dedupe key distinct per issue.
5. **Dispatch — Feature / Enhancement** — the router does **not** create the spec project itself. `create_project` is an agent-facing serverTool (`src/lib/assistant/tools/server/specs.ts`) that resolves its branch from the *conversation's* active feature branch, which is unset at router-dispatch time — a router-side call would land the project folder on base while the run's spec artifacts ride `bos/gitlab-<…>`, the exact read/write desync `specs.ts`'s header warns about. Instead the router (a) seeds a conversation for the run (self-heal's `saveConversationMessages` precedent) and (b) sets that conversation's active feature branch — `bos/gitlab-<sanitized-projectPath>-issue-<iid>` — server-side **before the first token** (the same `setConversationActiveFeatureBranch` step self-heal's `escalateCase` already performs before `runSubAgent`). The feature-intent brief's **first step** then instructs the RUN to create the project (`create_project`, name references `GitLab #<iid> <title>`; the store's method binding is the global spec-kit one — no per-project workflow needed), whose tool call now resolves the branch from the conversation → project, feature, and every spec artifact land on the **same** feature branch. The router then `runSubAgent(agent, brief, {conversationId, contentOnly:false, featureBranch, onEvent})` fire-and-forget with the (new, thin) run observer. The brief must state `featureBranch` and the store id explicitly so the run's first `create_project` uses the right values.
6. **Track (case observer)**: a second registered handler on `com.bos.self-heal.*` lifecycle events; for cases whose trigger context carries `extra.gitlab` (resolved from the self-heal case store — a local read, not GitLab I/O), mirror `caseId` (on `case_created`) and terminal state into the map (`fix_ready` → `agent:preview-ready`, case failure → `agent:failed`). There is **no dedicated `case_failed` lifecycle event** in `SELF_HEAL_EVENTS` (`src/lib/self-heal/types.ts`) — failure is a status transition — so the failure branch keys off the case STORE's status (a local read) or the run-end verdict, not an event subscription. Best-effort mirror: the **authoritative** label is the one on GitLab, maintained by the run.
7. **Ambiguity (M3 decision — option (a))**: the router emits a lightweight internal bus event `com.bos.gitlab.issues.ambiguity` (raw `emit`, *not* `emitIntegrationEvent`), and a thin registered handler starts a **one-shot agent run** whose brief is the `mode: "ambiguity"` GitLab section: post one note stating exactly one of `Bug`/`Feature`/`Enhancement` must be applied; no labels; no pipeline. **Trade-off stated:** one extra (cheap, rare) sub-agent run, bought by keeping the router strictly I/O-free — option (b) (defer to the run-start protocol) is impossible because no run starts for an ambiguous issue. Note dedupe: the map records `category:"ambiguous"` for the key; repeated `>1`-label events (GitLab re-delivers on every label change) are no-ops unless the *set* of classification labels changes. A later single-label event still dispatches (the ambiguity record does not block — it is not a pipeline run), satisfying the file-then-reclassify flow.
Composes: `resolution-state.ts`, `brief-section.ts`, the self-heal trigger contract + lifecycle events, `create_project`, `runSubAgent` + observer, `emitIntegrationEvent`'s derived types (consumed, not called).

**`src/lib/integrations/services/gitlab/resolution-state.ts`**
Responsibility: the thin per-issue map + connection state + `lastReceived`, all in `data/integrations/gitlab/state.json` via `readState`/`mutateState` (per-integration mutex + atomic write from `integrations/state/store.ts`), with all writes funneled through one in-process promise chain (the `self-heal/store.ts` serialization precedent). No secrets, no GitLab data beyond non-sensitive metadata (FR-029, NFR-005). Shape in §Data model. Composes: `integrations/state/store.ts`; imports the vocabulary constants from `issue-router.ts` (one-way).

**`src/lib/integrations/services/gitlab/brief-section.ts`**
Responsibility: `renderGitLabBriefSection({ mode: "bug" | "feature" | "ambiguity", ctx })` — the **GitLab-reporting protocol** as plain brief text: MCP tool names, note contents, label transitions, terminal behavior, the closed-mid-run rule, and the propose-only constraints. This is model change #4's "issue-update glue as a BRIEF SECTION". Pure string rendering; no I/O. The bug mode's delivery tail (push + draft MR) and the feature mode's tail (preview link, no MR) live here (§Resolution flow).

### Edits (wiring only)

- **`src/lib/integrations/index.ts`** — add `import "./services/gitlab";` (barrel pattern of gsuite/telegram).
- **`src/lib/integrations/webhooks/registry.ts`** — static `HANDLERS` entry `gitlab: { issues: issueWebhookHandler }`.
- **`src/lib/self-heal/brief.ts`** (+ a new `src/lib/self-heal/brief-augmenters.ts` registry module) — `buildAutonomousBrief` appends any registered augmenter sections matching the case's trigger context; the gitlab barrel registers `renderGitLabBriefSection` keyed on `extra.gitlab` presence. Dependency direction stays clean: *integrations → self-heal*, self-heal never imports gitlab.
- **`src/components/apps/settings/integrations/ServiceConfigView.tsx`** — render `<GitLabIssuesSection>` for `gitlab/issues`, above/beside the generic `WebhookSection`.
- **`src/components/apps/settings/integrations/GitLabIssuesSection.tsx`** (new, UI) — per the mockup: MCP-connection card (status pill + instance tag + "validated against <project>" + **Manage** → MCP Servers + **Configure gitlab-mcp** CTA in the not-configured variant B), target project field, **manual-registration steps** (numbered: open project → Settings → Webhooks; paste receiver URL; paste secret token; trigger = Issues events only — with copy buttons), label-provisioning status line ("8 / 8 labels present … created automatically (idempotently) the first time the system works on an issue here"), and last-received row. It **reuses** `WebhookSection` for URL/token reveal/rotate/enable/test/delete; "Send test event" rides the existing generic test route (`src/app/api/integrations/[id]/services/[serviceId]/webhook/test/route.ts`, which already synthesizes an `IntegrationEvent` without provider access) — for this service it synthesizes a *synthetic* issue event onto the bus (kernel-only, no GitLab I/O) and stamps `lastReceived` with a `synthetic:true` marker (mockup toast: "Synthetic issue event dispatched to the event bus").

**Explicitly absent** (model changes #1, #2, #4): no `api.ts`, no `labels.ts` (vocabulary constants live in `issue-router.ts`), no GitLab REST client anywhere in the kernel, no provider hook registration/removal code, no `onEnable`/`onDisable`.

---

## Data model

### GitLab Issue Event (IntegrationEvent)

`receive()` returns one object per qualifying hook, and `emitIntegrationEvent` requires **all** fields (M1) — `service` and `timestamp` included, because the legacy-fallback derivation **splits on `service`** (`"gitlab/issues"` → `com.bos.gitlab.issues.` prefix):

```
{
  service: "gitlab/issues",                  // required — <integrationId>/<serviceId>, feeds deriveLegacyType
  type: "issue_opened" | "issue_updated",    // required
  timestamp: <epoch millis, number>,          // required (confirmed: IntegrationEvent.timestamp is number)
  data: {
    projectPath: "acme/browseros",             // project.path_with_namespace
    issueIid: 482,                             // object_attributes.iid
    title: string,
    description: string,
    labels: string[],                          // object_attributes.labels (names)
    author: string,                            // username
    action: "open" | "update" | "close",
    issueUrl: string                           // object_attributes.url
  }
}
```

The bus-level `summary` (along with `_integrationType` and `_integrationService`) is **not** an `IntegrationEvent` field — `emitIntegrationEvent` adds all three to the emitted bus payload (`payload: {...ev.data, _integrationType, _integrationService, summary}`); the handler's `receive()` returns exactly the four-field shape above. (Confirmed against `src/lib/integrations/types.ts` and `src/lib/events/from-integration-event.ts` — R0 closed.)

Bus types (systematic fallback, no gitlab entry in `src/lib/events/legacy-mapping.ts`): `com.bos.gitlab.issues.issue-opened`, `com.bos.gitlab.issues.issue-updated`. Internal (router-only, raw emit): `com.bos.gitlab.issues.ambiguity`.

### Thin per-issue state (`data/integrations/gitlab/state.json`)

The map lives **inside the real `IntegrationState` shape** (`src/lib/integrations/types.ts`): `{ connected: boolean; lastConnected?; services: Record<svcId, { enabled; config; lastSync?; error? }>; scopeOverrides; oauthMeta?; lastError? }`. Nothing below invents a new top-level field; the additive data sits under `services["issues"]` (`config` is `Record<string, unknown>`, so `config.extras` is free-form, not part of the declared `configSchema`). Non-sensitive metadata only. Writes: `mutateState` (per-integration mutex + atomic write, `integrations/state/store.ts`), funneled by one in-process promise chain (the `self-heal/store.ts` serialization precedent).

```
// the REAL IntegrationState — only the gitlab-relevant fields shown
{
  connected: true,               // ← THE receiver gate. route.ts:64 reads this top-level
                                 //   BOOLEAN (404 when false). Flipped true/false by a
                                 //   mutateState on the MCP-validation outcome — NOT an
                                 //   invented connection.status enum (§Connection state).
  services: {
    "issues": {
      enabled: true,             // user-enabled (framework field)
      lastSync: 1716800000000,   // ← FR-006 basic "last received" — the receiver ALREADY
                                 //   writes this on every accepted delivery (route.ts:119).
      config: {
        targetProject: "acme/browseros",  // ← the declared configSchema field
        extras: {                            // ← ADDITIVE (free-form; never validated/overwritten)
          lastReceived: { at, issueIid?, projectPath?, title?, synthetic? } | null,  // the
          issues: {                                       // mockup's "issue #482 …" enrichment —
            "<projectPath>:<issueIid>": {                 //   a richer lastSync, NOT a replacement
              category: "bug" | "feature" | "enhancement" | "ambiguous",
              caseId?: string,           // bug path (filled by case observer)
              featureBranch?: string,    // feature path (set by router at dispatch)
              workflowLabel: "agent:in-progress" | "agent:needs-input"
                                        | "agent:failed" | "agent:preview-ready" | null,
              updatedAt: <ISO-8601>
            }
            // NOTE (FR-014): entries exist ONLY after an in-scope dispatch or an
            //   ambiguity record. A 0-label (out-of-scope) event creates NO entry
            //   (FR-020/SC-009), so a later classification label is not gated.
          },
          validation: { mcpConfigured, status, validatedProject?, validatedAt? }  // panel-only
        }                                                                // (receiver reads
      }                                                                //   only `connected`)
    }
  },
  scopeOverrides: {}
}
```

Authoritative deep state lives elsewhere by design: self-heal case store (`data/self-heal/cases/<id>.json`) for bugs; BS spec artifacts + feature branch for features (FR-029). The map's `workflowLabel` is a mirror for idempotency/panel; divergence when a run dies before its first GitLab mutation is accepted in v1 (Risk R8).

### Label vocabulary — exactly the spec's 8 labels (disjoint, no drift)

Constants in `issue-router.ts`; guarded by unit tests (exact set, exact casing, disjointness):

- **Classification** (human-set; the opt-in trigger; fixed, not configurable, case-sensitive exact strings): `Bug`, `Feature`, `Enhancement`.
- **Workflow** (system-set; mutually exclusive; `agent:`-prefixed): `agent:queued`, `agent:in-progress`, `agent:needs-input`, `agent:failed`, `agent:preview-ready`.

There is **no** "promoted"/"merged" label (FR-026, NFR-004). The full 8-label set is provisioned via MCP (idempotent) the first time the system works on an in-scope issue in a project — FR-031, as step 0 of the run brief, **not** a kernel enable step (model change #7).

### Workflow state machine (5 labels)

```
                    ┌───────────────────────────────────────────────┐
                    │  agent:queued  (RESERVED — see note below)    │
                    └──────────────┬────────────────────────────────┘
                                   │  (v1: never applied)
                    ┌──────────────▼────────────────────────────────┐
                    │          agent:in-progress                    │
                    │  applied by the run's FIRST GitLab mutation   │
                    │  (+ pickup note, FR-023)                      │
                    └──────┬───────────────────┬────────────┬───────┘
                           │                   │            │
              ┌────────────▼───┐   ┌───────────▼───┐   ┌────▼────────────────┐
              │ agent:needs-   │   │ agent:failed  │   │ agent:preview-ready │
              │ input (block.) │   │ (terminal)    │   │ (terminal)          │
              └───────┬────────┘   └───────────────┘   └─────────────────────┘
                      │ human answers (run resumes)
                      ▼
              agent:in-progress
```

- **`agent:queued` — PROVISIONED but the system does NOT transition through it in v1** (model change #5). FR-031 provisions all 8 labels (including `queued`) on first use, but v1 never *writes* `queued` to an issue: dispatch is immediate, the router is I/O-free (no kernel label write), so the run's **first** GitLab mutation is `agent:in-progress` + the pickup note. `queued` stays in the vocabulary (provisioned, present) for a future pre-dispatch acknowledgment; the system simply doesn't route through it yet. **Spec nuance stated:** FR-025's "transitions as the pipeline advances" is satisfied by *starting* at `in-progress` — the issue still reflects every subsequent stage; SC-007 (human can state status from label + notes) holds because `in-progress` + the pickup note is itself a legible state.
- **Allowed transitions:** `in-progress → needs-input` (question note, FR-027), `needs-input → in-progress` (resume after the human answers), `in-progress → failed` (terminal; FR-024 failed note), `in-progress → preview-ready` (terminal; success tail). **Abandoned-on-close (FR-028):** mapped to `failed` with an abandonment note ("issue closed mid-run — reopened work requires a new trigger"). *This is a spec gap: "abandoned" has no label of its own in the 5-label vocabulary; `failed` is the honest terminal — flagged as open question OQ-1.* Terminals have no exit in v1 (re-trigger is out of scope, spec "re-opened issue" edge case).
- Every transition is paired with a note (FR-024/025, NFR-006).

### Connection state (FR-001) — derived, never a provider hook

Because kernel headless handlers cannot call MCP tools, "validated" is produced by a **one-shot validation agent run**, not a kernel probe:

1. **Configured** — kernel-local read of the MCP server registry: is a gitlab MCP server (e.g. `gitlab-mcp`) present and enabled? (No network, no MCP call.)
2. **Validated** — the panel's enable/validate action starts a thin one-shot sub-agent run (brief: "call `list_labels` (or `get_issue` on a known issue) for project `<targetProject>` via gitlab-mcp; report success/failure"). Success ⇒ a `mutateState` **flips the top-level `connected = true`** and records additive panel metadata `config.extras.validation = { status:"validated", validatedProject, validatedAt }`. This is what "validated against `acme/browseros`" in the mockup means operationally: **a real `list_labels`/`get_issue` round-trip for the target project through the MCP server.**
3. **Unavailable** — a run that cannot reach GitLab (revoked credential, missing project permission — spec edge case) reports failure ⇒ a `mutateState` sets **`connected = false`** + `config.extras.validation.status = "unavailable"`.
- **Refresh:** on enable, on explicit re-validate (panel), and lazily on run-reported failure. No periodic refresh in v1.
- **Receiver gate:** `route.ts` reads the **top-level `state.connected` BOOLEAN** (`route.ts:64`) — not the additive `config.extras.validation.status`, which is panel-only. That boolean is what the `mutateState` above flips. (Cheap, synchronous, no MCP in the request path — protects NFR-003's <2s p95 and never hammers GitLab.) Not connected ⇒ service cannot be enabled ⇒ receiver 404s (spec US1 scenario 2, FR-013).

---

## Resolution flow

### Bug (`Bug` → self-heal)

1. **Intake.** GitLab POSTs the issue hook (open with `Bug`, or an update carrying the label — FR-014) → `route.ts`: handler exists, enabled, connected (state gate) → `verify()` (`X-Gitlab-Token`, constant-time) → idempotency ring (body-hash fallback; GitLab retries are byte-identical) → `receive()` → `emitIntegrationEvent` → bus event `com.bos.gitlab.issues.issue-{opened,updated}` (NFR-003: pure parse, well under 2s).
2. **Router (I/O-free).** Update `lastReceived`; `issueKey` gate (mapped ⇒ no-op); classify: exactly one of `Bug` ⇒ dispatch. Emit `com.bos.self-heal.trigger` (context in §Component plan; no `selfHeal.role` marker). Map entry `{category:"bug", workflowLabel:"agent:in-progress"}`.
3. **Self-heal spine (unchanged).** Intake guards → reentrancy (external trigger passes) → dedupe (distinct `toolName`) → cost cap → `createCase` → Diagnostician: diagnose → verify → implement in a BOS source worktree on `bos/self-heal-<caseId>` — all per spec 031.
4. **The run, driven by the augmented brief** (`buildAutonomousBrief` + `renderGitLabBriefSection({mode:"bug"})`), where the GitLab-reporting protocol (model change #4) says:
   - **Step 0a — provision labels (FR-031):** `list_labels` → `create_label` each of the 8 missing labels (idempotent). First time only; a no-op afterwards.
   - **Step 0b — closed check (FR-028):** `get_issue`; if `state == "closed"` ⇒ abandonment note + `agent:failed`, stop. No MR/preview against a closed issue.
   - **Step 0c — pickup (FR-023):** **the run's first GitLab mutation.** `update_issue` with `labels = <current classification labels from get_issue> + ["agent:in-progress"]` (drop any other `agent:*` — GitLab replaces the label set, so the merge is deliberate) + `create_issue_note`: automated resolution started (self-heal). *Note: `agent:queued` is never applied — first mutation is `in-progress` (state machine note above).*
   - **Stage notes (FR-024):** analysis note (root-cause summary, from the diagnosis phase) → verification note (reproduced? evidence).
   - **Mid-run closed rule (FR-028):** before **every** GitLab-mutating step, `get_issue`; closed ⇒ abandonment note + `agent:failed`, stop.
   - **Success tail — the NEW delivery tail (Risk R1):** (i) health gate: typecheck + build + tests pass on the worktree branch (the same gate `self_heal_complete_fix` re-checks); (ii) **push**: enumerate changed files (`git diff --name-status <base>..HEAD` in the worktree); if any binary or payload-impractical size ⇒ note + `agent:needs-input` and stop (API push can't carry those — see trade-offs); otherwise (a) `create_branch` `fix/gitlab-issue-<iid>` from the target's default head — GitLab's Commits API requires the target branch to pre-exist, so the branch is created before any commit is pushed — then (b) `push_files`/`create_or_update_file` onto it (actions create/update/delete with full file contents); (iii) `create_merge_request { source_branch: "fix/gitlab-issue-<iid>", target_branch: <default>, title, description: "<root-cause summary>\n\nCloses #<iid>", draft: true }`; (iv) `create_issue_note` with the MR URL + final summary; (v) `update_issue` labels → classification + `agent:preview-ready`; (vi) `self_heal_complete_fix` (case → preview-ready).
   - **needs-input (FR-027):** post the specific question as a note + `agent:needs-input`, and park via the run's own question mechanism (`self_heal_request_decision`) — dual channel: the issue thread *and* the user in BOS. Resume ⇒ back to `in-progress`.
   - **failed (US4-5):** not reproducible / fix unproducible / MCP access lost mid-run ⇒ explanatory note + `agent:failed` (`agent:needs-input` instead when a config fix would unblock).
5. **Case observer.** `case_created` → map `caseId`; `fix_ready` → map mirror `agent:preview-ready` (authoritative label is GitLab-side).
6. **Human** reviews the draft MR (Promote/Discard). The system has done nothing further — NFR-004.

**Delivery-tail choice, justified:** API-based push via MCP (`push_files`/`create_or_update_file`) over `git push` of the worktree branch. Rationale: (1) it preserves the defining invariant — the MCP server is the *single* credential holder and I/O layer (D2/NFR-007); a `git push` would use a second, separate credential (the repo's remote config), silently breaking the model. (2) For this user the watched project *is* the BOS checkout (repo id 42, assumption A6), so the worktree's diff is exactly what should land. Trade-offs (flagged): text-only, per-file full-content payloads (diff-size ceiling), branch base = the watched repo's current default head (drift vs. the worktree base ⇒ conflict risk, mitigated by the file-content check → graceful `needs-input`), and no binary support.

### Feature / Enhancement (`Feature`/`Enhancement` → Build Studio spec pipeline)

1–2. Same intake; router classifies exactly one of `Feature`/`Enhancement`.
3. **Router dispatch (local only):** the router does **not** create the spec project itself (M1 — `create_project` is an agent tool that resolves its branch from the conversation's active feature branch, unset at dispatch time). It (a) computes `featureBranch = bos/gitlab-<sanitized-projectPath>-issue-<iid>`, (b) seeds a run conversation and sets its active feature branch to `featureBranch` server-side **before the first token** (self-heal `setConversationActiveFeatureBranch` precedent), (c) builds the **feature-intent brief** whose *first step* tells the run to `create_project` in the user-specs store (named `GitLab #<iid> — <title>`), and (d) `runSubAgent` fire-and-forget + the new thin run observer. Map entry `{category, featureBranch, workflowLabel:"agent:in-progress"}` (created here by the dispatch — an in-scope event, so it IS gated going forward).
4. **The run** (autonomous spec-kit: specify → design → plan → tasks → implement → preview):
   - Steps 0a/0b/0c: identical GitLab protocol (provision, closed check, pickup with `agent:in-progress` + note).
   - **Specify:** the issue's title + body *is* the confirmed intent (A2/A4 analogue for features) — the brief pre-authorizes the conversation ("run the full pipeline autonomously… for this conversation only", citing self-heal `brief.ts` as the precedent; the interactive step-stop default is untouched). Assumptions are recorded, not interviewed. **Enhancement:** the brief carries "incremental scope-add to an existing feature — locate the existing spec project the issue references; if the reference is absent or ambiguous ⇒ `needs-input` with the specific question" (the issue body is expected to name the feature; OQ-3).
   - **Stage notes (FR-024):** requirements captured (link to spec artifact) → spec drafted (link) → implementation (link to branch/artifacts).
   - **Success tail (FR-022):** built + tested preview exists on the active feature branch ⇒ final note linking the branch, the preview (BOS preview deep link) and the spec artifacts ⇒ labels → classification + `agent:preview-ready`. **No GitLab push, no MR in v1** — FR-022 requires only the preview + links; promotion is human (A9). (Contrast with the bug path, where FR-021 explicitly requires the draft MR pushed to the target project.)
   - **needs-input / failed / abandoned:** same protocol as the bug path (`renderGitLabBriefSection` is shared; only the tail differs).
5. **Case/run observer:** the run observer (new thin code — see reuse map; the feature path has no self-heal case, so it keys off run completion + a `StuckDetector` wrap instead of a case) tracks progress/stuck; terminal mirroring into the map is best-effort.

**Honest reuse map (M4):**

| Mechanism | Reused from self-heal's autonomous slow path | New (thin) |
|---|---|---|
| Autonomous brief + per-conversation pre-authorization | `brief.ts` pattern (the "do NOT stop after any step… for this conversation only" line) | feature-intent brief builder (issue-router.ts): renders intent, store/project, branch, Enhancement targeting |
| Server-side `activeFeatureBranch` before first token | yes, same mechanism | — |
| `runSubAgent` fire-and-forget | yes, same mechanism | — |
| run observer | **NO — `createRunObserver` is case-bound** (takes a `caseId`, appends runs to that self-heal case via `appendRun`, `src/lib/self-heal/runs.ts:78`). The feature path has **no** self-heal case, so it cannot reuse it. The underlying `StuckDetector` class (`src/lib/self-heal/stuck-detector.ts`) is case-agnostic and **is** reusable. | **NEW thin observer** that wraps `StuckDetector` + the run's `onEvent` without any case binding (no `appendRun`, no `setStuckSignature` on a case) |
| Terminal signaling | `self_heal_complete_fix` (bug path only) | feature path: run completion event + brief-driven terminal GitLab steps |
| Cost caps / stuck detection / slots | **None of self-heal's apply to feature runs** — the router calls `runSubAgent` directly and bypasses `selfHealIntake` entirely, so the daily cost cap (`capExhaustedForToday`), the single in-flight slow-path slot (`claimInFlightSlot`/`inFlightSlowPathCaseId`), and the case-bound observer do not run. See boundedness below. | the new thin stuck detector (above) + the platform's per-run limits, which DO apply to every `runSubAgent` run |
| Spec project creation | — | **NOT router-side** (M1): the run's first step calls the agent tool `create_project`, whose branch resolves from the conversation the router pre-set. The router only seeds the conversation + sets the branch. |
| GitLab reporting | — | `brief-section.ts` (shared by both modes) |

**Boundedness, stated honestly:** feature runs **bypass `selfHealIntake` entirely** — the router calls `runSubAgent` directly — so self-heal's daily cost cap (`capExhaustedForToday`), single in-flight slow-path slot (`claimInFlightSlot`/`inFlightSlowPathCaseId`), and case-bound run observer **do not apply** to them. What actually bounds a feature run: (1) the router's `issueKey` gate — one run per key, ever; (2) the **platform's per-run limits** (max steps / tool timeout enforced in `runSubAgent`, `src/lib/agent/subagents/runner.ts`) — these apply to every sub-agent run, feature or otherwise; (3) the **new thin stuck detector** (M2) wrapping `StuckDetector` + `onEvent`, which catches a hung run without needing a self-heal case. FR-029/NFR-005 **require** concurrent feature runs, so "no shared slot" is spec-correct — this design does not (and must not) reintroduce one. The *only* honest residual: self-heal's tuned **cost cap is a bug-sized budget that feature runs do not inherit**. If a cost guard on feature runs is wanted, it must be **added explicitly** (a feature-specific cap, or a router-side budget check) — the platform per-run limits are otherwise the only bound. Decision deferred to plan (R2); the default is to accept the platform per-run limits as the bound and add nothing new.

---

## Headless pipeline-driving mechanism

The kernel never drives GitLab. **All GitLab I/O happens inside resolving agent runs, per a GitLab-reporting protocol written into each run's brief** (model change #4). Concretely:

1. **Self-heal bug runs:** the trigger context carries `extra.gitlab`. `buildAutonomousBrief` (self-heal `brief.ts`) gains a brief-augmenter seam: after rendering the autonomous pipeline brief, it invokes registered augmenters whose matcher matches the case's trigger context; the gitlab service registers `renderGitLabBriefSection({mode:"bug", ctx})` (registered by the gitlab barrel at startup — new `src/lib/self-heal/brief-augmenters.ts` registry, one-line call in `brief.ts`). The section content is fixed protocol text (see §Resolution flow, step 4) — deterministic, no kernel involvement at run time.
2. **Feature runs:** the router's feature-intent brief (built in `issue-router.ts`) embeds `renderGitLabBriefSection({mode:"feature", ctx})` directly — no self-heal involved. The router pre-sets the run conversation's active feature branch; the brief's **first step** is for the run itself to call `create_project` (the router never invokes that agent tool — M1), so the project lands on the same `bos/gitlab-…` branch the run's spec artifacts ride.
3. **Ambiguity / validation runs:** one-shot runs whose entire brief is `renderGitLabBriefSection({mode:"ambiguity"|"validate", ctx})`.
4. **Precedent cited:** self-heal `brief.ts` establishes (a) per-conversation autonomous pre-authorization that leaves the interactive step-stop default untouched, (b) server-side `activeFeatureBranch` set before the first token — the feature path reuses exactly this, so the run's first `create_project` resolves the branch from the conversation and the project/feature/spec artifacts stay on one branch (M1), (c) fire-and-forget `runSubAgent` with a run observer, (d) a terminal tool call (`self_heal_complete_fix`) the platform re-checks before reporting. This design reuses (a)–(d) verbatim and adds nothing to the run engine — only brief *content* is new.

---

## Security & invariants

- **NFR-001:** `verify()` uses constant-time comparison — a **local** `timingSafeStringEqual` (wrapping `node:crypto`'s `timingSafeEqual`, mirroring the Telegram precedent's own local helper; `webhooks/verify.ts` exports only the HMAC `verifySignature`, no token helper) — over `[extras.token?, primary, previous]`; no candidates ⇒ reject, missing header ⇒ reject (a **conscious, deliberate divergence** from the Telegram precedent, which returns `true` when no secret exists to avoid bricking — `bot-webhook.ts:110`; here the platform always mints a primary secret on enable, so an empty-candidate state must not silently accept). 404-not-401 for disabled/disconnected/unknown is the framework's (`route.ts`) — the response does not distinguish them (US2-4). Neither the receiver nor the handler logs or echoes the delivery body or the token (handler discipline; the framework already avoids it). 404-not-401 for disabled/disconnected/unknown is the framework's (`route.ts`) — the response does not distinguish them (US2-4). Neither the receiver nor the handler logs or echoes the delivery body or the token (handler discipline; the framework already avoids it).
- **NFR-002 / FR-012:** two independent layers — the framework idempotency ring (size 1000, body-hash fallback since GitLab sends no idempotency header; byte-identical retries dedupe) and the router's `issueKey` map (a deduped event that is still new for the key is processed once; any later event for a mapped key is a no-op). SC-003 follows.
- **NFR-004 (propose-only):** terminal state is `agent:preview-ready`; no merge/close/promote anywhere. Enforced twice: (1) the brief protocol enumerates the *complete allowed mutation set* — `get_issue`, `list_labels`, `create_label`, `create_issue_note`, `update_issue` (**labels only — never `state`**), `create_merge_request` (**`draft: true` only**), `create_branch` (target branch must pre-exist before the Commits API can push), `push_files`/`create_or_update_file` — and explicitly forbids closing the issue, removing classification labels, and any "promoted"/"merged" label; (2) structurally, the design contains **no** merge/close/promote call site, and the kernel has no GitLab credential to make one. SC-008 follows.
- **NFR-005 / FR-029:** no shared mutable pipeline state across issues — each run is its own conversation + worktree/branch; the map is keyed per `issueKey` and written under the per-integration mutex (atomic, serialized).
- **FR-030 (self-noise) — why it holds, and why author-checking is both unnecessary and insufficient:**
  - (a) The router keys off the **classification labels in the payload**, which BOS never writes — the allowed mutation set only ever adds/removes `agent:*` labels and posts notes. BOS's own label writes therefore never create or destroy an in-scope classification condition.
  - (b) Even so, BOS's `update_issue` triggers a GitLab *issue* hook (label change), which *is* delivered with the unchanged classification label present — but the `issueKey` idempotency gate makes it a no-op. (Note events don't arrive at all: the mockup registers **Issues events** only, so `object_kind: note` is never delivered.)
  - (c) An issue closed mid-run is handled at the *run* level (FR-028 get_issue checks), not the router.
  - **Author-checking is insufficient** because the agent's GitLab identity is the *user's own credential* (the MCP server is configured with the user's token) — BOS's notes/label changes appear authored by the same human whose real issue updates must be processed, so author filtering would both fail to filter self-noise and risk filtering genuine user activity. The design deliberately doesn't attempt it.
- **Credential hygiene:** the kernel holds no GitLab credential (NFR-007); the webhook token lives in the SecretsStore (`webhook:gitlab/issues` key family) with rotation grace (primary→previous, `manager.ts`); the verification token is user-visible only via the masked/reveal UI (mockup).

---

## Testing strategy

All e2e/driver tests carry the **M5 environmental guard**: a setup hook checks (1) a gitlab MCP server is configured in the registry, (2) a `list_labels` round-trip for the test project succeeds — via the same one-shot validation path the panel uses. Absent either ⇒ **fail fast with a clear message / skip — never hang** (bounded timeout on the validation run). All tests are **self-cleaning per the 034 convention**: temp data dirs for `data/integrations/gitlab/state.json` and the self-heal case store; reset seams clear the `issues` map, `lastReceived`, and `connection` between cases; teardown deletes test issues/MRs it created (labels are idempotent and left — documented).

**Unit**
- `verify`: correct token ⇒ accept; wrong/missing ⇒ reject; previous token accepted during rotation grace; new primary accepted after rotate; constant-time helper used (no early-exit string compare).
- `receive`: `object_kind: issue` open/update/close → correct event shape, **all IntegrationEvent fields present incl. `service:"gitlab/issues"` + `timestamp`** (M1 guard — assert the derived bus type is `com.bos.gitlab.issues.issue-opened`); non-issue object_kind (note, merge_request) → `[]` (FR-013); malformed JSON → `[]` + ack.
- Label vocabulary: exactly 8 constants; classification set `{Bug, Feature, Enhancement}` exact-casing; workflow set exact; the two sets disjoint; no promoted/merged member.
- Router dispatch: 0 classification labels ⇒ no-op with **zero** emitted events and zero GitLab-touching calls **and zero map entries** (FR-020/SC-009); 1 label ⇒ correct trigger context (Bug: `toolName` shape, `extra.gitlab` present, **no `selfHeal.role` marker**); >1 ⇒ `com.bos.gitlab.issues.ambiguity` emitted once (deduped on repeat, re-emitted only when the label *set* changes); already-mapped key ⇒ no-op (SC-003); `action: close` event for a mapped key ⇒ no-op at router (run-level FR-028 is run-side). **FR-014:** a 0-label event creates **no** map entry; a subsequent single-label event for the *same* key **does** dispatch (it was never gated).
- `create_project` dispatch (M1): the router does NOT call the `create_project` tool; it pre-sets the run conversation's active feature branch, and the run's first `create_project` resolves that branch — so the project folder and the run's spec artifacts land on the **same** `bos/gitlab-…` branch (assert no base-branch project folder).
- `resolution-state`: serialized writes (concurrent `mutateState` calls don't interleave/corrupt); per-key isolation; atomic write (no partial file on simulated failure).
- `renderGitLabBriefSection`: bug/feature/ambiguity/validate modes render; bug mode contains the allowed-mutation whitelist (incl. `create_branch`, which must appear before `push_files`/`create_or_update_file`) and the explicit no-close/no-merge prohibitions; feature mode contains no `create_merge_request`/push instructions; closed-mid-run rule present in both.
- ServiceConfigView wiring: gitlab/issues renders `GitLabIssuesSection`; enable gated on `state.connected === true` (the top-level receiver boolean — `config.extras.validation.status` is panel display only); not-configured variant shows the CTA (mockup variant B).

**E2E (webhook → bus → pipeline)**
1. **Valid delivery:** POST well-formed issue hook (token correct) ⇒ 200 + exactly one bus event with the asserted shape; `lastReceived` stamped (FR-006).
2. **Duplicate:** byte-identical re-POST ⇒ 200, no second event, no second run (NFR-002, SC-003).
3. **Auth:** wrong/missing token ⇒ 401, no event (US2-3).
4. **Disabled/disconnected:** disabled service ⇒ 401-indistinguishable 404, no event (US2-4); not-connected (MCP unvalidated) ⇒ same 404 + service cannot be enabled (US1-2).
5. **Non-issue object_kind:** no event (FR-013, US2-5).
6. **Driven Bug to terminal:** seed a `Bug`-labeled issue (test fixture via MCP) ⇒ webhook ⇒ bus ⇒ router ⇒ self-heal case ⇒ agent run (MCP against the test project) ⇒ assert, in order: pickup note + `agent:in-progress`, analysis note, verification note, draft MR exists **in draft**, issue↔MR linked, terminal `agent:preview-ready` (SC-005, NFR-004: MR unmerged, issue unclosed).
7. **Driven Feature to terminal:** `Feature`-labeled issue ⇒ spec project created ⇒ autonomous spec-kit run on `activeFeatureBranch` ⇒ stage notes (requirements/spec/implementation) with links, built+tested preview on the branch, terminal `agent:preview-ready`, **no MR** (SC-006).
8. **Out-of-scope / ambiguous:** unlabeled issue ⇒ 0 runs, 0 GitLab writes (SC-009); double-labeled ⇒ 0 runs + exactly one ambiguity note (SC-004).
9. **Self-noise:** during test 6/7, the agent's own label updates arrive back as issue hooks ⇒ assert no re-dispatch (FR-030).
10. **Closed mid-run (FR-028):** drive a bug run, close the issue during verification ⇒ run stops, abandonment note, `agent:failed`, no MR.

---

## Biggest risks & open questions

- **R1 — the bug-path delivery tail is genuinely new.** Self-heal has never pushed to an external project or opened an MR. Scope it precisely: `create_branch` a new `fix/gitlab-issue-<iid>` branch (GitLab's Commits API requires the target branch to pre-exist), push a text-only diff onto it via `push_files`, then open a draft `create_merge_request`. Concrete failure modes: (a) diff size / per-file content payload limits; (b) binary files (unsupported by API push ⇒ brief routes to `needs-input`, not silent failure); (c) **base drift** — the API branch is created from the watched repo's current default head, while the worktree was based on an older commit ⇒ stale-file conflicts; the brief must verify target-file contents before applying and degrade to `agent:needs-input` with an explanatory note. Mitigation: land this as the first e2e-driven spike (test 6) before any other polish.
- **R2 — feature runs have no self-heal cost cap to inherit.** Feature runs bypass `selfHealIntake`, so self-heal's daily cost cap, the single in-flight slow-path slot, and the case-bound observer do **not** apply (boundedness analysis above). The only honest residual: self-heal's tuned cost cap is a **bug-sized budget that feature runs never inherit** — by default there is no cost guard on a feature run beyond the platform's per-run limits (max steps / tool timeout, `runner.ts`). Decision at plan time: (a) accept the platform per-run limits as the sole cost bound, or (b) add an explicit feature-run cost guard (a router-side budget check / feature-specific cap). Note FR-029/NFR-005 *require* concurrent feature runs, so the single in-flight slot must **not** be reintroduced — "no slot" is spec-correct, not a gap. Default: (a); choose (b) only if a bug-sized cap would clearly be needed. Flagged, not hidden.
- **R3 — A1/A6 reachability & watched-project assumptions.** Push-only delivery requires GitLab → receiver URL reachability (spec A1); the bug path assumes the watched project is a repo the pipeline can build in and push to (A6; for this user: the browseros checkout, repo id 42). A non-buildable project must end in `agent:failed`/`agent:needs-input` with an explanatory note (spec) — the brief's step 0 must detect "this project ≠ the buildable repo" early (e.g. `get_issue`/`list_labels` succeed but the repo's default branch doesn't contain the worktree's files ⇒ base-drift path of R1 ⇒ graceful failure).
- **R4 — ambiguity handling under the I/O-free router (M3, decided: option (a)).** A lightweight `com.bos.gitlab.issues.ambiguity` bus event + a thin one-shot note-posting run. Trade-off: one extra (cheap, rare, deduped) run in exchange for a strictly I/O-free router; the alternative — a kernel-side GitLab write — is exactly what the corrected model forbids.
- **R5 — what "validated" means operationally, and its failure modes.** Defined as a `list_labels`/`get_issue` round-trip for the target project via a one-shot agent run (connection state machine, §Data model). Failure modes: (a) credential valid but *project* missing ⇒ `unavailable` with the project named — panel must surface which check failed; (b) validation run itself dies/hangs ⇒ bounded by the run observer + a timeout, after which status stays `unvalidated` (never falsely `validated`); (c) stale validation (credential revoked later) is caught by the run-level failure path flipping status to `unavailable`.
- **R6 — MCP tool availability inside headless sub-agent runs.** The entire resolution half assumes **gitlab-mcp tools are exposed to `runSubAgent` children** (the same MCP surface as interactive agents — configured servers in Settings attach to runs). If they are not: every run's GitLab protocol fails at step 0, all issues end in `agent:failed`, and the feature is non-functional end-to-end despite working intake. This is the design's load-bearing assumption — **verify in the first implementation spike** (run one validation sub-agent before writing any other run code). No fallback exists by design (the kernel holds no credential to fall back to — that would break D2/NFR-007).
- **R7 — self-heal interaction surface.** (a) Reentrancy: the trigger must never carry `selfHeal.role` (genuine external trigger) — unit-tested; (b) dedupe: `toolName` embeds the issue id so two gitlab bugs with identical titles don't collapse — unit-tested; (c) the brief-augmenter seam is the only self-heal-side change — it must be a no-op for all existing (non-gitlab) cases, guarded by a regression test; (d) a concurrent *manual* self-heal case and a gitlab bug run are independent (separate worktrees/cases) but share the platform's run slots — priority is unspecified (see R2).
- **R8 — state-mirror divergence.** The map says `in-progress` at dispatch, but the actual label is applied by the run's first mutation; a run that dies before step 0c leaves a stale non-terminal entry (issue un-labeled in GitLab, map busy). v1 accepts this **for entries that exist** (created by an in-scope dispatch or an ambiguity record): they are never auto-cleared, because re-triggering a finished issue is out of scope. Critically, an **out-of-scope (0-label) issue has no entry at all**, so the "never auto-cleared" rule does **not** trap it — a later classification label on that issue is picked up per FR-014 (it was never gated). A future cleanup job (stale non-terminal entries older than N days ⇒ clear) is noted, not designed.
- **R9 — `update_issue` label-replacement semantics.** GitLab replaces the *entire* label set on update; the brief protocol mandates "read current labels via `get_issue`, set = classification labels + exactly one `agent:*`". A run that forgets to merge would strip the classification label — which (harmlessly, per FR-030(a)) ends the system's interest in the issue but corrupts the human's categorization. Guarded by the brief's explicit rule + the e2e label assertions.
- **OQ-1 (spec gap):** abandonment (FR-028) has no dedicated workflow label; this design maps it to `agent:failed` with an explicit abandonment note. Confirm with the user at plan review.
- **OQ-2:** receiver URL origin — `webhookUrl(id, sid, origin)` takes the origin as a parameter; the panel passes the app origin (`NEXT_PUBLIC_APP_ORIGIN` per mockup foot note). Trivial, but must be sourced consistently with how the Telegram/Gmail panels do it.
- **OQ-3 (Enhancement targeting):** how the run locates "which existing feature" for `Enhancement` issues — v1: the issue body names it; unresolvable ⇒ `needs-input`. Whether the router should pre-resolve the target spec project (a local spec-store lookup at dispatch) is left to the plan — the brief-based approach is the simpler v1.
- **R0 (verification, closed this run):** confirmed by source read — `IntegrationEvent = { type: string; service: string; timestamp: number /* epoch millis */; data: Record<string, unknown> }` (`src/lib/integrations/types.ts`); `emitIntegrationEvent` lives at `src/lib/events/from-integration-event.ts` and adds `_integrationType`, `_integrationService` and `summary` to the emitted bus payload (none is part of the 4-field `IntegrationEvent`). Consequences folded in: `receive()` returns the 4-field shape; `summary` is derived at emission; the receiver route file is `src/app/api/integrations/webhooks/[integrationId]/[serviceId]/route.ts`. The `deriveLegacyType` split-on-`service` behavior (M1) is taken as given from the verified facts.
