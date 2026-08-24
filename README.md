# Trinity Emulator

Trinity is being rebuilt as a WSA-like Android subsystem for Windows 11. The new generation targets Windows x64 and Windows ARM64, with separate Android Desktop and Android TV editions sharing one modern runtime and host/guest protocol with LeapDroid for macOS.

> **Current status:** architecture and repository reset. The old QEMU 5.0/Android-x86 implementation is preserved on the [`legacy`](https://github.com/LamPPKK/TrinityEmulator/tree/legacy) branch. The `main` branch is the clean foundation for the native rewrite and is not yet runnable.

The canonical implementation plan is in [`docs/devel/trinity-leapdroid-development-plan`](docs/devel/trinity-leapdroid-development-plan/README.md). Start with the [overall plan](docs/devel/trinity-leapdroid-development-plan/plan.md), then use the phase documents as release gates.

## Product targets

| Product | Host | Guest ABI | Virtualization | Graphics baseline |
|---|---|---|---|---|
| Trinity Desktop | Windows 11 x64 | x86_64 | WHPX | Gfxstream + ANGLE/D3D11; SwiftShader fallback |
| Trinity TV | Windows 11 x64 | x86_64 | WHPX | Gfxstream + ANGLE/D3D11; SwiftShader fallback |
| Trinity Desktop | Windows 11 ARM64 | arm64 | WHPX ARM64, capability-gated | Gfxstream + ANGLE; SwiftShader fallback |
| Trinity TV | Windows 11 ARM64 | arm64 | WHPX ARM64, capability-gated | Gfxstream + ANGLE; SwiftShader fallback |

## Development checklist

Checkboxes describe implementation and release status on `main`. An item is checked only after its phase exit criteria and tests pass.

### Repository and runtime foundation

- [x] Preserve the complete previous implementation and history on `legacy`.
- [x] Publish the joint Trinity/LeapDroid architecture and phased development plan.
- [ ] Complete Phase 0 feasibility gates and record architecture decisions.
- [ ] Fix repository, process, IPC, and licensing boundaries for the shared runtime and native hosts.
- [ ] Pin a current supported QEMU release and maintain a small, reviewable patch queue.
- [ ] Implement versioned control, presentation, input, clipboard, notification, surface, and bulk-data protocols.
- [ ] Add reproducible native Windows x64 and ARM64 build toolchains and CI.
- [ ] Add signed development packages, SBOMs, provenance, license reports, and rollback-capable updates.

### Android guest and app compatibility

- [ ] Pin a current stable AOSP/GKI baseline with x86_64 and arm64 products.
- [ ] Build separate Desktop and Android TV images over a shared system/vendor base.
- [ ] Implement boot, stop, reset, suspend/resume, snapshots, storage quotas, and multi-instance lifecycle.
- [ ] Support APK install/uninstall, package discovery, host launcher registration, deep links, and opt-in ADB.
- [ ] Implement exactly three immutable service modes: `NoApp` (clean AOSP, default), `microG` (verified F-Droid + approved microG), and `GApps` (advanced user import).
- [ ] Keep user-imported GApps, Google Play Store/Services, and other proprietary Google system packages out of public source, CI, and ordinary release artifacts; never search for or download them, and accept only a compatible package explicitly selected by the user. A separate `CertifiedPartner` artifact requires exact written distribution rights.
- [ ] Build the user-import GApps path as a sandboxed, transactional, payload-free provider with validation, sealed read-only add-on storage, rollback, mode isolation, and a permanent uncertified/best-effort label.
- [ ] Add a separate `GAppsProviderClass=CertifiedPartner` gate for `RootMode=Off`; ship the certified label/SKU only with written GMS/Play Protect approval, legitimate attestation provisioning, official Play Integrity verification, applicable legacy SafetyNet results, licensed Widevine security level, and a passing regional banking-app blocking matrix.
- [ ] Keep user-imported GApps permanently uncertified; if certification, attestation, DRM licensing/security level, or banking gates fail on a target, withhold the certified SKU instead of spoofing or bypassing the verifier.
- [ ] Restrict microG signature spoofing to pinned identities and make device registration, Cloud Messaging, location, and network access independently controlled.
- [ ] Implement independent `RootMode=Off|KernelSU|Magisk`; default to `Off`, build rooted Developer artifacts reproducibly from pinned source, and keep providers mutually exclusive.
- [ ] Add default-deny root grants, authenticated management, module safe mode, host-owned recovery, update/rollback, and clean-absence tests; provide no concealment or attestation/DRM/banking/anti-cheat bypass features.
- [ ] Target-gate KernelSU on Windows x64 and do not ship it when the pinned version requires disabling syscall hardening; keep Magisk Zygisk off by default.
- [ ] Let users import compatible local KernelSU/Magisk module archives through a provider-scoped, sandboxed file picker with validation, risk acknowledgement, snapshot/stopped-instance requirement, enable/disable/remove controls, and host-owned safe-mode recovery; do not bundle or download module payloads.
- [ ] Detect APK ABI before install and report `native`, `translatable`, or `unsupported`.
- [ ] On Windows x64, support user-imported Houdini or `libndk_translation` providers through two payload-free adapters; do not bundle or download proprietary binaries.
- [ ] Maintain CTS, VTS, compatibility, upgrade, and real-application test matrices for every guest SKU.

### Graphics and native Windows experience

- [ ] Deliver compatibility-first accelerated GLES through Gfxstream and ANGLE.
- [ ] Add Vulkan support after conformance and isolation gates pass.
- [ ] Keep SwiftShader as a tested software fallback.
- [ ] Reimplement Trinity graphics projection as an optional backend only after the compatibility renderer is stable.
- [ ] Present every Android task in a real top-level WinUI 3/C++/WinRT `AppWindow`/HWND.
- [ ] Integrate native title bars, snap layouts, Alt+Tab, taskbar identity, jump/launcher activation, DPI, multi-monitor, fullscreen, and PiP behavior.
- [ ] Add per-app `Auto`, `Phone`, and `Tablet` presentation profiles.
- [ ] Add per-app orientation, adaptive/fixed form factor, size, density, scale, aspect/letterbox, placement, fullscreen/PiP, and safe reset controls.
- [ ] Apply Android configuration per task/display without changing unrelated applications or the whole guest.

### Input, host integration, and Android TV

- [ ] Implement Windows Raw Input keyboard/mouse handling and host IME composition.
- [ ] Implement absolute pointer, relative capture, cursor synchronization, focus recovery, touch, and stylus paths.
- [ ] Implement GameInput controller hot-plug, multiple players, stable mappings, battery state, and haptics.
- [ ] Synchronize clipboard text, sanitized HTML, and bounded images bidirectionally with loop and sensitive-content protection.
- [ ] Mirror Android notification post/update/remove/open/action/reply/dismiss lifecycle into Windows notifications with per-app/channel policy.
- [ ] Broker selected host folders, drag/drop, audio, microphone, camera, location, and host link/open-with flows through explicit permissions.
- [ ] Deliver a 10-foot Android TV launcher/SystemUI, D-pad/gamepad/media-key navigation, push-to-talk, 1080p and 4K SDR profiles, and TV-aware app discovery.
- [ ] Keep TV clipboard off by default and make notification previews opt-in and allowlisted.

### Privacy, sandboxing, and release quality

- [ ] Enforce a production zero-telemetry profile: no analytics, automatic crash upload, remote logging, experiments, stable tracking ID, or hidden metrics endpoint.
- [ ] Provide bounded local diagnostics with redaction preview and explicit manual export only.
- [ ] Enforce a checked-in first-party egress manifest and packet-capture tests for startup, idle, crash, update, suspend/resume, and app launch.
- [ ] Preserve Android per-UID isolation and harden SELinux, seccomp, spawning, memory safety, dynamic-code policy, and per-app network/sensor controls using reviewed GrapheneOS patterns.
- [ ] Isolate the UI, VM controller, renderer, media, updater, diagnostics, Native Bridge inspector, and integration brokers with least privilege, AppContainer, Job Objects, and authenticated capability-scoped IPC.
- [ ] Audit WSA Linux kernel, Waydroid guest/HAL, LineageOS microG/TV, and GrapheneOS security changes with pinned provenance and explicit `import`, `reimplement`, or `drop` decisions.
- [ ] Meet boot, resume, frame pacing, input latency, memory, CPU, energy, notification, and TV responsiveness budgets.
- [ ] Pass security review, fuzzing, long-run stability, upgrade/rollback, accessibility, localization, and release gates on all four Trinity SKUs.

## Delivery order

1. [Phase 0 — decision and feasibility gates](docs/devel/trinity-leapdroid-development-plan/phase-0-decision-gates.md)
2. [Phase 1 — shared runtime foundation](docs/devel/trinity-leapdroid-development-plan/phase-1-runtime-foundation.md)
3. [Phase 2 — Android guest](docs/devel/trinity-leapdroid-development-plan/phase-2-android-guest.md), [Phase 2A — Android TV](docs/devel/trinity-leapdroid-development-plan/phase-2a-android-tv.md), [Phase 2B — service modes/GApps](docs/devel/trinity-leapdroid-development-plan/phase-2b-service-modes-gapps.md), and [Phase 2C — root providers](docs/devel/trinity-leapdroid-development-plan/phase-2c-root-providers.md)
4. [Phase 3 — graphics](docs/devel/trinity-leapdroid-development-plan/phase-3-graphics.md)
5. [Phase 4 — seamless windows](docs/devel/trinity-leapdroid-development-plan/phase-4-seamless-windows.md), [input](docs/devel/trinity-leapdroid-development-plan/phase-4a-input-cursor-controller.md), and [native app/display profiles](docs/devel/trinity-leapdroid-development-plan/phase-4b-native-app-experience-display-profiles.md)
6. [Phase 5 — host integration](docs/devel/trinity-leapdroid-development-plan/phase-5-host-integration.md) and [clipboard/notifications](docs/devel/trinity-leapdroid-development-plan/phase-5a-clipboard-notifications.md)
7. [Phase 6 — security, updates, and compliance](docs/devel/trinity-leapdroid-development-plan/phase-6-security-update-compliance.md) plus [zero telemetry and sandboxing](docs/devel/trinity-leapdroid-development-plan/phase-6a-zero-telemetry-sandbox.md)
8. [Phase 7 — hardening and release](docs/devel/trinity-leapdroid-development-plan/phase-7-hardening-release.md)

## Legacy source and licensing

Use the `legacy` branch to inspect or reproduce the previous Trinity artifact:

```shell
git clone --branch legacy --recurse-submodules https://github.com/LamPPKK/TrinityEmulator.git
```

Do not forward-port the old tree wholesale. Import only justified behavior, tests, or small reviewed patches with provenance. QEMU-derived components remain subject to GPLv2 and their original notices; new components must keep explicit, compatible license boundaries. See `LICENSE`, `COPYING`, `COPYING.LIB`, and the plan's licensing section.
