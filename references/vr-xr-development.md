# Optional Application Context: Rust for VR/XR and Meta Quest

Use this reference only when a Tauri/Rust request explicitly extends into an immersive headset app or asks whether a shared Rust core belongs behind a native XR shell. Do not apply Quest/OpenXR/Spatial SDK assumptions to an ordinary desktop or mobile Tauri task. Verify current Meta, OpenXR, Android, and crate documentation against the pinned versions before implementation; headset APIs and Horizon OS behavior are volatile.

## Choose the Runtime Before the Language

Rust can be an excellent native library inside a VR product, but it is not automatically the best owner of every subsystem.

| Concern | Prefer | Reason |
|---|---|---|
| Researcher settings, session export, desktop control | Tauri + web frontend + Rust | Tauri is well suited to trusted desktop/native workflows around a web UI. |
| Quest immersive scene, panels, controller pointers, lifecycle | Native Android/Kotlin with Meta Spatial SDK, or another supported OpenXR engine | The immersive runtime must own the Activity, OpenXR session, entities, surfaces, focus, and headset lifecycle. |
| Portable deterministic math, hashing, validation, protocol, reducer | Pure Rust library | Rust can share semantics without forcing a UI/runtime onto every target. |
| Video playback supported by the headset | Media3/ExoPlayer to a Spatial SDK surface | This keeps container parsing and decode on Android's supported hardware path. |
| Unsupported codecs or offline transcoding | Qualify FFmpeg separately | Bundled software decode increases APK size, CPU, battery use, thermal load, and licensing work. |

Do not use a Tauri WebView as the immersive renderer merely because the wider product already uses Tauri. A common architecture is a Tauri desktop operator app plus a separate native Quest APK sharing pure Rust contracts/reducer code and golden fixtures. This is one product with two delivery forms, not one runtime forced onto both platforms.

## Recommended Quest Boundary

Default to one immersive runtime/session owner. Add Activities, processes, or APKs only when they represent an explicit product or isolation boundary.

- Kotlin/Android owns Activity lifecycle, Spatial SDK feature/system registration, controller and hand input, Media3 surfaces, Storage Access Framework grants, permissions, and UI-thread callbacks.
- Rust owns portable pure logic or native services that benefit from it: deterministic calculations, schema/hash validation, authorization, state reduction, LSL, or bounded network workers.
- Tauri may own a separate desktop researcher UI or session exporter; it should not be introduced into the headset process unless the product genuinely needs a non-immersive WebView panel.
- Keep Tauri, Android, JNI, UI, filesystem, transport, audio, and haptics types out of the shared domain crate. Put them in platform adapters.
- Local and remote controls may call the same reducer, but preserve an explicit origin. Authenticated remote input must never reuse a local-only alias or inherit local authority.
- Panels, video carriers, and scene objects are entities inside the same immersive app. They do not need separate controller ownership or separate applications.

OpenXR gives input focus to an application session, not separately to each entity. If a video panel works while another entity ignores the stick, investigate routing, system registration, focus, and component state before concluding that the panels are competing applications.

## Rust on Android

Build the Rust library for the exact Android ABI, normally `aarch64-linux-android` packaged as `arm64-v8a`, with the project's locked NDK. Keep the JNI/FFI boundary small.

- Expose coarse operations and bounded semantic commands/snapshots rather than one JNI call per vertex, sample, or frame field.
- Prefer immutable request/response structs, direct buffers, or bounded channels for high-rate data.
- Define ownership of every pointer, buffer, callback, and native handle. Make teardown idempotent.
- Never unwind a Rust panic across JNI/FFI. Convert failures to stable errors at the adapter boundary.
- Keep Android/Spatial objects and thread-affine APIs on their native owner thread. Do not pass Java or Spatial handles into arbitrary Rust workers.
- Pin crate, NDK, Gradle, vendor SDK, and native ABI versions. Validate the packaged `.so`, its architecture, load name, and exported JNI symbols in the final APK.
- Prefer an in-process Rust library for low-latency trusted code. Use a sidecar/process only when crash isolation or a non-embeddable runtime justifies the extra lifecycle and IPC cost; Android headset apps generally make sidecars awkward.

Rust is not faster by definition. Crossing JNI every frame, copying geometry, or replacing a hardware decoder with software can make the system slower. Measure the whole path on the worst supported headset.

## Build-System Correctness

Android's incremental build graph must include the full native source graph, and a build bypass must not package a stale generated library. Follow the detailed Rust `cdylib` input, stale-`.so`, and artifact-inspection gates in [mobile-development.md](mobile-development.md). Also verify the Gradle wrapper checksum, distinguish direct pins from transitive dependency verification, and resolve vendor sample/release-note version conflicts with a reproduced build and tests.

## Video and Media

For ordinary Quest playback, let Media3/ExoPlayer inspect the container, tracks, MIME types, dimensions, rotation, and codecs, then render directly to the Spatial SDK surface. "Supported formats" means formats that both Media3 and the active device decoder can prepare; it is not an unrestricted FFmpeg-style promise.

- Keep projection (`flat`, 180, 360) and stereo layout (mono, side-by-side, top-bottom) explicit in a session manifest. Do not guess from filenames or aspect ratios.
- Separate start request, decoder readiness, first rendered frame, pause/resume, end, and error markers.
- Keep video decoding independent from Rust LSL/network scheduling and custom-object drawing.
- Qualify FFmpeg only for a demonstrated missing requirement. Check codec patent/license obligations, native dependencies, APK size, thermal behavior, and whether hardware decoding remains available.

## Spatial SDK Controller Input

Start from a working official sample that matches the pinned Spatial SDK version.

- Register the documented feature/input systems and remove default locomotion only when the application must own those controls.
- Query live controller entities from the SDK's system execution context; observe active state and raw button changes before adding domain mapping.
- Treat invalid or sentinel entity IDs as unavailable. Do not repeatedly call native component APIs on them and flood the frame loop with errors.
- Horizon/SDK combinations may expose controller, hand, attachment, avatar, and direct-touch representations. Instrument them separately before merging only the representations justified by evidence.
- Do not assume Android `MotionEvent` joystick axes are delivered to a Spatial SDK immersive Activity. Prefer the SDK's official controller route unless device evidence proves otherwise.

A controller laser changing color proves that the runtime noticed input. It does not prove the application received a direction value or that domain state reached the renderer.

## Low-Latency Design

- Keep one frame owner. Sample input, update domain state, and submit visual state in a predictable order.
- Preallocate geometry, paths, and point buffers. Avoid steady-state allocations and per-point JNI calls.
- Run LSL or other network publishing on a dedicated native worker with a bounded queue. Never block media callbacks or the render/system thread.
- Acquire Android multicast resources only while discovery/outlets are active; document that the Wi-Fi access point must allow multicast and client-to-client traffic.
- Timestamp at the event source. Distinguish user start request from first rendered frame or physical output onset.
- Share semantic state-machine code without transferring timing claims. Desktop, browser, phone, and headset adapters each require target-specific physical qualification.

## Remote Control and Lifecycle

For browser or phone control, read [marionette-remote-control.md](marionette-remote-control.md) and preserve its proof, scope, origin, owner-generation, arming, and target-owned deadman rules. XR-specific lifecycle callbacks must route Activity pause/stop/destroy and headset focus loss into the same idempotent safe-state/revocation path. A build flag is a promotion gate, not an authorization implementation; leave ingress fail-closed while protocol, lifecycle, artifact, or device evidence is incomplete.

## Staged Experiment or Media Bundles

When an XR product stages experiment or media bundles, an app-private APK/data directory is not a user-friendly MTP drop folder. Prefer a clearly named shared `Documents/...` directory authorized through Android's Storage Access Framework. The manifest-last protocol below is a product pattern for atomic staged bundles, not a universal XR requirement.

- Persist the tree grant and handle loss/revocation explicitly.
- Copy media first and the active JSON manifest last.
- On a background I/O coroutine, wait for stable size and verify containment, declared byte length, digest, schema, projection/stereo metadata, and decoder preparation.
- Do not watch or replace the session during playback. Keep the last valid staged session if a new bundle is partial or invalid.
- Avoid broad/all-files permission merely to simplify development.

## Hierarchical Verification

Use gates that isolate failures before another APK/device cycle:

1. Pure Rust/JVM/JavaScript tests for schemas, mappings, authorization, lifecycle, and deterministic golden fixtures.
2. Host formatting, lint, unit tests, exact-MSRV checks, dependency locks, native cross-compilation, and reproducible APK build.
3. Inspect package identity, merged manifest, permissions, cleartext policy, ABI libraries, JNI symbols, and exact APK SHA-256.
4. Install serial-scoped to the intended headset and prove the launcher and immersive Activity are visible/focused.
5. Require app-owned markers for scene ready, decoder ready, visible custom entity, first custom draw, start request, and first rendered frame.
6. Prove controller response as a chain: physical SDK state changes, mapped domain values change, and the renderer acknowledges a draw from that change.
7. Prove remote control as a chain: browser proof/scopes, target acceptance, reducer transition, safe output, acknowledgement, reconnect, and deadman expiry.
8. Resolve external streams from an independent consumer and verify schema, channel order, timing, markers, and reconnect behavior.
9. Run performance and soak tests on the slowest supported headset, then smoke-test the remaining device matrix.

A debug command that injects input is useful: it proves the internal `input -> domain engine -> renderer` path without touching the headset. It is never evidence that physical input works. Likewise, entity creation, a neutral first draw, a successful APK install, or a passing remote protocol fixture is not final physical feature evidence.

## Authoritative Starting Points

- Meta Spatial SDK samples: <https://github.com/meta-quest/Meta-Spatial-SDK-Samples>
- OpenXR 1.1 specification: <https://registry.khronos.org/OpenXR/specs/1.1/html/xrspec.html>
- Android Media3 supported formats: <https://developer.android.com/media/media3/exoplayer/supported-formats>
- Android supported media formats/codecs: <https://developer.android.com/media/platform/supported-formats>
- Android JNI guidance: <https://developer.android.com/training/articles/perf-jni>
- Android Rust overview: <https://source.android.com/docs/setup/build/rust/building-rust-modules/overview>
- Tauri Android prerequisites: <https://v2.tauri.app/start/prerequisites/#android>
