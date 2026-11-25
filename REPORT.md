# REPORT.md — Xplayer.apk: findings, fixes, and what is still risky

## 1. Summary

- The APK is a modded XPlayer 3.0.0 (`video.player.videoplayer`). Phase 1 found 14 findings. Phase 2 fixed what could be fixed safely, one change at a time, and checked each change before the next. After you pushed me to close the "not verified" items, I found real problems in my own first fixes and corrected them.
- **This is not a claim that the app is safe.** Nobody can prove that from static analysis. The app was also run in one software-emulated phone (section 9): it starts, loads all its native libraries, shows its library, and plays a video to the end without crashing. But the biggest blind spot is gone: the app's code was 85% unreadable because of an obfuscator, and I got around that. I read it, decoded the ~870 strings it hides, and checked every native library by what it can call. I found no sign of data theft or a hidden control server in what I could read.
- **Important correction.** The two APKs I sent earlier were **wrong**. They removed the mod's whole startup hook. But the real XPlayer has its own anti-tamper (14 inlined checks plus its native decoder `libitcore.so`) that closes the app if the signing certificate is not the developer's. Those builds would have shut themselves down. The new builds below keep only the small Java hook that the app needs for that, and nothing else from the mod.

**Use these files (they replace the earlier APKs):**

| File | SHA-256 |
|---|---|
| `XPlayer_cleaned_signed.apk` (main build) | `03ed706b8fb4be77f2224b5f518f2f3f526ca8c85158410d55850b4697137128` |
| `XPlayer_strict_signed.apk` (no all-files and no overlay permission) | `8a58e9872019c07b0e49311985da597290c595ab424999d7237ac0afc572fe94` |

Both are in `/tmp/claude-0/-home-user-XPlayer-safe/49bf11d1-7853-5759-a5cd-203f796f3864/scratchpad/final/` and were sent to you as attachments.

| Secret | Where |
|---|---|
| Keystore | `/tmp/claude-0/-home-user-XPlayer-safe/49bf11d1-7853-5759-a5cd-203f796f3864/scratchpad/signing/xplayer-safe.jks` |
| **Keystore password (separate note)** | `/tmp/claude-0/-home-user-XPlayer-safe/49bf11d1-7853-5759-a5cd-203f796f3864/scratchpad/signing/KEYSTORE_PASSWORD.txt` |

Neither is in the repository (`.gitignore` blocks `*.apk`, `*.jks`, `*.keystore`, `KEYSTORE*`, `*PASSWORD*`). The scratchpad can be cleared when the session ends, so copy these files somewhere safe.

## 2. What I found while closing the "not verified" items

1. **The code was unreadable on purpose.** A tool called Smob adds one junk data block with an invalid size to almost every method (20,835 of 20,845 in one file). That crashes apktool and jadx. I made an **analysis-only** copy with those blocks blanked out (same size, never installed or shipped). Result: apktool errors went from 14,945 to 0, and jadx errors from 341,580 to 51. All 17,719 classes became readable.
2. **`libyrf.so` is the mod author's update/credits and anti-tamper module.** It reads `https://pastebin.com/raw/0NEmugw4` (a 96 KB JSON table of 691 apps' version numbers plus one link, `https://t.me/youarefinished_mods`). The author can change that page or link at any time. I found no data-upload code in what I could read. It is removed.
3. **The four odd asset files are not the mod's.** They are XOR-encoded config for the app's own downloader (site list, page scripts, a YouTube URL pattern), read by `libitcore.so`. No outside server in them.
4. **The real app protects itself.** `dev.in.decode.Decoder` (`libitcore.so`) embeds the developer's exact certificate (809 bytes) and refuses to load without it. 14 near-identical checks (`ez4` … `rz4`) read a 31-character slice of the certificate and kill the process on mismatch. This is why the mod faked the certificate. I kept only the small pure-Java `killPM` hook for that (see section 4).
5. **The app hides ~870 of its own strings** behind a native decoder. I reverse-engineered it (base64, then XOR on every second byte, with a key chosen by the string's 8-character argument) and decoded 869 of 873. Result: 502 unique strings. All point at video sites the downloader scrapes (Bilibili, Dailymotion, Rutube, Reddit, X, imgur, `cdnfast.sbs`, and several adult video sites) plus HTTP header names. **No telemetry, upload or control server, no shell commands, no passwords.**
6. **Cookies stay local.** The downloader reads each site's cookie and puts it into the download request for the same site, or checks it for a login marker. Nothing goes to a third party.
7. **Native libraries.** None of the 13 other libraries imports `exec`, `fork`, `system` or `popen`. Only `libinxffm` (FFmpeg) opens sockets, as expected for streaming. No hard-coded hosts.

## 3. All findings and what happened to each

| ID | Severity | Finding | Result |
|---|---|---|---|
| F-01 | High | Signature-spoofing code runs first at startup (`bin.ghost.yrf`) | **Replaced, not just removed.** Unknown native library, native helper and file redirect are cut. The small pure-Java `killPM` stays because the app's own anti-tamper needs it (Change 1) |
| F-02 | High | Hidden native library `libyrf.so`, remote Pastebin table and Telegram link | **Fixed** — removed with its duplicate `classes3.dex` |
| F-03 | High | Re-signed with a weak generic key (`youarefinished`) | **Fixed as far as possible** — now signed with your own RSA-4096 key. The developer's signature cannot be restored |
| F-04 | High | "Smob" obfuscation that breaks decompilers | **Not reversed in the APK** (it does not change what the code does). **Worked around for analysis**, so the code could be read |
| F-05 | Medium | Cleartext HTTP allowed app-wide | **Fixed** (Change 2) — see side effects |
| F-06 | Medium | Trust-everything HTTPS certificate checker + allow-all hostname check | **Fixed** (Change 5) |
| F-07 | Medium | Device-admin receiver | **Fixed** — receiver removed (Change 3). File Protection feature is lost |
| F-08 | Medium | Sensitive permissions | **Fixed**: `READ_CLIPBOARD` removed. **Main build keeps** all-files and overlay (you must grant them by hand). **Strict build removes** both. Code read shows the app works without them |
| F-09 | Low | Exported receivers | **Fixed** for `BootReceiver` and `ChangeMusicCovertReceiver` |
| F-10 | Medium | Extra downloader / status-saver package, cookie reads, JavaScript bridge | **Reviewed with readable code: no evidence of harm.** Not removed (see section 5) |
| F-11 | Low | Root/emulator/debugger detection (`za0`) | **Not changed** — harmless, removing adds crash risk |
| F-12 | Low | Shell commands | **Reviewed:** `getprop` with fixed keys (e.g. `ro.build.version.emui`), and an own-app logcat reader. **Not changed** |
| F-13 | Low | Third-party dynamic code loading (Google ads SDK) | **Not fixable** without removing ads |
| F-14 | Low | Ad/tracker SDKs, Firebase key, SDK weak hashes | **Not changed** |
| **F-15** | **Info (new)** | The real app's own anti-tamper and decoder depend on the original signature | **Handled** by keeping the Java hook; documented below |

## 4. Change 1, corrected

- Before: the mod's startup code loaded an unknown native library, called a native helper, installed the Java signature hook, and ran a file redirect.
- After: **only the Java signature hook is left.** It makes `PackageInfo.signatures` report the original certificate for this app's own package, inside this one process. It does not touch other apps.
- Why it stays: without it, `libitcore.so` fails to load (logs "signature is error"), `f82.g(...)` then returns raw undecoded text, and the 14 checks kill the process.
- The cost: the hook uses Java reflection and `sun.misc.Unsafe` to work. Google could break it on a newer Android version. If it does, the app will close itself or crash at start.

## 5. What was NOT fixed, and why

- **The Smob scrambling.** Removing it would not change what the code does. I did not rewrite 17,000 classes.
- **The downloader module (F-10).** It is a real feature. I decoded its data and read its bridge. It lets the page you are viewing hand video links to the app, and the app fetches pages and sites you open. I found no upload code. A malicious *web page* loaded in its browser could still call the bridge, so **don't browse unknown sites or log in to Instagram, Facebook or X inside it.**
- **All-files and overlay permissions in the main build.** Use the strict build if you want them gone.
- **Root/emulator checks (F-11), shell commands (F-12), ad SDK loader (F-13), trackers (F-14).** Standard SDK or app behavior. Removing them adds crash risk and no real safety.
- **The anti-tamper itself.** I did not remove the app's own self-checks. I made them pass, as the mod did, by keeping `killPM`.
- **The official APK comparison.** It could not be done. The one mirror I found blocked automated downloads, and I did not try to bypass that. So "what the mod added" comes from inside this APK only. This also means I **cannot prove** the downloader module is original.

## 6. Remaining risk (read this)

1. I read the code, but I cannot prove there is no hidden logic. I found no sign of any.
2. I did not disassemble the other 13 native libraries. They were judged by what they import and the strings they hold.
3. ~4 of 873 hidden strings and some call sites with computed strings were not decoded.
4. Ad and tracking SDKs (Google Ads, InMobi, Firebase) still run and still send device and ad-ID data. Normal for the real app too.
5. The in-process certificate hook (section 4) is a hack. It can fail on a future Android release.
6. The app can still read a lot if you let it: photos and files if you grant storage, and the internet.
7. No official signature: Play Protect cannot vouch for it, and it will not update itself.
8. The app was run in only one environment: a slow, software-emulated Android 11 phone (section 9). Android 12 to 15, a real phone, the browser/downloader, casting and in-app purchases were not run.

## 7. What to test in an Android emulator (and what could crash)

Use an emulator (Android 13 and 14 images with Google APIs; also try 15), with no Google account and no personal files. Watch network traffic with PCAPdroid or mitmproxy. Install the **main** build first.

**Must-pass**
1. **Installs and starts, and stays open.** If it closes by itself right after launch, the signature hook failed: check `adb logcat` for `signature is error`, `UnsatisfiedLinkError`, `ExceptionInInitializerError`, or `killProcess`.
2. **Play a local video** and a video over **HTTPS**.
3. **No outbound call at start** to `pastebin.com` or `t.me`. (It should now only talk to the video/ad/Firebase hosts.)
4. **Open the built-in browser/downloader screen** — it should load. Do not log in to anything.

**Things that may break or behave differently**
- **`http://` videos, DLNA/UPnP casting to a TV, `http://` sites in the browser** — by design (Change 2).
- **"File Protection"** — needs a device-admin request that no longer exists.
- **HTTPS servers with self-signed or wrong certificates** — now fail.
- **In-app purchases / VIP state** — the mod may have patched billing; Play Billing sees a different real signature than the hook shows.
- **Strict build:** the floating (pop-up) player is off. File browsing relies on media permissions only.
- **A `VerifyError` or `ClassFormatError` at launch** would mean one of my byte patches is wrong. Each is a straight `nop`, same-target `goto` or untouched code, and the DEX re-parses, so the risk is small. Send me the `logcat` if it happens.

## 8. Not verified (summary)

- Anything at run time, or that arrives from a server after install (Firebase Remote Config is present).
- The machine code of the 13 other native libraries.
- Whether the downloader module is in the official app.
- That the signature hook works on your exact Android version.
- Behavior on a real device, and on Android versions other than 11 (the in-process certificate hook is the part most likely to break on a newer Android).

## 9. Emulator test results (run in this environment)

**Setup.** Android 11 (API 30) x86_64 Google APIs image, headless, in **software emulation** (this environment has no hardware virtualization), 2 virtual CPUs, 3 GB RAM. The image runs this app's ARM-only native libraries through its ARM translation layer. Everything is very slow, so Android's own watchdog restarted the system process several times during boot and install. That is an environment limit and not an app problem. Evidence: `evidence/` (7 screenshots and `emulator_test_log.txt`).

| Test | Build | Result |
|---|---|---|
| Install | main | **Passes.** Android picked `arm64-v8a`, saw my new signing key |
| Start | main | **Passes.** No crash, no process kill, no `signature is error`, no `UnsatisfiedLinkError`. Splash, then the app's first-run permission screen |
| Native libraries | main | **All 10 app libraries loaded**, including `libitcore.so`. Its `JNI_OnLoad` runs the signature check and unloads on failure, so **the signature hook works and the app's anti-tamper accepted it** |
| Library screen | main and strict | **Passes.** "Premium" screen, folders, my test video found and shown |
| Thumbnail | main | **Passes.** The app decoded a frame from the test clip and read its length (00:06) |
| Play a video | **main (as shipped)** | **Fails in this emulator only.** The app shows "Playback failed" (its string `load_so_error_content`). Cause: it has a **10-second watchdog** on loading its native player libraries (`gh2.a` posts a 10,000 ms timer). Loading took 16.5 s and 32.6 s here, but did finish. On a normal phone loading takes well under a second |
| Play a video | test-only copies (watchdog stretched; 3 bytes differ) of **main** and **strict** | **Passes.** The app opened the clip from its library. Its own process logged `AudioTrack: stop: 265,344 frames delivered` = 6.02 s = the whole clip. The player closed itself on completion. The thumbnail got the "watched" progress bar. No crash |
| Strict build (no all-files, no overlay) | strict (test-only copy) | **Passes** the same library playback test. No special permission needed |
| Open by file path from outside (`file://` VIEW) | main | The player opens but asks "XPlayer can't access this video. Please allow access to All Files". This is Android 11 scoped storage and the app's own rule. The strict build cannot satisfy it. Opening from the app's library works without it |

**What this does and does not prove**
- It proves the fixed app is not broken in the ways I feared: no self-kill, the native libraries and certificate check work, and a real video decodes and plays through the app's own FFmpeg pipeline.
- It does **not** prove that picture frames were rendered correctly. The emulator was too slow to catch a frame on screenshot. The evidence for playback is the full audio delivery, the clean player lifecycle (open at 21:58:56, finish at 22:00:27), and the watched-state change.
- The two shipped APKs differ from the tested copies by one constant (the 10-second watchdog, 3 bytes plus the DEX header). I tested copies and not the shipped bytes, because the shipped bytes cannot finish loading inside this emulator's 10-second window.
- Only Android 11 was tested. Android 12 to 15 were not.
- The browser/downloader, casting, in-app purchase, notifications, picture-in-picture and background playback were not tested.
- Mistake worth knowing about: early in testing I hid Android's error dialogs, which made Android kill the app after its first "not responding" prompt. I turned that off and re-ran. The earlier "kills" were caused by that setting, not by the app.

**Test-only copies** (same signing key, same code except the 10-second timer, for slow emulators only; you do not need them on a real phone or a hardware-accelerated emulator):
`/tmp/claude-0/-home-user-XPlayer-safe/49bf11d1-7853-5759-a5cd-203f796f3864/scratchpad/final/XPlayer_TESTONLY_signed.apk` and `.../XPlayer_STRICT_TESTONLY_signed.apk`.
