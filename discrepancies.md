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

## Context compaction (022) has a soft dependency on 021 memory-loops

`022-context-compaction` is authored as a self-contained view transform on the
model input array — it does **not** require `021-memory-loops` to run. But when
021 is installed, the compaction summarizer invokes `runFastLoop({ onlyConversationId,
waiveIdle: true })` **before** persisting the summary so durable lessons from
the span-about-to-be-compacted hit the memory store first ("write before
compaction"). If the module is absent, compaction logs `fast-loop.skipped`
(once per conversation) and proceeds. Recovery of pre-summary facts on later
turns is via `memory_search` (021 tool surface) — spec 022 US-4.3, FR-014.

Authoritative: `022-context-compaction` FR-014; recovery path: US-4.3.

## Memory app (023) API surface drift

Spec `023-memory-app` FR-030 lists a set of API endpoints the redesigned Memory
app must consume. The following diverge from the implemented routes; the app
must be written against the code, and the spec should be treated as intent
rather than literal contract until a follow-up updates it:

- **Config API is `PATCH`, not `POST`.** `src/app/api/config/route.ts` exports
  `GET` + `PATCH`; the body shape is `{ namespace, values, secretsSet? }` and
  the response is `{ namespace, values, secretsSet }` after re-masking secrets.
  Spec FR-030 says `POST /api/config` with `{ namespace, values }` — read that
  as PATCH.
- **Logs filter is `component`, not `category`.** `src/app/api/logs/route.ts`
  accepts `component=` (plus `session`, `stream`, `level`, `conversation`,
  `since`, `limit`). Loop run history for spec `023` FR-023 must query the
  actual component names emitted by the loops: `memory.fast-loop`,
  `memory.slow-loop` (and related — see `src/lib/agent/memory/*` for
  authoritative component strings).
- **Search result navigation (FR-029) is deferred.** The search endpoint
  returns provenance strings, but the redesigned app does not currently
  navigate to the source entry in the appropriate tab. Tracked as a Phase-8
  follow-up; results render read-only.

Authoritative code paths: `src/app/api/config/route.ts`,
`src/app/api/logs/route.ts`, `src/app/api/memory/search/route.ts`.
