---
name: tauri-rust-developer
description: Build, extend, debug, review, test, and prepare Tauri v2 applications with idiomatic Rust backends and web frontends. Use for work involving src-tauri, Cargo or Rust code, tauri.conf.json, commands and invoke IPC, events or channels, managed state, async tasks, capabilities, permissions, scopes, CSP, plugins, sidecars, Python-to-Rust migration, desktop or mobile builds, packaging, Tauri migrations, or Rust quality in a Tauri project.
---

# Tauri + Rust Developer

Build production-minded Tauri applications as one coherent system: a least-privilege native core, a typed IPC boundary, and a maintainable web frontend. Adapt to the repository instead of imposing a template.

## Operating Contract

1. Inspect the repository, its instructions, and its current toolchain before proposing changes.
2. Preserve the selected frontend framework, package manager, Rust edition, Tauri major version, and local conventions unless the task requires a migration.
3. Treat WebView content and all IPC input as lower-trust than the Rust core. Validate on the Rust side and grant only the capabilities required.
4. Prefer the smallest vertical slice that proves the native-to-frontend path, then extend it incrementally.
5. Run verification proportional to the change and distinguish pre-existing failures from regressions.
6. Verify current official documentation before relying on volatile Tauri APIs, mobile behavior, WebDriver tooling, bundling, signing, or store requirements.

## Route the Work

Read only the references needed for the task:

- Read [project-workflow.md](references/project-workflow.md) when scaffolding, migrating, orienting in a repository, choosing dependencies, or planning a cross-layer change.
- Read [tauri-architecture.md](references/tauri-architecture.md) when designing commands, events, channels, state, async work, frontend wrappers, or module boundaries.
- Read [rust-quality.md](references/rust-quality.md) when implementing or reviewing Rust, errors, ownership, async code, serialization, logging, or tests.
- Read [tauri-security.md](references/tauri-security.md) for every change that adds IPC, plugins, filesystem, shell, HTTP, remote content, new windows, permissions, capabilities, scopes, or CSP changes.
- Read [desktop-integration.md](references/desktop-integration.md) for windows, WebViews, title bars, splash screens, tray icons, native menus, close/hide behavior, and desktop lifecycle work.
- Read [mobile-development.md](references/mobile-development.md) for Android/iOS setup, mobile windows, file associations, native bridges, device testing, and store-oriented constraints.
- Read [sidecars-and-processes.md](references/sidecars-and-processes.md) for external binaries, Node/Python helpers, child-process IPC, artifact naming, permissions, supervision, and shutdown.
- Read [python-to-rust-migration.md](references/python-to-rust-migration.md) when replacing Python modules, services, sidecars, numerical kernels, validation/parsing code, pandas/NumPy workloads, or data pipelines with Rust or a Rust-backed Python library.
- Read [plugin-development.md](references/plugin-development.md) when selecting, installing, authoring, permissioning, testing, or publishing a Tauri plugin.
- Read [files-and-persistence.md](references/files-and-persistence.md) for dialogs, filesystem access, watched paths, settings stores, SQLite/databases, migrations, and secret storage.
- Read [system-integrations.md](references/system-integrations.md) for notifications, deep links, single-instance behavior, autostart, global shortcuts, clipboard, external opening, and restored window state.
- Read [performance-and-production.md](references/performance-and-production.md) for polling, IPC batching, streaming, caches, startup, resource limits, large data, profiling, and production-readiness reviews.
- Read [latency-critical-systems.md](references/latency-critical-systems.md) when lowest possible latency, low jitter, deterministic timing, high-rate acquisition, multiple live devices, clock alignment, deadline-sensitive output, or scoped OS scheduling are central requirements.
- Read [marionette-remote-control.md](references/marionette-remote-control.md) when an external browser or phone controls a Tauri app, or for remote CLI actions, companion controls, BRSP/VDO.Ninja data-only sync, peer-to-peer WebRTC control, or shared Party scenes.
- Read [frontend-frameworks.md](references/frontend-frameworks.md) for static-output/SSR decisions, React/Vue/Svelte/Solid/Angular, meta-frameworks, routing, HMR, and Rust/WASM frontends.
- Read [networking.md](references/networking.md) for HTTP, WebSockets, transfers, OAuth, local servers, FFI, or shared native libraries.
- Read [testing-debugging.md](references/testing-debugging.md) for mocks, end-to-end tests, debuggers, failure artifacts, and platform-specific diagnosis.
- Read [release-distribution.md](references/release-distribution.md) for CI matrices, installers, signing, updater channels, rollback, and desktop/mobile stores.
- Read [verification.md](references/verification.md) before claiming completion, and for test, debug, build, packaging, or release tasks.
- Read [coverage-map.md](references/coverage-map.md) when a task falls outside the existing routes or when auditing whether a Tauri subsystem has been overlooked.
- Read [source-ledger.md](references/source-ledger.md) only when checking tutorial coverage, source currency, contradictions, or the evidence behind a skill rule.
- Read [python-migration-source-ledger.md](references/python-migration-source-ledger.md) only when auditing the Python-to-Rust tutorial/video corpus, source-specific lessons, or research saturation.
- Read [provenance.md](references/provenance.md) only when auditing origins, licenses, or the exact source snapshots used to create this skill.

## Core Workflow

### 1. Classify the Task

Identify whether the request is primarily:

- repository orientation or review;
- new application scaffolding;
- feature work across frontend and Rust;
- a Rust-only or frontend-only change;
- debugging, testing, performance, or security hardening;
- latency-critical acquisition, streaming, control, or presentation;
- opt-in remote companion control, browser-to-app synchronization, or presentation-only shared scenes;
- migration, packaging, signing, or release preparation;
- Python-to-Rust migration, extension packaging, or sidecar replacement.

Do not broaden a review or diagnosis into implementation unless requested.

### 2. Establish the Baseline

Find repository instructions first. Then inspect the relevant manifests, lockfiles, entry points, configuration, capability files, generated schemas, and existing tests. Determine the actual build and check commands from the repository rather than inventing script names.

For an existing app, pay particular attention to:

- `package.json` and the active JavaScript lockfile;
- `src-tauri/Cargo.toml`, `Cargo.lock`, `build.rs`, `src/lib.rs`, and `src/main.rs`;
- `src-tauri/tauri.conf.json` or its supported variant;
- `src-tauri/capabilities/`, `src-tauri/permissions/`, and generated schema files;
- Tauri command registration, plugin initialization, state management, and frontend invoke wrappers;
- current failures in formatting, compilation, tests, or builds.

### 3. Design the Boundary

Keep secrets, privileged operating-system access, durable state, validation, and policy enforcement in Rust. Keep presentation, transient UI state, and browser-safe logic in the frontend.

Choose the narrowest communication primitive:

- command for request/response;
- event for low-volume notification or broadcast;
- channel for ordered or streaming data;
- managed state for shared native resources, not arbitrary frontend state.

Specify request types, response types, error shape, cancellation or lifecycle needs, and security permissions before implementation.

For latency-critical work, also define the measured endpoints, monotonic clock domains, ordering requirements, overload behavior, and which data is authoritative before choosing threads, queues, IPC, or priority settings.

For remote companion work, first choose presentation-only versus native application control. Define typed actions/scopes, the Rust authority, explicit activation, pairing/proof/revocation, freshness and lease behavior, and exact transport/CSP dependencies before opening a connection. Never turn arbitrary DOM, global input, shell, file, URL, or credential access into a remote action surface.

### 4. Implement One Vertical Slice

For cross-layer work, use this order:

1. define Rust domain types and a serializable boundary error;
2. implement and unit-test non-Tauri business logic;
3. add a thin command, event, or channel adapter;
4. register it in the builder and manage only required state;
5. adjust the minimum capability, permission, scope, or CSP surface;
6. add or update one typed frontend wrapper;
7. connect the UI and test success, expected failure, and malformed input;
8. run a compilation gate before expanding the feature.

Avoid mixing a framework migration, dependency overhaul, and product feature in one patch unless explicitly requested.

### 5. Apply Quality and Security Gates

Use [rust-quality.md](references/rust-quality.md) for Rust implementation rules and [tauri-security.md](references/tauri-security.md) for boundary review. Do not silence lints, add broad wildcards, clone around ownership problems, hold synchronous locks across `.await`, log secrets, or introduce `unsafe` without a documented invariant and a clear necessity.

### 6. Verify and Report

Follow [verification.md](references/verification.md). Start with focused checks, then run the repository's broader gates. Report:

- the outcome and affected layers;
- the important files and design decisions;
- the exact verification commands and results;
- any skipped checks, platform limitations, pre-existing failures, or remaining security/release risks.

## Mandatory Pause Points

Stop and request direction before:

- destructive data or configuration migrations;
- changing the frontend framework, package manager, Rust edition, or Tauri major version without an explicit migration request;
- enabling remote WebView content or broad filesystem, shell, process, or network access;
- introducing `unsafe` when a safe design remains plausible;
- publishing, signing, notarizing, uploading to a store, or using account credentials or secrets;
- selecting a bundle identifier, signing identity, updater endpoint, or release channel that the user has not supplied.

These gates do not prevent read-only diagnosis, local builds, or drafting the exact changes needed.
