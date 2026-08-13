# Performance and Production Readiness

Use this guide when an app polls native state, processes large data, streams output, starts slowly, consumes excess CPU/memory, or is described as production-ready.

## Measure the Whole System

Separate frontend, IPC, Rust, OS API, database, and sidecar costs. Record debug versus release builds, platform, architecture, dataset, warm/cold state, and measurement method. Do not repeat headline bundle-size or memory comparisons from tutorials as facts without reproducing their conditions.

Start with a user-visible trace:

```text
input -> frontend handler -> IPC serialization -> command queue -> native service -> response/stream -> render
```

Then measure the slow segment rather than optimizing by intuition.

## IPC Frequency and Shape

- Coalesce related native facts into one snapshot command instead of calling several commands that each refresh the same system state.
- Avoid one-second frontend polling for expensive process, filesystem, database, or network enumeration when native change detection or a channel can push updates.
- If polling is appropriate, stop it for hidden/destroyed windows, prevent overlapping calls, add backoff after failures, and refresh only as often as product needs require.
- Batch small operations and paginate large collections.
- Stream logs/progress through channels with bounded buffering; do not append an unbounded array in the frontend.
- Version large payloads and send deltas when correctness is manageable.

## Native Services and Caches

- Initialize expensive clients/pools once in managed state.
- Separate refresh from read so three UI panels can share one snapshot.
- Put cache ownership and invalidation in the domain/service layer, not scattered commands.
- Avoid holding locks while querying the OS, database, network, or sidecar.
- Bound concurrency and cancellation; discard stale results when a newer request supersedes them.

## Frontend Rendering

- Virtualize or paginate long process/log/file tables.
- Use stable IDs, not array indices, for changing collections.
- Avoid a full component-tree update for every stream chunk; buffer and render at a controlled cadence.
- Clean timers and event/channel listeners on unmount and window destruction.
- Keep native errors as structured data instead of triggering expensive generic rerenders or alert loops.

## Startup

Render useful UI early and defer non-critical plugins/services. Profile frontend asset parsing, Rust setup hooks, database migrations, network checks, sidecar startup, and window creation independently. Do not hide an avoidable synchronous startup bottleneck behind a fixed-delay splash screen.

## Production Review

“Production-ready” requires evidence beyond a release build:

- threat model and least-privilege capabilities;
- safe errors, redacted/rotated logs, crash diagnostics, and user support path;
- schema/settings/update compatibility and backups;
- offline, timeout, retry, cancellation, and partial-failure behavior;
- platform CI, signing/notarization, installers, updater, and rollback;
- resource limits and sustained-run tests;
- accessibility, localization, keyboard/native conventions, and user-data lifecycle;
- dependency/license/SBOM and vulnerability policy.

Privileged product controls—Docker/process management, filesystem mutation, shell execution, database administration—need explicit confirmation, authorization, stable resource identifiers, race-resistant revalidation, and audit-safe diagnostics. A button label is not a security control.

## Profiling and Checks

- Use release builds for runtime comparison while retaining symbols where profiling requires them.
- Inspect Rust spans/flamegraphs and async task/lock behavior with suitable current tools.
- Use WebView performance tools for render, layout, JavaScript, and network activity.
- Track artifact sizes by component and Cargo features; remove unused plugins/features before exotic optimization.
- Add a repeatable benchmark or regression threshold for the actual bottleneck.

Report measurements and remaining platform gaps rather than claiming “native performance.”
