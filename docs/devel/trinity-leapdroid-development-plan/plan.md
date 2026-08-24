# Trinity Windows + LeapDroid macOS Desktop and Android TV Development Plan

Status: Draft for review  
Prepared: 2026-08-24  
Revised: 2026-08-25 — three service modes (`NoApp`, `microG`, user-installed `GApps`), KernelSU/Magisk developer root, WSA/Waydroid/LineageOS/GrapheneOS audits, ARM Native Bridge, Android TV, native per-app host UX, Phone/Tablet display profiles, unified input, clipboard, notifications, zero telemetry, and layered sandboxing
Planning horizon: 20–28 months with a 15–19 person core team  
Products: Trinity Desktop + TV for Windows x64/ARM64; LeapDroid Desktop + TV for macOS ARM64

## 1. Executive decision

Develop two native host products and two UX editions on top of one shared Android runtime instead of maintaining separate emulator stacks.

- Trinity remains the Windows product. Its QEMU and `direct-express` work are migrated to a current runtime. Its graphics-projection path becomes an optional high-performance backend after a compatibility-first renderer is stable.
- LeapDroid becomes the macOS product. The existing LeapGL repository is retained as historical/reference code, while the shipping runtime is rewritten for Apple Silicon around AArch64 virtualization and Metal-backed graphics.
- Both products share the Android guest, host/guest control protocol, app catalog, lifecycle manager, file/clipboard/notification bridges, test suites, update format, local diagnostic schema, and compatibility database.
- Native host presentation remains platform-specific: C++/WinRT + WinUI 3 on Windows and Swift/SwiftUI/AppKit on macOS.
- Microsoft has not published the WSA runtime, host shell, integration services, or graphics stack. Use WSA as a documented behavior/UX reference only. Audit the separately published WSA Linux kernel and reuse only relevant, licensed kernel changes after rebasing them onto the selected current AOSP/GKI kernel.
- Offer exactly three immutable per-instance service modes: `NoApp` (clean AOSP, default), `microG` (verified F-Droid + microG), and `GApps`. GApps is either an advanced uncertified add-on imported by the user or a separately licensed `CertifiedPartner` artifact for `RootMode=Off`; Trinity/LeapDroid never treat user import as certification, mirror/search/download/extract payloads, or grant rights to proprietary Google packages. microG and GApps never coexist.
- Offer an independent advanced `RootMode`: `Off` by default, or mutually exclusive KernelSU/Magisk Developer variants built from pinned source. Root never disables verified boot or SELinux, grants host capability, or supports root concealment, attestation/Play Integrity/SafetyNet/DRM/anti-cheat bypass.
- On Windows x64, support ARM-only APKs through a pluggable AOSP Native Bridge layer with user-supplied Houdini or `libndk_translation` packages; neither package is bundled or downloaded by Trinity.
- Use Waydroid and LineageOS as selective upstream references, not wholesale runtime bases. Waydroid contributes guest HAL/session/image lessons; LineageOS contributes reviewed framework, microG, TV SystemUI, product-overlay, and device-support patterns.
- Use GrapheneOS as a selective security reference over the AOSP baseline. Audit hardened SELinux/seccomp policy, network/sensor controls, secure spawning, allocator/memory hardening, dynamic-code restrictions, and local crash/log tooling; do not adopt the full GrapheneOS platform, Pixel-specific code, services, branding, or compatibility claims.
- Ship a Desktop edition and a TV edition on every host/architecture target. They share the VM, kernel, vendor image, graphics, services, and protocol, but use distinct Android product images and userdata because TV is a fixed `television` product characteristic with a 10-foot launcher and input model.
- Synchronize the current clipboard and Android notifications through dedicated least-privilege brokers. Clipboard changes are bidirectional and loop-free; notification posting, updates, removals, opening, supported actions, direct reply, and explicit dismissal are reconciled without exposing Android `PendingIntent` objects or requesting access to unrelated host notifications.
- Make every Desktop Android task feel host-native: a real top-level Windows `AppWindow`/HWND or macOS `NSWindow`, native title bar/caption controls, task switching/window menus, shortcuts, drag/drop, DPI/Retina behavior, lifecycle, and host discovery/activation. Android still renders the application's internal content; the project does not re-skin arbitrary app controls.
- Add per-app Desktop presentation profiles: `Auto` (recommended), `Phone`, and `Tablet`, with orientation, initial/remembered size, adaptive versus fixed form factor, aspect/letterbox, density/scale, fullscreen/PiP, and reset controls. Apply the resulting Android `Configuration` per task/display rather than changing the whole guest.
- Production Trinity/LeapDroid builds emit no automatic telemetry: no analytics SDK, usage tracking, crash upload, remote logging, experimentation client, stable installation identifier, or hidden metrics endpoint. Diagnostics remain local, bounded, redacted, user-viewable, and exported only through an explicit user action.

This is the shortest credible route to a WSA-like product. Updating Trinity and LeapGL independently would duplicate the majority of the difficult work and produce incompatible guest images.

## 2. Evidence and constraints

### Confirmed from the repositories

- Trinity is based on QEMU 5.0 and adds `hw/direct-express`, `hw/express-gpu`, custom input handling, and an `android64` CPU model.
- Trinity currently builds primarily for Windows/MSYS2 and uses WHPX or HAXM. Its released guest is Android-x86 9.
- Trinity's open repository contains host-side transport and renderer code, but not the complete modern guest projection library and guest kernel integration described by the paper.
- LeapDroid's repository contains only the older LeapGL host renderer, translator, codecs, tools, and tests. It does not contain the complete LeapDroid VM, Android guest, or product shell.
- LeapGL targets the AOSP 4.4-era EGL/GLES remoting design and is not an adequate base for modern GLES/Vulkan support.
- Trinity's current input is embedded in `express_gpu_render.c`: GLFW callbacks map keys to QEMU qcodes and send absolute mouse/buttons/wheel, then synchronize from the render loop. It supports only a narrow desktop path, mixes rendering/input/macro-like replay state, and lacks host IME, relative capture, cursor synchronization, device identity, robust focus teardown, and game-controller backends. Preserve behavioral tests only; replace this implementation.
- LeapDroid contains no shipping host input subsystem. Its `tests/event_injector` sends legacy emulator-console `event send` key and absolute-touch commands; retain it only as historical test intent, not as an architecture or production transport.
- Neither repository contains a product clipboard bridge or Android-to-host notification bridge. Trinity matches only generic QEMU notifier infrastructure, and LeapDroid contains no relevant implementation; both integrations are new shared-runtime work.

### Confirmed from current platform documentation

- Windows Hypervisor Platform exposes Arm64 capability discovery; current QEMU build-platform documentation lists WHPX for Arm and x86.
- Apple Hypervisor and Virtualization frameworks support ARM64 Linux guests on Apple Silicon.
- ANGLE supports Windows backends and a complete Metal backend on macOS for GLES 2.0/3.0, with broader support depending on the backend.
- AOSP Cuttlefish uses Gfxstream for accelerated GLES/Vulkan forwarding and SwiftShader as a software fallback.
- Android CTS and VTS are intended to run continuously against device and emulator implementations.
- The public `microsoft/WSA` repository is an issue/discussion and developer-documentation repository, not the shipping subsystem source. Microsoft separately publishes `WSA-Linux-Kernel`, including x86_64 and arm64 WSA configurations. Therefore “WSA-like” means documented feature/experience parity; only the kernel source is a potential code input.
- AOSP exposes `libnativebridge` so a foreign-ISA translator can integrate with ART, but explicitly does not provide an actual translation implementation.
- microG GmsCore is an Apache-2.0 free implementation of selected Play Services APIs and requires ROM-supported signature spoofing. F-Droid Client is GPL-3.0-or-later and validates repository index signatures and APK hashes.
- Android compatibility does not automatically grant Google Play/GMS access. Google documents that Play/GMS licensing is separate and that only Play Protect-certified devices are eligible to include licensed Google applications; imported GApps therefore remain uncertified and best-effort.
- KernelSU supports a GKI-oriented kernel root model and separately documents a current x86_64 path that can require disabling syscall hardening. That weakening is unacceptable for a production artifact, so KernelSU is target-gated. Magisk patches boot/init artifacts and can inject through Zygisk; Trinity/LeapDroid use reproducible separate boot variants, keep Zygisk off by default, and never disable AVB verification to integrate it.
- Waydroid runs Android in Linux namespaces/LXC with direct host-kernel and Wayland access, and its current guest work is LineageOS-derived. Those host assumptions do not transfer to Windows or macOS, but its Android HAL, session, image-split, app-launch, and multi-window work are relevant audit inputs.
- Current LineageOS source includes a restricted microG signature-spoofing change and maintained Android TV components including `android_device_lineage_atv` and `TvSystemUI`.
- AOSP already isolates every app with a kernel-enforced per-UID sandbox and strengthens it with per-app SELinux domains and `seccomp-bpf`; native code is subject to the same sandbox boundary. GrapheneOS builds on AOSP with additional attack-surface reduction, exploit mitigations, hardened SELinux/seccomp, per-app network/sensor controls, and user-controlled local log capture instead of automated crash reporting.
- Windows AppContainer provides least-privilege isolation for files, devices, network, processes, credentials, and windows; Job Objects provide process-tree resource limits/accounting. macOS App Sandbox limits resource access through entitlements, Hardened Runtime blocks classes of code injection/tampering, and XPC supports on-demand least-privilege helper processes.
- AOSP publishes TV product definitions for x86 and arm64 under `device/google/atv`, including TV product overlays, provisioning, Leanback components, and TV-specific SELinux/configuration inputs.
- Virtio input transports standard Linux evdev events and exposes device capabilities, identity, absolute-axis ranges, an event queue, and a status queue. It is the cross-platform transport for virtual keyboards, pointing devices, and game controllers; composition text and richer bidirectional metadata use the versioned control protocol.
- Windows Raw Input supplies per-device keyboard/mouse scan data and high-rate mouse deltas, while GameInput supplies hot-plugged controller state, standardized and raw device readings, haptics, and device callbacks. On macOS, window-scoped `NSEvent`/`NSTextInputClient` handle pointer, keyboard, and IME, while GameController handles controllers.
- Windows can observe current-clipboard changes through `AddClipboardFormatListener`/`WM_CLIPBOARDUPDATE`. AppNotificationManager can post actionable notifications and enumerate/remove only the calling app's notifications by stable ID/tag/group, so Trinity can reconcile its forwarded Android notifications without the privacy-heavy Windows all-notifications listener.
- Windows App SDK maps each `AppWindow` 1:1 to a top-level HWND and can expose it in Alt+Tab/taskbar, set per-window title/taskbar icons, and participate in native caption/snap behavior. Windows AppUserModelIDs can give windows distinct taskbar grouping/identity and per-app shortcuts can activate the shared subsystem through versioned launch arguments.
- Android exposes primary-clipboard change callbacks but restricts public background reads, so the guest bridge must be a narrowly privileged platform component without weakening access for ordinary apps. Android also marks remote/sensitive clips in `ClipDescription`. `NotificationListenerService` exposes active notification lifecycle, while Android notification actions and `RemoteInput` remain guest-owned capabilities.
- macOS uses `NSPasteboard`; its `changeCount` identifies ownership/content changes and its current access policy is user-controlled. UserNotifications supports stable identifiers, actions, text input, explicit dismiss callbacks, enumeration, replacement, and removal of the host app's delivered notifications.
- AppKit provides one `NSWindow` per onscreen host window plus native menu/window management, URL/file activation, state restoration, accessibility, and Spotlight indexing through Core Spotlight/`NSUserActivity`. Separate Dock/Launchpad identity for dynamically installed Android packages is not guaranteed by those APIs; a signed/notarized per-app launcher-shim design must pass Phase 0 or the supported fallback is one LeapDroid Dock identity with native per-window titles/icons and individual Spotlight launch results.
- Android selects resources and adaptive layouts from per-window `Configuration` values such as `screenWidthDp`, `screenHeightDp`, and `smallestScreenWidthDp`. Current guidance uses compact below 600dp, medium from 600dp, and expanded from 840dp; non-resizable/orientation-constrained apps can use size-compatibility letterboxing. Android 17 also provides per-display desktop windowing whose state is owned by the display `TaskDisplayArea`.

### Assumptions

- Minimum Windows target: Windows 11 on x64 and ARM64; the exact build floor is fixed after the WHPX capability spike.
- Minimum Mac target: Apple Silicon running macOS 14 or newer.
- Guest baseline: the latest stable AOSP release available at implementation kickoff, currently planned as Android 17/API 37, pinned to an exact release tag.
- Initial shipping ABIs: x86_64 guest on Windows x64; arm64 guest on Windows ARM64 and macOS ARM64.
- Each target offers distinct Desktop and TV instances. Edition selection occurs when creating an instance; converting existing userdata between editions is unsupported.
- Desktop app presentation defaults to `Auto`. `Phone` keeps compact-width resources even in a large host window; `Tablet` keeps at least medium-width resources; changing a locked form factor may recreate the task. TV remains a television product and does not silently inherit Desktop Phone/Tablet overrides.
- Every instance selects immutable `ServiceMode=NoApp|microG|GApps` and `RootMode=Off|KernelSU|Magisk` at creation. `NoApp` and `Off` are defaults; changing either dimension creates or clones to a new instance rather than reusing privileged/service state.
- `NoApp` contains no optional store/service bundle. `microG` contains verified F-Droid plus the approved microG set; Google device registration, Cloud Messaging, and network-location choices remain independent opt-ins. `GApps` contains either a compatible package explicitly imported by the user through the payload-free provider workflow or an exact signed provider artifact obtained through a separate licensed/certified partner program.
- Functional network operations are not first-party telemetry: user-requested app traffic, F-Droid synchronization, enabled microG functions, user-installed GApps, signed updates, and user-configured DNS/time/connectivity services are classified separately. First-party requests use no stable installation/device identifier, per-device query string, or tracking cookie.
- User-imported GApps, Google Play Services/Store, Widevine, Houdini, `libndk_translation`, proprietary codecs, root modules, and bypass payloads are not bundled or automatically downloaded. Imported packages are accepted only from files the user selects and remains responsible for using lawfully. A `CertifiedPartner` GApps/Widevine artifact is the sole exception and exists only inside its exact licensed access-controlled distribution channel.
- Root variants are Developer/Advanced SKUs. Enabling guest root lowers the guest-sandbox assurance and can break certification-sensitive applications, but never weakens VM/host broker isolation or first-party zero telemetry.
- Desktop clipboard sync is enabled only after onboarding disclosure and remains globally/per-instance controllable; notification forwarding requires both host authorization and Android notification-access consent. TV clipboard is off by default and TV notification previews are opt-in/allowlisted.

## 3. Product goals

### Experience goals

- Install Android APKs and launch each application from the host launcher as if it were a desktop app.
- Choose `NoApp`, `microG`, or user-installed `GApps` when creating an instance; use the host APK installer in every mode and F-Droid in `microG` mode.
- Optionally create a clearly marked KernelSU or Magisk Developer instance with default-deny root grants, module recovery, verified artifacts, and no concealment/bypass claim.
- Present each Android task in an independent native host window with resize, orientation, fullscreen, picture-in-picture, multi-monitor, keyboard, mouse, touch, stylus, and gamepad support.
- Make the host shell visually and behaviorally native: Windows 11 Fluent/WinUI controls and window conventions for Trinity; AppKit/SwiftUI menus, windows, sheets, commands, and Retina behavior for LeapDroid. Avoid custom-drawn imitations of host title bars or system controls.
- Expose `App settings` from the launcher, app context menu, and focused window. Users can choose Auto/Phone/Tablet, adaptive/fixed behavior, orientation, initial size, remembered placement, aspect/letterbox policy, display scale, fullscreen/PiP, input, and per-app integrations, then preview/relaunch/reset safely.
- Make host keyboard layouts and IMEs, low-latency cursor movement, relative pointer capture, controller hot-plug, multiple players, and haptics behave predictably in both Desktop and TV editions.
- Start the subsystem on demand, keep it warm under policy, suspend when idle, and resume rapidly.
- Synchronize clipboard text, sanitized HTML, and bounded images bidirectionally with loop prevention, source labeling, sensitive-content policy, multi-instance arbitration, and per-direction controls.
- Mirror Android notifications into Windows Notification Center and macOS Notification Center with lifecycle updates, grouping, progress, opening, safe actions, direct reply, per-app/channel policy, and dismissal reconciliation.
- Integrate selected host folders, drag and drop, audio, microphone, camera, location, links, and host-open-with behavior.
- Expose ADB only when developer mode is enabled and protect it with pairing and localhost-only defaults.
- Deliver signed guest and host updates with rollback.
- Provide a local diagnostics viewer and explicit, previewable export bundle without any automatic upload path.
- Provide a TV edition that starts as a single full-screen or windowed 10-foot experience, supports D-pad/gamepad/media-key navigation, host microphone/audio, 1080p and 4K display profiles, all gated service/root modes, and TV-aware app filtering.

### Engineering goals

- One versioned host/guest protocol shared across both products.
- A versioned `AppPresentationProfile` keyed by host user, instance, Android user, package, and optional activity; one policy engine resolves user choice, compatibility database, manifest capabilities, current window bounds, and host constraints into an auditable per-task configuration.
- Real native host chrome is authoritative for move/resize/minimize/maximize/fullscreen/close. Android WindowManager remains authoritative for activity/task lifecycle, layout, configuration dispatch, compatibility mode, and in-app surfaces.
- A shared system/vendor foundation with Desktop and TV product definitions per ABI; platform-specific HAL modules remain isolated under vendor/product partitions.
- Compatibility-first graphics through Gfxstream and ANGLE; Trinity projection remains a separately testable optimization.
- No guest-controlled parser in the privileged UI process.
- Reproducible builds, software bills of materials, signed artifacts, and automated CTS/VTS/deqp testing.
- A provider-neutral ARM-on-x64 compatibility layer that can be disabled, audited, upgraded, and tested independently from the base system image.
- Versioned `ServiceMode`, `GAppsProvider`, and add-on manifests that preserve a clean signed AOSP base, original APK signatures, exact privilege deltas, transactional import/update/rollback, and strict mode isolation.
- Versioned `RootMode` artifacts for clean `Off`, KernelSU, and Magisk variants with reproducible source builds, separate trust metadata, default-deny grants, module safe mode, and host-owned recovery.
- Defense in depth across the VM boundary, Android app sandbox, SELinux/seccomp, per-app network/sensor controls, and least-privilege host brokers without exceeding defined frame-time, input-latency, memory, CPU, or energy budgets.
- One input contract across Windows and macOS: standard Android key/axis semantics, deterministic focus and capture ownership, no duplicate event paths, and bounded queues at high polling rates.
- Versioned clipboard and notification models with monotonic revisions, idempotent reconciliation, opaque action capabilities, strict content limits, and local redacted diagnostics only.
- A CI-enforced zero-telemetry production profile whose startup, idle, crash, update, suspend/resume, and application-launch traffic has no undocumented first-party destination.

### Non-goals and certification boundary for the first general release

- Claiming Google Play/Play Protect/Google TV certification, distributing GApps/GMS/Play Store, or marking a provider `CertifiedPartner` without written approval for the exact artifact and target. A user import is never licensed/certified by the project.
- Play Integrity/SafetyNet/attestation spoofing or bypass, unauthorized Widevine/OEMCrypto/keybox/HDCP integration, root concealment, banking/DRM bypass, or anti-cheat circumvention. A conditional certified non-root SKU must use official partner certification, provisioning, and licensed DRM paths only.
- Direct USB, Bluetooth controller passthrough, NFC, telephony, or eSIM.
- Bundling, extracting, redistributing, or automatically downloading Houdini or `libndk_translation`; the project does not grant licenses for user-imported providers.
- Guaranteeing that microG, user-imported GApps, or even a certified SKU supports every Google API, licensed-media service, bank, or anti-cheat application. The certified `GApps + Off` target is gated by agreed verdicts/security levels and published blocking app matrices, while each service retains its own admission policy.
- Pixel-perfect compatibility with every phone sensor or OEM extension.
- Google TV certification, Play Store for TV, licensed Google TV launcher/content services, Widevine L1, HDCP, protected-video paths, or guaranteed commercial streaming-service compatibility.
- HDMI-CEC passthrough, tuner hardware, TV Input Framework sources, Dolby/DTS bitstream passthrough, or HDR certification in the first TV release.
- Host-global key capture, macro/rapid-fire tooling, host-side controller emulation drivers, anti-cheat bypasses, or direct USB/Bluetooth passthrough. Host APIs own physical-device pairing; the guest sees virtual devices only.
- Reading clipboard history, managing Windows Cloud Clipboard or Apple Universal Clipboard, silently exporting Android clips marked sensitive, mirroring arbitrary guest file/content URIs, or syncing clipboard clears that would erase unrelated host content.
- Accessing notifications from unrelated host applications, impersonating each Android package as a separately installed native host app, executing stale notification actions, or promising exact rendering of custom Android `RemoteViews`.
- Replacing an Android app's internal widgets, dialogs, web content, or application navigation with WinUI/AppKit controls. Native fidelity applies to the host shell and integrations, not arbitrary third-party UI rendered by Android.
- Promising a separate macOS Dock/Launchpad application identity for every Android package before the signed launcher-shim, Gatekeeper, notarization, App Sandbox, update, and uninstall spike passes. The native-window and Spotlight fallback remains a supported product path.
- Claiming that Trinity/LeapDroid can prevent telemetry inside every user-installed Android app or disable telemetry implemented by Windows/macOS themselves. The product can expose per-app network controls, a default-deny policy template, and auditable first-party behavior; the host OS and third-party apps remain separate trust domains.

## 4. Target matrix

| Product | Edition | Host | Guest ABI | CPU accelerator | Host graphics | Software fallback |
|---|---|---|---|---|---|---|
| Trinity | Desktop | Windows 11 x64 | x86_64 | WHPX | ANGLE D3D11 baseline; D3D12/Vulkan evaluated | SwiftShader |
| Trinity | TV | Windows 11 x64 | x86_64 | WHPX | ANGLE D3D11 baseline; D3D12/Vulkan evaluated | SwiftShader |
| Trinity | Desktop | Windows 11 ARM64 | arm64 | WHPX Arm64, capability-gated | ANGLE D3D11/D3D12 as supported | SwiftShader |
| Trinity | TV | Windows 11 ARM64 | arm64 | WHPX Arm64, capability-gated | ANGLE D3D11/D3D12 as supported | SwiftShader |
| LeapDroid | Desktop | macOS 14+ ARM64 | arm64 | QEMU HVF | ANGLE Metal | SwiftShader |
| LeapDroid | TV | macOS 14+ ARM64 | arm64 | QEMU HVF | ANGLE Metal | SwiftShader |

The Windows ARM64 target has a hard Phase 0 gate: boot, interrupt delivery, timers, virtio devices, suspend/resume, and graphics transport must work under WHPX without TCG on the critical path.

Windows x64 has an optional ARM application-compatibility matrix: native x86_64 only by default, or ARM translation through one selected user-supplied provider. This does not change the x86_64 guest architecture and is not used on native arm64 guests.

## 5. Recommended architecture

### 5.1 Repository and licensing boundaries

Create a shared runtime repository with explicit license boundaries:

- `runtime/qemu`: pinned current QEMU plus small, reviewable patches; GPLv2 obligations apply here.
- `runtime/devices`: shared virtio control, surface, input, and bulk-data devices.
- `protocol/input`: text-composition, cursor-state, capture, device metadata, battery, and haptics messages that do not belong in the high-rate virtio event path.
- `protocol/presentation`: task/window lifecycle, applied Android configuration, native-window state, app identity/activation, form-factor profile, compatibility mode, relaunch acknowledgement, and versioned failure reasons.
- `protocol/clipboard` and `protocol/notifications`: normalized data models, revisions, acknowledgements, capabilities, limits, redaction, and compatibility rules; no native handles, host paths, `PendingIntent`, or unbounded blobs cross these contracts.
- `runtime/graphics`: Gfxstream integration, ANGLE adapter, SwiftShader fallback, and optional Trinity projection plugin.
- `guest/aosp`: manifests, product definitions, kernel config, SELinux policy, HALs, system extensions, and guest agents.
- `guest/products/common`, `guest/products/desktop`, and `guest/products/tv`: shared product base plus edition overlays, launchers, feature declarations, input policy, and edition-specific tests.
- `guest/packages`: pinned microG and F-Droid integration manifests, signing-certificate allowlists, notices, and update policy.
- `guest/service-modes`: `NoApp`/`microG`/`GApps` schemas, payload-free GApps provider descriptors, add-on builder/validator, synthetic fixtures, privilege deltas, and compatibility tests; no proprietary Google payload enters source or CI.
- `guest/root`: clean-absence tests plus pinned KernelSU/Magisk source manifests, reproducible boot/kernel variants, manager policy, generic local-module import, recovery, update/rollback, and root compatibility tests; no bundled bypass/concealment payloads or module catalog.
- `guest/nativebridge`: AOSP-facing provider interface, Houdini and `libndk_translation` manifests/adapters, validator, activation logic, and compatibility tests without vendor binaries.
- `upstream-audits/wsa`, `upstream-audits/waydroid`, `upstream-audits/lineageos`, and `upstream-audits/grapheneos`: pinned source revisions, per-change provenance, license disposition, applicability, security/performance evidence, and `import`/`reimplement`/`drop` decisions.
- `protocol`: versioned schemas and compatibility rules; generated bindings for C++, Rust, Kotlin/Java, and Swift where needed.
- `host/common`: lifecycle state machine, app registry model, image/update verification, local diagnostics interfaces, egress policy, and compatibility policy.
- `host/common/presentation`: `AppPresentationProfile`, Auto/Phone/Tablet resolver, task/window ownership, placement restoration, display/monitor transforms, profile migrations, native command model, and conformance fixtures.
- `host/common/input`: canonical key, pointer, and controller descriptors; focus/capture arbitration; mappings; state reset; latency tracing; and conformance fixtures.
- `host/common/clipboard` and `host/common/notifications`: loop prevention, active-instance arbitration, MIME/action normalization, lifecycle reconciliation, privacy policy, rate limiting, and host-neutral tests.
- `host/windows`: Trinity service, WinUI shell, Windows integration adapters, packaging.
- `host/macos`: LeapDroid daemon, SwiftUI/AppKit shell, macOS integration adapters, packaging.
- `tests`: protocol, integration, compatibility, performance, security, and long-run suites.

Do not copy Trinity GPL code directly into the Apache-licensed LeapGL library. Keep GPL QEMU derivatives in a separately distributed process/component and maintain notices and source-offer obligations. Perform a formal license review for QEMU, ANGLE, Gfxstream, SwiftShader, AOSP, codecs, and all guest packages before public beta.

### 5.2 WSA source-use policy

- Treat `microsoft/WSA` as documentation and a feature inventory, not a source dependency; it contains no reusable implementation of the shipping runtime.
- Audit `microsoft/WSA-Linux-Kernel` by diffing its WSA configs and Microsoft patches against the selected upstream Android common/GKI kernel. Import only commits that solve a demonstrated Trinity requirement and remain compatible with the current kernel.
- Preserve original copyright/license notices, commit provenance, SPDX data, and corresponding-source obligations for every imported kernel change.
- Do not extract or depend on WSA MSIX contents, DLLs, services, guest images, graphics binaries, signing material, or undocumented protocols. Reimplement Windows integration from public Windows APIs and the project's own versioned protocol.
- Maintain a clean-room evidence log for each WSA-inspired feature: public reference, externally observed behavior, independent design, tests, and implementation owner.

Preliminary WSA kernel disposition:

| WSA kernel area | Decision for Trinity | Reason |
|---|---|---|
| x86_64/arm64 Android kernel configurations | Use as an audit checklist, then regenerate on current GKI | Useful evidence for Binder/BinderFS, cgroups, memory pressure, power, filesystems, and minimized virtual hardware; the published configs target older 5.10/5.15 lines |
| Upstreamable security, scheduler, Binder, and virtual-device fixes | Rebase selectively after per-commit diff/test | Potentially useful only when the fix is absent from the selected current kernel |
| Hyper-V timers/balloon/vsock configuration | Reference only | Trinity's cross-platform machine model uses QEMU virtio and a shared control protocol; direct reuse would create a Windows-only guest fork |
| `DXGKRNL`/WSA DirectX guest path | Drop from the shipping design | The corresponding closed WSA host protocol/runtime is unavailable, while the selected graphics path is Gfxstream + ANGLE |
| `ASHMEM` and other legacy Android compatibility options | Drop unless a measured legacy-app gate requires them | Current Android uses modern memory mechanisms; carrying obsolete kernel interfaces enlarges attack and maintenance surface |
| Whole WSA 5.10/5.15 kernel fork | Do not adopt | It would replace a current GKI baseline with an older, product-specific fork and make annual Android/security updates harder |

### 5.3 Waydroid and LineageOS source-use policy

Keep AOSP as the canonical guest baseline. Neither Waydroid nor LineageOS becomes an unbounded upstream merge dependency.

| Source area | Decision | Intended use |
|---|---|---|
| Waydroid LXC/namespaces host manager | Drop for Windows/macOS | It relies on a shared Linux kernel, Binder devices, Linux namespaces, and Wayland sockets; Trinity/LeapDroid require VM isolation and native host presentation |
| Waydroid `android_hardware_waydroid` | Audit and selectively port | Reference audio, gralloc/HWC, power, health, sensors, gatekeeper, memtrack, and host-interface designs; adapt only pieces that fit virtio/Gfxstream/AIDL |
| Waydroid vendor product, init, SELinux, host AIDL/HAL manifests | Audit and selectively port | Reference guest bootstrap, system/vendor image separation, host service discovery, session readiness, property policy, and OTA layout |
| Waydroid desktop entries, app launch, full-UI/multi-window behavior | Reimplement against native host APIs | Use behavior and tests as UX input; do not import Linux desktop/Wayland integration into Windows or macOS shells |
| LineageOS full platform manifest | Do not adopt wholesale | Its broad framework/app patchset would enlarge merge and CTS risk; AOSP remains easier to update and qualify |
| LineageOS restricted microG signature spoofing | Strong candidate for rebased import | It matches the requirement to permit only approved microG identities; preserve provenance and add stricter certificate tests |
| LineageOS ATV device tree, overlays, key layouts, `TvSystemUI` | Audit and selectively port | Reuse maintained TV-specific behavior where it improves on AOSP ATV; explicitly exclude its GMS RROs, vendor/Netflix configuration, and hardware-specific files unless separately justified |
| LineageOS updater/recovery and physical-device HALs | Reference or drop | The subsystem uses signed partition updates and virtual HALs; import only generic migration, signing, or test patterns that solve a demonstrated gap |

Every imported change must be reduced to the smallest current-AOSP patch, carry its original license/provenance, have an upstream replacement check, and include a removal condition. GPL Waydroid host code must not be copied into closed native UI/services without an approved license boundary.

### 5.4 GrapheneOS source-use, zero-telemetry, and sandbox policy

Keep current AOSP as the canonical guest and treat GrapheneOS as a bounded hardening audit, not a new product base or a claim of equivalent security.

| GrapheneOS area | Decision | Trinity/LeapDroid use |
|---|---|---|
| Hardened SELinux and seccomp policy | Audit and selectively rebase | Tighten guest services, brokers, media, graphics, and imported-code domains while preserving CTS/VTS and current AOSP interfaces |
| Network and Sensors permission controls | Reimplement for this guest | Add per-app controls with fail-closed enforcement and compatibility UX; provide optional default-deny policy templates without breaking normal AOSP permission semantics |
| `hardened_malloc`, zero-on-free, memory-safety hardening | Target high-risk processes first | Enable for parsers, media, graphics, integration agents, and Native Bridge inspection after per-process performance tests; expand only when budgets pass |
| Secure app spawning and dynamic-code/JIT restrictions | Audit and selectively rebase | Forbid dynamic code loading in first-party base services; use explicit per-app compatibility exceptions for user apps after measurement |
| Local log viewer/manual crash reporting | Reimplement | Bounded memory/ring-buffer logs, redaction preview, and manual export only; no automated crash or usage upload exists |
| Storage/Contact Scopes | Evaluate for Beta or later | Adopt only if current-AOSP integration, CTS behavior, maintenance cost, and UX are acceptable |
| Sandboxed Google Play | Do not import | The project's user-supplied GApps add-on is a separate local provider design; do not copy GrapheneOS implementation/services or claim its containment/security properties |
| Pixel firmware, radio, USB-C, hardware attestation, telephony | Drop | Hardware-specific and irrelevant to a virtual Android subsystem |
| Whole GrapheneOS manifest, infrastructure, branding | Do not adopt | Avoid a broad platform fork, external service dependency, trademark confusion, and unsupported security equivalence claims |

Production builds use a mandatory zero-outbound-telemetry profile. They contain no analytics SDK, crash uploader, remote logger, product experimentation service, stable install/device tracking ID, or metrics collector. Local health counters, traces, logs, and crash artifacts are bounded, privacy-redacted, stored locally, visible to the user, and exported only after an explicit action; release CI rejects any automatic upload path.

Maintain a machine-readable first-party egress manifest. It distinguishes functional traffic—signed static updates, F-Droid, enabled microG, user-installed GApps, rooted apps/modules, user apps, and user-configured connectivity/DNS/time/location—from telemetry and from each other. Product update requests carry no stable installation/device identifier, tracking cookie, or per-device query parameter. A host-provided clock or user-configured service is preferred over hidden guest probes; enterprise/offline policy can require `NoApp` + `Off`.

Defense in depth spans the VM, guest, and host. The guest keeps per-UID sandboxing, SELinux enforcing, seccomp, namespaces, Binder permission checks, scoped storage, user/profile separation, AVB/dm-verity, resource limits, and per-app Network/Sensors toggles. Host UI, VM controller, renderer, media codecs, clipboard, notifications, file/camera/microphone/location brokers, updater, diagnostics, and Native Bridge inspection run as separate least-privilege processes with capability-based IPC, quotas, restart boundaries, and no shared ambient authority.

On Windows, use restricted tokens and AppContainer/LPAC where compatible, Job Objects, process mitigation policies such as DEP/ASLR/CFG/CET and capability-gated dynamic-code/image-load restrictions, plus per-process firewall rules. On macOS, use App Sandbox where compatible, Hardened Runtime, least entitlements, security-scoped bookmarks, and isolated XPC services; Phase 0 must prove how these constraints coexist with HVF/hypervisor, JIT/dynamic-code needs, packaging, and notarization.

Sandboxing is a containment strategy, not a performance claim. Use bounded queues/backpressure, lazy helper startup, immutable shared data, validated zero-copy bulk paths, broker recycling, Android cgroups/LMKD, Windows Job Objects, and macOS pressure notifications. Measure every hardening change; optimize first, then scope an expensive mitigation to high-risk processes or a documented per-app compatibility exception rather than silently disabling protection globally.

Detailed implementation and acceptance gates are in `phase-6a-zero-telemetry-sandbox.md`.

### 5.5 Virtual machine core

- Rebase from QEMU 5.0 to the latest supported stable QEMU at kickoff, expected to be QEMU 11.x.
- Keep the upstream accelerator, machine, virtio, migration, block, networking, and security models wherever possible.
- Replace HAXM with WHPX on Windows; retain TCG only for diagnostics and CI smoke tests.
- Use QEMU HVF on macOS because custom devices and the shared runtime are central requirements. Evaluate Virtualization.framework only as a contingency or for isolated host integrations; do not split the product across two VM cores without a failed HVF gate.
- Define a minimal, versioned virtual hardware profile shared by x86_64 and arm64: virtio-blk, virtio-net, virtio-snd, virtio-rng, balloon, serial/control channel, shared-memory transport, input, and graphics.

### 5.6 Android guest

- Build AOSP Android 17/API 37 from a pinned manifest with `subsystem_desktop_x86_64`, `subsystem_desktop_arm64`, `subsystem_tv_x86_64`, and `subsystem_tv_arm64` products.
- Borrow current Cuttlefish product patterns for virtio devices, Gfxstream, host services, testing, and update separation rather than carrying Android-x86 9 forward.
- Place host-integration changes in `system_ext` and `product`; minimize changes to `system` and framework code.
- Use GKI-compatible kernels and maintain one small patch series per architecture.
- Enforce SELinux, AVB, read-only system partitions, A/B-style image updates, per-user userdata, and deterministic image builds.
- Implement PackageManager, ActivityTaskManager, WindowManager, notification, clipboard, file, input, camera, sensor, and power agents behind stable AIDL interfaces.
- Share immutable system/vendor partitions wherever Android compatibility permits, but give Desktop and TV separate product partitions, update manifests, instance identities, and userdata disks.

### 5.7 Service modes, application services, and developer root

- Every new instance selects exactly one immutable `ServiceMode`: `NoApp`, `microG`, or `GApps`. `NoApp` is the clean AOSP default and contains no store or compatibility-service bundle; all modes retain the host APK installer and optional developer ADB.
- `microG` provisions a verified official F-Droid Client plus the approved microG packages. Keep both in ordinary app sandboxes, restrict signature spoofing by package name and pinned certificate, and expose separate opt-ins for device registration, Cloud Messaging, network location, and network access.
- `GApps` is an advanced, uncertified local add-on. The user selects a lawfully obtained package; a sandboxed importer validates its descriptor, Android/API/ABI/edition compatibility, contents, signatures and hashes, privilege delta, and space requirements before constructing a sealed, versioned, read-only add-on. The project never bundles, mirrors, searches for, downloads, or extracts proprietary Google packages.
- A separately licensed `GAppsProviderClass=CertifiedPartner` may target `RootMode=Off` only. It requires exact written distribution/Play Protect approval, protected partner-provisioned identity and attestation, official server-verified Play Integrity results, applicable SafetyNet legacy tests, a licensed Widevine security level/secure media path, and the regional banking blocking matrix. It is never derived from or interchangeable with user import.
- Never mix microG and GApps in one instance. Changing `ServiceMode` creates or clones to a new instance and migrates only user-selected ordinary app/data exports—not system packages, accounts, credentials, tokens, or privileged state.
- Preserve the signed AOSP base, AVB/dm-verity, enforcing SELinux, rollback, and a host-owned safe-boot path in every mode. A GApps import may extend only reviewed allowlisted partitions/permissions; it may not make system writable or silently change the base image.
- Treat `GAppsProviderClass=UserImport` as best-effort and clearly label it uncertified; do not claim Google Play/Play Protect/Google TV certification, Play Integrity, Widevine/DRM, banking, anti-cheat, or complete API compatibility for it. `CertifiedPartner` claims only the exact approved/tested target, verdicts, DRM tier, and blocking-app matrix.
- `RootMode` is a separate immutable choice: `Off` (default), `KernelSU`, or `Magisk`. Rooted variants are Advanced/Developer artifacts built reproducibly from pinned upstream source, with default-deny per-app grants, no host capability, safe mode/recovery, and no concealment or attestation/DRM/banking/anti-cheat bypass support.
- KernelSU is target-gated: do not ship it on Windows x64 if the pinned upstream release requires disabling syscall hardening. Magisk uses a separately signed boot/init variant, keeps Zygisk off by default, and never requires disabling verified boot or SELinux. KernelSU and Magisk never coexist in one instance.
- Phase 2B defines service-mode/GApps implementation and Phase 2C defines KernelSU/Magisk integration, recovery, verification, and release gates.

### 5.8 Windows x64 ARM Native Bridge

- Implement `NativeBridgeProvider` around AOSP `libnativebridge`, with exactly three user-visible modes for the first release: `Off`, `Houdini`, and `libndk_translation`.
- Ship open provider descriptors, validation code, lifecycle hooks, and tests only. The user imports a legally obtained provider package; Trinity never searches for, downloads, extracts from another installed product, or redistributes provider binaries.
- Validate package manifest, expected ELF class/machine, supported ARM ABIs, Android/API compatibility, required linker/runtime files, version, cryptographic hash, and optional vendor signature before installation. Reject unknown or incomplete layouts.
- Store an accepted provider in a versioned, host-protected private directory; expose it read-only to the guest and activate it through supported product properties/linker namespaces. Never mix provider files into the immutable base image.
- Detect APK ABI before install and report `native`, `translatable`, or `unsupported`. Do not advertise ARM ABIs until the selected provider passes boot/self-tests for that ABI.
- Provider switching or upgrade requires the guest to stop, clears provider-specific ART/native-code caches, runs a self-test, and automatically rolls back to `Off` if boot or health checks fail.
- Sandbox the import/inspection worker, scan payloads with host security tools, show the source/license responsibility to the user, and keep proprietary file hashes, contents, and inspection results local.

### 5.9 Android TV editions

- Derive TV products from current AOSP `device/google/atv`, then evaluate LineageOS `android_device_lineage_atv`, `TvSystemUI`, overlays, key layouts, settings, and input fixes as a bounded patch queue.
- Set `PRODUCT_CHARACTERISTICS=tv`, declare television/Leanback features accurately, include TV provisioning, TvSystemUI/TvSettings-equivalent components, Leanback IME, and a fully open-source 10-foot launcher. Do not include Google TV components.
- Treat TV as a separate instance type, not a runtime theme. Present one Android display in a native full-screen/windowed canvas; disable Desktop per-task native-window export in TV mode.
- Map host D-pad, gamepad, keyboard, media keys, mouse fallback, and supported remotes to Android key/layout semantics. Provide explicit Back, Home, voice/microphone push-to-talk, and input-release shortcuts.
- Support 720p, 1080p, 1440p, and 2160p profiles with safe-area, density, aspect-ratio, refresh-rate, overscan, and UI-scaling policy. Start with SDR; enable HDR only after end-to-end color, metadata, tone-mapping, and conformance gates.
- Add a host-backed MediaCodec path: Media Foundation/D3D11 video on Windows and VideoToolbox/Metal on macOS, with software decode fallback. Protected buffers, HDCP, Widevine L1, and commercial DRM certification remain unsupported in open/user-import builds; a `CertifiedPartner + Off` SKU may enable only the separately licensed and proven secure path/tier.
- Route stereo and multichannel PCM through WASAPI/CoreAudio. Treat compressed passthrough and licensed codecs as later, separately licensed features.
- Filter TV discovery to Leanback-capable apps by default, expose incompatible sideloaded apps behind an explicit setting, and define portrait/touch-only fallback behavior.

### 5.10 Graphics

Use two graphics tiers:

1. Compatibility backend: Gfxstream guest/host protocol with ANGLE on the host. This provides a maintained GLES/Vulkan path and a shared implementation across Windows and macOS.
2. Trinity projection backend: port the projection-space idea and `direct-express` data path only after the compatibility backend passes conformance and real-app tests. It is opt-in until it matches compatibility and isolation gates.

Windows host rendering uses ANGLE over D3D11 as the baseline because of driver breadth, with D3D12 or Vulkan enabled only after per-adapter validation. macOS rendering uses ANGLE Metal. SwiftShader is the deterministic fallback for unsupported adapters, CI, and recovery mode.

The renderer runs in a sandboxed process separate from the native UI and VM controller. Guest-provided command streams are validated, length-bounded, fuzzed, and versioned.

### 5.11 WSA-like seamless windows

- Add a guest task/surface broker that observes Android task lifecycle and exports a stable task ID, package/activity identity, icon, title, bounds, orientation, z-order, and surface handle.
- Map each Android top-level task to one native host window: `AppWindow`/HWND on Windows and `NSWindow` with `CAMetalLayer` on macOS.
- Keep SurfaceFlinger and WindowManager authoritative for Android composition; the host owns desktop placement, native chrome, taskbar/Dock/Spotlight integration, host shortcuts, and native close/minimize/maximize/fullscreen behavior. Never draw an Android or cross-platform imitation of a Windows/macOS title bar around the surface.
- Forward resize and DPI changes to Android configuration and display areas; avoid scaling a fixed framebuffer.
- Route input by window/task ID and translate host IME composition to Android input-method events.
- Support freeform windows first, followed by picture-in-picture, multi-monitor, drag and drop, and accessibility semantics.
- Windows uses a real AppWindow/HWND per task with standard caption/system menu, Snap Layouts, Alt+Tab/taskbar presence, per-app icon/title, per-window AppUserModelID grouping, Start/Search shortcut, protocol/file activation where appropriate, native DPI, virtual-desktop behavior, and placement restoration.
- macOS uses a real NSWindow per task with traffic-light controls, spaces/fullscreen/minimize, Retina backing scale, Window menu, focused-app command routing, standard Edit/Services items, drag/drop, file/URL activation, accessibility, and Spotlight/`NSUserActivity` launch entries. Separate per-package Dock/Launchpad identity is capability-gated on the signed launcher-shim Phase 0 result.
- Use native control surfaces for settings, permission explanations, file/folder pickers, context menus, sheets/dialogs, update/recovery, diagnostics, and host integrations. Android app content—including its own toolbar, dialog, and navigation—remains an Android surface for compatibility.
- Define close semantics explicitly: native window close finishes/removes that Android task but does not shut down the VM; minimize preserves the task; `Quit/Force stop Android app` is a separate visible command; closing the last window follows the configured subsystem idle policy.
- Add an `App settings` entry to launcher/context menu/title-bar or application menu. Persist a versioned profile by host user + instance + Android user + package, with activity-level overrides deferred unless a demonstrated app requires one.

**Per-app form-factor profiles**

- `Auto` is the recommended default. Resolve the manifest's resizability/orientation/minimum/aspect declarations, Android compatibility data, current client bounds, density, input devices, and last successful state. Responsive apps receive live per-window configuration and can move through compact, medium, and expanded widths.
- `Phone` locks the application to compact-width Android resources (`smallestScreenWidthDp < 600`) with a portrait or landscape preset. A larger host window scales or letterboxes the compact canvas rather than making the app identify as a tablet.
- `Tablet` keeps the app at medium or expanded width (`smallestScreenWidthDp >= 600`, with 840dp as the expanded breakpoint). If the host window cannot provide the minimum native client area, use bounded size-compatibility scaling/letterboxing rather than sending an inconsistent configuration.
- Advanced per-app controls include adaptive versus locked form factor, App/Portrait/Landscape orientation, initial size, remember size/monitor, app aspect versus bounded presets, density/display scale, fullscreen/PiP, compatibility renderer, and reset to defaults. Keep the primary UI to Auto/Phone/Tablet; place unstable overrides under Advanced.
- Do not spoof build fingerprint, device model, Play certification, hardware features, or a global guest configuration to achieve Phone/Tablet mode. The app detects the intended layout through a truthful per-task/display Android `Configuration`.
- Compare a shared display with per-task configuration overrides against one virtual display/`TaskDisplayArea` per exported task. Select the model that produces correct resources/lifecycle with the lowest SurfaceFlinger/Gfxstream memory and latency; never allow one app's profile to reconfigure another task.
- Apply profile changes transactionally. Preview the resulting dp size/orientation, warn if a task restart is required, snapshot current placement, recreate only the affected task/display, confirm the applied configuration, and roll back on crash loop or surface timeout.
- Respect non-resizable and orientation-constrained applications through Android size-compatibility/letterbox behavior. A force-resize exception is explicit, per app, compatibility-database versioned, and resettable—not a global framework override.
- This per-app window model applies to Desktop editions. TV editions use the single-display 10-foot presentation defined in Section 5.9.

Detailed native-shell, profile schema, host behavior, validation, and macOS identity gates are in `phase-4b-native-app-experience-display-profiles.md`.

### 5.12 Unified keyboard, mouse, cursor, and controller input

Use one end-to-end input stack for all six SKUs. A focused native window captures physical input, the common host layer normalizes it, and one virtual device per logical keyboard, pointing device, or controller publishes Linux evdev events through `virtio-input`. A separate authenticated control channel carries IME composition, cursor metadata, capture state, battery, and haptics. Every event carries a monotonic host timestamp, device ID, instance ID, and destination display/task context.

**Keyboard and text**

- Keep two explicit, mutually exclusive paths per interaction. The physical-key path translates scan codes/HID usages, left/right modifiers, locks, media keys, press/release, and repeat into Android key events for games and shortcuts. The text path sends IME pre-edit, selection, replacement range, committed Unicode, and cancellation to a trusted guest IME/InputConnection bridge.
- Windows uses foreground-only Raw Input for physical keys and a native TSF/CoreText-compatible text client for composition. macOS uses the focused `NSResponder`/`NSEvent` chain and `NSTextInputClient`. Do not use global event monitors, accessibility privileges, or background keyboard registration for ordinary input.
- Define an ordered shortcut policy: host-reserved combinations remain on the host; product commands such as release-capture, Home, and Back are intercepted visibly; all remaining keys go to Android. Never forward the Windows secure-attention sequence or protected host shortcuts.
- Use one repeat owner, release all depressed keys on focus loss/suspend/disconnect, and suppress physical character generation while a host composition transaction is active. Test dead keys, AltGr, Vietnamese, CJK, emoji, US/ISO/JIS/AZERTY layouts, and lock-state transitions.

**Mouse, pointer capture, and cursor**

- Provide an absolute pointer/tablet device for normal Desktop and TV interaction and a relative mouse device for Android pointer capture. Map through the exact content rectangle, scale, density, crop, rotation, safe area, and monitor DPI; preserve vertical/horizontal wheels and buttons 1–5.
- Windows uses Raw Input and buffered reads for high-rate relative movement. macOS uses window-scoped AppKit events and a platform pointer-lock adapter. Capture begins only after explicit interaction in the focused window, hides/confines or detaches the host pointer as the platform permits, and always exposes a host-handled release shortcut and recovery overlay.
- Make the host cursor authoritative for latency. A guest cursor agent exports Android pointer-icon type, visibility, custom bitmap, hotspot, and focused surface; the host maps standard icons to native cursors and caches bounded custom cursors. Suppress the guest-rendered cursor to prevent a double cursor. During relative capture the host cursor is hidden and only deltas are delivered.
- Batch/coalesce motion at 1,000–8,000 Hz without crossing button, wheel, focus, capture, or timestamp-order boundaries. Cursor motion must not wait for an Android-rendered frame. On TV, hide the cursor after inactivity and restore it on mouse movement without stealing D-pad focus unexpectedly.

**Game controllers and remotes**

- Windows uses GameInput as the primary controller backend; an XInput adapter is legacy fallback only. macOS uses GameController. Keyboard and mouse remain on their native window-scoped paths so the same device is never read by two backends.
- Normalize each device into a versioned descriptor with stable connection ID/player slot, vendor/product identity when available, supported buttons/axes/hats, dead zones, battery, connection type, haptic channels, and optional capabilities. Support at least eight simultaneous controllers, hot-plug, reconnect, wired/Bluetooth devices managed by the host, and deterministic player reassignment.
- Map standardized controls to Android `KeyEvent` and `MotionEvent` conventions, including D-pad, face/shoulder/system buttons, sticks, triggers, and hats. Preserve raw-capability data only behind a compatibility-gated extension; do not expose host device handles to the guest.
- Route simple guest vibration/haptics requests back to GameInput/GameController with duration, amplitude, rate, quota, focus, and lifecycle limits. Treat LEDs, adaptive triggers, motion sensors, controller touchpads, wheels, flight sticks, and vendor-specific force feedback as capability-gated later work.
- In TV mode, exactly one device owns SystemUI navigation at a time while all connected controllers remain visible to the foreground game. Home/Back/recovery commands remain available even if an app consumes controller input.

**Focus, security, and failure behavior**

- Only the focused, visible instance receives keyboard/game input; pointer capture has one owner per host session. Focus transfer atomically synthesizes releases before the next destination receives input.
- Bound device count, queue depth, custom cursor dimensions, composition length, haptic duration/rate, and mapping-file size. Validate all guest-provided cursor and haptic messages in an unprivileged broker.
- Never log text, key content, or raw controller streams. Aggregate latency, device class, error code, and dropped/coalesced-event counts exist only in bounded local diagnostics and explicit performance-test artifacts; they are never uploaded automatically.
- On guest reset, renderer crash, host sleep, controller removal, or protocol timeout, release capture, restore the native cursor, stop haptics, clear all held inputs, and keep the host recovery shortcut functional.

Detailed implementation and acceptance gates are in `phase-4a-input-cursor-controller.md`.

### 5.13 Host/guest integration

- Control plane: versioned RPC over virtio-serial as the universal baseline; add vsock where reliable.
- Bulk plane: shared-memory ring buffers with explicit ownership, bounds, quotas, fences, and cancellation.
- Files: host broker plus Android DocumentsProvider for user-selected folders; add virtiofs only after ACL, path, symlink, and file-lock semantics are proven on both hosts.
- Network: NAT by default, explicit local-LAN toggle, per-instance firewall, host/guest localhost forwarding, DNS proxy, and IPv6 tests.
- Audio: virtio-snd or equivalent guest HAL mapped to WASAPI and CoreAudio.
- Camera/microphone/location: host permission broker mapped to virtual Android HALs; host denial must be visible to Android.
- Clipboard and notifications use the dedicated brokers in Section 5.14; neither payload is parsed in the VM controller or privileged UI process.
- Links: verified mapping between Android intents and Windows/macOS URL and file associations.
- TV mode: full-screen/windowed display control, D-pad/gamepad/media-key mapping, push-to-talk microphone, display profile selection, host sleep inhibition during playback, and remote-friendly recovery UI.

### 5.14 Clipboard and notification synchronization

**Clipboard**

- Define a bounded `ClipEnvelope` containing instance/user, origin, monotonically increasing revision, host timestamp, content digest, sensitivity/remote flags, and ordered MIME representations. Developer Preview supports plain Unicode text; Beta adds sanitized HTML and bounded PNG images. File lists and large content require an explicit file-broker/drag-and-drop flow later.
- Android uses a platform-signed, signature-permission and SELinux-confined clipboard agent because public background clipboard reads require input focus/default-IME status. The implementation must not grant additional clipboard access to normal apps. Mark host-origin clips with `ClipDescription.EXTRA_IS_REMOTE_DEVICE`; refuse automatic export of `EXTRA_IS_SENSITIVE` clips by default.
- Windows listens with `AddClipboardFormatListener` and reads/writes only allowlisted formats after `WM_CLIPBOARDUPDATE`, with delayed-rendering/retry handling outside the UI thread. macOS tracks `NSPasteboard.general.changeCount`, adapts observation frequency only while sync is enabled/running, checks `accessBehavior`, and decodes content only after a new change.
- Prevent ping-pong with origin/revision acknowledgements plus content digests; Android classification may emit multiple callbacks for one clip. Last accepted writer wins, but revisions remain scoped to one host user and instance. Host-origin content echoed by Android is acknowledged, not republished.
- By default, host-to-guest targets only the active instance; an unfocused instance never overwrites the host clipboard. Guest-to-host can be separately disabled per instance/app policy. Clipboard changes do not auto-start a stopped VM unless the user enables that behavior.
- Clipboard clear is local by default. It crosses the boundary only when it clears the same still-current bridged revision, preventing one side from deleting newer unrelated content.
- Sanitize HTML to inert markup, decode images in a sandbox, reject active content and arbitrary file/content URI dereference, enforce item/byte/dimension/time limits, and zero transient buffers. Never store clipboard history or place clipboard contents/digests in logs or diagnostic exports.

**Notifications**

- A user-consented guest `NotificationListenerService` emits a normalized model keyed by instance, Android user, notification key, package, channel, ID/tag/group, and revision. Carry bounded title/text/subtext, timestamp, app label/icon, importance/silent state, progress, grouping, clearable/ongoing/local-only/visibility flags, and supported action descriptors.
- Do not bridge `FLAG_LOCAL_ONLY`. For `VISIBILITY_SECRET`, publish no host notification; for private content, use Android `publicVersion` or a redacted host body according to the user's preview policy. Custom `RemoteViews` degrade to normalized text/icon/progress rather than being rendered or executed on the host.
- The guest retains content/delete/action `PendingIntent` objects. The host receives short-lived opaque capabilities bound to instance, user, notification key, action index, revision, authentication requirement, and expiry. Before open/action/reply, the guest verifies the notification is still active and unchanged; stale/replayed tokens fail closed.
- Map supported Android actions to Windows App SDK buttons/text inputs and macOS UserNotifications categories/`UNTextInputNotificationAction`. Direct reply returns bounded Unicode through Android `RemoteInput`. Authentication-required actions resume/open Android and complete only after guest unlock; the host must not substitute host login as Android authentication.
- Windows uses stable tag/group/ID and AppNotificationManager enumeration/removal to reconcile only Trinity's notifications. macOS uses stable request identifiers, `.customDismissAction`, delivered-notification enumeration, replacement, and removal. Do not request Windows access to all user notifications.
- Android update/removal immediately replaces/removes the host copy. Host open/action/direct reply synchronizes to the guest. Explicit host dismiss cancels only a clearable Android notification; ongoing/non-clearable notifications are hidden locally or opened. Passive host dismissal is detected by own-notification reconciliation and deduplicated against guest-origin removal.
- Respect host notification permission, Focus/Do Not Disturb, sound policy, and per-app/channel toggles without modifying Android channel importance or guest DND. Rate-limit abusive packages, aggregate floods, and keep active state recoverable after host/guest restart.
- Forwarded notifications remain branded as Trinity or LeapDroid at the OS level; show the Android app label/icon inside supported content. Do not imply each Android package is an independently installed native host application.
- Clicking a host notification may cold-start/resume its instance, then revalidate the current guest notification before invoking it. Persist only bounded redacted display state and opaque identifiers; never persist live action authority across host or guest reboot.

Detailed work, platform differences, validation, and release gates are in `phase-5a-clipboard-notifications.md`.

### 5.15 Lifecycle, storage, and updates

- Implement a single lifecycle state machine: absent, provisioning, stopped, starting, running, idle, suspending, suspended, updating, recovery, failed.
- Separate immutable signed system images from user data and cache disks.
- Use sparse copy-on-write storage with online compaction and quota reporting.
- Add memory ballooning and host-pressure response; use snapshots only after graphics, clocks, and host integrations restore deterministically.
- Ship signed host and guest manifests, staged rollout, health checks, automatic rollback, and schema migrations.
- Never update the guest image while user data migration or snapshot restore is incomplete.
- Keep Desktop and TV image channels and userdata migrations distinct, while enforcing the same host/protocol compatibility range and release-signing roots.

## 6. Feature tiers

| Capability | Developer Preview | Public Beta | General Availability | Later |
|---|---|---|---|---|
| Boot, lifecycle, ADB, APK install | Yes | Yes | Yes | — |
| GLES/Vulkan acceleration | GLES first | GLES + Vulkan | Conformance-gated | Advanced extensions |
| Per-app native windows | Basic | Resize/orientation/PiP | Multi-monitor polished | Accessibility depth |
| Host-native app identity/UX | Native title/icon/close + host launcher entries | Windows taskbar/AUMID; macOS menus/Spotlight; shim result published | Platform behavior/accessibility conformance | Separate macOS Dock identity only if signed-shim gate passes |
| Per-app form factor | Auto + Phone/Tablet presets | Adaptive/fixed, orientation, size/density/aspect controls | Cross-update profile stability + compatibility matrix | Activity-specific/device-posture profiles |
| Clipboard | Bidirectional text | Text + sanitized HTML + bounded PNG | Loop/sensitivity/multi-instance/format gates | File lists and large content via broker |
| Host files | Selected folders | DocumentsProvider | Drag/drop and robust semantics | Virtiofs optimization |
| Keyboard/mouse/cursor | Basic keys + absolute pointer | Host IME + relative capture + cursor sync | Layout, focus, recovery, and latency gated | Touch mapping/macros only after policy review |
| Game controllers | One standard controller | Hot-plug + multiple players + basic haptics | Eight-device matrix + stable reconnect | Motion, LEDs, specialty HID/force feedback |
| Audio | Output | Output + microphone | Permission and latency gated | Specialty routing |
| Notifications | Title/body/open | Update/remove/group/progress/actions/reply | Privacy, dismiss reconciliation, restart and flood gates | Rich media/platform-specific enhancements |
| Deep links | Basic | Verified routing | Production | Additional host targets |
| Camera/mic/location | No | Opt-in | Permission-brokered | Additional sensors |
| Suspend/resume | Process idle | Snapshot preview | Reliable fast resume | Multi-instance restore |
| App installation/catalog | Host APK installer in every mode; F-Droid in `microG` | Updates + host launcher sync | Hardened install flow | Enterprise repositories/catalogs |
| Service modes | `NoApp` + `microG` | Transactional user-supplied GApps importer | Independently gated `NoApp`/`microG`/`GApps`; certified label only with partner approval | Conditional certified partner providers after licensing |
| Root providers and local modules | `Off`; KernelSU/Magisk engineering builds | Default-deny grants, user-selected local module import, safe mode, recovery | Provider/target combinations only if all gates pass | Online module catalogs out of scope |
| ARM APKs on Windows x64 | Provider import spike | Houdini/`libndk_translation` preview | User-supplied providers if gates pass | Additional licensed providers |
| Privacy/diagnostics | Zero telemetry + manual local export | User-facing redaction/viewer + egress tests | Audited zero-telemetry production gate | Enterprise offline diagnostics tooling |
| Sandbox hardening | Threat model + per-process spikes | Guest/host broker isolation | Measured layered hardening on every SKU | Additional scopes/mitigations after compatibility gates |

Android TV-specific delivery:

| Capability | Developer Preview | Public Beta | General Availability | Later |
|---|---|---|---|---|
| 10-foot launcher/SystemUI | Basic open launcher | Settings/search polish | D-pad/accessibility gated | Multiple launcher choices |
| Display | Windowed/full-screen 1080p SDR | 4K SDR + refresh profiles | Stable 1080p/4K SDR | HDR after conformance |
| Input | Keyboard/gamepad/D-pad | Multi-controller, media keys + push-to-talk | Focus ownership + remote-friendly recovery | HDMI-CEC/remote apps |
| Clipboard | Off by default; text opt-in | D-pad-accessible policy | Text-only secure default | Rich formats if TV UX requires them |
| Notifications | Off/summary opt-in | Per-app/channel allowlist | Redacted remote-friendly center | Rich actions after TV UX review |
| Video decode | Software + initial host decode | H.264/HEVC/VP9 host paths as available | Per-host support matrix | AV1/advanced profiles |
| Audio | Stereo PCM | Multichannel PCM | Stable routing/latency | Licensed bitstream passthrough |
| App discovery | APK in every mode; F-Droid in `microG` | Leanback filtering + gated GApps import | Per-mode TV compatibility catalog | Curated enterprise catalogs |
| DRM/protected playback | Unsupported | Unsupported | Unsupported unless separately licensed | Separate certification program |

## 7. Delivery sequence

The detailed work is split into phase files in this directory:

1. `phase-0-decision-gates.md` — architecture and feasibility spikes.
2. `phase-1-runtime-foundation.md` — shared repo, modern QEMU, native builds, CI.
3. `phase-2-android-guest.md` — AOSP products, kernels, HALs, guest agents.
4. `phase-2a-android-tv.md` — TV products, 10-foot UX, input, media, and TV app compatibility.
5. `phase-2b-service-modes-gapps.md` — immutable NoApp/microG/GApps modes and payload-free user GApps import.
6. `phase-2c-root-providers.md` — Off/KernelSU/Magisk artifacts, grants, modules, recovery, and security gates.
7. `phase-3-graphics.md` — Gfxstream/ANGLE and Trinity projection migration.
8. `phase-4-seamless-windows.md` — Desktop per-app windows and input/composition.
9. `phase-4a-input-cursor-controller.md` — cross-platform keyboard, IME, pointer/cursor, capture, controllers, and haptics.
10. `phase-4b-native-app-experience-display-profiles.md` — native Windows/macOS app shell, host identity/activation, and per-app Auto/Phone/Tablet profiles.
11. `phase-5-host-integration.md` — Windows/macOS Desktop and TV host features.
12. `phase-5a-clipboard-notifications.md` — bidirectional clipboard and Android-to-host notification lifecycle/actions.
13. `phase-6-security-update-compliance.md` — security, update, licensing, CTS/VTS/TV gates.
14. `phase-6a-zero-telemetry-sandbox.md` — GrapheneOS-derived audit, zero telemetry, guest/host isolation, egress proof, and hardening performance gates.
15. `phase-7-hardening-release.md` — performance, compatibility, beta, and GA.

Desktop critical path: Phase 0 → Phase 1 → Phase 2 → Phase 2B → Phase 3 → Phase 4 → Phase 4A + Phase 4B → Phase 5A → Phase 6A → Phase 7. TV critical path: Phase 0 → Phase 1 → Phase 2 → Phase 2A + Phase 2B → Phase 3 → Phase 4A → Phase 5 → Phase 5A → Phase 6A → Phase 7. Phase 2C runs in parallel and gates each rooted provider/target without blocking the secure `RootMode=Off` release. Phase 6 and Phase 6A start during Phase 1 and gate every public release.

## 8. Milestones and schedule

| Milestone | Target from kickoff | Exit outcome |
|---|---:|---|
| M0 Architecture and source gates | 8 weeks | WHPX Arm64/HVF/TV boots proven; source/graphics/input, service-mode/GApps/root, native-shell/macOS-identity, per-task display-profile, clipboard/notification, GrapheneOS-audit, zero-telemetry, and sandbox gates frozen |
| M1 Native runtime boot | 4 months | QEMU current boots x86_64 and arm64 guests on all three host targets |
| M2 Modern Desktop guest | 7 months | AOSP Desktop baseline boots; ADB/APK/lifecycle/storage/network work; `NoApp` and `Off` absence tests pass |
| M2-TV Android TV guest | 9 months | x86_64/arm64 TV products boot with launcher, D-pad, per-mode provisioning, audio, and 1080p display |
| M3 Accelerated graphics | 11 months | GLES and Vulkan compatibility backend passes agreed conformance subset |
| M4 Desktop seamless windows and input | 15 months | Native independent host windows, Auto/Phone/Tablet per-app settings, resize, host IME, cursor sync/capture, multi-controller input, launcher/activation, text clipboard, basic notifications, audio |
| M5 Desktop Developer Preview | 16 months | Signed Desktop builds for all targets and published compatibility constraints |
| M5-TV TV Developer Preview | 19 months | Signed TV builds for Windows x64/ARM64 and macOS ARM64; 1080p/4K preview matrix |
| M6 Desktop Public Beta | 20 months | Update/rollback, security review, 500-app matrix, rich clipboard, actionable notifications, crash/latency targets |
| M6-TV TV Public Beta | 23 months | Media/input/accessibility hardening and 250-TV-app matrix |
| M7 Desktop General Availability | 24 months | Desktop release gates met; support and update operations active |
| M8 TV General Availability | 28 months | TV release gates met on both products; media/remote support matrices published |

With fewer than ten full-time engineers, treat 36–48 months as the realistic range or stagger TV by an additional release cycle. Desktop and TV should not share a GA date unless the team and hardware lab are funded for both matrices.

The dates above cover the open/user-import product. A certified `GApps + Off` SKU has no credible fixed delivery date until Google/GMS/Play Protect and Widevine/OEM partner acceptance, legitimate device-identity provisioning, achievable integrity/DRM levels, and banking test ownership are confirmed in Phase 0. If certification is mandatory for that SKU, its GA is blocked independently without delaying `NoApp`, `microG`, or uncertified Developer releases.

## 9. Team model

Minimum recommended core team:

- 2 virtualization/QEMU engineers.
- 3 Android platform/kernel/HAL engineers.
- 1 Android TV framework/launcher/input engineer.
- 3 graphics engineers across Gfxstream, ANGLE, Vulkan, Metal, and Direct3D.
- 1 media/audio engineer across MediaCodec, Media Foundation, VideoToolbox, WASAPI, and CoreAudio.
- 2 Windows native engineers.
- 2 macOS native engineers.
- 1 security/privacy/sandbox engineer.
- 2 quality/performance automation engineers.
- Shared product, release, design, legal/licensing, and developer-relations support.

Assign one architecture owner for the host/guest protocol and one release owner for cross-product version compatibility. Neither platform team may extend the protocol without cross-platform review and compatibility tests.

## 10. Quality and performance budgets

Initial targets, finalized after Phase 0 baselines:

- Cold subsystem boot to Android-ready: 8 seconds median, 12 seconds p95 on reference hardware.
- Resume from suspended state: 2 seconds median, 4 seconds p95.
- Warm Android app launch: 2.5 seconds p95 for the reference app set.
- Interactive frame pacing: 60 FPS with less than 5% jank on the reference UI suite.
- Host capture to virtio enqueue: under 4 ms p95 for keyboard, mouse buttons, and controller state on reference hardware.
- Guest receipt to Android dispatch: under 8 ms p95; report click/button-to-present separately because it includes app and rendering latency.
- Host cursor movement remains local and is not frame-bound; relative mouse deltas support 1,000 Hz without unbounded queues, with 8,000 Hz characterized and safely coalesced.
- Running-instance text clipboard propagation: under 500 ms p95 on Windows and under 1.5 seconds p95 on macOS; rich-format decoding is measured separately.
- Android notification post/update/remove to host reconciliation: under 1 second p95 while running; warm action/reply dispatch back to Android under 2 seconds p95. Cold-start time is reported separately.
- Idle working set after ballooning: under 1.5 GB for the phone profile.
- TV 1080p UI: stable 60 FPS with D-pad response under 100 ms p95; 4K UI performance is reported separately.
- Supported host-decoded 4K media: no dropped-frame bursts over 1% in a 10-minute reference clip and A/V drift under 40 ms.
- TV idle working set after ballooning: under 2.0 GB, excluding host decoder surfaces and file cache.
- No unbounded guest-controlled allocation in host processes.
- Crash-free host sessions: at least 99.8% in public beta and 99.95% at GA.
- Update rollback success: at least 99.9% in fault-injection tests.
- Added hardening overhead against the same current-AOSP baseline: no more than 3% geomean CPU on the agreed interactive/app suite, 5% settled resident-memory growth, 5% cold/warm app-launch regression, or 1 percentage point additional UI jank.
- Added broker control-message latency: under 1 ms p95 on reference hardware; bulk media/graphics paths have separate zero-copy and throughput budgets.
- First-party network egress: zero undocumented destinations and zero telemetry requests during boot, idle, application launch, induced crash, update, suspend/resume, and shutdown captures.
- Interactive host resize to guest configuration acknowledgement: under 100 ms p95; first correctly sized Android frame under 250 ms p95 for responsive reference apps, without blocking native window movement or cursor feedback.
- Form-factor profile changes that require task recreation meet the warm-launch target; rollback after crash loop or surface timeout restores the last known-good profile and placement automatically.

Performance claims must always include host CPU, GPU, memory, OS build, guest build, graphics backend, thermal state, and test version.

## 11. Validation strategy

- Unit tests for codecs, state machines, resource ownership, storage migrations, package catalog, and policy.
- Protocol golden tests across old/new guest and host versions.
- Coverage-guided fuzzing for every guest-controlled host parser and graphics/control message.
- QEMU qtests for custom devices and reset/snapshot behavior.
- Input protocol golden tests and virtio-input qtests for device discovery, event order, reset, hot-unplug, status/output handling, malformed capabilities, and queue pressure.
- ANGLE, Gfxstream, Vulkan CTS subsets, and dEQP for graphics conformance.
- Android CTS continuously; VTS for kernels and HALs; CTS Verifier before release candidates.
- A curated 100-app blocking suite and 500-app beta matrix covering games, browsers, media, productivity, banking, multi-window, camera, microphone, and native-code ABIs.
- A separate 50-app blocking and 250-app TV beta matrix covering Leanback launchers, D-pad focus, media sessions, subtitles, refresh changes, resolutions, codecs, multichannel PCM, screensavers, and touch-only fallback.
- Keyboard/IME matrix covering US/ISO/JIS/AZERTY hardware, Vietnamese and CJK composition, dead keys, AltGr, emoji, lock keys, repeats, shortcuts, focus transfer, and suspend/resume.
- Pointer matrix covering 125/500/1,000/8,000 Hz mice, multiple DPI/scaling/rotation configurations, multi-monitor movement, buttons 1–5, two-axis wheels, absolute/relative modes, capture/release, and standard/custom Android cursor hotspots.
- Controller matrix covering current Xbox, DualShock/DualSense, Switch Pro, 8BitDo/generic HID, wired/Bluetooth, 1–8 players, hot-plug/reconnect, battery, basic haptics, and TV navigation ownership.
- Clipboard matrix covering Unicode, long text, multiple items, sanitized HTML, PNG, empty/clear, sensitive/remote flags, classification duplicate callbacks, format-owner exit, delayed rendering, multi-instance races, crash/restart, and malformed/oversized payloads.
- Notification matrix covering post/update/group/progress/remove/open, clearable/ongoing/local-only/private/secret, actions, direct reply, authentication, passive/explicit dismiss, permission denial/revocation, DND, flood control, guest cold start, user/profile isolation, and stale-token replay.
- Native-shell matrix covering Windows caption/system menu/Snap/Alt+Tab/taskbar/AppUserModelID/Start/Search/DPI/virtual desktops and macOS traffic-light controls/menu bar/Window menu/Cmd shortcuts/Spaces/fullscreen/Retina/Spotlight/URL-file activation/accessibility/state restoration.
- Per-app presentation matrix covering Auto/Phone/Tablet, compact/medium/expanded thresholds, portrait/landscape, adaptive/locked mode, density, aspect/letterbox, min/max resize, multi-monitor DPI/Retina moves, task recreation, crash rollback, app update, guest update, user/profile and cross-instance isolation.
- Android TV CTS/CDD checks for every advertised television feature; TV-specific SystemUI, Settings, input, accessibility, media, and long-play tests.
- Hardware matrix covering Intel, AMD, Qualcomm Windows PCs and multiple Apple Silicon generations.
- Suspend/resume, low-memory, low-disk, GPU-reset, driver-upgrade, Windows Update, macOS update, network transition, and abrupt-power-loss testing.
- Security review, dependency scanning, SBOM verification, update-signing drills, and renderer sandbox escape assessment.
- Static dependency/SDK scan and runtime packet-capture tests proving that production artifacts contain no telemetry client or undocumented first-party egress; repeat with microG disabled/enabled, F-Droid idle/syncing, and online/offline updates.
- Service-mode matrix covering clean creation, import/provision, boot, update, clone, reset, rollback, account/state non-migration, negative package/signature tests, and cross-mode isolation for `NoApp`, `microG`, and `GApps`; use only synthetic/free fixtures in CI for the GApps importer.
- Certified `GApps + Off` matrix—on partner-approved targets only—covering exact build/fingerprint/patch level, locked verified boot, protected identity/attestation provisioning, official server-side Play Integrity verdicts/replay handling, applicable legacy SafetyNet, licensed Widevine security level and secure decode/render path, regional banking blocking apps, OTA/reset/rollback/key rotation, and rooted/tampered negative controls.
- Root-mode matrix covering clean absence, source reproducibility, grant/revoke, manager authentication, module install/update/remove, bootloop safe mode, host recovery, update/rollback, SELinux/AVB enforcement, and host-boundary attacks for `Off`, KernelSU, and Magisk on every eligible target.
- Guest sandbox tests for UID/profile separation, SELinux enforcing, seccomp, Binder/file/network/sensor boundaries, dynamic-code policy, malformed IPC, and resource exhaustion; host tests for per-broker tokens/entitlements, IPC ACLs, file/network/device denial, Job Object/pressure quotas, crash restart, and sandbox escape attempts.
- Performance A/B tests for every hardening change on Windows x64/ARM64 and macOS ARM64; failures require optimization, measured narrowing, or an explicit compatibility exception with owner and expiry.

## 12. Principal risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| WHPX Arm64 gaps | Blocks native Windows ARM64 | Phase 0 hard gate; capability detection; isolate accelerator adapter; no TCG performance promises |
| Missing Trinity guest projection source | Blocks exact migration | Treat old projection path as reference; compatibility backend ships first; reimplement behind a documented protocol |
| LeapGL age and incomplete repo | Blocks incremental port | Preserve as reference only; rewrite runtime around current QEMU/AOSP/Gfxstream/ANGLE |
| Graphics driver variability | Crashes and rendering defects | ANGLE abstraction, adapter deny/allow lists, SwiftShader recovery, GPU-process isolation |
| Per-app surface export complexity | Delays WSA-like UX | Deliver one seamless window first; stabilize task protocol before PiP/multi-monitor |
| Host shell looks native but behaves inconsistently | Product still feels like an emulator | Real AppWindow/HWND and NSWindow only; platform-native chrome/commands/lifecycle/accessibility conformance; no custom-drawn host controls |
| macOS cannot provide safe per-package Dock identity | Android apps group under LeapDroid in Dock | Phase 0 signed/notarized launcher-shim gate; preserve per-window native UX and on-device Spotlight activation; publish the limitation if the gate fails |
| Phone/Tablet override destabilizes apps | Relaunch loops, broken layouts, cross-app configuration leaks | Per-task/display configuration, Android size-compatibility fallback, transactional apply/rollback, last-known-good profile, compatibility database and reset action |
| Duplicate key/text paths or conflicting input backends | Double characters, missed shortcuts, nondeterministic games | One authoritative backend per device class; explicit physical-key versus IME transactions; golden event traces |
| Focus loss, capture lockout, or stuck input/haptics | Host UX and safety failure | Atomic release-all on every lifecycle edge; host-owned escape/recovery path; watchdog and fault-injection tests |
| Controller and high-polling mouse variability | Compatibility and latency regressions | Canonical descriptors, capability gates, bounded coalescing, stable player slots, representative hardware lab |
| Clipboard loops or sensitive-data exfiltration | Repeated writes, data loss, privacy breach | Origin/revision/digest deduplication, active-instance arbitration, sensitive block by default, no history/content in logs or exports, sandboxed rich formats |
| Stale notification actions or forged reply/dismiss | Cross-app action or privilege misuse | Guest-retained PendingIntent, short-lived revision-bound opaque capabilities, authentication checks, replay protection, per-user routing |
| Host notification API/identity differences | Inexact dismissal/rendering and user confusion | Own-notification reconciliation, normalized fallback, explicit Trinity/LeapDroid branding, per-platform capability matrix |
| GPL/Apache/proprietary mixing | Distribution risk | Process and repository boundaries, notices, source offers, automated license inventory, legal review |
| microG API gaps or signature-spoofing abuse | App failures or identity/security weakening | Officially signed microG, certificate/package allowlist, independent toggles, explicit compatibility claims, regression tests |
| F-Droid supply-chain or privileged install compromise | Malicious app installation | Pin official client signer, verify signed indexes/APK hashes, default to user-confirmed install, security-gate any privileged helper |
| GApps licensing, certification, compatibility, or malicious import | Distribution exposure, broken boot/update, credential or privilege compromise | User-import path uses local files only with no discovery/download/distribution and a strict sandboxed importer; certified path requires exact licensed partner supply chain, protected keys, signed provenance, and transactional rollback |
| Official certification/attestation or licensed DRM is unavailable for a virtual target | `GApps + Off` cannot meet protected-app release requirements | Early Google/Widevine/OEM feasibility gate; separate signed `CertifiedPartner` provider; no key/fingerprint spoofing; do not ship/label the certified SKU on failed targets; retain truthful uncertified import mode |
| Banking or protected-media policy changes after release | Previously passing apps/services reject the product | App-owner test accounts, regional blocking matrix, signed-build/OTA regression, published per-app/service results, rapid rollback, and no compatibility guarantee beyond tested versions |
| Guest root or a malicious root module escapes its intended boundary | Guest compromise, bootloop, data loss, or attempted host attack | `Off` default; separate signed artifacts; default-deny grants; no online module store; modules treated as arbitrary guest-root code; host-owned safe boot/recovery; broker/VM isolation and fuzzing remain mandatory |
| KernelSU x86_64 requires weaker syscall mitigation | Security regression on Windows x64 | Target-gate the pinned version; do not ship KernelSU if syscall hardening must be disabled; retain `Off` and independently gated Magisk |
| Houdini/`libndk_translation` legal and version uncertainty | ARM-only APK compatibility gaps | User-supplied provider model, no bundled/downloaded binaries, legal acknowledgement, strict validation, per-provider matrix, `Off` fallback |
| WSA code availability misconception | Schedule or licensing failure | Use only published kernel source after a current-kernel diff; independently implement all runtime and host features |
| Waydroid architecture mismatch | Time lost porting Linux container code | Reuse only guest/HAL/image/session patterns; retain QEMU VM isolation and native host presentation |
| LineageOS patchset drift | Annual Android upgrade and CTS cost | Keep AOSP canonical; maintain a small provenance-tracked import queue with upstream/removal checks |
| Whole-platform GrapheneOS fork or false equivalence | Merge/CTS cost and misleading security claims | Keep AOSP canonical; import only provenance-tracked changes; exclude Pixel-specific code/services/branding; publish the exact supported hardening set |
| Sandbox overhead causes jank or latency | Poor desktop/TV experience and pressure to disable protection | Per-process threat ranking, lazy helpers, bounded IPC, measured rollout, hard budgets, and narrow compatibility exceptions instead of global disablement |
| Telemetry or hidden endpoint reintroduced by a dependency | Privacy breach and loss of trust | No analytics/crash-upload dependencies, machine-readable egress manifest, static scans, packet-capture release gate, and manual local diagnostics only |
| TV media and DRM expectations | Commercial streaming apps fail despite good UI | Publish codec/DRM matrix, prioritize open media paths, make no Widevine L1/HDCP claims, separate any future licensing program |
| Desktop + TV scope expansion | Schedule or quality dilution | Share runtime/system/vendor; separate product/userdata and QA matrices; Desktop GA first, TV GA four months later |
| Two host platforms stretch team | Schedule and quality risk | Shared guest/protocol/runtime; stagger public previews; one cross-platform release train |
| Snapshot correctness | Data loss or stale host handles | Suspend without snapshot first; snapshot only after deterministic restore and fault injection |
| Attack surface from guest protocols | Host compromise | Least-privilege processes, sandboxed renderer, bounded messages, fuzzing, SELinux/AVB, signed updates |

## 13. Decisions required before implementation approval

1. Confirm exactly three immutable service modes: `NoApp` (default, clean AOSP), `microG` (verified F-Droid + approved microG), and `GApps` (user-imported uncertified provider or separately licensed `CertifiedPartner` artifact). Recommendation: locked; switching creates/clones a new instance and never migrates accounts/tokens/system state.
2. Confirm GApps policy: the project never mirrors, searches for, downloads, or extracts proprietary Google payloads. User import remains uncertified; only an exact licensed `GApps + RootMode=Off` partner artifact may pursue Play Protect/Play Integrity/Widevine/banking release gates, with unsupported targets withheld. Recommendation: locked.
3. Confirm independent `RootMode=Off|KernelSU|Magisk`, with `Off` default, one root provider per Developer instance, no concealment/bypass support, and target-gated KernelSU x86_64. Recommendation: locked.
4. Confirm the first two Windows x64 Native Bridge adapters are Houdini and `libndk_translation`, both user-supplied. Recommendation: approve only after Phase 0 legal/technical gates; keep `Off` as default.
5. Decide whether F-Droid app installs always require Android confirmation or may use the Privileged Extension after security review. Recommendation: confirmation for preview; privileged installs only after signer-restricted hardening.
6. Confirm minimum Windows and macOS versions and whether Microsoft Store/Mac App Store distribution is mandatory.
7. Confirm whether Trinity projection is a release requirement or a post-compatibility performance feature. Recommendation: post-compatibility feature.
8. Confirm whether the products share a public runtime repository or use synchronized internal mirrors. Recommendation: one canonical shared runtime repository.
9. Confirm product priority if schedules conflict. Recommendation: Trinity Windows x64 developer preview first, LeapDroid macOS ARM64 second, Trinity Windows ARM64 after the Phase 0 accelerator gate.
10. Confirm Desktop and TV use separate instances/userdata rather than an in-place mode switch. Recommendation: approve; Android product characteristics and app/input policies differ materially.
11. Confirm staged release order: Desktop GA at month 24 and TV GA at month 28. Recommendation: approve unless two additional TV/media engineers and expanded hardware QA are funded.
12. Decide the TV F-Droid UX after the Phase 0 D-pad audit: official client if usable, otherwise a minimal TV frontend over reviewed signed-index components. Recommendation: never ship a TV frontend that weakens F-Droid verification.
13. Confirm the host owns cursor rendering while Android supplies icon/visibility/hotspot metadata. Recommendation: approve; this removes a frame of cursor latency and prevents double cursors.
14. Confirm Windows uses Raw Input for keyboard/mouse and GameInput for controllers, while macOS uses AppKit/NSTextInputClient and GameController. Recommendation: approve; keep XInput only as a measured compatibility fallback and prohibit duplicate reads.
15. Decide whether host key-to-touch mapping, macros, and rapid-fire belong in first GA. Recommendation: defer; they expand security, accessibility, support, and game-policy scope beyond standards-based input.
16. Confirm Desktop clipboard direction and formats. Recommendation: two-way current clipboard after disclosure; text at Preview, sanitized HTML and bounded PNG at Beta, files only through the explicit file broker later.
17. Confirm sensitive and clear semantics. Recommendation: never auto-export Android clips marked sensitive; do not propagate clear unless it targets the same still-current bridged revision.
18. Confirm notification permissions and dismissal. Recommendation: require Android notification-access and host authorization, reconcile only the product's own host notifications, and never request Windows access to unrelated notifications.
19. Confirm TV defaults. Recommendation: clipboard off/text-only opt-in; notifications opt-in with hidden previews and per-app/channel allowlisting for a 10-foot environment.

Locked product decisions: production zero telemetry is mandatory and not an opt-out setting; diagnostics are local/manual only. Instances use immutable `ServiceMode=NoApp|microG|GApps` and `RootMode=Off|KernelSU|Magisk`, with `NoApp + Off` as the default. User-imported GApps is always uncertified; an exact `GApps + Off` artifact may be labeled certified only through written partner approval, legitimate attestation provisioning, official Play Integrity tests, licensed Widevine integration, and the banking blocking matrix. Root is an advanced developer feature without built-in concealment/bypass payloads. Rooted users may explicitly import compatible local KernelSU/Magisk modules as untrusted guest-root code; the product provides no module discovery/download/catalog and never grants a module host capability. GrapheneOS is a selective hardening reference over AOSP, not the guest distribution or a security-equivalence claim. Desktop uses real native host windows and native shell controls; `Auto`, `Phone`, and `Tablet` are first-class per-app presentation modes, with `Auto` as the default. Android renders the internal app UI. Separate macOS Dock identity remains gated on the signed launcher-shim proof.

## 14. Reference sources

- Microsoft Windows Hypervisor Platform API: https://learn.microsoft.com/en-us/virtualization/api/hypervisor-platform/hypervisor-platform
- Microsoft Windows App SDK: https://learn.microsoft.com/en-us/windows/apps/windows-app-sdk/
- Microsoft WSA repository and feature inventory: https://github.com/microsoft/WSA/
- Microsoft WSA Linux kernel source: https://github.com/microsoft/WSA-Linux-Kernel
- Apple Hypervisor framework: https://developer.apple.com/documentation/hypervisor
- Apple Virtualization framework: https://developer.apple.com/documentation/virtualization
- QEMU supported build platforms: https://www.qemu.org/docs/master/about/build-platforms.html
- ANGLE repository and backend matrix: https://chromium.googlesource.com/angle/angle/
- AOSP Cuttlefish GPU acceleration: https://source.android.com/docs/devices/cuttlefish/gpu
- AOSP Gfxstream: https://android.googlesource.com/platform/hardware/google/gfxstream/
- Android CTS: https://source.android.com/docs/compatibility/cts
- Android VTS: https://source.android.com/docs/core/tests/vts
- Android compatibility and GMS licensing FAQ: https://source.android.com/docs/compatibility/compatibility-faq
- Google Play Protect certification: https://support.google.com/googleplay/answer/7165974
- Google Play services overview: https://developers.google.com/android/guides/overview
- Google Play Integrity API overview: https://developer.android.com/google/play/integrity/overview
- SafetyNet Attestation deprecation notice: https://developer.android.com/privacy-and-security/safetynet
- Widevine DRM overview: https://developers.google.com/widevine/drm/overview
- Widevine partner access and L1 OEMCrypto requirement: https://developers.google.com/widevine/access
- AOSP ART Native Bridge: https://android.googlesource.com/platform/art/+/refs/heads/main/libnativebridge/
- microG GmsCore: https://github.com/microg/GmsCore
- microG signature spoofing requirements: https://github.com/microg/GmsCore/wiki/Signature-Spoofing
- F-Droid Client: https://f-droid.org/packages/org.fdroid.fdroid/
- F-Droid release channels and signing keys: https://f-droid.org/docs/Release_Channels_and_Signing_Keys/
- KernelSU installation and GKI integration: https://kernelsu.org/guide/installation.html
- KernelSU x86_64 support caveat: https://kernelsu.org/guide/x86_64-support.html
- KernelSU module guide: https://kernelsu.org/guide/module.html
- KernelSU source and licensing: https://github.com/tiann/KernelSU
- Magisk installation: https://topjohnwu.github.io/Magisk/install.html
- Magisk developer/module/Zygisk guides: https://topjohnwu.github.io/Magisk/guides.html
- Magisk build and signing: https://topjohnwu.github.io/Magisk/build.html
- Magisk source and licensing: https://github.com/topjohnwu/Magisk
- Intel Bridge Technology licensing context: https://www.intel.com/content/www/us/en/developer/topic-technology/bridge-technology.html
- Waydroid runtime: https://github.com/waydroid/waydroid
- Waydroid Android hardware integration: https://github.com/waydroid/android_hardware_waydroid
- Waydroid Android vendor integration: https://github.com/waydroid/android_vendor_waydroid
- Waydroid Desktop/full-UI/multi-window documentation: https://docs.waydro.id/usage/install-on-desktops
- LineageOS platform manifest: https://github.com/LineageOS/android
- LineageOS framework base: https://github.com/LineageOS/android_frameworks_base
- LineageOS Android TV device tree: https://github.com/LineageOS/android_device_lineage_atv
- LineageOS TV SystemUI: https://github.com/LineageOS/android_packages_apps_TvSystemUI
- AOSP Android TV products: https://android.googlesource.com/device/google/atv/+/refs/heads/main/products/
- LineageOS device-support requirements: https://github.com/LineageOS/charter/blob/main/device-support-requirements.md
- OASIS Virtio input device specification: https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html#x1-4130008
- Microsoft Raw Input overview: https://learn.microsoft.com/en-us/windows/win32/inputdev/about-raw-input
- Microsoft GameInput fundamentals and device model: https://learn.microsoft.com/en-us/gaming/gdk/docs/features/common/input/overviews/input-fundamentals
- Apple AppKit mouse, keyboard, and trackpad input: https://developer.apple.com/documentation/appkit/mouse-keyboard-and-trackpad
- Apple GameController framework: https://developer.apple.com/documentation/gamecontroller
- Android input architecture: https://source.android.com/docs/core/interaction/input
- Android natural input across form factors: https://developer.android.com/games/develop/multiplatform/enable-natural-input-on-all-form-factors
- Android game-controller input: https://developer.android.com/games/sdk/game-controller/controller-input
- Android PointerIcon API: https://developer.android.com/reference/android/view/PointerIcon
- Microsoft Windows clipboard listener: https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-addclipboardformatlistener
- Microsoft Windows clipboard guidance: https://learn.microsoft.com/en-us/windows/win32/dataxchg/using-the-clipboard
- Microsoft Windows App SDK notifications: https://learn.microsoft.com/en-us/windows/apps/develop/notifications/app-notifications/
- Microsoft AppNotificationManager notification management: https://learn.microsoft.com/en-us/windows/apps/develop/notifications/app-notifications/manage-app-notifications
- Microsoft AppNotificationManager own-notification enumeration: https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.windows.appnotifications.appnotificationmanager.getallasync
- Apple NSPasteboard: https://developer.apple.com/documentation/appkit/nspasteboard
- Apple UNUserNotificationCenter: https://developer.apple.com/documentation/usernotifications/unusernotificationcenter
- Apple actionable notifications: https://developer.apple.com/documentation/usernotifications/declaring-your-actionable-notification-types
- Android ClipboardManager: https://developer.android.com/reference/android/content/ClipboardManager
- Android ClipDescription remote/sensitive metadata: https://developer.android.com/reference/android/content/ClipDescription
- Android NotificationListenerService: https://developer.android.com/reference/android/service/notification/NotificationListenerService
- Android Notification actions and direct reply: https://developer.android.com/reference/android/app/Notification.Action
- GrapheneOS features and privacy/security design: https://grapheneos.org/features
- GrapheneOS source and repository map: https://grapheneos.org/source
- GrapheneOS platform manifest: https://github.com/GrapheneOS/platform_manifest
- AOSP Application Sandbox: https://source.android.com/docs/security/app-sandbox
- AOSP SELinux in Android: https://source.android.com/docs/security/features/selinux
- AOSP MTE configuration and performance scoping: https://source.android.com/docs/security/test/memory-safety/mte-configuration
- Microsoft AppContainer isolation: https://learn.microsoft.com/en-us/windows/win32/secauthz/appcontainer-isolation
- Microsoft Job Objects: https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects
- Microsoft Process Mitigation Options: https://learn.microsoft.com/en-us/windows/security/threat-protection/override-mitigation-options-for-app-related-security-policies
- Apple App Sandbox: https://developer.apple.com/documentation/security/app_sandbox
- Apple Hardened Runtime: https://developer.apple.com/documentation/security/hardened-runtime
- Apple XPC: https://developer.apple.com/documentation/xpc
- Microsoft Windows App SDK windowing overview: https://learn.microsoft.com/en-us/windows/apps/develop/ui/windowing-overview
- Microsoft AppWindow API: https://learn.microsoft.com/en-us/windows/windows-app-sdk/api/winrt/microsoft.ui.windowing.appwindow
- Microsoft AppUserModelID window identity sample: https://learn.microsoft.com/en-us/windows/win32/shell/samples-appusermodelidwindowproperty
- Microsoft Windows App SDK app activation: https://learn.microsoft.com/en-us/windows/apps/develop/launch/activate-an-app
- Apple NSWindow: https://developer.apple.com/documentation/appkit/nswindow
- Apple AppKit menus and Window menu: https://developer.apple.com/documentation/appkit/menus
- Apple Core Spotlight: https://developer.apple.com/documentation/corespotlight
- Apple NSUserActivity: https://developer.apple.com/documentation/foundation/nsuseractivity
- Android device compatibility and size-compatibility mode: https://developer.android.com/guide/practices/device-compatibility-mode
- Android multi-window support: https://developer.android.com/develop/ui/views/layout/support-multi-window-mode
- Android window size classes: https://developer.android.com/develop/ui/views/layout/use-window-size-classes
- Android `Configuration`: https://developer.android.com/reference/android/content/res/Configuration
- AOSP Android 17 desktop windowing: https://source.android.com/docs/core/display/desktop-windowing
