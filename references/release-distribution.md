# CI, Updates, Signing, and Distribution

Use this reference for release design. Exact action versions, store rules, certificates, and runner images are volatile; verify them in current Tauri, OS vendor, store, and action documentation before publishing.

## Release Invariants

- One reviewed version identifies frontend assets, Rust binary, installer, update manifest, migrations, and release notes.
- Build each artifact on a trusted host appropriate to its platform/architecture unless an officially supported cross-build path is verified.
- Use lockfiles and `--locked`/frozen installs. Pin third-party CI actions by full commit SHA for high-assurance releases and review automated updates.
- Separate untrusted pull-request tests from secret-bearing signing/publishing jobs.
- Give CI tokens minimum permissions; use protected environments and human approval for production.
- Produce checksums, an SBOM/provenance where required, and retain build logs without secrets.
- Test the exact artifact before promoting a draft release. Never rebuild after approval and call it the same artifact.

## Pipeline Shape

Run format/type/lint/unit/contract tests once, then a host-native build matrix for Windows, Linux, macOS Intel/ARM, and supported mobile targets. Cache dependency/build artifacts with keys derived from lockfiles, toolchain, target, profile, and relevant features—never use cache output as release provenance.

Run an explicit job with the workspace's declared `rust-version`; a newer floating stable job cannot prove MSRV compatibility, and the resolved Tauri dependency graph may require a newer compiler than the manifest claims. Keep one reviewed Cargo lock for the shipped workspace and use frozen frontend installs.

Treat generated native libraries and compiled frontend assets as release inputs with provenance, not convenient leftovers. Build-system dependency graphs must include shared path crates and root lockfiles. A bypass that skips a producer must also exclude its previous output from packaging. If compiled web assets are tracked, rebuild them in CI and fail on drift.

For GitHub releases, the official `tauri-action` can build and upload artifacts. Add the missing production gates around the tutorial baseline: current Linux build dependencies, explicit targets, tests before build, signing credentials only in protected jobs, updater artifact creation, draft release, checksum/signature verification, install/launch smoke tests, then promotion. Do not grant `contents: write` to unrelated jobs.

## Desktop Code Signing

### Windows

Use an organization-appropriate Authenticode certificate or managed signing service. Configure the digest/timestamp settings required by the current Tauri bundler and certificate provider. Keep private keys hardware-backed or in a managed vault where possible. Verify signatures after packaging and test SmartScreen/reputation behavior; a valid signature does not instantly create reputation.

For an HSM/cloud signer or cross-host build, use Tauri's current custom `signCommand` contract only after verifying the placeholder/quoting behavior with paths containing spaces. Prefer short-lived workload identity/OIDC over a long-lived client secret when the provider supports it, pin the signer tool, keep provider coordinates in a protected build overlay, and verify both the application executable and final installer were signed and timestamped.

Choose MSI/WiX vs NSIS based on install scope, enterprise management, customization, and updater compatibility. Decide WebView2 delivery mode (bootstrapper, embedded/offline, fixed runtime, or skip) from offline size, servicing, and supported-OS requirements. Test upgrade/downgrade, per-user/per-machine install, locked files, repair, uninstall, and long/non-ASCII paths. Store/MSIX distribution has separate identity, capability, signing, and sandbox rules.

For a Microsoft Store Win32 submission, maintain a store-specific config/build flavor. Verify the current required WebView2 delivery mode, silent-install arguments (NSIS and MSI differ), publisher metadata, update ownership, and clean unattended install/uninstall in the Store validation environment.

### macOS

Use the correct Apple certificate for direct distribution or the App Store. Sign nested binaries/frameworks/sidecars, use the hardened runtime and only required entitlements, then notarize and staple direct-distribution artifacts. Validate with platform tools after packaging and test on a clean quarantined machine. Ad-hoc signing can satisfy Apple Silicon's executable-signature requirement but does not establish developer identity or remove Gatekeeper prompts.

For the Mac App Store, enable App Sandbox, provision the registered bundle ID, declare entitlements, and audit filesystem/network/child-process behavior against sandbox constraints. Test Intel/ARM slices and any embedded frameworks.

### Linux

Match format to users and distribution policy:

- AppImage is portable but depends on host/runtime compatibility; its embedded signature is not automatically enforced, so publish verification instructions/key identity over an authenticated channel.
- Debian and RPM packages integrate dependencies, menus, upgrades, and package managers; declare dependencies and scripts conservatively and test clean install/upgrade/removal.
- Flatpak and Snap add sandbox, portal, filesystem, DBus, and review constraints; re-test plugins and native integrations inside the sandbox.
- AUR recipes are transparent build/package instructions maintained through Arch workflows; keep checksums and sources reproducible.

Test Wayland/X11, WebKitGTK requirements, desktop files/icons, MIME/deep links, tray dependencies, and distro age. Do not imply one Linux artifact covers every distribution.

## Updater Architecture

Tauri desktop updates are signature-verified; verification cannot be disabled. Protect and back up the updater private key: losing it can strand installed clients, while compromise permits malicious updates. Embed only the public key.

Choose exactly one update owner per distribution channel. Store-managed, package-manager, Flatpak/Snap, enterprise MDM, and similar builds should normally defer to that channel instead of running an independent self-updater. Encode this as a build flavor/capability decision so one installation cannot be advanced by two competing systems.

- Set `createUpdaterArtifacts` for the current updater format; use compatibility mode only while a documented legacy migration requires it.
- Keep signing secrets in the CI secret store/environment, not `.env` or the repository.
- Serve update metadata and artifacts over HTTPS. The insecure transport option is for constrained testing, not production.
- Publish the correct platform/architecture installer and detached signature in either a static manifest or a narrowly implemented dynamic endpoint.
- Validate manifest schema, semantic version policy, release notes rendering, content length, and artifact availability before promotion.
- Separate stable/beta/nightly channels with explicit runtime endpoints and authorization; prevent accidental cross-channel downgrade.
- Stream download progress at a bounded rate, support cancellation, and give the user a clear restart/installation state.
- On Windows, coordinate the pre-exit hook and open handles before installer handoff.

Design rollback as a new, signed forward release unless the updater's current version comparator and migration design explicitly permit downgrade. Database/settings migrations must be backward compatible across the supported rollback window, or backed up and restored transactionally. Test update from every supported source version, interrupted download/install, disk-full, bad signature, corrupt manifest, expired TLS, offline, and sidecar/resource replacement.

## Mobile Signing and Stores

### Android / Google Play

Create and protect an upload keystore; keep keystore and generated property files out of source control. Use Play App Signing where chosen, configure release signing in the generated Gradle app module, and inject secrets in protected CI. Build Android App Bundles for Play and APKs for direct/device testing; verify application ID, version code/name, icons, permissions, target/min SDK, ABI coverage, privacy/data-safety declarations, and physical-device behavior. Preserve key-recovery and ownership documentation.

### iOS / App Store

Register the bundle identifier, capabilities, certificate, and provisioning profile. Prefer Xcode-managed signing locally; in CI use protected App Store Connect credentials or deliberately provision certificate/profile secrets. Verify entitlements and usage descriptions against runtime calls. Archive and validate through current Xcode tooling, test TestFlight, review privacy manifests/labels, and retain symbol files needed for crash diagnosis.

## Final Promotion Checklist

Verify clean checkout, dependency integrity, version consistency, release notes, license/notice files, icons/metadata, migrations, resources/sidecars, installer signatures, updater signatures/manifests, checksums, install/launch/smoke tests, upgrades from supported versions, uninstall/data-retention behavior, crash symbols, rollback plan, and download links. Promote only the already-tested draft artifacts.
