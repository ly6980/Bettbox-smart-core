# Mihomo Smart Core Integration Notes

## Goal

Add an optional Mihomo Smart core to Bettbox without changing the default core or the `main` branch.

## Important architecture difference

Bettbox does not manage Mihomo as a replaceable sidecar binary in the same way Clash Party does.

- Bettbox's `core/go.mod` replaces `github.com/metacubex/mihomo` with the local source tree `./Clash.Meta`.
- `setup.dart` builds Bettbox's Go core into platform-specific artifacts (`BettboxCore` on desktop and a C shared library on Android).
- Runtime core paths currently resolve to the bundled Bettbox core executable/library name.

Clash Party's `smart_core` branch instead stores an external core in its sidecar directory and resolves `mihomo-smart` to a distinct executable. Its Smart integration also manages a generated Smart override and related settings. See:
- https://github.com/mihomo-party-org/clash-party/blob/smart_core/src/main/utils/dirs.ts
- https://github.com/mihomo-party-org/clash-party/blob/smart_core/src/main/config/smartOverride.ts
- https://github.com/mihomo-party-org/clash-party/blob/smart_core/src/main/core/manager.ts

## Recommended implementation strategy

1. Keep the current `core/go.mod` and `core/Clash.Meta` path as the stable core.
2. Add a separate Smart-core Go module file (for example `core/go.smart.mod`) that resolves the same Mihomo module path to a pinned Smart fork commit. Do not change the stable module's replacement.
3. Teach `setup.dart` to build a second artifact using the Smart module file, with a distinct name. The stable core remains the default.
4. Add a runtime core selection setting and make startup, version/hash checks, helper-service startup, and platform-specific library loading use the selected artifact consistently.
5. Add a Smart-specific configuration override only when Smart core is selected. Make it opt-in and removable, following Clash Party's generated-override lifecycle; never rewrite user config unconditionally.
6. Start with Windows desktop, validate the build and startup/config validation, then extend to Linux/macOS. Android needs separate validation because Bettbox uses a C shared library there rather than a standalone executable.

## Source and compatibility cautions

- Clash Party's code is an architectural reference, not code to copy wholesale: it is an Electron application while Bettbox is Flutter plus a Go core.
- Pin a specific Smart-core commit instead of tracking a moving branch, and record the upstream URL/commit so builds are reproducible.
- Verify Smart-core configuration fields and supported platform assets against the selected Smart fork before enabling any automatic override.
- Do not claim the integration is complete until the Smart artifact builds and can start with a real profile; a successful Dart formatting check alone is insufficient.
