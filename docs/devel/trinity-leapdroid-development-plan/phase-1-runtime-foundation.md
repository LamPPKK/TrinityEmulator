# Phase 1 — Shared Runtime Foundation

Duration: 8–12 weeks after Phase 0  
Purpose: Establish reproducible native builds and a maintainable VM/device base.

## Runtime work

- Import the pinned QEMU release with an upstream-first patch policy.
- Port only the required Trinity device concepts; do not forward-port unrelated QEMU 5.0 changes.
- Introduce versioned control, surface, input, and bulk-data devices with reset, cancellation, quota, and migration semantics.
- Add a versioned presentation protocol for task/window lifecycle, app identity/activation, host-native window state, requested and applied Android configuration, Auto/Phone/Tablet profile, size-compatibility state, restart acknowledgement, and bounded failure reasons.
- Implement standard virtio-input devices for keyboard, absolute pointer, relative pointer, and hot-pluggable controllers. Keep high-rate evdev events on virtqueues and reserve the control protocol for IME, cursor, capture, device metadata, player assignment, and haptics.
- Add `host/common/input` normalization, focus/capture ownership, monotonic timestamp conversion, bounded queues/coalescing, release-all recovery, mapping schemas, and latency trace points with platform backends behind narrow interfaces.
- Add `host/common/presentation` with the per-app profile store/resolver, task/window ownership, client-area-to-dp conversion, placement restoration, profile migration, last-known-good rollback, and host-neutral command/lifecycle conformance fixtures.
- Add versioned clipboard and notification schemas plus host-neutral brokers for revision/acknowledgement, active-instance routing, permission state, redaction, action capability expiry, reconciliation, rate limiting, and bounded bulk assets.
- Put HTML/image/icon decoding and normalization in sandboxed helpers; neither the VM controller nor UI process may parse guest rich content or retain Android action authority.
- Replace manual global/thread state in old `direct-express` code with documented ownership and synchronization boundaries.
- Run the native control center, per-task AppShell/presentation brokers, VM controller, renderer, media, update, diagnostics, Native Bridge inspection, and every host-integration broker as separate processes according to the Phase 6A privilege map. IPC is authenticated, capability-scoped, bounded, cancellable, and rejects cross-user/instance handles.
- Add bounded structured logs, crash artifacts, local correlation IDs, and opt-in trace points across guest, VM, renderer, and UI. Production builds contain no network sink, analytics/crash SDK, automatic uploader, stable tracking identifier, or remote experimentation path; users preview/redact and manually export diagnostics.
- Add a checked-in first-party egress manifest and production-build rule that fails on an undeclared network-capable component, telemetry dependency, or remote log sink.

## Native build work

- Windows: CMake/MSBuild/Ninja toolchains for x64 and ARM64, C++20, C++/WinRT, WinUI 3, signed development MSIX bundles.
- macOS: Xcode/Swift Package Manager/CMake toolchains for arm64, Swift/SwiftUI/AppKit, Objective-C++ adapters, hardened runtime and entitlements.
- Shared dependencies: pin ANGLE, Gfxstream, SwiftShader, protobuf/schema tools, tracing, and test frameworks through reproducible manifests.
- Build symbols and source maps separately from release payloads.
- Produce Desktop and TV host packaging from the same native binaries; edition-specific assets and guest manifests must not fork the VM/runtime build.

## CI and supply chain

- Hermetic or containerized Linux AOSP builds.
- Native Windows x64 and ARM64 build agents and Apple Silicon macOS agents.
- Build cache keyed by exact toolchain and dependency hashes.
- SBOM, license report, vulnerability scan, artifact signing, provenance, and reproducibility checks.
- Presubmit unit tests, formatting, static analysis, sanitizers where supported, and protocol compatibility tests.
- Static dependency/endpoint scans, SBOM policy, sandbox entitlement/token checks, and packet-capture jobs for production startup/idle/crash/update paths.

## Existing code touchpoints

- Trinity reference modules: `hw/direct-express`, `hw/express-gpu`, `include/direct-express`, `include/express-gpu`, `target/i386/cpu.c`, `hw/Kconfig`, and the Windows CI workflow.
- LeapGL reference modules: `LeapGL/host/libs`, `LeapGL/shared`, `LeapGL/host/tools/emugen`, tests, and `LeapGL/Android.mk`.
- Replace Trinity `express_gpu_render.c` GLFW/QEMU input callbacks and fixed replay arrays; do not carry input polling inside the render loop or the existing key-plus-mouse recording behavior into the product. Extract only mapping/test cases that remain correct.
- Treat LeapDroid `tests/event_injector` emulator-console key/touch injection as a legacy test fixture. It is not the macOS input backend and is not used for production host/guest transport.
- Mark old renderers as legacy/reference and prevent accidental linkage into the new shipping target.
- WSA reference inputs: public feature documentation and the separately licensed `WSA-Linux-Kernel` only. Track every reused kernel commit in a provenance manifest; do not add WSA runtime binaries or extracted package content.
- Waydroid reference inputs: runtime, hardware, and vendor repositories. Keep Linux LXC/Wayland host code out of the shipping runtime; track any guest/HAL import with its source revision and license.
- LineageOS reference inputs: manifest/framework microG changes, ATV device tree, TvSystemUI, overlays, key layouts, and device-support requirements. Maintain these as a small rebased patch queue over AOSP, not a full platform fork.
- Create payload-free build/test targets for `NativeBridgeProvider` descriptors so CI never requires Houdini or `libndk_translation` binaries.

## Exit criteria

- Clean native builds for Windows x64, Windows ARM64, and macOS ARM64.
- A generic Linux guest boots with accelerated CPUs and required virtio devices on all targets.
- VM start/stop/reset survives 1,000 automated cycles without leaked processes or corrupted disks.
- Protocol negotiation rejects incompatible versions safely.
- Presentation negotiation rejects unsupported profile/configuration fields safely; old host/new guest and new host/old guest preserve Auto mode and last known-good placement without reconfiguring unrelated tasks.
- Input devices enumerate/reset/hot-unplug cleanly; malformed capabilities and queue pressure fail safely; focus loss always clears held input, capture, and haptics.
- Clipboard/notification protocol negotiation, revision reset, broker crash, malformed payload, oversized asset, stale capability, and old-host/new-guest tests fail closed without cross-instance state.
- Every host helper launches with the approved least-privilege token/entitlements, Job Object or pressure/resource policy, network rule, and IPC ACL; killing or compromising one helper cannot grant another helper's capability.
- Production artifacts contain no automatic telemetry/upload path, and the initial offline/online packet-capture suite has no undocumented first-party destination.
- Signed development packages install, update, and uninstall cleanly.
- CI produces signed Desktop/TV guest manifests for x86_64 and arm64 and proves that both editions negotiate the same host protocol.
