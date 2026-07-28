# Feature Specification: VFS Mount (WebDAV)

**Feature Branch**: `033-vfs-mount`
**Created**: 2026-07-28
**Status**: Draft

**Input**: "It would be very cool if the user could mount BOS on their Linux/macOS computer — sort of like Google Drive or OneDrive. Maybe using FuseFS?"

> BOS exposes its VFS (`data/vfs/`) via a WebDAV server endpoint rather than a custom FUSE driver. macOS ships a native WebDAV client (`Finder → Go → Connect to Server`); Linux uses `davfs2`. This gives local-filesystem semantics with no kernel extension required on either platform. FUSE is handled by the OS's own WebDAV layer, not by BOS. Companion: `006-data-isolation` (VFS ownership), `007-gitfs` (VFS contents).

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Mount BOS as a local drive (Priority: P1)

The user wants to open, edit, and save BOS VFS files from their native file manager or any local app (VS Code, Terminal, Finder) without using the BOS browser UI.

**Independent Test**: Mount the WebDAV endpoint with `davfs2` or macOS Finder; open a file in a local editor, save it, then verify the change is visible inside BOS's Files app.

**Acceptance Scenarios**:

1. **Given** a running BOS instance, **When** the user mounts `http://localhost:3000/api/vfs/webdav` with a WebDAV client, **Then** the VFS root appears as a local directory tree.
2. **Given** the mounted drive, **When** the user creates, edits, or deletes a file, **Then** the change is immediately reflected in BOS's VFS (Files app, assistant file tools).
3. **Given** the mounted drive, **When** BOS itself creates or modifies a file, **Then** a WebDAV `PROPFIND` or file read by the client returns the updated content.

### User Story 2 — Authentication protects the mount (Priority: P1)

Only the authenticated BOS user can mount and access the drive; unauthenticated requests are rejected.

**Independent Test**: Attempt a `PROPFIND` request with no credentials; confirm 401. Then retry with a valid token; confirm 207.

**Acceptance Scenarios**:

1. **Given** an unauthenticated WebDAV request, **When** it arrives at the endpoint, **Then** BOS responds `401 Unauthorized` with a `WWW-Authenticate: Bearer` challenge.
2. **Given** a valid mount token (generated in Settings), **When** it is supplied as an `Authorization: Bearer <token>` header, **Then** the full VFS tree is accessible.
3. **Given** a token that has been revoked in Settings, **When** it is used to mount, **Then** BOS responds `401 Unauthorized`.

### User Story 3 — Mount token management in Settings (Priority: P1)

The user can generate and revoke mount tokens without restarting BOS.

**Independent Test**: Generate a token in Settings → Integrations → Local Mount; use it to mount; revoke it; confirm the mount stops responding.

**Acceptance Scenarios**:

1. **Given** the Settings → Integrations → Local Mount panel, **When** the user clicks "Generate token", **Then** a new bearer token is created, stored hashed in config, and displayed once for copying.
2. **Given** an existing token, **When** the user revokes it, **Then** subsequent requests using that token receive `401`.
3. **Given** multiple tokens, **When** one is revoked, **Then** other tokens remain valid.

### User Story 4 — Mount instructions are surfaced in the UI (Priority: P2)

The user does not need to remember the WebDAV URL or mount command; BOS shows ready-to-run commands for Linux and macOS.

**Acceptance Scenarios**:

1. **Given** the Local Mount settings panel, **When** a token exists, **Then** BOS displays copy-ready `davfs2` and macOS `mount_webdav` commands with the token and detected port pre-filled.
2. **Given** BOS running behind a custom `BASE_URL`, **When** commands are displayed, **Then** they use the configured public URL, not `localhost`.

### Edge Cases

- Requests targeting paths outside the VFS root (e.g. `../`) MUST be rejected with `403 Forbidden`.
- Symlinks within the VFS MUST be followed transparently; cycles MUST be detected and return `508 Loop Detected`.
- Files being written by BOS concurrently with a WebDAV PUT MUST not corrupt data; writes MUST use the same atomic-write path as the VFS layer.
- Very large files (> 100 MB) MUST stream rather than buffer in memory.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: BOS MUST expose a WebDAV endpoint at `GET|PUT|DELETE|MKCOL|COPY|MOVE|PROPFIND|OPTIONS /api/vfs/webdav/[...path]` that maps 1-to-1 onto the VFS root (`data/vfs/`).
- **FR-002**: All WebDAV requests MUST be authenticated via a `Authorization: Bearer <token>` header; unauthenticated requests MUST receive `401` with `WWW-Authenticate: Bearer realm="BOS"`.
- **FR-003**: Mount tokens MUST be generated as cryptographically random 256-bit values, stored as a bcrypt hash in the `system` config namespace, and MUST be displayable only at creation time.
- **FR-004**: Multiple tokens MAY coexist; each MUST be independently revocable from the Settings UI without affecting others.
- **FR-005**: Path traversal attempts (`../`, encoded variants) MUST be rejected with `403 Forbidden`; all resolved paths MUST be verified to remain under the VFS root before any I/O.
- **FR-006**: WebDAV `PROPFIND` with `Depth: 1` MUST return `207 Multi-Status` XML enumerating children with at minimum: `displayname`, `getcontentlength`, `getlastmodified`, `getcontenttype`, `resourcetype`.
- **FR-007**: `PUT` and `MKCOL` MUST delegate to the existing VFS write path to preserve atomicity and any existing hooks.
- **FR-008**: `COPY` and `MOVE` MUST be implemented as server-side operations (no round-trip through the client) and MUST update VFS state atomically.
- **FR-009**: File reads and writes MUST stream; BOS MUST NOT buffer entire files in memory for the WebDAV layer.
- **FR-010**: The WebDAV handler MUST set `DAV: 1, 2` in its `OPTIONS` response and MUST handle `Expect: 100-continue` correctly so that standard WebDAV clients (macOS `mount_webdav`, `davfs2`, Cyberduck) work without configuration changes.
- **FR-011**: A Settings panel ("Local Mount") MUST be added under Settings → Integrations providing: token generation, token list with revoke buttons, and copy-ready mount commands for macOS and Linux.
- **FR-012**: The feature MUST be opt-in; the WebDAV route MUST return `404` (or `503`) when no token has ever been generated, to avoid exposing the endpoint on instances where it is not needed.
- **FR-013**: BOS MUST log each authenticated WebDAV request (method, path, response code, duration) to the central log at `debug` level; authentication failures MUST be logged at `warn` level.

### Key Entities

- **Mount token** — a single-use-display, multi-use-auth bearer token stored hashed in `system` config; revocable independently.
- **WebDAV handler** — Next.js API route implementing RFC 4918 (WebDAV) verbs over the BOS VFS.
- **Local Mount settings panel** — UI surface under Settings → Integrations for token lifecycle and mount instructions.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A fresh `davfs2` mount on Linux and `mount_webdav` on macOS both succeed using a generated token and present the VFS tree as a local directory.
- **SC-002**: A file created via `cp` into the mounted directory appears in BOS's Files app within one `PROPFIND` poll cycle (≤ 2 s for typical `davfs2` cache TTL).
- **SC-003**: A `PROPFIND` request with no `Authorization` header returns `401`; a request with a valid token returns `207`.
- **SC-004**: A revoked token returns `401` on the next request without requiring a BOS restart.
- **SC-005**: A path-traversal attempt (`/../etc/passwd`) returns `403` and nothing outside `data/vfs/` is read.
- **SC-006**: Uploading a 200 MB file via `PUT` does not cause Node.js heap exhaustion (peak RSS increase < 50 MB).
- **SC-007**: `npx tsc --noEmit` and `npm run lint` pass for all changed files.

## Assumptions & Dependencies

- Depends on `006-data-isolation`: all I/O MUST go through the VFS layer, not raw `fs` calls.
- Depends on `007-gitfs`: GitFS-backed VFS paths work transparently (reads reflect HEAD, writes commit).
- The Next.js App Router `route.ts` handler MUST use `export const runtime = "nodejs"` to access `fs`; the edge runtime cannot serve this endpoint.
- macOS `mount_webdav` and `davfs2` both support HTTP (non-TLS) on loopback; TLS is not required for local mounts but MUST work if `BASE_URL` uses `https://`.

## Notes

- RFC 4918 is the WebDAV specification. The minimum compliant subset for macOS Finder + `davfs2` compatibility is: `OPTIONS`, `PROPFIND` (depth 0 and 1), `GET`, `PUT`, `DELETE`, `MKCOL`, `COPY`, `MOVE`. `LOCK`/`UNLOCK` (RFC 4918 §7) MAY be implemented as no-ops returning `200` to satisfy clients that require lock negotiation.
- A future iteration could expose a read-only subtree of BOS source (`src/`) for inspection, but that is out of scope for this spec.
- `davfs2` documentation recommends setting `use_locks 0` in `~/.davfs2/davfs2.conf` when connecting to servers that implement advisory-only locking; the mount instructions displayed in Settings SHOULD include this note.
