# Implementation Plan: Multi-User Usability

**Branch**: `026-multiuser-usability` | **Date**: 2026-07-14 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/bos-system-specs/026-multiuser-usability/spec.md`

## Summary

Make a `docker compose up` deployment of BOS usable by non-experts. The bulk of the work lives in the existing `bastion/` sub-project (from `024-docker-multiuser`): a guided first-run (set-admin-password for simple auth; setup/landing for Keycloak), persistent per-user provisioning logging surfaced in the UI, a redesigned admin portal (users + data-wipe, streamed image build, container start/stop/kill), and a redesigned self-service account page (password, profile image, lifecycle, wipe-data-with-confirm, open BOS). BOS itself gets two focused changes: `run_command` uses the `local` backend when running behind the Bastion (the admin-built user-container image IS the sandbox environment) while standalone keeps select-image/build-from-Dockerfile with a streamed log; and the desktop toolbar gains a multi-user-only "My profile" link to `/app/account`. The root `Dockerfile` for the user-container image is extended to bundle the run_command runtimes and the Claude/OpenCode CLIs.

**2026-07-27**: the dev-harness credential/auth work originally planned here (Phase E) has been extracted to `029-settings-dev-harness` — this plan no longer covers it; see that spec's own plan.md.

## Technical Context

**Language/Version**: TypeScript (Node ≥ 20) for the bastion; React + Vite for the bastion SPA (adding Tailwind). BOS is Next.js App Router + React; server-only code uses Node built-ins + the existing config store.

**Primary Dependencies (bastion)**:
- Existing: `express`, `http-proxy-middleware`, `dockerode`, `jsonwebtoken`, `cookie-parser`, `bcryptjs`, `js-yaml`, `chokidar`, `openid-client`, `react`, `vite`.
- New: `tailwindcss` (+ `postcss`, `autoprefixer`) for the SPA. `dockerode` already supports `buildImage` (streamed) — no new build dep. Avatar upload is handled by a **hand-rolled multipart parser** (single bounded file field, ~30 lines) to avoid adding a `multer` dependency; `busboy` is available in Node ≥ 20 via the `stream` module so no extra install is needed.

**Primary Dependencies (BOS changes)**: none new. Reuses `readNamespace`/`writeNamespace`/`patchNamespace` config store, existing `docker` CLI usage from `run-command.ts`, and Next.js route handlers for streaming (`ReadableStream`).

**Storage**:
- Bastion: existing `{dataDir}/config.json`, `users.yml`; NEW `{dataDir}/logs/<username>.log` (append-only per-user log) and `{dataDir}/avatars/<username>` (profile images). A bundled default avatar ships in the image.
- BOS: no new storage owned by this plan (dev-harness credential/config storage is `029-settings-dev-harness`'s).

**Testing**: Bastion `tsc --noEmit`. BOS `npm run typecheck` + `npm run lint`. Manual/e2e verification per Success Criteria (first-run, forced provisioning failure visibility, user create/remove+wipe, streamed image build, container kill, account lifecycle, run_command local execution, harness auth, toolbar link).

**Target Platform**: Docker Compose on a Linux host with Docker Engine ≥ 24 (rootful), consistent with `024`.

**Project Type**: Enhancement of the existing `bastion/` sub-project + minimal BOS source changes.

**Performance Goals**: Streamed build/log latency perceptibly "live" (< 1 s between docker output and browser). No regression to `024`'s login→session timings.

**Constraints**: Username charset `[a-z0-9_-]`. Docker socket is mounted into the Bastion only — NOT into user containers (rejected in clarifications in favor of the `local` backend). Last-admin protection retained.

**Scale/Scope**: Same ~50-concurrent-user envelope as `024`.

## Constitution Check

*GATE: must pass before design; re-check after.*

- **I. Spec-Driven — SAAP**: plan derives from `spec.md`; implementation follows `tasks.md`. PASS.
- **II. Server Authority & SSR Boundary**: bastion is its own server; BOS server boundary unchanged. New BOS image-list/build routes are server-only. PASS.
- **III. Always Delegate; Claude Codes**: implementation is source work on the feature branch via the Developer harness. PASS.
- **IV. Minimize Blast Radius (NON-NEGOTIABLE)**: most changes are in `bastion/`; BOS source changes are three narrow areas (run_command backend gate, dev-harness creds, toolbar link) + the root Dockerfile. No BOS runtime behaviour changes for existing single-user installs (Bastion-mode gate keys off `BOS_PUBLIC_PORT`). PASS.
- **V. The VFS Is Not the Source**: per-user `data/` remains the user's VFS; wipe-data touches only `data/`. PASS.
- **VI. Specs & Docs Stay in Sync (NON-NEGOTIABLE)**: closeout merges 026 into `024`/`019`, updates `discrepancies.md` + `overview.md`, and updates `docs/dev/**` + `docs/usage/**`. PASS.
- **VII. Respect Boundaries**: BOS `package.json`/lockfile/build config untouched; new deps land only in `bastion/`. Docker socket into user containers is explicitly avoided. PASS.

No violations → Complexity Tracking is empty.

## Project Structure

### Documentation (this feature)

```text
specs/bos-system-specs/026-multiuser-usability/
├── spec.md        # done
├── plan.md        # this file
└── tasks.md       # done
```

### Source Code

```text
bastion/
├── src/
│   ├── log-store.ts                  # NEW — append-only per-user log ({dataDir}/logs/<user>.log): append/read(tail)/list
│   ├── config.ts                     # EDIT — allow ADMIN_USER; helpers for avatars dir
│   ├── docker.ts                     # EDIT — buildImage (streamed), killContainer (force remove), listBosContainers status
│   ├── lifecycle.ts                  # EDIT — record steps/errors to log-store; expose `error` reason + kill
│   ├── provision.ts                  # EDIT — stream steps to log-store; ensure stop-before-wipe on data wipe
│   ├── auth/
│   │   ├── index.ts                  # EDIT — hasAnyAdmin()/adminExists() for bootstrap gate
│   │   └── simple.ts                 # EDIT — adminExists(); create admin from ADMIN_USER
│   └── routers/
│       ├── setup.ts                  # NEW — GET /setup (state), POST /setup (create admin, race-safe); Keycloak landing
│       ├── auth.ts                   # EDIT — route to setup when bootstrap pending
│       ├── admin.ts                  # EDIT — delete-with-wipe default; image build (stream) + list; kill; per-user log
│       └── account.ts               # EDIT — avatar upload/serve; start/stop/restart; wipe-data; log
├── ui/
│   ├── tailwind.config.js            # NEW
│   ├── postcss.config.js             # NEW
│   ├── src/
│   │   ├── index.css                 # NEW — Tailwind entry
│   │   ├── components/               # NEW — Button, Card, Table, Dialog, Toast, LogView, StatusBadge, Avatar
│   │   ├── App.tsx                   # EDIT — add /setup route
│   │   └── pages/
│   │       ├── Setup.tsx             # NEW — set admin password / Keycloak landing
│   │       ├── Login.tsx             # EDIT — redesign
│   │       ├── Admin.tsx             # EDIT — Users / Images / Containers / Logs views, redesigned
│   │       └── Account.tsx           # EDIT — password, avatar, lifecycle, wipe, open BOS, redesigned
│   └── assets/default-avatar.svg     # NEW — bundled system default avatar
├── package.json                      # EDIT — tailwind/postcss/autoprefixer (+ multer if used)
└── Dockerfile                        # EDIT if needed for new build assets

src/                                  # BOS — minimal changes
├── lib/system/run-command.ts         # EDIT — Bastion mode (BOS_PUBLIC_PORT) → force local backend
├── lib/config/registry.ts            # EDIT — run-command: hide image fields in Bastion mode
├── components/apps/settings/
│   └── CommandExecutionTab.tsx       # NEW — custom component: image picker + streamed build (standalone); local view (Bastion)
├── components/desktop/Topbar.tsx     # EDIT — My profile link (multi-user only) w/ avatar
└── app/api/
    ├── run-command/images/route.ts   # NEW — GET list local images (server-only)
    └── run-command/image/build/route.ts # NEW — POST streamed build (server-only, guarded)

# Dev-harness credential/provider/MCP-inclusion source changes (harness-config.ts, claude-runner.ts,
# DevHarnessTab.tsx) moved to 029-settings-dev-harness's Project Structure — not this plan's.

Dockerfile                            # EDIT (root) — bundle run_command runtimes + Claude/OpenCode CLIs into the user-container image
docs/dev/**, docs/usage/**            # EDIT (closeout)
```

## Design Notes

### First-run bootstrap (`bastion/src/routers/setup.ts`, `auth/*`)

`SimpleProvider.adminExists()` returns whether any user has `admin: true`. A tiny gate (in `auth.ts` / `index.ts` static serving) redirects unauthenticated entry to `/app/setup` while bootstrap is pending. `POST /setup` creates the admin (`ADMIN_USER`, default `admin`) with the submitted password, issues a session, and returns success → SPA redirects to `/app/admin`. Race-safety: creation is guarded by a single in-process lock + a re-check that no admin exists (first writer wins; the loser gets a 409). For Keycloak, `/setup` is purely informational (IdP owns users) with a link to `/auth/keycloak`.

### Per-user logging (`bastion/src/log-store.ts`)

A minimal append-only store: `append(username, line)`, `read(username, {tail})`, `list()`. `lifecycle.ts`'s existing `log()` and error paths, and `provision.ts` steps, call `append`. State gains a short `error` string. Admin (`GET /admin/instances/:user/log`) and account (`GET /account/log`) expose the tail; the SPA renders it in a `LogView`. This is the single fix for "provisioning fails silently".

### Admin portal (`bastion/ui` + `routers/admin.ts`)

Tailwind + a small shared component set replace inline styles. Views:
- **Users** — create/remove/toggle-admin. Remove → confirm dialog → `DELETE /admin/users/:user` with `wipeData` (default true here) → stop/remove container → wipe `user-data/<user>` (`deprovisionUser` with all wipes) → **also** delete `{dataDir}/logs/<username>.log` and `{dataDir}/avatars/<username>` (PII cleanup). In **Keycloak mode** (`AUTH_PROVIDER=keycloak`) the **Users tab is hidden entirely** — users and roles are owned by the IdP and the Bastion has no authority to create or delete them. The admin portal still shows Images, Containers, and Logs views, but user management is replaced by a message directing the admin to the Keycloak console.
- **Images** — `POST /admin/image/build` streams `dockerode.buildImage(tarContext, { t, dockerfile })` output line-by-line to the client (chunked/SSE); `GET /admin/images` lists local images; "Set active" persists `bosImage` via `saveConfig`. Single concurrent build guard.
- **Containers** — `GET /admin/instances` (existing) + live status; Start (`getOrProvision`), Stop (`stopInstance`), NEW Kill (`killContainer` = force remove + clear state).
- **Logs** — per-user log viewer.

### Account page (`bastion/ui` + `routers/account.ts`)

Redesigned with the same components. Adds: password (simple only), **avatar** upload (`POST /account/avatar`, validate mime+size, write `{dataDir}/avatars/<user>`; `GET /avatar/:user` serves it or the bundled default), lifecycle Start/Stop/Restart, Re-provision, **Wipe my data** (confirm dialog with explicit destruction warning → stop → wipe `data/` → offer restart), and **Open my BrowserOS** (navigates to `/`).

### BOS run_command backend gate (`run-command.ts`, `registry.ts`)

`loadRcConfig()` checks `process.env.BOS_PUBLIC_PORT` (the canonical multi-user signal, per `src/app/api/system/session/route.ts`). In Bastion mode it forces `backend: "local"` (runs inside the user container, whose image bundles the runtimes) and the settings custom component hides image build/selection. In standalone mode a new `CommandExecutionTab` lists local images (`GET /api/run-command/images`) and offers build-from-Dockerfile (default `docker/run-command/Dockerfile`) via `POST /api/run-command/image/build` (streamed `ReadableStream`, single-build guard, docker-availability check).

### User-container image (`Dockerfile` root)

Extend the image the admin builds so it doubles as the run_command `local` environment and the harness host: add the `docker/run-command/Dockerfile` toolchain (python venv, node + NODE_PATH, LibreOffice, poppler, python-pptx/pptxgenjs) and install the `claude` and `opencode` CLIs on PATH.

**Image size**: LibreOffice alone adds ~400 MB; the full image will be substantially larger than the existing BOS image. Mitigate with: (a) a multi-stage `Dockerfile` so build-time tooling (compilers, pip cache) is not in the final layer, (b) `apt-get clean && rm -rf /var/lib/apt/lists/*` in every `apt-get install` `RUN` block, (c) a single combined `RUN` for apt layer squashing where possible. Task X1 MUST record the resulting compressed image size as a reference number and note whether it fits within the operator's expected envelope; if it exceeds 3 GB compressed, document a slim-variant approach (e.g. separate LibreOffice layer or optional install).

### BOS toolbar "My profile" link (`Topbar.tsx`)

The toolbar component must know the current username to construct the avatar URL (`/avatar/<me>`). It MUST call `/api/system/session` (the existing session endpoint, already used by `024`) at mount time to obtain the username rather than hard-coding or guessing it. The session response also confirms multi-user context at the client level (presence of `username` field). The `GET /avatar/:username` route in the Bastion MUST validate the `username` path parameter against the `[a-z0-9_-]` charset constraint before constructing the file path, as defense-in-depth against path traversal even though the charset bound is already enforced at user creation.

## Out of scope (v1)

- Mounting the Docker socket into user containers (explicitly rejected; `local` backend used instead).
- Per-command isolated sandbox containers in Bastion mode (there is only the user container).
- Rootless Docker / socket-proxy hardening (inherited from `024`).
- Generic (non-Keycloak) OIDC.
- Encrypted-at-rest secret store beyond the existing config store's protection.
- Multi-host / Swarm / Kubernetes; automated volume backups.
- Rolling image upgrade orchestration (admin can rebuild + restart).
