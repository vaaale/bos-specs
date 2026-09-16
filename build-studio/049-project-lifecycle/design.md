# Design: Project lifecycle (049)

## 1. What is new

Three modules and one correction. Nothing in the method layer changes shape.

| | |
|---|---|
| `method/workflows.ts` | the unit a project binds to, enumerated across installed methods |
| `store-kind.ts` | binding scope and "what a project is", per repository kind |
| `lifecycle.ts` | create / rename / delete, dispatched on kind — ONE implementation |
| `create.ts` | FR-012: resolve the full binding chain, and accept a workflow |

## 2. Constitution check

- **Server decides, client displays.** Kind dispatch, workflow resolution and
  refusals are all server-side; the tree renders what it is told.
- **No workarounds.** Two operations turned out to be different things than they
  looked (§3.4) and are REFUSED with the reason rather than approximated.
- **One implementation.** §3.5.

## 3. Decisions

### 3.1 A workflow, not a method — and the compatible case is the DEFAULT

`resolveWorkflow()` accepts `<method>:<workflow>`, a bare workflow id, or a bare
**method** id. The last one matters most: it resolves to that method's default
workflow, so every existing `method: "bmad"` binding keeps working untouched.

A method declaring no workflows therefore has exactly **one**, named for itself.
That is expressed as the normal case rather than a compatibility shim, because a
shim is something a later refactor removes.

Two failure modes, deliberately different:

- **Unknown** → refuse, listing what exists. Never fall back. A project silently
  created under the wrong framework is discovered weeks later, by its artifacts
  having the wrong names.
- **Ambiguous** → report both providers and the qualified form. Resolving by
  registration order makes "which one did I get" depend on history nobody can
  inspect — the same reasoning as `045` FR-001a for duplicate agent ids.

A bare **method id beats another method's workflow of the same name**, so
`resolveWorkflow("bmad")` always means BMAD.

### 3.2 Binding scope is a property of the KIND — and it is NOT the lifecycle

The easiest confusion to fall into here, and it misleads in both directions.

| | asks | answer varies by |
|---|---|---|
| **Binding scope** | how many workflows can this repo have? | kind |
| **Lifecycle unit** | what does "a project" mean here? | kind |

They are different questions with different answers, and `user-specs` is where
that shows: it binds **one** workflow for everything in it **and** contains
folders. So `bindingScopeOf()` and `projectsAre()` are separate functions
returning different vocabularies, rather than one "kind capability" object.
Merged into one thing, one of the two answers is always wrong.

### 3.3 Bind at BIRTH, not after

`createItemSpec` now takes a workflow and decides the primary artifact's name
from it, and the item's binding is written in the same call.

This is the whole usability fix. Binding afterwards means the first artifact was
written under the old method's leaf marker (`spec.md`) and the new method does
not discover it (`product-brief.md`) — so the preflight correctly warns that a
unit would be hidden and the user renames a file by hand. Four steps, three of
them consequences of the first being wrong.

### 3.4 Two operations that are not what they look like

Both REFUSED with the reason, rather than approximated:

- **Renaming a marketplace item** is an install-identity change. The id is the
  `data/system/<id>` symlink, the marketplace manifest entry, and the app
  registration. A "rename" that moved only the directory would leave an item
  installed under its old identity.
- **Deleting one** is an uninstall plus a repository change.

Folders have neither problem, which is why the same verbs work there. The verb
being shared is not evidence the operation is.

### 3.5 The tools and the UI are ONE implementation

`lifecycle.ts` is called by the agent tools and (next) by the Build Studio
context menus. Not a convention — a response to evidence: every divergence in
this subsystem has come from two paths to one outcome (`getAgent` vs the
listings, the picker's method vs the resolved one, install vs boot
registration), and the UI path is the one found last.

### 3.6 Confirmation belongs to the surface, but the server still refuses

`deleteProject()` takes no `confirm` flag. A boolean passed through an API is
not a confirmation — it is a parameter an agent can set.

So confirmation is elicited **twice, by different mechanisms**: the tool returns
the unit count and refuses until told again, and the route refuses an
unconfirmed POST. A destructive operation reachable by one unguarded request is
not protected by a dialog living somewhere else.

The count is load-bearing. "Delete the empty folder I just made" and "delete 30
specs" are the same click without it.

## 4. Risks

- **R1 — arbitrary repos.** The lifecycle verbs may not make sense for a repo
  that is itself one project. Kept as-is pending `050`; flagged, not resolved.
- **R2 — the scaffold is declared but unused.** `ProjectScaffold` exists in the
  descriptor and no shipped pack declares one, so FR-007/FR-008 are exercised by
  fixtures only. `051` is its first real consumer — a field with no consumer is
  how `terminal` and `sections` came to be half-honoured (`045` D-4).
- **R3 — item rename/delete are refused**, so a marketplace store's projects are
  create-only. Acceptable for now; the alternative is an install-state refactor.

## 5. Files

```
src/lib/specs/method/workflows.ts    resolution, collisions, defaults
src/lib/specs/store-kind.ts          kindOf / bindingScopeOf / projectsAre
src/lib/specs/lifecycle.ts           create / rename / delete, kind-dispatched
src/lib/specs/item-binding.ts        setItemWorkflow (merge, never reconstruct)
src/lib/specs/projects.ts            renameProject / deleteProject / unit count
src/lib/specs/create.ts              FR-012 + workflow-aware artifact naming
src/app/api/specs/route.ts           the four lifecycle ops
src/lib/assistant/tools/server/specs.ts   the three tools
```
