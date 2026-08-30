# Latency-Critical Tauri and Rust Systems

Use this guide when the product requires the lowest practical latency, low jitter, reliable high-rate acquisition, deadline-sensitive output, multiple live sources, or defensible timestamp alignment. Also read [performance-and-production.md](performance-and-production.md), [tauri-architecture.md](tauri-architecture.md), and [verification.md](verification.md). Read [sidecars-and-processes.md](sidecars-and-processes.md) if process isolation is under consideration.

## Establish the Timing Contract

Do not treat "fast," throughput, latency, jitter, loss, and synchronization as synonyms. Define before implementation:

- the event that starts each latency measurement and the event that ends it;
- p50, p95, p99, worst observed latency, jitter, and loss separately;
- the authoritative data path and the work allowed to degrade;
- ordering, maximum tolerated backlog/age, and overload behavior;
- every clock domain, how clocks are mapped, and the uncertainty of that mapping;
- whether the target is soft real-time, deadline-oriented, or requires hardware-qualified timing.

Define completion words precisely. "Acquired," "published," "received," "recorded," and "durably flushed" are different boundaries. A successful producer push or IPC send does not prove that a consumer received, persisted, or physically presented the data; instrument the boundary the product actually promises.

Trace the complete path in a release build, for example:

```text
device capture -> OS callback -> decode -> authoritative publish
               -> derived processing -> IPC -> render/present
```

Instrument every boundary with one process-wide monotonic clock where possible. Keep raw device timestamps as evidence; do not replace them with wall-clock time or a smoothed value. Report only the segments actually measured, and identify transport, firmware, driver, display, audio, or remote-host latency that remains outside the application's observation boundary.

## Separate Control, Authoritative Data, and Presentation

Treat Tauri as a control and visualization shell around a native timing-sensitive core:

```text
WebView control/UI
      | typed commands and bounded display frames
native supervisor
      -> input owner -> timestamped bounded queue -> critical publisher
                                                   -> derived worker
                                                   -> persistence/network
                                                   -> coalescing display hub
```

- Keep device ownership, timestamp mapping, sequence tracking, reconnect, and authoritative output in Rust.
- Commands are control-plane request/response. Do not send one command or event per sample.
- Separate reliable low-rate lifecycle/health messages from lossy high-rate display data.
- Put a bounded native bridge in front of every Tauri channel used for high-rate data. Do not call a channel send directly from a device callback or assume the framework channel is the authoritative queue; confirm buffering, cancellation, serialization, and consumer-detach behavior against the pinned Tauri version.
- A hidden, slow, reloaded, or destroyed WebView must not stop authoritative acquisition unless the product explicitly defines window lifetime as session lifetime.
- Make renderer delivery endpoints replaceable. Keep low-rate lifecycle and health state queryable or replayable as a snapshot so a new WebView can reattach without reconnecting hardware; do not require high-rate history replay unless the product explicitly budgets it.
- Aggregate display updates at a controlled cadence and preserve source timestamps. The renderer may drop or coalesce frames without changing the native record.
- If a separately prioritized or crash-isolated worker is needed, define an in-process service boundary first. Move it to a supervised sidecar only when measurement or lifecycle requirements justify the packaging and IPC costs.

## Design the Critical Path

At the earliest callback boundary, capture host-monotonic receive time and do the minimum work needed to retain the data safely. Then hand off to a bounded sequential owner.

- Preallocate queues, batch storage, and reusable scratch buffers where profiling shows allocation pressure.
- Decode each frame once. Share immutable batches or compact owned buffers with downstream branches instead of repeatedly serializing or copying them.
- Preserve per-source ordering while allowing independent sources to run concurrently.
- Keep logging, formatting, JSON, frontend IPC, disk I/O, expensive metrics, and configuration work out of callbacks and critical publisher loops.
- Do not hold a lock acquired by a normal-priority control/UI task on a critical thread. Prefer actor ownership, revisioned immutable snapshots, or prepare/commit configuration changes.
- Run synchronous FFI, blocking device APIs, and CPU-heavy stateful processing on explicitly owned workers rather than occupying shared async runtime workers unpredictably.
- Use named tasks/threads and retain cancellation/join handles. Define startup, reconnect, renderer detach, shutdown, and panic behavior.
- Treat multi-source configuration as one lifecycle transaction: validate and prepare first, serialize connection against reconfiguration, commit every affected owner, and restore already-updated owners if a later commit fails. Never publish a global revision that only a prefix of sources applied.

Avoid adding a thread for every micro-stage. Start with one platform/device owner and one ordered critical worker per isolation domain, plus bounded shared pools for derived or persistence work. Change topology only after measuring queue age, scheduler delay, contention, and CPU occupancy.

## Make Backpressure and Loss Explicit

Every queue must be bounded and every full/disconnected outcome must have a product meaning.

| Lane | Typical overload policy |
|---|---|
| Authoritative raw acquisition/publication | Never silently overwrite. Fail the session or continue with an explicit sequence gap according to a documented integrity policy. |
| Stateful derived processing | Reset/invalidate that processor or stop it visibly; never delay authoritative raw data indefinitely. |
| Durable writer | Bounded queue, acknowledged flush, and fail-stop or explicit gap semantics. |
| Best-effort network output | Count and expose send failures/drops. |
| Display | Coalesce or drop old frames, retain latest state, and count renderer-only loss. |
| Control/health | Reliable, low-rate state that can also be queried as a snapshot. |

Track queue capacity, current depth, high-water mark, oldest-item age, sequence gaps, callback drops, decoder errors, output errors, reconnects, and renderer drops separately. A large queue may hide a latency failure; admission and health decisions should use age as well as depth.

Define load shedding before overload occurs. Reduce presentation cadence and optional derived work before compromising authoritative acquisition. If the critical lane cannot meet its contract, surface a failed/degraded session instead of producing silently incomplete data.

## Timestamp Multiple Sources Correctly

Different sample rates do not require a shared sample index or producer-side resampling. Keep independent raw streams with their native cadence and align them by timestamps.

Use a batch envelope that can represent:

```text
source instance and session generation
stream id and sequence range
device timestamp, if available
host receive monotonic time
mapped common-clock time
sample period or explicit per-sample timestamps
gap/reset flags
clock mapping revision, quality, and uncertainty
```

When the frontend is JavaScript, do not place nanosecond-scale `u64` values in ordinary JSON numbers. Serialize them as decimal strings, split integer fields, or use another lossless typed boundary, then convert only to the floating time resolution needed for presentation.

- Use one mapper per physical clock, not one independent first-arrival offset per output.
- For a monotonic device clock, estimate an affine mapping to the host or common output clock (including LSL where used), constrain drift, reject outliers, preserve monotonicity, and start a new segment on reconnect or clock reset.
- For devices without sample timestamps, use host receipt time near the transport boundary and backfill only when protocol cadence justifies it. Label the larger uncertainty honestly.
- Derived samples inherit the causal raw timestamp and mapping revision.
- Preserve unsmoothed producer timing evidence. Apply visualization smoothing, online control filtering, or offline dejittering as separate, reversible operations.
- For LSL or another stream bus, keep independent native-rate streams with accurate metadata and stable source/session identity. Treat producer push completion, remote inlet receipt, recorder acknowledgement, and durable file flush as separate measured endpoints.
- A fused algorithm should use an explicit cadence, interpolation/hold rule, causal watermark, missing-data mask, and provenance. Publish it as a separate derived stream; do not mutate the raw inputs.

For a WebView visualization, store timestamps in typed temporal rings and draw all traces against a common time axis. Decimate by time/pixel while preserving extrema, break paths across gaps, and keep incompatible physical units on separate axes. A visually aligned panel is not evidence of hardware synchronization.

## Treat Radios and Shared Transports as Finite Schedulers

For BLE, USB, audio, network, and similar transports, application threads are only one part of the latency budget. Firmware, controller scheduling, OS drivers, negotiated parameters, interference, retransmissions, and packetization remain outside direct Rust ownership.

- Scanning can often coexist with active BLE sessions, but it consumes controller airtime and is not guaranteed to be impact-free on every adapter/driver.
- Use an adapter-scoped discovery coordinator, incremental candidate registry, short scan windows, and passive scanning when sufficient.
- Do not replace live session authority merely to refresh discovery results.
- Treat connection-parameter and MTU requests as negotiated hints. Lowest single-link latency, maximum connection count, continuous scanning, power, and reliability are competing goals.
- Admit new sources based on measured inter-arrival, queue, loss, and radio health rather than a compile-time maximum alone.
- Qualify every supported adapter/driver/device combination physically under representative interference and concurrent load.

## Apply OS Scheduling Conservatively

Priority cannot make a general-purpose OS or radio hard real-time. Excessive priority can starve Bluetooth/USB driver work, networking, disk flushing, input, the watchdog, or the UI and cause more loss.

- Prefer event-driven work, short critical sections, bounded queues, and reduced contention before priority changes.
- Raise only acquisition/publication threads or an isolated broker, temporarily and reversibly. Do not elevate the whole Tauri/WebView process by default.
- Device callbacks may execute on OS-, driver-, COM-, or library-owned threads that the application cannot safely reprioritize. Elevating a downstream worker cannot accelerate radio/controller/driver delivery; measure the boundary before selecting a scheduling API.
- On Windows, evaluate HighQoS, Above Normal/High priority, or an appropriate MMCSS task only after measurement. Do not default to `REALTIME_PRIORITY_CLASS` or time-critical threads.
- Do not use fixed CPU affinity by default, especially on heterogeneous processors. Use scheduler hints or machine-qualified profiles only when benchmarks show a repeatable gain.
- Request finer timer resolution only for an actual timer-driven deadline. It does not improve the monotonic clock or accelerate hardware callbacks.
- Prevent automatic sleep during an active critical session when appropriate, and restore execution state, QoS, priority, affinity, and timer settings through same-thread/RAII cleanup.
- Linux realtime scheduling and macOS time-constraint policies require platform-specific privileges, quotas, watchdogs, and physical tests. Treat them as expert modes, not portable defaults.

Offer named performance profiles rather than one unsafe switch. A useful progression is system-managed, recording/low-latency, and expert laboratory mode, with the active controls and risks visible to the user.

## Distinguish Visualization From Stimulus Timing

A WebView chart can be low-latency biofeedback without being a psychophysics stimulus renderer. Rendering cadence, browser scheduling, compositor behavior, and display presentation must be measured separately from sensor acquisition.

- Coalesce native display data to a display-driven cadence derived from product data-age needs, `requestAnimationFrame`, refresh behavior, and measured WebView cost instead of forwarding every sample.
- Measure data age at paint/presentation and renderer frame misses separately from callback-to-authoritative-publication latency.
- For frame-accurate stimuli, use a dedicated native GPU/window path or integrate with a validated experiment tool. Record present/flip timing and validate physical output with a photodiode, microphone, loopback, or equivalent instrumentation.

## Verification Matrix

Test release builds with symbols where profiling needs them. Cover nominal and adversarial states:

- 1, 2, 4, and maximum-admitted source shapes;
- different source rates and clock domains;
- scan/discovery, connect, reconnect, cancellation, and shutdown during live acquisition;
- hidden, stalled, reloaded, and destroyed renderers;
- every output enabled together and each failure mode independently;
- CPU, disk, network/radio interference, and power-state changes;
- short latency trials plus multi-hour soak tests;
- queue saturation, malformed input, clock reset, sequence gap, and sidecar crash where applicable.

Make load profiles reproducible: record the load generator, duty cycle, duration, hardware/power state, and background services rather than saying only "under load." Require zero unreported loss. Define thresholds from the actual notification/deadline budget, then gate p95/p99, queue age, loss, clock residuals, and regression under scanning/load. Synthetic tests prove application logic and capacity; only physical tests can qualify transport reliability, sensor-to-host timing, audio/display onset, or cross-device synchronization.

## Completion Report

State the exact measured endpoints, build/profile, hardware and OS/driver versions, active sources/outputs, load conditions, sample duration, percentile results, observed loss, and untested physical boundaries. Never turn "native Rust," a raised priority, a successful compile, or a visually smooth chart into a low-latency or real-time claim without measurements.

## Authoritative References

- Tauri IPC and channels: <https://v2.tauri.app/develop/calling-frontend/>
- Windows scheduling priorities: <https://learn.microsoft.com/en-us/windows/win32/procthread/scheduling-priorities>
- Windows Multimedia Class Scheduler Service: <https://learn.microsoft.com/en-us/windows/win32/procthread/multimedia-class-scheduler-service>
- Windows high-resolution timestamps: <https://learn.microsoft.com/en-us/windows/win32/sysinfo/acquiring-high-resolution-time-stamps>
- Windows timer resolution: <https://learn.microsoft.com/en-us/windows/win32/api/timeapi/nf-timeapi-timebeginperiod>
- Windows BLE preferred connection parameters: <https://learn.microsoft.com/en-us/uwp/api/windows.devices.bluetooth.bluetoothledevice.requestpreferredconnectionparameters>
- LSL time synchronization: <https://labstreaminglayer.readthedocs.io/info/time_synchronization.html>
- LSL timestamp/latency FAQ: <https://labstreaminglayer.readthedocs.io/info/faqs.html>
- PsychoPy ioHub isolation: <https://devdocs.psychopy.org/api/iohub/index.html>
- PsychoPy frame timing: <https://psychopy.org/general/timing/detectingFrameDrops.html>
