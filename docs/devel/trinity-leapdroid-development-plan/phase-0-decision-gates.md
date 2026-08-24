# Phase 0 — Architecture and Feasibility Gates

Duration: 8 weeks  
Purpose: Eliminate platform risks before committing to the multi-year implementation.

## Workstreams

### Windows x64 and ARM64 acceleration

- Build a current stable QEMU for native x64 and ARM64 Windows.
- Prove x86_64-on-x64 and arm64-on-ARM64 guests under WHPX.
- Validate GIC/APIC behavior, timers, SMP, memory mapping, interrupt injection, virtio-blk, virtio-net, serial, shared memory, suspend, and clean reset.
- Record the minimum Windows build and runtime `Arm64Support` capability requirements.
- Demonstrate that TCG is not used on any interactive production path.

### macOS ARM64 acceleration

- Build current QEMU arm64 natively with HVF and required entitlements.
- Boot an AArch64 Linux/Android-derived kernel on M1 and a newer Apple Silicon generation.
- Validate timers, GIC, SMP, memory mapping, virtio devices, sleep/wake, app sandbox/signing constraints, and crash recovery.
- Compare QEMU HVF with a minimal Virtualization.framework spike for lifecycle, device extensibility, display, clipboard, and packaging. Select exactly one VM core for the product.

### Graphics proof

- Run current Gfxstream host/guest smoke tests on Windows x64, Windows ARM64, and macOS ARM64.
- Build ANGLE natively for x64, ARM64 Windows, and macOS ARM64.
- Render the same GLES sample through D3D11 and Metal, then validate a Vulkan sample or document the blocker.
- Verify SwiftShader fallback and GPU-process restart.
- Measure command throughput, frame latency, host copies, and memory use against the existing Trinity and LeapGL paths where runnable.

### Android guest proof

- Pin a candidate AOSP release and boot minimal x86_64 and arm64 images.
- Confirm binder, SELinux enforcing, PackageManager, ADB, virtio input, storage, networking, audio, and Gfxstream dependencies.
- Decide whether to derive product definitions from Cuttlefish or emulator/generic targets. Recommendation: Cuttlefish-derived virtual hardware and services with product-specific desktop integration.

### Native app shell and per-app display-profile proof

- Export at least three simultaneous Android tasks into real AppWindow/HWND and NSWindow containers. Verify native move/resize/minimize/maximize/fullscreen/close, Alt+Tab or Window menu, taskbar/Dock behavior, DPI/Retina monitor moves, focus, input, accessibility, and surface recovery.
- Windows: prove per-window title/icon, Snap Layouts/system menu, Start/Search shortcuts, protocol activation into an existing instance, and distinct taskbar grouping/identity with AppUserModelID without creating one VM per app.
- macOS: prove native traffic-light controls, menu bar/Window menu/Services/Edit commands, Cmd+W lifecycle, Spaces/fullscreen, Spotlight/`NSUserActivity` activation, URL/file open, restoration, and per-task AppKit accessibility.
- Run a hard feasibility spike for a separate macOS Dock/Launchpad identity per Android package. The design must use a safe signed/notarized launcher shim, survive Gatekeeper/App Sandbox/update/uninstall, contain no mutable executable payload, and avoid dynamically signing with a user/developer key. If it fails, freeze the supported fallback: one LeapDroid Dock identity, native NSWindows with per-app title/icon/menu context, and individual on-device Spotlight launch results.
- Compare a shared Android display with per-task configuration overrides against one virtual display/`TaskDisplayArea` per exported task on Android 17. Measure correctness, isolation, SurfaceFlinger/Gfxstream memory, frame latency, resize churn, input routing, and lifecycle before selecting one architecture.
- Prototype a versioned per-app `Auto`/`Phone`/`Tablet` profile. Confirm compact resources below 600dp, medium resources at 600dp or above, expanded resources at 840dp or above, correct `screenWidthDp`/`screenHeightDp`/`smallestScreenWidthDp`, density/orientation, and no global/cross-task configuration leak.
- Exercise resizable, non-resizable, fixed-orientation, min/max-aspect, legacy, Compose-adaptive, View-adaptive, camera, game, and multi-activity apps. Prove Android size-compatibility letterboxing, explicit force-resize exceptions, task-only recreation, crash-loop rollback, and profile reset.
- Freeze native close/minimize/quit/force-stop semantics, settings entry points, profile precedence, migration/versioning, last-known-good behavior, and TV exclusion before implementation.

### Waydroid and LineageOS source audits

- Pin current Waydroid runtime, `android_hardware_waydroid`, and `android_vendor_waydroid` revisions; inventory host assumptions, HALs, AIDL interfaces, init, SELinux, image split, session lifecycle, app launch, and full-UI/multi-window behavior.
- Mark Waydroid LXC, shared-host-kernel, Binder-device, and Wayland-socket code as non-portable to the Windows/macOS VM architecture. Evaluate guest/HAL changes individually against current AOSP and the planned virtio/Gfxstream protocol.
- Pin the matching current LineageOS manifest, `frameworks/base`, ATV device tree, and TvSystemUI revisions. Diff restricted microG signature spoofing and TV patches against the chosen AOSP tag.
- For every candidate change, record source commit, license, affected AOSP component, security/CTS impact, upstream equivalent, test, and `import`, `reimplement`, or `drop` disposition.
- Keep AOSP as the baseline unless this audit demonstrates a quantified lower-cost LineageOS base; the default decision is selective imports only.

### GrapheneOS hardening audit

- Pin the current GrapheneOS manifest and relevant forks against the selected AOSP tag. Inventory hardened SELinux/seccomp policy, secure app spawning, `hardened_malloc`, zero-on-free, dynamic-code/JIT restrictions, Network/Sensors permission controls, local log viewer, Storage Scopes, and Contact Scopes.
- For each candidate, record source commit, license, current-AOSP equivalent, guest/host threat addressed, CTS/VTS/app-compatibility impact, CPU/memory/startup/frame-time cost, test, rollback, and `import`, `reimplement`, `defer`, or `drop` decision.
- Prioritize SELinux/seccomp tightening, per-app Network/Sensors controls, local log/crash viewing, and targeted allocator/memory hardening for high-risk native processes. Evaluate Storage/Contact Scopes after the base permission controls stabilize.
- Explicitly reject the whole GrapheneOS manifest as the product base, GrapheneOS network infrastructure as a dependency, copying GrapheneOS Sandboxed Google Play implementation/services, Pixel firmware/radio/USB/attestation code, GrapheneOS branding, and claims of equal security. The separate user-supplied GApps provider must not inherit GrapheneOS security claims.
- Prove that microG and F-Droid remain ordinary sandboxed apps in `ServiceMode=microG`. Signature spoofing is limited to pinned microG identities; F-Droid receives no privileged installer role unless its separate review passes.

### Zero-telemetry and layered host sandbox proof

- Define a production build profile with no analytics SDK, crash uploader, usage tracking, remote logging, experiments, stable installation/device ID, or undocumented metrics endpoint. Development instrumentation must be compile-time or locally policy-gated and absent from production artifacts.
- Define bounded local ring-buffer logs, health counters, trace sessions, crash artifacts, user-facing preview/redaction, retention/size caps, and manual export. Prove that induced crashes never upload automatically.
- Produce a machine-readable first-party egress manifest separating signed update, F-Droid/opt-in microG, user-installed GApps, rooted apps/modules, and ordinary user-app traffic from telemetry. Packet-capture boot, idle, app launch, crash, update, suspend/resume, and shutdown across all eligible `ServiceMode`/`RootMode` combinations.
- Prototype static signed update metadata without stable identifiers, per-device query parameters, or tracking cookies. Prove offline/enterprise update and disabled or user-configured connectivity/time services.
- Freeze the process/privilege map for UI, VM controller, renderer, media, clipboard, notifications, files, camera/microphone/location, updater, diagnostics, Native Bridge inspection, GApps import, and root-artifact inspection/recovery. Each broker gets a narrow authenticated IPC contract, explicit file/network/device capabilities, quotas, and crash boundary.
- Windows: prototype restricted tokens plus AppContainer/LPAC where compatible, Job Objects, process mitigations, IPC ACLs, and per-process firewall policy. macOS: prototype App Sandbox, Hardened Runtime, least entitlements, security-scoped bookmarks, and per-capability XPC helpers; record HVF/hypervisor and JIT/dynamic-code conflicts.
- A/B measure every hardening layer for startup, app launch, UI jank, input latency, broker IPC, CPU, memory, energy, and long-play TV behavior. If a gate fails, optimize first, then narrow protection to the high-risk process or define a per-app compatibility exception with owner/expiry.

### Android TV proof

- Build current AOSP `aosp_tv_x86`/arm64-derived products and boot them on Windows x64, Windows ARM64, and macOS ARM64 acceleration paths.
- Validate TV product characteristics, open launcher, TvSystemUI/Settings, provisioning, Leanback intent discovery, D-pad focus, gamepad/media keys, Leanback IME, accessibility, audio, and 1080p/4K display configuration.
- Prototype a single-display full-screen/windowed host presentation and confirm Desktop per-task surface export is disabled for TV instances.
- Test official F-Droid Client entirely with D-pad and 10-foot scaling. If it fails, specify a minimal TV frontend that preserves signed-index and APK verification.
- Prove at least one host-decoded media path through MediaCodec to Media Foundation/D3D11 and VideoToolbox/Metal, with software fallback and explicit unsupported protected-content behavior.

### WSA source audit

- Inventory the public `microsoft/WSA` repository and record that no shipping runtime/UI/integration implementation is available there.
- Diff `microsoft/WSA-Linux-Kernel` configs and patch history against the selected current Android common/GKI kernels for x86_64 and arm64.
- Prototype only kernel changes that solve a measured requirement; record provenance, license, security status, upstream equivalent, and whether the change should be imported, reimplemented, or dropped.
- Explicitly prohibit dependencies on extracted WSA MSIX binaries, guest images, undocumented protocols, or signing assets.
- Start from this preliminary disposition: retain WSA x64/arm64 configs as a checklist; consider only still-missing upstreamable Binder/security/scheduler fixes; reject the whole old kernel fork, legacy `ASHMEM`, and WSA-specific Hyper-V/`DXGKRNL` paths unless a new spike proves they fit the QEMU/Gfxstream architecture.

### Service-mode, GApps, and root-provider proof

- Prove immutable `ServiceMode=NoApp|microG|GApps`, default `NoApp`, cross-mode package/account/token absence, clone-to-switch semantics, and a host APK installer in every mode.
- Build F-Droid Client and microG packages from pinned official sources or verify official prebuilt signatures and reproducibility metadata. Prove signer/package-restricted signature spoofing and test microG self-check, service opt-ins, F-Droid verification, updates, disable, and reset.
- Prototype the payload-free GApps provider with synthetic/free fixtures only: sandboxed local import, strict descriptor/content/signature/hash/privilege validation, sealed read-only add-on, AVB/SELinux preservation, transactional boot health, rollback, removal, and an uncertified/best-effort UX.
- Run a separate certified-provider feasibility gate for `GApps + RootMode=Off`: obtain written Google/GMS/Play Protect program disposition, define legitimate attestation provisioning and expected Play Integrity verdicts, record SafetyNet legacy status, obtain Widevine licensing/device-integration disposition and achievable security level, and agree a regional banking blocking suite with app-owner-approved testing. No user-imported payload or copied device key can satisfy this gate.
- Prove immutable `RootMode=Off|KernelSU|Magisk`, default `Off`, reproducible separate boot/kernel artifacts, default-deny grants, safe mode/recovery, clean absence, and host/VM boundary preservation. Target-gate KernelSU x86_64 if syscall hardening would be weakened; keep Magisk Zygisk off by default.
- Review the current F-Droid Privileged Extension before allowing unattended installs; default to Android user-confirmed installation if the review is not complete.

### Windows x64 Native Bridge proof

- Implement a payload-free `NativeBridgeProvider` prototype against current AOSP `libnativebridge`.
- Define and test provider descriptors for user-supplied Houdini and `libndk_translation`, including required files, API/ABI versions, hashes, linker namespace, health checks, and rollback.
- Validate ARM32 and ARM64 APKs separately on an x86_64 guest; measure install success, correctness, crash rate, startup, CPU, memory, graphics, JNI, signals, and self-modifying-code cases.
- Complete legal review of importing and using each provider. If rights or a current compatible package cannot be demonstrated, keep that adapter experimental/unavailable and ship x86_64-only behavior.

### Keyboard, pointer, cursor, and controller proof

- Prototype one virtio-input keyboard, absolute pointer, relative pointer, and hot-pluggable controller on each host/guest architecture pair. Confirm Android EventHub/InputReader discovers standard evdev capabilities without a product-specific app API.
- On Windows, compare foreground Raw Input and GameInput traces, then freeze Raw Input as the sole keyboard/mouse reader and GameInput as the controller reader unless evidence requires a narrowly deduplicated fallback. Exercise buffered reads at 1,000 and 8,000 Hz.
- On macOS, prove `NSEvent`/`NSTextInputClient` composition and relative pointer capture without a global event tap or Accessibility permission; prove controller discovery, readings, and basic haptics with GameController.
- Demonstrate Vietnamese and one CJK host IME through a guest `InputConnection` without duplicate physical characters, plus dead keys, AltGr, key repeat, focus loss, and all-up recovery.
- Prototype host-authoritative cursor rendering: export Android standard/custom pointer icon and hotspot metadata, suppress the guest cursor, switch absolute/relative modes, and recover the native cursor after forced guest/renderer/input-broker crashes.
- Connect at least four mixed controllers, preserve player slots across reconnect where identity permits, map Android standard axes/buttons, and stop haptics on focus loss, disconnect, suspend, and reset.
- Freeze the emergency release shortcut and recovery-overlay behavior on both hosts. It must not depend on a responsive guest or renderer.

### Clipboard and notification proof

- Prototype bidirectional plain-text clipboard with origin/revision/acknowledgement loop prevention on Windows and macOS. Exercise Android duplicate classification callbacks, identical content copied twice, competing host/guest writes, two running instances, and same-revision clear behavior.
- Prove the Android clipboard agent can observe clips as a narrowly privileged platform component without granting background clipboard access to ordinary apps. Confirm `EXTRA_IS_REMOTE_DEVICE` handling and block automatic export of `EXTRA_IS_SENSITIVE` content.
- Validate Windows `AddClipboardFormatListener` lock/retry/delayed-rendering behavior and macOS `NSPasteboard.changeCount`/`accessBehavior` behavior, including permission denial and an energy measurement for adaptive observation.
- Prototype `NotificationListenerService` snapshot/delta reconciliation into Windows App SDK notifications and macOS UserNotifications. Cover stable update/removal, group/progress, opening, one safe action, direct reply, explicit/passive dismissal, and host/guest restart.
- Demonstrate guest-retained `PendingIntent` objects with short-lived notification-revision-bound opaque capabilities. Replay, forged token, update-before-click, remove-before-click, user switch, guest reboot, and snapshot restore must fail closed.
- Confirm Trinity can reconcile its own Windows notifications through AppNotificationManager without requesting the all-user-notifications listener. Record every unavoidable Windows/macOS rendering or dismissal difference in the support matrix.
- Freeze Preview/Beta/GA formats, size/rate limits, per-direction and per-app/channel policy, TV defaults, sensitive/private redaction, cold-start activation, clear/dismiss semantics, and whether rich/file clipboard formats remain deferred.

### Legal and product proof

- Produce a dependency license matrix and identify source-offer obligations.
- Freeze `ServiceMode`/GApps import/certified-provider and `RootMode` policy, F-Droid/microG packaging/privacy, F-Droid install privilege level, Play Integrity/SafetyNet legacy/Widevine/banking gates, codecs/DRM scope, and user-supplied ARM-on-x64 provider policy.
- Freeze Desktop/TV instance separation, TV launcher source, media codec scope, supported display profiles, and staged Desktop-before-TV release order.
- Freeze minimum host OS versions and first-release feature tier.

## Deliverables

- Architecture decision records for VM core, guest baseline, graphics stack, IPC, packaging, and licensing boundaries.
- Reproducible Desktop and TV proof builds for all three host/guest architecture pairs.
- Baseline boot, graphics, memory, and latency measurements.
- Risk register with owner and resolution date for every red item.
- Exact pinned upstream versions and update policy.
- WSA kernel reuse report with an explicit `import`, `reimplement`, or `drop` decision for every candidate change.
- Waydroid and LineageOS reuse reports with pinned revisions and per-change provenance/disposition.
- GrapheneOS hardening report with pinned revisions, per-change provenance, AOSP-equivalent check, threat coverage, compatibility/performance evidence, and explicit exclusion list.
- Zero-telemetry specification, first-party egress manifest, clean packet-capture evidence, local diagnostics/redaction design, and third-party/host-OS scope statement.
- Windows and macOS process-sandbox matrix with tokens/entitlements, IPC ACLs, file/network/device access, resource limits, crash boundary, and measured overhead.
- Android TV feasibility report covering launcher/SystemUI, D-pad focus, per-mode app discovery, media, 1080p/4K presentation, accessibility, and DRM limitations.
- Service-mode/GApps and KernelSU/Magisk threat, legal, build, recovery, compatibility, and release-gate reports; certified-provider partner/attestation/Play Integrity/SafetyNet legacy/Widevine/banking feasibility disposition; Houdini/`libndk_translation` legal and technical reports without redistributing proprietary payloads.
- Input architecture ADR, event/latency traces, shortcut precedence table, cursor-ownership proof, controller mapping format, privacy threat model, and representative hardware results.
- Clipboard/notification architecture ADR, normalized schemas, permission/consent UX, platform capability matrix, loop/reconciliation model, security threat model, latency/energy results, and content/action fuzz plan.
- Native-shell ADR covering real host window/process ownership, Windows AppUserModelID/activation, macOS Dock-identity disposition, platform command/lifecycle mapping, accessibility, and the boundary between native chrome and Android app content.
- Per-app presentation ADR covering profile schema/precedence, selected task/display architecture, Auto/Phone/Tablet configuration mapping, compatibility/letterbox behavior, transactional apply/rollback, benchmarks, and test corpus.

## Exit criteria

- Native acceleration is proven on all committed targets, or the affected target is explicitly removed from the first release.
- ANGLE and Gfxstream render on each committed host architecture.
- One Android guest design and one host/guest protocol direction are approved.
- Desktop and TV products boot on all committed architecture pairs, use separate userdata, and share the approved system/vendor/runtime foundation.
- No unresolved license blocker exists for the planned public preview.
- `NoApp`, `microG`, and user-imported `GApps` pass provisioning, isolation, update, reset, and rollback tests; no package outside the microG allowlist can spoof a signature, and no user-imported proprietary GApps payload is present in public source, CI, or ordinary release artifacts. Certified partner payloads use a separate access-controlled licensed supply chain.
- A certified `GApps + Off` target proceeds only with written partner approval and a feasible legitimate identity/attestation, Play Integrity, licensed Widevine, and banking test path; failed targets remain explicitly uncertified and make no protected-app claim.
- `Off`, KernelSU, and Magisk pass their eligible target gates; root cannot grant a host capability, weaken AVB/SELinux, or leave a rooted artifact/state behind after selecting `Off`.
- Native Bridge remains `Off` by default; each exposed provider passes legal review, import validation, self-tests, and the agreed ARM-app threshold.
- Architecture review approves the shared-runtime approach.
- The input proof has one authoritative backend per device class, no duplicate IME characters, safe pointer-capture recovery, Android-standard gamepad mappings, bounded 8,000 Hz behavior, and measured latency breakdowns.
- Clipboard proof is loop-free and sensitive-by-default; notification proof updates/removes/actions/replies safely without broad host notification access or stale authority.
- Open/user-import TV scope makes no Google TV, Widevine L1, HDCP, or commercial streaming compatibility claim; any `CertifiedPartner + Off` TV claim is separately licensed, target-specific, and gated by its proven secure-media/service matrix.
- Ten simultaneous Desktop tasks use real native host windows without fake title bars, cross-task surface/configuration leaks, or VM shutdown on ordinary window close.
- Auto/Phone/Tablet proofs select correct Android resource/configuration classes, survive profile/monitor/orientation changes, and roll back a failing app without affecting other tasks.
- The macOS per-package Dock identity result is explicitly `ship`, `defer`, or `drop`; no release claim exceeds the proven signed/notarized behavior.
- Production-profile dependency scans and packet captures find no telemetry client, automatic crash upload, stable tracking identifier, or undocumented first-party destination.
- The approved GrapheneOS patch queue is bounded and provenance-tracked; AOSP remains canonical, and no Pixel-specific component, GrapheneOS service dependency, branding, or equivalence claim enters the product.
- Guest SELinux/seccomp/per-app network controls and host broker isolation fail closed in abuse tests while meeting the initial hardening overhead and broker-latency budgets.
