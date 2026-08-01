# Implementation Plan: VFS Mount (WebDAV)

**Branch**: `033-vfs-mount` | **Date**: 2026-08-01 | **Spec**: `./spec.md`

**Input**: Feature specification from `bos-system-specs/033-vfs-mount/spec.md`

## Summary

Implement a marketplace-distributed service item that exposes the host's VFS (`data/vfs/`) via a WebDAV (RFC 4918) endpoint. The installed service handles all required HTTP methods (GET, PUT, DELETE, MKCOL, COPY, MOVE, PROPFIND, OPTIONS) by delegating to the existing VFS layer, with path traversal protection, symlink-cycle detection, streaming for large files, and RFC-compliant XML error responses. The feature is implemented as a Node-runtime middleware handler, not an App Router `route.ts` — Next.js route handlers would reject non-standard WebDAV verbs before user code runs. The marketplace item bundles both the WebDAV service and a Settings panel at `Settings → Integrations → Local Mount` for token lifecycle management and mount instructions. The service supports three deployment modes (Docker+Bastion, Docker-standalone, local-dev), each with a different authentication boundary but all using the same bearer-token flow internally.

## Technical Context

**Language/Version**: TypeScript — the WebDAV handler runs as Node-runtime middleware; the Settings panel is a React component loaded by the BOS settings UI at `Settings → Integrations → Local Mount`.

**Primary Dependencies**: No new npm dependencies required. The WebDAV handler uses Node's built-in `crypto` for token generation and hashing, and the existing VFS layer for all I/O. XML construction for WebDAV responses and RFC 4918 error bodies use a minimal inline template system — no XML library dependency. The Settings panel reuses existing BOS Settings UI patterns.

**Storage**: All state is managed through existing BOS mechanisms:
1. **Bearer tokens** are stored hashed (via `crypto.scrypt` with salt and constant-time compare) in `SecretsStore` (`src/lib/integrations/secrets/store.ts`) with AES-256-GCM encryption at rest.
2. **Config** (e.g., the WebDAV URL prefix) lives in the CryptFS-encrypted config system.
3. **Mount instructions** are embedded in the Settings panel UI.

**Testing**: Unit tests for the WebDAV handler's path normalization, auth middleware, and method dispatchers using `jest` with `supertest` or manual `fetch` against a test server. Integration tests verify each HTTP verb's behavior against the VFS layer. The Settings panel is tested as a standalone React component. Separate manual/external tests verify each deployment mode (see tasks.md).

**Target Platform**: Linux (Docker Compose multi-user deployment), macOS (native WebDAV via Finder), and local development. The WebDAV endpoint must be reachable from external machines in all three deployment modes.

**Project Type**: Web service (middleware + settings panel) — one Node middleware, one React component, all distributed as a marketplace item.

**Performance Goals**: Each WebDAV request must complete without perceptible added latency at the default VFS scale. Streaming must handle files >100 MB without buffering in memory (peak RSS increase < 50 MB per SC-006).

**Constraints**: No new external services or databases — all state lives in existing BOS mechanisms. Must not buffer entire files in memory. Path traversal and encoded variants (`%2e%2e`, `%2f`, `%5c`, `./`, `//`, `\`) must all be rejected with `403`. Symlink cycles must be detected and return `508 Loop Detected`. The feature must be opt-in (returning `404`/`503` when no token has ever been generated).

**Scale/Scope**: Bounded by whatever VFS depth/width is normal for a single user — this is a personal file server, not a multi-tenant platform. No concurrent access from multiple users to the same VFS root is a concern; each user has their own BOS instance.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

This feature is distributed as a marketplace item, not a BOS-source change — but the WebDAV handler still touches BOS source (`src/middleware.ts` or `src/lib/...`) and requires the existing VFS layer (which must expose `readStream()`/`writeStream()`). Per Constitution §IV (Minimize Blast Radius), the WebDAV handler will modify `src/middleware.ts` to add a catch-all route for `/api/vfs/webdav/*`. Per Constitution §V, all I/O goes through the VFS layer, not raw `fs` calls. The SecretsStore integration (FR-003) follows the existing pattern. No violations to record — the middleware change is narrowly scoped to one route.

## Project Structure

### Documentation (this feature)

```text
bos-system-specs/033-vfs-mount/
├── spec.md      # Feature specification
├── plan.md      # This file
└── tasks.md     # Task breakdown
```

### Source Code (repository root)

```text
src/middleware.ts           # MODIFIED — add catch-all route for /api/vfs/webdav/*
src/lib/integrations/
└── secrets/store.ts       # UNCHANGED — already supports bcrypt/scrypt hashing + AES-256-GCM
src/
├── webdav/
│   ├── handler.ts       # NEW — catch-all route for /api/vfs/webdav/*
│   ├── auth.ts          # NEW — Bearer token validation via SecretsStore
│   ├── normalize.ts     # NEW — path traversal + symlink cycle detection
│   ├── methods/
│   │   ├── get.ts     # NEW — GET handler (file read or directory XML)
│   │   ├── put.ts     # NEW — PUT handler (file write)
│   │   ├── head.ts    # NEW — HEAD handler (metadata only)
│   │   ├── delete.ts  # NEW — DELETE handler
│   │   ├── mkcol.ts   # NEW — MKCOL handler
│   │   ├── copy.ts    # NEW — COPY handler
│   │   ├── move.ts    # NEW — MOVE handler
│   │   ├── propfind.ts # NEW — PROPFIND handler (depth 0 and 1)
│   │   └── options.ts # NEW — OPTIONS handler (DAV: 1, 2 + Allow header)
│   ├── xml/
│   │   └── errors.ts  # NEW — RFC 4918 error response builder
│   └── index.ts       # new — route registration entrypoint
data/user-apps/items/
└── vfs-webdav-mount/
    └── index.html     # NEW — the installed marketplace app (all CSS/JS inline)
```

**Structure Decision**: Flat module structure under `src/webdav/` — one file per concern (auth, normalize, each HTTP method). The handler routes each method to the appropriate handler. The Settings panel is bundled as an HTML-only app (all CSS/JS inline) at `data/user-apps/items/vfs-webdav-mount/index.html`.

### Integration Notes

1. The WebDAV handler MUST be registered as a Node-runtime middleware route, NOT an App Router `route.ts` handler, because Next.js rejects non-standard WebDAV verbs (`PROPFIND`, `MKCOL`, `COPY`, `MOVE`) before user code runs. This is stated in the spec's Assumptions.
2. The VFS layer must expose stream-based I/O (`readStream()`, `writeStream()`) to satisfy the streaming requirement for large files. If the VFS layer doesn't currently support streaming, this is a prerequisite for this feature.
3. The SecretsStore already exists and supports hashed token storage. The WebDAV handler will use it directly for token verification.
4. The Settings panel component will be loaded by the existing BOS Settings UI when navigating to `Settings → Integrations → Local Mount`. The panel renders inside a container provided by the Settings framework.

## Data Model

### MountToken

| Field         | Type   | Notes                                                      |
|---------------|--------|-------------------------------------------------------------|
| `tokenId`     | string | Unique identifier                                           |
| `hash`        | string | Salted, slow (scrypt-class) hash of the raw bearer token        |
| `createdAt`   | string | ISO timestamp                                               |
| `label`       | string | Optional, user-supplied                                     |

Stored in `SecretsStore` per user, keyed by `(service = "vfs-webdav", tokenId)`.

## Deployment Modes

```text
┌─────────────────────────────────────────────────────────────┐
│  Docker Compose + Bastion                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client (davfs2 / Finder)                     │   │
│  └──────────────────────────┬──────────────────────┘   │
│                         │ HTTPS (TLS)               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Bastion (reverse proxy)                      │   │
│  └──────────────────────────┬──────────────────────┘   │
│                         │ forwarded               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  BOS Container /api/vfs/webdav/               │   │
│  └───────────────────────────────────────────────────┘   │
│  Auth: Bearer token (from SSO/OAuth flow)          │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Docker Standalone                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client (davfs2 / Finder)                     │   │
│  └──────────────────────────┬──────────────────────┘   │
│                         │ HTTP (or HTTPS)           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  BOS Container /api/vfs/webdav/               │   │
│  └───────────────────────────────────────────────────┘   │
│  Auth: Bearer token (directly)                     │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Local Development                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Client (davfs2 / Finder)                     │   │
│  └──────────────────────────┬──────────────────────┘   │
│                         │ HTTP (localhost)          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  BOS Container /api/vfs/webdav/               │   │
│  └───────────────────────────────────────────────────┘   │
│  Auth: Bearer token (directly)                     │
└─────────────────────────────────────────────────────────────┘
```

All three modes ultimately authenticate with a bearer token. The difference is *how* the token is obtained:
- **Docker+Bastion**: User authenticates via Bastion's SSO/OAuth flow, then uses the token.
- **Docker standalone**: User generates a token via the Settings panel (or API) and supplies it directly.
- **Local dev**: User generates a token via the Settings panel and uses it on `localhost`.

## Complexity Tracking

*No constitution violations — table intentionally empty.*
