# Files and Persistence

Use this guide for native dialogs, file access, settings, databases, migrations, and secrets.

## Contents

- [Choose the Storage Boundary](#choose-the-storage-boundary)
- [Dialogs and User-Selected Files](#dialogs-and-user-selected-files)
- [Filesystem Design](#filesystem-design)
- [Settings Store](#settings-store)
- [SQLite and Databases](#sqlite-and-databases)
- [Migrations](#migrations)
- [Secrets and Stronghold](#secrets-and-stronghold)
- [OS Credential Stores](#os-credential-stores)
- [Tests](#tests)

## Choose the Storage Boundary

| Need | Preferred starting point |
|---|---|
| Small non-secret settings | Store plugin or typed Rust settings file |
| Structured local product data | SQLite with migrations |
| Credentials, tokens, private keys | OS credential service or Stronghold-style secret vault |
| User-selected documents | Dialog plus narrowly scoped filesystem access |
| Large/streaming file processing | Rust service with bounded commands/channels |

Putting data in a local file does not make it secret. Putting a database API in the frontend exposes its granted query authority to compromised WebView code.

## Dialogs and User-Selected Files

Treat cancel as a normal result. Apply extension filters for usability, then validate content and format after selection; filters are not a security boundary.

Desktop dialogs generally return paths. Mobile differs: iOS can return file URLs and Android content URIs. Use the current Tauri filesystem/dialog abstractions rather than coercing every result into a desktop path.

For destructive confirmation dialogs, describe the exact consequence and make the safer action the default. Avoid blocking Rust dialog APIs on an async or UI-sensitive path when a callback/non-blocking form works.

## Filesystem Design

- Prefer application config/data/cache/log directories for app-owned files.
- Prefer user selection plus a runtime scope for arbitrary documents.
- On the frontend, use base directories or Tauri path APIs; raw parent traversal is rejected by the official filesystem plugin.
- On the Rust side, use `std::fs` for short synchronous work or `tokio::fs`/a blocking worker for async flows. The frontend filesystem plugin does not replace Rust file APIs.
- Canonicalize and re-check policy before privileged access, while accounting for a path that does not exist yet.
- Use atomic save patterns for valuable documents: write a sibling temporary file, flush when durability matters, then rename/replace according to platform semantics.
- Bound file sizes and streaming buffers; avoid loading untrusted multi-gigabyte files into IPC or memory.
- Close frontend file handles in `finally`-equivalent control flow.

Filesystem scope denies take precedence over allows. Scope source and destination separately for copy/rename. `stat` follows symlinks while `lstat` describes the link; choose deliberately.

For watches, enable only the required crate feature, choose debounced versus immediate events based on the product, avoid recursive watching by default, and retain/unsubscribe the watcher handle. Expect duplicate, coalesced, rename, delete, overflow, and out-of-order events. Debounce bursts, rescan after overflow, and protect against a save-triggered feedback loop with a generation/content marker rather than timing alone.

When caching file-backed settings or documents, an mtime by itself is not a reliable version: timestamp granularity, replacement, and external tools can hide a change. Use a coherent snapshot with size plus mtime and, where correctness matters, a digest or watcher-maintained generation. Read related state under one documented lock order, keep lock scope short, and define an explicit poisoned-lock policy rather than silently exposing possibly inconsistent data.

Persisted runtime scopes restore authority across launches. Register the persisted-scope plugin after the filesystem plugin, review whether persistence matches user expectations, and provide a way to revoke previously selected access.

## Settings Store

The store plugin is asynchronous and can operate from Rust or the WebView. Choose a single ownership model to avoid racing writes.

- Use typed accessors around JSON values.
- Decide between explicit save, debounced auto-save, and save on graceful exit.
- Do not rely only on graceful-exit persistence for critical data; apps and devices crash.
- Version the settings schema and migrate old values defensively.
- Never store secrets in a plain settings store.
- Close resources and remove stale listeners when a store is no longer needed.

## SQLite and Databases

The official SQL plugin supports SQLite, MySQL, and PostgreSQL through feature-selected drivers. For local-first apps, SQLite is usually the simplest. For remote databases, do not ship database credentials or broad direct database authority to the WebView; route through a trusted service/API.

Decide who owns queries:

- frontend SQL plugin calls are convenient for low-risk local CRUD but require explicit query permissions and expose that authority to the WebView;
- Rust repository/service methods provide tighter domain validation, transaction boundaries, secret handling, and test seams.

Always parameterize values. Do not concatenate user input into SQL. Treat table/column/order identifiers separately because value parameters cannot safely stand in for arbitrary syntax.

Database encryption protects selected at-rest threats; it does not make a running compromised app or unlocked OS session trustworthy. Define who supplies and can recover the key, how it enters memory, whether it ever crosses the WebView, how rotation/backup works, and which metadata or temporary/WAL files remain exposed. SQLCipher or another alternative SQLite build changes native linking, licensing, target support, migration, and corruption-recovery requirements—test the exact packaged artifacts on every target.

Never expose a generic `execute_sql(sql, params)` or ORM proxy to untrusted WebView code merely to avoid writing domain commands. Even parameterized arbitrary SQL grants broad read/write/schema authority. Keep queries and transactions behind product-level Rust repository/service methods unless that full database authority is an explicit, reviewed requirement.

## Migrations

- Keep migrations immutable once released and give each a unique monotonically ordered version.
- Store non-trivial SQL in version-controlled files and embed/register it at build time where appropriate.
- Apply migrations before dependent queries, either through configured preload or a controlled load path.
- Although the plugin runs its migration batch transactionally, test upgrades from every supported prior schema and test rollback/failure behavior.
- Back up or checkpoint valuable user data before destructive transformations.
- Coordinate application version, database schema version, and updater rollback policy; an old binary may not understand a newly migrated database.

## Secrets and Stronghold

Use a secret-specific system rather than the settings store or frontend bundle. Keep the vault password out of source code and frontend storage.

- Use a current memory-hard password derivation design, unique persistent salt, and the exact key length required by the vault API.
- Do not hardcode a salt or password as tutorial examples sometimes do.
- Define unlock, lock, save, backup, corruption, reset, and password-change behavior.
- Minimize time plaintext secrets exist in JavaScript memory; prefer Rust-owned use of credentials where possible.
- Never log secret values or return them in diagnostic errors.

## OS Credential Stores

Use the operating system credential store for small authentication secrets that should follow platform login/unlock policy: macOS/iOS Keychain, Windows Credential Manager, Linux Secret Service, and Android Keystore-backed storage. Keep the API in Rust and expose product operations such as `save_session` or `sign_out`, never generic frontend read/write access to arbitrary service/account names.

The cross-platform Rust `keyring` ecosystem selects different backends/features per target and its current architecture is evolving, so verify its latest feature matrix before adoption. In particular:

- Linux desktop persistence may require a Secret Service user session and DBus libraries; headless CI and minimal window managers may have no usable store.
- Android/iOS access and background availability depend on native lifecycle and access policy.
- Biometric-bound native keys can become unavailable after enrollment or passcode/security-state changes, depending on the platform policy selected. Treat this as a recoverable product state with re-authentication, re-enrollment, backup, or explicit data-loss messaging—not an impossible error.
- Windows/Linux implementations may require serialized access; do not assume concurrent calls complete in order.
- App Sandbox, entitlements, access groups, package/bundle identity, and application renames can change credential visibility.

Store opaque tokens, not large documents. Use stable namespaced service/account identifiers, wrap secrets in redacting/zeroizing types where practical, never serialize them into command errors, and delete them on explicit sign-out. Test missing, locked, denied, corrupt, duplicate, migration, renamed-identifier, multi-user, and headless cases. Provide a typed in-memory fake for tests rather than touching a developer's live keychain.

Stronghold is an application-managed encrypted vault; an OS keychain delegates protection and unlock policy to the platform. Choose deliberately, or store only a random vault-unlock key in the OS keychain while keeping larger encrypted material in the vault. Document backup/recovery behavior: many credentials do not roam or survive OS/account changes.

## Tests

- filesystem: cancellation, traversal, symlink, scope denial, oversized file, atomic-save failure, non-ASCII/long paths;
- store: crash before save, concurrent writes, corrupt JSON, schema upgrade;
- database: clean install, sequential upgrades, failed migration rollback, concurrency, locked database, parameterization;
- secrets: wrong password, corrupt vault, missing salt, interrupted save, redacted errors.

## Authoritative Sources

- Filesystem plugin: <https://v2.tauri.app/plugin/file-system/>
- Dialog plugin: <https://v2.tauri.app/plugin/dialog/>
- SQL plugin: <https://v2.tauri.app/plugin/sql/>
- Store plugin: <https://v2.tauri.app/plugin/store/>
- Stronghold plugin: <https://v2.tauri.app/plugin/stronghold/>
- Persisted scope plugin: <https://v2.tauri.app/plugin/persisted-scope/>
