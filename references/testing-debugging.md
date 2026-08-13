# Testing and Platform Debugging

Use this reference to choose the smallest test that proves the behavior and to diagnose failures across the frontend, IPC boundary, Rust core, WebView, and bundle.

## Test Pyramid

1. **Pure Rust unit tests:** domain rules, parsers, path/policy decisions, migrations, retry state machines, and adapters behind traits.
2. **Rust integration tests:** repositories, real temporary databases/files, sidecar protocol fixtures, and Tauri code under its mock runtime where useful.
3. **Frontend unit/component tests:** UI state and typed native-client wrappers with IPC mocked.
4. **Contract tests:** the same command names, camelCase argument keys, DTO shapes, error codes, and event/channel payloads on both sides.
5. **Real-app end-to-end tests:** launch the bundled/development application and exercise actual WebView plus Rust behavior.
6. **Platform smoke tests:** install, launch, update, uninstall, deep links, files, tray, permissions, signing, and OS lifecycle on every supported target.

Do not claim native behavior was tested when only JavaScript mocks ran.

## Frontend Tauri Mocks

`@tauri-apps/api/mocks` can intercept `invoke`, simulate window labels, and partially mock events.

- Install the mock before importing modules that capture Tauri globals.
- Provide WebCrypto in DOM test environments when the API expects it.
- Assert command name, argument object, call count, result mapping, and typed error behavior.
- For shell/sidecar mocks, emit stdout/stderr and the terminating event; test malformed and missing termination too.
- Event mocking is partial: do not assume every targeted emission API is supported.
- `mockWindows` fakes labels, not window properties; intercept property calls separately.
- Always call `clearMocks()` after each test to prevent cross-test state leakage.

## Rust and Mock Runtime

Keep commands thin so most logic accepts ordinary dependencies and needs no Tauri runtime. For code that genuinely uses app handles/windows, use Tauri's mock runtime and generated mock context, then cover setup/command behavior without native WebView libraries. Avoid assertions tied to undocumented runtime internals.

## End-to-End

Current official docs distinguish two routes:

- WebdriverIO's Tauri desktop service supports Windows, Linux, and macOS. Its current default can use a test-only embedded WebDriver server, including on macOS; other providers use the platform driver or a third-party cross-platform driver.
- Direct desktop WebDriver can be driven on Windows and Linux; macOS has no equivalent general desktop WebDriver client in that direct route.

Verify the current platform table before creating a CI promise. E2E tests should use stable accessibility-oriented selectors, isolated app data, deterministic fixtures, bounded waits, screenshots/logs on failure, and a fresh process for lifecycle-sensitive cases. Run at least one packaged-build smoke test because development URLs, capabilities, resources, and signing differ from `tauri dev`.

The richer WebdriverIO route adds plugins that can expose script execution, invoke mocking, log forwarding, or an in-process HTTP driver. Put those dependencies, registration, global-Tauri setting, permissions, frontend import, and driver port behind an explicit E2E feature/config; prove release/store builds omit them. A browser-only WDIO mode is useful for fast mocked renderer tests but is not a native-app test.

Keep browser-engine coverage separate from native-runtime coverage. Playwright WebKit is valuable for WebKit-sensitive layout and JavaScript behavior, but it is not the same binary, process model, permission system, IPC bridge, or lifecycle as macOS/iOS WKWebView. Conversely, a single real-app test on one OS does not cover Chromium/WebView2 or WebKitGTK behavior elsewhere.

Test asynchronous subscription teardown explicitly. If a component unmounts while an awaited `listen`/deep-link registration is still pending, call the eventual unlisten handle as soon as it resolves; a cleanup flag alone does not unregister it. Apply the same rule to watchers, channels, sockets, and native permission callbacks.

## Debugging Ladder

1. Reproduce with exact OS/architecture, WebView version, Tauri/CLI/plugin versions, feature flags, and dev vs packaged mode.
2. Read the frontend build/dev-server output and Rust console. Retry with `RUST_BACKTRACE=1` (or the PowerShell equivalent).
3. Inspect the platform WebView console/network panel: WebKitGTK on Linux, Safari Web Inspector on macOS, and Edge DevTools on Windows.
4. Add structured, redacted correlation IDs spanning frontend request, command, service, and sidecar/network operation.
5. Reduce to one command/window/plugin and test its generated capability schema.
6. Attach LLDB/GDB or the Windows debugger to the Rust core. When launching via Cargo rather than the Tauri CLI, start/build the frontend separately because CLI hooks will not run.
7. Reproduce from a real installed bundle with a clean profile.

Keep devtools behind `cfg(debug_assertions)` unless a deliberate support build is required. Enabling the Tauri `devtools` feature in production uses private macOS APIs and prevents App Store acceptance.

## Platform-Specific Triage

### Windows

- record OS version, WebView2 runtime version/distribution mode, installer type, and code-signing result;
- inspect Event Viewer, installer logs, missing VC/runtime/native DLLs, filesystem virtualization, and antivirus quarantine;
- distinguish MSIX/Store restrictions from MSI/NSIS behavior.

### macOS

- inspect Console logs, crash reports, codesign details, entitlements, hardened runtime, quarantine, notarization, and stapling;
- reproduce on Intel and Apple Silicon when supported;
- use Safari's Develop menu for WebView inspection; test App Sandbox separately from direct distribution.

### Linux

- record distribution, desktop/session (X11/Wayland), WebKitGTK/GTK versions, GPU/driver, package format, and sandbox;
- launch from a terminal and inspect missing shared libraries and portal/DBus integration;
- for NVIDIA/Wayland blank/flickering/crash problems, test official workarounds in order and ship an override only after confirming affected hardware. WebGL may silently use a slow path, so provide a non-WebGL fallback for graphics-heavy views.

### Android

- use `adb logcat`, Android Studio, a physical device, and emulator; distinguish Rust, Kotlin plugin, WebView, permission, activity, and deep-link lifecycle failures;
- test cold/warm resume, rotation, background/foreground, process death, network changes, and API-level differences.

### iOS

- use Xcode console/debugger, device logs, Safari Web Inspector, and both simulator and physical device;
- inspect provisioning, entitlements, Info.plist usage descriptions, scenes, universal/deep links, background transitions, memory warnings, and App Transport Security.
- for physical-device HMR, verify the CLI-selected development host, dev-server/HMR bindings, local-network reachability, and capability remote URL rules; expose a dev server to the LAN only on a trusted network and never carry broad development origins into production.
- test safe-area insets, the on-screen keyboard and visual viewport, touch/coarse-pointer layouts, input focus zoom, rotation, reduced motion, dynamic text, and pinch-zoom accessibility on the actual supported iOS versions.

## Failure Artifact Checklist

Capture sanitized logs, command correlation IDs, test seed/fixture, screenshots, crash trace, platform/WebView versions, dependency lockfiles, config/capability files, installer/update channel, and exact reproduction steps. Never collect secrets or arbitrary user documents. Retention and opt-in support upload must be explicit.
