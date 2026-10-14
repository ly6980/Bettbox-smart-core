# Android Smart Core Bridge Compatibility Check

Date: 2026-10-14
Branch: mihomo-smart

## Findings

### Bettbox's Android runtime is a C-shared-library integration

- `setup.dart` builds Android core from the `core` Go module with `CGO_ENABLED=1` and `-buildmode=c-shared`, producing `libmeta.so` and its generated C header.
- `core/go.mod` declares `module core` and replaces `github.com/metacubex/mihomo` with the local `./Clash.Meta` directory.
- `core/lib.go` exposes the Bettbox/Dart API through cgo exports including `invokeAction`, `initNativeApiBridge`, `startListener`, `stopListener`, `getConfig`, and `suspend`.
- `core/lib_android.go` implements Android-specific TUN lifecycle and hooks, using the Mihomo APIs `listener/sing_tun`, `component/dialer`, `component/process`, `dns`, and `constant`.
- `android/core/src/main/cpp/core.cpp` consumes the generated `libmeta.h` exports and connects Bettbox's Kotlin VPN/TUN callbacks (protect socket, resolve process, release callback).

### Smart Core's release binary is not directly interchangeable

- `vernesong/mihomo` tag `Prerelease-Alpha` is the correct Smart Core source and contains Smart group implementation under `adapter/outboundgroup/smart.go` plus `component/smart/**`.
- Its published Android ARM64 asset is a standalone executable, not the Bettbox-specific `libmeta.so` C-shared interface.
- Renaming the executable to `libmeta.so` cannot satisfy the required exported functions or the JNI TUN bridge.

### Promising route: compile Bettbox's existing wrapper against Smart source

Both Bettbox's vendored Mihomo module and the Smart source use module path `github.com/metacubex/mihomo`. The wrapper imports that module via its existing local replacement. This suggests a relatively small compatibility experiment: keep Bettbox's wrapper/bridge files and compile them against the Smart source tree instead of the currently vendored `core/Clash.Meta`.

This is a hypothesis, not yet a verified integration. Source-level compatibility still needs a real build because:
- Bettbox calls many Mihomo APIs, including Android TUN and listener APIs.
- The Smart branch may have API or dependency changes relative to Bettbox's vendored source.
- cgo exports, generated header signatures, Android linker settings, and JNI lifecycle must remain compatible.
- A successful shared-library build alone does not prove the VPN starts or that callbacks work at runtime.

## Next validation experiment

1. Add a separate CI job/workflow on `mihomo-smart` only; do not alter `main`.
2. In a temporary workspace, fetch the exact `vernesong/mihomo` `Prerelease-Alpha` source and place it as the target of the wrapper module's Mihomo replacement. Keep Bettbox's wrapper files outside that source tree.
3. Build Android ARM64 with Bettbox's existing `setup.dart android --arch arm64 --out core-only` path (or equivalent Go build invocation), with cgo/NDK enabled.
4. Check generated `libmeta.so` exports against the functions consumed by `android/core/src/main/cpp/core.cpp`, and ensure Smart-specific packages are present.
5. If compilation fails, classify errors as API drift, dependency/module drift, build-tag/CGO problems, or Android linker problems before changing integration code.
6. Only after compile compatibility passes, add runtime tests for core initialization, config validation, TUN start/stop, socket protection, process resolution, and fallback to the unchanged default core.

## Constraints

- Supported targets only: Android `arm64-v8a` and Windows 11 x64.
- Preserve the existing Bettbox core as default and keep the Smart option opt-in.
- Do not replace or rewrite the normal core.
- Do not add Android UI selection until shared-library and runtime checks pass.
- Keep all work on `mihomo-smart`; `main` must remain unchanged.
