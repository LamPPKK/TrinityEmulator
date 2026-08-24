# Phase 2B — Service Modes and User-Installed GApps

Duration: 8–12 weeks overlapping Phase 2, Phase 5, and Phase 6
Purpose: Provide exactly three instance service modes—`NoApp`, `microG`, and `GApps`—without unauthorized distribution of proprietary Google packages, weakening the verified guest, or confusing first-party zero telemetry with third-party network behavior.

This phase does not grant a right to use or redistribute Google applications. Android compatibility does not automatically grant access to Google Play or a GMS license, and only Play Protect-certified devices are eligible to include licensed Google apps. A user-imported GApps add-on is therefore always uncertified and best-effort. A separately licensed `GApps + RootMode=Off` artifact may pursue official certification, integrity, DRM, and banking compatibility only through an approved Google/device/OEM partner program—never through spoofing or a user-imported payload.

## Locked service modes

Every instance selects exactly one immutable `ServiceMode` when it is created:

| Mode | Preinstalled application/service set | Store path | Google-compatible services | Intended use |
|---|---|---|---|---|
| `NoApp` | AOSP plus required Trinity/LeapDroid integration agents only | Host APK installer and ADB when developer mode is enabled | None | Minimal, private, offline-capable baseline |
| `microG` | Verified F-Droid Client plus the approved microG package set | F-Droid official repository by default; normal APK install remains available | Restricted microG compatibility; each network service is opt-in | Libre application discovery with partial Google API compatibility |
| `GApps` | Either a compatible user import or a separately licensed/certified partner artifact; no microG | Whatever the accepted provider lawfully supplies; F-Droid may later be installed as an ordinary user choice | Proprietary Google components under the provider's rights/status | User import: Advanced/uncertified; partner artifact: conditional certified non-root SKU |

Mode rules:

- `NoApp` is the default for every new Desktop and TV instance. It contains no F-Droid, microG, GApps, Play Store, or other optional application-service bundle.
- The `microG` mode is the recommended compatibility mode. F-Droid and microG are pinned and verified as described by the main plan; Google device registration, Cloud Messaging, and network-location backends remain separate opt-ins.
- The user-imported `GApps` path is visible only when its platform/ABI/Android-version/edition import gate is enabled. It does not bundle, mirror, search for, download, extract from another installed product, or provide bypass instructions. A `CertifiedPartner` artifact is built/distributed only through its exact approved private supply chain and rights.
- microG and GApps never coexist in one instance. Signature spoofing support is enabled only for the exact approved microG packages/certificates in `microG` mode and is unavailable to imported GApps or ordinary applications.
- Service mode is part of the signed instance manifest, snapshot format, compatibility key, diagnostics header, update preflight, and support report. Guest applications cannot change it.
- Changing mode creates a new instance. The migration wizard may copy user-selected files, APK lists, and app data that Android backup/export policy permits, but never migrates Google/microG accounts, tokens, device identifiers, service databases, privileged settings, system package data, or signing state.
- Within `GApps`, record immutable `GAppsProviderClass=UserImport|CertifiedPartner`. `UserImport` is always visibly uncertified. `CertifiedPartner` exists only for a signed, licensed distribution artifact approved for the exact product/host/ABI/Android build; it is never synthesized from a user import.

## Certified non-root GApps release gate

- Treat `GAppsProviderClass=CertifiedPartner + RootMode=Off` as a separate conditional release target. `RootMode=Off`, locked verified boot, SELinux enforcing, production keys, supported security patch level, exact certified build fingerprint, and no root/module residue are mandatory but do not by themselves confer certification.
- Obtain written GMS/Google Play distribution rights and Play Protect device certification for the exact virtual-device product before bundling Google applications or displaying `Certified`. Provision attestation/device identity only through the approved manufacturer/partner process; never import, clone, generate, share, or spoof another device's keys or identifiers.
- Define the required Play Integrity verdicts with Google and participating app/service owners. Run official server-verified standard/classic request tests, replay/nonce tests, app licensing tests, update tests, and negative rooted/tampered-image tests. Do not fake `MEETS_DEVICE_INTEGRITY`, `MEETS_STRONG_INTEGRITY`, or any virtual-device verdict.
- SafetyNet Attestation is deprecated and replaced by Play Integrity. Keep a legacy SafetyNet compatibility suite only for applications that still use it and only when the licensed Google stack legitimately provides the response; do not build a replacement/fallback that fabricates verdicts.
- Integrate Widevine only under the applicable license and partner repository access. Declare the proven security level truthfully: L1 requires the approved OEMCrypto/chipset and secure decode/render path; otherwise expose only the licensed lower level actually certified. Never import keyboxes, device blobs, HDCP keys, or OEMCrypto libraries from other products.
- Build a regional banking/financial blocking suite with app-owner-approved test accounts and policies. A `CertifiedPartner + Off` release candidate must pass the agreed blocking set for sign-in, biometric/credential flow, network/TLS, updates, suspend/resume, notifications, and transaction sandbox; this is a product test claim, not a guarantee that every bank will permit a virtual device forever.
- Keep anti-cheat, commercial streaming, and other protected-app results separately owned by each service. Official certification and a clean guest may improve compatibility but never override an app/server decision.
- If licensing, certification, attestation provisioning, Widevine tier, or the banking blocking threshold is unavailable on a host/ABI, do not ship or label the `CertifiedPartner` variant there. Retain `UserImport` as an explicitly uncertified Developer option and publish the failed gate.

## WSA-referenced Widevine L3 path

- Use the public WSA roadmap as the Windows behavior target: ClearKey/MPEG-DASH and software Widevine L3 are supported; hardware DRM is not. WSA's public repositories do not contain the runtime DRM implementation, so no WSA binary, guest image, CDM, device blob, keybox, provisioning response, or undocumented protocol is an implementation input.
- Implement the path independently through Android's MediaDrm/Crypto framework and HAL/service contracts. Integrate the Widevine CDM, provisioning, certificates, license policy, update channel, and test material only from the approved Widevine/GMS partner supply chain.
- Start with ClearKey and standards-based MPEG-DASH/CENC plumbing as a source-available functional baseline. Then add a software-security-level Widevine L3 provider behind the same versioned guest contract; report the actual security level to applications without property overrides or spoofing.
- Keep CDM parsing/decryption and license state in a dedicated SELinux-confined guest service. Bind storage to the instance/user/provider/build, encrypt it at rest where the partner design permits, exclude secrets and decrypted samples from logs/diagnostics/snapshots, and erase/rotate state under the licensed reset/update policy.
- Route only policy-approved non-secure decoded output through the normal guest MediaCodec/Gfxstream or host-decode surface path. If a license or application requires secure buffers, hardware-backed keys, trusted decode, HDCP, or Widevine L1, reject it truthfully; never downgrade the requested protection silently.
- On Windows, WSA defines expected L3 behavior but contributes no code. On macOS, reuse the same guest MediaDrm/Crypto/provider contract with the licensed CDM; do not copy Windows Edge's Widevine CDM or attempt to route Android licenses through a browser CDM.
- Classify provisioning and license-server traffic as third-party functional DRM traffic, not Trinity/LeapDroid telemetry. Maintain endpoint ownership, user-visible DRM state, bounded local errors, offline behavior, and zero secret/token/content logging.
- Validate provisioning, streaming and offline licenses where licensed, renewal/release, CENC `cenc`/`cbcs` combinations supported by the target Android release, key rotation, clock/network change, suspend/resume, app/guest update, reset, multi-instance isolation, decoder crash, output-resolution policy, and server revocation using approved test content/accounts.
- Treat Widevine L1/hardware DRM as a later, separate `CertifiedPartner + Off` path requiring OEMCrypto/chipset or equivalent approved TEE, secure decode/render, protected memory, output protection/HDCP, robustness review, and partner certification. WSA L3 is not evidence that those requirements are met.

## Shared base and add-on architecture

- Build one signed AOSP base per edition and ABI. Keep host integration agents, framework contracts, kernel/vendor configuration, SELinux base policy, and update logic identical across service modes.
- Express `NoApp` as the absence of an optional service add-on. Express `microG` and `GApps` through versioned, mode-specific add-on manifests and read-only layers rather than forks of the base source tree or mutable `/system` remounts.
- Freeze an approved add-on mechanism in Phase 0. The preferred design is a locally sealed, content-addressed `product`/`system_ext` add-on image chained into per-instance verified metadata. It must preserve base AVB/dm-verity, SELinux enforcing, rollback, and atomic update behavior.
- Do not ship GApps mode if the implementation requires root, a writable system partition, disabling AVB/SELinux, accepting arbitrary privileged permissions, spoofing a certified device fingerprint, bypassing attestation, or executing an unvalidated recovery script.
- Give every add-on a schema version, mode, Android API/build, ABI, Desktop/TV edition, required base-image range, package list, original APK signatures, requested privileged permissions, SELinux additions, disk budget, content hash, import time, and compatibility result.

## GApps import pipeline

- Define a payload-free `GAppsProvider` descriptor and importer. The product ships schemas, validation policy, synthetic fixtures, lifecycle hooks, and tests only—not proprietary APKs, configuration files, setup scripts, URLs, or signing assets.
- Accept only a package explicitly selected through the host file picker. The UI requires the user to acknowledge that they obtained it lawfully, that Trinity/LeapDroid does not license or certify it, and that imported Google components can communicate with Google independently of the product.
- Inspect archives in a dedicated no-network, low-privilege process. Bound compressed/uncompressed size, nesting, file count, path length, CPU, memory, and time; reject traversal, symlinks, devices, sparse-file abuse, duplicate paths, executable host payloads, malformed ZIP/APK data, and decompression bombs.
- Parse a declarative allowlisted format. Never execute provider scripts, recovery binaries, shell fragments, installers, post-install hooks, or guest-supplied native host tools.
- Verify every APK signature without resigning or modifying the APK. Validate internal package consistency, duplicates, split APK relationships, API/ABI/edition compatibility, required libraries, privileged-permission declarations, disk requirements, and conflicts with AOSP or host integration packages.
- Treat a successful cryptographic check as integrity evidence, not proof of ownership, authenticity, license, Play Protect certification, or Google approval. The UI records the local package hash and validation result without uploading either.
- Copy an accepted package into host-protected, content-addressed private storage, construct the add-on transactionally, seal it read-only, run offline PackageManager/SELinux/boot preflight, and commit the instance only after health checks pass.
- On failure, erase the partially generated layer, retain only bounded redacted local diagnostics, and leave the base instance unchanged. Never weaken a validation rule to make an imported package boot.

## Privilege, identity, and sandbox policy

- Preserve imported APK signatures and Android UIDs. Do not grant a shared system UID, platform signature, privileged permission, SELinux allow rule, hidden API exemption, or background capability unless the approved descriptor declares the exact package/certificate and Phase 0 proves it is required and containable.
- Maintain a minimal per-provider privileged-permission and SELinux delta. Diff every update, reject unexpected expansion, and expose the delta in the local import report.
- Keep Google account credentials, authentication UI, tokens, backups, billing, purchases, and Play data inside the guest Google applications. Host UI, diagnostics, clipboard, notification, file, and update brokers never read or export them.
- Do not register an uncertified instance as certified on the user's behalf, spoof an approved model/fingerprint, bypass Play Integrity/SafetyNet, hide root/modification state, import Widevine keys, or claim protected-content support.
- Apply the same per-app Network/Sensors policy and Android sandbox boundaries to imported applications where technically enforceable. Document that blocking required Google endpoints can break sign-in, updates, push messaging, licensing, purchases, location, backup, or applications that depend on them.
- Treat every GApps parser, package, service, native library, and update as third-party code outside the first-party zero-telemetry trust statement.

## Provisioning and host UX

- The create-instance wizard presents three native cards in this order: `NoApp`, `microG`, `GApps`. It shows included components, download/import behavior, network expectations, privacy boundary, certification status, disk cost, edition/ABI compatibility, and migration implications before creation.
- `NoApp` and `microG` can be created without any proprietary payload. Selecting `GApps` opens the import flow and cannot create an instance until validation succeeds.
- Display `Uncertified user-installed GApps` persistently in instance settings, diagnostics, compatibility reports, and support bundles. Do not use Google Play/Google TV branding as a product badge.
- Show the imported package source label entered by the user, local hash, provider descriptor, Android/API/ABI/edition match, validation date, current health, update compatibility, and disable/delete action. Never transmit the label or hash.
- Deleting a GApps instance removes its derived add-on, caches, snapshots, and account-bearing userdata through the normal secure instance-deletion flow. The shared AOSP base remains untouched.

## Updates, rollback, and recovery

- Bind each snapshot and update to the exact service mode and add-on content hash. Reject restore into a different mode or an incompatible base image.
- Before a guest OTA, test the installed add-on against the target Android API/build, PackageManager rules, SELinux policy, privileged-permission allowlist, and boot health checks. Block the update with an actionable explanation if compatibility is unknown or failed.
- Update imported GApps only through an accepted in-guest vendor mechanism or a new user-import transaction. Trinity/LeapDroid does not proxy, mirror, or silently replace Google packages.
- Stage base and add-on changes atomically and retain the previous known-good base/add-on pair for rollback. A failed health check returns to the previous pair without merging package state across modes.
- Safe mode can boot with third-party service packages disabled for diagnosis, but it cannot convert the instance to another mode or silently delete account data.

## Privacy and network accounting

- First-party Trinity/LeapDroid zero telemetry remains mandatory in all three modes. No product analytics, automatic crash upload, remote logging, experiment client, stable product tracking ID, or hidden endpoint is added for GApps integration.
- `NoApp` is the clean packet-capture baseline: only explicitly enabled updates and user-configured connectivity services may contact the network before user applications are installed.
- `microG` traffic is third-party compatibility traffic and occurs only for the separately enabled functions documented by the product.
- `GApps` traffic is third-party proprietary traffic controlled by the imported packages and Google account/settings UX. The product inventories which guest processes own observed connections locally but does not claim to enumerate, minimize, or eliminate Google's data collection.
- Packet-capture release gates separate first-party destinations from F-Droid, microG, GApps, and user-app traffic. A GApps endpoint never enters the first-party allowlist merely because the product supports the mode.
- Enterprise/offline policy can disable creation and startup of `GApps` mode while leaving `NoApp` available.

## Android TV rules

- `NoApp` remains the default TV mode. `microG` includes the reviewed TV-compatible F-Droid/microG UX described in Phase 2A.
- Enable `GApps` on TV only for an explicitly compatible arm64/x86_64 TV package descriptor that passes D-pad-only provisioning, account, permission, update, recovery, 10-foot readability, media, and long-play tests.
- Reject phone/tablet GApps packages for TV when they change the product characteristic, replace the approved TV SystemUI/launcher unexpectedly, require touch-only provisioning, or claim unavailable DRM/certification capabilities.
- User import does not make the product Google TV-certified, Play Protect-certified, Widevine-capable, or eligible for licensed Google TV services. Keep all such claims out of UI and release material.
- A certified TV provider is a distinct `CertifiedPartner + RootMode=Off` artifact and must independently pass the exact Google TV/Play Protect, attestation/Play Integrity, Widevine/secure-decoder/HDCP, remote provisioning, and commercial-service gates approved for that target. Desktop approval never transfers automatically to TV.

## Verification matrix

- **Mode isolation:** create/delete, failed import, restart, snapshot/restore, clone, update/rollback, low disk, power loss, corrupted layer, and attempted cross-mode restore for all three modes.
- **Import security:** malformed archives/APKs, traversal, symlink/device entries, compression bombs, duplicate packages, signature mismatch, split mismatch, API/ABI/edition mismatch, conflicting package/permission, oversized native library, executable host payload, and malicious metadata.
- **Guest security:** AVB/dm-verity, SELinux enforcing, exact privileged-permission deltas, Binder/file/socket isolation, per-app Network/Sensors denial, account/token non-export, safe mode, and no signature-spoof access outside approved microG identities.
- **Compatibility:** boot, provisioning, app install/update, sign-in where legally testable, notifications, clipboard redaction, backup disclaimers, base OTA, package update, Gfxstream/ANGLE, Native Bridge combinations, Desktop windows, and TV D-pad/media behavior.
- **Certified provider:** `RootMode=Off` absence, locked boot chain, exact certified build/patch level, partner-provisioned identity, official Play Integrity server verification, legacy SafetyNet where still available, licensed Widevine security-level/secure-path tests, banking blocking suite, factory reset, OTA, rollback, snapshot restrictions, and rooted/tampered negative controls.
- **WSA-class L3:** ClearKey/MPEG-DASH, licensed CDM/provisioning provenance, honest software security level, streaming/offline license lifecycle, CENC, non-secure decoder/surface path, multi-instance key isolation, secret-free diagnostics, output-policy enforcement, and secure-buffer/L1 negative tests.
- **Privacy:** dependency/endpoint scan plus packet captures for fresh boot, idle, account signed out/in, store idle/update, app launch, induced crash, diagnostics export, suspend/resume, and deletion. Results classify first-party, mode-provider, user-app, and host-OS traffic separately.
- CI uses synthetic signed GApps-like fixtures for importer/parser/security tests. Proprietary packages never enter public CI artifacts, logs, caches, test reports, or repositories; any private legal compatibility test remains access-controlled and produces only non-content results.

## Deliverables

- Service mode ADR and versioned `ServiceMode`/add-on manifest schemas.
- Payload-free `GAppsProvider` format, import threat model, sandbox/capability specification, synthetic corpus, and negative-test plan.
- Legal/product review covering import UX, absence of redistribution/download assistance, certification language, privacy disclosure, trademarks, and support boundaries.
- Conditional certified-provider program plan covering partner ownership, GMS/Play Protect approvals, attestation provisioning, Play Integrity verdict contract, SafetyNet legacy disposition, Widevine license/security level/secure media path, banking test governance, incident response, and per-target go/no-go evidence.
- WSA Widevine L3 behavior ledger and independent AOSP MediaDrm/Crypto architecture covering provider/CDM boundary, provisioning/license flow, instance-bound storage, decoder/surface routing, update/revocation, privacy, test content, and explicit exclusions.
- Mode-specific provisioning, settings, migration, deletion, update, rollback, diagnostics, and recovery UX specifications for Windows and macOS Desktop/TV.
- Compatibility and egress matrices for `NoApp`, `microG`, and every separately accepted GApps provider/Android/ABI/edition combination.

## Exit criteria

- Exactly one mode is bound to each instance, survives restart/update, cannot be changed by the guest, and cannot share account/service state with another mode.
- `NoApp` contains no optional store or Google-compatible service packages; `microG` contains only the pinned F-Droid/microG set; `GApps` contains only the accepted user import plus required AOSP integration.
- No public source, public CI/cache, user-import provider, ordinary update server, support bundle, or log contains or downloads proprietary GApps payloads. A licensed `CertifiedPartner` payload may exist only in its access-controlled build/update channel under the approved distribution terms.
- Import and add-on construction preserve the original APK signatures, verified base, SELinux enforcement, immutable system partitions, exact privilege policy, atomic rollback, and local-only diagnostics.
- An uncertified or incompatible GApps package fails with a truthful local explanation; the product never bypasses certification, attestation, licensing, signature, or DRM controls.
- First-party zero telemetry passes in all modes, while UI/tests clearly classify F-Droid, microG, GApps, user-app, and host-OS traffic as separate trust domains.
- Desktop and TV expose GApps mode only for provider combinations that pass their legal, security, provisioning, update, input, graphics, and compatibility gates; unsupported combinations remain hidden.
- Any `CertifiedPartner + RootMode=Off` SKU has written distribution/certification approval and passes the agreed Play Integrity, legacy SafetyNet, licensed DRM, and banking blocking gates on that exact target; otherwise it is not shipped or described as certified.

## Primary references

- Android Compatibility FAQ (Google Play/GMS access is separate from Android compatibility): https://source.android.com/docs/compatibility/compatibility-faq
- Google Play Help — Play Protect certification status and eligibility for Google applications: https://support.google.com/googleplay/answer/7165974
- Google Play services overview and background-service/update model: https://developers.google.com/android/guides/overview
- Google Play Integrity API overview: https://developer.android.com/google/play/integrity/overview
- SafetyNet Attestation deprecation notice: https://developer.android.com/privacy-and-security/safetynet
- Widevine DRM overview and device integration boundary: https://developers.google.com/widevine/drm/overview
- Widevine partner access and L1 OEMCrypto requirement: https://developers.google.com/widevine/access
- Microsoft WSA public capability roadmap (software Widevine L3 available; hardware DRM unavailable): https://github.com/microsoft/WSA/
- Microsoft WSA Linux kernel source (kernel-only reuse input): https://github.com/microsoft/WSA-Linux-Kernel
