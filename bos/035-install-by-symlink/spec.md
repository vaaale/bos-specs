# Feature Specification: Install Is a Symlink — One Item, One Link, No Copies

**Feature Branch**: `035-install-by-symlink`

**Created**: 2026-07-29

**Status**: New

**Input**: "When installing an app, plugin, service or anything from the marketplace, it should not be copied anywhere. This is core in the new architecture. Only symlinks shall be created in `data/system/`. We are removing `app/` and `services/` from `data/system/` — instead there is one symlink per item, e.g. `data/system/terminal -> data/marketplace/bos-marketplace/items/terminal`. The registries do a depth-2 scan of `data/system/` to find apps, services, plugins. Config is copied into `data/system/config/<item-id>/`."

> This spec owns **what installing means**: the on-disk representation of installed
> state, how facets are discovered, and the rule that install copies nothing.
> `034-user-apps-marketplace-parity` owns the shape of a marketplace repository;
> `user-specs/002-service-daemons` owns the item model and service lifecycle;
> `009-installed-apps` owns the app surface; `028-marketplace-sandbox` owns item
> provenance and sandboxing. This spec supersedes their descriptions of the
> installed-state layout.

## Why this exists

Installing used to **copy**: a service item was copied into `user-apps/<id>/`, an
app's files into `user-apps/<id>/app/`, a plugin into `dataDir()/bos-plugins/<id>/`.
Then symlinks were created per facet (`system/services/<id>`, `system/app/<id>`,
`system/hooks/<id>`, `config/<id>`, `specs/external-specs/<id>`,
`docs/external-docs/<id>`).

Four things were wrong with that:

1. **It polluted the user's own marketplace.** Installing someone else's app
   committed a copy of it into `user-apps` — a repository the user may publish.
2. **Copies go stale.** `git pull` on a marketplace updated the clone but not the
   installed copy.
3. **The app registry read the wrong source of truth.** It scanned
   `user-apps/` for installed apps rather than the installed-state directory, so
   "installed" and "present in my marketplace" were conflated. This is the root
   misconception behind a whole family of bugs.
4. **Half-items.** A plugin install copied only `app/app.json` into `user-apps`,
   leaving a stub that could not be reinstalled from.

## Clarifications

### Session 2026-07-29

- Q: Does install ever copy content? → A: **No.** Only a symlink is created. The single exception is *state*, not content: an item's `config/` defaults are seeded into `data/system/config/<item-id>/`, because config is mutable and must never be written inside a read-only marketplace clone (a service writes `runtime.json` there at start).
- Q: One symlink per facet, or per item? → A: **Per item.** `data/system/<item-id>` points at the item directory itself, wherever it lives. `app/` and `services/` under `data/system/` are removed.
- Q: How are facets found then? → A: A **depth-2 scan** of `data/system/`: depth 1 enumerates item symlinks, depth 2 identifies facets inside each.
- Q: Do apps, services and plugins each scan independently? → A: **No — one shared scanner.** Three independent scans of the same root is exactly how the previous divergence arose (the service registry scanned `user-apps` flat while the marketplace client scanned `items/`, and they silently disagreed).
- Q: Are plugins special? → A: **No.** A plugin is a facet of an item like any other. `dataDir()/bos-plugins/` is removed.
- Q: Is `app.json` still needed? → A: **Yes, but as the item author's manifest, inside the item.** BOS no longer writes it. Everything BOS used to inject is derivable from the symlink target: `origin` (resolves under `user-apps/items/` → local, else marketplace), `marketplaceId` (the path segment), `appUrl` (the item has a `plugin/` facet), `createdAt` (the symlink's own ctime).
- Q: What happens when a marketplace is removed while its items are installed? → A: **Refuse**, naming the installed items. Silently uninstalling is too much collateral for one click.
- Q: Is config cleaned up on uninstall? → A: **No.** Uninstall is exactly `rm data/system/<item-id>`; seeded config is left, so reinstalling preserves the user's settings.
- Q: Can an item be named `config`? → A: **No.** `data/system/config/` shares the flat namespace with item symlinks, so `config` is a reserved item id and MUST be rejected at install.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — Installing takes nothing from the marketplace but a reference (Priority: P1)

A user installs an item from a registered marketplace. Nothing is copied; one symlink appears.

**Independent Test**: Install a service item from a marketplace. Confirm `data/system/<id>` is a symlink into `data/marketplace/<mkt>/items/<id>`, that `user-apps` is completely untouched (no new files, no new commits), and that the service starts.

**Acceptance Scenarios**:

1. **Given** an item in a registered marketplace, **When** it is installed, **Then** the only filesystem changes are `data/system/<id>` (a symlink) and `data/system/config/<id>/` (seeded from the item's `config/`, if it has one).
2. **Given** an installed marketplace item, **When** the marketplace is synced (`git pull`), **Then** the installed item reflects the new content with no reinstall.
3. **Given** an installed item, **When** it is uninstalled, **Then** exactly one symlink is removed; the item's source and its seeded config remain.
4. **Given** an item whose id is `config`, **When** installation is attempted, **Then** it is rejected — that name is reserved by `data/system/config/`.

### User Story 2 — `user-apps` holds only the user's own work (Priority: P1)

**Why this priority**: `user-apps` is a repository the user may publish (`034`). Other people's apps must never be committed into it.

**Acceptance Scenarios**:

1. **Given** any install from a *registered* marketplace, **When** it completes, **Then** `user-apps` has no new content and no new commit.
2. **Given** an item authored by the user under `user-apps/items/<id>/`, **When** it is installed, **Then** `data/system/<id>` points there and it behaves identically to a marketplace item.

### User Story 3 — One scanner, one truth (Priority: P1)

Apps, services and plugins are all discovered by the same depth-2 scan.

**Acceptance Scenarios**:

1. **Given** an installed item with several facets, **When** the registries load, **Then** each facet is discovered by its own registry from the same scan result, and no registry can disagree about which items are installed.
2. **Given** a dangling `data/system/<id>` symlink (its target was removed outside BOS), **When** the scan runs, **Then** the entry is reported as broken rather than crashing the scan or silently vanishing.

### User Story 4 — Removing a marketplace is refused while in use (Priority: P2)

**Acceptance Scenarios**:

1. **Given** installed items sourced from marketplace `M`, **When** the user removes `M`, **Then** it is refused with a message naming those items.
2. **Given** none of `M`'s items are installed, **When** the user removes `M`, **Then** it is removed as before.

### Edge Cases

- An item offering no recognised facet → not installable; reject with a clear reason.
- A plugin-served app has only `app/app.json` and no `index.html` → still an app facet (already fixed in `034`).
- Two marketplaces offering the same item id → the flat `data/system/` namespace permits only one installed at a time; the second install MUST be refused, naming the incumbent and its source.
- Pre-035 installed state (per-facet symlinks, copies in `user-apps`, `data/bos-plugins/`) → migrated once (FR-010).
- Config seeding when `data/system/config/<id>/` already exists (reinstall) → left as-is; the user's settings win over the item's defaults.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Installing an item MUST NOT copy item content anywhere. The only filesystem effects are the item symlink (FR-002) and config seeding (FR-004).
- **FR-002**: Installed state MUST be exactly one symlink per item: `dataDir()/system/<item-id>` → the item directory, whether that is `dataDir()/marketplace/<mktId>/items/<item-id>` or `dataDir()/user-apps/items/<item-id>`. The per-facet directories `system/app/`, `system/services/`, `system/hooks/`, `system/settings/` and the symlinks `specs/external-specs/<id>` and `docs/external-docs/<id>` are REMOVED. (The latter two had no readers.)
- **FR-003**: Uninstalling MUST be exactly the removal of that symlink.
- **FR-004**: On install, an item's `config/` contents MUST be copied into `dataDir()/system/config/<item-id>/` if that directory does not already exist. This is the sole permitted copy, and it exists because config is **mutable state**: a service writes `runtime.json` into its config directory at start, which must never land inside a read-only marketplace clone. An existing seeded directory MUST be preserved on reinstall. `configDirPath` for a service is `dataDir()/system/config/<id>`.
- **FR-005**: `config` MUST be a reserved item id, rejected at install, because `dataDir()/system/config/` shares the flat namespace with item symlinks.
- **FR-006**: Facet discovery MUST be a **depth-2 scan** of `dataDir()/system/`: depth 1 enumerates entries (skipping `config`), depth 2 identifies facets — `app/` → app, `services/service.json` → service, `plugin/bos-plugin.json` → plugin, `spec/` → spec, `hooks/` → hooks.
- **FR-007**: There MUST be exactly **one** implementation of that scan, shared by the app registry, the service registry and the plugin loader. No subsystem may scan the install root independently. A dangling symlink MUST be reported as a broken entry, never crash the scan and never be silently dropped.
- **FR-008**: A plugin MUST be an ordinary item facet, discovered through the same scan. `dataDir()/bos-plugins/` is removed and the plugin loader MUST load from `dataDir()/system/<id>/plugin/`.
- **FR-009**: BOS MUST NOT write `app.json`. It remains the item author's manifest inside the item. Provenance MUST be **derived** from the symlink target: `origin` = `local` when it resolves under `user-apps/items/`, else `marketplace`; `marketplaceId` = the marketplace path segment; `appUrl` = present when the item has a `plugin/` facet; `createdAt` = the symlink's ctime.
- **FR-010**: A one-shot migration MUST convert pre-035 state: replace per-facet symlinks with one item symlink per installed id, seed `system/config/<id>/` from the item's current config (following the old `config/<id>` symlink before replacing it), remove the emptied facet directories and the two unused symlink trees, and remove `dataDir()/bos-plugins/` once its plugins are reachable as item facets. It MUST be idempotent and MUST NOT delete anything from `user-apps` (see FR-011).
- **FR-011**: The migration MUST NOT delete pre-existing copies in `user-apps` that came from a marketplace, because BOS never deletes from the user's repository. They become inert (installed state no longer points at them). Cleanup MUST be an explicit, separate user action.
- **FR-012**: Removing a registered marketplace MUST be refused while any installed item resolves into that marketplace's clone, with a message naming those items.
- **FR-013**: **Purge** (delete an item's content) MUST be offered only for items whose symlink resolves into `user-apps/items/` — the user's own work. For a marketplace item there is nothing to purge; removing the marketplace is the equivalent operation. Settings → Apps MUST reflect this.

### Key Entities

- **Item symlink** — `dataDir()/system/<item-id>`; the single, authoritative record that an item is installed, and the only thing uninstall removes.
- **Seeded config** — `dataDir()/system/config/<item-id>/`; BOS-owned mutable state, seeded from the item's defaults, surviving uninstall.
- **Installed item** — what the shared scanner returns: `{ id, itemPath (resolved), facets, origin, marketplaceId, broken }`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Installing anything from a registered marketplace produces zero new files and zero new commits in `user-apps`.
- **SC-002**: `git pull` on a marketplace updates every installed item from it, with no reinstall.
- **SC-003**: Uninstall is a single `rm` of one symlink.
- **SC-004**: Apps, services and plugins cannot disagree about which items are installed — they consume one scan.
- **SC-005**: No BOS-written per-item state exists except seeded config.

## Notes

- Supersedes the installed-state layout described in `user-specs/002-service-daemons` ("Installed state (symlinks)"), `009-installed-apps`, and `028-marketplace-sandbox`'s three-source model, all of which describe per-facet symlinks and/or install-time copies.
- Resolves the "half-item / plugin stub" gap recorded in `034-user-apps-marketplace-parity`.
- The `data/config/<id>` → `data/system/config/<id>` move means an item's config is no longer mixed in with BOS's own config namespaces (`marketplaces.json`, `plugins.json`, `provider.json`).
