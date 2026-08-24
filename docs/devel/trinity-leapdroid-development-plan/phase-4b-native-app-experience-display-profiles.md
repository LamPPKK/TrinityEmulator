# Phase 4B — Native App Experience and Per-App Display Profiles

Duration: 14–20 weeks overlapping late Phase 4, Phase 4A, and early Phase 5  
Purpose: Make Android Desktop tasks behave like first-class Windows/macOS applications and let each app receive a stable Auto, Phone, or Tablet Android configuration without affecting other tasks.

## Product contract

- Every exported Android top-level task lives in a real host top-level window: AppWindow/HWND on Windows and NSWindow on macOS.
- The host operating system draws and owns title bar/caption controls, move, resize, minimize, maximize/fullscreen, task switching, window menus, placement, DPI/Retina behavior, and native accessibility for the shell.
- Android owns application content, internal dialogs/navigation, activity/task lifecycle, resource selection, configuration callbacks, size-compatibility mode, and app-rendered surfaces.
- Do not imitate Windows/macOS chrome inside Android and do not re-skin third-party Android widgets. “Native” means correct host shell behavior and integrations around a compatible Android app surface.
- Desktop presentation defaults to `Auto`. Each app can be locked to `Phone` or `Tablet`; the setting applies only to that host user + instance + Android user + package.
- TV remains a separate television product and single-display experience. Desktop presentation profiles never change a TV instance's product characteristic.

## User-facing App settings

Expose `App settings` from:

- The Trinity/LeapDroid app catalog and search result context menu.
- The Android app's Start-menu/Spotlight activation result.
- The focused window's native system/application menu.
- The host control center's Applications page.

Primary settings stay simple:

| Setting | Values | Default/behavior |
|---|---|---|
| Device layout | Auto, Phone, Tablet | Auto; controls which Android resource/layout class the app receives |
| Orientation | App controlled, Portrait, Landscape | App controlled; locked modes are per app and reversible |
| Window size | Recommended, Last used, named native presets | Last used after a successful launch; presets describe host client area and effective dp |
| Resize behavior | Adaptive, Lock device layout | Auto uses Adaptive; Phone/Tablet default to Lock device layout |
| Fullscreen/PiP | App controlled, allowed/disabled policy | App controlled unless a compatibility rule says otherwise |
| Reset | Reset window only, reset all app presentation settings | Restores product defaults without clearing app data |

Advanced settings:

- Aspect behavior: app requested, free resize where supported, or bounded 16:9/16:10/4:3/portrait presets.
- Display scale/density: system recommended plus a small validated range; show effective dp and expected resource class before applying.
- Letterbox matte: system/native theme color or app-derived neutral color; never execute guest-provided styling code.
- Compatibility renderer and force-resize exception, each versioned and accompanied by a warning/recovery action.
- Remember monitor, position, size, fullscreen, and window state independently from Android app data.
- Per-app input/integration links to keyboard, mouse, controller, clipboard, notifications, files, microphone/camera, network, and background settings owned by their respective phases.

Activity-level overrides are deferred from first GA. Add them only for a demonstrated multi-activity package whose activities require incompatible form factors, and keep the package default authoritative otherwise.

## Presentation profile model

Define a versioned `AppPresentationProfile` keyed by:

- Host user.
- Product/instance ID.
- Android user or work profile.
- Package name.
- Optional activity override only after the later gate.

The profile records:

- User-selected device layout and whether the selection is locked.
- Orientation policy, resize behavior, aspect policy, and display scale/density preference.
- Initial and last-known-good host bounds, monitor, scale, window state, fullscreen, and PiP policy.
- Resolved effective Android width/height/smallest-width dp, density, orientation, windowing/compatibility mode, and selected renderer.
- Source and precedence of every resolved value: user, administrator policy, app manifest, compatibility database, product default, or temporary recovery fallback.
- Profile schema version, compatibility-rule version, last successful launch, failed candidate, rollback reason, and expiry for temporary overrides.

Precedence from highest to lowest:

1. Safety/recovery block that prevents a known crash loop or unusable window.
2. Explicit current user setting.
3. Enterprise/administrator restriction where the deployment supports it.
4. Versioned app compatibility rule.
5. App manifest/runtime capability.
6. Product default and last successful Auto state.

Never allow a compatibility database update to silently replace an explicit Phone/Tablet lock. It may warn, recommend a reset, or block only a proven unsafe configuration with a visible explanation.

## Auto, Phone, and Tablet resolution

### Auto — recommended

- Inspect the root activity's resizability, orientation, minimum width/height, min/max aspect, PiP, target API, multi-activity behavior, and known compatibility record.
- Give responsive apps live `Configuration` values derived from current native client bounds and density. Allow compact, medium, and expanded transitions as the user resizes.
- Use Android's width classes as the behavior boundary: compact below 600dp, medium from 600dp, and expanded from 840dp. Width and height remain independent inputs.
- Preserve the last stable size on relaunch. If no history exists, choose a recommended starting size from manifest/runtime evidence and app category, not package-name heuristics alone.
- For constrained legacy apps, select the smallest safe canvas and Android size-compatibility mode rather than forcing arbitrary live resize.

### Phone

- Keep `smallestScreenWidthDp` below 600 so the app selects compact/phone resources even when its native host window is large.
- Offer portrait and landscape starting canvases, but derive exact pixels from host scale and selected Android density rather than pretending to be a specific branded phone.
- When the host window grows, scale or letterbox the compact Android canvas with correct input transforms. Do not cross into tablet resources unless the user switches to Auto or Tablet.
- When the host window shrinks below the safe canvas, enforce a native minimum size or use bounded down-scaling with readable warnings and pixel-perfect input mapping.

### Tablet

- Keep `smallestScreenWidthDp` at least 600 and use 840dp as the expanded-width breakpoint where the window can provide it.
- Start with a landscape-friendly medium/expanded canvas while respecting portrait-only apps through Android compatibility behavior.
- Prevent native resizing below the validated minimum client area where platform conventions permit. Otherwise retain the tablet configuration and letterbox/scale instead of reporting contradictory phone dimensions.
- Do not advertise tablet-only hardware, model, certification, telephony, sensors, or Play properties. Tablet detection is based on truthful per-task window configuration.

## Android task/display architecture

Phase 0 compares two designs before implementation is frozen:

1. Shared guest display with per-task configuration/window-container overrides.
2. A bounded virtual display and TaskDisplayArea for each exported host task.

Selection criteria:

- Correct `screenWidthDp`, `screenHeightDp`, `smallestScreenWidthDp`, density, orientation, `screenLayout`, windowing mode, insets, and compatibility behavior.
- No configuration, focus, input, clipboard, notification, surface, or lifecycle leakage between simultaneous apps/users/instances.
- Correct multi-activity task behavior, activity launch into existing tasks, PiP, dialogs, permission UI, IME, camera rotation, and relaunch.
- SurfaceFlinger/Gfxstream surface count, host/guest memory, command latency, resize churn, frame pacing, and suspend/resume cost.
- Compatibility with Android 17 per-display desktop windowing and its TaskDisplayArea source of truth.
- Recovery after guest/renderer/AppShell crash without orphan displays, handles, windows, or task records.

WindowManager remains authoritative. The host requests a bounded presentation intent and waits for the guest to return the actually applied configuration. The host never claims a profile was applied until task/display ID, revision, configuration, and first correctly sized surface are acknowledged.

## Windows-native experience

- Use WinUI 3 for the control center/settings and a real AppWindow/HWND for every Android task.
- Retain the standard Windows 11 caption buttons, system menu, borders, resize hit targets, keyboard movement, Snap Layouts, Snap groups, minimize/maximize/restore, and fullscreen behavior. Customize only supported title/icon/color regions and retain high-contrast behavior.
- Show every eligible task in Alt+Tab and the taskbar with Android app label/icon. Use a stable per-package/per-user AppUserModelID for grouping and activation without creating a VM per app.
- Create/remove per-app Start/Search shortcuts transactionally on Android install/update/uninstall and user/profile changes. Pinning remains a user-controlled Windows action.
- Route shortcut, protocol, file, notification, and command-line activation to the correct instance/user/package/activity; cold-start the shared subsystem, deduplicate activation, then focus or create the matching native window.
- Respect per-monitor DPI, work areas, virtual desktops, taskbar location, high contrast, reduced motion, text scaling, screen readers, keyboard access, and window-placement restoration.
- Map close to Android task finish/removal, not VM shutdown. Minimize preserves the task. `Force stop Android app`, `App settings`, `Back`, `Home`, and `Restart app` remain separate explicit commands.

## macOS-native experience

- Use SwiftUI/AppKit for settings and a real NSWindow with CAMetalLayer-backed content for each Android task.
- Retain native traffic-light controls, title/represented identity where appropriate, resize/minimum size, zoom, minimize, fullscreen, Spaces, window tabbing policy, Retina backing scale, state restoration, and VoiceOver semantics.
- Update the application menu context from the focused Android task while preserving standard About/Settings/Services/Hide/Quit, Edit clipboard commands, View, Window, and Help conventions. Cmd+W closes the focused task; force-stop/restart are distinct commands.
- Add app launch items to the local Core Spotlight index and use NSUserActivity or declared URL activation to restore the correct package/activity. Keep the index on device and remove stale entries on uninstall/profile removal.
- Integrate native file/URL open, drag/drop, Share/Services where supportable, notification activation, Dock recent behavior, and AppKit permission/file panels through existing brokers.
- Treat separate Dock/Launchpad identity for each dynamic Android package as a hard capability gate. The candidate launcher shim must be signed/notarized, immutable in executable content, App-Sandbox-compatible, updateable, removable, and unable to expand guest/host authority.
- If that gate fails, ship one LeapDroid Dock identity with separate native NSWindows, focused app title/icon/menu context, and individual Spotlight launch results. Document this honestly; do not create unsigned/generated `.app` bundles or request broad Gatekeeper exceptions.

## Resize, relaunch, and rollback lifecycle

- During interactive resize, host movement/chrome/cursor remain local and smooth. Coalesce configuration requests without losing the final size, orientation, monitor scale, or profile revision.
- Responsive apps receive normal Android configuration/lifecycle dispatch. Apps that do not handle a change may recreate their activity according to Android rules.
- Fixed Phone/Tablet profiles preserve their Android canvas class while host bounds change; input and cursor transforms use the exact content rectangle including letterbox offsets.
- Before a profile change that requires recreation, show the effective form factor/dp/orientation and save the current host placement plus last-known-good configuration.
- Recreate only the affected task/display. Do not reboot the guest, stop unrelated apps, clear app data, or mutate the TV product.
- Declare success only after guest configuration acknowledgement, task health, and a correctly sized first surface. On timeout, repeated crash, black surface, or impossible bounds, close the candidate display, restore the previous profile/placement, and offer safe Auto mode.
- App update, guest update, host update, snapshot restore, and compatibility-database migration retain explicit user profiles unless a schema or safety rule requires a visible migration.

## Security and privacy

- Validate icon/title/menu metadata, window bounds, insets, custom colors, launch URIs, profile fields, and display acknowledgements in the least-privilege AppShell/presentation broker.
- Bound visible task count, virtual display count, title/icon bytes, configuration rate, dp/density range, resize queue depth, profile file size, restart rate, and placement coordinates.
- Never let an Android package set a raw AppUserModelID, bundle identifier, executable path, file association, entitlement, URL scheme, or host command. Derive identities from trusted package/user records and escape all display metadata.
- Per-app launch shortcuts/Spotlight items expose only label/icon and opaque product activation identifiers; no Android private data or stable cross-install tracking ID is indexed.
- Profile settings and locally observed window behavior remain local diagnostics; zero-telemetry and Phase 6A process isolation apply to every AppShell and launcher shim.
- A generated/unsigned macOS application bundle, runtime code signing with a user key, Gatekeeper bypass, or entitlement widening is prohibited.

## Performance budgets

- Interactive host window movement and resize chrome never wait for a guest round trip.
- Host resize to guest configuration acknowledgement: under 100 ms p95 on reference hardware.
- Final resize event to first correctly sized Android frame: under 250 ms p95 for responsive reference apps.
- Crossing compact/medium/expanded boundaries may recreate activities, but UI remains responsive and the transition has visible progress only when it exceeds the normal resize budget.
- Phone/Tablet scaling adds no extra full-frame CPU copy on the accelerated path; letterbox and transform composition remain in the renderer/surface pipeline.
- A profile change requiring task recreation stays within the warm app-launch target; automatic rollback adds no guest reboot.
- Ten simultaneous visible AppShell windows remain within the main memory/frame-pacing budgets; the Phase 0 architecture comparison sets a per-window surface/display memory cap.

## Validation matrix

- Windows: caption/system menu, Snap/maximize/fullscreen, Alt+Tab/taskbar grouping, AppUserModelID, Start/Search shortcut lifecycle, activation deduplication, per-monitor DPI, virtual desktops, high contrast, text scaling, Narrator/UI Automation, input/focus, and placement restoration.
- macOS: traffic lights, menu bar/Window menu/Services, Cmd commands, minimize/zoom/fullscreen/Spaces, Retina/multi-display, Spotlight/NSUserActivity/URL-file activation, VoiceOver/NSAccessibility, restoration, Dock fallback, and launcher-shim gate where enabled.
- Android configurations: widths around 599/600/839/840dp, independent height classes, portrait/landscape, density range, resizable/non-resizable, min/max aspect, PiP, multi-activity, dialogs, IME, camera, games, legacy apps, Compose-adaptive, and View-adaptive layouts.
- Isolation: two apps with different profiles, two Android users/work profiles, two subsystem instances, profile changes during notification/file activation, snapshot/restore, and app/guest/host update.
- Recovery: invalid bounds/density, stale task/display revision, guest refusal, renderer/AppShell crash, black/late surface, crash loop, profile corruption, monitor removal, update migration failure, and forced subsystem shutdown.
- Security: hostile labels/icons/URIs, identity collision, shortcut/Spotlight injection, configuration floods, virtual-display exhaustion, cross-user activation, unsigned macOS shim, IPC replay, and malformed profile migration.

## Delivery sequence

1. Freeze native-shell boundary, close/minimize/quit semantics, profile schema/precedence, Android configuration mappings, and macOS Dock-identity gate.
2. Select the shared-display/per-task-override or per-task virtual-display/TaskDisplayArea architecture through Phase 0 correctness and performance evidence.
3. Implement the host-neutral presentation protocol, profile store/resolver, last-known-good transaction, and conformance fixtures.
4. Deliver real AppWindow/HWND and NSWindow shells with surface, focus, lifecycle, placement, DPI/Retina, and basic accessibility.
5. Deliver Windows AppUserModelID/Start/Search/activation and macOS menu/Spotlight/NSUserActivity/URL-file activation, including install/update/uninstall reconciliation.
6. Deliver Auto with live responsive resize and Android size-compatibility fallback.
7. Deliver locked Phone/Tablet profiles, native settings UI, transactional recreate/rollback, and compatibility database integration.
8. Complete platform accessibility, multi-monitor/Spaces/virtual-desktop, fuzz/security, performance, app-matrix, and update-migration gates.
9. Graduate any macOS launcher shim only after signing/notarization/App Sandbox/security/update tests; otherwise permanently select and document the fallback for that release.

## Deliverables

- Native-shell behavior specification and Windows/macOS parity/capability matrix.
- Windows AppWindow/AppUserModelID/Start/Search/activation design and lifecycle tests.
- macOS NSWindow/menu/Spotlight/NSUserActivity design plus signed launcher-shim feasibility disposition.
- Versioned `AppPresentationProfile` schema, precedence/migration rules, local storage boundaries, and administrator policy surface.
- Android task/display architecture ADR with configuration correctness, isolation, graphics/memory/latency measurements, and rollback design.
- Auto/Phone/Tablet UX specification, effective-dp preview, compatibility warnings, reset/recovery flow, and accessibility review.
- Cross-platform conformance suite, adaptive/legacy app corpus, resize/profile performance dashboard, and security/fuzz evidence.

## Exit criteria

- Every Desktop task is hosted in a real AppWindow/HWND or NSWindow with native chrome, commands, task switching/window management, DPI/Retina behavior, restoration, and accessible shell semantics.
- No custom-drawn Windows/macOS title bar, global phone framebuffer, or Android-window decoration is used for the Desktop host shell.
- Windows per-app Start/Search/taskbar identity and activation survive install/update/uninstall, reboot, guest update, and multi-user/profile use without identity collisions or one VM per app.
- macOS provides native NSWindows and per-app on-device Spotlight activation; separate Dock/Launchpad identity ships only if the signed/notarized launcher-shim gate passes and otherwise uses the documented single-Dock fallback.
- Auto, Phone, and Tablet produce correct per-task Android configurations and compact/medium/expanded resources without changing another app, Android user/profile, instance, or TV product.
- Non-resizable/fixed-orientation apps use correct Android size-compatibility behavior, and explicit force-resize exceptions are versioned, reversible, and app-specific.
- Profile changes are transactional, recreate only the affected task when required, preserve userdata/placement, and automatically restore last-known-good Auto or prior state after failure.
- Ten simultaneous apps pass native window, input/focus, accessibility, profile isolation, multi-monitor/scale, update migration, crash recovery, frame pacing, memory, and resize-latency gates on Windows x64/ARM64 and macOS ARM64.
- Native-shell/AppShell processes and any approved launcher shim pass Phase 6A sandbox, zero-telemetry, identity-injection, IPC, resource-exhaustion, and independent security review gates.
