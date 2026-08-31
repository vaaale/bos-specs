# Feature Specification: File-Type Handler Registry & "Open With"

**Feature Branch**: `036-file-type-handlers`

**Created**: 2026-08-31

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "When I click an html file in the Files app, open it in web_view. Also add a right-click context menu where I can choose to open a file in an app that has registered itself as capable of rendering or editing that file type (mimetype). The Agentic Editor should register itself as such a handler (alongside the default web_view), and the Editor should properly render HTML in Preview mode."

## Context

BOS has no way for an application to say "I can open files of type X," and no OS-level "open with" mechanism. Today the built-in **Files** app (`src/apps/files/`) hardcodes its open behavior: images render in-app (an `IMAGE_RE` extension test), and **every other file — including `.html` — is dumped into a plain-text editor** (raw source in a `<textarea>`). There is no way for an installed app to participate in "how a file opens."

Meanwhile the pieces needed for a real file-type system already exist:

- **`web_view`** is the built-in **`html-viewer`** app (`src/apps/html-viewer/`, manifest `name: "HTML Preview"`, `hidden: true`), a sandboxed preview window launched via the OS store with params (`launch("html-viewer", { url, title })`). The `web_view` tool handler already does exactly this for VFS files (rewrites a VFS path to the raw-file URL, then `launch("html-viewer", …)`).
- **`AppManifest`** (`src/os/types.ts`) already carries a declarative, boot-time-registered extension point: `eventHandlers` (spec 034) — "statically-declared UI event handlers… surfaced into the handler registry at boot." A **`fileHandlers`** field is the same pattern.
- **Launch-with-params** is a first-class OS primitive: `launch(appId, params)` produces a `WindowInstance` whose `params` are handed to the app via `AppProps`. So "open file in app X" is just `launch(x, { path })`.
- **MIME resolution** already exists: `mimeForPath` (`src/lib/mime.ts`) and the raw-file route's MIME map (`src/app/api/fs/raw/route.ts`).

This feature introduces the missing platform layer — a **file-type-handler registry** that both built-in and installed (marketplace) apps declare into, a **"which apps handle this type"** query, a **per-type selected (default) handler**, and a **"launch app with file"** path — and wires the **Files app** up as its first consumer. `web_view`/`html-viewer` is registered as the default renderer for `text/html`, which makes "double-click an HTML file → opens in web_view" fall out of the general mechanism rather than being a special case.

**Related / follow-on work (NOT part of this spec's implementation):** the Agentic Editor is the installed marketplace item **`agentic-text-editor`**. Registering it as a `text/html` handler and fixing its HTML rendering in Preview mode is an incremental change to that *item* — a separate `marketplace-item` spec, built via `agent_delegate` + `app_build`, and it depends on this platform mechanism existing first. It is listed here only so the dependency is explicit; this spec is complete and independently valuable without it.

## User Scenarios & Testing

### User Story 1 - An app declares the file types it can handle (Priority: P1)

An application — built-in or installed — declares in its manifest, per MIME type, that it can **render** and/or **edit** files of that type, and that it is the **default** handler for the type. At boot the OS collects these declarations from every installed app into a single **file-type-handler registry**, so any part of the system can later ask "which installed apps can handle this type?" and "which app is selected for this type?"

**Why this priority**: This is the platform primitive everything else builds on. Without it, there is no mechanism for `web_view` or the Editor (or any future app) to advertise file-type capability, and no way for the Files app to present choices.

**Independent Test**: Fully testable by inspecting the registry after adding a manifest declaration to a test app: the declaration is present, queryable by MIME type, and reports the app's render/edit capabilities and default flag. This alone delivers the reusable infrastructure (no UI yet).

**Acceptance Scenarios**:

1. **Given** an app's manifest declares it can render `text/html` and is the default for that type, **When** the OS boots and builds the handler registry, **Then** querying the registry for `text/html` returns that app as a render-capable, default handler.
2. **Given** two apps declare they can edit `text/markdown`, **When** the registry is queried for `text/markdown`, **Then** both apps are returned as edit-capable handlers.
3. **Given** an app declares a handler for a MIME type but is not installed (uninstalled), **When** the registry is queried for that type, **Then** the app is **not** returned (the registry reflects only currently-installed apps).
4. **Given** a MIME type with no registered handlers, **When** the registry is queried for it, **Then** an empty result is returned (no crash, no phantom default).

---

### User Story 2 - HTML files open in web_view by default (Priority: P1)

`web_view` (the `html-viewer` app) is registered as a render handler for `text/html` and is selected as the default handler for that type. This is done via its manifest declaration, so the registration is data (not a code special-case) and survives like any other app.

**Why this priority**: This is the user's literal headline request ("click an html file → open in web_view"). It also proves the registry works end-to-end with a real built-in app.

**Independent Test**: Fully testable by confirming `html-viewer`'s manifest declares a `text/html` render handler and that the registry reports it as the selected handler for `text/html`. Combined with Story 3, double-clicking any `.html` file opens it in the web_view preview.

**Acceptance Scenarios**:

1. **Given** `html-viewer` declares a `text/html` render handler marked as the default, **When** the OS builds the registry, **Then** `html-viewer` is the selected handler for `text/html`.
2. **Given** the selected handler for `text/html` is `html-viewer`, **When** a file with MIME `text/html` is asked to open, **Then** the OS launches `html-viewer` with the file (its rendered HTML is shown in a preview window, not raw source).

---

### User Story 3 - Double-click opens a file with the selected handler (Priority: P1)

In the Files app, the open gesture (double-click on a file) resolves the file's MIME type, looks up the **selected** handler for that type, and opens the file with it. If no handler is selected/registered for the type, the Files app's existing in-app behavior applies (image viewer for images, text editor for text) — so this is backward-compatible and strictly additive.

**Why this priority**: This is the user-facing behavior change that makes "click an HTML file opens in web_view" actually happen in the Files app.

**Independent Test**: Fully testable in the Files app: double-click a `.html` file → a web_view preview window opens showing the rendered HTML. Double-click a file of a type with no registered handler → current in-app behavior is unchanged.

**Acceptance Scenarios**:

1. **Given** `text/html`'s selected handler is `html-viewer`, **When** the user double-clicks `report.html` in the Files app, **Then** a web_view preview opens showing the rendered HTML (not raw source).
2. **Given** a file type has a registered handler, **When** the user double-clicks a file of that type, **Then** the file opens in the app selected for that type.
3. **Given** a file type has **no** registered handler, **When** the user double-clicks the file, **Then** the Files app's existing in-app behavior occurs (image preview for images, text editor for text), unchanged.
4. **Given** the selected handler for a type is an app that is not currently installed, **When** the user double-clicks a file of that type, **Then** the OS falls back to the Files app's existing in-app behavior (no error, no dead launch).

---

### User Story 4 - Right-click lists every app that can handle the file ("Open with") (Priority: P1)

In the Files app, right-clicking a file shows, for each installed app registered as a handler for the file's MIME type, an **"Open with \<App\>"** entry. Choosing one opens the file in that app immediately. This is how the user picks a non-default handler for a given file.

**Why this priority**: This is the explicit context-menu request and the mechanism for choosing among multiple handlers.

**Independent Test**: Fully testable in the Files app: right-click a `.html` file → the menu lists every app registered for `text/html` as "Open with \<App\>"; choosing one opens the file in that app. For a type with no registered handlers, no "Open with" entries appear.

**Acceptance Scenarios**:

1. **Given** two apps are registered for `text/html`, **When** the user right-clicks a `.html` file, **Then** the context menu lists both as "Open with \<App\>" entries.
2. **Given** an "Open with \<App\>" entry is chosen, **When** the user selects it, **Then** the file opens in that app.
3. **Given** a hidden app (e.g. `html-viewer`, `hidden: true`) is a registered handler, **When** the user right-clicks a file of its type, **Then** it still appears in the "Open with" list using its own display label (it is not in the dock, but it is a valid handler).
4. **Given** a file type has no registered handlers, **When** the user right-clicks the file, **Then** no "Open with" entries appear (the menu still shows the existing actions such as Download).
5. **Given** a directory is right-clicked, **When** the menu is shown, **Then** no "Open with" entries appear (handlers apply to files only; directories keep their existing actions).

---

### User Story 5 - Choose which app is selected (default) for a file type (Priority: P2)

The user can change which registered app is the **selected** handler for a MIME type. Double-click (Story 3) uses this selection, so changing it changes what double-click does for that type going forward.

**Why this priority**: The user explicitly said double-click "should choose the one that is selected for the given file type" — implying a selectable per-type default, not just a manifest-fixed one. It is P2 because the feature is fully usable with manifest defaults alone (web_view for HTML); changing the default is a refinement.

**Independent Test**: Fully testable: change the selected handler for `text/html` from `html-viewer` to another registered handler, then double-click an `.html` file — it now opens in the newly selected app.

**Acceptance Scenarios**:

1. **Given** `text/html` has multiple registered handlers and a current selection, **When** the user opens the right-click "Open with" menu on an HTML file, **Then** the currently selected handler is marked (checkmark) among the entries.
2. **Given** a marked (selected) handler, **When** the user picks a different "Open with" entry, **Then** the file opens in the newly chosen app now, and it becomes the selected handler for the type — subsequent double-clicks on that type open in the new selection.
3. **Given** the user has set a selected handler for a type, **When** that app is uninstalled, **Then** the selection is invalidated and double-click falls back to the Files app's existing in-app behavior.

---

### Edge Cases

- **File type with no registered handler**: double-click keeps the Files app's existing in-app behavior (image preview / text editor); the right-click menu shows no "Open with" entries.
- **Selected handler not installed** (app removed after being selected): selection is invalidated; fall back to existing in-app behavior. No error, no dead window.
- **Hidden handler app** (e.g. `html-viewer`, `hidden: true`): appears in "Open with" using its own label, even though it has no dock icon.
- **Multiple handlers for one type**: all appear in "Open with"; exactly one is "selected" for double-click.
- **A handler that can only *edit* (not render) a type**: it appears in "Open with" for that type, but is not a valid *selected* (double-click "open") handler — double-click requires a render-capable selection.
- **A directory**: no handlers apply; existing directory behavior (navigate on open, "Download as zip" on right-click) is unchanged.
- **MIME type resolution**: the file's type is derived from its extension using the existing MIME map; an extension not in the map yields an unknown/`application/octet-stream` type, for which (typically) no handler is registered, so existing behavior applies.
- **Launching a handler that is a sandboxed iframe app**: the OS launches it with the file path as a launch param; the app is responsible for reading/previewing/editing the file (out of scope for this platform mechanism, in scope for the consuming app such as the Editor follow-on).

## Requirements

### Functional Requirements

- **FR-001**: The OS MUST allow an application to declare, in its manifest, that it can **render** and/or **edit** files of a given MIME type. A declaration identifies the MIME type (an exact type, or a type prefix such as `image/`), the capability kind (`render` and/or `edit`), an optional display label for use in "Open with", and an optional flag marking it as the default handler for the type.
- **FR-002**: At boot, the OS MUST collect file-handler declarations from **all installed apps** (built-in and marketplace) into a single **file-type-handler registry**, replacing the prior contents (uninstalled apps' declarations MUST NOT persist in the registry).
- **FR-003**: The OS MUST provide a query that returns, for a given MIME type, the list of installed apps registered as handlers for it, each with its render/edit capabilities, display label, and whether it is the current selected (default) handler for the type.
- **FR-004**: The OS MUST resolve a file's MIME type from its extension using the existing MIME map, and treat an unresolvable/unknown extension as a type for which no handler is registered.
- **FR-005**: The built-in `html-viewer` app (web_view) MUST be registered as a **render** handler for `text/html` and MUST be the **selected** handler for `text/html` by default (declared in its manifest, not hardcoded in the Files app).
- **FR-006**: In the Files app, the open gesture (double-click) on a file MUST open the file with the **selected** handler for the file's MIME type, when a render-capable selected handler exists and is installed.
- **FR-007**: In the Files app, when no render-capable handler is registered/selected (or the selected handler is not installed) for a file's type, the open gesture MUST preserve the Files app's existing in-app behavior (image preview for image types, text editor otherwise).
- **FR-008**: In the Files app, right-clicking a file MUST show an **"Open with \<App\>"** entry for **each** installed app registered as a handler for the file's MIME type (render- or edit-capable, including hidden apps shown by their own label).
- **FR-009**: Selecting an "Open with \<App\>" entry (or the open gesture, per FR-006) MUST launch the target app via the **open-file launch contract** (FR-014) and open the file there.
- **FR-010**: The OS MUST let the user change the **selected** handler for a MIME type. In the Files app right-click menu, the currently selected handler MUST be marked (e.g. a checkmark) among the "Open with" entries; choosing a different handler MUST both open the file in it now and set it as the new selected handler for that type going forward (an "always open with" interaction). A selected handler MUST be render-capable.
- **FR-011**: If a previously selected handler for a type is uninstalled, the OS MUST invalidate that selection and fall back per FR-007 (no error, no launch of a missing app).
- **FR-012**: The "Open with" affordance and the selected-handler lookup MUST apply to **files only**; directory behavior in the Files app MUST be unchanged.
- **FR-013**: Handler declarations MUST be declarative data in the app manifest (surfaced into the registry at boot), consistent with the existing `eventHandlers` registration pattern — the Files app and the registry MUST NOT contain per-app special cases for specific handlers.
- **FR-014**: There MUST be a single, documented **open-file launch contract** by which the OS hands a file to a registered handler: the OS launches the handler app carrying (a) the file's VFS path and (b) the requested action — *open* (render/preview) for the open gesture and for render-capable "Open with" choices, or *edit* for an edit-capable "Open with" choice. A handler app's obligation is defined entirely by this convention: when launched with a file, it reads, previews, or edits that file. The contract MUST be identical for built-in (component) apps and installed (iframe) apps, and MUST require no per-app code in the Files app or the registry.

### Key Entities

- **File-handler declaration** (per app, in its manifest): the MIME type(s) an app handles; the capability kind(s) — `render` and/or `edit`; an optional display label for "Open with"; an optional `default` flag (initial selected handler for the type).
- **File-type-handler registry**: the OS's boot-time, in-memory view of all installed apps' handler declarations, keyed by MIME type; the single source the Files app queries. Reflects only currently-installed apps.
- **Selected handler** (per MIME type): the app currently chosen to open files of that type on the open gesture; initially from manifest `default`, changeable by the user (FR-010), invalidated when the app is uninstalled (FR-011). Must be render-capable.
- **Open-with entry** (Files app, transient): a context-menu row per registered handler for a file's type, labelled "Open with \<App\>".
- **Open-file launch contract**: the OS↔handler convention for handing a file to a registered app — the launch carries the file's VFS path plus an action (*open*/*edit*); a handler's obligation is to read/preview/edit that file. Identical for built-in and iframe apps (FR-014).

## Success Criteria

### Measurable Outcomes

- **SC-001**: Double-clicking a `.html` file in the Files app opens a web_view preview showing the rendered HTML in 100% of cases (no raw-source text editor), with no code change to the Files app beyond the general handler lookup (the HTML→web_view binding exists only as `html-viewer`'s manifest declaration).
- **SC-002**: For any MIME type with ≥1 installed registered handler, the Files app's right-click menu shows an "Open with \<App\>" entry for each such handler (including hidden handler apps), and choosing one opens the file in that app in 100% of cases.
- **SC-003**: For any file type with no registered handler, the Files app's open and right-click behavior is byte-for-byte the existing in-app behavior (backward compatible; zero regressions for unregistered types).
- **SC-004**: Adding a new file-type handler requires only a manifest declaration in the app — zero changes to the Files app or the registry code (verified by registering a test app and seeing it appear with no core edits).
- **SC-005**: The selected handler for a type can be changed by the user and the change takes effect on the next open gesture; if that app is later uninstalled, the type falls back to existing in-app behavior with no error.

## Assumptions

- **A-1**: The open gesture is the Files app's existing **double-click** (the user clarified "click" = the open gesture). Single-click is not an open trigger.
- **A-2**: A file's MIME type is derived from its **extension** via the existing MIME map (`mimeForPath` / the raw-file route map). Content sniffing is out of scope.
- **A-3**: "Open with" and the selected-handler mechanism apply to **files only**; directories keep their current behavior.
- **A-4**: The open-file launch contract (FR-014) is implemented as standard launch params carrying the file's VFS path (and, for render handlers such as `html-viewer`, its raw-file URL) plus the requested action. **How the target app actually reads, previews, or edits the file is the consuming app's responsibility**, not this platform mechanism — the OS's job ends at launching the handler with the contract's params.
- **A-5**: An edit-only handler (no `render`) is a valid "Open with" choice but is **not** a valid *selected* (double-click) handler — double-click requires a render-capable selection.
- **A-6**: Hidden apps (`hidden: true`, e.g. `html-viewer`) are valid handlers and appear in "Open with" by their own label, despite having no dock icon.
- **A-7**: This spec implements the **platform mechanism + the Files app consumer + `html-viewer` registration**. The Agentic Editor's registration and its HTML-preview fix are a separate `marketplace-item` increment (`agentic-text-editor`) that depends on this mechanism; they are intentionally out of scope here so the platform ships independently.
- **A-8**: Handler declarations live in the app **manifest** (the self-describing `src/apps/<id>/manifest.ts` for built-ins and the item's `app.json` for installed apps), mirroring `AppManifest.eventHandlers` (spec 034). The registry is boot-time and in-memory (it reflects installed apps, which are already discovered at boot); persisting a user's per-type *selection* (FR-010) is the only durable state this feature introduces.

## Dependencies

- Existing OS store `launch(appId, params)` and `AppProps` param delivery (present).
- Existing MIME map: `src/lib/mime.ts` (`mimeForPath`) and `src/app/api/fs/raw/route.ts` (present).
- Existing manifest extension-point pattern: `AppManifest.eventHandlers` + boot-time registry (spec 034; present).
- Built-in `html-viewer` app (web_view) and its raw-file URL resolution (present).
- **Follow-on (consumer):** `agentic-text-editor` marketplace item — declares a `text/html` handler in its `app.json` and fixes HTML rendering in Preview mode. Depends on FR-001 (declaration), FR-003 (registry query), FR-009 (launch), and FR-014 (the open-file launch contract it must honor to read/preview/edit the handed file).


