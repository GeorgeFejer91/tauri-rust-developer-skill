# Experiment remote panels

Read this when building an experiment runner with an HTML remote viewer, or integrating one into Remote LSL Recorder. Existing apps opt in when requested; a viewer integration must not silently add remote experiment control.

## Hook and ownership

Use [Remote Panel/1](https://github.com/GeorgeFejer91/Remote-LSL-Recorder/blob/main/docs/remote-panel-profile.md) and its [schema](https://github.com/GeorgeFejer91/Remote-LSL-Recorder/blob/main/docs/remote-panel.schema.json). Generate a UTF-8 panel JSON containing exactly:

```json
{"id":"my-experiment","name":"My experiment","url":"https://example.org/my-experiment/operator/"}
```

Use a stable app/instance ID (ASCII letters, digits, underscore or hyphen; 1–64), printable name (1–80), and absolute HTTPS URL (4096 normalized bytes); bound the descriptor to 16 KB. Export it beside the existing experiment pairing controls or as a public static asset. Include no participant identifiers, native paths, secrets, or extra executable fields. Use a stable controller path: query strings and fragments are removed from remembered URLs.

This is a navigation hook. HTML stays on its HTTPS host; VDO.Ninja carries that app's data-only connection. Do not create an HTTP webhook server, generic command broker, native plugin, or arbitrary postMessage handler to register a tab.

The recorder owns its desktop workspace/catalog. Adding once on PC persists base pages and mirrors the catalog to an approved recorder phone session. Distribute current catalog metadata by revision; fetch bounded entries only after approval. Keep runtime experiment invitations in memory, preserve unchanged iframe instances across updates, and clear mirrored pages on recorder revocation. Restore settings with recording stopped and grants inactive.

Each experiment owns pairing, scopes, approval, authoritative state, command receipts, acquisition and LSL timestamps. Recorder approval does not grant experiment control. Model a single-controller limit deliberately; desktop and phone iframe pages do not share a WebRTC session. Never route experiment actions through arbitrary Tauri commands or infer them from DOM controls.

## Embeddable controller

- Support an opaque iframe with only `allow-scripts allow-forms`; grant no remote Tauri capabilities or parent DOM access. A blocked page gets an actionable explanation.
- Test actual CORS/frame headers, module loading, Web Crypto and data-only WebRTC. Storage, service workers, downloads and browser permissions may be unavailable. Do not require localStorage/IndexedDB to connect.
- With pinned VDO.Ninja SDK 1.5.5, use the recorder's reviewed [connector](https://github.com/GeorgeFejer91/Remote-LSL-Recorder/blob/main/web/external-page-connector.js) or equivalent disabled cache hooks when origin storage is denied. Preserve SDK session/signaling options and requalify hooks on upgrades.
- Opening/restoring the controller must remain disconnected. If the established app has automatic public discovery, add an explicit embedded mode that waits for Connect without changing its top-level policy. Never start an experiment from preload.
- Keep Connect, pending/applied, stale/disconnected and Stop visible. Use the existing typography fitter and reflow at 320 CSS px and enlarged text.

## Links and QR

The recorder's + tab imports panel JSON and can generate a local QR and a share link. The share URL carries UTF-8 JSON encoded as unpadded base64url in `#panel=` at the recorder companion URL. Decode and validate with the same strict importer; scrub the fragment immediately and require review/load. Do not treat the descriptor as authentication. Default sharing exports base URLs; fresh private experiment invitations remain separate and must not be sent to a QR service, analytics, query string or persistent storage.

Offer JSON/download and copy-link fallbacks when a URL exceeds QR capacity. Do not shorten a private URL through an external service or silently truncate it.

## Verification

Import a real app descriptor on desktop, approve a recorder phone session, observe automatic tab creation, preserve the controller through heartbeats, then edit/remove/revoke. Restart must restore base pages without grants. Exercise the actual sandboxed controller: explicit Connect, target-returned state/receipt, denied authority, stale revision, disconnect and suspension. Test link/QR round trips, malformed/oversized links, Unicode labels, and no secret persistence. Report deterministic/browser, public VDO route, packaged native and physical-phone evidence separately. Validate LSL/XDF alignment independently before claiming experimental timing correctness.

