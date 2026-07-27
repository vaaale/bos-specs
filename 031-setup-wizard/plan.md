# Implementation Plan: Configuration Wizard (First-run Setup)

## Overview

Replace `src/components/desktop/FirstRunWizard.tsx` with a 6-step wizard modal. Steps 1–5 are navigable forms; Step 6 is a non-interactive progress screen that fires all setup operations and then closes to the desktop.

The wizard is purely a shell: most steps embed existing Settings tab components unchanged. New work is concentrated in the wizard shell (navigation, state), Step 4 (new git-repos form), Step 5 (new marketplace table), and Step 6 (setup execution + progress display).

---

## Architecture

### Component tree

```
SetupWizard                          ← replaces FirstRunWizard.tsx
  WizardShell                        ← modal overlay, step indicator, Back/Next
    Step1_AiProvider                 ← extracts fields from old FirstRunWizard.tsx
    Step2_DevHarness                 ← embeds <DevHarnessTab> as-is
    Step3_DataIsolation              ← embeds <DataFsTab> as-is
    Step4_GitRepos                   ← new component (3 repo rows)
    Step5_Marketplace                ← new component (checked table)
    Step6_SettingUp                  ← new component (progress list, auto-close)
```

### State management

All wizard state lives in `SetupWizard` as plain `useState`. Nothing is persisted between renders — the wizard runs once. Each embedded tab component (`DevHarnessTab`, `DataFsTab`) persists its own namespace via its existing Save button; the wizard does not intercept those saves.

Step 4 and Step 5 values are held in wizard state and consumed only by Step 6.

```ts
interface WizardState {
  step: 1 | 2 | 3 | 4 | 5 | 6;
  // Step 1
  provider: string; model: string; baseUrl: string; apiKey: string;
  // Step 4
  bosSource: { url: string; branch: string };
  bosSpecs:  { url: string; branch: string };
  userApps:  { url: string; branch: string };
  // Step 5
  marketplaces: Array<{ name: string; url: string; checked: boolean }>;
}
```

---

## Step-by-step design notes

### Step 1 — AI Provider

Extract the provider/model/baseUrl/apiKey fields from the current `FirstRunWizard.tsx`. Save is deferred to Finish (written on Step 6 entry alongside the other setup operations, or written immediately before Step 6 starts — either works; immediate write is simpler and means later steps can test against a saved provider).

Next is disabled until `provider` is non-empty.

### Step 2 — Dev Harness

`<DevHarnessTab>` is mounted inside the wizard body. Its existing Save button works normally — it writes to `data/config/dev-harness.json` whenever the user clicks it. The wizard's Next button does **not** trigger a save; the user is responsible for saving if they want their changes kept.

This is the same pattern as Settings: the tab is self-contained. The wizard just provides the surrounding chrome.

### Step 3 — Data Isolation

Same pattern as Step 2: embed `<DataFsTab>` as-is.

### Step 4 — Git Repositories

New component with three labeled rows. Each row: a URL text input + a branch text input. Default values pre-filled (see FR-007). All optional — no validation beyond "if URL is non-empty, branch should also be set" (show a soft warning, not a hard block).

Values are held in wizard state and not written to any config file. They are consumed by Step 6's setup routine.

### Step 5 — Marketplace

New component: a table of `{ name, url, checked }` rows. Initial rows:

| Name | URL | Checked |
|------|-----|---------|
| BOS Central marketplace | https://github.com/vaaale/bos-marketplace.git | ✓ |
| Claude Superskills | https://github.com/ericgandrade/claude-superskills.git | ☐ |
| Anthropic Skills | https://github.com/anthropics/skills.git | ☐ |

An "Add" button lets the user append a custom row (name + URL, checked by default).

Values held in wizard state; consumed by Step 6.

### Step 6 — Setting Up

Entered by pressing "Finish & Set Up" on Step 5. Non-interactive — no Back/Next buttons. Runs the following operations in sequence/parallel as noted:

**Parallel group (repo operations):**
1. BOS Source: if URL non-empty → add git remote `origin` to `bos-src` GitFS; if empty → no-op (the bos-src directory already exists as the running process CWD).
2. BOS Central Specs: if URL non-empty → clone repo into system spec-store location, skip seed; if empty → run seed routine + git-init if not already a git repo.
3. User Apps: if URL non-empty → clone into `data/user-apps/`; if empty → `mkdir -p data/user-apps/` + git-init.

**Write Step 1 values** (can be done before parallel group, blocking):
- `PATCH /api/config` with `namespace: "ai-provider"` and entered values.

**Sequential group (marketplace operations, after repo group):**
4. For each checked marketplace row: `POST /api/marketplace { op: "add", url }`.
   Sequential because `add` clones to `data/marketplace/<id>/` and concurrent git clones to sibling directories have caused issues historically.

**Completion:**
5. `POST /api/system/setup` → `setupComplete = true`.
6. Wizard state machine exits → `Desktop` renders.

Each operation maps to one `SetupOperationRow` in the UI:
```
[ spinner | ✓ | ✗ ]  Operation label            [ optional error message ]
```

---

## BOS Central Specs — seed guard

`src/lib/specs/seed.ts` currently seeds unconditionally on first run. After this change it must check whether the target system spec-store path is already a git repository (i.e. contains a `.git` directory) before running. If it is, skip seeding. This is a one-line guard; it does not change the seed logic itself.

---

## Deletion of old wizard

`src/components/desktop/FirstRunWizard.tsx` is deleted. The import in `src/components/desktop/Desktop.tsx` is updated to point at the new component. The `GET /api/system/setup` and `POST /api/system/setup` routes are unchanged.

---

## File layout

```
src/components/desktop/
  SetupWizard.tsx            ← new root (replaces FirstRunWizard.tsx)
  setup-wizard/
    WizardShell.tsx          ← modal, step indicator, Back/Next
    Step1AiProvider.tsx
    Step2DevHarness.tsx
    Step3DataIsolation.tsx
    Step4GitRepos.tsx
    Step5Marketplace.tsx
    Step6SettingUp.tsx
    wizard-types.ts          ← WizardState, SetupOperation types
```
