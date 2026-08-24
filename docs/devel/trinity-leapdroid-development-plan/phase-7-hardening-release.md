# Phase 7 — Hardening, Beta, and General Availability

Duration: 12–16 weeks after feature complete  
Purpose: Convert feature completeness into reliable supported releases.

## Compatibility program

- Maintain a 100-app blocking suite and at least a 500-app beta matrix.
- Cover Java/Kotlin apps, NDK apps, GLES/Vulkan games, media, browsers, office/productivity, camera/mic, multi-window, accessibility, banking, and apps with unusual lifecycle behavior.
- Record package/version, ABI, Android API target, graphics backend, host hardware, result, workaround, and owner.
- Use app-specific overrides only when documented, bounded, versioned, and removable.
- Run the matrix under `ServiceMode=NoApp|microG|GApps` and every eligible `RootMode=Off|KernelSU|Magisk`; on Windows x64 also cover Native Bridge `Off`, Houdini, and `libndk_translation` wherever legally enabled.
- Tag every result with package version, APK ABI set, service/root mode and artifact provenance, APIs used, Native Bridge provider/version/hash, translated ABI, and whether failure reproduces with `NoApp + Off` and a native-ABI APK.
- Maintain a separate 50-app blocking/250-app TV matrix tagged with Leanback support, focus behavior, input device, display profile, codec/backend, subtitles, audio layout, service/root mode, and DRM requirement.
- Add input result dimensions for host layout/IME, physical-key versus composed-text path, mouse polling rate, pointer mode/icon, controller model/connection/player slot, haptics, host OS build, and mapping version.
- Add clipboard/notification dimensions for direction/format/size bucket, active instance, sensitivity/visibility class, host permission/DND, package/channel, action class, guest lifecycle state, bridge/protocol version, and platform capability downgrade—never content.
- Add native-shell and presentation dimensions for AppWindow/NSWindow behavior, activation source, host monitor/scale, form-factor mode, effective dp/size class/density/orientation, adaptive/fixed/letterbox state, task recreation, profile version, and last-known-good rollback.

## Performance program

- Establish reference x64 Intel/AMD, Windows ARM64 Qualcomm, and Apple Silicon machines.
- Track cold boot, resume, app launch, frame time, jank, input latency, audio latency, memory, CPU, GPU, disk amplification, battery/energy, and thermal behavior.
- Compare compatibility backend, Trinity projection, and SwiftShader under identical conditions.
- Track TV 1080p/4K UI frame pacing, D-pad latency, decoder throughput, dropped frames, A/V drift, seek/flush latency, multichannel audio, energy, thermal state, and playback sleep behavior.
- Split input latency into host capture, normalization/queue, virtio transport, Android dispatch, and app/frame presentation. Test 1,000 Hz continuously and characterize bounded 8,000 Hz mouse behavior without losing ordered buttons/wheels.
- Track clipboard observation/transport/set/echo latency and macOS observer energy; track notification post/update/remove reconciliation plus warm/cold action/reply latency and flood-coalescing cost.
- Track hardening A/B deltas for CPU, settled memory, cold/warm launch, UI jank, broker IPC, helper wakeups, energy, TV playback, and input latency. No mitigation is globally disabled to make a benchmark pass.
- Track interactive native resize to guest configuration acknowledgement and first correctly sized frame, including compact/medium/expanded threshold crossings and fixed Phone/Tablet letterbox paths.
- Block regressions beyond agreed budgets unless explicitly waived with owner and expiry.

## Reliability program

- Multi-day soak with app churn, install/uninstall, suspend/resume, display changes, network transitions, GPU resets, and storage pressure.
- Crash-loop detection, safe mode, renderer fallback, disk repair, snapshot invalidation, and diagnostic export.
- Upgrade testing from every public beta and supported stable version.
- Validate host OS feature updates and GPU driver updates before broad rollout.
- Run multi-day TV playback/navigation soak with resolution and refresh switches, controller reconnects, decoder/GPU resets, subtitle changes, network loss, suspend/resume, and low-memory pressure.
- Run multi-day input soak with focus/task churn, repeated capture/release, IME switches, key holds during suspend/reset, 1–8 controller hot-plug/reorder, active haptics during failure, and input/renderer/guest crash injection.
- Run multi-day clipboard/notification soak with alternating writes, multi-instance focus changes, rich payloads, progress floods, actions during guest lifecycle transitions, permission/DND changes, user switches, snapshot/update/rollback, and broker crash injection.
- Run production network captures through the full soak and failure matrix; only destinations/functions in the reviewed egress manifest may appear, and diagnostics/crashes never upload automatically.
- Run multi-day native-window/profile soak with task churn, Start/Spotlight activation, monitor/scale/Spaces/virtual-desktop moves, minimize/fullscreen/close, Auto threshold crossings, Phone/Tablet switches, activity recreation, app/guest/host updates, and injected crash/rollback.

## Release stages

### Developer Preview

- ADB, sideloading, basic accelerated graphics, single/multiple native app windows, physical keyboard/mouse, host IME, cursor capture/sync, standard controllers, text clipboard, basic notification open/update/remove, audio, lifecycle, and known-limit documentation.
- Native app settings include Auto/Phone/Tablet and reset; Windows exposes per-app Start/taskbar identity, while macOS exposes native NSWindows and Spotlight activation with the Phase 0 Dock-identity result documented.
- `NoApp + Off` is the preview baseline; `microG`, user-supplied GApps, KernelSU/Magisk, and user-supplied ARM-on-x64 providers are independently gated preview features with published limits. No promise of Google certification, GMS completeness, Play Integrity, DRM, banking, anti-cheat, camera, or snapshot reliability.
- TV preview follows Desktop preview and includes an open 10-foot launcher, D-pad/gamepad, 1080p SDR, initial 4K/media acceleration, and explicit codec/DRM limitations.

### Public Beta

- Signed auto-update/rollback, broader host integration, 500-app matrix, local crash diagnosis with manual redacted export, privacy controls, compatibility database, and support intake.
- Feature flags allow rapid backend rollback without replacing userdata.
- Zero telemetry, local diagnostics/manual export, first-party egress disclosure, per-app Network/Sensors controls, and process sandbox status are release features, not experimental flags.
- `NoApp` and `microG` flows are supported first; GApps and each root/Native Bridge provider-target combination graduate individually only after legal, build/provenance, security, recovery, update, telemetry, and compatibility thresholds pass.
- TV beta requires remote-only provisioning/recovery, 250-app results, stable host decode fallback, edition-safe updates, and published per-host media/input matrices.

### General Availability

- Security and licensing gates passed.
- Phase 6A zero-telemetry, sandbox, independent review, and hardening performance gates passed on every Desktop and TV SKU.
- Phase 4B native-shell, app identity/activation, Auto/Phone/Tablet isolation, compatibility/rollback, accessibility, and resize-latency gates passed on every Desktop SKU.
- Performance and crash-free targets met for four consecutive release candidates.
- CTS/VTS/graphics thresholds met and published internally.
- Support policy, update cadence, deprecation policy, recovery tooling, and incident response are operational.
- Each shipping service/root mode has independent compatibility and risk results; every production GApps/root/Native Bridge provider has upgrade, disable, safe-boot, rollback, removal, and explicit end-of-support procedures. A failed advanced provider never blocks or weakens `NoApp + Off`.
- Desktop GA may precede TV GA. TV reaches GA only after four consecutive candidates meet TV UI, media, accessibility, reliability, licensing, and support gates on all committed targets.

## Exit criteria

- No release-blocking data-loss, host-security, update, or isolation defect.
- Crash-free, latency, boot/resume, and memory targets are met on the reference matrix.
- Keyboard/IME correctness, cursor recovery, focus isolation, high-rate pointer ordering, eight-controller reconnect, and haptics teardown pass on every supported host architecture.
- Clipboard loop/sensitivity/isolation and notification update/remove/action/reply/dismiss/privacy/restart gates pass on every supported host architecture and edition.
- Product can recover from GPU failure, broken snapshot, failed update, low disk, and guest boot failure without manual file deletion.
- Windows x64, Windows ARM64, and macOS ARM64 releases are independently supportable but protocol-compatible.
- Clean-install and upgraded production artifacts contain no telemetry client/automatic uploader; network captures have no undocumented first-party destination, and every local diagnostic export requires explicit user action.
- Every exposed `ServiceMode`/`RootMode` passes its absence, isolation, provenance/import/build, privilege, recovery, update/rollback, packet-capture, and compatibility matrix; proprietary GApps payloads and unreviewed root modules remain absent from source, CI, and releases.
- Android tasks behave as native host windows without fake chrome; per-app presentation profiles survive app/guest/host updates and cannot alter another task, Android user/profile, instance, or TV product.
