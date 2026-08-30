# Tauri + Rust Developer Skill

A production-minded Codex/Claude-compatible agent skill for building, extending, debugging, reviewing, testing, securing, and releasing Tauri v2 applications with idiomatic Rust backends and web frontends.

The skill treats a Tauri application as one system:

- a least-privilege native Rust core;
- a typed IPC boundary;
- a maintainable WebView frontend;
- platform-aware desktop and mobile integrations;
- tested, signed, and recoverable release pipelines.

## Coverage

The compact [SKILL.md](SKILL.md) routes work into focused references covering:

- repository orientation, scaffolding, and Tauri v1-to-v2 migration;
- commands, events, channels, managed state, async work, and typed frontend adapters;
- idiomatic Rust ownership, errors, serialization, concurrency, logging, and tests;
- capabilities, permissions, scopes, CSP, remote content, and supply-chain security;
- windows, WebViews, title bars, splash screens, tray icons, menus, and lifecycle;
- Android/iOS architecture, native plugins, JNI/FFI, devices, signing, and stores;
- sidecars and long-lived Python, Node, Go, or native child processes;
- incremental Python-to-Rust migration, PyO3/maturin extensions, sidecar replacement, and Rust-backed data-science pipelines;
- filesystem access, settings, SQLite, migrations, keychains, and encrypted storage;
- networking, OAuth, WebSockets, transfers, localhost services, and native libraries;
- opt-in smartphone/browser Marionette control with typed Rust authority, BRSP/VDO.Ninja/WebSocket transport choices, scoped capabilities, and physical cross-runtime qualification;
- latency-critical acquisition, bounded live-data pipelines, clock mapping, deadline-aware output, and measured OS scheduling;
- React, Vue, Svelte, Solid, Angular, Next.js, Nuxt, Qwik, and Rust/WASM frontends;
- mocks, WebdriverIO, real-app E2E, platform debugging, performance, and profiling;
- CI matrices, installers, code signing, notarization, updater channels, and rollback.

See the [coverage map](references/coverage-map.md) for the complete checklist.

## Installation

### Codex

Clone this repository into the Codex skills directory under the folder name `tauri-rust-developer`:

```bash
git clone https://github.com/GeorgeFejer91/tauri-rust-developer-skill.git \
  ~/.codex/skills/tauri-rust-developer
```

Restart or open a new Codex session, then invoke it explicitly with:

```text
Use $tauri-rust-developer to review this Tauri project.
```

Codex may also select it automatically when the request matches the description in `SKILL.md`.

### Claude Code

For a project-local installation:

```bash
mkdir -p .claude/skills
git clone https://github.com/GeorgeFejer91/tauri-rust-developer-skill.git \
  .claude/skills/tauri-rust-developer
```

Follow the current Claude Code skill-discovery rules if your installation uses a different user-level location.

## Evidence and Version Policy

The guidance was synthesized from pinned Rust/Tauri/Codex/Claude skill repositories, current official Tauri, Rust, PyO3, maturin, Arrow, Polars, DataFusion and CPython documentation, written tutorials, production case studies, the Browser Remote Sync Protocol and Affect Tracker cross-runtime architecture, a 270-page community book, 21 Tauri tutorial transcript sets, and 23 individually reviewed Python-to-Rust video transcript sets.

The [source ledger](references/source-ledger.md) records every processed source as adopted, confirmed, corrected, partial, or rejected. It deliberately rejects stale Tauri v1 configuration, disabled CSP, broad filesystem/shell/network authority, fictional APIs, panic-heavy runtime code, unauthenticated localhost protocols, and unsafe release shortcuts.

Tauri and platform tooling change quickly. A project's pinned versions take precedence, and current official Tauri, Rust, frontend-framework, operating-system, and store documentation override community examples when they conflict.

## Repository Structure

```text
.
├── SKILL.md
├── agents/
│   └── openai.yaml
└── references/
    ├── project-workflow.md
    ├── tauri-architecture.md
    ├── rust-quality.md
    ├── tauri-security.md
    ├── latency-critical-systems.md
    ├── marionette-remote-control.md
    ├── python-to-rust-migration.md
    ├── python-migration-source-ledger.md
    └── ...
```

The research clones, downloaded captions, and PDF corpus are intentionally excluded. The skill has no runtime dependency on them; exact snapshots and license decisions are documented in [provenance](references/provenance.md).

## Validation

This repository is validated with OpenAI's skill validator:

```bash
python3 /path/to/skill-creator/scripts/quick_validate.py .
```

The published package has also been checked for missing internal Markdown links, duplicate source-ledger identifiers, and missing contents indexes in long references.

## License

The original synthesis is released under the [MIT License](LICENSE). Upstream notices and source-specific licensing decisions are preserved in [provenance](references/provenance.md).
