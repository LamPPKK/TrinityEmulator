# Phase 2A — Android TV Editions

Duration: 16–22 weeks overlapping late Phase 2 and Phase 3  
Purpose: Deliver native TV product images and a 10-foot host experience for Trinity Windows x64/ARM64 and LeapDroid macOS ARM64 without forking the shared runtime.

TV inherits the same Phase 6A zero-telemetry production profile, AOSP/GrapheneOS-derived guest hardening, per-app Network/Sensors controls, host broker isolation, and local-only diagnostics as Desktop. A 10-foot UI or long-running playback mode is never a reason to widen privileges or add background analytics.

Desktop Auto/Phone/Tablet profiles do not change the TV product's `television` characteristic or single-display presentation. Sideloaded non-TV apps may receive a bounded TV compatibility canvas/orientation rule, but this is a separate compatibility setting and cannot turn a TV instance into a Desktop/phone/tablet product.

## Product and source baseline

- Derive `subsystem_tv_x86_64` and `subsystem_tv_arm64` from the pinned AOSP release and current `device/google/atv` product structure.
- Share kernel, vendor virtual hardware, Gfxstream, ANGLE, protocol, guest agents, service/root-mode frameworks, security policy, and update infrastructure with Desktop editions wherever compatibility permits.
- Give every TV instance its own product partition, edition manifest, instance identity, cache, and userdata. Do not support in-place conversion between Desktop and TV userdata.
- Audit LineageOS `android_device_lineage_atv`, `android_packages_apps_TvSystemUI`, overlays, key layouts, input fixes, and device-support requirements. Rebase only changes that improve the selected current AOSP TV baseline and pass license/security/CTS review; exclude GMS RROs, Netflix/vendor configuration, and physical-device files that do not apply to virtual hardware.
- Do not port Waydroid's LXC or Wayland host path. Use its full-UI session, image separation, guest lifecycle, audio/input HAL, and app-discovery behavior only as reference or narrowly licensed guest code.

## TV framework and 10-foot UX

- Set the Android television product characteristic and declare only the Leanback, input, display, audio, codec, camera, microphone, location, and networking features actually implemented.
- Integrate an open-source TV launcher, TvSystemUI, TV settings/provisioning, Leanback IME, screensaver/ambient behavior, media session controls, and accessibility services. Exclude Google TV launcher and proprietary Google TV content services.
- Ensure every first-run, permission, recovery, update, low-storage, crash, and network-error flow is readable and operable from ten feet with D-pad only.
- Define deterministic focus order, focus restoration after activity/window changes, long-press behavior, Back/Home semantics, screen-edge safe areas, and escape from apps that trap focus.
- Provide a host recovery overlay that can always stop an app, return Home, release captured input, change display mode, open diagnostics, or stop the TV instance independently of guest UI health.

## Host presentation

- Present TV as one Android display inside one native Windows or macOS window with windowed, borderless, and exclusive-like full-screen modes as supported by the host.
- Disable Desktop per-task host window export, host app shortcut mirroring, and host placement of individual Android tasks for TV instances.
- Support 720p, 1080p, 1440p, and 2160p display profiles; track logical density, aspect ratio, safe area, refresh rate, scaling, monitor move, full-screen transition, sleep/wake, and hot-plug behavior.
- Start with SDR sRGB/BT.709 behavior. Gate HDR behind proven guest-to-host color metadata, transfer function, tone mapping, 10-bit surfaces, capture behavior, and per-display fallback.
- Keep Android authoritative for TV task composition; the host owns the containing window, display selection, full-screen state, host overlays, and recovery shortcuts.

## Input and remote behavior

- Reuse the Phase 4A virtio-input/control stack; map keyboard arrows/Enter/Escape, D-pad, gamepad, media keys, supported Bluetooth/USB remotes visible to the host, and mouse fallback to Android events and maintained key layouts.
- Provide configurable host shortcuts for Home, Back, Recents if exposed, play/pause, seek, volume, mute, full-screen, input release, and diagnostics.
- Implement push-to-talk as an explicit host microphone session with visible permission/state; deliver audio only while authorized and active.
- Support at least eight controllers with stable player slots and reconnect behavior. Exactly one eligible remote/controller owns SystemUI navigation at a time while every connected controller remains visible to a foreground game.
- Hide the mouse cursor after inactivity, restore it on movement, and transfer navigation ownership only after an intentional pointer action. Host Home/Back/recovery and input-release commands remain available when an app traps focus or capture.
- Test remote-only provisioning and recovery; no required flow may depend on touch or a physical Android TV remote.

## Media and audio

- Implement a virtual Codec2/MediaCodec path that can export decode workloads and surfaces to a sandboxed host media process.
- Use Media Foundation and D3D11 video acceleration on Windows; use VideoToolbox and Metal-compatible surfaces on macOS. Keep a software decoder fallback and per-codec/device denylist.
- Begin with H.264 and open/unencumbered baseline formats supported by the selected dependencies; enable HEVC, VP9, AV1, profiles, levels, and encoders only after platform, patent, license, and conformance review.
- Preserve timestamps, seeking, flush/drain, playback-rate changes, subtitle timing, resolution switches, surface recreation, suspend/resume, and audio/video synchronization.
- Support stereo PCM first and multichannel PCM after channel-layout/routing tests. Treat Dolby/DTS and compressed bitstream passthrough as separately licensed later features.
- Reject protected-buffer requests safely and report DRM limitations. Do not emulate Widevine L1, HDCP, secure decoder, hardware TEE, or certified protected playback.

## Service modes, root, and app policy

- Offer the same immutable `ServiceMode=NoApp|microG|GApps` contract as Desktop, with `NoApp` default and the host APK installer in every mode. `microG` includes verified F-Droid plus approved microG; `GApps` accepts only a user-selected package through Phase 2B and never implies Google TV/Play Protect certification.
- Test the official F-Droid Client in `microG` mode for D-pad navigation, focus visibility, 10-foot text, repository management, install confirmation, updates, and error recovery.
- If the official client fails the TV usability gate, implement a minimal TV catalog using reviewed F-Droid signed-index/APK-verification components; retain Android package-install confirmation until privileged installation is separately approved.
- Offer `RootMode=Off|KernelSU|Magisk` only through the Phase 2C Advanced/Developer gate, default to `Off`, and keep all root grant/module/recovery UI D-pad-operable. Do not let root bypass the host recovery overlay or media/DRM policy.
- Show Leanback-capable applications by default. Put touch-only, portrait-only, and non-TV applications in an explicit compatibility section with warnings and per-app scaling/input policy.
- Maintain separate TV compatibility results for each service/root mode, F-Droid, sideloaded APKs, microG/GApps APIs, Native Bridge provider/ABI, graphics backend, codec path, resolution, and input device.
- Default TV clipboard forwarding to off; when enabled, expose text only with an obvious D-pad-accessible toggle and active-instance indicator. Do not auto-export sensitive clips or enable rich/file clipboard formats in first TV GA.
- Default TV host notification forwarding to off or redacted summary mode. Require a per-app/channel allowlist, keep previews hidden until opted in, and place ordinary notifications in a remote-friendly center rather than interrupting playback/gameplay.

## Validation and release gates

- Run Android TV CTS/CDD checks, relevant VTS, TvSystemUI/Settings tests, accessibility scans, D-pad focus fuzzing, gamepad automation, media conformance, and long-play soak tests.
- Maintain a 50-app blocking suite and 250-app beta matrix spanning launchers, video/audio players, games, browsers, signage, education, media sessions, subtitles, background playback, and non-Leanback fallbacks.
- Test 1080p and 4K profiles on Intel/AMD/Qualcomm Windows hardware and multiple Apple Silicon generations, including monitor hot-plug, refresh changes, sleep/wake, GPU reset, decoder reset, low memory, and network transitions.
- Require stable 1080p 60 FPS UI, D-pad response under 100 ms p95, defined 4K results, A/V drift under 40 ms, and no dropped-frame bursts over 1% for supported reference media.
- Publish per-host codec, resolution, refresh, HDR, audio-channel, controller, service/root mode, ARM translation, and DRM support matrices.
- Test D-pad-only clipboard/notification settings, notification opening/dismissal/actions, preview redaction, focus restoration, flood suppression, and behavior during video/game playback.
- Packet-capture long-play idle/navigation/media sessions across eligible service/root modes and verify zero telemetry plus only declared application, F-Droid/opt-in microG, user-installed GApps, rooted app/module, and update traffic. Measure sandbox helper wakeups, CPU, memory, dropped frames, and A/V drift.

## Exit criteria

- Signed x86_64 and arm64 TV images boot on every committed host architecture and negotiate the same runtime protocol as Desktop.
- Provisioning, launcher, Settings, per-mode app discovery, root safe mode/recovery, permission prompts, update/recovery, and core apps are fully operable with D-pad only.
- 1080p and 4K SDR presentation, host audio/video decode fallback, input reconnect, suspend/resume, and renderer/decoder crash recovery meet their gates.
- TV instances cannot enable Desktop per-task export or reuse Desktop userdata, and edition-specific updates cannot cross-install.
- No release material implies Google TV/Play Protect certification, guaranteed Play Store access, Widevine L1, HDCP, or commercial streaming compatibility; any user-imported GApps mode remains clearly uncertified and best-effort.
- The TV image and host presentation pass Phase 6A sandbox, egress, local-diagnostics, and hardening performance gates without a TV-wide compatibility bypass.
