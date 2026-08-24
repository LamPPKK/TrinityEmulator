# Phase 5A — Clipboard and Notification Synchronization

Duration: 12–18 weeks overlapping late Phase 4 and Phase 5  
Purpose: Synchronize the current clipboard and Android notification lifecycle between guest and host without clipboard loops, cross-instance leaks, stale actions, broad host surveillance, or content-bearing logs/diagnostic exports.

## Shared protocol and process boundaries

- Add versioned `ClipboardBridge` and `NotificationBridge` schemas over the authenticated control plane. Large approved image/icon payloads use the bounded bulk plane with length, type, digest, ownership, cancellation, and timeout metadata.
- Run clipboard format conversion and notification icon/media decoding in low-privilege brokers. The native UI and VM controller receive only validated normalized models or native-safe handles; they do not parse guest HTML, images, icons, or arbitrary bundles.
- Namespace every message by host user, instance UUID, Android user/profile, protocol session generation, and monotonic revision. A broker restart invalidates in-flight capabilities and triggers state reconciliation rather than replaying stale events.
- Persist policy, mappings, and bounded redacted notification display state. Do not persist clipboard history, clipboard content hashes, live Android action authority, `PendingIntent`, raw `RemoteViews`, host-native handles, or guest content/file URIs.
- Define capability negotiation for formats, size limits, actions, direct reply, dismissal observation, grouping, progress, icons, privacy levels, and TV behavior. Unsupported fields degrade visibly and safely.

## Clipboard data model and flow

- Represent the current clipboard as one `ClipEnvelope`: origin, origin instance, source revision, timestamp, content digest used only in memory, Android remote/sensitive hints, ordered items, and allowlisted MIME representations.
- Developer Preview supports UTF-8/UTF-16 normalized plain text. Public Beta adds inert sanitized HTML with a plain-text fallback and bounded PNG images. Preserve multiple items only where both endpoints support them; otherwise use the first safe representation and report the downgrade in diagnostics.
- Do not transport Android `Intent`, arbitrary `content://`/`file://` references, Windows shell object handles, macOS security-scoped URLs, promised files, or delayed provider callbacks across the protocol. Files and large objects go through the existing user-mediated file broker/drag-and-drop path.
- Enforce configurable hard limits for item count, text length, HTML bytes/tree depth, image compressed/decompressed bytes, dimensions, conversion time, and total envelope size. Reject rather than truncate a structured payload into misleading content; retain a safe text fallback when one exists.

## Android clipboard agent

- Implement a platform-signed, signature-permission and SELinux-confined `HostClipboardService`. Public `ClipboardManager.getPrimaryClip()` is unavailable to ordinary unfocused apps; do not weaken that platform restriction or grant background clipboard access to third-party packages.
- Listen for primary-clip changes and snapshot the current allowed representations exactly once per accepted revision. Android text classification can trigger duplicate callbacks, so compare source metadata and the in-memory content digest before exporting.
- Mark host-origin clips with `ClipDescription.EXTRA_IS_REMOTE_DEVICE` and an internal session/revision marker. Preserve MIME labels and safe metadata, but never trust an app to retain those markers; digest/revision acknowledgement remains the loop-prevention authority.
- Treat `ClipDescription.EXTRA_IS_SENSITIVE` as a mandatory no-auto-export signal. The host receives only a redacted “sensitive clipboard blocked” state if UI feedback is enabled. A future one-time explicit transfer requires a separately reviewed user-presence flow.
- Clear imported transient data and URI grants on guest shutdown, user switch, or bridge disable. Clipboard data must never cross Android users/profiles without an explicit cross-profile product policy and Android enterprise review.

## Windows clipboard adapter

- Register a message-only or broker-owned HWND with `AddClipboardFormatListener`; react to `WM_CLIPBOARDUPDATE` instead of polling or using the fragile legacy viewer chain.
- Handle temporary clipboard lock contention with a bounded retry/backoff. Copy accepted data before closing the clipboard, respect delayed rendering/owner exit, and perform conversion outside the UI thread.
- Map `CF_UNICODETEXT` to plain text, reviewed CF_HTML to sanitized HTML, and approved PNG/DIB representations to bounded PNG. Treat `CF_HDROP`, owner-display, private, OLE, executable, and unknown formats as unsupported until their brokered security model is approved.
- Record the sequence/revision written by Trinity and ignore the matching listener callback. Re-read and compare current clipboard sequence/ownership after delayed rendering or competing writes; never overwrite a newer clipboard.
- Do not access Windows clipboard history or Cloud Clipboard APIs. The product handles only the current clipboard exposed through the documented Win32 clipboard contract.

## macOS clipboard adapter

- Use `NSPasteboard.general`, record `changeCount`, and inspect/convert only when ownership changes. Because AppKit does not provide the same event listener as Windows, use an adaptive observer only while clipboard sync and an instance are active; back off when idle/backgrounded.
- Check and respect `NSPasteboard.accessBehavior` and the user's per-app pasteboard choice. Permission denial or restricted behavior disables that direction with a visible, recoverable state; do not bypass it with Accessibility, automation, or event-tap permissions.
- Map plain strings, safe HTML/RTF-derived text, and bounded PNG/TIFF-derived images through allowlisted UTTypes. Avoid reading promised files, arbitrary serialized objects, custom pasteboard types, or security-scoped URLs automatically.
- NSPasteboard already participates in Apple Universal Clipboard according to system policy. LeapDroid neither controls nor claims to synchronize that service and must not create a second history or cross-device layer.

## Clipboard arbitration and loop prevention

- Default host-to-guest destination is the focused Desktop instance. TV and unfocused instances do not receive host changes unless explicitly selected. Only an enabled running instance may publish guest-to-host changes.
- Use origin + session generation + source revision + in-memory digest + acknowledgement as the loop key. Never rely only on content equality: copying the same text intentionally at a later time is a new user action.
- Last accepted writer wins at the host broker. Serialize competing guest/host events, retain causal order, and reject late acknowledgements or updates from a previous broker/guest generation.
- Clipboard clear remains local unless the target still contains the exact bridged revision being cleared. This prevents an Android clear or lifecycle cleanup from erasing newer host content.
- Disabling sync stops observation, zeros transient buffers, invalidates revisions, and does not clear either side. Re-enabling does not copy historical content until the next change or an explicit “sync current clipboard” action.

## Android notification agent

- Implement a product-owned `NotificationListenerService` enabled only after an explicit Android notification-access grant and matching host setting. On connect/reconnect, fetch active notifications and reconcile a complete snapshot before applying deltas.
- Normalize stable notification key, package/user, ID/tag/group/channel, revision, post time, app label/icon, bounded title/text/subtext, public version, category, importance, silent state, progress, grouping/summary, flags, and supported actions.
- Exclude notifications marked `FLAG_LOCAL_ONLY`. Suppress `VISIBILITY_SECRET`; for `VISIBILITY_PRIVATE`, send the public version or a redacted representation according to preview policy. Never flatten or execute custom `RemoteViews` on the host.
- Retain content, delete, full-screen, and action `PendingIntent` objects inside Android. Generate short-lived opaque capabilities bound to the active notification key/revision, Android user, action index, authentication rule, guest boot generation, and expiry.
- When host activation arrives, re-read the active notification, compare revision/action semantics, validate user/instance/unlock policy, then invoke within Android. One-time/replay-protected tokens fail after success, expiry, update, removal, reboot, or bridge reconnect.
- Translate bounded text replies back into the original Android `RemoteInput` result. Do not execute a reply or destructive/authentication-required action while the relevant Android user is locked; resume the UI and require guest authentication.

## Host notification adapters

- Map Android notification keys to stable host identifiers. Reposting the same revision is idempotent; a higher revision replaces the current host representation; guest removal removes the mapped host notification.
- Windows uses Windows App SDK `AppNotificationManager`, stable tag/group/ID, actionable buttons/text input, `NotificationInvoked`, `GetAllAsync`, and targeted removal. Enumerate only Trinity's own notifications; do not request the Windows `UserNotificationListener` all-notification capability.
- macOS uses `UNUserNotificationCenter`, stable request identifiers, registered categories, `UNNotificationAction`, `UNTextInputNotificationAction`, `.customDismissAction`, delivered-notification enumeration, replacement, and targeted removal.
- Both platforms reconcile their own delivered set after startup/resume, guest reconnect, and while bridged notifications exist. An absent clearable notification that Android still considers active is treated as host dismissal only after excluding adapter-origin removal, expiry, replacement, and reconciliation races.
- Explicit dismiss cancels a clearable Android notification through the guest service. Ongoing or non-clearable notifications may be hidden locally but are not cancelled; explain this limit and allow them to reappear on meaningful guest updates.
- Notification click opens/resumes the owning instance and task only after live-key validation. If the VM is stopped, start it under the normal lifecycle policy, reconcile, and discard the activation if the notification no longer exists.
- Use the Trinity or LeapDroid host app identity as the OS notification source. Show the Android app label/icon inside supported fields and group by app/channel/conversation where possible; never register deceptive native identities for each APK.

## Notification policy, flood control, and TV behavior

- Provide master, instance, app, and Android-channel forwarding toggles plus preview, sound, actions/reply, dismissal sync, wake-on-activation, and lock-screen policy. Notification access revocation immediately removes actionable authority and optionally clears forwarded host entries.
- Respect host Focus/Do Not Disturb and notification authorization. Do not rewrite Android channel importance or guest DND to force delivery; reflect suppressed/denied status in settings and diagnostics.
- Rate-limit per package/channel and globally. Collapse repeated progress updates, group bursts, cap outstanding notifications and icon cache, and quarantine a package that repeatedly sends malformed or oversized content without blocking other apps.
- TV defaults to forwarding off or redacted summaries with an explicit allowlist. Provide a D-pad-operable notification center/host overlay; do not interrupt video/gameplay for ordinary notifications. Clipboard is off by default and text-only when enabled.
- Separate policies and keys by host user, Android user/profile, instance, and Desktop/TV edition. Never leak notification content or actions between profiles or reuse a token after user switch.

## Security and privacy

- Threat-model malicious guest apps, compromised system services, HTML/image/icon parsers, forged notification capabilities, replay, notification floods, clipboard bombs, cross-instance races, host lock/privacy, and crash recovery.
- Sanitize Unicode controls/bidirectional text for host display without changing user-visible meaning silently; preserve original text only inside the bounded trusted transaction. Validate URLs before host launch and keep Android intents opaque.
- Do not log clipboard/notification text, image bytes, reply content, content digests, action arguments, or PendingIntent data. Metrics include only format/class, byte bucket, timing, redaction reason, lifecycle transition, rate-limit count, and error code.
- Clear clipboard buffers and notification action tokens on policy disable, Android user switch, guest reset, update, snapshot restore, host logout, or broker crash. A restored snapshot must never revive a pre-snapshot action token.
- Fuzz clipboard schemas, MIME converters, HTML sanitizer, image/icon decoders, notification models, action/reply endpoints, reconciliation state machines, and malformed bulk transfers. Run rich-content decoders with memory/CPU/time quotas.

## Performance and observability

- Target running-instance plain-text propagation under 500 ms p95 on Windows and under 1.5 seconds p95 on macOS. Measure observation, snapshot, transport, guest/host set, and listener-echo acknowledgement separately.
- Target notification post/update/remove to host under 1 second p95 and warm action/reply dispatch under 2 seconds p95. Report guest cold-start/resume separately rather than hiding it in bridge latency.
- Trace revisions, state transitions, sizes/buckets, queue delay, conversion time, reconciliation results, and redaction/rate outcomes using opaque correlation IDs. No trace field may reconstruct content.
- Bound clipboard observation energy on macOS and notification reconciliation work on both hosts; brokers sleep when disabled, no instance runs, or no bridged state needs reconciliation.

## Validation matrix

- **Clipboard content:** empty, ASCII, Vietnamese, emoji, RTL/bidirectional controls, very long Unicode, multiple items, HTML with script/style/external URLs, malformed HTML, PNG/TIFF/DIB, decompression bombs, unsupported/private formats, delayed rendering, and owner termination.
- **Clipboard lifecycle:** rapid alternating writes, identical content copied twice, Android classification duplicate callbacks, active-instance switch, two/three running instances, focus loss, disable/re-enable, guest/host restart, update, suspend/snapshot restore, host logout, and clear races.
- **Clipboard privacy:** Android sensitive/remote flags, macOS access denial, per-direction/app policy, Android users/work profiles, host users, TV defaults, local log/export inspection, and memory zeroing.
- **Notifications:** post/update/remove, same ID with new revision, groups/conversations, progress storms, silent/importance, local-only, public/private/secret, custom RemoteViews fallback, app/channel blocks, DND, permission revocation, icon failures, and host center cleanup.
- **Actions:** open, Back/cold start, clearable dismiss, non-clearable/ongoing dismiss, reply, multiple actions, destructive/authentication-required actions, stale/expired/replayed/forged tokens, update-before-click, remove-before-click, Android user lock/switch, and guest reboot.
- **Platforms/editions:** Windows x64/ARM64 and multiple Apple Silicon generations; Desktop and TV; Core and Compatible/microG; host restart, OS update, guest update, snapshot restore, clock changes, and notification/clipboard broker crash injection.

## Exit criteria

- Plain text, sanitized HTML, and bounded PNG synchronize according to the release tier without ping-pong, content reordering, stale overwrite, cross-instance leakage, or unintended clear across 100,000 randomized state-machine operations.
- Android sensitive clips are never automatically exported; unsupported URIs/files remain inaccessible; malformed rich formats cannot crash or escape the broker; no clipboard content/history/digest appears in logs or diagnostic exports.
- Notification post/update/remove/open, supported action, direct reply, and clearable-dismiss flows reconcile on Windows and macOS. Ongoing, local-only, private/secret, and authentication-required behavior matches documented policy.
- No host path requests access to unrelated Windows notifications, stores a live Android `PendingIntent`, replays an old action after update/reboot/snapshot, or crosses host/Android user boundaries.
- Permission denial/revocation, DND, offline guest, cold start, broker crash, rate flood, update, and rollback recover to a consistent state without losing host control or exposing stale content.
- Performance and energy budgets pass on every committed host architecture, and TV defaults remain D-pad-operable, redacted, and non-disruptive.
