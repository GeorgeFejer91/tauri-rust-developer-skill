# Tauri Security

Read this guide for any IPC or privilege change. A desktop application is not safe merely because its frontend files are local: treat WebView content, user-controlled data, links, imported files, and all command arguments as untrusted at the native boundary.

## Contents

- Security Review Sequence
- Capabilities, Permissions, and Scopes
- Command Design
- Content Security Policy
- Remote Content and Navigation
- Sidecars and Plugins
- Lifecycle Threat Model and Supply Chain
- Incident-Safe Observability
- Security Checklist
- Authoritative Live References

## Security Review Sequence

For each exposed native operation:

1. identify which windows or WebViews can reach it;
2. validate every argument in Rust;
3. identify the operating-system, filesystem, shell, process, or network authority involved;
4. confirm the exact capability and permission required by the pinned Tauri/plugin version;
5. narrow any scope to the minimum paths, commands, URLs, or operations;
6. review output for secrets and internal details;
7. test denied and malformed cases as well as success.

Treat application-defined commands as reachable unless the project explicitly restricts them and the current Tauri schema confirms that restriction. Never infer protection from a hidden UI button.

## Capabilities, Permissions, and Scopes

- A capability binds permissions to specific windows, WebViews, and optionally platforms or remote origins.
- A permission enables a command or operation.
- A scope constrains the resources that an enabled operation may access.

Keep capability files small and purpose-specific. Prefer exact labels and narrow permissions over broad bundles. Do not add a wildcard because a plugin call failed; inspect the generated schema and plugin documentation to find the missing permission.

For filesystem access, scope to product-owned directories or explicitly selected paths. For shell and sidecar access, allowlist exact programs, subcommands, and safe arguments. For HTTP access, constrain schemes and hosts and guard redirects or user-supplied URLs. Platform-specific authority belongs in platform-specific capabilities when possible.

Scopes for commands implemented inside the application are not self-enforcing policy. Define the scope type, resolve the applicable scope for the caller, and enforce it inside the command or service before performing the privileged operation. Test both allowed and denied scope entries.

After a plugin or Tauri upgrade, regenerate or inspect the capability schemas and repeat the review. Names and semantics can change.

## Command Design

- Expose product operations, not generic primitives.
- Parse and canonicalize paths before policy checks; defend against traversal, symlinks, and confusing relative roots where relevant.
- Parse URLs and enforce schemes, hosts, ports, redirect behavior, and size/time limits.
- Bound collection sizes, string lengths, file sizes, and concurrency.
- Avoid arbitrary SQL, shell strings, script evaluation, or unrestricted environment access.
- Keep secrets and authorization decisions in Rust.
- Return stable sanitized errors to the WebView.

If a privileged command is intended for only one window, enforce that policy with the current capability/permission model and, where necessary, an explicit runtime caller check.

## Content Security Policy

Keep CSP restrictive and compatible with the actual frontend build. Avoid adding `unsafe-eval`, broad `*` sources, or arbitrary remote origins to make a dependency work. Prefer nonces or hashes when inline content is unavoidable and supported by the architecture.

When changing CSP:

1. identify the exact blocked resource or execution mode;
2. decide whether the dependency can be configured more safely;
3. add the narrowest source directive;
4. test development and production builds, because their asset behavior may differ;
5. document any deliberate weakening.

Do not place secrets in frontend environment variables, bundled JavaScript, HTML, local storage, or CSP configuration. Anything shipped to the WebView is recoverable by the user or an attacker with renderer access.

## Remote Content and Navigation

Enabling remote WebView content materially expands risk. Pause for explicit authorization and then review:

- exact remote origins and whether navigation can escape them;
- capability attachment to remote origins;
- CSP and content integrity;
- open-redirect, deep-link, and custom-protocol behavior;
- whether remote content can invoke app commands;
- link opening and new-window behavior.

Open untrusted links in the system browser through a narrow, validated operation rather than navigating the privileged app WebView.

## Sidecars and Plugins

- Prefer official or well-maintained plugins with reviewed permissions.
- Initialize only plugins actually used.
- Pin and verify sidecar artifacts through the project's release process.
- Treat sidecar arguments, stdin, stdout, and update channels as trust boundaries.
- Do not pass secrets on command lines when process listings or logs may expose them.
- Apply least privilege separately on every supported platform.

## Lifecycle Threat Model and Supply Chain

Write a compact threat model for each privileged feature: assets, principals, entry points, trust boundaries, abuse cases, mitigations, residual risk, and verification. Include the entire lifecycle—developer machine, source control, dependencies, CI, signing, update host, distribution channel, runtime WebView, imported content, sidecars, and support diagnostics. The weakest stage can invalidate later controls.

- Keep Tauri, Rust, Node/frontend tooling, the system WebView, and direct dependencies within a documented support/update window.
- Review maintainer health, permissions, transitive native code, install/build scripts, features, licenses, advisories, and lockfile changes before adding dependencies.
- Run `cargo audit` and the repository's npm advisory check, but triage reachability and fixes rather than mechanically accepting breaking upgrades.
- For higher assurance, maintain `cargo deny` policy and/or a reviewed `cargo vet` supply-chain policy; generate an SBOM or auditable dependency metadata when required.
- Pin source dependencies to immutable revisions and pin release CI actions. Protect branches/environments and require review for capability, signing, updater, workflow, and lockfile changes.
- Never expose a mobile development server on an untrusted network. Physical-device development may bind broadly and typically lacks mutual authentication/encryption.
- Keep production secrets off developer machines where possible. Use hardware-backed or managed signing keys and rehearse revocation/recovery.
- Treat reproducible builds as a measured property, not an assumption; Rust and frontend tools may still inject nondeterminism.

Run fuzz/property tests on parsers, file formats, URL/deep-link inputs, IPC DTOs, sidecar protocols, migration code, and unsafe/FFI adapters when their risk merits it. Arrange coordinated vulnerability disclosure and an emergency signed update path before launch.

## Incident-Safe Observability

Log event type, timestamp, version, platform, correlation ID, duration, and sanitized error code. Do not log secrets, cookies, authorization headers, passwords, key material, full URLs with queries, clipboard contents, arbitrary file contents, or unredacted personal paths. Use allowlisted structured fields rather than trying to redact arbitrary strings after capture.

Bound log size and retention, restrict local file permissions, and make remote diagnostic upload opt-in with a preview. Security events should be actionable without becoming a surveillance stream. Preserve enough release/build identifiers to distinguish a compromised or vulnerable artifact, and document log/key/update-host response procedures.

## Security Checklist

- [ ] IPC inputs validated in Rust
- [ ] Responses and logs contain no secrets or internal diagnostics
- [ ] Capability targets only necessary windows/WebViews/platforms
- [ ] Permissions and scopes are narrower than the underlying plugin's maximum authority
- [ ] Filesystem paths, URLs, shell arguments, and sidecar inputs are constrained
- [ ] CSP remains restrictive
- [ ] Remote content is absent or explicitly authorized and isolated
- [ ] Denied, malformed, oversized, and cancellation paths are tested
- [ ] Current generated schema and official documentation were consulted
- [ ] Dependency, build, signing, update, and support-data lifecycle was threat-modeled
- [ ] Advisories/policies/SBOM needs were checked and CI actions are pinned appropriately
- [ ] Logs and crash/support artifacts are bounded, redacted, retained deliberately, and user-consented

## Authoritative Live References

- Tauri security overview: <https://v2.tauri.app/security/>
- Capabilities: <https://v2.tauri.app/security/capabilities/>
- Permissions: <https://v2.tauri.app/security/permissions/>
- Scopes: <https://v2.tauri.app/security/scope/>
- Content Security Policy: <https://v2.tauri.app/security/csp/>
- Application lifecycle threats: <https://v2.tauri.app/security/lifecycle/>
- Ecosystem security: <https://v2.tauri.app/security/ecosystem/>

Use the official pages and the project's generated schemas together; neither substitutes for the other.
