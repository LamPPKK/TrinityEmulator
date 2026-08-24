# Phase 2B — Service Modes and User-Installed GApps

Duration: 8–12 weeks overlapping Phase 2, Phase 5, and Phase 6
Purpose: Provide exactly three instance service modes—`NoApp`, `microG`, and `GApps`—without distributing proprietary Google packages, weakening the verified guest, or confusing first-party zero telemetry with third-party network behavior.

This phase does not grant a right to use or redistribute Google applications. Android compatibility does not automatically grant access to Google Play or a GMS license, and only Play Protect-certified devices are eligible to include licensed Google apps. Trinity and LeapDroid therefore treat GApps as a user-supplied, uncertified, best-effort add-on and make no Play Store, Play Protect, Play Integrity, DRM, banking, or anti-cheat compatibility promise.

## Locked service modes

Every instance selects exactly one immutable `ServiceMode` when it is created:

| Mode | Preinstalled application/service set | Store path | Google-compatible services | Intended use |
|---|---|---|---|---|
| `NoApp` | AOSP plus required Trinity/LeapDroid integration agents only | Host APK installer and ADB when developer mode is enabled | None | Minimal, private, offline-capable baseline |
| `microG` | Verified F-Droid Client plus the approved microG package set | F-Droid official repository by default; normal APK install remains available | Restricted microG compatibility; each network service is opt-in | Libre application discovery with partial Google API compatibility |
| `GApps` | A compatible GApps package imported by the user; no bundled F-Droid or microG | Whatever the accepted user package provides; F-Droid may later be installed as an ordinary user choice | Proprietary Google components from the user-supplied package | Advanced, uncertified compatibility testing |

Mode rules:

- `NoApp` is the default for every new Desktop and TV instance. It contains no F-Droid, microG, GApps, Play Store, or other optional application-service bundle.
- The `microG` mode is the recommended compatibility mode. F-Droid and microG are pinned and verified as described by the main plan; Google device registration, Cloud Messaging, and network-location backends remain separate opt-ins.
- The `GApps` mode is visible only when its platform/ABI/Android-version/edition import gate is enabled. The project does not bundle, mirror, search for, download, extract from another installed product, or provide bypass instructions for GApps.
- microG and GApps never coexist in one instance. Signature spoofing support is enabled only for the exact approved microG packages/certificates in `microG` mode and is unavailable to imported GApps or ordinary applications.
- Service mode is part of the signed instance manifest, snapshot format, compatibility key, diagnostics header, update preflight, and support report. Guest applications cannot change it.
- Changing mode creates a new instance. The migration wizard may copy user-selected files, APK lists, and app data that Android backup/export policy permits, but never migrates Google/microG accounts, tokens, device identifiers, service databases, privileged settings, system package data, or signing state.

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

## Verification matrix

- **Mode isolation:** create/delete, failed import, restart, snapshot/restore, clone, update/rollback, low disk, power loss, corrupted layer, and attempted cross-mode restore for all three modes.
- **Import security:** malformed archives/APKs, traversal, symlink/device entries, compression bombs, duplicate packages, signature mismatch, split mismatch, API/ABI/edition mismatch, conflicting package/permission, oversized native library, executable host payload, and malicious metadata.
- **Guest security:** AVB/dm-verity, SELinux enforcing, exact privileged-permission deltas, Binder/file/socket isolation, per-app Network/Sensors denial, account/token non-export, safe mode, and no signature-spoof access outside approved microG identities.
- **Compatibility:** boot, provisioning, app install/update, sign-in where legally testable, notifications, clipboard redaction, backup disclaimers, base OTA, package update, Gfxstream/ANGLE, Native Bridge combinations, Desktop windows, and TV D-pad/media behavior.
- **Privacy:** dependency/endpoint scan plus packet captures for fresh boot, idle, account signed out/in, store idle/update, app launch, induced crash, diagnostics export, suspend/resume, and deletion. Results classify first-party, mode-provider, user-app, and host-OS traffic separately.
- CI uses synthetic signed GApps-like fixtures for importer/parser/security tests. Proprietary packages never enter public CI artifacts, logs, caches, test reports, or repositories; any private legal compatibility test remains access-controlled and produces only non-content results.

## Deliverables

- Service mode ADR and versioned `ServiceMode`/add-on manifest schemas.
- Payload-free `GAppsProvider` format, import threat model, sandbox/capability specification, synthetic corpus, and negative-test plan.
- Legal/product review covering import UX, absence of redistribution/download assistance, certification language, privacy disclosure, trademarks, and support boundaries.
- Mode-specific provisioning, settings, migration, deletion, update, rollback, diagnostics, and recovery UX specifications for Windows and macOS Desktop/TV.
- Compatibility and egress matrices for `NoApp`, `microG`, and every separately accepted GApps provider/Android/ABI/edition combination.

## Exit criteria

- Exactly one mode is bound to each instance, survives restart/update, cannot be changed by the guest, and cannot share account/service state with another mode.
- `NoApp` contains no optional store or Google-compatible service packages; `microG` contains only the pinned F-Droid/microG set; `GApps` contains only the accepted user import plus required AOSP integration.
- No product artifact, repository, CI cache, update server, support bundle, or log contains or downloads proprietary GApps payloads.
- Import and add-on construction preserve the original APK signatures, verified base, SELinux enforcement, immutable system partitions, exact privilege policy, atomic rollback, and local-only diagnostics.
- An uncertified or incompatible GApps package fails with a truthful local explanation; the product never bypasses certification, attestation, licensing, signature, or DRM controls.
- First-party zero telemetry passes in all modes, while UI/tests clearly classify F-Droid, microG, GApps, user-app, and host-OS traffic as separate trust domains.
- Desktop and TV expose GApps mode only for provider combinations that pass their legal, security, provisioning, update, input, graphics, and compatibility gates; unsupported combinations remain hidden.

## Primary references

- Android Compatibility FAQ (Google Play/GMS access is separate from Android compatibility): https://source.android.com/docs/compatibility/compatibility-faq
- Google Play Help — Play Protect certification status and eligibility for Google applications: https://support.google.com/googleplay/answer/7165974
- Google Play services overview and background-service/update model: https://developers.google.com/android/guides/overview
