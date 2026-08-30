# Project Workflow

Use this guide for repository orientation, scaffolding, migrations, dependencies, and changes that span the frontend and Rust core.

## Inspect Before Editing

1. Find `AGENTS.md` and other repository instructions from the working directory upward and inside the target subtree.
2. Inspect `git status` and preserve unrelated user changes.
3. Detect the frontend package manager from its lockfile. Do not generate a second lockfile.
4. Read `package.json`, `src-tauri/Cargo.toml`, `Cargo.lock`, Tauri configuration, capability files, Rust entry points, and existing test setup.
5. Determine exact installed versions and feature flags. Do not assume the latest documentation matches a pinned project.
6. Run a cheap baseline check when feasible and record existing failures before modifying code.

Useful read-only probes, adjusted to the repository:

```bash
git status --short
cargo metadata --format-version 1 --no-deps --manifest-path src-tauri/Cargo.toml
cargo tree --manifest-path src-tauri/Cargo.toml -e features
```

Inspect scripts before invoking the frontend package manager. A project may use npm, pnpm, Yarn, Bun, Deno, Trunk, or another tool.

## Scaffold a New Application

Collect or infer only low-risk choices. Ask for choices that materially affect identity or architecture:

- target platforms: desktop, mobile, or both;
- frontend framework and language;
- package manager;
- product name and bundle identifier;
- native features such as filesystem, shell, database, updater, deep links, global shortcuts, or sidecars;
- packaging and signing targets.

Use the current official Tauri scaffolder and documentation instead of preserving a historical command from this skill. Do not invent a production bundle identifier or signing identity.

After generation:

1. inspect the generated configuration and capability model;
2. compile the untouched Rust crate;
3. run the generated frontend build;
4. launch the development app if the environment supports a GUI;
5. commit or checkpoint the clean baseline only if requested.

Add features one at a time. After each native plugin, command group, or configuration module, run at least a Rust compilation gate and the relevant frontend check. This makes the first breaking change easy to identify.

## Plan a Cross-Layer Feature

Write down the contract before code:

- user-visible behavior and failure behavior;
- which side owns validation and business logic;
- request, response, event, or stream types;
- state lifetime and concurrency needs;
- permissions, scopes, CSP, and window or WebView exposure;
- test seams that do not require a live GUI.

Then implement the smallest end-to-end path. Prefer a small typed frontend API module over scattered raw `invoke` calls.

## Dependencies

- Reuse an existing dependency when it meets the requirement.
- Add crates with the project's normal Cargo workflow and frontend packages with the detected package manager.
- Decide workspace ownership deliberately. Prefer one Cargo workspace and lockfile for a Tauri shell plus shared Rust crates; do not create a nested workspace merely to evade an existing repository policy. Update that policy explicitly when the new root files are intentional.
- Inherit edition, license, repository, and minimum supported Rust version consistently across shipped workspace crates unless a crate has a documented reason to differ.
- Treat `rust-version` as a tested compatibility claim. Run meaningful checks with that exact toolchain against the resolved `Cargo.lock`; passing on a newer floating `stable` does not validate the declared MSRV, and transitive Tauri dependencies can raise it.
- Preserve one frontend package manager and canonical lockfile. A multi-page browser/desktop frontend should normally share the same build and dependencies rather than creating a second phone application package.
- Keep feature flags narrow; many Rust crates and Tauri plugins expose privileged or platform-specific features.
- Inspect maintenance status, license compatibility, platform support, and security implications before introducing a new dependency.
- Do not update unrelated dependencies or regenerate lockfiles unnecessarily.
- Distinguish version pins and lockfiles from stronger supply-chain claims. Android Gradle dependencies, wrapper distributions, Git sources, and CI actions may need their own checksums, locks, or verification metadata.
- If current information is necessary, consult the crate's documentation, the Tauri plugin documentation, and authoritative release notes.

## Migration Work

For Tauri major-version, framework, or configuration migrations:

1. inventory current APIs, plugins, features, capability files, custom build logic, and platform targets;
2. read the official migration guide for the exact source and destination versions;
3. separate mechanical changes from behavior changes;
4. migrate configuration and build plumbing before product features;
5. compile after each migration unit;
6. verify permissions and scopes again, because a successful build does not prove equivalent security behavior;
7. test each supported platform or state the untested matrix explicitly.

Do not combine a major migration with unrelated refactoring unless the user asks for it.

## Electron-to-Tauri Migration

Do not translate Electron APIs one-for-one. Inventory preload/context-bridge methods, Node built-ins, IPC handlers, BrowserWindows, menus/tray, auto-updates, protocols, child processes, filesystem/data paths, native modules, deep links, and packaging/signing. Classify each as frontend-only, Tauri core/plugin, Rust service, sidecar, or feature to remove.

Build one product-level typed Rust command and its least-privilege capability at a time. Node's `fs`, `path`, `child_process`, `crypto`, server routes, and native modules do not exist in the Rust core; select Rust equivalents based on the actual requirement. Replace arbitrary Electron IPC with explicit serializable DTOs and stable errors. Re-test window-close/background behavior, multi-window state, CSS/WebView differences, file paths, updater migrations, and installers on every supported OS before removing the Electron release path.

Do not recreate Node integration by enabling a generic shell/sidecar escape hatch. A Node sidecar is justified for a dependency migration bridge or a capability unavailable in Rust, and still needs fixed artifacts, a narrow protocol, supervision, and an exit plan where appropriate.

## Authoritative Live References

- Tauri start guide: <https://v2.tauri.app/start/>
- Tauri configuration reference: <https://v2.tauri.app/reference/config/>
- Tauri migration guides: <https://v2.tauri.app/start/migrate/>
- Cargo reference: <https://doc.rust-lang.org/cargo/reference/>

Treat these URLs as entry points and verify the pages applicable to the project's pinned version.
