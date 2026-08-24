# Phase 6 — Security, Updates, Compliance, and Licensing

Duration: Continuous from Phase 1; 16–24 weeks of focused release work  
Purpose: Make the subsystem distributable and supportable rather than merely functional.

## Security architecture

- Threat-model guest apps, compromised Android system services, malformed graphics/control streams, hostile APKs, user-selected GApps/Native Bridge archives, rooted guest services/modules, network attackers, local unprivileged users, update compromise, and supply-chain compromise.
- Run VM controller, GPU renderer, media, integration brokers, Native Bridge/GApps/root-artifact inspection, root recovery, diagnostics, updater, and UI with the minimum privileges and explicit capability-based IPC ACLs. Follow the per-platform process matrix in Phase 6A.
- Apply Windows restricted tokens/AppContainer where compatible, Job Objects and process mitigations; apply macOS App Sandbox/Hardened Runtime/XPC where compatible. Enforce process/resource limits, memory and message quotas, watchdogs, egress rules, and crash containment.
- Require bounds-checked generated codecs for new protocols; fuzz all guest-controlled parsers continuously.
- Enforce SELinux, AVB, signed boot/system images, encrypted or host-protected userdata, secure key storage, and production ADB policy.
- Store update keys offline and separate development, beta, and production trust roots.
- Threat-model input as sensitive content: forbid host-global key collection and all raw input-content logging/export, constrain routing to the focused visible instance, and retain a host-owned emergency release path independent of guest/renderer health.
- Sandbox custom cursor decoding and controller-haptics handling; bound cursor bytes/dimensions, composition length, device count, event queues, haptic rate/duration/duty cycle, and mapping files. Fuzz their guest-controlled protocols continuously.
- Treat clipboard and notification content/actions as hostile and sensitive. Sandbox HTML/image/icon decoding; bound formats/items/bytes/dimensions/rates; keep `PendingIntent` in guest; authenticate short-lived revision-bound actions; and forbid content, reply text, digests, or action arguments in logs or diagnostic exports.
- Enforce host-user, instance, Android-user/profile and edition namespaces. Sensitive clips, secret notifications, stale tokens, restored snapshots, revoked permissions, and stopped brokers must fail closed without cross-boundary leakage.
- Treat app labels/icons, native window metadata, activation URIs, AppUserModelIDs, Spotlight items, presentation profiles, bounds/density, and task/display revisions as hostile guest-derived data. Derive host identities from trusted package/user records, bound all fields/rates, reject cross-user activation, and prevent one app from changing another app's window/configuration.

## Zero telemetry and local diagnostics

- Production binaries and guest images contain no analytics SDK, automatic crash uploader, remote logger, usage collector, experimentation client, stable installation/device tracking ID, or undocumented metrics endpoint. This is a mandatory build policy, not a user opt-out.
- Keep diagnostics in bounded local ring buffers and host-protected crash storage. Apply retention/size caps, secret/content redaction, a user-facing viewer, and a preview step before explicit manual export; no background export or automatic support upload exists.
- Maintain a machine-readable first-party egress manifest for each artifact. Signed static updates, F-Droid/explicitly enabled microG, user-installed GApps, rooted apps/modules, ordinary user apps, and user-configured DNS/time/connectivity/location are distinct functional traffic classes and must be independently auditable.
- Update checks use no stable installation/device ID, tracking cookie, or per-device query parameter. Support offline/enterprise update and host-provided or user-configured time/connectivity behavior.
- Release packet captures cover boot, idle, app launch, induced crash, update, suspend/resume, and shutdown across every eligible service/root combination and connectivity state. Any undocumented first-party destination or telemetry request blocks release.
- Document that third-party Android apps and Windows/macOS are separate trust domains. Provide per-app guest Network permission and optional default-deny templates, but do not promise to eliminate traffic outside the product's own components.

## Update system

- Signed manifest covering host components, guest partitions, firmware, protocol range, migration version, dependencies, and rollback metadata.
- Staged channels: development, canary, beta, stable, and enterprise/LTS if funded.
- Download resume, delta updates where safe, preflight disk checks, transactional install, health check, and automatic rollback.
- Preserve userdata across guest updates and test every supported forward migration and one-version rollback.
- Publish security advisories and define emergency-update and key-rotation procedures.
- Keep rollout selection channel/local-policy based rather than server-side per-device experimentation. Static signed manifests must support staged channels without tracking individual installations.

## Android and graphics compliance

- Run CTS continuously with stored result history and owner triage.
- Run VTS for kernel/HAL changes and CTS Verifier on release candidates.
- Run ANGLE/Gfxstream suites, dEQP, and Vulkan CTS subsets appropriate to advertised features.
- Advertise only features and extensions that pass the required conformance and real-app gates.
- Run Android TV CTS/CDD checks and TV-specific SystemUI, Settings, input, accessibility, media, resolution, refresh, and long-play suites for each TV ABI/host target.
- Run Android keyboard/key-character-map, pointer capture/icon, game-controller, InputDispatcher, focus-isolation, reset, and hot-plug tests on every host/ABI/edition target; verify release-all and cursor/haptics recovery under fault injection.
- Run Android clipboard privacy/current-focus/system-agent tests and notification-listener/action/RemoteInput/visibility tests; run host permission, DND, lock/privacy, own-notification reconciliation, parser-fuzz, flood and multi-instance isolation suites.
- Advertise only codecs/profiles/levels, resolutions, refresh rates, audio layouts, and HDR modes proven on the relevant host. Protected-content capabilities remain absent unless a separate licensed design is approved.

## Service modes, root, and Native Bridge security

- Prove `NoApp` and `Off` package/file/permission/egress absence. Treat `ServiceMode` and `RootMode` as signed immutable instance metadata; changes create/clone to a new instance and never migrate service accounts, credentials, tokens, system packages, root grants, or modules.
- Pin and verify microG and F-Droid source/release provenance, APK signing certificates, license texts, update channels, and SBOM entries.
- Keep microG and F-Droid in ordinary app sandboxes. Restrict microG signature spoofing to exact pinned identities; do not grant F-Droid a privileged installer identity without the separate signer-restricted review.
- Test that signature spoofing is impossible outside the exact approved microG packages/certificates and cannot be enabled by a user-installed APK.
- Treat F-Droid repositories as separate trust roots; verify signed indexes and APK hashes, require explicit confirmation for new repositories/keys, and prevent host catalog code from bypassing client verification.
- Treat a user-selected GApps package as hostile proprietary input: validate it in a no-network sandbox, retain original APK signatures, allowlist the descriptor/files/privilege delta, construct a sealed read-only add-on, preserve AVB/SELinux, and fail transactionally to the clean base. Never mix it with microG or describe it as certified.
- Treat `CertifiedPartner` as a distinct signed supply-chain class: exact approved GMS payload/build fingerprint/security patch level, protected unique identity/attestation provisioning, non-clonable key lifecycle, snapshot/reset rules, official server-side Play Integrity validation, SafetyNet legacy disposition, licensed Widevine component/security-level provenance, and revocation/incident response. `RootMode=Off` is mandatory.
- Pin KernelSU/Magisk source and licenses, build each root artifact reproducibly, preserve AVB/dm-verity and SELinux, default-deny grants, authenticate management, and treat every module as arbitrary guest-root code. Keep Zygisk off by default; prohibit concealment and attestation/DRM/banking/anti-cheat bypass features. KernelSU x86_64 remains unavailable if its pinned version requires weaker syscall hardening.
- Treat imported Native Bridge files as untrusted proprietary code: inspect in a sandbox, scan on the host, validate an allowlisted manifest/hash when available, mount read-only, isolate caches, keep payload data/results local, and support immediate disable/rollback.
- Publish a support matrix per provider/Android version/ABI. A provider compatibility failure must degrade to native x86_64 operation, never relax AVB, SELinux, or linker namespace policy.

## Licensing and distribution

- Maintain machine-readable license metadata and SBOM for every host and guest artifact.
- Satisfy QEMU GPL source distribution requirements and keep proprietary/Apache host UI components clearly separated.
- Do not distribute GApps packages, Play Store, GMS, Widevine, Houdini, `libndk_translation`, proprietary codecs, or any other proprietary ARM translator without written redistribution rights. The GApps provider does not search, download, mirror, or extract them and does not grant certification or license rights.
- Distribute a `CertifiedPartner` GApps/Widevine artifact only within the exact written license, trademark, territory, update, provisioning, audit, and end-of-life terms. Keep confidential partner material, device keys/keyboxes, OEMCrypto, and signing assets out of public source, CI logs, diagnostics, snapshots, and user export.
- Distribute F-Droid Client under GPL-3.0-or-later obligations and microG under its Apache-2.0 terms; include notices, corresponding source links/offers as applicable, signing provenance, and update policy.
- Satisfy the pinned KernelSU and Magisk GPL obligations for every distributed source-built component, including notices, corresponding source/source offers, reproducible build inputs, local patches, and module-manager provenance; do not bundle unreviewed third-party modules.
- User import is not proof of legal entitlement. Require an acknowledgement and legal review of the product workflow; do not provide download URLs, extraction automation, or instructions that bypass vendor terms.
- Review every imported Waydroid and LineageOS change independently; preserve commit/license provenance and keep GPL Waydroid host code behind compliant source/process boundaries.
- Review every GrapheneOS-derived change independently against current AOSP; preserve commit/license provenance, security rationale, compatibility/performance evidence, and removal condition. Do not use GrapheneOS trademarks or imply equivalent security.
- Do not use Android TV or Google TV branding in a way that implies certification. Do not bundle proprietary Google TV launcher/content services, Widevine, HDCP keys, or licensed codecs without written rights and the required security architecture.

## Exit criteria

- Independent security review has no unresolved critical/high findings.
- Update rollback and key-rotation drills pass under network, disk, and power fault injection.
- Release candidate meets defined CTS/VTS/graphics pass thresholds.
- Complete SBOM, notices, source offers, privacy documentation, and support lifecycle exist.
- Each exposed Native Bridge provider has an approved legal disposition and technical support matrix; otherwise it is hidden/disabled in production.
- Each exposed service/root combination passes provenance, absence, privilege, import/build, isolation, recovery, update/rollback, telemetry, and compatibility gates; a failed provider/target remains hidden without blocking `NoApp + Off`.
- Each certified non-root GApps target has current written approval and passes partner-agreed Play Integrity, applicable SafetyNet legacy, licensed Widevine security-level, and banking blocking tests; any expired/failed gate removes the certified claim and blocks that SKU's release.
- Desktop and TV artifacts have separate signed manifests and migration paths; edition cross-install and userdata conversion tests fail closed.
- No release path captures ordinary input outside the focused product window, logs text/key content, leaves pointer capture active, or allows haptics/held keys to survive focus loss, device removal, suspend, reset, or crash.
- No release path reads clipboard history, exports sensitive clips automatically, requests unrelated host notifications, logs clipboard/notification/reply content, executes stale actions, or crosses host/Android users or instances.
- Production dependency scans and runtime captures prove no telemetry component, automatic upload, stable tracking identifier, or undocumented first-party network destination.
- All guest and host sandbox boundaries in Phase 6A pass negative, fuzz, escape, resource-exhaustion, crash-recovery, and performance gates without a global bypass.
