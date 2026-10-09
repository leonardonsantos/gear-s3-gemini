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
- Write a Tizen **web app** (JS) instead of native C. **Not a real way around the blockers** (see "Why a web app doesn't help" below).

## Follow-up web research (2026-10-09)

A research pass looked for setups other people have published. Searches covered GitHub, Docker Hub, Stack Overflow, the Samsung Developer Forum (through the Wayback Machine), and Reddit/XDA (through search caches). We found no maintained recipe. The likely final blocker is Samsung's certificate service, not the SDK.

### Samsung watch certificates appear broken (strongest reason to stop)
- Samsung Developer Forum thread [*"Solution for PKCS#12 error for creating certificates on a Tizen Watch3"*](https://web.archive.org/web/20251007114450/https://forum.developer.samsung.com/t/solution-for-pkcs-12-error-for-creating-cetificates-on-a-tizen-watch3/42338) (archived Oct 2025):
  - Creating a **watch** author certificate fails with `java.security.KeyStoreException: Key store type should be PKCS12 ...`. This happened with Tizen Studio 6.1 and with the latest Certificate Extension on Linux.
  - The generated `author.pri` and `author.crt` files are not valid PKCS#12/PEM.
  - The **TV** certificate flow works for the same user, which points to a server-side problem with the wearable certificates only.
  - The archive shows no fix.
- r/GalaxyWatch (Oct 2025), *"Are we doomed with TizenOS watches??? Failing to create new certificates to sideload Tizen apps on the watch"*, reports the same error.
- The Galaxy Store for Tizen watches is shut down. New downloads stopped on 2025-05-31, and re-downloads stopped on 2025-09-30.
- Without a Samsung distributor certificate that includes the watch DUID, nothing installs on a Gear S3. So even a working SDK install would very likely have hit a dead end at step 2.

### Docker images and existing setups
| Image / repo | Last update | Verdict |
|---|---|---|
| [`cirocavani/tizen-studio-docker`](https://github.com/cirocavani/tizen-studio-docker) | 2018-06 | Showed WEARABLE-4.0 installing back when it was the current profile. The installer URLs and repo state are from that time and the setup is x86 only. Useful only as a reference. |
| `dhsshine/tizen-studio` (Docker Hub) | 2020-09 (tag 3.7) | amd64 only; no evidence it includes the wearable profile or certificates |
| `tuduongquyet/tizen-studio-cli` (Docker Hub) | 2020-02 | Says "SDK 5.5 and 3.0". Stale, amd64 only, no Dockerfile source. |
| `vitalets/tizen-webos-sdk` (Docker Hub) | 2023-11 | For TVs and webOS, not wearables |
| TizenBrew / [`hcbille/install-tizenbrew-tizen`](https://github.com/hcbille/install-tizenbrew-tizen) | n/a | Samsung **TVs** only. There is no equivalent for watches. |

None of these is arm64, maintained, or ships the Samsung Certificate Extension in a working state.

### Other alternatives checked
- **Choosing an older snapshot in the GUI Package Manager:** we found no report confirming that it avoids the forced update to `Tizen_SDK_10.0`. The GUI uses the same backend as the CLI that failed above, so this is unverified.
- **A newer rootstrap (wearable 5.5/6.0) with `api-version="4.0"`:** we found no public report of this working for native C on a Gear S3. `api-version` doesn't change which libraries the binary links against, so it could fail at runtime on the watch's older libraries.
- **GBS / building with only a toolchain and rootstrap:** GBS builds platform RPMs, not signed TPKs, so it doesn't solve signing.
- **Open-source client for Samsung's certificate service:** none exists. Certificate issuance is only available through Samsung's Java Certificate Manager.
- **Tizen web app (JS):** see the next section.
- **Docs:** docs.tizen.org now redirects to samsungtizenos.com, which has no API reference content.

### Why a web app doesn't help
A Tizen web app (HTML/JS packaged as a `.wgt`) looked like a way around the missing Wearable 4.0 *Native* packages. The Wearable 4.0 *Web* development CLI did install in attempt #3. We found no evidence that a web app would work.

**What supports it:**
- The `WEARABLE-4.0-WebAppDevelopment-CLI` package installed fine.
- HTTPS POST to Gemini with `fetch`/XHR is routine. It needs only the `http://tizen.org/privilege/internet` privilege and an access policy in `config.xml`.
- Speech output may be possible through a Tizen JS speech API. We did not confirm this on Tizen 4.0.

**What argues against it:**
- **Microphone capture:**
  - We found no example of a Tizen 4.0 wearable web app recording audio with `getUserMedia`/`MediaRecorder`.
  - Tizen's JS device APIs don't appear to include an audio recorder.
  - Without recording, the app would need a native service component. That brings back the Wearable 4.0 Native SDK we couldn't install.
- **Signing:**
  - Web apps also need a Samsung author certificate and a distributor certificate that includes the watch DUID before they install on a Gear S3.
  - Creating watch certificates is the step reported broken since Oct 2025.
  - So a web app runs into the same final blocker as the native app.
- **Docs:** docs.tizen.org no longer serves the API reference, so we couldn't check which web APIs exist on Tizen 4.0.

**Verdict:** a web app does not get around the problem. Its core feature (microphone capture) is unproven, and it still depends on the Samsung watch certificate service, which is reported broken.

### Signing blocks every alternative
Every alternative still produces a package (`.tpk` or `.wgt`) that must install on a retail Gear S3. The watch only accepts packages signed with a **Samsung author certificate** and a **Samsung distributor certificate that includes the watch DUID**. So the broken certificate service blocks all of them, not just the native SDK path.

| Alternative | Needs a Samsung watch certificate? |
|---|---|
| GUI Package Manager with an older snapshot | Yes. It only fixes the SDK install. |
| Newer rootstrap with `api-version="4.0"` | Yes |
| Linux VM or Docker with an older Tizen Studio | Yes. The failure looks server-side (TV certificates still work), so older Certificate Extension versions would probably fail too. Unverified. |
| Tizen web app (JS) | Yes |

**Possible exceptions:**
- **Emulator:** it accepts the default Tizen certificates. That's useful for development, but it isn't the real watch.
- **An existing Samsung distributor certificate:** a certificate issued before the breakage that already includes this watch's DUID would still work. We don't have one.
- **A rooted or modified watch that skips signature checks:** we found no evidence of this for the Gear S3.

**Conclusion:** the project's real blocker is signing, not the SDK. Fixing the SDK install alone would not make the project work.

## Other ways to reuse a Gear S3 (research, 2026-10-09)
A short research pass, stopped early. It asked whether the watch can still run custom software without a new Samsung certificate. Items marked *unverified* were found but not confirmed.

| Option | Status | Notes |
|---|---|---|
| Install **already Samsung-signed** apps from community backups (e.g. "Gear Browser 1.2.1", "G-Voice Assistant", "GAssistNet") with `sdb connect <ip>:26101` and `sdb install` | Possible | No new certificate is needed, because Samsung already signed these packages. They now come only from community backups, because the watch app store is closed. See the [XDA extraction thread](https://xdaforums.com/t/root-required-how-to-extract-gear-s3-watch-faces-and-apps-from-the-galaxy-store.4687851/). **Unverified:** whether Gear Browser can open a self-hosted page, and whether it can use the microphone. |
| Some Galaxy Watch apps reportedly install on the Gear series "without any signing" | Unverified | Claimed in the same thread. It may only apply to packages Samsung already signed, not to custom ones. Details may be in the [no-root extraction thread](https://xdaforums.com/t/no-root-method-to-extract-and-backup-tizen-apps-for-samsung-galaxy-watch-active-2.4693807/) (not opened). |
| Root via Samsung's leaked engineering ("combination") firmware for SM-R760, flashed with Odin/netOdin | Works but crippled | It gives a root shell (`sdb root on`). In this mode the watch "can't connect to a phone, all apps are disabled", so it's only useful for experiments. Many download links are dead. There is a risk of bricking the watch. See the [XDA root thread](https://xdaforums.com/t/gear-s3-root-and-kernel-source-android-wear-port-thread.3584588/). |
| Use root to bypass signature checks | Dead | The XDA extraction thread (Aug 2024) says extracted apps need a Samsung platform-level certificate, and that a leaked platform certificate "is no longer valid on Tizen 4.0.0.7". Root alone doesn't make installing a custom app possible. |
| Replace the OS (AsteroidOS, Wear OS 2 port) | Dead | [AsteroidOS](https://asteroidos.org/watches/) doesn't support the Gear S3. The [Wear OS 2 port thread](https://xdaforums.com/t/closed-rom-dev-wearos-2-for-gear-s3.4400757/) is closed, and the community Android Wear port never reported a booting build. |
| Stock Bixby voice on the watch | Not researched | |

**Most realistic next step (not attempted):** find a backup of Gear Browser, sideload it, and test whether it can load a self-hosted page that talks to Gemini.

All local artifacts from these attempts (`~/tizen-studio`, `~/tizen-studio-data`, `~/.package-manager`, `~/tizen-mirror`) were removed.
