# System Integrations

Use this guide for notifications, deep links, single-instance delivery, autostart, global shortcuts, clipboard, drag/drop, opener behavior, and restored window state.

## Notifications

Install and initialize the notification plugin, grant only its required permission, then:

1. check notification permission;
2. request it in response to a clear user action or onboarding explanation;
3. send only if granted;
4. handle denied and permanently denied states without repeated prompting.

Do not assume a tutorial's `sendNotification` call is sufficient. Notification actions are currently mobile-only, attachment support varies, and Android-style channels must exist before use. Treat action payloads as untrusted input and route them through the same validation/authorization as commands. Avoid secrets or sensitive message bodies on lock screens; choose channel visibility accordingly.

## Deep Links

Treat a deep link like hostile command-line/network input, not proof of identity.

- Prefer verified Android App Links and iOS Universal Links for web origins; keep custom schemes for controlled use cases.
- Validate scheme, host, path, parameters, length, encoding, and one-time authentication state in Rust.
- Never perform destructive or privileged behavior without user confirmation and application authorization.
- Handle both cold start (`getCurrent`/stored links) and warm delivery (`onOpenUrl`). Deduplicate and make processing idempotent.
- On Windows/Linux, combine with the single-instance plugin when the running process should receive new links. Register single-instance first.
- Statically configured desktop schemes are filtered, but users can fabricate arguments; runtime-registered schemes require manual argument validation.
- Desktop testing usually requires an installed app. Windows/Linux can register schemes for development; macOS requires the bundled app in Applications.

Universal/App Link association files depend on the production bundle ID, signing certificate/team, HTTPS host, content type, and platform validation. Verify those live before release.

## Single Instance

Register the single-instance plugin first. In its callback:

- parse and validate arguments and the incoming working directory;
- route files/deep links to one coordinator;
- restore/unminimize/show/focus an existing window defensively;
- queue input until the frontend is ready;
- avoid panicking when the expected window is absent.

Snap and Flatpak need explicit session-DBus own/talk permissions derived from the sanitized bundle identifier. Test packaged behavior; development mode is insufficient.

## Autostart

Autostart is a persistent user/system preference. Never enable it silently. Expose current state, let the user opt in/out, and handle OS policy failures. Keep launch arguments fixed and safe; distinguish an autostart/background launch from an interactive launch so the app does not unexpectedly steal focus.

Use desktop target dependencies and platform gates. Test upgrade, uninstall, app relocation, and multiple user accounts.

## Global Shortcuts

- Let the user choose or at least disable the shortcut.
- Detect registration conflicts and show actionable feedback.
- Handle pressed/released states deliberately; debounce repeated activation.
- Unregister on reconfiguration and shutdown.
- Route actions through native application services rather than simulating unsafe UI clicks.
- Test keyboard layouts, accessibility conflicts, remote desktops, and each OS.

## Clipboard

Clipboard read is sensitive and may trigger platform privacy behavior. Read only after a user action or an explicit product feature. Do not monitor or persist clipboard contents silently. Avoid logging copied values and clear secrets from app-owned clipboard flows when appropriate. Grant read and write separately if the current permission model supports it.

## Open Files and URLs Externally

Use the opener plugin instead of navigating the privileged WebView. Scope exact paths and URL patterns. Parse URLs before opening and prefer an allowlist of schemes and hosts; reject `javascript:`, unexpected custom protocols, credential-bearing URLs, and user-controlled application names. Reveal-in-folder and open-with operations still expose local information and require policy review.

## Native File Drag and Drop

Use `getCurrentWebview().onDragDropEvent` in the frontend or `Builder::on_webview_event` in Rust for native file drops. Handle enter, over, drop, and leave/cancel states and always unregister frontend listeners on teardown. The Rust event enum is non-exhaustive, so include a wildcard arm.

Dropped paths represent user-controlled input, not blanket trust. Apply the same canonicalization, type/size/content validation, symlink policy, duplicate handling, and confirmation rules as a file picker. Tauri may add dropped files/directories to runtime scopes, but the operation that consumes them still needs product authorization and validation. Never execute a dropped file based on its extension.

Native Tauri drag/drop is enabled by default. On Windows, disable it for that WebView only when the product instead needs HTML5 in-page drag/drop; test that tradeoff on every target. Provide an accessible non-drag alternative such as an Open button and keyboard-operable drop zone.

## Window-State Restoration

The window-state plugin restores after creation. Create restored windows hidden to avoid a visible jump, then let restoration reveal them. Validate restored geometry against current monitors/DPI so a removed display does not strand a window off-screen. Decide which flags apply to transient/dialog windows and avoid persisting sensitive screen state.

## Accessibility and Native Conventions

- Use semantic HTML, associated labels, meaningful names, correct roles, visible focus, logical tab order, and keyboard equivalents for every pointer/drag/tray-only action.
- Announce asynchronous command progress and errors without stealing focus. Respect reduced-motion, contrast, font scaling, zoom, and OS theme settings.
- Keep platform-standard menu roles and accelerators where possible; do not replace native text input, selection, clipboard, or window behavior without a tested need.
- Test with keyboard only, screen readers on each supported OS family, high contrast, 200% scaling, and narrow/mobile layouts. Automated accessibility checks are a floor, not proof.
- Design an offline state explicitly: distinguish local capability from network-backed actions, queue only operations with clear conflict/idempotency rules, show freshness, and never spin in reconnect loops.

## Authoritative Sources

- Notifications: <https://v2.tauri.app/plugin/notification/>
- Deep links: <https://v2.tauri.app/plugin/deep-linking/>
- Single instance: <https://v2.tauri.app/plugin/single-instance/>
- Autostart: <https://v2.tauri.app/plugin/autostart/>
- Global shortcuts: <https://v2.tauri.app/plugin/global-shortcut/>
- Clipboard: <https://v2.tauri.app/plugin/clipboard/>
- Opener: <https://v2.tauri.app/plugin/opener/>
- Window state: <https://v2.tauri.app/plugin/window-state/>
- WebView drag/drop API: <https://v2.tauri.app/reference/javascript/api/namespacewebview/#ondragdropevent>
