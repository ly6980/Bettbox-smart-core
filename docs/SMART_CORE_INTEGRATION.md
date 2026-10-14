# Mihomo Smart Core Integration Notes

## Goal

Add an optional Smart Core to Bettbox while preserving the default core and keeping `main` untouched. Upstream Bettbox and Smart Core updates should be synchronized by automation; our custom integration should remain a small, replayable patch.

## Verified Smart Core source

Clash Party's `smart_core` branch downloads prebuilt Smart binaries from the `Prerelease-Alpha` release of **https://github.com/vernesong/mihomo**. This is the Smart-enabled Mihomo fork to use, not the ordinary `MetaCubeX/mihomo` release and not an unrelated fork.

Release metadata and asset names were checked on 2026-10-14:
- Release: https://github.com/vernesong/mihomo/releases/tag/Prerelease-Alpha
- Windows x64 assets include `mihomo-windows-amd64-v3-alpha-smart-<commit>.zip` and `mihomo-windows-amd64-v1-alpha-smart-<commit>.zip`.
- Windows ARM64 assets use `mihomo-windows-arm64-alpha-smart-<commit>.zip`.
- Assets are rolling prerelease builds; do not hardcode the current short commit suffix or guess a URL. The Windows build workflow resolves the current release metadata, selects the matching asset, and verifies the archive SHA-256 against `checksums.txt` before extracting it.
- Bettbox's Windows x64 build selects the v3 asset by default and the v1 asset for its compatibility build; ARM64 uses the ARM64 asset.

Clash Party references:
- https://github.com/mihomo-party-org/clash-party/blob/smart_core/scripts/prepare.mjs
- https://github.com/mihomo-party-org/clash-party/blob/smart_core/src/main/utils/dirs.ts
- https://github.com/mihomo-party-org/clash-party/blob/smart_core/src/main/config/smartOverride.ts
- https://github.com/mihomo-party-org/clash-party/blob/smart_core/src/main/core/manager.ts

## Bettbox-specific constraints

- Bettbox currently builds its normal core from `core/Clash.Meta` via `setup.dart`; desktop uses `BettboxCore[.exe]`, while Android uses a shared library. Do not replace or rewrite the existing core module.
- On Windows, the privileged helper service validates the requested executable's SHA-256 against the compile-time `TOKEN` before launching it. Simply adding a Smart executable path will fail that validation (and may fall back to direct, non-helper startup). The integration must explicitly authorize the exact Smart binary hash while preserving the helper's allowlist/security check.
- Windows packaging recursively copies the build's `runner/Release` directory, so a separately named Smart executable placed there can be bundled without renaming or replacing `BettboxCore.exe`.
- The current core priority-setting calls use the normal core process name. If Smart is selected, priority management and installer shutdown handling must target the selected executable as well.
- Bettbox's current core hash is also used by its Windows helper build process; after downloading the Smart asset, compute its hash and pass both allowed hashes to the helper, or implement an equivalent explicit hash allowlist. Do not disable hash validation.

## Minimal implementation strategy

1. Keep `core/go.mod`, `core/Clash.Meta`, and the default core build unchanged.
2. Start with Windows desktop only. In the build workflow, fetch the matching Smart binary from the verified `vernesong/mihomo` release, verify its checksum, rename it to a stable sidecar name such as `mihomo-smart.exe`, and place it in the release/package directory.
3. Add a persisted, opt-in setting that defaults to the normal core. Resolve the selected executable path centrally; the normal core remains the default and remains available as a fallback option.
4. Extend the Windows helper's allowed SHA-256 values to include the exact Smart binary hash. Preserve validation and helper-managed lifecycle; do not silently fall back to unprivileged direct execution when Smart is selected.
5. Make process-priority and installer shutdown handling account for `mihomo-smart.exe`.
6. Do not add Smart-specific config rewriting in the first implementation. First verify that an existing profile starts and validates under Smart Core. If Smart-only group settings are needed, add a separate opt-in override that is active only when Smart is selected and can be cleanly removed.
7. Add an automation workflow that checks upstream versions, prepares the integration patch, runs format/code-generation/build checks, and stops on patch conflicts or checksum/build failures. Do not auto-publish a broken or unverified artifact.
8. Only after Windows passes startup, profile validation, stop/restart, helper-service, and installer/portable checks should Linux/macOS be considered. Android needs a separate design because Bettbox loads a C shared library there.

## Implementation status (2026-10-14)

The first Windows integration changes are committed on `mihomo-smart`:

- `.github/workflows/build.yaml` dynamically resolves the Smart Core release, verifies the selected ZIP against `checksums.txt`, extracts the executable to `libclash/windows/mihomo-smart.exe`, and passes its exact SHA-256 to the helper build.
- `services/helper/src/ops.rs` keeps the original `TOKEN` health-check hash and allows additional core hashes only through an explicit comma-separated allowlist. Hash validation remains enabled.
- `setup.dart` supports the optional `--allowed-core-hashes` build argument. Helper-only builds no longer rebuild/overwrite the already signed core before embedding its hash.
- Windows packaging conditionally includes the sidecar; installer shutdown handling includes `mihomo-smart.exe`.
- The Windows settings page has an opt-in Smart Core switch, persisted separately from the existing application config. BettboxCore remains the default. If the helper cannot start Smart Core, the app refuses to fall back to direct, unprivileged launch.

This is source-level implementation only. A Windows CI build and runtime checks (helper startup, profile validation, stop/restart, installer and portable packaging) have not yet been run. Treat the integration as unverified until those checks pass.

## Non-goals and safety rules

- Do not modify the `main` branch.
- Do not fork or merge the Smart source tree into Bettbox's existing `core/Clash.Meta` tree.
- Do not overwrite or replace the normal core with Smart Core.
- Do not remove the Windows helper's hash verification.
- Do not claim completion based only on source edits or formatting; a real Windows build and runtime test are required.


## Android ARM64 investigation (2026-10-14)

Target scope is Android `arm64-v8a` only. Windows target is Windows 11 x64 only.

### Confirmed architecture

- `setup.dart` builds Android's default core as a Go C shared library (`-buildmode=c-shared`) named `libmeta.so`; the generated Dart FFI layer opens `DynamicLibrary.open('libmeta.so')`.
- Android's `ClashLib` sends JSON actions through the existing service-engine/IPC path. The FFI surface includes many Bettbox-specific actions (configuration validation and setup, profile management, traffic, logs, providers, DNS, connections, Android VPN options, etc.), not just a generic Mihomo REST API.
- Android TUN integration is tied to the existing native `libcore.so` JNI shim. Its `startTun(fd, callback)`, `stopTun()`, and suspend calls invoke exported functions from `libmeta.h`; the callback protects sockets and resolves process ownership.
- The Smart Core artifact verified by CI is a standalone AArch64 executable, not a drop-in `libmeta.so`. Copying or renaming it into the JNI/FFI path is incompatible.
- The standalone executable does expose normal Mihomo configuration/controller behavior, but that alone does not replace Bettbox's current IPC action protocol or the Android TUN file-descriptor/callback integration.

### Design conclusion

Do not implement an Android executable sidecar yet. A subprocess adapter would need to re-create or translate the current Bettbox action API and correctly integrate the Android VPN TUN descriptor, socket protection, process attribution, service lifecycle, and restart/cleanup. The current investigation has not established a safe way to pass the existing TUN FD and callbacks to this standalone executable. A REST controller alone is insufficient evidence of compatibility.

The lowest-risk route to preserve current Android behavior is to investigate whether the Smart fork can be built as a C shared library with the exact Bettbox export/API surface and Android JNI hooks. This must be demonstrated first; the upstream standalone executable cannot be assumed to export those symbols. If that is not feasible, a larger adapter design will need a separate technical spike and device-level validation before any UI toggle or production integration is added.

### Build verification

- Android ARM64 CI: https://github.com/ly6980/Bettbox-smart-core/actions/runs/38569348565 — passed APK build/output checks and verified the upstream Smart executable's checksum and AArch64 ELF architecture. This is artifact/build validation only, not runtime integration.
- Windows CI: https://github.com/ly6980/Bettbox-smart-core/actions/runs/38566898021 — passed build and sidecar packaging checks. This does not prove runtime behavior on Windows 11.

### Next research steps

1. Inspect the Smart fork's Go exports/build tags and compare them with Bettbox's `libmeta.h` and generated FFI symbols.
2. Check whether the fork can produce an Android ARM64 `c-shared` artifact while retaining the Smart-specific functionality.
3. If feasible, build a separate proof-of-concept artifact and compare its exported symbols/API before wiring it into Bettbox.
4. Keep the normal Android core selected by default and do not add a Smart Core toggle until the proof-of-concept passes build and device-level TUN/lifecycle checks.


## Android ARM64 interface comparison (second pass, 2026-10-14)

### New lead: an existing Android c-shared bridge

Found a relevant public implementation: https://github.com/oviron/libmihomo-android . It builds Mihomo as `libclash.so` using Go `-buildmode=c-shared`, with a C/JNI facade and an action dispatcher. Its documented Android TUN API accepts a file descriptor and callbacks for socket protection/process resolution. This demonstrates that a standalone Mihomo executable is not the only possible Android packaging model.

However, it is not a drop-in dependency for Bettbox:
- It targets the MetaCubeX Mihomo source, not the `vernesong/mihomo` Smart fork currently used by this project.
- Its bridge has its own JSON action vocabulary and Kotlin facade. Bettbox's generated `clash_ffi.dart`, `libmeta.so` symbols, IPC action methods, and JNI `libcore.so` are a separate contract.
- Therefore, adopting the prebuilt AAR directly would not make Bettbox's existing calls compatible. It would require either porting Smart-specific changes onto a bridge-compatible source tree or implementing and validating an adapter/bridge.

### Comparison result

| Requirement | Bettbox Android today | Smart release artifact | Existing libmihomo-android approach |
|---|---|---|---|
| Core format | Go C shared library `libmeta.so` | Standalone AArch64 ELF executable | Go C shared library `libclash.so` |
| Control interface | Generated Dart FFI + Bettbox IPC actions | CLI/config/controller API | Custom Go exports + JSON action dispatcher |
| Android TUN | Existing JNI shim and callbacks | No compatible Bettbox JNI contract verified | Explicit TUN FD + callbacks in its own API |
| Smart functionality | No Smart feature | Yes | Not confirmed; built against MetaCubeX source |

### Updated recommendation

The strongest next proof of concept is **not** to swap in the prebuilt Smart executable or adopt the third-party AAR unchanged. Instead:
1. Compare the bridge/export implementation in Bettbox's pinned `core/Clash.Meta` with the Smart fork's source/build tags and Smart-specific changes.
2. Determine whether the Smart fork can compile the same Bettbox C-shared bridge while preserving its Smart implementation.
3. Build only `arm64-v8a`, inspect exported symbols and run a minimal FFI/init/config validation test.
4. Only then wire the result to the Android TUN JNI path and test on an ARM64 device.

No Android runtime code or UI switch has been changed by this research pass. Original core remains the default.
