# TabPilot — Product and Technical Specification

**Status:** Initial specification / agent implementation brief  
**Target:** Windows 11, Google Chrome, Elgato Stream Deck XL (32 keys)  
**Language/tooling:** TypeScript, pnpm, Chrome Manifest V3, official Stream Deck SDK

## 1. Purpose

Control an explicitly selected YouTube video tab from an Elgato Stream Deck **without requiring Chrome to be foregrounded**. The user may be playing a full-screen game while several different YouTube tabs remain open. Playback commands must reach the chosen tab only.

TabPilot must be free, local-first, privacy-conscious and extensible to other Chromium media sites later. YouTube is the only required provider for V1.

## 2. User experience

- Stream Deck buttons are **independent actions**, not a forced 32-key layout; users may place them in existing profiles.
- A **global target** is shared across TabPilot actions by default (per Stream Deck plugin instance). Changing it updates all bound playback controls.
- A **Select Video** action allows cycling through discovered YouTube tabs, with title or thumbnail when feasible; the selected target is visibly indicated.
- A Property Inspector dropdown lists open supported tabs by page/video title, allowing deliberate selection. If browser/extension is disconnected, show a meaningful disconnected state.
- Optional fixed-tab mode lets a particular action always target a chosen tab rather than the global target.
- Commands must not automatically switch the active Windows application or bring Chrome to the front.
- On missing/closed target, show 'no target' rather than silently falling back to the active tab.
- Several YouTube tabs, including paused tabs, must remain independently addressable.

## 3. Feature scope

| Feature | Priority | Behavior |
|---|---|---|
| Play / pause toggle | P0 | Toggle target's video; display playing/paused state |
| Seek backward / forward | P0 | Configurable steps, default 10 seconds; clamp to valid range |
| Tab discovery and target selection | P0 | Enumerate YouTube tabs, stable selection while tab exists |
| Background control | P0 | No Chrome focus required for supported operations |
| Connection/target status | P0 | Handle startup, extension disconnected, stale/closed tabs |
| Volume up/down, mute toggle | P1 | Affect only selected HTMLMediaElement |
| Like/unlike | P1 | Toggle target video's like using YouTube UI when available; no YouTube account APIs |
| Key label/icon/progress state | P1 | Dynamic key title/icon and status with conservative update rate |
| Theater mode | P2 | YouTube UI theater toggle if reliably supported |
| Native fullscreen toggle | Research | Browser user-activation/visibility requirements may prevent background-tab fullscreen. Do not promise it. |
| Multi-site support | Future | Provider adapter architecture; not required for V1 |

### Like constraints

Like depends on YouTube's logged-in UI, which changes over time. Interact only with the selected page, and prefer accessible labels/semantic state over brittle CSS. Determine whether liked/unliked from actual UI state and verify result. If the user is signed out, the control is unavailable, or a consent/login dialog appears, return an explicit failure. Never click an unknown button.

### Fullscreen constraints

`Element.requestFullscreen()` generally requires transient user activation and the browser may reject it on a background tab. A Stream Deck press communicated through an extension does **not** automatically constitute a trusted page user gesture. Do not fake native fullscreen by maximizing browser windows or focus-stealing. Detect capability; if impossible, clearly mark unavailable. Theater mode can be a separate action.

## 4. Architecture

```text
Stream Deck XL
  -> Stream Deck plugin (TypeScript, official SDK)
     -> authenticated local transport (loopback)
        -> Chrome extension (Manifest V3)
           -> service worker router
              -> content script in selected youtube.com tab
                 -> provider-specific YouTube controller
```

Recommended monorepo:
- `apps/streamdeck`: plugin actions, state, Property Inspector and connection endpoint.
- `apps/extension`: MV3 manifest, background service worker, tab inventory, content script.
- `packages/protocol`: shared runtime-validated messages and TypeScript types.

The agent must choose a viable local bridge and document the trade-offs. A loopback WebSocket server hosted by the persistent Stream Deck plugin, with an extension-initiated WebSocket connection, is a candidate; validate that the chosen runtime, packaging, MV3 lifecycle and permissions really support it. An MV3 service worker may be suspended; implement reconnect and re-sync. Avoid reliance on undocumented privileged browser APIs. Do not require a standalone third daemon unless justified.

### Transport/security requirements
- Bind bridge to `127.0.0.1` (or explicitly validated loopback interface), never `0.0.0.0`.
- Authenticate the extension connection using a locally provisioned secret or other verified mechanism; **do not** treat localhost or the WebSocket Origin header as sufficient authentication.
- Document secure pairing/provisioning (avoid baking a shared public secret into both distribution artifacts).
- Validate incoming JSON with shared versioned schemas, bounded payload sizes, allowlisted commands and request IDs.
- Rate-limit commands and status events to protect the plugin and browser.
- Do not expose a generic browser scripting endpoint or a remotely controllable debugging port.
- No direct page access to the privileged bridge; content scripts relay narrowly scoped messages via extension messaging.
- Least-privilege host permissions and CSP; no remote code or eval.

## 5. State and identity

Each discovered tab is represented by a session-scoped Chrome `tabId`, window ID, page title, current video ID/URL, media state and a last-seen timestamp. The browser extension owns the authoritative tab inventory.

- Tab IDs are only meaningful within a browser session; do not treat them as persistent durable identifiers.
- Persist optional user target preference as a hint, not proof of tab identity; after browser restart require safe reconciliation or explicit reselection.
- When an existing tab navigates to a different YouTube video, update its metadata and follow the same **tab** by default. Expose an optional strict-video lock for future use.
- When the tab closes or navigates away from supported content, invalidate the target.
- If multiple YouTube frames/videos exist, explicitly choose the primary player and reject ambiguity.
- Associate commands with a request ID; return the actual target tab ID, result/error and updated state. Discard stale or out-of-order responses.
- A newly connected plugin receives a fresh snapshot before actions become enabled.

## 6. Suggested protocol

Version every message (`v: 1`) and validate it at runtime. These are conceptual examples, not fixed wire contracts.

- `hello` / `helloAck`: handshake and protocol compatibility
- `tabs.snapshot` / `tabs.changed`: inventory and state events
- `target.select`: set global target by current tab identity
- `player.command`: `playPause`, `seekBy`, `setVolume`, `muteToggle`, `likeToggle`, optionally `theaterToggle`
- `player.result`: correlated success/error + state
- `heartbeat`: optional bounded health check

Use explicit error codes, e.g. `EXTENSION_DISCONNECTED`, `TARGET_NOT_FOUND`, `UNSUPPORTED_PAGE`, `PLAYER_NOT_READY`, `NOT_AUTHENTICATED`, `LIKE_UNAVAILABLE`, `FULLSCREEN_RESTRICTED`, `TIMEOUT`.

## 7. Stream Deck action design

1. **Select target:** cycle next/previous or display a configurable tab; label with video title and selection status.
2. **Play/pause:** toggle with dynamic icon/state.
3. **Seek:** reusable action with signed offset and configurable number of seconds.
4. **Volume:** reusable +/- step with mute action.
5. **Like:** toggle with liked/unliked/unknown feedback.
6. **Theater/fullscreen:** only if proven reliable; keep separate actions.

Use the official Property Inspector for selectable target, seek interval, action-specific settings and connection diagnostics. Never require all 32 XL keys, even though XL is the primary test device. Do not assume every button can show long labels clearly.

## 8. Acceptance scenarios

1. With two YouTube tabs A and B playing different videos, select A. Press Play/Pause while a game is focused: A toggles, B is unchanged, game keeps focus.
2. Switch target to B using Stream Deck; seek backward 10 s: only B changes.
3. Close B. Subsequent commands show no target and never control A implicitly.
4. Navigate A to a new YouTube video: metadata refreshes and tab lock continues following A.
5. Disconnect and reconnect extension: tab list and selected target resynchronize without duplicated commands.
6. Browser restarts: prior numeric tab IDs are not blindly reused; no misrouting.
7. Like works on an authenticated YouTube video or reports a meaningful unavailable/error state.
8. Native fullscreen is tested from a background tab and labeled unsupported when browser restrictions prevent it.
9. Malformed/oversized/unauthenticated bridge messages are rejected without executing actions.
10. Plugin/exension installation and operation work on Windows 11 with a Stream Deck XL; Mac compatibility is best-effort, not a V1 release gate.

## 9. Testing and deliverables

- Unit tests: protocol validation, tab selection/state transitions, seek clamping, stale-message handling.
- Integration tests: plugin ↔ bridge ↔ extension routing (mock browser and SDK where necessary).
- Manual E2E matrix: Chrome foreground/background, multiple YouTube tabs, game focused, closed/navigated tabs, Chrome restarted, YouTube signed in/out.
- Scripts: `pnpm lint`, `pnpm typecheck`, `pnpm test`, `pnpm build`.
- README: prerequisites, pnpm install/build, unpacked Chrome extension installation, Stream Deck plugin sideload/debugging, pairing/connection troubleshooting.
- `docs/architecture.md`: exact transport, pairing, permissions, known Chrome/YouTube limitations, verified SDK API versions.
- No runtime telemetry or tracking.

## 10. Explicit non-goals for V1

- Automatic control of the focused window, global media key interception, or foreground tab shortcuts.
- YouTube Data API/OAuth, subscription or playlist management, downloading media or DRM bypass.
- A standalone cloud service or paid service.
- Guaranteed native fullscreen from unfocused Chrome tabs.
- Supporting Netflix, Twitch, Spotify or every HTML5 player on the first release.

## 11. Agent starting task

Implement the smallest **working vertical slice** first: a genuine Stream Deck Play/Pause action targeting a user-selected YouTube tab through a secure local extension bridge, without Chrome focus. Prove the routing and failure behavior before adding features. Keep an explicit list of tested behaviors and browser-imposed limitations.
