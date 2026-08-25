# Implementation Plan: Media Support in `web_view` (Images & Video)

**Branch**: `web-view-media` | **Date**: 2026-08-25 | **Spec**: [spec.md](./spec.md)

**Design**: [design.md](./design.md) — read for full ADRs, component diagram, and risk register.

## Summary

Extend the `web_view` tool and its built-in `html-viewer` app to render image and video targets with a media presentation mode (centered, fit-to-frame, neutral stage, native playback controls). The tool handler classifies the target (extension/MIME) using a shared framework-free module, and for media passes an explicit `{ mode, src, poster?, autoplay?, loop?, muted? }` to the app, which renders native `<img>`/`<video>` instead of the existing iframe. Document and extend all three tool declaration surfaces and the raw-route MIME map.

## Technical Context

**Language/Version**: TypeScript (Next.js 14, React 18)

**Primary Dependencies**: None new — native `<img>`/`<video>` + Tailwind (already in the app)

**Storage**: N/A (media served from VFS or external URLs)

**Testing**: `npx tsc --noEmit`, `npm run lint` (standard gates); manual verification via `web_view` tool calls

**Target Platform**: Chromium-based desktop (BOS runs as a local desktop app)

**Project Type**: Web application (single Next.js app)

**Performance Goals**: Image preview loads within 200ms for files < 5MB; video streams (no full-client buffering)

**Constraints**: Must not alter existing HTML/URL iframe rendering (NFR-004); sandbox boundary preserved for documents (FR-008); no new external dependencies

**Scale/Scope**: ~6 files modified, 1 file created; single feature branch, single `dev_delegate`

## Constitution Check

| Principle | Status |
|---|---|
| I. Spec-Driven | ✅ spec.md + design.md precede implementation |
| II. Server Authority | ✅ Only server edit is two keys in the raw-route MIME map (a lookup table); no new client→server paths |
| III. Always Delegate | ✅ `dev_delegate` on `web-view-media` branch |
| IV. Minimize Blast Radius | ✅ One feature branch; media mode is additive (new render path), iframe path unchanged |
| V. VFS Is Not the Source | ✅ All edits under `src/` via Developer |
| VI. Specs & Docs Stay in Sync | ⚠️ Must update `docs/dev/assistant/actions-and-tools.md` and create/update the `docs/usage` page for `web_view` (reviewer finding: no dedicated usage page exists yet — the Developer should create one at `docs/usage/assistant/web-view.md` or fold the media params into `docs/usage/assistant/using-the-assistant.md`) |
| VII. Respect Boundaries | ✅ No package.json/lockfile/build-config change |

**Gate**: PASS — no violations requiring justification.

## Project Structure

### Source files (create/modify)

```text
src/
├── lib/apps/media.ts                    # NEW — shared classifier (IMAGE_EXTENSIONS, VIDEO_EXTENSIONS, classifyMediaTarget)
├── apps/html-viewer/index.tsx           # MODIFY — add media render branch (mode → <img>/<video> on #1a1a1a stage)
├── components/agent/v2/FrontendToolsV2.tsx  # MODIFY — v2 handler: classify + pass media params
├── components/agent/OSActions.tsx       # MODIFY — v1 handler + description + params (consistency)
├── lib/assistant/tools/frontend-declarations.ts  # MODIFY — v2 model-facing description + new params
├── lib/agent/capabilities-registry.ts   # MODIFY — Settings catalog one-liner
└── app/api/fs/raw/route.ts              # MODIFY — add .m4v / .avi to MIME map

docs/
├── dev/assistant/actions-and-tools.md   # MODIFY — web_view row/description
└── usage/assistant/web-view.md          # CREATE (or fold into existing page) — user-facing docs
```

### Documentation (this feature)

```text
/Specs/user-specs/html-viewer/web-view-media/
├── spec.md          # Feature specification
├── design.md        # Architecture, ADRs, risk register (architect-authored)
├── mockup.html      # UI mockup (ui-designer-authored)
├── plan.md          # This file
└── tasks.md         # Task breakdown (next step)
```

**Structure Decision**: Single Next.js project; all changes are within existing app structure. No new directories beyond `src/lib/apps/` for the shared classifier.

## Key Design Decisions (from design.md)

1. **ADR-1 — Native media elements** (not iframe): `<img>`/`<video>` rendered directly in the React component. Satisfies FR-008 security property + SC-002 fullscreen + FR-007 clean error handling. Precedent: Files app.
2. **ADR-2 — Handler-classified, mode-driven app**: The tool handler calls `classifyMediaTarget()` and passes explicit `mode`. The app is dumb — it renders what `mode` says. No extension-sniffing in the app.
3. **ADR-3 — Extend MIME map**: Add `.m4v` → `video/mp4`, `.avi` → `video/x-msvideo` to the raw route so the classifier and served Content-Type agree.

## Implementation Notes

- The media element is `key`'d on `src` so `update=true` always re-fetches (FR-009).
- `poster` VFS paths must be rewritten to raw URLs (same rule as `url`/`filePath`) — design risk R4.
- In-window error card: centered message on the `#1a1a1a` stage when `<img onError>` / `<video onError>` fires (FR-007).
- External URLs are not server-side verified (risk R3); the in-window error card handles bad external media.
- The v1 `OSActions.tsx` handler mirrors v2 logic using the same shared `media.ts` module (risk R2 mitigation).
