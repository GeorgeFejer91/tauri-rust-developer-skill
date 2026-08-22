# Provenance and License Notices

This skill is an original synthesis of three permissively licensed skill repositories plus independently reviewed, unlicensed Tauri source repositories. It consolidates compatible ideas, resolves overlaps into one workflow, and adds explicit live-documentation guards where Tauri tooling is version-sensitive.

## Contents

- [Source Snapshot](#source-snapshot)
- [Research Corpus](#research-corpus)
- [Synthesis Decisions](#synthesis-decisions)
- [MIT License Notice: leonardomso/rust-skills](#mit-license-notice-leonardomsorust-skills)
- [MIT License Notice: stuffbucket/skills](#mit-license-notice-stuffbucketskills)
- [MIT License Notice: glebis/claude-skills](#mit-license-notice-glebisclaude-skills)

## Source Snapshot

| Source | Local clone | Commit | License | Primary influence |
|---|---|---|---|---|
| [leonardomso/rust-skills](https://github.com/leonardomso/rust-skills) | `sources/rust-skills` | `fd2a861ab0406a4ac536a55274d14ea6fd1ca9c9` | MIT | Idiomatic ownership, errors, async safety, serialization, logging, project structure, tests, and lint gates |
| [stuffbucket/skills](https://github.com/stuffbucket/skills) | `sources/stuffbucket-skills` | `ef6df09e5c0eb9639276760f8d74b29ad4ef54e7` | MIT | Tauri v2 architecture, command/event/channel/state patterns, capabilities, permissions, scopes, CSP, and debugging |
| [glebis/claude-skills](https://github.com/glebis/claude-skills) | `sources/claude-skills` | `46edb03cc915adbd78ee81e3406bc4480aefcaba` | MIT | Tauri application initialization, modular feature planning, and incremental build gates |
| [Takazudo/zudo-tauri-wisdom](https://github.com/Takazudo/zudo-tauri-wisdom) | `sources/zudo-tauri-wisdom` | `a3ba300cf4829c2f0735942193a715084fd82de3` | No reuse license found; README says personal notes/use at own risk | Independently synthesized lifecycle, watcher, testing, deployment, and iOS lessons; no source prose or templates copied |
| [dseirz-rgb/worker: tauri-v2-dev](https://github.com/dseirz-rgb/worker/tree/main/.kiro/skills/tauri-v2-dev) | `sources/dseirz-tauri-v2-dev` (sparse) | `5c0a3621c1adf090f4317676a4e498d4082e1898` | No reuse license found | Fully reviewed and rejected as stale/unsafe/placeholder-heavy; retained only as auditable negative evidence, with no content copied |

The clone paths refer to the local research workspace used to construct this skill; they are not included in the published skill repository. They are evidence sources, not runtime dependencies: `tauri-rust-developer` remains usable if copied or installed by itself.

## Research Corpus

The parent workspace also preserves pinned official-document and community indexes under `research-sources/`, 21 downloaded Tauri YouTube caption/metadata sets, the official 270-page community PDF plus extracted text and representative renders, and snapshots of selected written tutorials. These are research evidence, not redistributed skill dependencies. Exact Tauri source decisions are in [source-ledger.md](source-ledger.md).

A separate 2026-08-22 research pass reviewed 23 complete English YouTube caption tracks about Python-to-Rust migration, PyO3, Pydantic, Polars and scientific/data workloads, plus current primary Rust, PyO3, maturin, Arrow, Polars, DataFusion, CPython and production-case-study documentation. Captions and source prose are not included. The source-by-source decisions and saturation boundary are published in [python-migration-source-ledger.md](python-migration-source-ledger.md).

## Synthesis Decisions

- Tauri v2 is the default target; pinned project versions always take precedence.
- Rust domain logic is kept independent from thin Tauri adapters for testability.
- Commands, events, channels, and managed state are presented as one boundary-design system.
- Security guidance is mandatory for new IPC or privileged plugins, not an optional appendix.
- Current official documentation overrides embedded source advice for mobile APIs, automated testing, plugins, bundling, signing, notarization, and stores.
- No upstream boilerplate application or generated project asset is embedded in this skill.

## MIT License Notice: leonardomso/rust-skills

```text
MIT License

Copyright (c) 2025 Leonardo Maldonado

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## MIT License Notice: stuffbucket/skills

```text
MIT License

Copyright (c) 2025 Stuffbucket

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## MIT License Notice: glebis/claude-skills

```text
MIT License

Copyright (c) 2025-2026 Gleb Kalinin

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
