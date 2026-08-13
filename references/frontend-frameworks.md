# Frontend Framework Integration

Use this reference when adding Tauri to a frontend or diagnosing development-server, asset, routing, HMR, or mobile-device failures. Treat exact framework versions and commands as volatile; inspect the repository and current framework documentation before editing.

## Universal Model

Tauri packages static web assets and serves them in an OS WebView. It does not supply a production Node/server runtime. Prefer an SPA, static-site generation, or a classic multi-page build. If the product truly needs server rendering, keep the server remote or deliberately ship and supervise a sidecar; do not accidentally depend on framework SSR.

Verify this contract as a unit:

- `build.beforeDevCommand` starts the chosen development server;
- `build.devUrl` uses its exact fixed port;
- `build.beforeBuildCommand` produces a static build;
- `build.frontendDist` points to that build directory relative to `src-tauri`;
- client-side routing has a static fallback where the framework needs one;
- the mobile dev server binds to `TAURI_DEV_HOST` or the internal network interface, with HMR using a reachable host;
- the frontend watcher ignores `src-tauri` to prevent rebuild loops;
- Tauri and application secrets are never exposed through frontend environment prefixes.

Keep the existing package manager and lockfile. Use the repository's frozen/locked dependency workflow rather than replacing it.

## Vite: React, Vue, Svelte, Solid, Angular, and Plain Web Apps

Vite is the default low-friction choice for SPA frontends. React, Vue, Svelte, Solid, and Vite-based Angular projects share the same Tauri-side pattern even though their component and test ecosystems differ.

- Default output is commonly `dist`, so `frontendDist` is usually `../dist`; inspect overrides before assuming.
- Set `server.strictPort: true`; a silently changed port breaks `devUrl`.
- Preserve Rust compiler output with `clearScreen: false`.
- Ignore `**/src-tauri/**` in the Vite watcher.
- For physical-device mobile development, use `TAURI_DEV_HOST` for `server.host` and HMR host.
- Select browser targets compatible with the actual OS WebViews, and emit source maps for debug builds.
- Put shared `invoke` wrappers and DTOs outside components. Mock that boundary in frontend tests.

Angular CLI projects that are not Vite-based can still work if they generate static assets. Inspect `angular.json` for the actual output path, make its dev server port fixed/reachable, and avoid server-side rendering builders.

## Next.js

Use static export (`output: 'export'`) and point `frontendDist` at `../out`. A default Next production server is not present inside the packaged app.

- Treat unsupported dynamic server features, middleware, API routes, server actions, and request-time rendering as architecture mismatches.
- Configure images for static export, or replace the optimizer with a remote/build-time strategy.
- Validate asset prefixes and nested-route loading in the packaged build, not only in `next dev`.
- Browser-only Tauri APIs must run after client initialization.

## Nuxt

Generate a client/static build rather than requiring Nitro at runtime. Current official guidance uses `ssr: false`, a generated `dist`, a fixed Vite port, and an ignored `src-tauri` watch tree.

- Check the current Nuxt output contract before editing `frontendDist`.
- Bind the dev server for physical mobile devices deliberately; never ship that development exposure.
- Keep server routes and server-only composables out of the packaged-runtime dependency graph.

## SvelteKit

Use the static adapter. SPA mode with a fallback page is often the simplest Tauri arrangement because `load` functions then run in the WebView. With prerendered SSG, build-time `load` functions cannot call Tauri APIs.

- Point `frontendDist` at the adapter's `build` directory.
- If using SPA mode, set the root layout to disable SSR and configure an `index.html` fallback.
- If retaining prerendering, separate build-safe data loading from WebView-only native calls.
- Test direct navigation and reload for every routed page in the bundled app.

## Qwik and Other Meta-Frameworks

Select their static adapter/SSG output and point Tauri at the generated directory. Audit each route and integration for runtime server dependencies. Do not infer compatibility merely because development mode works.

## Rust/WASM Frontends: Trunk, Leptos, Yew, Dioxus-Web

For a Trunk-style WASM frontend, use its static `dist`, ignore `src-tauri`, and configure a reachable WebSocket HMR protocol for mobile development. The official Leptos/Trunk examples enable `withGlobalTauri` so `wasm-bindgen` can reach `window.__TAURI__`; this expands the global surface, so prefer typed Rust wrappers and enable it only when the chosen integration requires it.

- Separate WebView WASM state from native Rust core state; they are different processes/targets and do not share memory.
- Do not compile OS-native crates into `wasm32`; place native services under `src-tauri` or a shared crate with explicit target features.
- Keep serializable DTOs in a small shared crate only when both targets can compile it without platform dependencies.
- Check binary size and duplicate dependency costs; a Rust/WASM frontend plus a Rust backend may compile two dependency graphs.

## Packaged-Build Gate

Before declaring a framework integration complete:

1. build the frontend alone and inspect the output directory;
2. run its typecheck, lint, and unit tests;
3. run Tauri development mode on each targeted WebView family;
4. build a real bundle and launch it without a development server;
5. test cold start, nested-route reload, assets/fonts, offline behavior, and CSP;
6. test on a physical mobile device when mobile is supported.

