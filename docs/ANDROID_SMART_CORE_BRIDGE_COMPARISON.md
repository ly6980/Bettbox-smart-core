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

This is not an NDK or architecture failure. It confirms Bettbox's wrapper calls APIs that are absent or changed in the current Smart source revision.

## Comparison with KevinChen222/Bettbox

Repository branch: https://github.com/KevinChen222/Bettbox/tree/feat/smart-core

- Its latest observed workflow run, [Build Smart #32](https://github.com/KevinChen222/Bettbox/actions/runs/38515250989), completed successfully on 2026-10-14 at commit `b757c1e8644b01bd1a91bd17bd48b2021cf49afd`.
- Its workflow builds Windows and Android (as well as Linux); its Android job prepares the Smart kernel, runs Go bridge and Smart-kernel tests, then builds the Android core and APK.
- It pins Smart source revision `83e67d0c4415b1114f9e01d8e9a00133caf89f6c` and uses `tool/prepare_smart_core.dart` to copy Bettbox's existing `core/Clash.Meta` into a disposable `core/.smart-mihomo` tree, then apply `core/smart.patch` with a three-way merge.
- This differs materially from our first attempt, which replaced `core/Clash.Meta` with the Smart source tree and then tried to compile Bettbox's wrapper unchanged. Kevin's successful build therefore validates his pinned source + overlay patch + wrapper combination, not a raw source replacement.
- His patch adds Bettbox-specific compatibility and integration code (for example, the missing `GetSubscriptionInfo()` accessor). The rest of the API differences must be evaluated in the context of the exact base and patch; copying isolated method names is unsafe.

## Revised implementation plan

The next experiment should follow the **overlay model**, not direct replacement:

1. Keep `core/Clash.Meta` as Bettbox's original kernel/base.
2. Generate a disposable Smart kernel directory and apply a pinned, reviewable Smart patch there.
3. Build the Bettbox Go bridge against that generated tree; do not add no-op compatibility stubs to force compilation.
4. Run Android ARM64 C-shared build and Windows x64 build before adding any UI selector.
5. Keep the original kernel as the default and keep all changes on `mihomo-smart`; `main` remains untouched.

The Kevin branch's patch is large and tied to a particular base. Before porting it, compare its pinned Bettbox baseline with our branch and check which Smart patch hunks apply cleanly. Do not copy the entire patch blindly.

## Constraints

- Supported targets only: Android `arm64-v8a` and Windows 11 x64.
- Preserve the existing Bettbox core as default and keep the Smart option opt-in.
- Do not replace or rewrite the normal core.
- Do not add Android UI selection until shared-library and runtime checks pass.
- Keep all work on `mihomo-smart`; `main` must remain unchanged.
