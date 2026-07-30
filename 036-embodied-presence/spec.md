# Feature Specification: Embodied Presence — A Voice Engine With a Face

**Feature Branch**: `036-embodied-presence`

**Created**: 2026-07-30

**Status**: New

**Input**: "I want to add support for integrating live avatar into the assistant app. mic = voice input toggle, speaker = audio output toggle, video = audio output + video toggle. If the user toggles video on, the avatar window opens and audio output is on. The face should open in a window, much smaller than today, placed in the upper left quadrant of the screen (not jammed up in the corner). I don't see any need for keeping the app. I do however want to be able to integrate the 'face' in other situations — for example, if I get a new email, the agent could pop up and say 'Hey you have a new email'. Out of scope for now, but keep it in mind. It connects when the user toggles the video button."

> This spec owns **the agent's visible embodiment**: how a voice engine declares a
> visual surface, how BOS hosts it, and the output control model (off / audio /
> avatar). `033-pluggable-voice-engines` owns the engine contract, the TTS
> pipeline and the audio-sink handoff — §9a there owns the single-producer rule
> this spec extends. `009-installed-apps` owns the app surface;
> `035-install-by-symlink` owns installed state. This spec supersedes 033's
> assumption that a plugin engine's UI is an ordinary installed app.

## Why this exists

`live-avatar` today is one item with two facets: a Node plugin (voice engine +
routes + settings panel) and a *plugin-served app* — an `app.json` whose only
content is `appUrl: /api/plugin/live-avatar/app`, so the app window is an iframe
onto a plugin route serving one static HTML file. Inside that iframe lives the
`RTCPeerConnection` (recvonly audio + video); the avatar's face and voice arrive
straight from the AVTR-1 server. BOS's server pushes mp3 into AVTR-1 over a
server-held WebSocket, and suppresses local `<audio>` playback while that socket
is open (`/api/voice/tts` returns `audioUrl: null, routedTo: "live-avatar"`).

That works, and four things about it don't:

1. **The avatar is an app, so it behaves like one.** It occupies a dock entry, a
   desktop icon and an 80%-of-viewport window, and the user connects it with a
   button *inside* the iframe. None of that is what a face is for. It is not a
   place you go; it is how the assistant appears while it talks to you.
2. **The sink outlives the surface.** `isSinkActive()` is true whenever the
   server's WebSocket is open, and closing the window without clicking Disconnect
   leaves it open. Every reply is then routed to an avatar nobody can see, and the
   browser plays nothing: a **silent assistant with no error**.
3. **Turning it on is three unrelated acts** — enable spoken replies in one place,
   open an app, click Connect inside it — for what the user experiences as one
   decision: "talk to me with a face".
4. **Nothing else can use it.** A face that can only be summoned by opening an app
   cannot be summoned by an event ("you have a new email").

## Clarifications

### Session 2026-07-30

- Q: How do the controls relate? → A: `mic` = voice input, `speaker` = audio
  output, `video` = audio output **and** video. Toggling video on opens the face
  and turns audio output on.
- Q: Where does the face open? → A: In a window, much smaller than the app is
  today, in the upper-left quadrant — offset from the corner, not jammed into it.
- Q: Does the standalone app survive? → A: No need for it, at least not as it is
  today. The face must remain usable from contexts other than the Assistant
  (e.g. an incoming-email notification) — out of scope here, but not precluded.
- Q: When does it connect? → A: When the user toggles the video button.
- Q: How is the window sized? → A: Deduce the aspect from the video stream; set
  height to 25% of the viewport and compute width from the ratio.
- Q: Does the face survive a page reload? → A: It follows the Assistant app.
  (BOS persists no window state and auto-launches nothing, so this means: it does
  not survive, and nothing reconnects on load.)
- Q: Always-on-top? → A: Yes, as a toggle on the window itself — placed on the
  right of the title bar rather than beside the traffic lights.

## User Scenarios & Testing *(mandatory)*

### User Story 1 — One decision turns on a talking face (Priority: P1)

The user clicks the video button in the Assistant input row. The face appears in a
small window in the upper-left quadrant, already connected, and the next reply is
spoken by it. No app to find, no Connect button, no separate "enable speech"
setting.

**Acceptance**: one click yields a connected face and spoken replies. Clicking it
again closes the face and leaves audio replies on. Clicking the speaker button
turns all output off, face included.

### User Story 2 — The output control cannot contradict itself (Priority: P1)

There is no state in which the face is on screen and spoken output is off, because
output is one setting with three values rather than two booleans that can disagree.

**Acceptance**: no sequence of clicks on speaker/video produces a visible face with
silent output, or an "avatar" state with no window.

### User Story 3 — A dead avatar never swallows the conversation (Priority: P1)

If the avatar's media path fails or its window disappears, replies are spoken by
the browser instead. The user never faces a silent assistant.

**Acceptance**: killing the surface (close the window, break WebRTC, kill the
avatar server) causes the next reply to play locally, and the video toggle to fall
back to `audio`.

### User Story 4 — The face is not tied to the Assistant (Priority: P2)

Any BOS subsystem can bring the face up and have it speak — the Assistant's video
toggle is one caller among several. (Only the mechanism is in scope here; event
triggers such as incoming mail are not.)

**Acceptance**: opening the presence window and speaking through it requires no
Assistant window to be open.

### User Story 5 — Any window can be pinned (Priority: P2)

A pin on the right of the title bar keeps a window above unpinned ones, so a
talking face (or any reference window) stays visible while the user works
elsewhere.

**Acceptance**: a pinned window stays above every unpinned window regardless of
focus order; unpinning restores normal stacking.

### Edge Cases

- The stream's aspect ratio is unknown until the first frame: the window opens at
  a provisional size and snaps once on `loadedmetadata`.
- Two callers ask for the face at once: the presence window is a singleton, and one
  media session exists at a time.
- The engine that provides the face is **not** necessarily the selected TTS engine
  (the avatar renders audio from any engine, as an audio sink).
- No engine declares a surface: the video button is not rendered at all.
- A reply arrives while the face is connecting: it is spoken by the browser, since
  the lease is not yet live.

## Requirements *(mandatory)*

### Functional Requirements

**Output control**

- **FR-001**: Voice output is ONE persisted setting, `VoiceConfig.voiceOutput ∈
  {"off", "audio", "avatar"}`, replacing the `speakReplies` boolean. Stored
  `speakReplies: true` migrates to `"audio"`, `false`/absent to `"off"`.
- **FR-002**: TTS is gated on `voiceOutput !== "off"` — the single gate from
  033 §9a FR-033a-01, unchanged in spirit.
- **FR-003**: The speaker button maps `off → audio` and anything else `→ off`.
  The video button maps `≠ avatar → avatar` and `avatar → audio`.
- **FR-004**: Closing the presence window sets `voiceOutput` to `"audio"`.
- **FR-005**: A fresh page load demotes a persisted `"avatar"` to `"audio"` and
  persists it. BOS restores no windows and auto-connects nothing; the audio
  preference survives a reload, the face does not.
- **FR-006**: The video button renders only when some registered voice engine
  declares a surface.

**Engine surface**

- **FR-007**: `VoiceEnginePlugin` gains an optional `surface: { url: string;
  label?: string }`. `GET /api/voice` exposes it on each entry of `engines[]`.
- **FR-008**: BOS hosts the surface in a hidden, singleton, built-in app window —
  no dock entry, no desktop icon, launched programmatically.
- **FR-009**: The presence provider is selected independently of `ttsProvider`:
  the first registered engine declaring a surface provides the face.
- **FR-010**: The surface reports its intrinsic aspect ratio to BOS, which sizes
  the window to 25% of viewport height and the corresponding width, positioned in
  the upper-left quadrant offset from the corner.
- **FR-011**: BOS instructs the surface to connect when the window opens; the
  surface exposes no connect/disconnect UI of its own.
- **FR-012**: BOS tells the surface when the agent starts and stops speaking, so
  it can show that state.

**Sink safety**

- **FR-013**: An engine's audio sink is used ONLY while a presence **lease** is
  live: the surface renews it while it is actually rendering media, and it expires
  on its own. `findActiveAudioSink()` requires both `isSinkActive()` and a live
  lease.
- **FR-014**: When no lease is live, audio plays in the browser as usual — the
  fallback is automatic and needs no user action.

**Window shell**

- **FR-015**: `launch()` accepts explicit placement (size and position) that
  bypasses the "80% of viewport" default sizing. That default remains for apps
  launched normally.
- **FR-016**: `WindowInstance` gains `alwaysOnTop`, toggled from a pin control on
  the RIGHT of the title bar (replacing the balance spacer, so the centred title
  does not shift). Pinned windows occupy a z-band above all unpinned windows, so
  focusing an unpinned window cannot cover them.
- **FR-017**: Programmatic resize is not subject to the user-drag minimum that
  would distort a small portrait surface.

**Item shape**

- **FR-018**: `live-avatar` keeps only its `plugin/` facet. The `app/` facet is
  removed, and with it the dock entry, desktop icon and installed-app record.

### Key Entities

- **Voice output intent** — `voiceOutput`, persisted in `voice-config.json`.
- **Engine surface** — a URL a voice engine declares for its visual presence.
- **Presence lease** — short-lived, renewed server-side record proving a surface
  is rendering; the only thing that authorises sink routing.
- **Presence window** — hidden singleton built-in app hosting the surface.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: One click on the video button produces a connected face and spoken
  replies, with no other user action.
- **SC-002**: No click sequence yields a visible face with silent output, or
  `voiceOutput === "avatar"` with no window.
- **SC-003**: With the surface gone or broken, the next reply is audible in the
  browser and `voiceOutput` has fallen back to `"audio"`.
- **SC-004**: The presence window's aspect matches the video stream and its height
  is ~25% of the viewport.
- **SC-005**: A pinned window remains above unpinned windows across any focus
  sequence.
- **SC-006**: `live-avatar` appears in neither the dock nor the desktop grid, and
  the face still works.
- **SC-007**: After a page reload, no face is present and `voiceOutput` is at most
  `"audio"`.

## Notes

The lease in FR-013 is the load-bearing correction. The failure it prevents —
audio routed to an invisible sink — produced a silent assistant with no error, and
"remember to disconnect" is not a fix for it: liveness must be something the
renderer keeps proving, not something a socket implies.

Out of scope, deliberately: event-triggered presence (the email case). FR-008 and
FR-009 exist so that adding it later is a caller, not a redesign.
