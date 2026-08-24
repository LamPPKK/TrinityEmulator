# Trinity and LeapDroid Development Plan

This directory is the canonical development plan for:

- Trinity Desktop and Android TV on Windows x64 and ARM64.
- LeapDroid Desktop and Android TV on macOS ARM64.

Start with [the overall plan](plan.md). Detailed execution and release gates are split into these phase documents:

1. [Phase 0 — architecture and feasibility gates](phase-0-decision-gates.md)
2. [Phase 1 — shared runtime foundation](phase-1-runtime-foundation.md)
3. [Phase 2 — modern Android guest](phase-2-android-guest.md)
4. [Phase 2A — Android TV editions](phase-2a-android-tv.md)
5. [Phase 2B — NoApp, microG, and user-installed GApps modes](phase-2b-service-modes-gapps.md)
6. [Phase 2C — KernelSU and Magisk root providers](phase-2c-root-providers.md)
7. [Phase 3 — graphics compatibility and performance](phase-3-graphics.md)
8. [Phase 4 — seamless application windows](phase-4-seamless-windows.md)
9. [Phase 4A — input, cursor, and controllers](phase-4a-input-cursor-controller.md)
10. [Phase 4B — native app experience and display profiles](phase-4b-native-app-experience-display-profiles.md)
11. [Phase 5 — native host integration](phase-5-host-integration.md)
12. [Phase 5A — clipboard and notifications](phase-5a-clipboard-notifications.md)
13. [Phase 6 — security, updates, compliance, and licensing](phase-6-security-update-compliance.md)
14. [Phase 6A — zero telemetry and layered sandboxing](phase-6a-zero-telemetry-sandbox.md)
15. [Phase 7 — hardening, beta, and general availability](phase-7-hardening-release.md)

Keep architecture decisions, scope changes, release gates, and cross-repository protocol changes synchronized through this canonical copy to avoid separate Trinity and LeapDroid plans drifting apart.
