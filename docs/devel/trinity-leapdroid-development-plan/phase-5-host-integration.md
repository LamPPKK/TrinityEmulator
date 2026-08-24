# Phase 5 — Native Host Integration for Desktop and TV

Duration: 20–28 weeks in parallel platform tracks  
Purpose: Reach the practical desktop-integration level users associate with WSA.

## Shared guest-facing features

- Integrate the Phase 5A clipboard broker: current text first, then sanitized HTML/bounded PNG, per-direction and active-instance policy, sensitive blocking, loop prevention, and file-broker separation.
- Host folder access through a broker and Android DocumentsProvider.
- Drag and drop between host and Android windows.
- Integrate the Phase 5A notification broker: guest lifecycle snapshots/deltas, host update/removal, safe open/actions/direct reply, own-notification dismissal reconciliation, privacy controls, and per-app/channel settings.
- Android intents mapped to host links, files, and share targets; host links optionally routed into Android apps.
- Audio output/input, camera, location, Phase 4A input/controller state, and selected virtual sensors.
- Developer mode with paired ADB, local log collection, port forwarding, and previewable/redacted diagnostic bundles exported only by explicit user action.
- Expose immutable `ServiceMode=NoApp|microG|GApps` during instance creation. Surface F-Droid state and microG opt-ins only in `microG`; provide a user-file picker, inspection result, license/certification warning, progress, restart, rollback, and removal UX for the payload-free GApps provider.
- Expose independent `RootMode=Off|KernelSU|Magisk` under Advanced/Developer with a persistent rooted warning, default-deny grant queue, provider/manager/build provenance, module safe mode, host-owned recovery, and one-click boot to clean `Off` recovery—not root concealment or bypass controls.
- Add edition-aware instance creation and management. Desktop and TV receive separate userdata, shortcuts, update channels, settings surfaces, and compatibility reporting.
- Add a native per-app settings surface shared at the policy/schema level but rendered with WinUI 3 on Windows and SwiftUI/AppKit on macOS. Expose Auto/Phone/Tablet first, hide risky compatibility overrides under Advanced, show the effective dp/orientation/profile, and provide preview/relaunch/reset/last-known-good recovery.
- For TV, provide one remote-friendly full-screen/windowed display, display-profile selector, D-pad/gamepad/media-key routing, push-to-talk, playback sleep policy, and a host recovery overlay.

## Trinity Windows native track

- C++/WinRT and WinUI 3 settings/control center.
- Native x64 and ARM64 binaries for UI, services, VM controller, renderer, updater, and diagnostics.
- Isolate the control-center UI, per-task AppShell/presentation brokers, VM controller, renderer, media/decoders, updater, diagnostics, clipboard, notifications, files, camera/microphone/location, and Native Bridge/GApps/root-artifact inspection with restricted tokens, AppContainer/LPAC where compatible, Job Objects, narrow IPC ACLs, process mitigations, and per-process egress rules.
- MSIX bundle with architecture-specific payloads, clean uninstall, repair, and signed update channels.
- Windows App SDK notifications with stable tag/group/ID and own-notification enumeration/removal; Win32 clipboard listener/format conversion; one real AppWindow/HWND per Android task; native title bar/system menu/Snap/Alt+Tab; per-app icon, AppUserModelID, Start/Search shortcut and activation; URI/file associations; per-monitor DPI; virtual-desktop behavior; power notifications; and Windows permission prompts.
- WASAPI audio, Media Foundation camera/video paths where appropriate, Windows location, clipboard, foreground Raw Input keyboard/mouse, GameInput controllers/haptics, native text-service integration, cursor/capture, and accessibility adapters.
- Per-user service model with no always-elevated UI; perform privileged setup only through a narrowly scoped installer/service boundary.
- Add an ARM Compatibility panel with `Off`, `Houdini`, and `libndk_translation`; show provider source, version, hash, ABI/API compatibility, validation result, legal acknowledgement, active-app impact, and restart requirement.
- Run provider import and inspection in a low-privilege worker, scan with Windows security services, copy accepted files into protected versioned storage, and never automatically locate or download proprietary payloads.
- TV-specific Windows integration uses Media Foundation/D3D11 video surfaces, WASAPI stereo/multichannel PCM, Windows game input/media keys, display refresh/full-screen events, and monitor power notifications.

## LeapDroid macOS ARM64 native track

- Swift/SwiftUI settings and launcher with AppKit windows for performance-sensitive presentation.
- Native arm64 VM controller, renderer, updater, diagnostics, and Objective-C++ adapters.
- Isolate the control-center UI, per-task AppShell/presentation brokers, VM controller, renderer, media/decoders, updater, diagnostics, clipboard, notifications, files, camera/microphone/location, and Native Bridge/GApps/root-artifact import inspection into least-entitled App Sandbox/XPC processes where compatible; use Hardened Runtime and security-scoped bookmarks, and document every unavoidable entitlement exception.
- Signed, notarized app bundle with hardened runtime and required hypervisor/virtualization entitlements.
- NSWindow/CAMetalLayer presentation with native traffic-light controls, menu bar/Window menu/Services/Cmd commands, Spaces/fullscreen, Retina scaling and restoration; Dock/LaunchServices plus local Spotlight/`NSUserActivity` activation; capability-gated signed launcher shims for separate per-package Dock identity; UserNotifications notification/action/reply/dismiss adapter; NSPasteboard change/access adapter; file-provider/security-scoped bookmarks; CoreAudio, AVFoundation, CoreLocation, AppKit/NSTextInputClient keyboard/mouse/IME/cursor capture, GameController controllers/haptics, and accessibility adapters.
- Host sleep/wake, display changes, thermal pressure, memory pressure, and app termination coordination.
- TV-specific macOS integration uses VideoToolbox/Metal-compatible surfaces, CoreAudio stereo/multichannel PCM, GameController/media keys, display refresh/full-screen events, and playback-aware sleep assertions.

## Policy and privacy

- Global and per-app toggles for clipboard, files, microphone, camera, location, local network, notifications, and background execution.
- Clear host indicators for active microphone/camera and developer connections.
- TV push-to-talk and playback sleep inhibition are visible, revocable, and scoped to the active TV instance.
- Clipboard and notification permissions expose master, per-instance, per-direction, preview, app/channel, action/reply, and wake-on-activation controls. TV defaults remain off/redacted and D-pad-operable.
- Guest access must never exceed permissions granted to the host application.
- Production hosts make no automatic analytics, crash-report, remote-log, usage, or experiment request. Settings expose the first-party egress manifest and a local diagnostics viewer/export flow, not a telemetry opt-out switch.
- Ordinary input is window-scoped and foreground-only. Do not request host Accessibility/global-monitoring permission merely to capture keyboard or mouse input; expose any future accessibility automation as a separately reviewed feature.

## Exit criteria

- Host integrations pass permission-denied, revocation, device-removal, and sleep/wake tests.
- Native packages meet signing, notarization/store, update, repair, and uninstall requirements.
- No integration requires disabling host security features.
- Privacy settings are enforced across guest restart and product update.
- All service/root choices, warnings, imports, opt-ins, grant queues, safe mode, rollback, and recovery work natively on both hosts; `NoApp + Off` contains no provider residue, disabling microG stops its opted-in network functions, and switching/removing an ARM provider leaves no advertised ABI or translated-code cache behind.
- Desktop and TV packages can coexist, update, recover, and uninstall without crossing userdata or edition policy.
- Input backends, IME, pointer capture/recovery, native cursor synchronization, eight-controller hot-plug, and basic haptics pass the Phase 4A matrix on each host architecture.
- Clipboard and notifications pass the Phase 5A loop, redaction, reconciliation, permission, stale-action, restart, latency, and energy gates on every host architecture and edition.
- Native-window behavior, app discovery/activation, close/minimize/force-stop semantics, per-monitor/Retina restoration, accessibility, and Auto/Phone/Tablet settings pass Phase 4B on both hosts; macOS UI and documentation match the proven Dock-identity capability.
- Runtime network captures match the declared functional egress manifest, all host helpers enforce their approved capabilities/resource limits, and the Phase 6A security/performance gates pass without a global sandbox bypass.
