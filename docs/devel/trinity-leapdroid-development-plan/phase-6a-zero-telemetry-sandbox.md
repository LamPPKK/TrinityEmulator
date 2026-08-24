# Phase 6A — Zero Telemetry, GrapheneOS-Derived Hardening, and Layered Sandboxing

Duration: Continuous from Phase 1; 12–18 weeks of focused integration and release qualification  
Purpose: Make privacy and containment enforceable product properties while preserving desktop/TV responsiveness on Windows x64/ARM64 and macOS ARM64.

## Non-negotiable policy

- Production Trinity and LeapDroid emit no automatic product telemetry. There is no analytics SDK, usage tracker, automatic crash uploader, remote logger, experimentation client, stable installation/device tracking ID, or undocumented metrics endpoint.
- This is not an opt-out switch. Production builds omit the upload capability and release CI proves its absence through dependency inspection, endpoint scanning, and runtime network captures.
- Diagnostics are local, bounded, redacted, user-viewable, and exported only after an explicit user action. Development-only instrumentation is compile-time or locally policy-gated and cannot enter signed production artifacts.
- Functional traffic remains possible: user application traffic, F-Droid synchronization, explicitly enabled microG functions, signed updates, and user-configured DNS/time/connectivity/location services. Each is documented, independently controllable where technically possible, and distinguished from telemetry.
- The guarantee covers first-party Trinity/LeapDroid host and guest components. It cannot disable telemetry built into Windows/macOS or guarantee the behavior of user-installed apps; per-app guest network control and optional default-deny templates reduce that exposure.

## GrapheneOS reuse boundary

AOSP remains the canonical guest. GrapheneOS is a security research and implementation source whose changes are evaluated one at a time.

For every candidate change, record:

- Exact upstream repository, commit, license, authorship, and selected AOSP base.
- Whether current AOSP already contains an equivalent or newer implementation.
- Threat addressed, privilege boundary changed, code/attack surface added or removed, and expected failure mode.
- CTS/VTS/app compatibility impact plus CPU, memory, startup, frame-time, input-latency, and energy results.
- Import/reimplementation design, tests, owner, rebasing strategy, rollback, and removal condition.

First audit queue:

1. Hardened SELinux domains and seccomp-bpf policies for guest agents, media, graphics, parsers, updater, and imported-code paths.
2. Per-app Network and Sensors permission controls with profile-aware enforcement and compatibility UX.
3. Secure app-spawn patterns and reduced zygote/system-server attack surface.
4. Dynamic-code/JIT restrictions for first-party base services plus explicit, versioned per-app compatibility exceptions.
5. Targeted `hardened_malloc`, zero-on-free, integer/CFI/stack hardening, and MTE where the ARM host/guest path supports it.
6. Local log/crash viewer patterns with in-memory or bounded storage and manual sharing.
7. Storage Scopes and Contact Scopes after the base sandbox and permission model are stable.

Explicit exclusions:

- Do not adopt the whole GrapheneOS manifest or turn GrapheneOS into a merge dependency.
- Do not import Sandboxed Google Play; the product uses F-Droid and optional microG.
- Do not import Pixel firmware, radio, USB-C, telephony, hardware attestation, or other physical-device-only work.
- Do not depend on GrapheneOS network infrastructure or copy its service defaults without a subsystem-specific review.
- Do not use GrapheneOS branding or claim equivalent hardening, verification, hardware roots of trust, or security support.

## Guest containment model

- Preserve AOSP's kernel-enforced per-UID application sandbox for both managed and native code.
- Keep SELinux enforcing with per-app and per-service domains; no broad permissive domain or shared system UID is allowed in production.
- Keep seccomp-bpf, namespaces, Binder permission checks, scoped storage, Android user/work-profile separation, AVB/dm-verity, read-only system partitions, verified updates, and production ADB policy.
- Give every host-integration guest agent its own UID/domain and signature permission. Clipboard, notifications, input/IME, files, media, location, camera/microphone, lifecycle, and update services do not share a general privileged identity.
- Enforce per-app Network denial as a second layer covering direct and indirect guest access, including localhost where the implementation permits. Provide a compatibility error surface instead of silently bypassing policy.
- Add a Sensors toggle for sensor classes not already covered by dedicated Android runtime permissions. Scope policy per Android user/profile and make reset/backup semantics explicit.
- Keep microG in a normal app sandbox. Signature spoofing is limited by package name and pinned certificate; device registration, Cloud Messaging, location backends, and network access remain separate opt-ins.
- Keep F-Droid in a normal app sandbox with normal user-confirmed installation initially. Any privileged extension requires a signer-restricted IPC/service/domain design and independent security approval.
- Treat Native Bridge providers as higher-risk native code. Provider payloads remain read-only, versioned, separately cached, switchable only while stopped, and unable to weaken SELinux, AVB, linker namespaces, or the base guest ABI policy.

## Host process and capability model

No guest-controlled data parser runs in the privileged UI or the process that owns installation/update authority.

| Process | Allowed capabilities | Explicit denials and limits |
|---|---|---|
| Native UI/control center | Windowing, user prompts, capability requests | No guest stream parsing, VM memory mapping, update signing/install authority, unrestricted file scan, or hidden network client |
| Per-task AppShell/presentation broker | One native window, trusted app identity, bounded presentation protocol, focused task activation | No VM control, guest stream parsing, raw package-chosen host identity/command, unrelated task/profile access, updater, general files, or network |
| VM controller | Hypervisor/VM lifecycle, bounded device setup, guest memory ownership | No clipboard/notification/media parsing, user-document enumeration, general host credentials, or Internet egress |
| Renderer/GPU broker | GPU device, validated graphics rings, surface export | No user files, clipboard, notifications, updater, camera/mic, or general network; bounded memory/command queues and restart on fault |
| Media decoder broker | Selected codec APIs and bounded shared surfaces | No arbitrary file paths, UI authority, updater, host credentials, or unrestricted network; per-stream CPU/memory/time quotas |
| Input/cursor/haptics broker | Foreground window input, virtual device protocol | No global key capture, text logging, network, user files, or persistent input history; rate/device/bitmap limits and all-up recovery |
| Clipboard broker/parser | Current clipboard allowlisted formats for active policy | No history/cloud clipboard, arbitrary URI/file dereference, network, or notification authority; strict byte/item/time limits |
| Notification broker/parser | Product-owned host notifications and guest opaque actions | No unrelated host notifications, live `PendingIntent`, arbitrary URI execution, or general network; rate/token/asset limits |
| File broker | User-selected roots/security-scoped bookmarks | No ambient home-disk access, executable launch, hidden paths, or unrelated instance access; path/symlink/lock/quota validation |
| Camera/mic/location brokers | One user-approved device/capability at a time | No background activation without visible policy, unrelated sensors, guest-chosen host handles, or persisted raw media |
| Updater | Static signed metadata/package endpoints, staged artifact install | No telemetry IDs/cookies, guest content, clipboard/input, arbitrary URL, or unsigned payload; transactional rollback only |
| Diagnostics viewer/exporter | Local bounded logs/crashes and user-selected export destination | No automatic network, background export, raw secrets by default, or modification of evidence; preview/redaction required |
| Native Bridge inspector | User-selected provider package, host malware scan, private versioned store | No network/download/search, execution before validation, general home access, or mutation of base images; strict archive/ELF limits |

Every IPC method specifies caller identity, instance/user namespace, capability token, payload schema and limit, timeout/cancellation, idempotency, version behavior, audit event, and failure cleanup. Bulk data uses bounded shared memory after metadata validation; control messages never embed unbounded blobs.

## Windows sandbox implementation

- Run the user interface per user and unelevated. Privileged installation/setup uses a narrow service boundary that does not accept general guest or UI commands.
- Use restricted tokens and AppContainer or LPAC for compatible brokers. When AppContainer cannot access a required hypervisor/GPU API, isolate that capability in the smallest possible non-AppContainer process and document the failed proof.
- Place each process tree in a Job Object with memory, CPU, process-count, handle, and termination policies appropriate to its role; collect resource accounting locally.
- Enable applicable compile/link and runtime mitigations: DEP, ASLR, CFG, CET/shadow stack on supported hardware, stack protections, and capability-gated dynamic-code/image-load policies. Test every mitigation against ANGLE/Gfxstream/QEMU and do not request broad exceptions for the entire product.
- Apply explicit named-pipe/RPC/shared-memory ACLs, handle non-inheritance by default, brokered file/device handles, and per-process Windows Firewall policy.
- Keep the renderer, media decoders, clipboard/notification parsers, Native Bridge inspector, and diagnostics exporter outside the UI/installer trust boundary and independently restartable.

## macOS sandbox implementation

- Enable Hardened Runtime and notarization for every production executable. Grant only required entitlements and maintain an executable-by-executable entitlement inventory.
- Use App Sandbox for the shell and compatible helpers. Use security-scoped bookmarks only for user-selected content, stop access promptly, and avoid broad temporary exceptions.
- Implement renderer, media, clipboard/notification, file/device, updater, diagnostics, and import roles as narrow XPC services where compatible. Validate connection identity and message bounds before granting any capability.
- Prove the exact HVF/hypervisor, Metal/GPU, JIT/dynamic-code, and service-management constraints in Phase 0. If a component cannot be App-Sandboxed, isolate its narrow entitlement in a signed least-privilege helper and document the boundary rather than disabling App Sandbox globally.
- Use App Group containers only for the minimum explicitly shared state. Userdata, update staging, diagnostics, provider payloads, and per-instance credentials remain separated with least file access.
- Use macOS memory/thermal/energy pressure notifications and on-demand XPC lifecycle to stop idle helpers and reduce both attack surface and background resource use.

## Zero-telemetry implementation

### Build and dependency policy

- Ban analytics, advertising, remote crash reporting, remote logging, feature experimentation, device fingerprinting, and session-replay SDK categories from production dependency manifests.
- Generate an endpoint inventory from source, binaries, manifests, certificates, and network policy. Every first-party hostname/IP purpose, protocol, caller process, user control, data class, retention, and offline behavior has an owner.
- Fail CI when a production artifact adds a socket-capable component or endpoint not present in the reviewed egress manifest.
- Remove server-selected A/B testing. Feature flags are signed release/channel configuration or local compatibility policy without per-device tracking.

### Functional network policy

- Updates retrieve static signed manifests and content-addressed packages without stable identifiers, tracking cookies, or per-install query parameters. Rollout uses release channels and locally deterministic policy.
- F-Droid traffic belongs to the guest client and its configured repositories; do not proxy it through a first-party analytics/catalog service.
- microG traffic exists only when its individual functions are enabled and remains identifiable in the UI/network policy as third-party compatibility traffic.
- Prefer host-provided wall/monotonic time. Connectivity checks, captive-portal probes, DNS, network time, and location backends are documented, user-configurable or disableable, and absent in an offline profile.
- User applications obey Android network policy. Offer per-app Network toggles and optional new-install default-deny templates, while explaining compatibility impact.

### Local diagnostics

- Store logs in memory or bounded rotating files with per-component size/age limits. Avoid raw key/text/controller streams, clipboard/notification content or digests, media frames, user paths, tokens, credentials, provider contents/hashes, and stable cross-install identifiers.
- Use per-run random correlation IDs that are regenerated and included only in a manual export if the user approves.
- Crash artifacts default to minidump/backtrace, component/build/version, error code, local resource state, and redacted recent events. Full memory dumps require a separate explicit developer action and warning.
- The viewer shows exactly what will be exported, supports field/file removal, creates a local archive in a user-selected location, and never transmits it.
- Support workflows instruct the user to attach the exported archive manually through their chosen channel; the product does not embed a support uploader.

## Performance and smoothness design

- Start helpers lazily and terminate or suspend them when their capability is unused. XPC/on-demand service and Windows process-pool behavior must not cause repeated visible launch stalls.
- Use bounded queues, backpressure, batching, cancellation, and work prioritization. Never let guest flood control messages starve input, audio, renderer presentation, or host UI.
- Use immutable shared data and validated shared-memory/zero-copy paths for graphics/media/bounded image payloads. Capability validation occurs before mapping or consuming bulk buffers.
- Apply Android cgroups/LMKD and host Job Object or pressure-notification policies so a guest/app/parser cannot exhaust host memory, CPU, processes, handles, or disk.
- Recycle a broker after corruption, crash, or a measured high-water mark; preserve idempotent state and never widen permissions to recover performance.
- Disabling telemetry removes background CPU, storage, wakeup, and network cost, but the product must still benchmark local logging overhead and keep release logging minimal.

Initial hardening gates relative to the same current-AOSP/product baseline:

| Metric | Maximum allowed regression |
|---|---:|
| Interactive/app-suite geomean CPU | 3% |
| Settled resident memory | 5% |
| Cold and warm app launch | 5% |
| UI jank | 1 percentage point |
| Broker control-message overhead | 1 ms p95 |
| Input and TV D-pad latency | Must remain within the main plan's absolute p95 budgets |
| Long-play media dropped frames/A-V drift | Must remain within the main plan's absolute gates |
| Idle first-party network requests | 0 telemetry; only declared enabled functional traffic |

When a mitigation misses a gate: optimize it; rerun all security and performance tests; then, only if necessary, restrict it to high-risk processes or create a visible per-app compatibility exception with owner, rationale, expiry, and regression coverage. A hidden product-wide disable is not acceptable.

## Verification matrix

- Static scans: forbidden SDKs, URLs/endpoints, socket-capable dependencies, exported IPC, entitlements/capabilities, executable-memory/JIT exceptions, signing, SBOM, and provenance.
- Network captures: clean install, first boot, idle 30 minutes, app launch, induced guest/host crash, local diagnostic export, update success/failure, suspend/resume, shutdown, F-Droid sync, and each microG opt-in state.
- Guest isolation: cross-UID/profile file/Binder/socket access, SELinux denials, seccomp violations, network/sensor revocation, dynamic-code denial, malformed intents/AIDL, resource exhaustion, snapshot restore, and update/rollback.
- Host isolation: token/entitlement denial, IPC impersonation/replay, handle/port leakage, path traversal/symlink race, parser fuzzing, firewall denial, process escape, quota exhaustion, crash restart, and cross-user/instance attempts.
- Native-shell isolation: hostile labels/icons/URIs, AppUserModelID or Spotlight identity collision, cross-user/task activation, profile/configuration replay, virtual-display exhaustion, macOS launcher-shim signature/entitlement validation, and AppShell crash/restart.
- High-risk parsers: graphics command streams, media, HTML/images/icons, archives/ELF/provider manifests, protocol schemas, update manifests, and diagnostic artifacts.
- Compatibility: Core/Compatible profiles, F-Droid, microG APIs, Native Bridge Off/Houdini/`libndk_translation`, Desktop/TV, all three host architecture pairs, and app-specific dynamic-code/network exceptions.
- Performance A/B: boot/resume, cold/warm launch, UI frame pacing, input/cursor/controller latency, audio, broker IPC, clipboard/notification latency, memory/CPU/GPU/disk/energy, TV 1080p/4K navigation, and long-play media.

## Delivery sequence

1. Freeze threat model, telemetry definition, functional-traffic taxonomy, third-party/host-OS scope, process map, and performance baselines.
2. Add production dependency bans, local diagnostics format/viewer specification, and machine-readable egress manifest.
3. Split high-risk host parsers/brokers and apply Windows/macOS least-privilege controls with negative tests.
4. Enforce guest SELinux/seccomp/service domains plus per-app Network/Sensors controls.
5. Integrate the first GrapheneOS-derived high-value/low-overhead changes with provenance and CTS/VTS gates.
6. Add update privacy, offline profile, packet-capture CI, sandbox fuzz/escape/resource tests, and automated performance A/B.
7. Expand targeted allocator/runtime hardening only where the measured security/performance decision supports it.
8. Complete independent security/privacy review, publish the exact hardening/traffic statement, and make all gates release-blocking.

## Deliverables

- Zero-telemetry product specification and signed-release checklist.
- Machine-readable first-party egress manifest per SKU plus packet-capture evidence.
- Local diagnostics schema, retention/redaction rules, viewer/export UX, and no-upload verification.
- GrapheneOS audit ledger and bounded patch queue with provenance, current-AOSP comparison, test evidence, and exclusions.
- Guest SELinux/seccomp/Network/Sensors/dynamic-code/memory-hardening policy and compatibility matrix.
- Windows process/token/AppContainer/Job Object/mitigation/firewall/IPC matrix.
- macOS executable/App Sandbox/Hardened Runtime/XPC/entitlement/container matrix.
- Native AppShell/launcher identity threat model covering Windows AppUserModelID/shortcuts and the macOS signed-shim or single-Dock fallback.
- Broker protocol threat models, fuzz corpora, quotas, lifecycle/recovery tests, and independent review findings.
- Cross-SKU security/performance dashboard containing local CI results, not user telemetry.

## Exit criteria

- Production binaries and guest images contain no analytics/crash-upload/remote-log/experiment SDK or automatic diagnostics transmission path.
- Runtime captures show zero undocumented first-party destinations and zero telemetry requests across the complete verification matrix.
- Signed updates work without stable device/install identifiers or tracking cookies, and the offline/enterprise path passes update/rollback tests.
- Users can view, redact, and manually export bounded local diagnostics; induced crashes and export creation cause no network transmission.
- AOSP remains canonical; every GrapheneOS-derived change is provenance-tracked, measured, removable, and free of Pixel-specific/service/branding dependency.
- Guest UID/profile, SELinux, seccomp, Binder, file, Network/Sensors, dynamic-code, and resource boundaries pass abuse and CTS/VTS gates.
- Windows and macOS process/capability matrices are enforced on every shipping executable; no helper has an undocumented entitlement, token privilege, file/device grant, or egress permission.
- Renderer, media, clipboard/notification parsers, update parser, diagnostics exporter, and Native Bridge inspector pass fuzz, quota, crash-restart, and sandbox-escape review with no unresolved critical/high finding.
- All hardening changes meet the CPU, memory, app-launch, jank, broker-latency, input, and TV/media performance gates, or have a narrowly scoped approved exception with owner and expiry.
- Documentation clearly separates first-party zero telemetry, optional functional services, third-party app traffic, and host-OS behavior without overclaiming GrapheneOS equivalence.
