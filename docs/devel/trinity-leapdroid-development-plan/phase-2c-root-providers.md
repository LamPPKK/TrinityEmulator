# Phase 2C — KernelSU and Magisk Root Providers

Duration: 8–12 weeks overlapping Phase 2, Phase 6, and Phase 7
Purpose: Offer explicit developer root through KernelSU or Magisk without weakening non-root images, hiding device state, bypassing Google security controls, or extending guest root into host capabilities.

Root is a development and compatibility feature, not a prerequisite for Trinity/LeapDroid operation. A rooted guest must be treated as a potentially compromised guest: Android per-app isolation and some SELinux assumptions can no longer support the same security claim, while the VM boundary, host process sandboxes, authenticated brokers, user consent, quotas, and host filesystem/device policy remain mandatory.

## Locked root modes

Every instance selects exactly one immutable `RootMode` at creation:

| Mode | Guest boot/runtime | Availability | Security position |
|---|---|---|---|
| `Off` | Standard signed GKI and init/boot artifacts | Default; all stable Desktop/TV SKUs | Full planned guest and host security gates |
| `KernelSU` | Separately built KernelSU-enabled GKI artifact plus pinned manager | Advanced Developer channel; target/architecture gated | Kernel-level root for explicitly approved apps; rooted-guest warning |
| `Magisk` | Separately built Magisk-patched init/boot artifact plus pinned manager | Advanced Developer channel; target/architecture gated | Userspace/init root and optional modules/Zygisk; rooted-guest warning |

Root rules:

- KernelSU and Magisk never coexist in one instance. Their daemon, manager, policy, module directories, caches, boot artifacts, and backup metadata remain separate.
- `RootMode` is independent from `ServiceMode`, but the full matrix is not automatically supported. `GApps` + root is experimental and receives no Play Protect, Play Integrity, SafetyNet, DRM, banking, anti-cheat, or Google compatibility promise.
- `RootMode` is stored in the signed instance manifest, snapshot metadata, compatibility key, support report, update preflight, and diagnostics header. Guest applications cannot enable or replace the provider.
- Changing `RootMode` creates a new instance or explicit clone. Migration never copies root grants, superuser databases, modules, Zygisk state, denylists, scripts, injected policies, service credentials, or root-generated device identifiers.
- Rooted instances show a persistent native warning and distinct icon/state in launcher, settings, diagnostics, snapshot, export, and support surfaces.
- The project does not provide root concealment, certified-device fingerprint spoofing, attestation bypass, Play Integrity/SafetyNet bypass, anti-cheat evasion, DRM circumvention, banking bypass, or instructions/modules whose primary purpose is defeating those controls.

## Artifact architecture

- Build `Off`, KernelSU, and Magisk variants from the same pinned AOSP/GKI source, compiler, configuration, and reproducible pipeline. Root variants are explicit derived artifacts, never runtime patches to the shared clean image.
- Keep the standard base partitions read-only and AVB/dm-verity verified. Sign each root boot chain with the appropriate project development/root-channel keys and chain it through verified metadata; never use `--disable-verity`, `--disable-verification`, an unlocked verification policy, or a writable `/system` as the integration method.
- Bind every root artifact to exact Android build, kernel/KMI, ABI, edition, provider version, manager certificate, policy version, and service-mode compatibility range.
- Keep `Off` artifacts free of KernelSU/Magisk binaries, managers, daemons, init hooks, policies, module directories, Zygisk code, root sockets, and dormant feature switches. CI compares manifests and binaries to enforce absence.
- Root artifacts and updates use separate channel metadata so a stable non-root update cannot cross-install a rooted boot artifact and a rooted update cannot enter an `Off` instance.

## KernelSU integration

- Pin an audited upstream KernelSU source revision. Build the kernel portion under GPL-2.0-only obligations and the manager/userspace portions under their applicable GPL-3.0-or-later obligations; retain source, notices, build scripts, and exact provenance.
- Prefer a product-built GKI integration for the virtual device, matching KernelSU's emulator-oriented GKI model. Do not ask users to flash arbitrary upstream boot images or run remote setup scripts.
- Build and sign the KernelSU manager from the same pinned source policy, bind its certificate to the daemon, and expose root only to packages explicitly approved by the user.
- Default every application to no root. A grant shows package identity, requested UID/GID/groups/capabilities, duration, command context where available, and a persistent revocation path. Support one-time/session grants before permanent grants.
- Keep KernelSU module/metamodule support disabled until module safe-mode, bootloop recovery, update/rollback, disk quotas, SELinux delta reporting, and destructive-script warnings pass their gates.
- Gate x86_64 separately. Current KernelSU documentation notes that modern x86_64 integration can require disabling syscall hardening; Trinity must not disable that mitigation in a production artifact. If a pinned upstream revision cannot operate without the weakening, `KernelSU` remains unavailable on Windows x64 while `Magisk` and `Off` continue.
- Reject KMI/kernel mismatches before boot and retain the previous known-good kernel artifact for automatic rollback.

## Magisk integration

- Pin an audited upstream Magisk source revision and build it reproducibly under GPL-3.0-or-later obligations, including its submodules, source offer, notices, build configuration, and signing provenance.
- Produce the Magisk boot/init artifact inside the controlled guest-image build. Do not rely on a user-patched unknown image, custom recovery, guest download bootstrap, or live flash operation.
- Sign the project-built Magisk manager with the matching product key and preserve Magisk's manager/daemon certificate binding. Development/debug signature checks never enter production root artifacts.
- Keep Zygisk off by default because it injects code into application processes. Enabling it requires a per-instance warning, restart, compatibility record, and separate performance/security matrix.
- Do not configure Magisk DenyList, property replacement, namespace hiding, or third-party modules to misrepresent certification or bypass attestation. Upstream mechanisms may remain available for ordinary process isolation/compatibility, but the project does not ship bypass lists, spoofed properties, concealment defaults, or support claims.
- Treat Magisk's root daemon and SELinux policy changes as an explicit security delta. Diff policy on every update, preserve SELinux enforcing, forbid broad unrelated permissive domains, and publish the local delta in diagnostics.
- Preserve a clean source boot/init artifact and support atomic return to the previous known-good Magisk artifact after failed install, update, policy load, or health check.

## Root grants and module policy

- Root managers are the only guest authority that can grant `su`. Host UI may display/revoke state through a narrow read-only/control protocol but never stores root-manager credentials or silently approves a package.
- A root grant never creates a host capability. File, clipboard, notification, input, camera, microphone, location, update, diagnostics, Native Bridge, and GApps import brokers continue to require authenticated instance/user capabilities and host consent.
- On first root enablement, reset all host integration consent to safe defaults and require the user to re-enable sensitive bridges. Rooted TV instances keep clipboard/notifications off and developer interfaces local-only by default.
- Treat KernelSU/Magisk modules as arbitrary guest-root code. Module installation is advanced-only, requires a stopped/snapshotted instance and explicit warning, and cannot imply review or endorsement.
- Provide a native `Import local module` action for KernelSU and Magisk. Accept a compatible module archive only after the user explicitly selects it from local storage, chooses the rooted instance/provider, reviews package metadata and requested effects, and acknowledges that it is untrusted guest-root code.
- Do not bundle an online module store, remote module search, automatic module downloader, bypass/concealment payload, proprietary payload, or community repository trust root. The project does not recommend or certify a module's purpose merely because the generic importer accepts its format.
- Before accepting a module archive, the host sandbox checks provider/format compatibility, size/path/archive safety, records a local hash and signature metadata when present, and rejects host executables or traversal. These checks do not make guest scripts safe. Execution occurs only inside the selected rooted guest; no module can ship or execute host binaries through the import path.
- Provide a host-owned recovery boot that starts with all modules, Zygisk, and third-party root startup scripts disabled. Users can inspect local boot diagnostics, disable/delete modules, roll back the root artifact, export selected files, or delete the instance without needing a healthy guest UI.
- Bound module storage, startup time, processes, logs, and crash loops. Repeated failed boots automatically enter recovery and never broaden SELinux or host permissions as a workaround.

## Updates and lifecycle

- Root provider updates are explicit, signed, version-pinned transactions. Automatic base OTA first validates the matching root artifact, manager, policies, modules, KMI, AVB chain, and rollback pair.
- If no compatible root artifact exists, hold the guest OTA and offer: wait for support, clone to a new `Off` instance with permitted user data, or delete the rooted instance. Never silently remove root or boot a mismatched kernel/init image.
- Snapshot and restore require the exact `RootMode`, provider version/policy range, service mode, Android build, kernel/KMI, ABI, and edition. Cross-root restore fails closed.
- Provider removal is implemented as migration to a new `Off` instance, not partial deletion of files from rooted userdata. This prevents residual grants, modules, injected policy, modified properties, and daemon state from being treated as clean.

## Security and privacy boundaries

- Root invalidates the ordinary assumption that Android app UIDs, app SELinux domains, package permissions, guest secrets, and other guest users are protected from an approved root client. UI and documentation state this plainly.
- Continue to treat the entire guest as hostile at every virtio, graphics, media, control, clipboard, notification, file, input, and integration boundary. Root must not make a guest-controlled parser run in the host UI/installer process or widen VM-controller privileges.
- Production Trinity/LeapDroid host components remain zero telemetry in every root mode. Root providers/managers built and distributed by the project undergo endpoint/dependency review; automatic crash/usage upload is removed or disabled in project builds.
- User-installed root modules and rooted apps are third-party code and may collect or transmit data. Per-app Network controls and offline/default-deny templates remain available, but the product cannot guarantee their behavior after guest root deliberately changes guest enforcement.
- A root process cannot access host logs, package-import archives, provider payloads, user-selected host paths, credentials, hypervisor handles, update keys, or another instance except through the same bounded brokers and explicit host authorization as non-root guests.

## Host UX

- Add an Advanced `Root access` panel with `Off`, `KernelSU`, and `Magisk` cards; supported architecture/edition matrix; provider/source/license/version; security delta; disk/memory cost; module/Zygisk state; update compatibility; and create/clone/delete actions.
- Add a provider-scoped local module importer with native file selection, compatibility/metadata summary, local hash, risk acknowledgement, stopped-instance/snapshot requirement, install progress, enable/disable/remove controls, and direct entry to host-owned safe mode. Do not include discovery, download, ranking, or recommended-module feeds.
- `Off` is selected by default. Root cards require developer mode plus a typed/explicit acknowledgement that guest app isolation and certification-sensitive applications may fail.
- Show root grants, last grant/revocation time, module count, provider health, policy version, boot fallback, snapshot status, and update blocker without recording command contents, secrets, or raw app activity.
- Keep ADB disabled unless separately enabled and paired. Root mode does not automatically enable network ADB, host shell access, port forwarding, or an elevated host process.

## Android TV rules

- `Off` remains the only default TV root mode. KernelSU/Magisk TV instances are Developer-only and managed primarily through the host recovery/settings UI so a broken manager or module cannot make the 10-foot UI unrecoverable.
- Require D-pad-operable grant/revoke dialogs if superuser approval is exposed in TV SystemUI. Otherwise root approvals occur through the authenticated host panel with a clear active-instance/app identity and no unattended allow-all option.
- Rooted TV receives no protected-media, Google TV, Widevine, HDCP, commercial streaming, or certification claim and is excluded from the normal media compatibility score unless tested as a separate rooted SKU.

## Verification matrix

- **Targets:** Windows x64/x86_64, Windows ARM64/arm64, macOS ARM64/arm64; Desktop and TV; each supported `ServiceMode`; KernelSU/Magisk separately.
- **Boot chain:** reproducible build, AVB chain, correct KMI/ABI, clean `Off` absence test, install/update/rollback, power loss, low disk, corrupted artifact, wrong provider, and cross-mode restore rejection.
- **Privilege:** default deny, one-time/session/permanent grant, revoke, UID reuse, package update/reinstall/signature change, Android user/profile separation, manager removal, forged manager, daemon restart, and concurrent requests.
- **Modules:** malformed archive, path escape, bootloop, fork/storage/log bomb, SELinux expansion, Zygisk crash, provider update with modules, recovery disable-all, individual removal, snapshot restore, and cross-provider rejection.
- **Host containment:** rooted guest fuzzing and abuse of every protocol/broker, cross-instance/user attempts, host path/device/credential access, resource exhaustion, renderer/media escape, updater/importer separation, and recovery while guest system_server is compromised.
- **Compatibility:** app/NDK/GLES/Vulkan, input/controllers, clipboard/notifications, Native Bridge, GApps/microG, OTA, suspend/resume, native windows, TV navigation/media, and explicit certification-sensitive failure reporting.
- **Privacy:** endpoint scans and packet captures for `Off`, KernelSU, Magisk, Zygisk off/on, zero/one/multiple modules, grant/revoke, induced crash, update, recovery, and diagnostics export; first-party, provider, module/app, and host-OS traffic are classified separately.

## Deliverables

- Root-mode ADR and compatibility matrix across host/guest architecture, edition, service mode, Android build, kernel/KMI, and release channel.
- Reproducible KernelSU and Magisk source/build/signing/license manifests plus clean-`Off` absence proof.
- Provider-specific threat models, SELinux/boot deltas, manager trust design, root-grant schema, local module-import schema/policy, update/rollback plan, recovery image, fuzz corpora, and security review.
- Native Windows/macOS root settings, warning, grant-view/revoke, module recovery, update blocker, clone/migration, and deletion UX specifications.
- Published statement distinguishing rooted guest risk from VM/host containment and prohibiting certification/attestation/DRM/anti-cheat bypass claims.

## Exit criteria

- `Off` remains the default and contains no dormant root implementation; KernelSU/Magisk are mutually exclusive, explicit, versioned Developer artifacts.
- Every root artifact preserves the verified boot chain and SELinux enforcing. No supported path disables verity/verification, makes base partitions writable, or applies an unreviewed global permissive policy.
- KernelSU ships on a target only when the pinned integration preserves required kernel mitigations; otherwise that target truthfully reports unavailable.
- Magisk is reproducibly built and signed from pinned source, Zygisk is off by default, and no project default/module/property policy hides root or bypasses certification/security controls.
- Root grants are default-deny, attributable, revocable, profile-aware, and unable to create any host capability.
- Module bootloops and provider failures recover through a host-owned safe path without data loss, manual file deletion, sandbox bypass, or cross-provider state reuse.
- Rooted-guest protocol fuzzing and independent review find no unresolved critical/high host escape or cross-instance issue.
- First-party zero telemetry remains proven; documentation clearly states that rooted apps/modules can subvert guest privacy/security controls and are outside that guarantee.

## Primary references

- KernelSU official installation and GKI/LKM model: https://kernelsu.org/guide/installation.html
- KernelSU official x86_64 security caveat: https://kernelsu.org/guide/x86_64-support.html
- KernelSU official module model: https://kernelsu.org/guide/module.html
- KernelSU upstream source and licensing: https://github.com/tiann/KernelSU
- Magisk official installation/boot-image model: https://topjohnwu.github.io/Magisk/install.html
- Magisk official modules and Zygisk developer guide: https://topjohnwu.github.io/Magisk/guides.html
- Magisk upstream source and GPL-3.0-or-later license: https://github.com/topjohnwu/Magisk
