# Phase 2 — Modern Android Guest Platform

Duration: 10–14 weeks after the runtime foundation  
Purpose: Replace Android-x86 9 and missing LeapDroid guest components with a shared current AOSP platform.

## AOSP products

- Pin Android 17/API 37 or the approved stable release tag.
- Create `subsystem_desktop_x86_64`, `subsystem_desktop_arm64`, `subsystem_tv_x86_64`, and `subsystem_tv_arm64` products with common framework/vendor configuration.
- Separate `system`, `system_ext`, `product`, `vendor`, `boot`, `init_boot`, and userdata concerns so annual Android upgrades do not require rewriting desktop integration.
- Put edition differences in product overlays/packages/feature declarations. Desktop and TV use separate userdata and update manifests; never convert one edition's userdata in place.
- Use one shared protocol and base compatibility database with edition-specific Desktop and TV result dimensions.

## Kernel and virtual hardware

- Maintain GKI-aligned x86_64 and arm64 kernels with virtio block, network, input, sound, balloon, serial/vsock, shared-memory, and graphics support.
- Keep kernel patches small, reviewed, and continuously rebased.
- Enable AVB, dm-verity, SELinux enforcing, secure debug policy, and production ADB restrictions.
- Preserve AOSP per-UID app isolation, per-app SELinux domains, seccomp-bpf, namespaces, Binder permission checks, scoped storage, and Android user/profile separation. Add GrapheneOS-derived changes only through the provenance/performance-gated patch queue.

## Guest services and HALs

- Lifecycle/power agent for ready, idle, suspend preparation, resume, shutdown, health, and watchdog signals.
- App registry agent for packages, launchable activities, icons, labels, install state, and shortcuts.
- Task/surface agent for per-app window metadata, surface lifecycle, host activation, and applied-configuration acknowledgement.
- Presentation agent that applies versioned configuration only to the selected task/display container: client width/height dp, smallest width, density, orientation, windowing mode, form-factor lock, and size-compatibility/letterbox state. It never changes the guest-wide device model, fingerprint, certification, or another task's configuration.
- Platform-signed, signature-permission and SELinux-confined clipboard agent with remote/sensitive metadata, current-clip snapshotting, revision acknowledgements, per-user isolation, and no broad third-party background-read permission.
- User-consented `NotificationListenerService` plus action broker that snapshots/reconciles active notifications, normalizes safe content, retains all `PendingIntent` objects in Android, issues expiring revision-bound opaque capabilities, and returns validated actions/direct replies.
- Host-files DocumentsProvider, link, drag/drop, and share agents remain separate from clipboard; file/content URIs are never dereferenced automatically by the clipboard bridge.
- Virtual audio, camera, microphone, location, sensor, input, and display HAL integration.
- Configure Linux/Android virtio-input for distinct keyboard, absolute pointer, relative pointer, and controller devices; maintain reviewed `.kl`, `.kcm`, axis, and controller mappings for Desktop and TV without bypassing EventHub/InputReader/InputDispatcher.
- Add a signed `HostInputMethod` service for host pre-edit/commit/cancel transactions against the focused `InputConnection`, plus a cursor agent that exports resolved Android pointer icon, custom bitmap/hotspot, visibility, and pointer-capture state.
- Add a narrowly scoped haptics bridge from Android vibrator/controller output to the authenticated host broker and publish only supported device/battery metadata back to guest/system UI.
- Permission broker UI hooks that preserve Android permission semantics while respecting host permission decisions.
- Add fail-closed per-app Network and Sensors controls inspired by GrapheneOS. Network denial covers direct/indirect app access including localhost as far as the guest enforcement design permits; compatibility behavior and profile scoping are tested explicitly.
- Forbid dynamic code loading in first-party base services. Evaluate targeted `hardened_malloc`, zero-on-free, MTE on capable ARM hosts, and other allocator/runtime hardening first in parsers, media, graphics, brokers, and other privileged native processes; user-app exceptions remain explicit and versioned.
- Add a local-only log/crash viewer backed by bounded ring buffers and redacted artifacts. No guest component automatically transmits diagnostics, usage, identifiers, or crash data.
- Edition service selects Desktop task/surface export or TV single-display presentation before SystemUI starts; this is immutable for the lifetime of an instance.
- Base Desktop integration on Android 17 per-display desktop windowing where it passes the Phase 0 architecture gate. Keep WindowManager/TaskDisplayArea authoritative and use Android configuration/lifecycle dispatch rather than scaling a fixed framebuffer behind the app's back.

## Application service modes

- Provision exactly one immutable `ServiceMode` per instance: `NoApp`, `microG`, or `GApps`; default to `NoApp`. Every mode supports the host APK installer and optional developer ADB.
- `NoApp` contains no optional app store or compatibility-service bundle. Add negative artifact/package/permission/egress tests so F-Droid, microG, GApps, Google accounts/tokens, and provider state cannot leak into it.
- `microG` contains a pinned official F-Droid Client and approved microG packages with verified signing/reproducibility provenance. Enable only F-Droid's signed official repository by default; keep both apps sandboxed, restrict signature spoofing by package and certificate, and make device registration, Cloud Messaging, location, and network access independently opt-in.
- `GApps` consumes either a compatible package explicitly selected by the user through Phase 2B's payload-free transactional provider or an exact signed `CertifiedPartner` artifact delivered through its approved licensed supply chain. Never mix it with microG or infer certification from package presence; user import remains uncertified, while a certified artifact is target/build/root-mode specific.
- Switching modes creates or clones a new instance; never carry system packages, accounts, credentials, tokens, or privileged state between modes. Use normal Package Installer confirmation initially; any privileged F-Droid helper requires separate signer/domain/IPC/audit/update approval.

## Developer root providers

- Provision an independent immutable `RootMode=Off|KernelSU|Magisk`; default to `Off`, permit only one provider, and keep rooted images in Advanced/Developer channels.
- Use the reproducible, separately signed boot/kernel artifacts, default-deny per-app grants, safe mode, module recovery, updates, rollback, and target gates in Phase 2C. Never disable AVB/dm-verity or SELinux, grant host authority, or ship concealment/attestation/DRM/banking/anti-cheat bypass features.

## Windows x64 Native Bridge

- Implement the AOSP-facing `NativeBridgeProvider` contract and provider descriptors for `Off`, user-supplied Houdini, and user-supplied `libndk_translation`.
- Keep provider payloads outside system images. At guest start, mount the selected validated version read-only and configure ART, linker namespaces, ABI lists, and package extraction consistently.
- Advertise ARM ABIs only after provider health checks succeed. Detect APK ABI and give an actionable compatibility error instead of silently installing an unlaunchable package.
- Provider changes require full guest stop/start, provider-specific ART/native cache invalidation, self-test, and rollback to `Off` on failure.
- Keep provider selection unsupported on native arm64 guests unless a future requirement proves otherwise.

## App compatibility

- Declare only supported hardware features and ABIs.
- Windows x64 advertises x86_64 by default. It adds only the ARM ABI(s) proven by the currently selected, legally user-supplied Native Bridge provider.
- Windows ARM64 and macOS ARM64 advertise arm64-v8a; add 32-bit ARM only if the guest and product policy support it.
- Keep an app-specific compatibility database for graphics backend selection, window policy, DPI, orientation, and known device-feature overrides.
- Track `android.software.leanback`, launcher category, touchscreen assumptions, portrait policy, D-pad focus, media-session behavior, and TV resolution/codec results for TV apps.
- Track Desktop manifest/runtime presentation signals: resizability, orientation, min width/height, min/max aspect, PiP, multi-activity behavior, adaptive-resource support, known compact/medium/expanded results, and last known-good Auto/Phone/Tablet profile.

## Exit criteria

- Desktop and TV products for both guest ABIs boot to their correct launchers on every host target.
- APK installation, PackageManager events, activity launch, ADB, networking, storage, audio, and input pass integration tests.
- Keyboard layouts/IME, pointer transforms/capture/cursor, controller hot-plug/mapping/haptics, focus-loss all-up, and TV navigation-owner tests pass without duplicate input paths.
- Clipboard text/HTML/PNG, sensitive/remote flags, loop prevention and user isolation pass; notification lifecycle, redaction, action/reply authentication, stale-token, restart and permission-revocation tests pass.
- SELinux is enforcing with no broad permissive domains.
- System images are deterministic, signed, and updateable independently from userdata.
- A defined CTS smoke plan passes on x86_64 and arm64.
- `NoApp`, `microG`, and user-imported `GApps` pass package-absence, provisioning, isolation, update, clone, reset, health-check, and rollback tests; F-Droid verifies indexes/APK hashes and microG opt-outs remain reversible.
- `Off`, KernelSU, and Magisk pass clean-absence/build/grant/module/safe-mode/recovery/update/rollback tests on every eligible target without weakening AVB, SELinux, or the VM/host boundary.
- Every enabled Native Bridge provider passes import, ABI, JNI, signal, linker, crash-containment, cache-switch, and rollback suites.
- Auto/Phone/Tablet changes are isolated per task/display, emit truthful `Configuration` values, handle or recreate the affected activity correctly, use Android size-compatibility for constrained apps, and roll back crash loops without changing other running apps.
- Per-app Network/Sensors controls, SELinux/seccomp policies, dynamic-code rules, profile isolation, local-only diagnostics, and targeted memory hardening pass negative tests, CTS/VTS gates, and the Phase 6A performance budget.
