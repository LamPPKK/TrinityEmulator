# Phase 3 — Graphics Compatibility and Performance

Duration: 16–24 weeks, overlapping late Phase 2  
Purpose: Deliver a maintained accelerated graphics stack before optimizing the Trinity projection path.

## Compatibility backend

- Integrate current Gfxstream guest libraries, transport, host renderer, synchronization, buffer sharing, and Vulkan support.
- Host Gfxstream inside a sandboxed GPU process with restart and device-loss recovery.
- Use ANGLE D3D11 on Windows as the initial default and ANGLE Metal on macOS.
- Evaluate D3D12, Vulkan, and adapter-specific paths behind runtime feature flags.
- Provide SwiftShader recovery and CI modes with identical protocol behavior.

## Surface and composition model

- Export Android task surfaces without forcing the entire guest display through a single framebuffer.
- Define explicit buffer ownership, color space, alpha, transform, crop, damage, fence, and lifetime semantics.
- Support density/orientation changes, multiple display areas, picture-in-picture, protected-content policy, and host window occlusion.
- Support simultaneous per-task display/configuration surfaces selected by Phase 4B, including adaptive resize and fixed Phone/Tablet canvas scaling/letterboxing without an extra full-frame CPU copy.
- Avoid CPU readback on the normal presentation path.
- Support both presentation contracts: per-task surfaces for Desktop and one composed guest display for TV. They share buffer/fence/color primitives but never switch modes within an instance.
- Add TV resolution/refresh switching, full-screen monitor moves, SDR color correctness, decoder-surface import, and 4K memory/bandwidth accounting. Keep HDR behind a separate conformance flag.

## Trinity projection migration

- Write a protocol specification from observable current behavior rather than copying undocumented assumptions.
- Reimplement guest shadow contexts and resource handles against the current GLES/Vulkan stack only after compatibility tests exist.
- Replace fixed call buffers, manual pointer access, and unbounded scatter/gather handling with generated bounded codecs and explicit memory ownership.
- Keep the projection backend disabled per app until its output and API behavior match the compatibility backend.
- Support runtime fallback from projection to compatibility on restart, never mid-command-stream without a defined recovery point.

## Validation

- ANGLE and Gfxstream unit/end-to-end suites.
- dEQP GLES 2/3/3.1/3.2 as supported and Vulkan CTS subsets.
- Golden-image and API-state differential tests across ANGLE backends and SwiftShader.
- TV 1080p/4K frame-pacing, color, refresh transition, video-overlay/composition, decoder reset, subtitle, and long-play tests.
- GPU reset, driver update, sleep/wake, multi-monitor, color/HDR policy, and long-run resource-lifetime tests.
- Fuzz command decoders and shader inputs in isolated processes.

## Exit criteria

- Compatibility backend runs the blocking application suite on all host targets.
- No normal frame path requires full-frame CPU copies.
- GPU-process crashes do not corrupt userdata or crash the native shell.
- SwiftShader recovery is automatic and clearly reported.
- Trinity projection ships only if it meets the same correctness gates and demonstrates a meaningful measured improvement.
- Desktop and TV presentation modes pass independently on every host graphics backend; TV can recover from GPU/decoder loss without losing userdata or requiring host restart.
- Ten native AppWindow/NSWindow surfaces with mixed Auto/Phone/Tablet profiles meet memory, resize-to-frame, DPI/Retina, letterbox transform, input-alignment, and crash-recovery gates.
