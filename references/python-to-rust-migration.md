# Python-to-Rust Migration

Use this module when replacing Python modules, services, sidecars, numerical kernels, validation/parsing code, or data pipelines with Rust. The default is an incremental, reversible migration that preserves a Python-facing or Tauri-facing contract. A full rewrite needs stronger evidence than a successful prototype.

## Contents

- [Choose the Smallest Useful Move](#choose-the-smallest-useful-move)
- [Prove the Case Before Porting](#prove-the-case-before-porting)
- [Freeze Behavior as an Executable Contract](#freeze-behavior-as-an-executable-contract)
- [Use a Strangler Migration](#use-a-strangler-migration)
- [Translate the Model, Not the Syntax](#translate-the-model-not-the-syntax)
- [Build a Python Extension Correctly](#build-a-python-extension-correctly)
- [Keep Data Crossing Cheap](#keep-data-crossing-cheap)
- [Migrate a Python Tauri Sidecar](#migrate-a-python-tauri-sidecar)
- [AI-Assisted Ports](#ai-assisted-ports)
- [Release Gate](#release-gate)
- [Authoritative Live References](#authoritative-live-references)

## Choose the Smallest Useful Move

| Need | Default route | Why |
|---|---|---|
| Faster pandas-style transformations | Try Rust-backed Python tools such as Polars or DataFusion first | Gains may require no Rust maintenance |
| One CPU-heavy parser, validator, or algorithm | Mixed Python/Rust package with PyO3 and maturin | Preserves Python callers and tests |
| NumPy/Arrow batch kernel | PyO3 plus rust-numpy or Arrow's standardized interchange | Avoids per-element object conversion |
| Existing Python worker in Tauri | Keep it as a supervised sidecar, then replace one protocol operation at a time | Provides rollback and isolates packaging risk |
| New native Tauri capability | Implement an in-process Rust service behind typed commands | Avoids adding a Python runtime when none is needed |
| Independent service with a stable network contract | Build one Rust service alongside the Python service | Enables shadow traffic and staged cutover |
| Python application whose main constraint is not solved by Rust | Keep Python | A rewrite adds cost without removing the bottleneck |

Prefer an existing native-backed Python library over custom Rust when it already implements the required semantics. NumPy, PyTorch, Polars, PyArrow, DuckDB, DataFusion, Numba, and domain libraries may already execute the expensive work outside the Python interpreter.

## Prove the Case Before Porting

Write a migration brief containing:

- the user-visible or caller-visible contract;
- the actual constraint: CPU, peak/resident memory, tail latency, startup, concurrency, deployment, correctness, or security;
- a profile or trace locating the cost;
- an idiomatic optimized Python/library baseline, not a deliberately slow Python loop;
- representative datasets, platforms, architectures, and warm/cold conditions;
- a rollback path and an owner for the Rust code.

Reject "Python is slow" as a sufficient diagnosis. Measure I/O, database/network waits, Python/native conversion, allocation, serialization, memory layout, query plans, subprocess startup, and frontend IPC separately. Use optimized release builds for Rust comparisons.

A Tauri shell around an unchanged Python/PySide worker can improve UI architecture and isolation, but it does not deliver the main runtime, startup, or installed-size reduction while the default artifact still ships Python and its native dependencies. State that packaging boundary explicitly instead of attributing a size win to the shell.

Do not proceed merely because a toy benchmark is faster. A port is justified when the measured end-to-end benefit exceeds boundary, packaging, training, and maintenance costs.

## Freeze Behavior as an Executable Contract

Before changing the implementation, inventory and test:

- public functions, classes, CLI flags, environment variables, configuration and wire formats;
- return types, exception classes/messages that callers rely on, exit codes and logging fields;
- accepted coercions, missing/null behavior, defaults and invalid-input behavior;
- integer width/overflow, float/NaN/infinity and comparison tolerances;
- Unicode normalization, code points versus UTF-8 byte offsets, invalid byte handling;
- ordering, hashing, duplicates, categorical values, time zones, daylight-saving transitions;
- RNG algorithm/seed behavior and reproducibility requirements;
- cancellation, retries, partial writes, concurrency and resource cleanup.

Build golden fixtures from production-shaped edge cases. Run the old and new implementations against the same inputs and compare normalized outputs. Add property tests for invariants and fuzz parsers or other untrusted-input boundaries. Compilation proves type and ownership consistency; it does not prove behavioral parity.

For schedule and timing ports, freeze the exact cursor semantics rather than comparing only aggregate counts. Fixtures should cover tie-breaking in numeric rounding, priority and stable ordering for events at the same sample, half-open callback windows and the final-boundary flush, explicit-field precedence when a present but invalid value suppresses a fallback, signed pre-zero samples and their discard rule, aliases/defaults, and reset behavior. Compare the complete ordered event payload and sequential buffer consumption. A mathematically plausible result is still a compatibility regression when it differs from the established contract.

When replacing a legacy final-buffer `frames + 1` or equivalent boundary workaround, model the terminal software boundary explicitly. Emit an engine-owned final-frame-submitted record after all metadata events at that sample, exactly once, and keep device drain/presentation/onset as a separate adapter receipt. Bound both maximum callback frames and the number/density of metadata events during plan construction; otherwise a nominally fixed event buffer can still hide an unbounded schedule scan on the real-time thread.

## Use a Strangler Migration

1. Select one high-value component with a clean boundary.
2. Give the Rust implementation the same public API or versioned protocol.
3. Keep the Python implementation as an explicit fallback during validation.
4. Run differential tests in CI; shadow or dual-run in production where side effects can be isolated.
5. Compare correctness, latency distributions, memory, throughput and failure behavior.
6. Cut over a bounded caller cohort or feature flag.
7. Expand only after the evidence holds under representative load.
8. Remove the old path only after the rollback window and compatibility obligations are satisfied.

Avoid editing both implementations independently for long periods. Define one conformance suite and one ownership plan for shared schemas and fixtures.

## Translate the Model, Not the Syntax

| Python habit | Rust model | Migration warning |
|---|---|---|
| `None` or sentinel | `Option<T>` or an explicit enum | Distinguish absent, null, empty and invalid |
| exceptions | `Result<T, E>` with domain error enums | Convert expected failures; do not expose panics |
| dict-shaped records | typed structs plus Serde | Preserve unknown-field and coercion policy |
| inheritance-heavy classes | structs, enums, traits and composition | Do not reproduce a Python class hierarchy mechanically |
| mutable shared objects | ownership, borrowing, channels or deliberate synchronization | Cloning everything hides a poor ownership design |
| generators | `Iterator`, async streams or channels | Preserve laziness, cancellation and backpressure |
| context managers | RAII guards plus explicit fallible shutdown when needed | `Drop` cannot report cleanup failure |
| `asyncio` everywhere | async Rust for concurrent I/O; worker pools/Rayon for CPU work | Do not hold locks across `.await` or block an async executor |
| arbitrary-size integers | a chosen fixed width or big-integer crate | Test overflow and serialization boundaries |
| Python strings | UTF-8 `String`/`&str` or explicit bytes | Byte indexing is not Python string indexing |
| unordered mappings | `HashMap` or ordered alternatives | Never depend accidentally on a different iteration order |

Treat borrow-checker friction as design feedback. First simplify ownership and data flow; add `Arc`, locks, interior mutability, lifetime complexity, or cloning only when the domain genuinely requires them.

## Build a Python Extension Correctly

For an existing Python package, prefer maturin's current mixed Rust/Python layout. Keep the Python package as the public facade and give the native module a private name such as `_core`. This keeps docstrings, type hints, compatibility adapters and a fallback path easy to maintain.

Start with one tiny function and verify `maturin develop` plus a Python import test. Then:

- make each boundary call coarse enough to amortize conversion and call overhead;
- accept and return batches, buffers, arrays or record batches instead of Python objects per element;
- use `PyResult`/error conversions to raise idiomatic Python exceptions;
- catch and prevent Rust panics at the boundary; validate sizes and recursion depth;
- keep Python object access inside an attached interpreter context;
- detach from the interpreter for long Rust-only work, including on free-threaded builds;
- do not retain borrowed Python views beyond their permitted lifetime;
- make `#[pyclass]` thread-safety intentional; immutable/frozen objects are easier to reason about;
- test GIL-enabled and free-threaded interpreters only when the supported matrix promises both;
- ship `.pyi` stubs and `py.typed`, and test the installed wheel rather than only a source-tree import;
- decide whether `abi3` reduces the wheel matrix without excluding APIs or performance the package needs.

`maturin develop --release` is a development benchmark, not release validation. Build wheels in CI for every supported operating system, architecture, Python ABI and libc family; install each wheel into a clean environment and run Python-level contract tests. If no matching wheel exists, users need a working Rust build toolchain, so missing artifacts are a product compatibility failure.

## Keep Data Crossing Cheap

### NumPy and numerical code

Use rust-numpy/`ndarray` views for compatible contiguous or strided data instead of converting arrays to Python lists. State dtype, dimensions, shape, strides, mutability, alignment and aliasing requirements. Prefer a new output buffer unless in-place mutation is part of the public contract and can be proven safe.

Compare against vectorized NumPy, compiled ufuncs, Numba and existing native libraries before writing Rust. Preserve broadcasting, dtype promotion, reduction order, BLAS/thread-pool behavior and numerical tolerances. Reproducibility may change when parallel reductions reorder floating-point operations.

### Arrow, Polars and query engines

Use Arrow's C Data/PyCapsule interfaces and `__arrow_c_array__`/`__arrow_c_stream__` rather than materializing rows as Python dictionaries. Honor producer lifetimes, null bitmaps, schemas, chunking, extension types and ownership callbacks.

For pandas-to-Polars work, adopt expressions and lazy query plans; do not translate index mutation and row-wise `apply` calls literally. Check:

- null versus NaN behavior and dtype strictness;
- index/multi-index assumptions that need real columns;
- categorical ordering, time zones and joins with duplicate keys;
- stable ordering only where explicitly requested;
- projection/predicate pushdown and query-plan inspection;
- peak memory, streaming support and storage throughput on the actual workload;
- plotting, scikit-learn and notebook integrations at the boundary.

For custom analytics, first consider a Rust-backed Python DataFusion/Polars extension or Arrow-native UDF. Row-by-row conversion to Python objects is usually the wrong seam.

## Migrate a Python Tauri Sidecar

Read [sidecars-and-processes.md](sidecars-and-processes.md), [tauri-architecture.md](tauri-architecture.md), and [tauri-security.md](tauri-security.md) as well.

Treat the existing sidecar protocol as the migration boundary:

1. capture protocol frames, errors, readiness, cancellation and shutdown in conformance tests;
2. isolate Python domain logic from process/bootstrap code;
3. implement one message or command in a Rust library;
4. expose it through the existing sidecar during the first parity phase, or add a Rust-native Tauri command behind the same typed frontend wrapper;
5. dual-run read-only operations and compare replies;
6. switch the frontend wrapper, capability and managed-state wiring for that operation only;
7. preserve a bounded fallback until packaged-target tests pass;
8. remove the Python runtime, PyInstaller artifacts and shell authority only after every operation has moved.

Do not replace a narrow sidecar with broad generic shell access. If the Rust implementation moves in process, reduce capabilities and delete obsolete process permissions. Re-test app startup, shutdown, crash isolation, cancellation, logs, installer size, signing and updater artifacts on every target.

## AI-Assisted Ports

Use an AI tool only on bounded components with an objective conformance suite. Require it to explain ownership, error, unsafe, dependency and semantic choices. Review every new crate and generated `unsafe` block. Reject ports that compile by adding blanket clones, `unwrap`, broad locks, lossy casts, unbounded allocation, ignored errors or silent semantic changes.

An effective loop is: translate one seam, compile, run differential/property tests, profile, review idioms/security, then refactor. Never ask a model to rewrite the whole repository and treat successful compilation as completion.

## Scientific and Hardware-Coupled Applications

For experiment, acquisition, media, or device-control software, define parity as several separate contracts rather than one feature checklist:

- accepted packages, schemas, legacy aliases, ordering, and hashes;
- sample-clock scheduling, callback boundaries, response timestamps, and clock mapping;
- device routing, calibration, safety limits, and target-specific output adapters;
- event logs, acquisition streams, durable artifacts, exports, and recovery behavior;
- numerical analysis, tolerances, model selection, and generated reports.

Keep the validated Python implementation as a temporary behavioral or scientific oracle while these seams move. Prefer development/CI probes that cannot become runtime authority. Use golden and differential fixtures for software semantics, but do not substitute them for physical loopback, onset, device-route, or long-run qualification. Each Windows, macOS, Linux, mobile, or XR backend needs evidence for the hardware boundary it actually owns.

Model “accepted and verified” separately from “executable and qualified.” A Rust parser may prove legacy schema, ordering, path, and hash parity before the native scheduler or device adapter exists. Let the reducer adopt that verified identity, but keep arming/start unavailable until the execution adapter is real; never route a verified production package through a demo executor. Local package replacement should disarm and invalidate stale remote/controller authority. Retain path-bearing verification receipts only in native state, make those receipts and raw compiled plans non-serializable by default, and return an explicitly constructed path-free summary to the WebView.

At every integrity-sensitive handoff, enforce a byte bound and hash and parse/compile the exact same byte snapshot or retained file handle. A successful digest check followed by reopening the mutable path does not close the time-of-check/use race. For a streaming implementation, tee the same chunks into the digest and consumer or into an owned staged resource; a retained handle alone does not rule out in-place mutation. If an asset is prepared later—for example, a WAV decoded during native audio preload—carry its expected digest and package generation to that boundary, hash and decode the same bounded bytes or handle, and retain the prepared resource or verified immutable staged copy until use. Only then may that asset contribute to execution qualification.

Treat container-declared lengths as untrusted even after the outer file length and digest have been verified. Before reserving decoded or expanded storage, prove that the declared sample/row/frame byte count is representable, fits inside the bytes available from the already opened source, and stays within the product's decoded-memory budget. Use a fallible exact reservation or bounded chunk store so geometric `Vec` growth cannot exceed the stated cap. When moving a large prepared buffer into shared ownership, prefer ownership transfer such as `Arc<Vec<T>>` over conversions that allocate and copy the full payload again. Give large native receipts and media blocks a custom bounded `Debug` implementation that omits paths, digests, secrets, fingerprints, and payload contents; non-`Serialize` alone does not prevent accidental multi-gigabyte or private diagnostic output.

When verification or compilation runs off-thread, capture the authoritative generation and package fingerprint, then publish the result only if both still match; a late result from a replaced package must be inert. Preserve legacy source-host path semantics while reading the old format; add an explicit relocatable/content-addressed schema instead of silently reinterpreting embedded absolute paths on another operating system.

For an expensive differential oracle, batch several independent fixtures through one native probe process and compare both success/failure and the stable first-failure message. Include a valid modern case, provenance/hash drift, ordering/count drift that reaches the intended check, and a legacy/defaulted case. This gives the pure Rust seam a fast executable contract without making Python the final runtime authority.

Bound the migration seam cumulatively as well as per record. A safe per-row limit can still multiply into an unsafe schedule or ledger, especially when cloned metadata appears on every event. Cap total encoded bytes, record counts, nesting and retained diagnostic payloads; avoid trusting attacker-controlled counts for initial allocation; and test that overflow fails before large duplication occurs.

Do not infer audio parity from schedule parity. Probe representative WAV headers and decoded layouts, query the selected device's supported rates, channels, sample formats and routing, and define conversion policy explicitly. Then measure physical onset/loopback and long-run behavior on each promised backend before calling the output path qualified.

Define the Python-free finish line before claiming completion: a clean-machine default installation starts and completes every required operation without Python, PySide, PyInstaller, or a Python worker. A Python differential oracle may remain in development or CI. An optional, bounded, supervised compatibility worker can be a migration tool only while the product permits a hybrid release; it must not ship as a fallback, hidden authority, or runtime dependency when the declared release is Python-free.

## Release Gate

Before deleting Python or declaring the migration complete, verify:

- old and new conformance suites pass on production-shaped fixtures;
- expected failures map to stable Python/Tauri errors rather than panics;
- release benchmarks improve the declared constraint end to end;
- memory, thread count, file descriptors and long-run behavior are bounded;
- Python/native boundaries use batched or zero-copy interchange where practical;
- wheels, binaries, sidecars, licenses, SBOM and signatures cover the target matrix;
- clean-machine installs do not unexpectedly require Cargo or Python;
- rollback, data compatibility and observability work in packaged builds;
- the team can maintain, review and operate the Rust component.

## Authoritative Live References

- PyO3 guide: <https://pyo3.rs/main/>
- maturin guide: <https://www.maturin.rs/>
- Rust for Python Programmers: <https://microsoft.github.io/RustTraining/python-book/>
- Polars migration guide: <https://docs.pola.rs/user-guide/migration/pandas/>
- rust-numpy: <https://docs.rs/numpy/latest/numpy/>
- Apache Arrow Python integration: <https://arrow.apache.org/docs/python/integration.html>
- DataFusion Python: <https://datafusion.apache.org/python/>
- CPython free-threading guide: <https://docs.python.org/3/howto/free-threading-python.html>

For the tutorial/video corpus, source-specific lessons and currency notes, read [python-migration-source-ledger.md](python-migration-source-ledger.md).
