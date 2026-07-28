# Pluggable Voice Engines

**Status:** Draft  
**Scope:** Voice pipeline  
**Depends on:** `032-dynamic-integration-plugins` (Phase A + C)

---

## 1. Problem Statement

BOS's TTS engine selection is hardcoded in `src/lib/voice/tts/index.ts`. Two engine
types are supported (`openai-compatible`, `omnivoice`) and the selection logic is a
plain `switch`. A marketplace plugin cannot register a new engine type — it would
require editing source and recompiling.

The `live-avatar` marketplace item needs to register a `live-avatar` engine that
replaces the normal TTS audio path with a WebSocket-based avatar rendering pipeline.
Other future plugins may need similar hooks (e.g. a local Kokoro TTS engine, a
custom voice cloning service).

The goal is to make the voice engine selection dynamic: engines are registered at
server startup by plugins (and by the built-in engine modules themselves), and
`useVoice.ts` selects between them based on the configured engine id.

---

## 2. Goals

- **G1** — A plugin can register a new voice engine type via `sdk.registerVoiceEngine()`.
- **G2** — The BOS voice hook (`useVoice.ts`) routes TTS calls through whatever engine
  is currently configured, including plugin-registered engines.
- **G3** — The voice settings UI discovers available engines at runtime and lists them
  in the engine selector.
- **G4** — Built-in engines (OpenAI-compatible, OmniVoice) continue to work without
  any changes to their current behaviour.
- **G5** — When a plugin-provided engine is active, the voice hook suppresses the
  default `Audio` element and defers entirely to the engine's speak/interrupt contract.

## 3. Non-Goals

- **NG1** — STT (speech-to-text) extensibility. Only TTS/output engines are in scope.
- **NG2** — Multiple simultaneous engines. Exactly one engine is active per voice
  session.
- **NG3** — Engine hot-swap mid-session. Engine selection takes effect on the next
  voice session start.

---

## 4. Current Architecture

```
useVoice.ts
  └── speak(text)
        └── fetch POST /api/voice/tts { text }
              └── streamSpeech(text, cfg)    [src/lib/voice/tts/index.ts]
                    ├── streamOpenAI(...)     [if cfg.ttsProvider === "openai-compatible"]
                    └── streamOmnivoice(...)  [if cfg.ttsProvider === "omnivoice"]
```

`VoiceConfig.ttsProvider` is a string union `"openai-compatible" | "omnivoice"`.
The Settings UI exposes a static dropdown of these two values.
`/api/voice/tts` returns a streaming HTTP response containing audio/mpeg.

There is no registration mechanism. Adding an engine requires editing
`src/lib/voice/tts/index.ts` and `src/lib/voice/types.ts`.

---

## 5. Voice Engine Interface

```typescript
// src/lib/plugins/types.ts (new, exported via @bos/plugin-sdk)

export interface VoiceEnginePlugin {
  /** Unique identifier; matches VoiceConfig.ttsProvider when selected. */
  id: string;

  /** Human-readable name shown in the Settings engine selector. */
  displayName: string;

  /**
   * Called once per voice session start, before the first speak() call.
   * Use to open persistent connections (e.g. WebSocket to avatar server).
   * sessionId is a stable id for this voice session.
   */
  onSessionStart?(sessionId: string): Promise<void>;

  /**
   * Called when the voice session ends (user leaves, tab closes, engine
   * changes). Close any persistent connections here.
   */
  onSessionEnd?(sessionId: string): Promise<void>;

  /**
   * Produce speech for the given text. The engine is responsible for
   * delivering audio to the user by whatever means it uses (audio element,
   * WebRTC track, etc.). Returns { durationMs } so the voice hook knows
   * when to transition back to listening state.
   *
   * Invoked via POST /api/voice/tts when this engine is active; the route
   * calls speak() and returns the result as JSON instead of streaming audio.
   *
   * sessionId matches the value passed to onSessionStart.
   */
  speak(
    text: string,
    config: VoiceConfig,
    sessionId: string,
  ): Promise<{ durationMs: number }>;

  /**
   * Interrupt any in-progress speech immediately (barge-in).
   * Called by the voice hook when VAD detects the user speaking mid-utterance.
   */
  interrupt(sessionId: string): Promise<void>;

  /**
   * Optional: JSON Schema for engine-specific configuration fields appended
   * to the voice settings panel when this engine is selected.
   */
  configSchema?: JSONSchema;
}
```

---

## 6. Voice Engine Registry

**New file:** `src/lib/voice/engine-registry.ts`

```typescript
const KEY = "__bos_voice_engines__";
const registry: Map<string, VoiceEnginePlugin> =
  (globalThis as any)[KEY] ??= new Map();

export function registerVoiceEngine(engine: VoiceEnginePlugin): void {
  registry.set(engine.id, engine);
}

export function unregisterVoiceEngine(id: string): void {
  registry.delete(id);
}

export function getVoiceEngine(id: string): VoiceEnginePlugin | undefined {
  return registry.get(id);
}

export function listVoiceEngines(): VoiceEnginePlugin[] {
  return [...registry.values()];
}
```

Built-in engines register themselves at module load, exactly like adapters:

```typescript
// src/lib/voice/tts/openai.ts  (new, split from index.ts)
import { registerVoiceEngine } from "../engine-registry";

registerVoiceEngine({
  id: "openai-compatible",
  displayName: "OpenAI-compatible TTS",
  async speak(text, config, _sessionId) {
    // existing streamOpenAI logic, returns { durationMs }
  },
  async interrupt(_sessionId) { /* no-op for HTTP TTS */ },
});
```

`src/lib/voice/tts/index.ts` becomes a thin dispatcher:

```typescript
import { getVoiceEngine } from "../engine-registry";

export async function streamSpeech(
  text: string,
  cfg: VoiceConfig,
  sessionId: string,
): Promise<{ durationMs: number }> {
  const engine = getVoiceEngine(cfg.ttsProvider);
  if (!engine) throw new Error(`Unknown TTS engine: ${cfg.ttsProvider}`);
  return engine.speak(text, cfg, sessionId);
}
```

---

## 7. Changes to `/api/voice/tts`

The route currently returns a streaming HTTP response (audio/mpeg body). This must
change to accommodate engines that deliver audio through other channels (WebRTC,
WebSocket) and have nothing to stream back to the HTTP caller.

**New contract:** `POST /api/voice/tts` always returns JSON:

```json
{ "ok": true, "durationMs": 3200 }
```

For built-in engines that produce a streamable audio response, the route writes the
audio to a temporary buffer, measures its duration, returns `{ durationMs }` to the
caller, **and** triggers playback on the client by setting a response header or
returning an audio URL:

```json
{ "ok": true, "durationMs": 3200, "audioUrl": "/api/voice/tts/audio/<token>" }
```

`useVoice.ts` fetches `audioUrl` and plays it as before. Plugin-registered engines
that deliver audio through other means return `{ ok: true, durationMs }` with no
`audioUrl`; `useVoice.ts` skips the audio element and uses `durationMs` to time the
speaking state.

> **Note:** This is a breaking change to the `/api/voice/tts` response shape. The
> migration path is: add `audioUrl` support first, migrate `useVoice.ts` to use it,
> then change built-ins to return JSON.

---

## 8. Voice Session Management

Plugin engines that hold persistent connections (WebSocket, etc.) need a stable
session identifier to key their state on. The voice hook generates a session ID on
first activation and passes it through all engine calls.

**New API routes:**

```
POST /api/voice/session/start   →  { sessionId }
POST /api/voice/session/end     →  { ok: true }
```

`useVoice.ts` calls `session/start` when voice mode is activated and `session/end`
when it is deactivated. The route calls `engine.onSessionStart(sessionId)` /
`engine.onSessionEnd(sessionId)` on the configured engine if those methods exist.

The session ID is stored in a `useRef` in `useVoice.ts` and included on every
`/api/voice/tts` and `/api/voice/interrupt` call.

---

## 9. Voice Settings UI — Engine Discovery

**Modified:** `src/app/api/voice/route.ts` — `GET /api/voice` response gains an
`engines` field:

```json
{
  "config": { … },
  "engines": [
    { "id": "openai-compatible", "displayName": "OpenAI-compatible TTS", "configSchema": null },
    { "id": "omnivoice",         "displayName": "OmniVoice TTS",         "configSchema": null },
    { "id": "live-avatar",       "displayName": "Live Avatar",           "configSchema": { … } }
  ]
}
```

**Modified:** `src/components/apps/settings/VoiceTab.tsx` — the engine selector
dropdown is populated from `engines[]` instead of a hardcoded list. When the
selected engine has a `configSchema`, the settings panel renders a schema-driven form
for those fields appended below the standard voice controls.

`VoiceConfig.ttsProvider` type widens from a string union to `string` to accommodate
plugin-registered ids.

---

## 10. `useVoice.ts` Changes

The voice hook requires targeted changes to support engine-agnostic output:

1. **Session lifecycle:** Call `POST /api/voice/session/start` on activation,
   `POST /api/voice/session/end` on deactivation. Store `sessionId` in a ref.

2. **Engine-agnostic speak:** After calling `POST /api/voice/tts`:
   - If response contains `audioUrl` → fetch and play via `Audio` element (existing
     behaviour, used by built-in engines).
   - If response contains only `durationMs` → set status to `"speaking"` and
     schedule a transition back to `"dormant"` after `durationMs` ms (used by
     plugin engines that deliver audio elsewhere).

3. **Interrupt:** `POST /api/voice/interrupt` is added (new route); also calls
   `engine.interrupt(sessionId)` server-side. `useVoice.ts` calls this route when
   barge-in is detected, in addition to stopping any local `Audio` element.

4. **Audio suppression flag:** If the active engine has no `audioUrl` in its speak
   response, the hook must not create an `Audio` element at all. This prevents any
   double-audio when the engine delivers sound through another channel (e.g. WebRTC
   track in the avatar app iframe).

---

## 11. `voiceEngine` Marketplace Facet

Defined in `032-dynamic-integration-plugins` §7.7. The marketplace install logic for
this facet:

1. Copies the plugin to `data/bos-plugins/<id>/`.
2. Calls the plugin loader to activate it.
3. The plugin's `activate()` calls `sdk.registerVoiceEngine(engine)`.
4. The engine appears in the voice settings selector on the next `GET /api/voice`
   call — no restart required.

---

## 12. Affected Files

### New files
- `src/lib/voice/engine-registry.ts` — voice engine registry
- `src/lib/voice/tts/openai.ts` — OpenAI engine split from index.ts
- `src/lib/voice/tts/omnivoice.ts` — OmniVoice engine split from index.ts
- `src/app/api/voice/session/route.ts` — session start/end routes
- `src/app/api/voice/interrupt/route.ts` — server-side interrupt route

### Modified files
- `src/lib/voice/tts/index.ts` — becomes thin dispatcher via `getVoiceEngine()`
- `src/lib/voice/types.ts` — `ttsProvider: string` (widened from union)
- `src/app/api/voice/tts/route.ts` — returns JSON `{ durationMs, audioUrl? }` instead of streaming audio body
- `src/app/api/voice/route.ts` — add `engines[]` to GET response
- `src/components/apps/settings/VoiceTab.tsx` — dynamic engine selector
- `src/hooks/useVoice.ts` — session lifecycle, engine-agnostic speak, interrupt route, audio suppression

---

## 13. Phasing

### Phase 1 — Engine registry + built-in migration
Split built-in engines into `openai.ts` / `omnivoice.ts`, both self-registering.
`tts/index.ts` dispatches via registry. No behaviour change; validation: existing
voice mode works identically.

### Phase 2 — `/api/voice/tts` JSON response + `useVoice` update
Add `audioUrl` to built-in engine responses. Update `useVoice.ts` to consume it.
Add session start/end routes and session ID threading. Add interrupt route.

### Phase 3 — Engine discovery in settings
`GET /api/voice` includes `engines[]`. VoiceTab populated dynamically.

### Phase 4 — Plugin engine support
`sdk.registerVoiceEngine()` wired to registry. `useVoice.ts` audio suppression for
engines without `audioUrl`. `voiceEngine` marketplace facet install/uninstall.
Validation: live-avatar plugin installed, engine selectable, speak/interrupt
functional end-to-end.
