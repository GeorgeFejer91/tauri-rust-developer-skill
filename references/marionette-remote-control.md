# Marionette Remote Control for Tauri v2

Use this guide when a phone or another browser should act as an explicit, app-specific remote controller for a Tauri application. Also read [tauri-architecture.md](tauri-architecture.md), [tauri-security.md](tauri-security.md), [networking.md](networking.md), and [verification.md](verification.md). Read [latency-critical-systems.md](latency-critical-systems.md) when control latency, jitter, clock alignment, or deadline behavior is a product requirement.

## Define the Boundary Before the Transport

A Marionette socket is an opt-in session for typed product semantics, not remote desktop access. The remote peer may request allowlisted operations such as `presentation.next`, `player.seek`, or `scene.pointer`; it must not send JavaScript, selectors, DOM events, shell strings, paths, URLs, credentials, or generic command names for dynamic dispatch.

Use four distinct owners:

```text
external browser / phone companion
          | authenticated, bounded application messages
local settings WebView: transport + protocol bridge
          | narrow typed invoke/channel boundary
Rust domain service: authorization + authoritative reducer
          | versioned authoritative snapshot
local and remote presentation adapters
```

- The companion owns touch UI and replaceable local intent.
- A local, packaged WebView may own the browser-native VDO.Ninja/WebRTC or WebSocket connection and BRSP framing.
- Rust owns privileged state, accepted scopes, action validation, revisions, and application side effects.
- Each renderer projects authoritative state into its own viewport. Presentation state is not automatically new control intent.

Do not make the network handler mutate DOM controls and then read them back as application state. Do not feed returned authoritative display placement into the next upstream local-intent frame; that creates a feedback loop.

## Separate Public Discovery From Private Control

A public discovery plane may advertise a bounded target label, transport route, protocol/profile version, and short-lived invitation locator. It must not carry the pairing secret, proof, accepted grant, participant/private state, or native owner handle. Treat a discovered room, stream, peer UUID, or beacon as a routing hint; only the private authenticated control session can establish identity and authority.

A static HTTPS companion page may distribute reviewed, hash-pinned SDK and application bytes without constructing the SDK, joining a room, or opening a network connection on load. Construction and networking still require a current explicit Start/Connect gesture, and Stop/page hide must quiesce producers before disconnecting. Publicly reachable bytes or discovery infrastructure prove neither authenticated authority nor an availability, privacy, relay-capacity, or support SLA; record which signaling/TURN/discovery service the product actually operates or depends on.

## Make the App Remote-Ready, Not Remote-Open

For a new app, design one internal semantic action boundary even if networking is not yet shipped. Local UI, an optional local CLI, automation tests, and a future remote companion can adapt into that boundary without sharing their parsers or authority:

```text
src-tauri/src/
  domain/actions.rs       typed full application action enum
  domain/reducer.rs       authoritative validation and state transition
  remote/contract.rs      strictly smaller remotely eligible DTO/action subset
  remote/session.rs       activation, peer grant, lease, revocation, audit metadata
  remote/mod.rs           feature-gated coordinator; no listener at construction
src/remote/
  profile.ts              bounded manifest and exact frontend validators
  bridge.ts               narrow invoke/channel adapter
  transport.ts            replaceable BRSP/VDO/WebSocket adapter
```

Treat “remote CLI” as a semantic command catalog, not as access to `argv`, a shell, or a text command interpreter. Each remotely eligible action should declare:

- its stable action/version token and exact payload/result schema;
- required scope and whether local confirmation is required;
- reliable-command or replaceable-intent semantics;
- expected-revision and idempotency behavior;
- rate, byte, and collection bounds;
- stale/lease behavior and a safe neutral state when momentary;
- sanitized audit fields and errors.

Model local arming or operator approval as a generation-scoped safety acknowledgement, not a Boolean that survives every configuration change. A material package, participant/setup, device-route, output, or calibration change should revoke it. Remote grants, reconnects, and replayed commands must not inherit or recreate local arm authority.

The remote contract should be a strict subset or deliberate projection of the domain action enum. Map it with exhaustive Rust matches; never accept a string and look up an arbitrary command, Tauri invoke name, function, plugin, or CLI parser. The same pure reducer may serve local UI, tests, and remote requests only after each adapter has applied its own authentication and authorization policy.

A manifest can let the phone render compatible buttons, sliders, toggles, and joysticks, but it is descriptive, versioned data. It cannot grant a scope, lower a bound, select a native function, or prove identity. Keep the remote feature disabled by default, construct transports inertly, and require an explicit local activation path before any discovery, listener, server, or peer connection starts.

## Choose Native Authority or Presentation-Only Mode

For a real controller, route every accepted operation through a pure Rust reducer or service method. Keep the Tauri command thin:

```text
deserialize -> structural limits -> session/scope check -> allowlisted reducer
            -> revisioned result -> sanitized acknowledgement/snapshot
```

Represent scopes and actions with enums or exact matches. Include session ID, connection epoch, sender sequence or command ID, requested scope, expected revision when needed, and a bounded typed payload. Reject unknown variants, non-finite numbers, unsafe field names, oversized values, stale epochs, duplicate transitions, and actions outside the Rust-owned grant. Never trust a frontend `authenticated: true` flag as authorization.

Use presentation-only mode when the remote result should affect only what the settings WebView displays. Native snapshots may flow out to the bridge, while returned shared scenes terminate at a renderer and have no `invoke` path into Rust. Document this boundary and test that remote frames cannot alter durable state, device output, files, settings, markers, history, or native stream publication.

Observation authority is separate from mutation authority. A read scope must gate every state-bearing path: the initial snapshot, requested snapshots, live publication, state heartbeats, and any snapshot embedded in an acknowledgement. When a peer has mutation scope but no read scope, return only the bounded command/result metadata needed to acknowledge that operation.

When the transport and native mutation seam are both new, qualify an authenticated read-only observer first: grant only the read scope, advertise no mutating actions, and publish a sanitized authoritative snapshot. This smaller slice exercises packaged WebView-to-hosted-browser transport, proof, route diagnostics, stale behavior, Stop, and lifecycle without creating a second mutation authority. Add mutation only after that path is qualified; do not quietly route observer traffic through a local-action API.

## Activation, Pairing, Proof, and Revocation

Remote networking must be inert on page load. A defensible session flow is:

1. The user opens a dedicated local settings/control window and explicitly enables Host or Broadcast.
2. The app creates a fresh session ID, connection epoch, peer identity, and high-entropy pairing secret. For direct HMAC proof, use at least 192 random bits; do not substitute a six-digit code without a reviewed PAKE or trusted invitation service.
3. Transfer invitation material through an intentional channel such as a QR code, fragment-bearing companion URL, or authenticated backend. Do not put bearer secrets in query strings, telemetry, referrers, crash reports, or routine logs.
4. Both peers exchange bounded hello messages containing roles, nonces, epochs, capabilities, and requested/grantable scopes, then prove possession of the secret over one canonical transcript with role-bound HMAC-SHA-256.
5. Both independently compute the capability/scope intersection. The local UI shows the peer and requested authority; Rust records only the accepted, expiring grant.
6. Enable application messages only after mutual proof, matching negotiation, and any required local Accept action. Return a fresh authoritative snapshot rather than replaying an unbounded history.
7. Stop and revoke on explicit Stop, logout, invitation expiry, application shutdown, destroyed owner window, unrecoverable protocol error, or policy change. Reconnect creates a new epoch and repeats proof; late frames from an earlier epoch remain invalid.

Bind active ownership to the complete authenticated generation, such as `(session, epoch, controller, owner token)`. Disconnect and `Drop` cleanup must compare-and-clear that exact identity. An old socket must not revoke a replacement controller, overwrite a rotated session, or change a disabled target back to a waiting state.

Invalidation is an application transition, not just credential cleanup. Disabling remote access, rotating pairing material, or changing granted scopes/configuration must atomically drive an active remotely owned operation to its documented safe state before the owner and lease are cleared. Test these races while the target is active, not only while idle.

The WebView can own connection and framing without owning native authorization. Where remote actions can mutate native state, have Rust retain the proof policy, independently canonicalize the bounded hello objects, verify/generate role-bound proofs, and only then mint a short-lived grant bound to peer, epoch, scopes, and owning window. If a presentation-only profile keeps proof entirely in browser code, give that WebView no native mutation capability and document the lower-trust boundary. Never expose a command that lets arbitrary WebView data manufacture or widen a grant.

A minimal native-authority IPC contract can use four fixed operations: local-only `remote_begin_session(profile)` creates Rust-held session/secret/epoch state and returns bounded public invitation material; `remote_verify_peer(transcript, peer_proof)` independently canonicalizes the transcript, verifies the peer's role-bound proof, generates and returns the local role-bound proof for the bridge to send, and stores an opaque expiring grant bound to the caller window; `remote_apply_action(grant_handle, typed_action)` rechecks that stored grant, scope, epoch, revision, and action; and `remote_stop_session(grant_handle)` revokes and neutralizes it. The verify result should be an exact DTO such as `{ local_proof, grant_handle, accepted_scopes, expires_at }`, and the WebView must not expose the handle to the remote peer. Do not return the pairing secret after initial handoff, accept a frontend-supplied grant object, or let a grant handle select arbitrary scopes. Define exact DTOs, proof ordering, and secret zeroization/lifetime for the pinned application rather than copying these names blindly.

If a bundled WebView retains the browser-native handshake, every mutation-capable path still needs an owner-fenced native seam equivalent to `claim`, `renew`, `dispatch`, and `revoke`. Claim only the currently enabled session and a scope subset allowed by local policy, then have Rust mint a fresh opaque owner token bound to the exact window, session/epoch, controller, and scope set. Renew only after a fresh canonical control sequence and use a Rust monotonic deadline. Dispatch a closed typed command through the remote reducer origin—never through the local dispatcher—and recheck owner, scope, sequence, revision, action, and current preconditions. Revoke or deadman expiry must compare-and-clear that exact owner and pause or neutralize active output atomically. A late renew, dispatch, close, or page-hide from an old owner must be inert against its replacement. This pattern preserves native safety fencing; it does not turn browser-side proof into a native attestation, so document that trust choice or move proof verification into Rust when the threat model requires it.

Public labels, room names, discovered stream IDs, peer UUIDs, and successful transport connection are routing metadata, not identity. Production account identity, invitation expiry/revocation, abuse controls, and audit policy are separate product services.

## Separate Reliable Control From Replaceable State

Do not give every message the same delivery semantics.

| Lane | Examples | Required behavior |
|---|---|---|
| Reliable control | hello/proof/ready, discrete button action, acknowledgement, revoke, snapshot request | ordered, bounded queue, explicit result/error, idempotency or duplicate cache |
| Replaceable intent/state | joystick, pointer, slider, current affect state, live scene | complete current value, newest-only backpressure, sequence/epoch checks, heartbeat and stale policy |

Use one duplex peer connection or socket unless isolation or measurements justify more. A separate reverse media stream is unnecessary for ordinary state return.

When a transport opens separate control and state channels, install SDK/channel listeners before connect, join, announce, or view. The peer may deliver either channel before the corresponding `openChannel` promise returns or before the other lane and application consumer are ready. Bind every event to the selected stream, peer UUID, connection generation, and exact channel instance; retain only a bounded pre-open reliable-control queue and the newest pending state. Enter application-ready only after all required lanes are bound, then drain reliable control in order and publish only the latest state. Reject overflow, duplicate lanes, wrong-peer channels, and late open/message/close events from an earlier generation.

For continuous input:

- send normalized semantic values, not raw `pointermove`, client pixels, DOM targets, touch trajectories, or event timestamps;
- coalesce at a bounded cadence and retain at most the newest pending frame when the transport is congested;
- use receiver-local monotonic time for freshness, plus connection epoch and unsigned sequence for ordering;
- send an unchanged heartbeat only as often as liveness requires;
- define stale, holding, recovering, and closed states visibly;
- require several fresh frames if the product needs recovery hysteresis.

Momentary controls need a target-enforced lease/dead-man rule. Each accepted frame renews a bounded lease; release sends an explicit neutral value, and expiry forces the Rust authority to the documented safe state. Do not let a backgrounded phone leave a control pressed indefinitely. Discrete nonreplaceable operations remain reliable commands with an acknowledgement; they must not travel on a lossy latest-state lane.

Namespace application-command deduplication by authenticated principal/origin plus command ID and the target's application-authority generation, not by socket. Fingerprint the logical command body, not a fresh transport-envelope sequence, because an identical retry may be re-enveloped after a lost acknowledgement or transport reconnect. Return the exact cached acceptance or rejection for an identical retry and reject reuse of the same ID with a different payload. Preserve the cache across transport recovery while application authority remains valid; rotate or clear it only when that application authority generation is invalidated. This prevents one controller from pre-seeding IDs used by a local operator or another controller while preserving at-most-once effects across reconnect.

For an ordinary interactive starting profile—not a timing guarantee—BRSP uses a 100 ms active-intent heartbeat, 500 ms target lease, 250 ms authoritative-state heartbeat, 2,000 ms stale threshold, and three accepted frames before clearing the stale presentation. Change these only from product consequences and measured browser/network behavior; keep the lease enforced by native/target authority, and never claim real-time performance from the constants alone.

Bound encoded bytes, nesting, collections, strings, rates, pending reliable bytes, participant count, and concurrent sessions. Under overload, shed visual cadence and optional diagnostics before authoritative control. Close or visibly degrade when reliable command integrity cannot be maintained.

## Keep the Tauri Boundary Narrow

A WebView bridge normally needs only a few product operations, for example:

- create/stop a remote session and obtain public invitation material;
- submit a verified typed remote action or complete current intent;
- subscribe to a bounded authoritative snapshot/channel;
- report sanitized connection health and route class.

Prefer one typed wrapper over scattered raw `invoke` calls. Commands should never expose `eval`, arbitrary DOM dispatch, global keyboard/mouse injection, unrestricted shell/process execution, arbitrary filesystem/database access, credential retrieval, generic URL fetch, or dynamic module/function lookup. Do not enable the global Tauri object to simplify the bridge.

For high-rate native state, place a bounded coalescing adapter before a Tauri channel. A slow, reloaded, hidden, or destroyed settings WebView must not create unbounded native memory or corrupt the authoritative service. Retain cancellation handles, close listeners/channels on teardown, and make the current low-rate snapshot queryable after reattachment.

## Window Capabilities and CSP

Use a dedicated local WebView label for remote settings and grant it only the commands/events needed by the bridge. Keep ordinary content windows and any remote-origin WebView outside that capability. Tauri application scopes are not self-enforcing; resolve and enforce them in Rust.

Prefer a locally vendored, pinned transport SDK with reviewed license and integrity over a runtime `latest` script. Keep `script-src 'self'` when possible. Add only the exact signaling, credential, or API origins to `connect-src`; do not add `*`, `unsafe-eval`, arbitrary remote navigation, broad HTTP-plugin access, or permissive development origins to make networking work.

WebRTC ICE traffic and firewall behavior are not fully described by CSP `connect-src`. Test the packaged app on every WebView engine and network class. A restrictive CSP does not replace transport authentication, and a Tauri capability does not make JavaScript-held credentials secret.

Never load the phone companion or other untrusted remote content inside a privileged application WebView. Host it separately over HTTPS and open links in the system browser through a narrow validated operation.

## Select the Transport Deliberately

### VDO.Ninja data-only WebRTC

Use the reviewed SDK as a replaceable WebRTC adapter when serverless peer discovery, NAT traversal, and direct-or-relayed peer data are useful. Create no media stream, request no camera/microphone permission, and explicitly disable audio/video. One peer may `announce`, the other may `view`, and their data channel is duplex. Use the SDK API rather than freezing its private signaling WebSocket protocol.

VDO.Ninja still depends on hosted signaling and normally STUN; restrictive networks may require TURN. Two devices on the same Wi-Fi may negotiate a direct ICE route after signaling, but this is not proof of an offline LAN connection. Record the selected candidate/relay route without logging SDP, ICE credentials, or pairing material. A fully offline Wi-Fi product needs an independently designed discovery/signaling service and the same authentication, origin, TLS, and abuse controls.

### WebSocket

Prefer an authenticated product WebSocket when the application already has accounts, centralized authorization, audit, revocation, durable invitations, or many observers. Use TLS, explicit session authentication, bounded per-client queues, newest-only coalescing for live state, server-enforced scopes/rates, reconnect backoff, and a defined source of authority. Central fan-out is often simpler than peer mesh.

Keep the socket in Rust when native proxy/client-certificate behavior, protected credentials, or a connection shared across windows is required. A WebView socket is appropriate for a public browser-compatible protocol when no durable secret must be hidden from the renderer.

### Raw WebRTC

Choose raw WebRTC only when owning signaling, ICE/TURN configuration, protocol stability, and operational infrastructure justifies the added work. Keep BRSP/application envelopes transport-neutral so a VDO adapter can later be replaced without changing reducers or UI semantics.

Do not introduce an unauthenticated localhost or LAN server as a shortcut. Local processes and browser pages can attack loopback services; a network listener needs explicit user intent, authentication, origin/host checks, limits, lifecycle ownership, and firewall/installer qualification.

## Multiple Peers and Shared Scenes

Start with one controller and one target. Multiple controllers require an explicit Rust-owned policy: single active lease, per-scope ownership, priority, queueing, or deterministic combination. Never accept last-writer-wins accidentally.

For a Party/shared-scene profile, maintain one independently authenticated connection per guest. The host receives each guest's latest bounded offer, constructs one stable ordered authoritative aggregate, renders it locally, and fans the identical versioned aggregate to every connected guest. A guest accepts a scene only when its own fresh identity appears and the authenticated host connection owns that scene. Receiving a roster must not grant access to other guests or invitation authority.

“Same scene” means the same semantic aggregate. Different viewport sizes, device-pixel ratios, frame scheduling, and a phone-only local camera can produce different pixels. Keep pan/zoom local unless synchronized camera state is an explicit product field.

## Lifecycle and Failure Behavior

Define behavior for owner-window reload/destruction, phone background/lock, network loss/change, sleep/wake, signaling outage, TURN failure, duplicate Start, simultaneous Stop, stale state, and reconnect. Stop reconnecting after explicit Stop or revocation. Keep connection status, accepted scopes, peer label, route class, stale state, and Stop affordance visible and accessible.

Do not silently transfer authority to another peer or local automation on failure. For presentation, retaining the last scene with an explicit stale indication may be appropriate. For an active momentary control, lease expiry should neutralize it. Privileged irreversible actions need stronger confirmation/interlocks than a low-latency companion button.

Distinguish a momentary-input lease from a whole-controller deadman. The target owns both expiry decisions using its monotonic clock; never trust a controller-supplied deadline. Refresh only after a fresh, valid canonical message and do not invent an undocumented private keepalive when the protocol already has a suitable control such as a snapshot request. Lease freshness usually belongs to transport state and should not churn the semantic compare-and-swap revision. Deadman expiry should pause or neutralize active output and revoke the exact owner in one target-local operation.

Keep listener construction inert. Bind a LAN listener only after explicit local enable, return a port/firewall/bind failure to the UI without crashing startup, and keep disabled ingress fail-closed. A packaged startup test should verify that the feature owns no attributable LAN/remote-control listener before activation.

## Cross-Language Wire Checklist

When Rust and JavaScript share the protocol:

- keep browser-visible integers within JavaScript's safe range, use unsigned 32-bit sequence spaces, or encode larger values losslessly as decimal strings;
- define canonical text bytes, key decoding, HMAC output encoding, field order, role order, and newline rules with shared fixtures;
- sort scopes by their wire strings and reject duplicates before proof calculation;
- reject unknown fields and bound nesting/collections, but avoid relying on Serde combinations whose documented behavior is incompatible, such as `flatten` with `deny_unknown_fields` without a full-frame test;
- use one documented sequence space per lane and exercise wrap/replay rules in both implementations.

## Verification and Claim Discipline

Automated coverage should include:

- canonical transcript/HMAC fixtures shared across implementations;
- malformed, oversized, replayed, reflected, stale-epoch, out-of-scope, and unknown-action rejection;
- reducer and revision/idempotency rules independent of Tauri;
- newest-only backpressure, reliable-queue limit, heartbeat, lease expiry, stale hold, recovery, and teardown;
- denied Tauri capability/scope and caller-window cases;
- CSP, vendored SDK, no-media, and no-page-load-connection checks;
- presentation-only tests proving inbound scenes have no native mutation path.

Then qualify the real matrix: packaged WebView2, WKWebView, and WebKitGTK targets as supported; physical Android Chrome and iOS Safari companions; both host directions; direct and forced-relay routes; phone background/lock; WebView reload/destruction; network handoff; congestion; and multi-peer limits. Browser responsive emulation, mocks, a successful Rust compile, or a Vite build do not prove physical cross-runtime behavior.

Report exact measured endpoints and untested boundaries. Do not claim “direct Wi-Fi,” “offline LAN,” “real time,” “same pixels,” “secure identity,” or protocol conformance unless the corresponding route, timing, identity service, and conformance fixtures were actually exercised.

## Keep Product Contexts Separate

The discovery/control split, typed owner-fenced native seam, target-owned deadman, bounded channel bring-up, and staged read-only qualification are general Tauri remote-control patterns. Product policy belongs in the application profile:

- A PPS experiment runner may revoke local arm when participant, package, calibration, device route, or output configuration changes, and may classify browser audio/vibration as exploratory. Those are PPS safety and evidence rules, not defaults for every Tauri remote.
- An immersive Quest target is a separate native Android/OpenXR or Spatial SDK application context. Reuse transport-neutral contracts and Rust reducer logic through a narrow JNI/native adapter, but keep the headset lifecycle, frame loop, input, audio/haptics, packaging, and physical-device qualification outside the Tauri shell. Read [vr-xr-development.md](vr-xr-development.md) only for an explicitly requested XR task.
- A generic desktop or browser product should define its own neutral state, local confirmation, identity, audit, and availability policy instead of inheriting either PPS or Quest assumptions.

## Architecture Sources and Qualification Limits

- [Browser Remote Sync Protocol](https://github.com/GeorgeFejer91/browser-remote-sync-protocol/tree/62ff66c6df724847c1e54161feabb470b67b1192), source snapshot `62ff66c6df724847c1e54161feabb470b67b1192`, supplies the transport-neutral BRSP/1 handshake, scoped command/state lanes, latest-intent rules, target authority, VDO.Ninja adapter, reusable application-integration starter, Marionette profile, and qualification model.
- [Affect Tracker Web/Desktop](https://github.com/GeorgeFejer91/affect-tracker-web/tree/9e45c4cdc987a91a8cdb00ec3b52cc335ebcf8cb), commit `9e45c4cdc987a91a8cdb00ec3b52cc335ebcf8cb`, demonstrates a Tauri settings WebView and external browser exchanging data-only Flubber Party state, with native snapshots flowing outward and returned aggregate scenes remaining presentation-only.

Affect Tracker is an architecture case study, not a BRSP/1-conformant security example: its experimental Party uses a fixed public room and `password: false`, with explicit source selection and bounded membership/sequence checks but no BRSP mutual HMAC proof or scope negotiation. Its current automated tests and desktop CI verify reducers, static transport/CSP wiring, frontend builds, and Rust gates; physical smartphone-to-Tauri Party qualification remains open. Preserve those qualifications when transferring the pattern.

Before claiming BRSP interoperability, read the pinned repository's normative `docs/03-protocol-specification.md` and `docs/04-vdo-ninja-adapter.md`, run its canonical/HMAC and adapter fixtures, and pin the exact BRSP commit actually implemented. Architectural similarity or use of JSON over a data channel is not conformance.

## Authoritative Live References

- Tauri calling Rust: <https://v2.tauri.app/develop/calling-rust/>
- Tauri calling the frontend and channels: <https://v2.tauri.app/develop/calling-frontend/>
- Tauri capabilities: <https://v2.tauri.app/security/capabilities/>
- Tauri command scopes: <https://v2.tauri.app/security/scope/>
- Tauri CSP: <https://v2.tauri.app/security/csp/>
- Tauri WebSocket plugin: <https://v2.tauri.app/plugin/websocket/>
