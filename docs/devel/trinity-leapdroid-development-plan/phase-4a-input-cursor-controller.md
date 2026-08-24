# Phase 4A — Keyboard, Mouse, Cursor, and Game Controllers

Duration: 14–20 weeks, beginning when the Phase 1 input protocol and Phase 4 focus/task routing are stable  
Purpose: Deliver one low-latency, recoverable, standards-based input stack for Trinity Windows x64/ARM64 and LeapDroid macOS ARM64 across Desktop and TV editions.

## Architecture and ownership

- Define one normalized `HostInputDevice` model for keyboards, pointing devices, controllers, and remotes. Every device has a connection ID, capability descriptor, host timestamp domain, focus/capture owner, lifecycle state, and destination instance/display/task.
- Use a separate `virtio-input` device for each logical keyboard, absolute pointer/tablet, relative pointer, and game controller. Transport standard Linux evdev events over the virtio event queue so Android uses its normal kernel input, EventHub, InputReader, InputDispatcher, key-layout, and key-character-map path.
- Use the versioned control protocol—not a second high-rate event stream—for IME composition, cursor type/bitmap/hotspot/visibility, pointer-capture negotiation, device battery/connection metadata, player assignment, and haptics.
- Keep exactly one authoritative host reader per physical device class. Windows keyboard/mouse events originate from Raw Input and controller events from GameInput. macOS keyboard/mouse/text originate from AppKit and controllers from GameController.
- Treat direct USB/Bluetooth passthrough as out of scope. Pairing, permissions, drivers, and transport remain host responsibilities; Android receives virtual devices with accurately declared capabilities.
- Write protocol versioning, maximum sizes, reset behavior, fallback behavior, and old-host/new-guest compatibility before implementation. Unknown capabilities are ignored safely; incompatible mandatory semantics fail closed.

## Keyboard and host IME

- Normalize Windows scan codes and macOS hardware key codes to Linux evdev codes while preserving left/right modifiers, keypad distinction, function/media keys, lock state, press/release, repeat, and device identity.
- Maintain two non-duplicating paths:
  - **Physical keys:** deliver key state through virtio-input for games, Android shortcuts, key-navigation, and apps that handle `KeyEvent` directly.
  - **Composed text:** deliver host pre-edit text, selection, replacement range, committed Unicode, and cancellation to a signed guest `HostInputMethod` service that updates the currently focused Android `InputConnection`.
- On Windows, register Raw Input only for the focused product window and integrate text through a TSF/CoreText-compatible native text client. Use `GetRawInputBuffer` where profiling proves high event volume. Do not register an ordinary UI for background keyboard capture.
- On macOS, use the focused `NSResponder`/`NSEvent` path plus `NSTextInputClient` for marked text, ranges, commands, and commit. Do not require a global event tap, global monitor, or Accessibility permission for normal operation.
- Freeze a shortcut precedence table. OS-reserved combinations stay on the host; release-capture/recovery and configured product commands are host-owned; all other physical keys are routed to Android. UI must expose conflicts before a shortcut is saved.
- Select one repeat owner and encode repeat count explicitly. During host composition, suppress the corresponding physical character production while still preserving navigation/modifier events required by the IME.
- Atomically synthesize key-up for every held key on focus loss, instance switch, capture release, host sleep, guest reset, or device removal. Lock state is resynchronized without inventing printable characters.

## Mouse, coordinates, capture, and cursor

- Present an absolute pointer/tablet device during ordinary use and a relative mouse device after Android requests pointer capture and the host grants it. Support primary, secondary, middle, back, forward, vertical wheel, and horizontal wheel.
- Maintain one tested transform from host client coordinates to Android logical display coordinates, including window chrome, letterbox/crop, safe area, DPI, fractional scale, orientation, density, and multi-monitor transitions. Clamp outside motion predictably and never transform relative deltas through display scaling.
- Bind that transform to the active Phase 4B presentation-profile and surface revision. Drop stale events after Auto/Phone/Tablet changes or task recreation instead of applying compact/tablet coordinates to a replacement canvas.
- Implement Windows mouse input through foreground Raw Input; use buffered reads and event coalescing for high-polling devices. Implement macOS pointer input through window-scoped AppKit events and a small pointer-lock adapter selected by the platform spike.
- Pointer capture requires a visible focused surface and an explicit click/action. While captured, hide the native cursor, deliver raw deltas, retain button/wheel order, and keep a host-owned escape shortcut. Focus loss, window destruction, guest timeout, app crash, or recovery overlay immediately releases capture.
- Make the native host cursor the display authority. Add a guest cursor agent at the Android input/window boundary to export resolved `PointerIcon` type, custom bitmap, hotspot, visibility, and target surface. Map standard Android icons to native Windows/macOS cursors and cache bounded custom cursor assets.
- Suppress the guest-rendered pointer cursor when host cursor mode is active. If cursor metadata negotiation fails, fall back to a documented default host cursor; do not render both cursors. During relative capture, no visible Android or host cursor is drawn.
- Coalesce movement only when doing so preserves the last coordinate/delta and never crosses a key/button/wheel, focus, capture, device, or timestamp-order boundary. Keep queues bounded, expose dropped/coalesced counts, and test 125, 500, 1,000, and 8,000 Hz devices.
- TV mode hides the mouse cursor after inactivity and restores it on movement. Mouse movement does not silently replace the current D-pad/controller navigation owner until an actual pointer action occurs.

## Game controllers and remotes

- Use GameInput as the Windows controller backend and GameController as the macOS backend. Keep XInput behind a compatibility flag only for a controller that demonstrably fails GameInput; deduplicate it by stable device identity before publishing events.
- Normalize D-pad, face buttons, shoulders, triggers, stick clicks, View/Menu/Options-equivalent buttons, two sticks, hats, and available system controls into a canonical descriptor. Map these to Android standard gamepad key codes and `MotionEvent` axes/ranges.
- Support at least eight simultaneous controllers. Assign stable player slots across temporary disconnect/reconnect when the device exposes a usable identity; otherwise preserve the slot only for the connection lifetime and make reassignment visible.
- Calibrate center, range, inversion, trigger rest value, and dead zones per device/mapping version. Ship reviewed mappings with provenance and version them separately from the guest image; let users reset overrides safely.
- Publish hot-plug and removal through the virtual device lifecycle. Removal first releases all buttons/axes, stops haptics, detaches the virtio device, and then notifies UI/state services.
- Send guest vibration requests to a low-privilege host haptics broker. Validate target device, foreground ownership, supported motors/channels, amplitude, duration, update rate, and total duty cycle. Stop effects on focus loss, disconnect, suspend, reset, or timeout.
- Report battery and connection type to product UI and, only where Android has a stable semantic mapping, to the guest. Keep LEDs, motion sensors, controller touchpads, adaptive triggers, wheels, flight sticks, and vendor-specific force feedback capability-gated until separate mappings and tests exist.
- For TV, maintain a navigation-owner state machine. The last intentionally active eligible remote/controller may own SystemUI focus, but every controller remains available to the foreground game. Host Home/Back/recovery actions cannot be trapped by the guest.

## Desktop and TV policy

- Desktop routes keyboard/mouse/controller input to the focused Android task window. A task switch performs release-all before the new task receives input. Background Android tasks receive no physical input unless a separately approved accessibility feature requires it.
- TV routes all devices to the one composed Android display and arbitrates SystemUI navigation centrally. Provisioning, permission, update, recovery, F-Droid, and Settings flows must work with D-pad only.
- Store mappings and shortcut preferences per host user and edition, not inside mutable app data. Migration validates schema/version and falls back to safe defaults on error.
- Do not include key-to-touch overlays, macros, rapid fire, scripted input, controller-to-host virtual drivers, or anti-cheat workarounds in first GA. They require a separate product, security, accessibility, and game-policy decision.

## Security, privacy, and recovery

- Accept events only from the active host session and route them only to the focused visible instance, except for explicit host recovery commands. Never collect input system-wide to improve warm-start behavior.
- Do not log committed text, pre-edit contents, keys, mouse paths, or raw controller streams. Diagnostics may record device class/capabilities, timing distributions, state transitions, and numeric drop/coalesce counts after privacy review.
- Bound device count, custom cursor dimensions/bytes, composition length, mapping size, event queue depth, haptic duration/rate, and message frequency. Fuzz every guest-controlled cursor/haptics decoder and malformed virtual-device capability set.
- Keep cursor decoding and haptics outside the privileged UI/VM controller. Validate custom cursor pixels in a sandboxed broker and pass only a bounded native cursor object to the UI process.
- A host watchdog owns the emergency release path. It must restore the native cursor, release confinement/capture, synthesize all-up, stop haptics, and expose recovery even when Android, the renderer, or the input broker is hung.

## Performance and observability

- Use one monotonic timebase conversion per host and propagate capture timestamps through host normalization, virtio enqueue, guest receive, InputReader, and InputDispatcher trace points.
- Target host capture-to-virtio enqueue under 4 ms p95 and guest receive-to-Android dispatch under 8 ms p95 on reference hardware. Measure click/button-to-present separately with a high-speed or hardware-assisted latency rig.
- Keep host cursor updates off the Android frame round trip. Measure pointer smoothness, late/lost events, queue occupancy, coalescing, controller polling cadence, and haptic command delay without recording input content.
- Establish bounded behavior for 8,000 Hz mice and event storms: motion may coalesce, but key/button transitions, wheels, focus/capture boundaries, and final position cannot disappear or reorder.

## Validation matrix

- **Keyboard/IME:** ANSI/ISO/JIS hardware; US, UK, French AZERTY, German, Vietnamese, Japanese, and Chinese input; dead keys, AltGr, emoji, function/media/numpad keys, locks, repeat, shortcuts, clipboard commands, focus transfer, disconnect, and suspend/resume.
- **Mouse/cursor:** 125/500/1,000/8,000 Hz; buttons 1–5; horizontal/vertical wheels; absolute/relative mode; capture/release; DPI and fractional scaling; rotation; letterboxing; multiple monitors; standard and custom cursors with non-zero hotspots; guest/renderer crash while captured.
- **Controllers:** current Xbox, DualShock/DualSense, Switch Pro, 8BitDo and generic HID devices; wired and Bluetooth; 1–8 players; non-centered axes; trigger variants; reconnect/reorder; battery; haptics; unsupported capabilities; focus/suspend/reset during active vibration.
- **TV:** remote/D-pad-only provisioning and recovery; navigation-owner changes; media keys and push-to-talk coexistence; controller input in games; cursor inactivity; host Home/Back/recovery under a misbehaving app.
- Run the matrix on Windows x64, Windows ARM64, and at least two Apple Silicon generations, with Desktop and TV product images and both 60 Hz and high-refresh displays where available.

## Exit criteria

- Physical keys and host-composed text never duplicate characters in the golden application/IME suite; all supported layouts and composition languages pass committed-text and cancellation tests.
- Absolute pointer mapping is pixel-correct for every supported orientation/scaling profile; relative capture passes 1,000 Hz latency/order gates and safely degrades at 8,000 Hz without an unbounded queue.
- Android cursor type, visibility, bitmap, and hotspot stay synchronized with the native host cursor with no persistent double cursor. Every crash/focus-loss path restores a usable host pointer.
- At least eight controllers can connect, receive stable slots, drive Android games, disconnect/reconnect, and stop all input/haptics cleanly. The published controller matrix identifies any capability-specific limitations.
- Desktop focus isolation and TV navigation ownership pass automated fault injection; no background window receives ordinary keyboard or game input.
- Security review finds no host-global key capture, unbounded guest-controlled allocation, privileged cursor decoder, stuck capture, or haptics path that survives lifecycle teardown.
- Mixed Auto/Phone/Tablet windows preserve exact content/letterbox transforms through live resize, orientation/profile changes, task recreation, rollback, and monitor DPI/Retina moves.
- Latency targets pass on the reference hardware matrix and regression dashboards identify host capture, transport, guest dispatch, and frame-presentation costs separately.
