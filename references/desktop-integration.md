# Desktop Integration

Use this guide for windows, WebViews, native title bars, splash screens, system trays, menus, and application lifecycle.

## Windows and WebViews

- Define predictable startup windows in Tauri configuration; use builders for windows created from runtime data or platform events.
- Treat the window label as a stable security and routing identity. Keep labels aligned with capabilities and avoid user-controlled labels.
- Decide whether closing the last window exits, hides to the tray, or keeps background work alive. Make the behavior visible and consistent with platform conventions.
- Keep per-window frontend routing, state restoration, event subscriptions, and cleanup explicit.
- Restrict privileged windows separately. A settings, login, or remote-content window should not inherit the main window's full capability set.
- Store only window-independent domain state globally. Keep ephemeral view state in the owning frontend or a window-keyed native structure.

When creating windows asynchronously, listen for created/error events and handle duplicate labels. Clean up listeners and native resources when windows are destroyed.

## Custom Title Bars

Custom decorations trade native behavior for visual control. Before disabling decorations, evaluate accessibility, touch/pen input, window snapping, macOS alignment/movement features, keyboard behavior, and high-DPI layouts.

- Grant only the exact window operations used, such as close, minimize, toggle maximize, or start dragging.
- `data-tauri-drag-region` applies to the element carrying it, not automatically to every child; keep buttons and inputs interactive.
- Prefer native or transparent title-bar options when they preserve required platform behavior.
- Isolate platform-native code behind `cfg` modules. If an implementation needs `unsafe`, document native-pointer lifetime and thread-affinity invariants and keep the block minimal.
- Test resize edges, double-click behavior, full screen, multiple displays, keyboard navigation, and screen-reader labels.

## Native Menus

Use stable, namespaced item identifiers and route menu actions into the same application services used by commands and shortcuts. Prefer predefined native edit items for copy, paste, undo, redo, and selection behavior.

- On macOS, organize the application menu as submenus; top-level items are not treated the same as on Windows/Linux, and the first submenu has application-menu significance.
- Keep checked/enabled/text state synchronized from one state owner.
- Load icon formats only when the matching Tauri image feature is enabled.
- Avoid `unwrap` in menu callbacks; callbacks can outlive assumptions about windows and state.
- Test accelerators for conflicts and platform conventions.

## System Tray

Enable the required Tauri tray feature and choose one owner—Rust for application lifecycle/background behavior, or JavaScript for presentation-led behavior. Do not create duplicate tray icons during hot reload or window recreation.

- Build tray menus from stable IDs and handle unknown events safely.
- Decide whether left click opens a menu, restores/focuses a window, or does nothing.
- Unminimize, show, and focus defensively because the target window may no longer exist.
- Tray mouse events are not uniformly supported; current official guidance notes that Linux may show the icon/menu without emitting those pointer events. Provide menu-based behavior and test each target desktop.
- Remove or update the tray when application state changes, and ensure quit terminates supervised tasks cleanly.

## Startup and Splash Screens

First try to make startup incremental: render the main UI, show progress, and load non-critical services in the background. Use a splash screen only for a product requirement or a true startup gate.

If a splash is justified:

1. create a visible splash and hidden main window;
2. start native initialization from `setup` using non-blocking async work;
3. let frontend and native tasks report typed completion or failure into a coordinator;
4. publish safe progress instead of sleeping or polling;
5. once required tasks complete, close the splash and show/focus the main window;
6. present a recoverable error or quit cleanly if startup fails.

Do not block an async runtime thread with `std::thread::sleep`. Do not panic on an unrecognized completion name. Use an enum or validated task ID, avoid `unwrap`, and release state locks before manipulating windows or emitting events.

## Verification

- test cold start, repeated window creation, close/reopen, hide-to-tray, explicit quit, and abnormal startup failure;
- test each supported desktop because tray/menu/title-bar behavior differs;
- verify capability labels against every static and dynamic window label;
- check listener cleanup and background-task shutdown;
- exercise keyboard-only operation and native accessibility expectations.

## Authoritative Sources

- Window customization: <https://v2.tauri.app/learn/window-customization/>
- Window menus: <https://v2.tauri.app/learn/window-menu/>
- System tray: <https://v2.tauri.app/learn/system-tray/>
- Splash screen: <https://v2.tauri.app/learn/splashscreen/>
