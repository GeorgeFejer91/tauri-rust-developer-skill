# Networking, Transfers, and Embedded Servers

Use this reference for HTTP, WebSocket, downloads/uploads, OAuth callbacks, local servers, proxies, TLS, and network-facing sidecars.

## Pick the Owner First

Use browser `fetch`/WebSocket when ordinary WebView networking, browser security rules, and a public API are sufficient. Use Rust (`reqwest`, a Tauri plugin, or an application service) when you need native TLS/proxy behavior, client certificates, streaming to disk, secret-bearing headers, strict host policy, or one connection shared across windows.

Never place durable API secrets in frontend code. A Tauri capability controls whether a WebView can call a plugin; it does not make credentials in JavaScript secret.

## HTTP

The official HTTP plugin exposes a Rust-backed fetch API and requires URL scopes for frontend callers.

- Allow exact HTTPS hosts and paths where practical; add deny rules for sensitive subdomains.
- Treat redirects as new destinations. Bound redirect count and revalidate the final host when policy matters.
- Set connect, request, and idle timeouts; support cancellation.
- Bound response size before buffering; stream large bodies.
- Validate status, content type, schema, and integrity before use.
- Retry only transient, safe/idempotent operations with capped exponential backoff and jitter. Respect server retry guidance.
- Avoid the `unsafe-headers` feature unless the requirement and threat review justify it.
- Respect the system proxy by default; make bypass/custom proxy behavior explicit and test authenticated proxies separately.
- Keep certificate validation enabled. Certificate pinning adds rotation and recovery obligations; use it only with a documented key-rotation plan.

For OAuth/OIDC, prefer system-browser authorization plus a validated loopback or deep-link callback, PKCE, state, nonce, short-lived tokens, and Rust/keychain-owned refresh tokens. Never embed a client secret that is expected to remain secret in a distributed desktop binary.

## WebSockets and Long-Lived Streams

The WebSocket plugin grants a frontend the ability to open Rust-client connections; its default permission must still be scoped and reviewed.

- Define connection states: idle, connecting, open, backoff, closing, and closed.
- Authenticate without leaking bearer tokens into logs or URLs.
- Validate every inbound message and bound message/frame size.
- Use heartbeat/idle detection and capped reconnect backoff with jitter.
- Stop reconnecting on explicit logout, app shutdown, or permanent authentication failure.
- Bound queues and expose dropped/coalesced-message policy. A slow WebView must not create unbounded Rust memory growth.
- Remove listeners and close the socket when the owning window/component disappears.

## Uploads and Downloads

The upload plugin reads/writes paths with progress callbacks; file and URL authority are both security boundaries.

- Require explicit user intent for destination/source selection.
- Canonicalize and authorize paths in the Rust/plugin scope actually doing the I/O.
- Write downloads to a temporary sibling, enforce length limits, optionally verify a cryptographic digest/signature, then atomically rename.
- Never execute/open a downloaded file automatically without a separate explicit decision and type/source validation.
- Support cancellation, partial-file cleanup, resumability only with server validators, and crash recovery.
- Throttle progress events; do not emit one IPC message per network chunk.

## Embedded Local Servers

Prefer Tauri's default custom protocol. The official localhost plugin explicitly warns that serving packaged assets over localhost creates considerable security risk.

If an embedded server is unavoidable:

- bind loopback only, select and communicate the port safely, and handle port collisions;
- use an unguessable per-launch token and validate `Host`/origin where applicable;
- expose the smallest route set, no directory listing, and no unauthenticated privileged API;
- apply request/body/concurrency limits and graceful shutdown;
- assume any local process and browser page can attempt requests to it;
- test DNS rebinding, cross-origin requests, CSRF-like navigation, and restart races.

For frontend assets, use the localhost plugin only when a concrete WebView compatibility constraint outweighs the enlarged attack surface. Do not use a local HTTP server merely to reproduce an SSR deployment model.

## FFI and Shared Native Libraries

Prefer a pure Rust crate for shared domain logic, a Tauri plugin for a reusable native/WebView API, and a sidecar for fault isolation or non-Rust runtimes. Use direct FFI only when an in-process native SDK is required.

- Put `unsafe` declarations in a small adapter crate with safe domain methods.
- Pin ABI/library versions, document ownership and thread-affinity rules, and verify architecture-specific packaging.
- Convert callbacks into bounded channels; prevent callbacks after teardown.
- Never unwind Rust panics across FFI; convert errors at the boundary.
- Test missing/wrong-architecture libraries and native crashes. If a crash must not kill the app, use a sidecar instead.

## Network Verification

Test offline startup, DNS failure, TLS failure, proxy authentication, timeout, cancellation, redirect to a disallowed host, oversized/malformed data, server throttling, reconnection storms, sleep/wake, network changes, and shutdown during I/O. Logs may include operation IDs, hosts after redaction, status classes, durations, and byte counts—not tokens, cookies, request bodies, or user file contents.

