# Tauri/Rust Coverage Map

Use this map to route uncommon tasks and identify missing expertise. A checked item means the skill contains a workflow or routed reference, not that every platform/version combination is timelessly encoded.

## Foundation

- [x] repository orientation and instruction discovery
- [x] prerequisites, scaffolding, project structure, package-manager preservation
- [x] Tauri/Rust/frontend version and feature inspection
- [x] Tauri v1-to-v2 and configuration migration strategy
- [x] framework-specific integration pitfalls for React, Vue, Svelte, Solid, Angular, Next.js, Nuxt, and Rust/WASM frontends

## Native Boundary and Architecture

- [x] typed commands and invoke wrappers
- [x] events, channels, managed state, async tasks, cancellation, and cleanup
- [x] domain/service/adapter separation and multi-binary Rust cores
- [x] multi-window and multi-WebView lifecycle patterns
- [x] tray, menus, window state, and startup/splash coordination
- [x] single-instance, deep-link, autostart, and global-shortcut coordination
- [x] native dialogs, notifications, clipboard, opener, and window-state restoration
- [x] drag/drop
- [x] mobile file associations and cold/warm delivery

## Data and Integrations

- [x] incremental Python-to-Rust migration, PyO3/maturin packaging, semantic parity, and rollback
- [x] NumPy/Arrow/Polars/DataFusion boundaries and Rust-backed data-science pipelines
- [x] Python sidecar-to-native Tauri migration and permission reduction
- [x] filesystem, dialogs, watches, path safety, handles, and persisted scopes
- [x] SQLite/database ownership, migrations, transactions, and upgrade safety
- [x] settings stores and Stronghold-style secret-vault strategy
- [x] OS keychain integrations
- [x] HTTP/WebSocket networking, downloads, retries, proxies, and TLS
- [x] sidecars, child processes, supervision, and local IPC choices
- [x] embedded servers, FFI, and shared Rust libraries
- [x] plugin use, plugin authoring, permissions, scopes, and mobile plugin bridges

## Security

- [x] trust boundaries, input validation, safe errors, and secret handling
- [x] capabilities, permissions, scopes, CSP, remote content, and navigation
- [x] sidecar/plugin privilege review
- [x] updater signing and supply-chain hardening
- [x] threat modeling, audit tooling, dependency policy, and incident-safe logging

## Quality and Operations

- [x] idiomatic Rust ownership, errors, async safety, serialization, tests, and lints
- [x] layered frontend/Rust/Tauri verification
- [x] mocked Tauri runtime and current WebDriver end-to-end recipes
- [x] debugging on Windows, macOS, Linux, Android, and iOS
- [x] performance, polling/IPC batching, streaming, startup, memory, and bundle-size profiling
- [x] accessibility, keyboard behavior, native conventions, and offline UX

## Delivery

- [x] CI matrices and GitHub Actions release pipelines
- [x] updater architecture, channels, rollback, and migration compatibility
- [x] Windows installers/signing/Store delivery
- [x] macOS bundles, signing, notarization, entitlements, and App Store delivery
- [x] Linux AppImage, Debian, RPM, Flatpak, Snap, and AUR considerations
- [x] Android/iOS architecture, lifecycle, windows, associations, and device-test strategy
- [x] Android/iOS signing and store delivery

## Research Rule

Before filling an unchecked area, prefer this source order:

1. current official Tauri documentation and generated project schemas;
2. current official plugin documentation and source examples;
3. maintained production repositories and CI workflows;
4. full written tutorials or full video transcripts;
5. community reports for failure modes and operational caveats.

Record Tauri tutorials in [source-ledger.md](source-ledger.md) and Python migration sources in [python-migration-source-ledger.md](python-migration-source-ledger.md), including sources that are rejected as stale, duplicated, inaccessible, or unsafe. Update this map only when the synthesized skill actually gains the coverage.
