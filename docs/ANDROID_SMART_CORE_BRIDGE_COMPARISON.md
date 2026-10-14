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

### First compatibility build: failed due to API drift

Workflow: https://github.com/ly6980/Bettbox-smart-core/actions/runs/38573488262
Commit: `f3e8c02c8dcd1fad1d0944a73e70f305f273dc96`
NDK installation and fetching the correct Smart source both succeeded. The Android ARM64 Go compile failed before linking.

Compiler errors include:
- `ProxySetProvider.GetSubscriptionInfo` missing.
- `features.Android` missing.
- `listener.StopListener` missing.
- `statistic.DefaultManager.NowTraffic` and `TotalTraffic` missing.
- `updater.UpdateMMDBWithPath` and `UpdateASNWithPath` missing.
- A `constant.DefaultTestURL` reference/assignment incompatibility in `core/common.go`.

This is not an NDK or architecture failure. It confirms Bettbox's wrapper calls APIs that are absent or changed in the current Smart source revision. The earlier assumption that the wrapper might compile unchanged was too optimistic.

## Next step: classify API drift before editing

Do not replace the whole Bettbox wrapper and do not change the default core. Compare each failing call site with the Smart source APIs and classify each as:
1. A small wrapper-side adaptation.
2. A feature that needs equivalent functionality in the Smart source.
3. A behavioral mismatch requiring a larger port.

Determine whether each call is Bettbox-specific integration or ordinary Mihomo API drift. Keep experiments isolated on `mihomo-smart`. Apply only a minimal, reviewed compatibility patch, then rerun the same ARM64 C-shared build.

## Constraints

- Supported targets only: Android `arm64-v8a` and Windows 11 x64.
- Preserve the existing Bettbox core as default and keep the Smart option opt-in.
- Do not replace or rewrite the normal core.
- Do not add Android UI selection until shared-library and runtime checks pass.
- Keep all work on `mihomo-smart`; `main` must remain unchanged.
