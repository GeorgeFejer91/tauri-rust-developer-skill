# Python-to-Rust Migration Source Ledger

Checked 2026-08-22. This ledger records the public research behind [python-to-rust-migration.md](python-to-rust-migration.md). Every listed video had an accessible English caption track that was downloaded and read in full; captions and source prose are not redistributed. Version-sensitive code is subordinate to current primary documentation.

## Contents

- [Primary Documentation and Maintained Guides](#primary-documentation-and-maintained-guides)
- [Individually Reviewed YouTube Transcripts](#individually-reviewed-youtube-transcripts)
- [Consensus and Corrections](#consensus-and-corrections)
- [Search Boundary](#search-boundary)

## Primary Documentation and Maintained Guides

| ID | Source | Decision and key takeaway |
|---|---|---|
| P001 | [The Rust Programming Language](https://doc.rust-lang.org/book/) | adopted; canonical ownership, borrowing, enums, errors, traits, concurrency, testing and Cargo learning path |
| P002 | [Rust By Example](https://doc.rust-lang.org/rust-by-example/) | adopted; compact executable companion for `Option`, `Result`, strings, traits and modules |
| P003 | [Rust documentation index](https://doc.rust-lang.org/stable/) | adopted; routes to Cargo, rustdoc, Clippy, editions, the Reference and Rustonomicon |
| P004 | [Rust for Python Programmers](https://microsoft.github.io/RustTraining/python-book/) | adopted; broad Python-oriented language course |
| P005 | [Migration Patterns](https://microsoft.github.io/RustTraining/python-book/ch15-migration-patterns.html) | adopted; profile, port one clean hotspot with the same API, expand gradually, then reassess full rewrite |
| P006 | [Corrode: Migrating from Python to Rust](https://corrode.dev/learn/migration-guides/python-to-rust/) | adopted with consultancy bias noted; useful decision criteria, boundary batching, microservice/CLI options and incremental delivery |
| P007 | [PyO3 user guide](https://pyo3.rs/main/) | adopted; current authority for Rust extensions and embedding Python |
| P008 | [PyO3: calling Python from Rust](https://pyo3.rs/main/python-from-rust) | adopted; attachment tokens, Python reference counting and object lifetime model |
| P009 | [PyO3 conversions](https://pyo3.rs/main/conversions/traits) | adopted; extraction and conversion are fallible boundary work, not free casts |
| P010 | [PyO3 error handling](https://pyo3.rs/main/function/error-handling.html) | adopted; convert domain errors into intentional Python exception classes with `PyResult`/`From`/`map_err` |
| P011 | [PyO3 performance](https://pyo3.rs/main/performance.html) | adopted; detach for long Rust-only work and minimize interpreter/object overhead |
| P012 | [PyO3 parallelism](https://pyo3.rs/main/parallelism) | adopted; Rust threads help only when work does not continuously require attached Python access; avoid GIL deadlocks |
| P013 | [PyO3 free-threading](https://pyo3.rs/main/free-threading) | adopted; free-threaded CPython is a separate compatibility target governed by `Send`/`Sync`, attachment and stop-the-world behavior |
| P014 | [PyO3 Python typing hints](https://pyo3.rs/main/python-typing-hints.html) | adopted; native packages still owe users `.pyi`/`py.typed` and current stub-generation evaluation |
| P015 | [PyO3 building and distribution](https://pyo3.rs/main/building-and-distribution) | adopted; extension versus embedded-Python distribution are different products |
| P016 | [PyO3 version migration appendix](https://pyo3.rs/main/migration) | adopted as a currency guard; explains why tutorial-era GIL refs, pointer APIs and conversions must not be copied blindly |
| P017 | [maturin tutorial](https://www.maturin.rs/tutorial.html) | adopted; current scaffold/develop/build and `abi3` entry point |
| P018 | [maturin project layouts](https://github.com/PyO3/maturin/blob/main/guide/src/project_layout.md) | adopted; mixed layout, private native submodule and typing layout avoid import/IDE pitfalls |
| P019 | [maturin packaging overview](https://github.com/PyO3/maturin) | adopted; release builds, wheels, manylinux and clean install expectations |
| P020 | [maturin GitHub Action](https://github.com/PyO3/maturin-action) | adopted; platform/architecture/interpreter wheel matrices, including explicit free-threaded variants |
| P021 | [rust-numpy](https://docs.rs/numpy/latest/numpy/) | adopted; borrow NumPy arrays as ndarray views and make mutability/aliasing explicit |
| P022 | [`ndarray`](https://docs.rs/ndarray/latest/ndarray/) and [NumPy-user guide](https://docs.rs/ndarray/latest/ndarray/doc/ndarray_for_numpy_users/) | adopted; memory layout and higher-order traversals matter; it is not a complete NumPy/SciPy replacement |
| P023 | [Serde](https://serde.rs/) | adopted; typed format-independent serialization is the default replacement for dict-shaped parsing code |
| P024 | [Rust Cookbook CSV recipes](https://rust-lang-nursery.github.io/rust-cookbook/encoding/csv.html) | adopted; typed/byte CSV paths, invalid-data policy and explicit flush/error handling |
| P025 | [Arrow RS PyArrow bridge](https://arrow.apache.org/rust/arrow/pyarrow/index.html) | adopted; automatic Arrow RS/PyArrow interchange through the C Data Interface |
| P026 | [PyArrow integrations](https://arrow.apache.org/docs/python/integration.html) | adopted; Arrow is both an in-memory format and a cross-language interchange contract |
| P027 | [Arrow PyCapsule interface](https://arrow.apache.org/docs/python/extending_types.html) | adopted; prefer `__arrow_c_schema__`, `__arrow_c_array__` and `__arrow_c_stream__` to bespoke conversions |
| P028 | [Polars: coming from pandas](https://docs.pola.rs/user-guide/migration/pandas/) | adopted; no index, Arrow memory, strict types, expressions, parallelism and lazy execution require a model change |
| P029 | [Polars lazy optimizations](https://docs.pola.rs/user-guide/lazy/optimizations/) | adopted; predicate/projection/slice pushdown, common-subplan elimination and join planning explain many gains |
| P030 | [DataFusion Python](https://datafusion.apache.org/python/) | adopted; Rust query engine can remain behind a Python SQL/DataFrame API |
| P031 | [DataFusion DataFrames](https://datafusion.apache.org/python/user-guide/dataframe/) | adopted; lazy plans and Arrow streaming enable batch/zero-copy composition |
| P032 | [DataFusion UDFs](https://datafusion.apache.org/python/user-guide/common-operations/udf-and-udfa.html) | adopted; conversion to Python objects is often the slow path, so prefer built-ins, Arrow batches or Rust UDFs |
| P033 | [DataFusion custom table providers](https://datafusion.apache.org/python/user-guide/io/table_provider.html) | adopted; stable FFI/PyCapsule seam for Rust-native data sources |
| P034 | [DataFusion Python FFI guide](https://datafusion.apache.org/python/contributor-guide/ffi.html) | adopted; Rust has no stable native ABI, so separately compiled crates need an explicit stable FFI rather than shared internal Rust types |
| P035 | [CPython free-threading guide](https://docs.python.org/3/howto/free-threading-python.html) | adopted; packages may re-enable the GIL and must be tested on the promised interpreter build |
| P036 | [CPython extension support for free threading](https://docs.python.org/3/howto/free-threading-extensions.html) | adopted; native modules must explicitly declare/support GIL-disabled execution |
| P037 | [Python `timeit`](https://docs.python.org/3/library/timeit.html) | adopted only for small controlled snippets; end-to-end and memory/load measurements are still required |
| P038 | [Pydantic architecture](https://pydantic.dev/docs/validation/dev/internals/architecture/) | adopted; Python analyzes annotations and sends a structured core schema to a reusable Rust validation/serialization engine |
| P039 | [Pydantic V2 design](https://pydantic.dev/articles/pydantic-v2) | adopted as a production case study; stable Python API, small recursive validators, structured errors and a comprehensive wheel matrix |
| P040 | [Ruff/Astral origin](https://astral.sh/blog/announcing-astral-the-company-behind-ruff) | partial; confirms a Rust implementation can improve Python tooling while remaining a Python ecosystem product, but marketing benchmarks are not migration estimates |

## Individually Reviewed YouTube Transcripts

| ID | Video | Decision and key takeaway |
|---|---|---|
| Y001 | [Quickstart Guide: Bridge Python & Rust with PyO3](https://www.youtube.com/watch?v=01hYL76B_d8) | partial; good one-function environment/import smoke test, not production packaging guidance |
| Y002 | [Rust for Python Developers — Dave Halter](https://www.youtube.com/watch?v=24E4meYni6s) | adopted; candid fit matrix and warning that microbenchmarks do not predict rewrite duration or total-system gain |
| Y003 | [Explicit is Better than Implicit—Rust for Pythonistas](https://www.youtube.com/watch?v=62yfBiHrUis) | partial; useful paradigm/type orientation, but first-month and historical API claims require primary verification |
| Y004 | [Scaling Data Processing at DeepL Research with Rust](https://www.youtube.com/watch?v=7YmaBitI4aY) | adopted; optimize the actual constraint, reuse Arrow RS, preserve Python delivery and accept lower throughput when predictable memory is the product need |
| Y005 | [From Python to Rust 00: Hello World](https://www.youtube.com/watch?v=7odJDwhjCXQ) | partial; scientific-Python decision framing is strong, exact tooling/GIL statements are historical |
| Y006 | [Why We Made the Switch to Polars](https://www.youtube.com/watch?v=CtkMzCIXOWk) | adopted with current-doc corrections; bounded pilot, native expressions, lazy plans, representative benchmarks and team fit |
| Y007 | [Calling Rust Code from Python](https://www.youtube.com/watch?v=DpUlfWP_gtg) | partial; excellent minimal PyO3 scaffold, but per-call dictionaries can consume the benefit |
| Y008 | [PyO3 101—Writing Python Modules in Rust](https://www.youtube.com/watch?v=FWkCPYl_58M) | adopted with corrections; exceptions, conversion table, numeric widths, Python tests, typing and wheel CI; Numba may be simpler |
| Y009 | [Rust for Scientific Computing — ARCHER2](https://www.youtube.com/watch?v=H6ONF4nTIHc) | adopted; new bounded CPU/distributed subsystem can fit Rust while C++-heavy or GPU-dependent projects may not |
| Y010 | [Speed Up Python With Rust](https://www.youtube.com/watch?v=MbI4eMQ0ycg) | correction source; batched Rust beat per-element calls, but idiomatic vectorized NumPy beat the custom Rust implementation |
| Y011 | [An Introduction to Coding in Rust for Pythonistas](https://www.youtube.com/watch?v=MoqtsYLGCC4) | adopted as language orientation; learn Rust idioms instead of transliterating Python OOP |
| Y012 | [Interfacing Rust and Python — Nordic-RSE](https://www.youtube.com/watch?v=UQF2Ez8GyL8) | adopted with production additions; prototype in Python, compare outputs, release-build benchmark, then parallelize and package |
| Y013 | [How Python Harnesses Rust through PyO3](https://www.youtube.com/watch?v=UilujdubqVU) | adopted; generated C-ABI trampolines, panic containment, reference-count safety and conversion costs explain the real boundary |
| Y014 | [PyO3: From Python to Rust and Back Again](https://www.youtube.com/watch?v=UmL_CA-v3O8) | adopted; keep Python business logic, expose stable Rust primitives, and test free-threading/subinterpreters/stale builds explicitly |
| Y015 | [Polars: The Next Great Python Data Science Library?](https://www.youtube.com/watch?v=VHqn7ufiilE) | partial; useful pandas syntax bridge, but the expression/query-engine model is more important than method translation |
| Y016 | [PyO3 and Rust in Action](https://www.youtube.com/watch?v=Y3e1BwGDOxc) | historical case study; non-vectorizable numerical kernel, NumPy views, near-C performance and reduced binding boilerplate |
| Y017 | [Pydantic V2 and Its Rust Superpowers](https://www.youtube.com/watch?v=YynpIOnGcto) | adopted; stable schema seam, generic Rust core, recursion guards, structured errors and binary compatibility matrix |
| Y018 | [Python Rust Extensions: Massively Speed Up Your Code](https://www.youtube.com/watch?v=_2ENYAWwqyk) | partial; current basic scaffold and release benchmark, not a production layout/release guide |
| Y019 | [From Python to Rust Part 2](https://www.youtube.com/watch?v=i957TYyN_SY) | partial; useful strings/borrowing/crates comparison; added Unicode byte-boundary and reproducibility corrections |
| Y020 | [How To Make Your Python Packages Really Fast With Rust](https://www.youtube.com/watch?v=jlWhnrk8go0) | correction source; toy Fibonacci highlights release builds, overflow and call overhead, not general speedup |
| Y021 | [Why Python Developers Should Learn Rust](https://www.youtube.com/watch?v=lkSxUIdP6Ds) | adopted as complementary-language guidance; compiler/Clippy/rustfmt and explicit mutability support robust native components |
| Y022 | [Combining Rust and Python: The Best of Both Worlds?](https://www.youtube.com/watch?v=lyG6AKzu4ew) | adopted; hybrid architecture, structs/traits, `Option`/`Result`, and Python retained for iteration/ecosystem |
| Y023 | [Real Python Podcast: Polars](https://www.youtube.com/watch?v=vLYCKWFeC3k) | adopted with current-doc corrections; training aids, eager exploration, lazy pipelines, Arrow interoperability and storage bottlenecks |

## Consensus and Corrections

The strongest consensus across primary sources and case studies is:

1. profile before choosing a language migration;
2. keep Python for orchestration, notebooks and mature libraries when it remains effective;
3. port one CPU/memory/reliability-sensitive component behind a clean contract;
4. batch calls and use NumPy/Arrow buffers instead of Python objects per row or element;
5. preserve behavior with differential, property, fuzz and installed-wheel tests;
6. use release builds and representative end-to-end benchmarks;
7. design errors, thread safety, cancellation, typing and packaging as part of the boundary;
8. expand only after measured production evidence; full rewrites are the exception.

Repeated corrections applied to tutorial advice:

- Rust is not automatically faster than vectorized NumPy, a query engine, a database, or an existing native library.
- Safe Rust prevents broad classes of memory/data-race bugs; it does not prevent logic errors, deadlocks, resource leaks, panics or unsafe/FFI defects.
- Free-threaded CPython changes the concurrency landscape but does not make every extension or workload parallel-safe.
- PyO3 and maturin APIs evolve; current official docs override copied code.
- A source-tree import or `maturin develop` success is not proof that wheels install and run across the supported matrix.
- Data semantics—nulls, dtypes, Unicode, order, time zones, overflow, floats and RNG—are more likely to break parity than syntax.

## Search Boundary

"All online tutorials" is not a finite set. Broad web, official-domain, GitHub, community and YouTube searches were refined until new results were duplicates, superficial introductions, inaccessible media, or repeated patterns already represented here. Search snippets and video titles were discovery aids only; no source was used as evidence until its accessible documentation or full caption track was inspected.
