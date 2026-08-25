# Feature Specification: Media Support in web_view (Images & Video)

**Feature Branch**: `web-view-media`

**Created**: 2026-08-25

**Status**: Draft

**App Target**: bos-core

**Input**: User description: "Add support for images (if not already supported) and video to web_view."

## Context

`web_view` is a BOS built-in tool that opens a sandboxed preview window (the hidden `html-viewer` app) for agent-produced content. It currently accepts `html` (a full HTML document, rendered via iframe `srcdoc`), `url` (an absolute or same-origin/VFS URL), and `filePath` (a VFS path rewritten to the raw-file route). The raw-file route (`/api/fs/raw`) already streams image and video bytes with correct `Content-Type` headers (e.g. `.png`, `.jpg`, `.webp`, `.svg`, `.mp4`, `.webm`, `.mov`, `.ogv`), and the preview is a plain `<iframe>`, so a media file pointed at directly already renders *somewhat* today via the browser's native media handlers.

What is missing is **first-class** support: the tool contract never advertises that media is possible (so the agent has no reason to do it), there is no media-mode styling (image centering / fit-to-frame, video player chrome), and there are no media playback options (autoplay, loop, muted, poster). This feature adds that first-class treatment across the tool handler, the tool declarations the model sees, and the built-in `html-viewer` renderer.

## User Scenarios & Testing

### User Story 1 - Preview an image file (Priority: P1)

An agent has produced or located an image (a generated chart, a screenshot, a mockup still) and wants to show it to the user. It calls `web_view` with the image's VFS `filePath` (or a `url` / inline `html` embedding an `<img>`). The preview window shows the image centered, scaled to fit the frame without cropping, on a neutral background — not a blank white page with the image shoved into the top-left corner.

**Why this priority**: Images are the most common media an agent produces (mockup screenshots, generated visuals, data plots). Making them look intentional is the core of "support for images."

**Independent Test**: Fully testable by calling `web_view` with a known image VFS path and confirming the window displays the image centered and fit-to-frame. This alone delivers a viable MVP.

**Acceptance Scenarios**:

1. **Given** an image exists at a VFS path, **When** the agent calls `web_view` with that `filePath`, **Then** the preview window displays the image centered and scaled to fit within the frame.
2. **Given** the agent calls `web_view` with any URL (external `http://` or `https://`, same-origin, or data URI) that resolves to an image, **When** the window opens, **Then** the image is displayed using the same centered, fit-to-frame media treatment.
3. **Given** the image is larger than the window, **When** the preview renders, **Then** the image is scaled down to fit (not clipped, not forcing a scrollable overflow on the outer window).

---

### User Story 2 - Preview a video file (Priority: P1)

An agent has produced or located a video and wants to play it back for the user. It calls `web_view` with the video's VFS `filePath` (or a `url` / inline `html` embedding a `<video>`). The preview window shows a video with native playback controls (play/pause, seek bar, volume, fullscreen), ready to play on user interaction.

**Why this priority**: Video support is the second half of the explicit request. A playable, controlled video element is what "support for video" means to the user.

**Independent Test**: Fully testable by calling `web_view` with a known video VFS path and confirming the window shows a controllable video player.

**Acceptance Scenarios**:

1. **Given** a video exists at a VFS path, **When** the agent calls `web_view` with that `filePath`, **Then** the preview window displays a video player with native controls.
2. **Given** a video preview is open, **When** the user clicks play, **Then** the video plays within the window.
4. **Given** the agent calls `web_view` with an external video URL (e.g. `https://cdn.example.com/clip.mp4` or `http://cdn.example.com/clip.mp4`), **When** the window opens, **Then** the video player loads and plays from that URL.
3. **Given** the video is larger than the window, **When** the preview renders, **Then** the player is scaled to fit the frame.

---

### User Story 3 - Media playback options (Priority: P2)

For videos the agent wants to behave a certain way, `web_view` accepts media options: autoplay, loop, muted, and a poster image. These are honored for video (and sensibly ignored where a format doesn't support them, e.g. poster for an image). Autoplay-with-audio is subject to browser autoplay policy (muted autoplay is permitted; unmuted autoplay is not).

**Why this priority**: Options refine the experience (e.g. a looping muted background clip, an autoplaying narrated clip) but are secondary to "the media just displays and plays."

**Independent Test**: Fully testable by calling `web_view` with a video path plus each option and observing the corresponding behavior (e.g. `muted` + `autoplay` starts playing immediately; `loop` restarts on end).

**Acceptance Scenarios**:

1. **Given** a video preview, **When** the agent passes `muted` and `autoplay`, **Then** the video begins playing automatically without user interaction.
2. **Given** a video preview, **When** the agent passes `loop`, **Then** playback restarts from the beginning when it reaches the end.
3. **Given** a video preview, **When** the agent passes a `poster` image path/URL, **Then** that image is shown in the player before playback starts.

---

### User Story 4 - Robust media loading, errors, and an honest tool contract (Priority: P2)

The agent is told, by the tool's own description, that it can preview images and video, so it will actually try. When a media target is missing, or is a format the browser cannot render (unsupported video codec, an extension not mapped to a media type), the preview surfaces a clear in-window error rather than a silent blank page, and the tool returns a failure message instead of falsely reporting success.

**Why this priority**: Because the tool contract now advertises media, bad inputs become more likely; without honest errors the agent and user are left guessing. This keeps the feature trustworthy.

**Independent Test**: Fully testable by (a) reading the `web_view` tool description and confirming it mentions image/video, (b) calling `web_view` with a nonexistent media path and confirming a clear error, and (c) calling it with an unsupported-media file and confirming a clear in-window error.

**Acceptance Scenarios**:

1. **Given** the `web_view` tool is listed for the agent, **When** the agent inspects its description, **Then** it states that images and video can be previewed.
2. **Given** the agent calls `web_view` with a VFS path that does not exist, **When** the tool runs, **Then** it returns an error (not a success) and the window shows a clear "not found / could not load" message.
3. **Given** the agent calls `web_view` with a file whose format the browser cannot render, **When** the window opens, **Then** it shows a clear in-window error message identifying the problem.

## Requirements

### Functional Requirements

- **FR-001**: The `web_view` tool contract (the description text the agent sees, and its parameter documentation) MUST state that images and video can be previewed, and MUST document the media-related parameters (`poster`, `autoplay`, `loop`, `muted`) introduced by this feature.
- **FR-002**: `web_view` MUST detect, from a VFS `filePath` or a `url` (any absolute or same-origin URL, including external `http://` and `https://` endpoints and data URIs), that the target is an image or a video and render it using a media presentation mode (not the default top-left, white-background document framing). Detection is based on the file extension or MIME type of the target.
- **FR-003**: In image mode, the preview MUST display the image centered in the frame and scaled to fit (fit-to-viewport, no cropping), on a neutral background.
- **FR-004**: In video mode, the preview MUST display the video with native playback controls and scaled to fit the frame.
- **FR-005**: `web_view` MUST accept a `poster` parameter (image path/URL) that sets the video player's poster frame.
- **FR-006**: `web_view` MUST accept `autoplay`, `loop`, and `muted` boolean parameters that map to the video element's corresponding attributes; `muted` + `autoplay` MUST be allowed to start without user interaction, while unmuted `autoplay` MUST respect browser autoplay restrictions.
- **FR-007**: When a media target does not exist or cannot be loaded, the tool MUST return a failure message (not report success), and the preview window MUST display a clear in-window error rather than a silent blank page.
- **FR-008**: Media preview MUST NOT grant previewed content access to BrowserOS APIs on the parent origin. The `sandbox="allow-scripts"` iframe boundary is preserved for **document** content (HTML/URL mode); media is rendered via native `<img>`/`<video>` elements, which carry no executable code and therefore satisfy the same security property without an iframe (see design.md ADR-1).
- **FR-009**: The `update=true` behavior (reuse/refresh the existing preview window) MUST work for media targets as it does for HTML.

### Non-Functional Requirements

- **NFR-001**: **Consistency** — media presentation MUST match the existing preview window chrome (title bar, close, fullscreen) and theme.
- **NFR-002**: **Performance** — image preview for files under 5 MB MUST load and render promptly; large media MUST stream rather than buffer the entire file in memory on the client where the underlying route supports streaming.
- **NFR-003**: **Responsiveness** — the media frame MUST adapt to window resizing, re-centering/re-fitting the media.
- **NFR-004**: **Backward compatibility** — existing `html`, `url`, and `filePath`-to-HTML usage MUST be unchanged; media detection is additive and MUST NOT alter how an HTML document is rendered.

## Edge Cases

- **Format not in the MIME map** (e.g. an unusual video extension served as `application/octet-stream`): the browser cannot render it; the preview MUST show a clear in-window error rather than a blank page.
- **Unsupported video codec** (file is a valid container but the browser lacks the decoder): the player MUST surface a load/decode error clearly.
- **Inline media in `html`**: when the agent embeds an `<img>` or `<video>` inside an `html` document, it is rendered as HTML (existing behavior); the media presentation mode applies to direct image/video targets, not to embedded HTML.
- **Inline data URIs**: an image or video provided as a `url`/`filePath` that is a data URI MUST render like any other media target.
- **Virtual media endpoints** (e.g. `http://server:8188/view?filename=clip.mp4&...`): the URL path has no media extension; the classifier MUST fall back to the query string to detect the media type, and the window title MUST default to the filename from the query param (not the endpoint path).
- **Very large video**: the player MUST be usable (seek/play) via streaming; MUST NOT hang the window during initial load.
- **Autoplay with audio**: MUST NOT be forced against browser policy; unmuted autoplay MAY be blocked by the browser and that is acceptable behavior (player simply waits for interaction).
- **Image/video target + `update=true`**: MUST refresh the existing preview window to the new media, reusing the same window lifecycle.
- **A `.svg` file**: treated as an image and rendered in image mode.

## Assumptions

- **A-1**: Media is previewed by pointing the existing sandboxed preview iframe at the raw-file route (which already streams correct media `Content-Type`s); no new server media-serving route is required.
- **A-2**: The set of "image" and "video" extensions is derived from the raw-file route's MIME map (images: png, jpg, jpeg, gif, webp, svg, avif; videos: mp4, ogv, webm, mov) plus sensible additions (e.g. `.m4v`, `.avi`); anything unmapped is treated as non-media and falls back to current behavior. For external URLs (not served by the raw-file route), detection relies on the URL path's file extension. When the path has no media extension (e.g. a virtual endpoint like `/view`), the classifier falls back to inspecting the query string: it checks common filename-bearing parameter names (`filename`, `file`, `path`, `name`) and any query parameter whose value ends in a known media extension, using that value's extension for classification.
- **A-3**: Media options are additive parameters with sensible defaults (no autoplay, controls shown, not muted, no poster) when omitted.
- **A-4**: This is a `bos-core` change spanning the `web_view` tool handler, the tool declarations the model sees (capabilities registry / frontend declarations / OS actions), and the built-in `html-viewer` app renderer.
- **A-5**: Browser-native media codecs (what Chromium supports) are the supported set; server-side transcoding is out of scope.
- **A-6**: The existing raw-file route's branch-scoping and its client-side "verify the target resolves" check apply to media paths unchanged.

## Success Criteria

- **SC-001**: An agent can preview an image by VFS path in a single `web_view` call and the window shows it centered and fit-to-frame.
- **SC-002**: An agent can preview a video by VFS path in a single `web_view` call and the user can play, seek, and fullscreen it from the window.
- **SC-003**: A developer reading the `web_view` tool description knows that images and video are supported and sees the `poster`/`autoplay`/`loop`/`muted` parameters documented.
- **SC-004**: A missing or unrenderable media target produces a clear in-window error and a non-success tool return in 100% of such cases (no silent blank pages).
- **SC-005**: Existing HTML-preview behavior (documents, mockups, branch-scoped `/Specs` paths) is unchanged after the feature ships.

## Key Entities

- **Media Target**: an image or video identified by a VFS `filePath` or a `url` (including data URIs); has a type (image | video) inferred from its extension/MIME and a rendered representation (image element or video player).
- **Media Options**: a set of preview parameters applied to a media target — `poster`, `autoplay`, `loop`, `muted`.
- **Preview Window**: the existing `html-viewer` window, extended with a media presentation mode (image fit-to-frame, video player) in addition to its current HTML/URL modes.
