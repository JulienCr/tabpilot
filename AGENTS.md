# TabPilot — Agent Instructions

## Mission
Build TabPilot: a free, local-first Stream Deck plugin and Chromium Manifest V3 extension that remotely controls a **specific YouTube tab**, even when Chrome is not focused (for example, while the user is gaming).

Read `docs/specs.md` before implementing anything. It is the product/behavior source of truth. Keep it synchronized with implementation and explicitly document design changes.

## Engineering constraints
- TypeScript, strict mode, pnpm workspace. Never use npm or yarn.
- Prefer small, testable modules and reuse shared TypeScript contracts and runtime schemas.
- Stream Deck plugin: use the current official Elgato Stream Deck SDK and its supported action / Property Inspector APIs. Check official docs and version compatibility before choosing APIs.
- Browser: Chromium Manifest V3 (Chrome first), minimum permissions, content scripts scoped to YouTube. Respect MV3 service-worker suspension.
- Local communication only. No cloud service, analytics, telemetry, external API keys, or YouTube API dependency.
- Do not execute arbitrary scripts, navigate tabs, change Chrome focus, or use global shortcuts to implement playback.
- Security: validate every cross-process message (schema, origin, size, type), authenticate local connections, bind to loopback only, and avoid leaking session tokens in logs, URLs, or extension code exposed to websites.
- Do not assume a browser extension can directly open a WebSocket server. Choose and document a viable bridge, its lifecycle and authentication; test it end-to-end.
- Do not bypass browser user-activation or fullscreen restrictions. Report unsupported full-screen behavior honestly and provide a supported fallback.
- Use comments in English only. Avoid comments that simply restate the code.
- Prefer dependency-light code, DRY abstractions, explicit errors, graceful reconnect and teardown.
- Never install or copy closed-source assets or proprietary plugins.

## Expected repository structure (adapt if needed)
```text
apps/extension/             # Chrome MV3 extension
apps/streamdeck/            # Stream Deck plugin & Property Inspector
packages/protocol/          # Shared message schemas/types
docs/specs.md               # Product & technical specification
docs/architecture.md        # Implemented decisions and constraints
```
Use a root pnpm workspace, uniform lint/typecheck/test/build scripts and a practical README with Windows installation, unpacked extension loading, Stream Deck plugin deployment and development workflow.

## Build phases
1. **Scaffold** the workspace; document technical choices (especially the loopback transport and extension authentication) and write shared protocol schemas.
2. **Vertical slice:** one action controls play/pause on one explicitly chosen YouTube tab while Chrome is unfocused. Validate with an actual local end-to-end test or clearly label manual-only validation.
3. Implement tab discovery, persistent selection, reconnect, navigation/closed-tab handling and useful button status.
4. Add seek backward/forward (configurable), volume/mute and like/unlike with explicit status/disabled behavior.
5. Add dynamic Stream Deck key images/titles and Property Inspector tab selection. Keep controls composable in any Stream Deck profile.
6. Investigate fullscreen experimentally. Never present a fallback as native fullscreen. Add reliable tests and packaging.

## Definition of done
- `pnpm install`, `pnpm typecheck`, `pnpm lint`, `pnpm test`, `pnpm build` work (or explicitly document any missing tooling).
- Tests cover routing to the correct tab, message validation, disconnect/reconnect, tab closure/navigation and play/pause/seek.
- No command inadvertently targets another tab; fail visibly if target is missing/ambiguous.
- No Chrome foreground/focus requirement for supported actions.
- No unsupported claims about Like or native fullscreen.
- No secrets committed. Validate extension permissions and localhost security.
- Keep documentation and a short manual test matrix current.

## Agent workflow
- Inspect repo state and existing files before editing.
- Keep changes incremental and focused; avoid unrelated refactors.
- State assumptions and blockers. Verify claims against official documentation when possible.
- Run checks after each milestone, and report exactly which tests were run versus not run.
- If a behavior depends on a browser restriction or unstable YouTube DOM, make it an isolated adapter with a tested failure mode.
- Do not claim completion based only on compilation.
