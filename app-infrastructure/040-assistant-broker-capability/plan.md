# Implementation Plan: Assistant Broker Capability

**Branch**: `bos/040-assistant-broker-capability` | **Date**: 2026-08-25 | **Spec**: [spec.md](./spec.md) | **Design**: [design.md](./design.md)

**Input**: Feature specification from `spec.md`; architectural decisions and the broker method contract from `design.md`.

## Summary

Add an `assistant` app capability so opaque-origin (marketplace) apps can drive the assistant run API through the sandbox postMessage broker. The parent frame (trusted, same-origin) becomes a *viewer* of each run's existing server-side NDJSON stream and relays events into the child iframe via a new unsolicited `__bos_event` postMessage — so an app that cannot `fetch` BOS APIs cross-origin can still start runs, stream replies, and participate in frontend tool calls. Six broker methods map 1:1 to the HTTP run API; the capability is **declaration-gated** (ADR-6) and wired through the existing four-places (+ SDK) capability pattern.

The technical approach, transport decision (parent-push + bounded buffer + authoritative re-fetch), ADRs, and the full broker method contract are in [design.md](./design.md) — this plan references them rather than restating them.

## Technical Context

**Language/Version**: TypeScript, ES2020+, the existing Next.js 14 App Router client/server split.

**Primary Dependencies**: none new — reuses the in-browser `fetch` `ReadableStream` reader (the `run-client.ts` pattern) and the existing postMessage broker protocol.

**Storage**: N/A — the broker holds transient in-memory per-run buffers; run state itself stays in the server's `runManager()` singleton.

**Testing**: Playwright e2e (the repo's existing e2e suite) for the broker path + capability gate; `tsc --noEmit` strict typecheck. A same-origin app already exercises the run API; the e2e target is an opaque-origin (marketplace) app driving the assistant through the broker.

**Target Platform**: Browser (BOS parent frame + sandboxed app iframe); Node 18+ server (Next.js route handlers).

**Project Type**: Web application (client + server, same repo).

**Performance Goals**: event-delivery latency through the broker ≤ 500 ms over direct HTTP (NFR-001); capability-denied rejection ≤ 100 ms (SC-002).

**Constraints**: no CORS headers added anywhere; no new external dependency; no new service/container/port (ADR-5); backward compatible with all existing direct-HTTP consumers (NFR-004).

**Scale/Scope**: 6 source files modified/created (see Project Structure) + 1 companion doc. Bounded per-run broker buffer (~2000 events / ~1 MB default, tunable).

## Constitution Check

*Gates from `/Specs/bos-system-specs/.specify/memory/constitution.md`. Re-verified against design.md.*

- **II. Server Authority & SSR Boundary** — ✅ PASS. The run API stays server-only. The broker is a client-side same-origin relay; it adds no CORS headers, leaks no secrets, and the parent's per-run buffer is a cache of data already public to the BOS origin.
- **IV. Minimize Blast Radius** — ✅ PASS. Additive capability + additive broker methods on a feature branch. Existing capabilities and direct-HTTP consumers untouched.
- **VI. Specs & Docs Stay in Sync** — ⚠️ CONDITIONAL PASS. Requires the companion dev doc (`docs/dev/assistant/assistant-broker.md`). The pre-existing `storage`/`VALID_CAPS` drift is recorded, not fixed here.
- **Technology Constraints** — ✅ PASS. Follows the existing sandbox/capability pattern; no new dependency, no new container.

**No violations** — no Complexity Tracking entries required.

## Project Structure

### Documentation (this feature)

```text
Specs/user-specs/app-infrastructure/040-assistant-broker-capability/
├── spec.md          # (written — specify)
├── design.md        # (written — design, with ADRs + broker method contract)
├── plan.md          # (this file — plan)
├── tasks.md         # (next step — tasks)
└── checklists/
    └── requirements.md  # (written — specify)
```

No separate `research.md`/`data-model.md`/`contracts/` — the design already captures the broker method contract (design.md §6) and there is no new persisted data model. A single companion **dev doc** is added in-repo (Constitution VI):

```text
docs/dev/assistant/assistant-broker.md   # the capability, broker methods, parent-push+buffer model, four(+SDK) places, reconnect semantics, isolation rule
```

### Source Code (repository root)

```text
src/
├── os/
│   └── types.ts                          # MODIFY: add "assistant" to AppCapability
├── app/
│   └── api/
│       └── apps/[id]/capabilities/route.ts  # MODIFY: VALID_CAPS += "assistant"; PUT rejects assistant if manifest undeclared (ADR-6)
├── components/
│   └── apps/
│       ├── assistant-broker.ts            # CREATE: client-side stream owner — module-level Map<appId, AppBroker>, refcounted windows, per-run tails, bounded buffer, authoritative re-fetch, owned-run isolation, six methods
│       ├── IframeApp.tsx                  # MODIFY: CAP_FOR_METHOD += assistant:* → "assistant"; route assistant:* to getBroker(appId) post cap-gate; refcount + dispose on last-window unmount
│       └── settings/
│           └── AppsTab.tsx                # MODIFY: add assistant row, rendered conditionally (manifest declares it — ADR-6)
└── lib/
    └── iframe-sdk/
        └── index.ts                       # MODIFY: window.__bos.assistant.* group + __bos_event listener (returns unsubscribe); keep dependency-free
```

**Structure Decision**: This is a focused addition to the existing BOS monorepo — no new package, app, or container. One new client module (`assistant-broker.ts`) holds all the stream/buffer/registry state; the other five files are small, targeted modifications to the existing capability + broker + SDK wiring. The unchanged assistant routes and `run-manager.ts` are integration points (design.md §5), not deliverables.

## Design Notes (pointers to design.md)

- **Transport** — parent-push + bounded per-run ring buffer + per-child cursor + authoritative server re-fetch for stale cursors (ADR-1, ADR-2). §3.5.
- **Isolation & lifecycle** — module-level `Map<appId, AppBroker>` with refcounted window tracking and per-app owned-run sets; survives iframe reload and mid-run capability revoke (ADR-3). §3.4, §3.5.
- **Capability gating** — declaration-gated grant (ADR-6): conditional Settings row + server-side `PUT` rejection when the manifest is silent. §3.7.
- **Broker method contract** — the six methods with exact params/response/error mapping, 1:1 with the HTTP API. §6.
- **Surface tools & first-claim-wins** — tools ride the `startRun` body via unchanged `startAssistantRun`; claim semantics delegated to unchanged server code. §3.6.
- **Risks** — browser connection count per run, page-unload/long-gap 404 fallback, single-tail ordering constraint, `storage` drift (do-not-copy), SDK surface growth, information-disclosure disclosure, mid-run surface-tools deferral. §8.
