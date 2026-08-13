# Rust Quality for Tauri

Use this guide while writing or reviewing the Rust half of a Tauri application. Apply rules at the smallest scope that preserves clarity; do not mechanically rewrite correct code.

## Ownership and API Shape

- Borrow with `&T` or `&str` when the function only reads data.
- Move values when the callee needs ownership.
- Clone only when two owners are genuinely required and the cost is acceptable.
- Prefer concrete domain types over loosely structured maps or JSON values.
- Make invalid states difficult to construct through enums, newtypes, and validated constructors.
- Keep public APIs narrow and document invariants, errors, and surprising behavior.

Do not use `clone()` merely to silence a borrow-checker error. First reconsider ownership, lifetime, and data flow.

## Error Boundaries

- Return `Result` for recoverable failure; reserve panics for violated internal invariants or truly unrecoverable startup conditions.
- Avoid `unwrap()` and `expect()` on runtime input, files, IPC payloads, network responses, database values, or lock acquisition in production paths.
- Preserve useful internal context, but translate it at the Tauri boundary into a stable serializable error.
- Separate a machine-readable error code from a safe user-facing message.
- Do not expose secrets, filesystem internals, SQL, tokens, or backtraces to the WebView.

Framework-independent services should return domain errors. Tauri command handlers should perform the final conversion into the wire error type.

## Async and Concurrency

- Never hold `std::sync` lock guards across `.await`.
- Keep synchronous critical sections short and free of I/O.
- Use async-aware synchronization when a lock must span async operations.
- Prefer message passing or owned task state when it reduces lock contention.
- Send blocking or CPU-heavy work to an appropriate blocking worker.
- Give long-running tasks explicit ownership, cancellation, shutdown, and duplicate-start semantics.
- Propagate task errors; do not silently detach important work.

Measure before optimizing. If performance matters, inspect allocation, serialization, lock contention, IPC frequency, and large payload copies rather than guessing.

## Serialization and IPC Types

- Use dedicated request and response structs for non-trivial commands.
- Define the Rust/JavaScript naming convention with Serde attributes where appropriate.
- Avoid untyped `serde_json::Value` when a stable schema is possible.
- Validate lengths, ranges, identifiers, paths, URLs, and enum variants in Rust.
- Consider forward compatibility for persisted or long-lived payloads.
- Avoid serializing secret-bearing internal structs directly.

## Logging and Observability

- Use structured, level-appropriate diagnostics.
- Log operation identifiers and safe context, not credentials, tokens, personal data, full document contents, raw IPC payloads, or unrestricted paths.
- Do not return internal logs to the frontend as an error message.
- Make expected user errors distinguishable from bugs and infrastructure failures.

## Unsafe Code

Prefer safe Rust and audited crates. If `unsafe` is necessary:

1. isolate it behind a small safe abstraction;
2. document the exact safety invariant next to the block;
3. validate all FFI inputs and ownership assumptions;
4. add targeted tests for boundary conditions;
5. avoid expanding the unsafe surface during unrelated work.

Pause for user direction if the request did not already authorize a design that requires unsafe code and a safe alternative may exist.

## Structure

- Keep `main.rs` as a thin executable entry point.
- Put reusable setup in `lib.rs` so it can be tested.
- Separate domain logic from Tauri command adapters.
- Group by product responsibility rather than creating a file for every function.
- Keep platform-specific code behind explicit modules and `cfg` gates.

## Testing

- Unit-test domain logic without starting Tauri.
- Put public-behavior integration tests in `tests/` when they fit the crate.
- Use `#[tokio::test]` for genuinely async behavior and keep runtime-dependent tests explicit.
- Test success, validation failure, authorization failure, cancellation, and error translation.
- Mock the frontend IPC wrapper for UI tests instead of requiring a desktop shell for every test.

## Standard Rust Gates

Adjust the manifest path and feature flags to the repository:

```bash
cargo fmt --all -- --check
cargo check --manifest-path src-tauri/Cargo.toml
cargo clippy --manifest-path src-tauri/Cargo.toml --all-targets --all-features -- -D warnings
cargo test --manifest-path src-tauri/Cargo.toml --all-features
```

Do not blindly enable `--all-features` when features are mutually exclusive or platform-specific. Follow the project's CI matrix and explain any narrower command.

## Authoritative Live References

- The Rust Book: <https://doc.rust-lang.org/book/>
- Rust API Guidelines: <https://rust-lang.github.io/api-guidelines/>
- Tokio documentation: <https://docs.rs/tokio/latest/tokio/>
- Serde attributes: <https://serde.rs/attributes.html>
