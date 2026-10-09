# gear-s3-gemini
The goal was to repurpose a Samsung Gear S3 Frontier (Tizen 4.0) as a push-to-talk voice assistant that calls Google Gemini directly from the watch.

> **Status: abandoned.** The project stopped at step 1 (environment setup, [#1](https://github.com/leonardonsantos/gear-s3-gemini/issues/1)). We could not install a working Tizen SDK with the Wearable 4.0 Native profile on the host. See [docs/plan.md](docs/plan.md) for the original plan.

## Environment setup attempts (2026-10-09)

Host: macOS 26.6.2, Apple Silicon (arm64), Rosetta 2 enabled.
Target: Tizen Studio + Wearable 4.0 Native + Samsung Wearable Extension + Samsung Certificate Extension, with `tizen` and `sdb` on PATH.

| # | Approach | Result | Why it failed |
|---|---|---|---|
| 1 | **Tizen Studio 6.1** (latest, as the issue specified) | ❌ Abandoned before install | The 6.1 package repo (`download.tizen.org/sdk/tizenstudio/official/pkg_list_macos-64`) only ships profiles from 6.0 up (WEARABLE-6.0/6.5/7.0, TIZEN-8.0+). **It has no WEARABLE-4.0.** Snapshots 6.0, 6.1, and 6.1.1 don't have it either. |
| 2 | **Tizen Studio 5.6 Baseline DMG** (GUI installer) | ❌ Can't automate | The DMG wraps a Java/SWT installer (`installer.jar`). With headless Java and `--accept-license`, it fails with `SWTException: Invalid thread access`: a Cocoa display is required, so there is no unattended install. |
| 3 | **Tizen Studio 5.6 web-cli installer** (`web-cli_Tizen_Studio_5.6_macos-64.bin --accept-license`) + `package-manager-cli install` | ⚠️ Partial | The base SDK, `sdb`, `tizen` CLI, NativeIDE/CLI, GCC 9.2 toolchain, `cert-add-on` 2.0.75, and `tizen-wearable-extension` 1.5.0 all installed. **WEARABLE-4.0-NativeAppDevelopment(-CLI) was never installed.** The Package Manager auto-updates to the latest remote snapshot (`/snapshots/Tizen_SDK_10.0`), where Wearable 4.0 Native doesn't exist. It also upgraded NativeIDE/NativeCLI to 3.1.3. |
| 4 | Same as #3, but pin the snapshot by editing `~/tizen-studio/.info/repository.info` (`Target-Snapshot: /snapshots/Tizen_Studio_5.6`, `Auto-Update: false`) | ❌ | `show-pkgs` then listed WEARABLE-4.0 packages. But `install` rewrites `repository.info` back to `Tizen_SDK_10.0` / `Auto-Update: true` before resolving, reports "Nothing to install", and installs nothing for 4.0. Wiping the install and pinning again gave the same result. |
| 5 | Local SDK image: mirror the 5.6 snapshot's `pkg_list_macos-64` and the dependency closure of the needed packages (SHA256-verified), zip it with `os_info`/`image.info`, and install with `package-manager-cli install -f <image>` | ❌ | `show-repo-info -f` listed WEARABLE-4.0 correctly. But install failed with `Cannot install the Tizen Studio package`. The Package Manager also wants packages that the requested ones don't depend on, such as the `Baseline-SDK` meta-package, `*-vs-tools-add-ons`, `ocf-sdk-*` plugins, and `web-privilege-checker`. Every missing one aborts the whole transaction. A first bug in our resolver (deps carry an `[macos-64]` suffix) made the closure too small. Fixing it raised the closure to 88 packages (5.6 GB), and install still failed on more missing meta-packages. Mirroring the whole 5.6 snapshot would be about 19 GB. We stopped here. |

### What still worked at the time
- The SDK mirror at `https://download.tizen.org/sdk/` was online. The `Tizen_Studio_5.6` snapshot still listed WEARABLE-4.0 packages, and their binaries returned HTTP 200.
- The Samsung extension zips could still be downloaded:
  - `https://download.tizen.org/sdk/extensions/tizen-certificate-extension_2.0.75.zip`
  - `https://developer.samsung.com/sdk-manager/repository/tizen-wearable-extension-sdk-1.5.0.zip`
- Samsung certificate issuance, the biggest risk in the plan, was **never tested**. We never got far enough to connect a device.

### Ideas not tried
- Use the 5.6 GUI Package Manager interactively and pick the older snapshot in its Configuration screen.
- Build with a newer wearable rootstrap and set `api-version="4.0"` in `tizen-manifest.xml`.
- Use a Linux x86_64 VM or Docker image with an older Tizen Studio (e.g. 4.x/5.0), where Wearable 4.0 was still the default.
- Write a Tizen **web app** (JS) instead of native C. It would only need the web CLI, which did install.

All local artifacts from these attempts (`~/tizen-studio`, `~/tizen-studio-data`, `~/.package-manager`, `~/tizen-mirror`) were removed.
