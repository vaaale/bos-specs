# Implementation Plan: VFS Mount (WebDAV)

**Branch**: `033-vfs-mount` | **Date**: 2026-08-01 | **Spec**: `bos-system-specs/033-vfs-mount/spec.md`

**Input**: Feature specification from `/specs/bos-system-specs/033-vfs-mount/spec.md`

## Summary

A WebDAV service exposes the user's VFS (`data/vfs/`) as a local-mountable drive via RFC 4918. macOS Finder, Linux `davfs2`, and other WebDAV clients can mount the service for local-filesystem semantics. The feature is a marketplace-distributed service item (with a companion Settings panel) that activates only when the user generates a mount token.

**Critical architecture note**: The spec assumes the WebDAV endpoint should live in `src/middleware.ts` because "Next.js route handlers reject non-standard WebDAV verbs." This is a false premise — a WebDAV service runs as a plain Node worker thread on its own port (per `target-marketplace-service.md`), completely bypassing Next.js's request pipeline. The service is self-contained, has no `src/` involvement, and is installed as a marketplace service item. The spec's "App Target: marketplace-app" header is misleading; the primary facet is `services/` with a companion `app/` for the Settings panel.

## Technical Context

**Language/Version**: Node.js (plain worker thread, no Next.js/webpack)

**Primary Dependencies**: `node:http` + `node:fs` + `node:path` + `node:crypto`; XML templating for WebDAV responses

**Storage**: User VFS at `data/vfs/` — all I/O MUST go through the VFS layer, not raw `fs` calls (spec FR-001, FR-005)

**Testing**: Node's built-in `node:test` or `jest`/`vitest` if available; integration tests against a WebDAV client (`curl` or a mock WebDAV client)

**Target Platform**: Any OS running BOS — the service binds to a configurable host/port

**Project Type**: Background daemon (service) with a companion Settings panel app

**Performance Goals**: Stream file reads/writes (no full-file memory buffering); handle >100 MB files; ≤50 MB RSS headroom for large uploads (SC-006)

**Constraints**: 
- Service runs outside Next.js (worker thread), so it cannot import `src/lib/...` TypeScript modules directly
- All I/O goes through the VFS layer — direct filesystem access is forbidden (FR-001, FR-005)
- Path traversal (`../`, encoded variants, `./`, `//`, Windows separators) must be rejected with `403`
- `PROPFIND` depth-1 responses bounded at 5000 children (FR-016)
- The service must be opt-in — no token = 404 (FR-012)

**Scale/Scope**: Single-user deployment; multiple concurrent mounts from different machines

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **I. Spec-Driven** ✅ — spec.md exists and is the source of truth
- **II. Server Authority & SSR Boundary** ✅ — all Node-side logic lives in the service, secrets/Node APIs stay server-side
- **III. Always Delegate** ✅ — implemented via `agent_delegate` + `app_build` (marketplace service)
- **IV. Minimize Blast Radius** ✅ — no BOS source changes; runs in `data/user-apps/items/<id>/services/`
- **V. The VFS Is Not the Source** ✅ — service uses the VFS layer, not raw fs calls
- **VI. Specs & Docs Stay in Sync** ⚠️ — spec.md says "middleware" but correct architecture is a service; spec needs updating
- **VII. Respect Boundaries** ✅ — no changes to `package.json`, lockfiles, or build config

## Project Structure

### Documentation (this feature)

```text
bos-system-specs/033-vfs-mount/
├── spec.md                 # Feature specification
└── plan.md               # This file — implementation plan
```

### Marketplace Item (what the Developer writes)

```text
vfs-webdav-mount/
├── services/
│   ├── service.json          # {id: "vfs-webdav-mount", entry: "index.js", configSchema, settingsRegistration}
│   └── index.js            # The WebDAV daemon — plain Node, handles all HTTP methods
│
├── app/
│   └── src/main.tsx        # Companion UI: Settings → Integrations → Local Mount panel
│                         # Provides: token generation/revocation, mount instructions
│                         # Loaded by BOS settings UI at Settings → Integrations → Local Mount
│
└── config/
    └── vfs-webdav.json     # User-editable config (seeded from item defaults)
```

**Structure Decision**: This is a **marketplace-service** item (background daemon) with a companion app facet (Settings panel). The service handles all WebDAV HTTP verbs (GET, PUT, DELETE, MKCOL, COPY, MOVE, PROPFIND, OPTIONS, HEAD) and exposes the VFS root via a configurable port. The Settings panel provides the UI for token lifecycle management and mount instructions. This structure mirrors the Terminal service precedent (see `target-marketplace-service.md`) — no `src/` involvement, fully self-contained Node code.

**Spec discrepancy note**: The spec.md declares `App Target: marketplace-app`, but this feature's primary implementation mechanism is `marketplace-service` (the Settings panel is secondary). The `app/` facet exists only as a companion UI. The spec should be updated to `marketplace-service` to reflect the correct implementation path.

**Note on spec discrepancy**: The spec's Assumptions & Dependencies section states the WebDAV endpoint MUST be implemented in `src/middleware.ts`. This is a false premise — a service daemon binds its own port and doesn't go through Next.js. The correct implementation lives in `services/` as a plain Node worker thread. The spec should be updated to reflect this. This is noted but the spec header's "App Target" is still `marketplace-app` (per spec.md), while the actual implementation is a `marketplace-service`. For the plan, I'll plan the correct architecture (service).

## Design Notes

### WebDAV Implementation

The service implements the minimum WebDAV (RFC 4918) subset required for macOS Finder + `davfs2` compatibility: `OPTIONS`, `PROPFIND` (depth 0 and 1), `GET`, `PUT`, `DELETE`, `MKCOL`, `COPY`, `MOVE`, `HEAD`. `LOCK`/`UNLOCK` MAY be no-ops returning `200`.

The handler must:
- Route all HTTP methods to a single path prefix (`/api/vfs/webdav/[...path]` is the URL, but the service binds its own port — so the actual URL is `<host>:<port>` with the path relative to the VFS root)
- Parse and dispatch: `OPTIONS` (returns `DAV: 1, 2` + `Allow:` header), `PROPFIND` (XML multi-status), `GET`/`HEAD` (file content with content-type), `PUT`/`MKCOL` (write to VFS), `DELETE` (remove from VFS), `COPY`/`MOVE` (server-side)

### Authentication

All requests require `Authorization: Bearer <token>`. Unauthenticated requests receive `401` with `WWW-Authenticate: Bearer realm="BOS"`. Tokens are stored hashed in the service's own config (or an encrypted store) with AES-256-GCM encryption at rest.

### VFS Layer Integration

The service delegates I/O to the VFS layer. Since it runs outside Next.js, it cannot import `src/lib/...` TypeScript modules. However, the spec explicitly states I/O must go through the VFS layer, not raw `fs` calls. This requires either:
1. The VFS layer to be made importable from outside the module graph, OR
2. The service to use the VFS through a different mechanism (e.g., a shared memory channel, or direct filesystem access to `data/vfs/` with path-traversal guard)

Given the spec's explicit dependency on `006-data-isolation` and `007-gitfs`, the service must interact with the VFS in a way that respects ownership and GitFS semantics. This is a non-trivial architectural decision that needs to be resolved before implementation.

**Open question**: How does a worker-thread service access the VFS layer without importing TypeScript modules from `src/`? The service must respect VFS ownership, GitFS semantics, and data isolation. This may require a dedicated VFS client or an abstraction that the service can use.

### Settings Panel

The Settings panel (loaded at `Settings → Integrations → Local Mount`) provides:
- Token generation (cryptographically random 256-bit values)
- Token revocation (independent, no restart needed)
- Mount instructions for macOS (`mount_webdav`) and Linux (`davfs2`)
- The `use_locks 0` note for `davfs2` config

### Deployment Modes

The service must function in three deployment scenarios:
1. **Docker Compose + Bastion**: exposed via Bastion's HTTPS listener, SSO/OAuth auth
2. **Standalone Docker**: exposed on routable IP/port, bearer token auth
3. **Local development**: exposed on `localhost`, bearer token auth

## Complexity Tracking

No complexity violations requiring justification — the spec is already aligned with the constitution.
