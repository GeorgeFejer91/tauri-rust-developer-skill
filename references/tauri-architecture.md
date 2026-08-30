# Tauri Architecture

Use this guide to design the native/WebView boundary and organize commands, events, channels, state, and async work.

## Ownership Boundary

The Rust core should own privileged behavior: secrets, filesystem and process access, durable storage, system integration, validation, and authorization. The WebView should own presentation and browser-safe interaction state.

Do not expose a generic native escape hatch such as “run command,” “read any path,” or “request any URL.” Expose product-level operations with narrow typed inputs.

## Choose the IPC Primitive

| Need | Primitive | Typical use |
|---|---|---|
| One request and one result | Command | Save settings, query status, open a project |
| Low-volume notification | Event | Theme changed, window state changed |
| Ordered progress or streaming data | Channel | Downloads, logs, long-running task progress |
| Shared native service or resource | Managed state | Database pool, client, task registry |

Commands are not streaming transports. Events are not ideal for large or high-frequency payloads. Channels should have lifecycle and cancellation behavior rather than leaking tasks after a window closes.

## Command Contract

Keep command handlers thin:

1. deserialize and validate the request;
2. acquire or clone only the state handle needed;
3. call framework-independent Rust logic;
4. convert the domain result into a stable serializable response or boundary error.

Use explicit request structs once a command has more than a trivial scalar argument. Keep frontend property names consistent with the Tauri version and serializer configuration. When Rust types use `snake_case` and JavaScript uses `camelCase`, declare the serialization convention rather than manually renaming fields throughout the codebase.

Use a stable error object, for example a machine-readable code plus a safe message. Do not send `anyhow` context, filesystem internals, tokens, SQL fragments, or debug dumps directly to the frontend.

Use raw IPC bodies or `tauri::ipc::Response` only for a measured binary-transfer need. Bound byte lengths, validate content, and keep authorization in Rust; arbitrary frontend-supplied headers do not create a trusted authentication channel. For sustained or progressive transfer, use a channel or write/read through a narrowly authorized file instead of allocating one giant message.

Register every command deliberately. Keep command registration near the application's builder or group related registrations behind a small module function. A defined but unregistered command is dead code; a registered command without a reviewed exposure policy is a security gap.

## State and Async Work

- Put services or resource handles in managed state, not product presentation state.
- Tauri already shares managed state safely; do not wrap every managed value in an extra `Arc` without an independent ownership need.
- Prefer types whose methods enforce their own invariants.
- Use an async-aware mutex for data that must remain locked during async work; use a synchronous mutex only for tiny non-awaiting critical sections.
- Never hold a synchronous lock guard across `.await`.
- Clone cheap handles such as `Arc` or client handles before awaiting instead of retaining a global lock.
- Send blocking filesystem, compression, CPU-heavy, or process-wait work to an appropriate blocking worker rather than starving the async runtime.
- Define task ownership, cancellation, duplicate-start behavior, and shutdown cleanup for long-running work.

## Events and Channels

Use small versionable payload structs. Name events as stable application contracts rather than UI implementation details. Document whether delivery is global, window-specific, or WebView-specific.

Prefer typed events/channels over evaluating JavaScript strings. If `WebviewWindow::eval` is unavoidable, execute a fixed trusted script and serialize data with a purpose-built escaping/serialization library; never interpolate user-controlled text into code.

For channels:

- define progress, data, completion, cancellation, and error messages;
- preserve ordering requirements explicitly;
- avoid unbounded production when the frontend cannot keep up;
- clean up producers when the consumer disappears;
- avoid sending secrets or raw internal logs.

## Module Layout

Keep `main.rs` minimal. Put reusable application setup in `lib.rs` and split by responsibility, for example:

```text
src-tauri/src/
  main.rs
  lib.rs
  commands/
  domain/
  services/
  state.rs
  error.rs
```

Use the project's existing layout if it is coherent. Architecture changes should reduce coupling, not merely rename folders.

On the frontend, centralize native calls in a small typed API layer. Components should not need to know raw command names, plugin permission details, or wire-level error shapes.

## Shared Core Across Native Shells

When desktop, browser, mobile, or XR targets share product semantics, separate application authority from framework adapters:

```text
contracts       closed actions, scopes, DTOs, snapshots
protocol        transport-neutral proof, negotiation, sequencing
domain/core     pure authoritative reducer
tauri adapter   desktop IPC, networking, lifecycle
JNI adapter     Android/XR lifecycle and native APIs
browser adapter strict web implementation or an explicit Wasm build
```

- Keep Tauri, Android/JNI, UI, filesystem, networking, audio, and device types out of the shared domain crate. Platform adapters should translate bounded requests into domain operations and translate results back.
- Separate semantic actions from transport envelopes. Replacing WebSocket with WebRTC, or IPC with JNI, should not change the reducer or action catalogue.
- Preserve the caller origin explicitly. Local-only operations must not become remote merely because both adapters call the same reducer.
- Prefer one semantic gateway per target over sockets or `invoke` calls scattered through UI components and subsystems.
- If a browser mirrors a Rust reducer in JavaScript because Wasm is not justified, use the same golden fixtures and differential tests. Similar type names or transitions are not evidence of parity.
- Treat timing tier as a target capability. Shared contracts and state-machine code do not make WebView, browser, desktop-audio, and XR output timing equivalent.

## Multi-Runtime Frontends

If the same UI must run in Tauri, a browser-only mock, and a hosted web application, define one typed product API and provide explicit adapters—for example native IPC, an in-memory test fake, and a REST/SSE client. Select the adapter once during bootstrap and make unavailable native behavior an explicit capability/result, not scattered `window.__TAURI__` checks or silent no-ops.

Keep the adapters contract-compatible with shared fixtures and contract tests. A mock should model error, cancellation, delay, event ordering, and lifecycle paths that affect the UI, while tests that claim native coverage still run the actual packaged application. Do not enable the global Tauri object merely to avoid importing the typed API package.

## Authoritative Live References

- Calling Rust from the frontend: <https://v2.tauri.app/develop/calling-rust/>
- Calling the frontend from Rust: <https://v2.tauri.app/develop/calling-frontend/>
- Tauri state management: <https://v2.tauri.app/develop/state-management/>

Confirm signatures and platform behavior against the version pinned in the repository.
