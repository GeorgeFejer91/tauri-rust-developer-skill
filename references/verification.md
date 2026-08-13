# Verification and Release Gates

Use this guide before claiming completion. The goal is evidence appropriate to the risk, not a ritual list of commands that may not exist in the project.

## Discover the Real Gates

Inspect `package.json`, lockfiles, Cargo configuration, CI workflows, task runners, and repository instructions. Use the project's commands and versions. Do not introduce a second JavaScript package manager or claim that a check ran when the environment could not run it.

Establish a pre-change baseline when practical. If a command already fails, record the failure and determine whether the change affects it.

## Verification Ladder

### 1. Focused Checks

- tests for changed domain modules;
- frontend tests for changed components or API wrappers;
- serialization round trips and validation edge cases;
- denied capability/scope behavior for privilege changes;
- cancellation and cleanup for long-running tasks.

### 2. Rust Gates

Adapt features and manifest paths to the project:

```bash
cargo fmt --all -- --check
cargo check --manifest-path src-tauri/Cargo.toml
cargo clippy --manifest-path src-tauri/Cargo.toml --all-targets --all-features -- -D warnings
cargo test --manifest-path src-tauri/Cargo.toml --all-features
```

Use the repository's CI feature matrix when `--all-features` is invalid, overly expensive, or platform-specific.

### 3. Frontend Gates

Run only scripts that exist, using the detected package manager. Common categories are:

- formatting and linting;
- type checking and unit or component tests;
- production frontend build.

A production build matters because development servers can hide asset paths, CSP differences, dead-code behavior, and environment-variable mistakes.

### 4. Tauri Integration

- confirm commands are registered and frontend names match;
- test serializable success and error paths;
- verify plugin initialization and required permissions;
- inspect generated capability schemas after relevant changes;
- launch the development app when a GUI is available and inspect safe native/WebView diagnostics.

For frontend tests, mock the typed IPC wrapper rather than scattering mocks of raw command names. For Rust tests, keep domain logic independent of a live Tauri runtime where possible.

### 5. End-to-End and Platform Tests

Tauri's end-to-end tooling is version- and platform-sensitive. Consult the current official testing documentation before configuring WebDriver or automation. Do not preserve historical platform limitations or driver setup as permanent facts.

Test platform-specific features on their target operating systems. If only one platform is available, clearly state the untested matrix. Cross-compilation or a successful compile does not prove runtime integration, installer behavior, permissions, or signing.

### 6. Build and Packaging

For package-related changes, run the repository's production Tauri build when the environment and requested scope permit it. Inspect generated artifacts without publishing them. Verify:

- product and bundle identifiers;
- icons, resources, sidecars, and file associations;
- minimum operating-system versions and architectures;
- updater configuration, if present;
- installer/runtime behavior and absence of development credentials or debug capabilities.

## Release Boundary

Building a local artifact does not authorize distribution. Get explicit direction before:

- importing or using signing certificates or account credentials;
- notarization, store submission, updater publication, or release creation;
- uploading artifacts or changing production endpoints, identities, keys, or release channels.

Use current official platform and Tauri distribution documentation at release time. Requirements and provider workflows change too frequently to encode as timeless instructions.

## Completion Report

Report verification as concrete evidence:

```text
Changed: native command, state service, typed frontend wrapper, capability scope
Passed: cargo fmt, cargo check, focused Rust tests, frontend typecheck, production frontend build
Not run: GUI smoke test (no display), macOS packaging (Linux host)
Pre-existing: unrelated lint failure in ...
Remaining risk: target-platform signing and installer behavior unverified
```

Never collapse “not run” into “passed.” Include enough command detail that the user can reproduce the result.

## Authoritative Live References

- Tauri testing: <https://v2.tauri.app/develop/tests/>
- Tauri debugging: <https://v2.tauri.app/develop/debug/>
- Tauri distribution: <https://v2.tauri.app/distribute/>
- Cargo commands: <https://doc.rust-lang.org/cargo/commands/>
