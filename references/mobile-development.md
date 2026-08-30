# Mobile Development

Use this guide for Tauri applications targeting Android or iOS. Mobile APIs and store requirements change rapidly; verify the current Tauri, Android, and Apple documentation before implementation.

Only when the request explicitly includes an immersive Quest/OpenXR application, also read [vr-xr-development.md](vr-xr-development.md). Tauri mobile support does not make a WebView the correct owner of an immersive Activity, scene, frame loop, controller input, or headset lifecycle.

## Architecture and Setup

- Keep `run` in a reusable library entry point and preserve the generated mobile entry-point attribute.
- Put Android Kotlin and iOS Swift/native bridge code behind platform-specific modules.
- Validate every Rust dependency against the actual mobile target; desktop support does not imply Android/iOS compatibility.
- Use Tauri path APIs and mobile-safe storage locations instead of assuming desktop home/config paths.
- Keep capabilities platform-specific where authority is only needed on one mobile OS.
- Test on physical devices early for filesystem, notifications, biometrics, camera, backgrounding, deep links, and performance.
- Treat the generated Xcode/Gradle projects as build-system outputs until the current Tauri guide identifies a supported customization point; document every native edit and verify regeneration does not erase it.
- Keep development URLs and HMR hosts confined to trusted networks. A phone cannot reach a laptop-only loopback server, but broad LAN binding also exposes the development surface to peers.

Do not edit generated platform projects blindly. Determine which files are designed for persistent customization and whether a Tauri regeneration step will overwrite them.

## Multi-Window

Current Tauri supports mobile multi-window with different native models:

- Android uses activities and Activity Embedding for side-by-side layouts on large screens. Additional window types need registered `TauriActivity` subclasses and split rules.
- iOS uses `UIScene`; concurrent user-visible windows are primarily an iPad/Stage Manager experience.
- Phone-sized devices may push/replace UI instead of showing simultaneous windows.

Check support at runtime. Official current guidance identifies Android 12L/API 32+ and iOS 13+ as the relevant baseline, but verify this against the pinned version.

- Grant the create-WebView-window permission only to the initiating capability.
- Bind dynamic labels with a deliberate pattern such as `main-*`, not a global wildcard.
- On Android, preserve the creator/activity relationship so split rules use the intended task stack.
- On iOS, preserve requesting scene identity when creating related scenes.
- Use per-window routes and lifecycle cleanup; do not assume one global foreground window.

## File Associations

Declare associations in Tauri bundle configuration so platform metadata is generated consistently. Specify extensions, MIME types where required, application role/rank, and a reverse-DNS exported type for app-owned custom formats.

Handle both delivery modes:

1. cold start: store opened URLs in managed state until the frontend is ready and expose a draining or acknowledged command;
2. warm app: emit a typed event to active consumers.

Deduplicate repeated delivery, validate URL schemes and file types, canonicalize paths where applicable, enforce file-size limits, and never process a file merely because the OS associated it with the app. Decide whether initial items are returned repeatedly or consumed exactly once.

## Native Plugins and Bridges

- Start with an official plugin when it supports the required platforms.
- For a custom mobile plugin, keep the Rust command contract platform-neutral and implement Android/iOS adapters behind it.
- Map native errors into stable Rust/plugin errors; do not expose stack traces to the WebView.
- Request OS permissions at the point of need and distinguish Tauri capability permission from Android/iOS runtime permission.
- Test denied, permanently denied, interrupted, backgrounded, and restored flows.
- For custom plugins, use the generated plugin scaffold and current Kotlin/Swift command conventions. Optional native arguments need explicit nullable/default handling; shared Rust invoked from Kotlin/Swift requires an audited JNI/FFI boundary, target-specific linking, panic containment, and device tests.

For a frontend that also runs on the web, isolate native functionality behind a typed adapter. Platform detection is a usability branch, not an authorization boundary; the backend capability and OS permission still govern the call. Lazy-loading an optional guest package can keep browser builds clean, but verify the bundler actually splits it and never swallow failures that represent required product behavior.

When Gradle builds and packages a Rust `cdylib`, declare the root `Cargo.toml`, `Cargo.lock`, leaf crate, and every shared path crate as task inputs. Otherwise a shared-core change can leave an incremental build packaging a stale `.so`. A `skipRustBuild` or fallback flag must also exclude the generated JNI library directory; skipping only the producer task can package bytes from an earlier build.

## Lifecycle

Mobile apps are suspended, resumed, rotated, memory-pressured, and sometimes terminated without a desktop-style quit path. Persist critical work incrementally, make commands idempotent where retries are possible, reconnect listeners after recreation, and cancel or checkpoint long tasks when background rules require it.

Design the WebView UI for native constraints: safe-area insets, software-keyboard/visual-viewport changes, touch targets, coarse pointers, reduced motion, dynamic text, screen readers, and input focus zoom. Do not disable user zoom as a shortcut; accessibility obligations still apply inside an app WebView. Treat service-worker, cookie, media, and storage behavior as WebView- and OS-version-specific and test it rather than inheriting browser assumptions.

## Verification

- compile and run on each target architecture used for release;
- test simulator/emulator and at least one physical device when hardware or lifecycle matters;
- exercise cold/warm file opens, deep links, permission denial, rotation, background/foreground, and process restoration;
- verify generated manifests/plists, capability files, icons, bundle IDs, signing settings, and store privacy declarations;
- inspect APK/AAB ABI entries, merged manifest, cleartext policy, permissions, and native symbols rather than treating `assemble` success as sufficient;
- test any native-build bypass by seeding a prior `.so`, rebuilding with the bypass and forced task execution, and proving the stale library is absent from the artifact;
- state the exact untested device/OS matrix.

## Authoritative Sources

- Mobile multi-window: <https://v2.tauri.app/learn/mobile-multiwindow/>
- Mobile file associations: <https://v2.tauri.app/learn/mobile-file-associations/>
- Mobile plugin development: <https://v2.tauri.app/develop/plugins/develop-mobile/>
- Android prerequisites: <https://v2.tauri.app/start/prerequisites/#android>
- iOS prerequisites: <https://v2.tauri.app/start/prerequisites/#ios>
