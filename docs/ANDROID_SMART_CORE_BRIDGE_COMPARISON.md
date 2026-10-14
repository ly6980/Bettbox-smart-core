# Android Smart Core Bridge Compatibility Check

Date: 2026-10-14
Branch: mihomo-smart

## Key correction

The requirement from the start is to **retain Bettbox's original core and add Smart Core as an optional choice**. The earlier direct-source compatibility experiment was only a diagnostic check; it is not the intended final architecture.

## Findings

### Existing core and Smart source layout

- Bettbox builds Android core as a Go C-shared library (`libmeta.so`) and connects it to Kotlin VPN/TUN callbacks through JNI. The Smart standalone Android executable is not a drop-in replacement.
- Our `main` branch and `mihomo-smart` branch currently have the same original kernel file hashes for the files checked, including `core/Clash.Meta/adapter/adapter.go` (`eeb33fe0...`) and `adapter/provider/provider.go` (`fa7d56bc...`). The previous failed experiment therefore does not justify changing the default kernel or its API.
- KevinChen222's `feat/smart-core` branch keeps the original `core/Clash.Meta` tree as the patch base. Its `core/go.mod` points to the generated `./.smart-mihomo` tree, while `tool/prepare_smart_core.dart` copies the original tree into that disposable directory and applies `core/smart.patch` using a three-way merge.
- The patch is approximately 453 KB and touches 69 kernel files. Its initial upstream file hashes match our original kernel for the files checked. A new CI job successfully applied the full patch to a disposable copy of our current `core/Clash.Meta` using Git's three-way apply; no original kernel files were changed. The first build attempt then stopped at module checksum resolution (`github.com/vernesong/leaves` missing from `go.sum`), so the workflow now runs `go mod tidy` against the generated Smart tree before compiling. The corrected run is the next validation gate.
- Kevin's branch's latest observed CI run, [Build Smart #32](https://github.com/KevinChen222/Bettbox/actions/runs/38515250989), passed both the Android ARM64 job and Windows job. Android ran model tests, prepared the Smart kernel, ran Go bridge and Smart kernel tests, built the Android core/APK, and uploaded the arm64 artifact. Windows prepared the Smart kernel, ran tests, built the core/helper and client, and uploaded the package.
- This CI validates Kevin's **Smart-enabled build**. It does not prove a dual-core runtime selector: his build points the Go module at `.smart-mihomo` for that build, rather than preserving both kernels as independently selectable binaries.

### Why our first compatibility build failed

Workflow: https://github.com/ly6980/Bettbox-smart-core/actions/runs/38573488262
Commit: `f3e8c02c8dcd1fad1d0944a73e70f305f273dc96`

The diagnostic attempted to compile Bettbox's wrapper against a separately fetched Smart source tree without first applying Bettbox's Smart integration patch. Compilation failed due to API differences, including `GetSubscriptionInfo`, `features.Android`, `listener.StopListener`, traffic statistic methods, updater functions, and `constant.DefaultTestURL`.

This was an incomplete integration strategy, not evidence that Smart cannot be integrated.

### Clash Party dual-core implementation lessons

Clash Party's desktop implementation provides a useful process-level pattern: a persisted core choice is resolved to a distinct executable path; profile validation is performed against the selected core; only one child core is active at a time; startup readiness, process exit, watchdog cleanup, orderly stop, and restart are managed centrally. Its process manager also recognizes the Smart executable as a distinct process name.

That architecture is not directly portable to Android. Clash Party launches separate executable processes, while Bettbox Android builds a Go C-shared library (`libmeta.so`) and loads a fixed JNI shim (`System.loadLibrary("core")`) that links against it. Android therefore needs two separately named core libraries and a controlled bridge selection path, with TUN callbacks and start/stop lifecycle bound to the selected implementation. Merely adding a `smart` option or swapping a file on disk would be unsafe.

Reference: [Clash Party core manager](https://github.com/mihomo-party-org/clash-party/blob/smart_core/src/main/core/manager.ts).

## Design constraint for our project

The final implementation must be dual-core, not merely a Smart-enabled build:

1. Keep `core/Clash.Meta` and the existing core build path unchanged.
2. Generate and patch a separate Smart kernel tree, pinned to an exact upstream revision.
3. Build a separate Smart bridge/core artifact from Bettbox's wrapper against that tree; do not overwrite `libmeta.so` or the normal Windows core.
4. Add a guarded, opt-in selector only after both artifacts build and their launch/lifecycle paths can be switched safely.
5. Preserve the original core as the default and provide a reliable fallback if Smart fails to start or fails integrity checks.
6. Test Android ARM64 VPN/TUN lifecycle and Windows x64 launch, stop, restart, and configuration persistence before exposing the selector.

## Next comparison steps

- Inspect Kevin's patch by file and identify which hunks are strictly kernel Smart features, which are Bettbox API adapters, and which support additional client features not needed for our first milestone.
- Confirm all patch base hashes and apply it to a disposable copy of our `core/Clash.Meta`; do not replace tracked files yet.
- Inspect the Dart FFI loader and Android JNI binding to determine how both named libraries can coexist without symbol collisions; then prototype lifecycle switching before any UI selector.
- Keep all experiments on `mihomo-smart`. `main` remains untouched.

## Scope

- Supported targets: Android `arm64-v8a` and Windows 11 x64 only.
- Original core remains default.
- Smart Core is optional and must fail safely.
- No no-op stubs or silent error suppression to force compilation.
