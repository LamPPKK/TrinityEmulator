# Phase 4 — WSA-like Seamless Application Windows

Duration: 16–24 weeks after task/surface and graphics protocols stabilize  
Purpose: Turn Desktop editions into an application subsystem rather than a phone-in-a-window emulator. TV editions use the single-display 10-foot path in Phase 2A.

## App discovery and launch

- Mirror launchable Android activities into a host-side catalog with icon, label, package, activity, user/profile, install state, and deep links.
- Register Windows Start-menu entries and macOS LaunchServices/Spotlight metadata through supported host mechanisms.
- Launch the subsystem on demand, wait for readiness, start the requested activity, and focus the correct task window.
- Remove stale shortcuts transactionally when apps are uninstalled or profiles are removed.

## Window lifecycle

- Map each Android top-level task to a real native AppWindow/HWND or NSWindow. Use the operating system's caption/traffic-light controls, system/Window menus, resize, minimize, maximize/fullscreen, snapping/Spaces, task switching, accessibility, and restoration rather than drawing host chrome inside Android or a custom cross-platform frame.
- Synchronize title, icon, bounds, density, orientation, minimum size, fullscreen, PiP, visibility, focus, and close/back semantics.
- Handle host close as an Android task action, not a VM shutdown.
- Treat minimize as task preservation and expose force-stop as a separate explicit command. Closing the last task follows subsystem idle policy rather than terminating userdata/runtime abruptly.
- Persist host window placement without violating Android orientation and display constraints.
- Support multiple simultaneous apps and multiple host monitors.

## Native identity and per-app presentation

- Windows gives each task native title/icon, Alt+Tab/taskbar visibility, AppUserModelID identity/grouping, Start/Search launch entry, protocol activation, Snap behavior, and per-monitor DPI handling.
- macOS gives each task AppKit title/icon/window controls, menu/command routing, Cmd+W, Window menu, Spaces/fullscreen, Retina scale, URL/file activation, state restoration, accessibility, and on-device Spotlight/`NSUserActivity` discovery.
- Gate separate macOS Dock/Launchpad identities on the Phase 0 signed/notarized launcher-shim result. If it fails, keep one LeapDroid Dock identity without reducing native NSWindow behavior or per-app Spotlight activation.
- Add native `App settings` access from the catalog, context menu, and focused window. The primary control is `Auto`, `Phone`, or `Tablet`; Advanced contains adaptive/fixed, orientation, initial/remembered size, aspect/letterbox, density/scale, fullscreen/PiP, compatibility renderer, and reset.
- Apply profiles per task/display. Phone stays below 600dp smallest width, Tablet stays at or above 600dp, and Auto follows live compact/medium/expanded bounds plus app compatibility. Non-resizable/fixed-orientation apps use Android size compatibility unless a versioned force-resize exception is explicitly enabled.
- Changing a locked profile previews the effective dp configuration and may recreate only that task. Persist last-known-good state and roll back crash loops/surface timeouts automatically.

## Input and text

- Integrate the shared Phase 4A input stack with task/window focus; this phase owns destination routing, while Phase 4A owns physical-device capture, normalization, virtio transport, IME, cursor, controllers, and recovery.
- Translate absolute touch, multitouch, and stylus input with per-window routing; route mouse, keyboard, media keys, and gamepads through their single authoritative Phase 4A backends.
- Negotiate pointer capture and relative input for games with an explicit host-owned escape gesture and atomic release on task/window focus loss.
- Attach native IME composition, candidate selection, clipboard commands, and keyboard layout changes to the focused Android `InputConnection`; prevent stale task IDs from receiving late commits.
- Preserve Android back/home/recents behavior through host commands and configurable shortcuts.
- Route clipboard target and notification activation through stable instance/user/task IDs. A late clipboard event or cold-start notification action must revalidate the current destination instead of using a stale focused task.

## Accessibility

- Phase 1 accessibility: accessible native shell, focus announcements, window titles, keyboard navigation, and screen-reader-compatible settings.
- Phase 2 accessibility: mirror Android AccessibilityNodeInfo semantics into UI Automation on Windows and NSAccessibility on macOS.
- Treat accessibility mapping as a GA gate for declared supported workflows, not an optional polish item.

## Exit criteria

- At least ten simultaneous Android tasks operate as independent host windows.
- Resize/orientation/focus/close survive repeated stress without orphan surfaces or tasks.
- Keyboard, IME, mouse/cursor/capture, touch, and gamepad pass platform-specific automation and Phase 4A latency/recovery gates.
- App shortcuts survive host reboot, product update, guest update, and Android app update.
- Basic screen-reader and keyboard-only workflows pass on both hosts.
- TV instances cannot enter per-task export and remain covered by separate TV presentation/input gates.
- Native-shell and Auto/Phone/Tablet requirements in Phase 4B pass on Windows x64/ARM64 and macOS ARM64, including the documented macOS Dock-identity disposition.

Detailed behavior and acceptance gates are in `phase-4b-native-app-experience-display-profiles.md`.
