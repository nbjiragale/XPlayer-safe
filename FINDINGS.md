# FINDINGS.md — Xplayer.apk (Phase 1: FLAG)

Static analysis only. The APK was never installed or run. Work was done on a copy.
Phase 2 (FIX) has **not** started. It waits for your approval.

## 1. Basics

| Item | Value |
|---|---|
| File | `Xplayer.apk` (from your Google Drive), 43,652,658 bytes. Size matches Drive. ZIP test passes. |
| SHA-256 | `08fc8cca89c1315954a9d9dd82218dc901ef58e334e1b2b7f778f8dc41988847` |
| MD5 | `d7a18c97dac8eb7685d017e6c4072e8c` |
| Package | `video.player.videoplayer` (this is the package id of the real XPlayer by InShot) |
| Label | XPlayer - Video Player |
| Version | 3.0.0 (versionCode 300004) |
| minSdk / targetSdk | 24 / 36 |
| Launcher | `com.inshot.xplayer.activities.SplashActivity` |
| Application class | `com.inshot.xplayer.application.XPlayerApplication` |
| Debuggable | No (attribute not set) |
| allowBackup | false |
| Native ABIs | arm64-v8a, armeabi-v7a (14 libs each) |
| DEX files | `classes.dex` 15.7 MB (13,149 classes), `classes2.dex` 6.8 MB (4,551 classes), `classes3.dex` 19 KB (19 classes, DEX version 035) |

### Signing certificate (Confirmed)
- Signed with v2 and v3 schemes (valid). Old v1 files exist (`META-INF/YOUAREFINISHED.*`).
- Subject: `C=US, ST=California, L=LA, O=Google, OU=android, CN=youarefinished`
- 1024-bit RSA, SHA-1 signature, valid 2021-11-20 to 2046-11-14.
- Cert SHA-256: `7981162e75cd768fc2efacee773c924be0c52217a86d85c4d2a8304f2bf1c4b4`
- This is **not** the developer's key. It is a generic, weak modder key. It claims "Google" but is not Google.

### Odd files
- `classes3.dex`: tiny, has a different date from the other DEX files, and holds the `bin.ghost` code. See F-01.
- `lib/*/libyrf.so`: the only native lib with a different date (2026-06-14). It is not part of a normal video player. See F-01 and F-02.
- `assets/sadfds_l23ol-zp5`, `assets/q-peai6_mck`, `assets/23o-as_pawer2tw`, `assets/pqhp8-64_36mu`: random-looking names, high entropy, not plain text. See "Not verified".
- `META-INF/services/kb`, `META-INF/services/zh0`: tiny files with obfuscated names (normal R8 side effect).
- `META-INF/README.md` is the jsoup library readme. It is harmless.
- `res/raw/hosts.txt` is a 7,171-line ad-block host list for the in-app browser. It is harmless.

## 2. How the scan was done (and its limits)

- `apktool` 2.7.0 and 2.11.1 **both failed on 14,945 of ~17,700 classes** ("Invalid element width"). Smali is only complete for `classes3.dex` and a small part of the rest.
- `jadx` 1.5.1 finished with **341,580 errors**. Many methods cannot be shown as Java.
- Because of that I added a second pass: **androguard** read the raw bytecode of all 78,429 methods (0 unreadable) and I also dumped every string in all three DEX files. All "code" findings below come from these two views.
- Line numbers exist only where smali exists (`smali_classes3/bin/ghost/*.smali`). For other code I give class and method names.

## 3. Findings

Status: **Confirmed** = I saw it directly in the file. **Suspected** = it looks wrong, but I could not prove intent or full behavior.

### F-01 — Signature-spoofing code injected into the Application chain
- **Severity:** High — **Status:** Confirmed
- **Where:**
  - `classes3.dex`, `smali_classes3/bin/ghost/yrf.smali:10-33` (static initializer). The same 19 `bin.ghost.*` classes are also in `classes.dex`.
  - Chain: `XPlayerApplication` → `y90` (`defpackage/y90.java:7`, in `classes.dex`) → `bin.ghost.yrf` → `android.app.Application`.
  - `lib/arm64-v8a/libyrf.so` and `lib/armeabi-v7a/libyrf.so`.
- **What it is:** The code is the open-source "ApkSignatureKillerEx" (the class has the string `https://github.com/L-JINBIN/ApkSignatureKillerEx`). When the app starts, before anything else, it:
  1. loads `libyrf.so`;
  2. turns off Android's hidden-API protection (`VMRuntime.setHiddenApiExemptions`, via `sun.misc.Unsafe`);
  3. calls `killPM("video.player.videoplayer", <base64 cert>)`, which hooks the PackageManager so any signature check sees a **different certificate**. The cert it injects is `O/OU/CN=CameraIdeas`, from 2014. It is probably the original developer's, but I could not verify that.
  4. calls `killOpen(...)` with empty strings. It returns at once, so the file-redirect part is **off**.
- **Why it matters:** The mod hides its own re-signing, so the app and other code believe they are untouched. It also loads an unreviewed native library at the earliest point in startup. This is how modded APKs defeat license and integrity checks.
- **Proposed fix:** Make `yrf.<clinit>` do nothing (patch the DEX directly), in both `classes.dex` and `classes3.dex`. Delete both `libyrf.so`. Remove `classes3.dex` after checking that nothing needs it. Remove the four `assets/` blobs only if F-02 shows they belong to this code.

### F-02 — Native library `libyrf.so` with hidden behavior (and a hidden dialog)
- **Severity:** High — **Status:** Confirmed (updated in Phase 2: the library's strings were decoded; its machine code was not disassembled)
- **Update (Phase 2):** The library holds base64 strings that decode to `https://pastebin.com/raw/0NEmugw4`, the hash of the mod's own signing key, and the names `META-INF/BNDLTOOL.RSA` and `META-INF/YOUAREFINISHED.RSA`. It is an **update/credits notice and anti-tamper module** run by the mod author. The Pastebin page (read once, as data) is a 96 KB JSON table of app package names with version numbers (691 apps) plus one link, `https://t.me/youarefinished_mods`. The author can change that page or link at any time, so the dialog can be pointed anywhere. I found no stolen-data upload in what I could read.
- **Where:** `lib/*/libyrf.so` (stripped; arm64 473 KB). Java side: `bin/ghost/YRF.smali`, `cp.smali`, `gho.smali`, `cup.smali`, `dw.smali` (all `native` methods).
- **What I saw:**
  - JNI bridge strings for `HttpURLConnection.openConnection`, `AsyncTask`, `SharedPreferences`, `Toast`, `startActivity`, `AlertDialog`, `MessageDigest`, `CertificateFactory`, `ZipEntry`, and reflection.
  - `bin.ghost.YRF` is a dialog class with Accept and Exit buttons, a checkbox and a 10-second countdown. Its text is hidden inside the native code.
  - No readable URL inside the library. Nothing in the Java code calls `YRF.execute`, so a native call is the only way it runs.
- **Why it matters:** The library can talk to the network and show screens, and I cannot see where or what. It cannot be called safe.
- **Proposed fix:** Remove `libyrf.so` (see F-01). Nothing else should need it.

### F-03 — Re-signed with an unknown, weak key
- **Severity:** High — **Status:** Confirmed
- **Where:** `META-INF/YOUAREFINISHED.RSA/.SF`. v2/v3 signing block.
- **Why it matters:** The original developer's signature is gone. Updates and Play Protect trust are lost. A 1024-bit SHA-1 key is weak, and it can be used by anyone who has it.
- **Proposed fix:** Re-sign with a new keystore you control (Phase 2). The original signature cannot be restored.

### F-04 — Anti-analysis: "Smob" obfuscator marker and broken bytecode
- **Severity:** High — **Status:** Confirmed (marker and tool failures); Suspected (that the breakage is deliberate)
- **Where:** The string `Smob - Mod obfuscation tool v4.6 by Kirlif'` is in `classes.dex` and in `classes2.dex` (class `w53`, constructor). 14,945 classes fail normal disassembly with impossible array-payload widths (7, 9, 11, 13, 15).
- **Why it matters:** Smob is a modder tool that adds junk code to stop decompilers. That is why apktool and jadx fail. It also means ~85% of the app cannot be reviewed with normal tools, and a normal apktool rebuild would **drop those classes** and break the app.
- **Proposed fix:** Cannot be "removed". For Phase 2 I will avoid rebuilding code with smali. I will patch the DEX bytes directly and use apktool only for the manifest and resources.

### F-05 — Cleartext (HTTP) traffic allowed for the whole app
- **Severity:** Medium — **Status:** Confirmed
- **Where:** `res/xml/network_security_config.xml:3` → `<base-config cleartextTrafficPermitted="true" />`
- **Why it matters:** All traffic can be read or changed on the network.
- **Proposed fix:** Set it to `false`. **Risk:** A video player opens `http://` streams, so some links will stop working. I need your decision.

### F-06 — A "trust everyone" certificate checker exists
- **Severity:** Medium — **Status:** Suspected
- **Where:** `classes.dex`, class `w53` (`X509TrustManager`, `checkServerTrusted` and `checkClientTrusted` only 6 instructions each). A custom `SSLContext`/`HttpsURLConnection` is built in class `vf` (`vf.d`) and a separate hostname verifier `v53` (51 instructions).
- **Why it matters:** If `w53` is used, HTTPS traffic can be read by anyone on the network. I could not confirm that `vf` actually uses `w53`, or whether the original app has it too.
- **Proposed fix:** Make `w53` reject instead of accept, or reroute `vf` to the default trust. Needs testing.

### F-07 — Device admin receiver
- **Severity:** Medium — **Status:** Confirmed (it exists); purpose Suspected
- **Where:** Manifest `com.inshot.xplayer.receiver.PlayerDeviceAdminReceiver` (exported, requires `BIND_DEVICE_ADMIN`). `res/xml/device_admin.xml` has an **empty** policy list.
- **Why it matters:** Once the user enables device admin, uninstalling needs an extra step. An empty policy list limits the harm, but a video player has no need for it.
- **Proposed fix:** Remove the receiver and its XML. I must check that no app code calls it, which would crash if it did.

### F-08 — Unneeded or sensitive permissions
- **Severity:** Medium — **Status:** Confirmed
- **Where:** Manifest.
  - `SYSTEM_ALERT_WINDOW` (draw over other apps)
  - `MANAGE_EXTERNAL_STORAGE` (all files)
  - `READ_CLIPBOARD`
  - `WRITE_EXTERNAL_STORAGE`
  - `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`
  - `USE_FINGERPRINT`
  - `RUN_USER_INITIATED_JOBS`
  - `NEARBY_WIFI_DEVICES`
  - Ads: `AD_ID` and AdServices permissions
- **Not present (good):** SMS, contacts, call log, accessibility, install-packages, camera, microphone, location.
- **Why it matters:** The overlay and all-files permissions are the ones abused most. The code uses `canDrawOverlays` in classes `ob5` and `ua3`, probably for the floating/PIP player.
- **Proposed fix:** Remove `READ_CLIPBOARD`, `SYSTEM_ALERT_WINDOW` and `MANAGE_EXTERNAL_STORAGE`. **Risk:** floating player and file browsing may break on Android 11+.

### F-09 — Exported components without protection
- **Severity:** Low — **Status:** Confirmed
- **Where (manifest):** `BootReceiver` (boot + HOME), `ChangeMusicCovertReceiver` (custom action), `XMediaButtonReceiver`, `PlayerActivity` + `PlayerActivity$WebDelegate` (VIEW for http/https/ftp/smb), `MusicPlayActivity`, `ShareEntryActivity` (SEND text/plain).
- **Why it matters:** Other apps can start these. Most are normal for a player. `ChangeMusicCovertReceiver` and `BootReceiver` are the doubtful ones.
- **Proposed fix:** Set `exported="false"` for `ChangeMusicCovertReceiver`. Keep the player ones.

### F-10 — Extra "downloader / status saver / Reels" feature package
- **Severity:** Medium — **Status:** Suspected (cannot compare to original)
- **Where:** Package `a.m.z.*` — 17 activities (`MainActivity`, `ReelsHowtoActivity`, `StatusSaverActivity`, `BrowserHistoryActivity`, `DownloadingActivity` …) plus an in-app browser. Instagram login URLs (`https://www.instagram.com/accounts/login/`) are in the code. 10 classes read WebView cookies (`CookieManager.getCookie`).
- **Why it matters:** An in-app browser with Instagram login and cookie reads is a classic place for credential theft. I found **no** unknown server that receives data, and I found no code that sends cookies anywhere. But 85% of code is hard to read.
- **Proposed fix:** Needs a diff against the original XPlayer 3.0.0. If these are not in the original, remove the `a.m.z` activities from the manifest.

### F-11 — Root, emulator and debugger detection
- **Severity:** Low — **Status:** Confirmed (code exists); owner Suspected
- **Where:** `classes.dex` class `za0` (`/system/xbin/su`, `Superuser.apk`, `test-keys`, `sdk`/`goldfish`/`ranchu`, `Debug.isDebuggerConnected`). Also `goldfish` checks in `mf5` and the InMobi SDK.
- **Why it matters:** It stops sandbox and debugger analysis. Could be from the ad SDK.
- **Proposed fix:** Leave it unless you want it gone. Removing it is safe but low value.

### F-12 — Shell commands run from app code
- **Severity:** Low — **Status:** Confirmed
- **Where:** `p32.s` (`Runtime.exec("getprop " + x)`), `cast/core/core/Util.getIpUnstableVersion` (`/system/bin/getprop`), and a logcat-collector thread in `d.run` (`Runtime.exec`).
- **Why it matters:** Fixed, read-only commands (device info, logs). I saw no way for outside input to choose the command.
- **Proposed fix:** None needed. The log reader should be reviewed if you do not trust the logs leaving the device.

### F-13 — Third-party dynamic code loading
- **Severity:** Low — **Status:** Confirmed (it exists); Google ads SDK
- **Where:** `y96.d` (decrypts an embedded blob and writes `.jar` files — the Google Mobile Ads SDK loader), `DynamiteModule`, `zzbcg`, `zzgab`.
- **Why it matters:** Code is loaded at run time and cannot be reviewed here.
- **Proposed fix:** Only by removing the ad SDK. Not recommended in Phase 2.

### F-14 — Other low items
- **Severity:** Low — **Status:** Confirmed
- Firebase API key in resources: `AIzaSyD8XGJlTw…` (normal for Firebase, but it should be restricted in Google Cloud).
- Ad and tracking SDKs: Google Mobile Ads, InMobi, Firebase Analytics/Measurement, Google Play Billing, Google Cast, Picasso, FileDownloader (liulishuo). Ad ID permissions present.
- Weak hash names (`MD5`, `SHA-1`, `DES`) appear in ~29 classes. Most are Google/ads SDK code. Not checked one by one.
- WebView JavaScript is on, with `addJavascriptInterface`, in the in-app browser and in the ad SDKs.

## 4. What I checked and found clean

- No Telegram or Discord webhooks, no paste sites, no dynamic-DNS or onion hosts, no hardcoded IP servers. (`239.255.255.250:1900` is the standard Cast/DLNA discovery address.)
- No SMS, contacts, call-log, accessibility, keylogger, screen-capture or install-package code or permissions.
- No password or token strings for third-party services, apart from the Firebase key above.
- URLs found belong to InShot (`inshot.cc`, `inshotapp.com`), OpenSubtitles, search engines, social sites, and standard ad SDKs. `cdn.darkadv.com` is inside the HLS/M3U8 stream parser (class `ra`). `spadsync.com` belongs to the InMobi SDK.
- 13 of the 14 native libs (`libdav1d`, `libx264`, `libinxffm`, `libmmkv`, `libmp3lame`, `libinxplayer`, `libinxapi`, `libinxutil` …) show no URLs, shell or anti-debug strings. They look like a normal video player stack. Not disassembled.

## 5. Compare against the original

**Not done.** I do not have a copy of the real XPlayer 3.0.0 (build 300004) here. From what I can see in the APK itself, these are the **mod additions** (Confirmed or very likely):

1. The `bin.ghost.*` package and the superclass change `y90 → bin.ghost.yrf`.
2. `lib/*/libyrf.so` (both ABIs) and `classes3.dex`.
3. The `YOUAREFINISHED` signature files and the new signer.
4. The Smob obfuscation marker.

For F-06, F-07, F-10 I cannot say if the original has them. A diff against the official APK would settle it.

## 6. Not verified

- **Full behavior of `libyrf.so`.** Its strings were decoded (see F-02), but I did not disassemble the machine code. The library is removed in the fixed APK, so this no longer matters for that build.
- **The four `assets/` blobs — RESOLVED in Phase 2.** `sadfds_l23ol-zp5`, `q-peai6_mck`, `23o-as_pawer2tw` and `pqhp8-64_36mu` are XOR-encoded (6-byte key) JSON for the app's built-in **downloader/browser**: a list of supported sites, JavaScript snippets that watch page requests on Instagram, Facebook and X and pass video links to the app (`GetPear` bridge), and a YouTube URL pattern. They are read by `libitcore.so`, not by `libyrf.so`. No outside server and no cookie/password upload is in them. They are part of the downloader feature (see F-10) and were left in place.
- **~85% of the Java code** cannot be read as source (F-04). My scan covered the raw bytecode for known APIs and strings. It cannot find logic that hides behind reflection, string building or native code.
- **Behavior of `w53`/`vf`** (F-06) and the cookie handling in the in-app browser (F-10).
- **Anything that happens at run time**: server responses, remote config (Firebase Remote Config is present), and downloaded code.
- **Original app diff** (section 5).
- The 13 "clean" native libs were checked by strings only.

## 7. Verdict (not a safety claim)

The mod is **not** a plain re-sign. It adds signature-spoofing code and a hidden native library, and it hides the rest of the code behind a decompiler-breaking tool. I found no direct proof of data theft in what I could read. The unreadable parts could hide it. I would not trust this APK.

---

## 8. Addendum (end of Phase 2): updates and corrections to the findings above

- **Code is now readable.** The Smob junk (an invalid-size data block added to almost every method) can be blanked in an analysis-only copy. With that, apktool reads all 17,719 classes (0 errors) and jadx reads them with 51 errors. Sections 2 and 6 ("~85% unreadable") are therefore out of date. Everything below was checked on readable code.
- **F-01 correction.** The mod's Java hook `killPM` is not only the mod's trick: the real app *needs* it to keep working after any re-signing (see F-15). A fix that removes it entirely makes the app close itself. The right fix is to remove only the unknown parts (native library, native helper, file redirect) and keep `killPM`.
- **F-02 (confirmed).** `libyrf.so` is an update/credits and anti-tamper module that reads `https://pastebin.com/raw/0NEmugw4` and a Telegram link. Removed.
- **F-10 (reviewed).** `defpackage.ma2` is the JavaScript bridge (its name is decrypted at runtime). It builds download candidates from pages the user opens. Cookies are read per site URL, stored in the download job for the same site, or checked for login markers (`c_user`, `ds_user_id`, `sessionid`). No upload code. Not removed.
- **F-12 (reviewed).** `Runtime.exec`/`ProcessBuilder` run `getprop` with fixed keys (for example `ro.build.version.emui`, `arc.net.ipv4.host_address`) and one own-app logcat reader. Not changed.
- **Native libraries (13 non-mod).** No imports of `exec`, `fork`, `system`, `popen`. Only `libinxffm` (FFmpeg) imports socket functions. `libmmkv` uses `sendfile` for file copy. `libx264` and `libinxplayer` use `dlopen` for optional system libraries. No hard-coded hosts.
- **Hidden strings.** The app hides ~870 of its own strings via `f82.g(base64, key)` → native `Decoder.decodeBytesNative`. Algorithm: base64 decode, then XOR every second byte (positions 0, 2, 4 …, only `len/2` of them) with a 16-byte key. The key is a 12-byte constant (one of three, chosen by which characters appear in the 8-character second argument) joined with 4 characters of that argument. 869 of 873 strings decode; 502 are unique. They contain only video-site URLs, HTTP header names, content types and X's well-known public web bearer token.

### F-15 — The real app protects itself against re-signing (new, Info)
- **Status:** Confirmed
- **Where:** `defpackage.ez4` … `rz4` (14 classes, checks inlined in `p32` and the `a.m.z.*` activities); `libitcore.so` `JNI_OnLoad` (length check `0x652`, then `strcmp` against the embedded 809-byte developer certificate).
- **What it does:** Each check takes `signatures[0].toCharsString()`, cuts a 31-character slice (for example `substring(1250, 1281)`), compares it with a baked-in constant such as `774d6bfd1d85da3a45284cd5f00ebde`, and on mismatch calls `Process.killProcess` and `System.exit(0)`. `libitcore.so` logs "signature is error" and fails to load. When it fails to load, every `f82.g(...)` string is returned undecoded.
- **Why it matters:** Any re-signed build, including mine, trips this. It is the developer's anti-piracy, not a user-facing security feature. It also explains why the mod author installed `killPM`.
- **Handling:** Keep the pure-Java `killPM` hook (in-process only). Do not try to patch 14 randomised inline checks and two native libraries by hand.
