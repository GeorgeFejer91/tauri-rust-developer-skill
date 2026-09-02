# Browser Companion Reliability and Security

Use this reference when a browser or phone must observe or control a local Rust/Tauri application with low perceived latency. It supplements [marionette-remote-control.md](marionette-remote-control.md): that reference defines the safe semantic action surface and native ownership model; this one concentrates on password pairing, remembered devices, connection recovery, command/state convergence, moving timelines, loopback bridges, bulk transfers, and qualification.

## Contents

- [Outcome and Non-Negotiable Invariants](#outcome-and-non-negotiable-invariants)
- [Separate the Planes](#separate-the-planes)
- [Choose the Transport Deliberately](#choose-the-transport-deliberately)
- [Define Two Generations](#define-two-generations)
- [Treat Connection as a State Machine](#treat-connection-as-a-state-machine)
- [Pairing and Password Protocols](#pairing-and-password-protocols)
- [Remember a Browser Without Remembering the Password](#remember-a-browser-without-remembering-the-password)
- [Make Commands Retry-Safe](#make-commands-retry-safe)
- [Reconcile Authoritative State](#reconcile-authoritative-state)
- [Give the Local Operator Bounded Priority](#give-the-local-operator-bounded-priority)
- [Harden a Browser-to-Loopback Bridge](#harden-a-browser-to-loopback-bridge)
- [Keep Interactive Work Off Slow Paths](#keep-interactive-work-off-slow-paths)
- [Design Bulk Transfer Beside Control](#design-bulk-transfer-beside-control)
- [Resource and Abuse Bounds](#resource-and-abuse-bounds)
- [Measure the User Path](#measure-the-user-path)
- [Qualification Gate](#qualification-gate)
- [Zuradio Audit: Defects Converted Into Rules](#zuradio-audit-defects-converted-into-rules)
- [Primary References](#primary-references)

## Outcome and Non-Negotiable Invariants

A dependable companion has one native authority and several replaceable adapters:

```text
local UI / CLI ---------\
                         -> one serial Rust authority -> revisioned public projection
loopback host WebView --/              ^                         |
                                      | typed commands            | snapshots/media
static companion -> rendezvous/auth -> transport adapter --------+
```

Keep these invariants explicit:

1. The Rust reducer, actor, or transactional service is the only state-transition authority.
2. Discovery/routing, transport encryption, peer identity, authorization scope, application consistency, and availability are separate layers.
3. The hosted companion contains executable UI assets only. Private data remains on the local app unless the user starts an authenticated live transfer.
4. A connection is not ready until mutual authentication and an initial authoritative projection complete. Media readiness may be later than control readiness.
5. Every callback, command, acknowledgement, grant, and state projection is fenced by an application-authority generation and, where applicable, a transport epoch.
6. Interactive control cannot wait behind catalog scans, media decoding, uploads, visualizers, optional audio reception, or database maintenance.
7. “Cross-platform,” “low latency,” and “reliable” describe measured boundaries, not architecture choices.

## Separate the Planes

Model at least four planes, even if one physical connection carries several:

| Plane | Owns | Must not imply |
|---|---|---|
| Distribution | Static HTML/CSS/JS and pinned assets | Backend, availability, identity, or private-data hosting |
| Rendezvous | Locating the currently active local app/session | Authentication or authorization |
| Control/state | Typed commands, acknowledgements, snapshots, heartbeats | Media delivery or file-transfer capacity |
| Media/bulk | Live media and bounded file chunks | Authority over queues, files, shell, or arbitrary native APIs |

A static host such as GitHub Pages is useful for distribution, but it is a static site service. Identify who operates signaling, WebSocket relay, TURN, authentication, revocation, rate limits, telemetry, and incident response. Do not hide these dependencies behind “serverless” wording.

## Choose the Transport Deliberately

| Architecture | Strengths | Costs and failure modes | Good default when |
|---|---|---|---|
| Direct WebRTC data/media | Low path length after ICE; browser-native audio/video; reliable ordered or partial/unordered data channels | Signaling and ICE lifecycle; restrictive NAT requires TURN; relay operations still exist; mobile/background behavior varies | Direct live media and no cloud-hosted collection are core product requirements |
| Authenticated WSS relay | Predictable browser support; centralized identity, revocation, audit, fan-out, and reconnect | Server RTT and bandwidth; trusted service stores or forwards traffic; bounded per-client queues required | Control/state reliability and operations matter more than direct media |
| Hybrid WSS + WebRTC | WSS can own discovery/auth/fallback while WebRTC carries direct media or latency-sensitive data | Most components and qualification paths | Production reachability plus direct media justify the operational cost |

WebRTC data channels use SCTP over DTLS and can be reliable/ordered by default or explicitly partially reliable/unordered. That protects channel traffic and delivery properties; it does not establish application identity or exactly-once effects. RFC 8835 requires TURN support for endpoint-dependent NAT scenarios, so a “works over the Internet” claim needs a configured relay and forced-relay tests, not only a same-network success.

Keep application envelopes transport-neutral. A VDO.Ninja adapter, raw WebRTC implementation, or WSS relay should be replaceable without changing the reducer, authorization scopes, command identity, revision rules, or UI semantics.

## Define Two Generations

Do not overload “session”:

- **Application-authority generation** changes when the local app restarts, broadcast/control is stopped, credentials rotate, policy changes, or a new authoritative run supersedes the old one. Grants and deduplication belong here.
- **Transport epoch** changes for every connection attempt or socket/data-channel replacement. Callbacks, pending sends, and liveness belong here.

A reconnect may create a new transport epoch while preserving the application-authority generation and command duplicate cache. A Stop or credential rotation changes authority and makes all old transports, grants, commands, and snapshots inert.

A bounded envelope should carry only fields needed by its kind, drawn from:

```text
protocol_version, message_kind
authority_generation, transport_epoch
principal_id, grant_id, scopes
message_id or command_id, sequence
expected_revision, target_precondition
bounded typed payload
```

Canonicalize the logical command when hashing or signing it. Do not include a fresh transport sequence in the duplicate fingerprint because an identical command retry may be re-enveloped on a new connection.

## Treat Connection as a State Machine

Use explicit states rather than one `connected` Boolean:

```text
idle
  -> discovering
  -> private_connecting
  -> proving
  -> snapshot_syncing
  -> control_ready
  -> media_ready                 (optional, may happen in parallel)
  -> degraded / reconnecting
  -> stopped
```

Requirements:

- Attach all listeners before announcing, joining, viewing, or opening dependent channels.
- Bind every event to the exact authority generation, transport epoch, peer, and channel instance. Late open/message/close events from an earlier attempt do nothing.
- Give discovery, private connection, proof, initial snapshot, and optional media separate deadlines and error codes.
- Start independent work concurrently: derive proof material while establishing the private route; tear discovery down asynchronously; attach optional media after controls are usable.
- Use capped exponential backoff with full jitter for transient reconnects. Reset after a stable interval, not every brief open.
- Explicit Stop, revocation, logout, permanent proof failure, or incompatible protocol disables reconnect.
- On reconnect, authenticate again, obtain the latest snapshot, reconcile pending commands by identity, and never replay raw UI events.
- Show `control ready`, `audio connecting`, `degraded`, `reconnecting`, and `stopped` honestly. Rendering controls before mutual proof plus state sync is not readiness.

## Pairing and Password Protocols

### Keep routing separate from the password

Avoid deriving a stable public room or global lookup key directly from a human password. It correlates reuse, exposes a target for online enumeration, and an observed proof can permit offline guessing depending on the construction. A clean design uses a nonsecret station/account identifier, QR/deep-link invitation, or saved device association for discovery, then performs password authentication on a fresh private route. The UI may still present only a Connect button and password dialog; hiding URLs does not require combining identity and password cryptographically.

Choose one reviewed pattern:

| Situation | Preferred pattern |
|---|---|
| High-entropy one-time invitation | At least 192 random bits, transcript-bound mutual HMAC proof, short expiry, one-use or explicit revocation |
| Symmetric peers sharing a human password | Reviewed SPAKE2 implementation with identities, transcript binding, and explicit key confirmation |
| Browser client authenticating to an owned account/service | OPAQUE or another reviewed augmented PAKE, plus server-authenticated TLS and online-guessing controls |

Do not invent a PAKE. Pin a maintained implementation and ciphersuite, use its exact wire format and test vectors, and obtain cryptographic review before claiming hostile-network resistance.

Every proof transcript should bind protocol/ciphersuite version, client and server roles/identities, station/account, application-authority generation, requested mode/scopes, route identity where appropriate, and fresh nonces. Require mutual confirmation before revealing a catalog, state snapshot, media route, or grant. Use uniform errors and rate limits so proof failure does not become an enumeration oracle.

Password storage and network authentication are different problems. For a server-side password record, follow current memory-hard KDF guidance such as Argon2id; use a unique salt per record and benchmark a defensible work factor. PBKDF2-HMAC-SHA-256 is a compatibility/FIPS-oriented fallback with a currently recommended work factor substantially above older 210,000-round profiles. A slow KDF does not turn a challenge/HMAC exchange into a PAKE.

If product constraints retain deterministic password discovery, state the limitation, require a generated high-entropy password rather than a memorable phrase, domain-separate each derived purpose, throttle public beacons, and make password rotation easy. Never present it as resistant to offline guessing.

## Remember a Browser Without Remembering the Password

“Do not ask again for 24 hours” should create a revocable device session, not store the password or long-lived derived password key.

Preferred server-side design:

1. On successful fresh authentication, the browser generates a device key pair with WebCrypto and stores the non-extractable private `CryptoKey` in origin storage.
2. Rust stores a device record containing the public key, allowed station/user, issue/expiry time, current credential generation, scopes ceiling, last-use metadata, and a hash of any opaque refresh secret—not the secret itself.
3. Reconnect begins with a fresh server nonce bound to the station, mode, authority generation, and transport epoch.
4. The browser signs a canonical transcript containing that nonce, credential identifier, requested mode, and short-lived proof ID. Rust verifies possession, expiry, revocation, and scope, then returns mutual confirmation and a new short-lived session grant.
5. Rotate refresh material after use where practical. Support per-device Forget, global revoke, password-change policy, and automatic expiry.

RFC 9449 DPoP is an instructive proof-of-possession design: bind a short-lived proof to key, request context, unique identifier, token hash, and optionally a server nonce. A companion does not need OAuth or JWT to reuse those principles.

A non-extractable WebCrypto key is stronger than a copied bearer token but is not hardware identity and does not defeat same-origin script injection: malicious script can ask the key to sign. Keep CSP/dependencies strict, minimize third-party script, and make revocation real. Browsers may clear origin storage, so loss must be recoverable through normal password pairing.

If a self-contained token in `localStorage` is the MVP choice, call it a short-lived bearer credential. Bind it to device metadata and fresh nonce proofs, expire it quickly, invalidate it on password/credential-generation change, and provide Forget; do not claim that a browser-generated device ID prevents theft when the ID and token can be copied together.

## Make Commands Retry-Safe

Reliable ordered transport still leaves the classic ambiguity: the authority may apply a command and the acknowledgement may be lost. The client must retry without duplicating effects.

Use these rules:

1. Generate a stable command ID once per semantic user action.
2. Namespace duplicate detection by `(application-authority generation, authenticated principal, command ID)`.
3. Store a canonical logical-payload fingerprint plus the exact accepted or rejected outcome.
4. An identical retry receives the cached outcome. The same ID with a different payload is a protocol violation.
5. Preserve the cache across transport reconnection while the authority generation remains valid.
6. Send `expected_revision` and target-specific preconditions for noncommutative operations.
7. On timeout, reconnect if needed and retry the same command ID. Do not manufacture a new ID for an ambiguous result.
8. Keep a bounded cache with an expiry at least as long as the maximum retry/session window; persist it when process-crash replay is in scope.

This provides at-most-once effects within the declared authority/cache boundary. Do not call it exactly once: if both the authority and durable duplicate record are lost, the old outcome may be unknowable.

Separate the cached action outcome from current public state. A duplicate acknowledgement may return the original result metadata, but any attached snapshot must be freshly projected. Never regress the client by replaying a historical snapshot from the original acknowledgement.

Order application writes through one owned sender or sequence discipline so an applied acknowledgement cannot be overtaken by a dependent older state frame. If the reliable queue cannot preserve order, fail the route rather than dropping one command and continuing.

## Reconcile Authoritative State

Every state and acknowledgement carries the application-authority generation and a monotonic revision. The client:

- rejects the wrong generation;
- ignores revisions older than the last accepted projection;
- accepts an exact duplicate idempotently;
- detects a revision gap when deltas are used and requests a full fresh snapshot;
- never converts an authoritative display snapshot back into a user command;
- keeps only the newest replaceable projection under pressure.

Prefer full bounded snapshots for low-rate application state. Use deltas only when measured size/frequency justifies the added repair protocol.

### Moving timelines

Playback position, animation, experiment timing, and similar state cannot be synchronized by repeatedly forcing a delayed snapshot value. Publish a timeline anchor:

```text
authority_generation
item_or_track_id
timeline_generation
status and playback_rate
position_at_anchor
authority_monotonic_timestamp or sender-relative anchor
revision
```

The client extrapolates locally while playing. Hard-seek only when item/timeline generation changes, a command explicitly changes position/status, or measured drift exceeds a product threshold. Ordinary queue, volume, metadata, or playlist snapshots must not rewind the timeline.

Keep a bounded pending local intent for seeks or scrubs until Rust acknowledges the matching item/timeline and position. Drop it on timeout, rejection, item change, or authority change. This prevents an older broadcast projection from snapping the UI/player back while preserving eventual authority.

## Give the Local Operator Bounded Priority

A local PC may outrank a remote controller, but “local always bypasses stale revisions” is too broad: a delayed old local event could apply long after its context disappeared.

Implement local priority through explicit causality:

- serialize all host-origin actions before sending them to Rust;
- give every local origin a monotonic sequence and bind it to the observed authority/timeline generation;
- include the selected track/item or other target precondition;
- allow rebase only inside a short declared causal window or against the immediately competing revision;
- never rebase a stale action onto a different target;
- publish the winning revision and require subsequent remote intent to use it.

Test both race orders. The desired rule is “the current local operator intent wins a concurrent conflict,” not “every historical local message is valid.”

## Harden a Browser-to-Loopback Bridge

The remote browser should not connect to the laptop loopback daemon. A local packaged/dedicated WebView or browser host may bridge browser-native WebRTC/Web Audio to Rust through a narrow local API.

For that local API:

- bind only to loopback on a random port;
- generate a high-entropy per-launch bootstrap token and put it in a URL fragment, not a query;
- consume the bootstrap token atomically once, exchange it for an `HttpOnly`, `SameSite=Strict` session cookie, erase the fragment, and rotate/expire failed or abandoned bootstraps;
- validate an exact parsed `Host`, reject non-loopback and unexpected ports, set no permissive CORS, and use restrictive CSP/security headers;
- for browser cookie authentication, require an exact `Origin` on every state-changing HTTP request and validate `Origin` during WebSocket upgrade; missing Origin is a rejection;
- allow CLI bearer authentication as a separate credential class that does not rely on browser Origin, with an explicit closed endpoint/action policy;
- enforce content type, body/frame size, rate, connection, and timeout limits;
- never expose remote paths, SQL, shell/process, arbitrary URL fetch, generic function names, or Tauri command lookup;
- keep the host WebView's Tauri capability empty or minimal. Rust remains authority even if same-origin host script is compromised.

Tauri's localhost plugin carries an explicit security warning. Prefer the custom protocol when browser-native localhost semantics are unnecessary; when a loopback server is required, threat-model local malicious processes, DNS rebinding, cross-site requests, restart races, and copied runtime files.

## Keep Interactive Work Off Slow Paths

Use one serial Rust authority actor with bounded, priority-aware inputs. The authority should validate and commit short transitions; it should not scan directories, parse media, hash large files, wait for codecs, or hold a `std::sync::Mutex` while doing SQLite/disk work on an async request path.

A robust shape is:

```text
high priority: auth completion, command, revoke, state repair
normal:        low-rate host reports and metadata commits
bulk:          upload chunks, scan/recognition progress

slow worker -> validated prepared result -> short authority commit
```

Bound every mailbox by item count and bytes. Shed visual/diagnostic cadence first. Never drop a discrete control command silently. Track queue age, not only depth; a short queue can still violate the latency budget.

Create readiness milestones from the dependency graph. For example, controls can become ready after private transport, proof, and initial snapshot while audio receiver, cover art, analyzer, and visualizer continue independently.

## Design Bulk Transfer Beside Control

Correctness baseline:

- declare bounded files/relative paths/sizes;
- reject traversal, special files, unsupported types, and duplicate IDs;
- stream into a private staging area;
- verify ordered offsets and a final per-file digest;
- parse and validate before an atomic move/commit;
- define whether files commit independently or as a batch;
- remove only incomplete staging on abort/restart;
- publish completed files through the authority without blocking subsequent control.

Performance baseline:

- slice `Blob` data incrementally instead of `arrayBuffer()` on the whole file;
- send binary `ArrayBuffer` frames rather than base64 where supported;
- use a bounded sliding window, not stop-and-wait for every tiny chunk;
- pause at a `bufferedAmount` high-water mark and resume on `bufferedamountlow` or equivalent transport pressure;
- adapt chunk/window sizes from measured RTT and receiver write latency within fixed safety caps;
- checkpoint `(transfer, file, offset, rolling/final digest)` for resumability when large or unreliable-network transfers justify it;
- keep transfer identifiers and grants scoped, expiring, and revocable.

Measure interactive command acknowledgement and state freshness during a saturated upload. Maximum throughput is not a success if controls become sluggish or authentication times out.

## Resource and Abuse Bounds

Set explicit caps for unauthenticated peers, authenticated peers, grants, sessions, messages, nesting, strings, collection sizes, proof attempts, commands per second, pending reliable bytes, transfer bytes, transfer duration, and idle time. Acquire admission before spawning per-peer work and release it on every error/timeout path.

Replace an old grant for the same authenticated owner explicitly and revoke its Rust record; removing only a JavaScript map entry is not revocation. Bound retained nonces, command outcomes, and telemetry with expiration. Rate-limit public discovery and password proof independently so one cannot starve established control.

Never log passwords, derived keys, tokens, grants, full route identifiers, SDP, ICE credentials, file contents, or unredacted paths. Log sanitized state transitions, authority/transport correlation IDs, scope, route class, durations, byte counts, queue pressure, error codes, and revocation reason.

## Measure the User Path

Define separate latency endpoints:

```text
password submit
  -> discovery
  -> private channel
  -> mutual proof
  -> first authoritative snapshot
  -> control ready

command gesture
  -> encoded/sent
  -> Rust accepted or rejected
  -> acknowledgement received
  -> authoritative UI rendered

media requested
  -> receiver route ready
  -> first packet
  -> first playable sample
```

Report p50, p95, p99, worst, sample count, direct/relay route, current RTT where available, reconnect recovery, stale/gap/drop counts, control latency during upload, CPU, memory, and test hardware/runtime. Use `RTCPeerConnection.getStats()` for selected candidate-pair type, current RTT, and byte counters when supported; do not expose sensitive ICE material.

Regression ceilings are useful gates but are not targets. Optimize the common path only after preserving proof strength, authorization, state correctness, and cleanup.

## Qualification Gate

Test layers independently and then together:

1. Pure reducer: scopes, revisions, target preconditions, local-priority window, duplicate IDs/fingerprints, exact cached outcomes, authority rotation.
2. Protocol: mutual proof, role/mode mismatch, replay, expired/revoked device, wrong generation, stale/duplicate/gap state, acknowledgement loss and same-ID retry.
3. Loopback: host/origin parsing, missing Origin with browser cookie, CLI credential separation, one-use bootstrap, body/rate/connection limits, restart race, unauthenticated state/media denial.
4. Transport: direct and forced TURN, callbacks from stale epochs, data before listener readiness, partial channels, signaling outage, relay failure, reconnect storms, explicit Stop.
5. Browser UI: real click-to-authority workflows, responsive/mobile layout, background/foreground, file and folder pickers, autoplay policy, codec behavior, passwordless reconnect.
6. Load: upload plus commands, multiple listeners, bounded queues, slow receiver, malformed messages/media, disconnect at every transfer stage.
7. Installed bytes: actual daemon/package/WebView or dedicated browser shell against the published companion, not a development substitute.

Run Chromium, Firefox, and WebKit automation where applicable, then physical Android Chrome and iOS Safari plus the exact Windows WebView2, macOS/WKWebView, Linux WebKitGTK, or dedicated Chromium runtime shipped. Browser-engine automation does not qualify a platform WebView build or physical phone lifecycle.

Preserve failure artifacts, exact commands, dependency versions, test data provenance, percentile measurements, and intentionally open gates. Never weaken a gate to fit an implementation.

## Zuradio Audit: Defects Converted Into Rules

Audit baseline: [Zuradio commit `89d927439d532cdc5e2ef420d8e939a2db30d4b8`](https://github.com/GeorgeFejer91/zuradio/tree/89d927439d532cdc5e2ef420d8e939a2db30d4b8), reviewed 2026-09-02. The app is a useful implementation case study, not a generic protocol standard.

### Proven strengths at the audited commit

- GitHub Pages contains only the static companion; music, catalog, password, media URL, and Rust authority stay on the laptop.
- CLI, local host UI, and remote controls converge on one closed Rust action schema and revisioned state.
- The host uses fresh private control/audio routes after deterministic password discovery; Rust verifies mode/peer/session/epoch-bound proof, returns a server proof, and mints listen/control/upload-scoped grants with monotonic request sequences.
- A 24-hour browser credential avoids storing the raw password, uses fresh nonce-bound reconnect proof, expires, survives daemon restart, supports Forget, and invalidates when the password changes.
- Control readiness no longer waits for optional audio attachment; proof derivation and transport setup overlap; direct ordered data channels avoid polling.
- Local seek intent is bound to the selected track and held against unrelated stale projections, eliminating the observed snap-back defect.
- Uploads enforce declarations, ordered 8 KiB chunks, per-file SHA-256, private staging, per-file parse/organized commit, and cleanup; metadata precedence and user overrides are explicit.
- The full installed/public path passed real independent browser contexts. The recorded smoke timings were about 3.894 seconds from controller password to ready, 5.996 seconds for listener password to live, and 165 ms for one acknowledged remote command. These single-run figures are useful regression evidence, not percentile guarantees.
- Rust, CLI, browser, installed-byte, companion-only artifact, format, permission, responsive UI, and public Pages gates passed. Defects such as JavaScript-unsafe random epochs, proof asymmetry, early ready state, early-but-unwritable channels, host mutation races, service readiness races, and missing WebKitGTK WebRTC were found by end-to-end qualification rather than compilation.

### Residual findings and generalized corrections

| Area | Audited status | General correction |
|---|---|---|
| Password discovery | Deterministic route and proof use one fixed application salt with PBKDF2-HMAC-SHA-256 at 210,000 rounds; documentation correctly admits offline guessing for weak passwords | Separate station routing from password; adopt reviewed PAKE; use current password-storage work factors only for record storage; require generated high entropy if retaining the MVP protocol |
| Remembered browser | Tested 24-hour signed bearer plus browser-generated device ID in origin storage | Prefer server-side per-device record and non-extractable-key proof of possession; add per-device revocation/rotation; keep bearer wording if storage can be copied |
| Command retry | Rust caches by command ID, but the browser times out without automatic same-ID retry; cache is not namespaced by principal/generation and does not fingerprint payload | Apply the retry-safe command contract above and test applied-but-ack-lost behavior across reconnect |
| Snapshot ordering | Companion assigns pushed revisions without rejecting old generation/revision or repairing gaps | Fence every projection; ignore stale, request fresh state on gaps, and preserve acknowledgement/state causality |
| Local priority | Local player actions may bypass any stale revision; track-bound seeks prevent the worst target error | Add local origin sequence/generation and a bounded causal rebase window so delayed old local messages cannot win forever |
| Loopback bootstrap | Fragment token is high entropy and cookie exchange is protected, but the audited launch token remains reusable for the process lifetime; missing Origin is accepted to accommodate non-browser callers | Atomically consume bootstrap once; separate CLI bearer endpoints from cookie browser endpoints and require exact Origin for the latter, including WebSocket upgrade |
| Grant/admission lifecycle | Grants are scoped, expiring, peer-bound, sequenced, and Stop-revoked | Add hard total/per-principal grant caps, proof/message token buckets, exact replacement revocation, and leak tests across every error path |
| Authority concurrency | Rust owns policy, but synchronous mutex/SQLite work can serialize slow operations with control | Move authority to a bounded actor; parse/hash/scan off-authority and commit prepared results briefly; instrument queue age |
| Upload performance | Integrity and incremental publication are strong; whole-file browser reads, base64, and 8 KiB stop-and-wait yielded only about 0.25–0.30 MB/s from the recorded real-upload sizes and elapsed times | Stream binary slices with bounded adaptive window/backpressure and resumable checkpoints; gate command latency under upload |
| Internet/runtime coverage | Chromium contexts and installed Linux Chromium shell passed; WebKitGTK runtime probe correctly rejected a build lacking WebRTC | Add physical Android/iOS, Windows WebView2, macOS WKWebView, forced TURN, multiple listeners, background/network handoff, and soak before broad release claims |

### Durable defect-to-rule mapping

- A channel `open` event did not guarantee a successful first write: retain proof material until send succeeds, install listeners early, and make handshake sends retryable within one fenced epoch.
- A controller appeared connected before mutual proof and initial state: readiness is a protocol milestone, not a socket property.
- Rust generated `u64` epochs exceeded exact JavaScript integer range: specify cross-language integer ranges or encode large identifiers as strings/bytes.
- Host actions raced automatic playback: serialize each origin before the authority and keep target preconditions in the command.
- Reapplying ordinary snapshots rewound playback: moving state needs a timeline anchor and pending-intent fence, not repeated absolute positioning.
- The installer scanned before the service published credentials: launch success requires an authenticated readiness probe, not process existence or a fixed delay.
- WebKitGTK exposed a setting while the distro build lacked working WebRTC: probe the exact packaged runtime capability and exercise the installed bytes.
- Late listeners missed current state: every newly authenticated observer receives a fresh bounded snapshot after authorization; do not depend on an earlier broadcast.
- Password derivation, private connection, and optional audio were serialized: derive a dependency graph, overlap independent work, and define control-ready separately from media-ready.

## Primary References

- [WebRTC Recommendation](https://www.w3.org/TR/webrtc/) and [WebRTC Statistics](https://www.w3.org/TR/webrtc-stats/)
- [RFC 8831: WebRTC Data Channels](https://www.rfc-editor.org/rfc/rfc8831.html) and [RFC 8835: Transports for WebRTC](https://www.rfc-editor.org/rfc/rfc8835.html)
- [RFC 9807: OPAQUE](https://www.rfc-editor.org/rfc/rfc9807.html) and [RFC 9382: SPAKE2](https://www.rfc-editor.org/rfc/rfc9382.html)
- [RFC 9449: Demonstrating Proof of Possession](https://www.rfc-editor.org/rfc/rfc9449.html) and [RFC 8785: JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785.html)
- [Web Cryptography Level 2](https://www.w3.org/TR/WebCryptoAPI/)
- [OWASP Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) and [WebSocket Security](https://cheatsheetseries.owasp.org/cheatsheets/WebSocket_Security_Cheat_Sheet.html)
- [Tauri runtime authority](https://v2.tauri.app/security/runtime-authority/), [capabilities](https://v2.tauri.app/learn/security/capabilities-for-windows-and-platforms/), and [localhost warning](https://v2.tauri.app/plugin/localhost/)
- [GitHub Pages static hosting boundary](https://docs.github.com/en/pages/getting-started-with-github-pages/what-is-github-pages)
