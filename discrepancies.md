# Discrepancies — where code diverges from the specs

Living notes on where the implementation currently differs from the written specs.
Keep entries short; link the authoritative spec.

## Per-conversation agent (supersedes "global active agent")

Specs `011`, `012`, and `016` describe a **global active agent** (e.g. "the active
agent's allowed tools", "regardless of the globally active agent"). The
implementation has **no global active-agent state**: each conversation carries its
own `agentId`, and `composeInstructions(agentId)` throws on an empty id rather than
falling back. `DEFAULT_AGENT_ID = "assistant"` survives only for delete-protection
and blank/bootstrap seeding. Authoritative: `019-tools-and-sandbox` FR-003/FR-004.
The "active agent" phrasing in `011`/`012`/`016` should be read as "this
conversation's agent."

## Capability registry context enum

`016` FR-003 describes contexts as `client` and/or `server`. The implemented
`capabilities-registry.ts` uses a single `context: "action" | "tool" | "both"`
field (action = client, tool = server, both = both surfaces). Semantically
equivalent; naming differs. Authoritative: `019` FR-001.

## Sandboxed `run_command` replaces `runBash`

Earlier specs assumed an unsandboxed `runBash`. It is replaced by `run_command`
(docker/local backends, off by default, VFS-backed `/workspace`). Authoritative:
`019` FR-006..FR-009. Dev doc: `docs/dev/run-command/run-command.md`.

## GEPA is "GEPA-lite"

`003-self-improvement` `skill_improve` is a single reflective rewrite recording a
self-reported score, not the full GEPA loop (candidate generation + evaluation +
Pareto selection + versioned rollback). See `003-self-improvement`.

## Spec-store mount & candidate model reworked by 020

`018-external-spec-store` FR-005 (global `spec-candidate` branch + per-store
promote/discard) and FR-007 (read-only symlink mount of the stores into the
preview worktree) are superseded: the symlink broke Turbopack, and harness
writes were in practice write-through, not inert. `020-branch-coupled-specs`
replaces both — stores are mounted as git worktrees on the code's feature
branch; direct edits commit-on-save; review/promote of specs is coupled to the
code promote. Authoritative: `020` FR-001..FR-007.

## 040-okf-knowledge-base — convergence pass (2026-08-15)

The installed `okf-knowledge-base` marketplace item passed its e2e suite (8/8,
see `user-specs/040-okf-knowledge-base/test-results.md`) but a spec/design audit
against the as-built `services/lib/http-bridge.js`, `services/lib/tools.js`, and
`app/config-app/index.html` found three gaps + one drift, now closed in a
converge staging dir (`/tmp/okf-knowledge-base-converge`, pending `app_build`):

- **Tool count: 28 vs. 29 — resolved as "no undocumented extra tool."**
  `design.md` Risk #3 and `plan.md` approximated the declared tool count as
  "≈28"; the actual implementation (`services/lib/tools.js` `DECLARATIONS`)
  declares **29** `okf_*` tools, which matches the full tool table already
  present in `design.md` §"Tool contracts" / `tool-surface.md` §3 when summed
  (5 bundle/namespace + 5 concept CRUD + 7 graph + 5 ingest/query/lint + 3
  temporal/lifecycle + 4 provenance/trust = 29). There is no rogue/undeclared
  extra tool and no `okf_query`/`okf_add_source_document` tool was added —
  `design.md`, `plan.md`, and `tool-surface.md` were updated to state 29
  explicitly instead of "~28".
- **FR implementation differs from spec text — design.md §5 "Graph view" had
  no backing HTTP route.** `design.md` maps the config-app's Graph view to
  `okf_list_links`/`okf_backlinks`/`okf_neighbors` "over the bridge", but
  `services/lib/http-bridge.js` had no `/links`, `/backlinks`, `/neighbors`,
  or single-concept-read routes — only the `okf_*` tools (agent-facing)
  implemented that behavior, not the browser-facing bridge the config-app
  actually calls. Added `GET /bundles/<name>/links|backlinks|neighbors?concept=<id>`
  and `GET /bundles/<name>/concepts/<id>`, wired to the same okf-core graph
  functions the tools use, plus a config-app Graph tab.
- **FR implementation differs from spec text — no raw-source "add" path
  existed (design.md Risk #7).** `okf_ingest` reads from `raw/` but nothing
  wrote into it from the HTTP bridge/config-app; `design.md` had left this an
  open risk. Added `POST /bundles/<name>/raw` (`{ path, content }`,
  append-only — rejects overwrite and path traversal) + a config-app "Add
  Source" dialog. Kept HTTP-only (no new `okf_*` tool) per FR-scope guidance
  in `design.md`/`plan.md`, so the tool count stays at 29; agents still add
  sources via `okf_ingest`/`okf_import_bundle`.
- **FR-006 (039 FR-009 loopback-scoped auth) was unasserted by any test.**
  `services/lib/http-bridge.js` bound `127.0.0.1` only but had no explicit
  request-origin check and no e2e coverage. Added an `isLoopbackAddress`
  check against `req.socket.remoteAddress` plus rejection of any
  `X-Forwarded-For`/`X-Real-IP` header naming a non-loopback address (403),
  and an e2e assertion (`GAP3` test in `e2e/040-okf-knowledge-base.spec.ts`)
  that a non-loopback-presenting request is rejected while a loopback
  request succeeds.

Authoritative: `user-specs/040-okf-knowledge-base/{design.md,plan.md,tool-surface.md,tasks.md}`
(Phase 8, "Convergence" section of `tasks.md`).
