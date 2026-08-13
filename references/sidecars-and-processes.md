# Sidecars and Child Processes

Use this guide when a Tauri app bundles or launches an external executable, Node/Python runtime, CLI, local server, model worker, or other child process.

## Decide Whether a Sidecar Is Necessary

Prefer Rust in-process code when it is available, portable, and safe. A sidecar is appropriate when reusing a mature non-Rust stack, isolating a crash-prone component, shipping an existing CLI, or maintaining a separately testable worker.

Account for larger artifacts, per-platform builds, process supervision, IPC security, startup latency, signing/notarization, antivirus reputation, updates, and license notices.

## Package Artifacts

- Build one artifact for every target triple; a host build is not a release matrix.
- Follow the current Tauri sidecar naming convention, including target triple and Windows extension.
- List the logical binary path under `bundle.externalBin`; let the Tauri bundler select the target-specific file.
- Make artifact production deterministic and verify hashes in CI.
- Sign/notarize sidecars wherever the containing application requires it.
- Never download an executable at runtime without a separately authorized, signed, and rollback-capable update design.

JavaScript/TypeScript can be compiled into a self-contained binary so end users do not need Node installed. Compare the selected packager's maintenance, native-addon support, licensing, binary size, and target coverage. Embedding a runtime plus readable resources is a different tradeoff, not an equivalent security boundary.

## Permission the Launch

Initialize the shell plugin only when used and define a narrow sidecar permission:

- exact binary name;
- `sidecar: true` where required by the schema;
- fixed subcommands and validated argument patterns;
- only the windows/WebViews that need initiation authority.

Avoid unrestricted `args: true` for production. Prefer frontend calls to a typed Rust command that validates product-level input and constructs the sidecar arguments internally.

## Choose IPC

| Model | Best for | Main risks |
|---|---|---|
| argv + exit output | short one-shot operations | argument leakage, quoting, output size |
| stdin/stdout framing | long-lived ordered worker | framing, backpressure, restart state |
| local socket | structured/high-throughput worker | authentication, stale endpoints, permissions |
| localhost HTTP | reusable service stack | port exposure, origin/auth, CSRF-like calls |

Define message framing, maximum payloads, timeouts, backpressure, cancellation, protocol version, and safe error envelopes. Never treat binding to loopback as authentication. Generate an unguessable session secret or use an OS-protected local transport where cross-process callers are a risk.

For a sidecar that announces a random TCP port on stdout, treat startup as a bounded protocol: accept only one size-limited, strictly parsed handshake record; distinguish logs from control messages; require a per-launch secret on the socket; retain the child handle; fail if the process exits before readiness; and time out without holding application-state locks. Newline-delimited JSON is acceptable only with a maximum frame size and explicit malformed-frame policy.

## Supervise the Process

- distinguish spawn failure, protocol failure, non-zero exit, crash, timeout, and user cancellation;
- consume stdout and stderr without deadlocking pipes;
- bound retained logs and redact secrets;
- prevent accidental duplicate workers;
- restart only with bounded backoff and a clear state-recovery policy;
- terminate children on app exit and window-driven cancellation when appropriate;
- handle orphan processes after crashes.
- decide whether closing the last window means hide, keep background service alive, or quit; do not let platform defaults accidentally orphan or kill the worker.

Use `Result` throughout. Validate UTF-8 or support binary framing rather than unwrapping child output.

## Testing

- unit-test command construction and protocol parsing;
- run against a deterministic fake sidecar for timeout, malformed output, crash, and cancellation;
- verify the packaged app can locate the bundled binary on every target;
- inspect signatures, executable permissions, quarantine behavior, and antivirus results;
- test paths containing spaces and non-ASCII characters.

## Authoritative Sources

- Node.js sidecar tutorial: <https://v2.tauri.app/learn/sidecar-nodejs/>
- Embedding external binaries: <https://v2.tauri.app/develop/sidecar/>
- Shell plugin: <https://v2.tauri.app/plugin/shell/>
