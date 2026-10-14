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
