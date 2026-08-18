# Feature Specification: user-apps is a Marketplace (Layout & Manifest Parity)

**Feature Branch**: `034-user-apps-marketplace-parity`

**Created**: 2026-07-29

**Status**: New

**Input**: "user-apps is the user's own private marketplace. It must have exactly the same layout as any external/central marketplace — `marketplace.json` plus `items/<id>/` — and BOS (the build system) maintains that manifest. The only differences are that the user owns it and that Build Studio places newly built apps there. It must be possible to point user-apps at the central marketplace repository and develop apps to share with the world."

> This spec owns the **structural equivalence** between `dataDir()/user-apps/` and any
> registered marketplace clone. `028-marketplace-sandbox` owns marketplace item
> semantics (facets, provenance, sandboxing); `user-specs/002-service-daemons` owns
> the item/installed-state model; `009-installed-apps` owns the app Item lifecycle.
> This spec owns only: the layout, who maintains `marketplace.json`, how it is
> maintained, and the migration.

## Clarifications

### Session 2026-07-29

- Q: Should BOS write into the user's own git repo? → A: **Yes, deliberately.** This reverses the earlier principle that BOS must never write a generated artifact into `user-apps`. The whole point is that the repository is a real marketplace, and a marketplace has a `marketplace.json`. The build system maintains it.
- Q: Regenerate the manifest from a directory scan, or merge? → A: **Merge, never regenerate.** A scan cannot reproduce curated metadata. Evidence from the central marketplace: `lunar-lander`'s entrypoint is `items/lunar-lander/app/dist` (a scan probing `<id>/app/index.html` finds nothing); `live-avatar` declares `runtime: "plugin-served"` and a `voiceEngine` facet with an `engineId` (underivable from disk); `welcome`, `pomodoro`, `unit-converter` and `color-palette` carry descriptions, `tags` and `icon` that exist *only* in the manifest. Regenerating would strip a curated marketplace to a skeleton and commit that.
- Q: What id/name does a fresh `user-apps` get? → A: `<username>-marketplace` for both. Only relevant when `user-apps` is uninitialised (absent, or not yet a git repo) — i.e. after a wipe or on first login.
- Q: Where does the username come from? → A: The bastion already injects `x-bos-username` on every proxied request (`024-docker-multiuser`), and BOS already reads it in `/api/system/session`. No new plumbing. It is therefore only available **in a request context**, which is why naming happens lazily on first catalog read rather than at boot.
- Q: What if the user tries to register a repository that is already their `user-apps`? → A: **Reject as a duplicate id.** This is the same rule for all three variants: the central repo used as `user-apps`, another maintainer's repo, or the user's own private marketplace hosted in git.
- Q: How does the Marketplace app present the local slot? → A: As one source among equals in the master-detail sidebar (`028-marketplace-sandbox` US6/UI-002), labelled from its own manifest ("My Apps" by default) and marked as the user's own.
- Q: Does `LOCAL_MARKETPLACE_ID` still exist? → A: Yes, but only as the **internal slot key** for "the marketplace at `dataDir()/user-apps`". It is no longer stamped into the repository's manifest as its identity.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — The same repository works as either marketplace (Priority: P1)

A maintainer points `user-apps` at the central marketplace repository, develops an app there, and publishes it — while other users consume that same repository as a registered external marketplace.

**Why this priority**: This is the motivating requirement. It only works if the two are structurally identical; any divergence makes the repository valid in one role and broken in the other.

**Independent Test**: Clone the central marketplace into `dataDir()/user-apps/`. Confirm the Marketplace app lists every item, with the curated names, descriptions, tags and icons intact. Add a new item via Build Studio, confirm it appears in `items/` and in `marketplace.json`, and confirm the repository is still valid when registered as an external marketplace elsewhere.

**Acceptance Scenarios**:

1. **Given** `user-apps` contains a marketplace with curated metadata, **When** BOS reads the catalog, **Then** every declared field is preserved exactly — nothing is regenerated or dropped.
2. **Given** an item whose entrypoint is a non-standard subpath (e.g. `items/x/app/dist`), **When** BOS reconciles the manifest, **Then** the declared entrypoint is left untouched.
3. **Given** an item declaring a facet BOS cannot infer from disk (`voiceEngine`, `serverPlugin`, `runtime: "plugin-served"`), **When** BOS reconciles, **Then** the facet survives.

### User Story 2 — Dropping a folder in still works (Priority: P1)

A user (or an agent) creates `items/<id>/` by hand. It becomes an installable item without anyone editing `marketplace.json`.

**Why this priority**: Auto-discovery is the one genuinely good property of the previous flat design and must not be lost in exchange for layout parity.

**Acceptance Scenarios**:

1. **Given** a new directory under `items/` with a recognisable facet, **When** the catalog is read, **Then** an entry is added to `marketplace.json` with whatever metadata is inferable, and the change is committed.
2. **Given** an item directory that has been deleted, **When** the catalog is read, **Then** its manifest entry is pruned.
3. **Given** reconciliation finds nothing to change, **When** the catalog is read, **Then** no write and no commit occur — reading a catalog MUST NOT dirty the user's repository.

### User Story 3 — Build Studio's output lands in the marketplace (Priority: P2)

Asked to build an app, BOS places it in `user-apps/items/<id>/` — not in BOS source, and not anywhere else.

**Acceptance Scenarios**:

1. **Given** a request to build a new app, **When** it is created, **Then** it appears at `user-apps/items/<id>/` and as a `marketplace.json` entry.

### User Story 4 — Registering a duplicate is refused (Priority: P2)

**Acceptance Scenarios**:

1. **Given** `user-apps` whose manifest id is `X`, **When** the user adds a remote marketplace whose manifest id is also `X`, **Then** it is rejected with a message naming the conflict.
2. **Given** a remote whose manifest id matches an already-registered marketplace, **When** the user adds it, **Then** it is rejected (existing behaviour, retained).

### Edge Cases

- `user-apps` exists but is not a git repo → boot's `ensureRepo` makes it one; the manifest is created lazily on first catalog read.
- `user-apps` has no `marketplace.json` but does have items → the manifest is created and populated from the scan.
- `marketplace.json` is malformed → surfaced as an error; BOS MUST NOT silently overwrite a file it cannot parse, because that file is the user's curated content.
- BOS runs standalone (no bastion, so no username) → fall back to a neutral marketplace name.
- A remote's manifest declares the literal id `user-apps` → rejected; that string remains reserved as the internal slot key.
- Items existing under the **old flat layout** → migrated once (see FR-008), including re-pointing installed-state symlinks.
- An item whose app is **plugin-served** legitimately has only `app/app.json` in its item directory (the app is served from `/api/plugin/<id>/app`, not from files). It MUST still be discovered — keying discovery on `app/index.html` alone made such an item invisible in its own marketplace while installed and running.
- ~~**Known gap**: installing a plugin item from a remote marketplace leaves a stub in `user-apps/items/<id>/`.~~ **Resolved by `035-install-by-symlink`**: installing copies nothing at all, so no stub can be created; the pre-existing stubs are repaired by that spec's migration.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `dataDir()/user-apps/` MUST have the identical on-disk shape of any marketplace clone: a `marketplace.json` at its root and items under `items/<item-id>/`. No BOS code path may assume a different layout for it than for `dataDir()/marketplace/<id>/`.
- **FR-002**: `marketplace.json` in `user-apps` MUST be a real, on-disk, **authoritative** file, read through the same path as any other marketplace's manifest. The previous behaviour — synthesizing it in memory on every catalog read and never reading the file — is removed.
- **FR-003**: BOS MUST maintain that manifest, and MUST do so by **reconciliation, not regeneration**:
  - add entries for item directories present under `items/` but absent from the manifest, populating only what is inferable from disk (`app/app.json`, `services/service.json`, facet detection);
  - remove entries whose item directory no longer exists;
  - **never** modify a field it did not author on an entry that already exists.
- **FR-004**: Reconciliation MUST write and commit **only when it actually changed something**. A catalog read that finds the manifest already consistent MUST leave the working tree clean — reading MUST NOT produce git churn in a repository the user may be tracking against a remote.
- **FR-005**: Mutating operations (install, create, uninstall, delete) MUST update the manifest entry and commit it, following the existing commit convention for this repository.
- **FR-006**: When `user-apps` is uninitialised, BOS MUST create the manifest with `id` and `name` set to `<username>-marketplace`, where the username comes from the bastion-injected `x-bos-username` request header. Because that is only available per-request, this MUST happen lazily on the first catalog read, NOT at boot. Boot MUST continue to only ensure the directory is a git repo. Absent a bastion (standalone BOS), a neutral fallback name MUST be used.
- **FR-007**: `addMarketplace` MUST reject a remote whose manifest id equals the id of the local `user-apps` marketplace, with a message naming the conflict. `LOCAL_MARKETPLACE_ID` MUST remain the internal key identifying the `user-apps` **slot** (by location) and MUST no longer be written into the repository's manifest as its identity; a remote declaring that literal id MUST still be rejected.
- **FR-008**: A **one-shot migration** MUST convert an existing flat `user-apps` to the new layout: move `user-apps/<id>/` → `user-apps/items/<id>/`, create or reconcile `marketplace.json`, and **re-point every installed-state symlink** under `dataDir()/system/{app,services,settings,hooks}/` and `dataDir()/config/<id>` — those are absolute paths into the old locations and would otherwise all dangle. The migration MUST be idempotent and MUST commit its result.

### Key Entities

- **Marketplace repository** — a git repo with `marketplace.json` at its root and items under `items/`. `user-apps` is one; a registered clone is one. There is no third shape.
- **Local marketplace slot** — the marketplace at `dataDir()/user-apps`, identified internally by `LOCAL_MARKETPLACE_ID` (a location, not an identity).
- **Reconciliation** — the add-missing / prune-vanished / preserve-authored merge that keeps the manifest in step with `items/`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The central marketplace repository can be used as `user-apps` with zero edits, and every curated field survives a catalog read and a commit.
- **SC-002**: A repository that works as `user-apps` also validates as a registered external marketplace, and vice versa.
- **SC-003**: Creating an item by hand under `items/` makes it installable with no manual manifest editing.
- **SC-004**: Reading the catalog never leaves the user's repository dirty when nothing changed.
- **SC-005**: After migration, no installed item's symlink dangles.

## Notes

- Supersedes the "BOS never writes into `user-apps`" principle previously stated in `user-specs/002-service-daemons` and in `docs/dev/apps/services.md`. That principle protected the user's git history from generated artifacts; it is replaced by the narrower guarantee in FR-003/FR-004 — BOS writes only the manifest, only by merge, and only when something actually changed.
- Related: `028-marketplace-sandbox` (item semantics, provenance), `user-specs/002-service-daemons` (item/installed-state model), `009-installed-apps` (app Item lifecycle), `024-docker-multiuser` (source of `x-bos-username`).
