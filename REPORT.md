# REPORT.md — Xplayer.apk: findings, fixes, and what is still risky

## 1. Summary

- The APK was a modded XPlayer 3.0.0 (`video.player.videoplayer`). Phase 1 found 14 findings. Phase 2 fixed the ones that could be fixed safely, one at a time, and checked each fix before the next.
- **This is not a claim that the app is safe.** I cannot make it "100% safe", and nobody can from static analysis. About 85% of the app's code is deliberately scrambled so that normal tools cannot read it. I checked it by scanning bytecode for known dangerous calls, which cannot prove the absence of hidden logic. I also could not run the app.
- What changed: the signature-spoofing hook, the hidden native library (with its remote Pastebin/Telegram link), the trust-everything HTTPS code, the device admin, plain-HTTP traffic, one useless permission, and two unneeded exported receivers. The APK is re-signed with a key you control.
- What did not change: the Smob scrambling, the ad/tracker SDKs, the downloader module, the all-files and overlay permissions, root/emulator checks, and the other items in section 4.

**Deliverables**

| File | Where |
|---|---|
| Cleaned, signed APK | `/tmp/claude-0/-home-user-XPlayer-safe/49bf11d1-7853-5759-a5cd-203f796f3864/scratchpad/final/XPlayer_cleaned_signed.apk` (43,333,739 bytes, SHA-256 `899ae278…080a10`) |
| Findings | `FINDINGS.md` (this repository) |
| Every change, with diffs and byte patches | `CHANGES.md` (this repository) |
| This report | `REPORT.md` (this repository) |
| Keystore | `/tmp/claude-0/-home-user-XPlayer-safe/49bf11d1-7853-5759-a5cd-203f796f3864/scratchpad/signing/xplayer-safe.jks` |
| **Keystore password (separate note)** | `/tmp/claude-0/-home-user-XPlayer-safe/49bf11d1-7853-5759-a5cd-203f796f3864/scratchpad/signing/KEYSTORE_PASSWORD.txt` |

The keystore and password are **not** in the repository (`.gitignore` blocks `*.apk`, `*.jks`, `*.keystore`, `KEYSTORE*`, `*PASSWORD*`). The scratchpad folder can be cleared when the session ends, so copy these files somewhere safe now. If you lose the keystore, you cannot ship an update that installs over this build.

## 2. All findings and what happened to each

| ID | Severity | Finding | Result |
|---|---|---|---|
| F-01 | High | Signature-spoofing code runs first at startup (`bin.ghost.yrf`) | **Fixed** (Change 1) |
| F-02 | High | Hidden native library `libyrf.so`; remote Pastebin table and Telegram link; anti-tamper | **Fixed** — library and its duplicate `classes3.dex` removed (Change 1) |
| F-03 | High | Re-signed with a weak generic key (`youarefinished`) | **Fixed as far as possible** — now signed with your own RSA-4096 key. The original developer's signature cannot be restored (Change 7) |
| F-04 | High | "Smob" obfuscation that breaks decompilers | **Not fixable** — cannot be reversed without the original build. Worked around it (see section 4) |
| F-05 | Medium | Cleartext HTTP allowed app-wide | **Fixed** (Change 2) — see side effects |
| F-06 | Medium | Trust-everything HTTPS certificate checker, plus allow-all hostname check | **Fixed** (Change 5) |
| F-07 | Medium | Device-admin receiver | **Fixed** — receiver removed (Change 3). File Protection feature is lost |
| F-08 | Medium | Sensitive permissions | **Partly fixed** — `READ_CLIPBOARD` removed. **Kept:** `MANAGE_EXTERNAL_STORAGE`, `SYSTEM_ALERT_WINDOW` (see section 4) |
| F-09 | Low | Exported receivers | **Fixed** for `BootReceiver` and `ChangeMusicCovertReceiver`. Player/share entry points stay exported on purpose |
| F-10 | Medium | Extra downloader / status-saver package (`a.m.z.*`), cookie reads, Instagram login | **Not fixed** — see section 4 |
| F-11 | Low | Root/emulator/debugger detection (`za0`) | **Not fixed** — see section 4 |
| F-12 | Low | Fixed shell commands (`getprop`, logcat) | **Not fixed** — see section 4 |
| F-13 | Low | Third-party dynamic code loading (Google ads SDK) | **Not fixed** — see section 4 |
| F-14 | Low | Ad/tracker SDKs, Firebase API key, weak hash names in SDKs | **Not fixed** — see section 4 |

## 3. How each fix was checked

- **F-01/F-02:** re-parsed the final DEX: `bin.ghost.yrf.<clinit>` starts with `return-void`; no `libyrf`, `classes3`, `pastebin` or `youarefinished` string anywhere in the APK.
- **F-05/F-07/F-08/F-09:** read the **compiled** manifest and network config in the final APK: cleartext off, no device admin, no `READ_CLIPBOARD`, both receivers `exported=false`.
- **F-06:** re-parsed the DEX: the `sslSocketFactory` calls and the allow-all hostname verifier set are gone; the `qe0` branch is now an unconditional jump to the old skip target.
- **Whole package:** ZIP test passes; `apksigner verify` passes (v2 and v3); DEX checksum and SHA-1 verified; all 17,700 classes and 78,000+ methods still parse; resource names identical to the original (13,108 vs 13,108); `aapt` badging unchanged (package, version, SDKs, launcher).
- **Re-scan:** I re-ran the Phase 1 scan. Counts are the same except for the removed duplicate `bin.ghost` copy. Most categories are normal ad-SDK, Google or OkHttp code, so a count that stays the same does not mean "still a problem". The specific fixes were proven by the targeted checks above.

## 4. What was NOT fixed, and why

- **F-04 Smob obfuscation.** It is baked into the bytecode. I did not try to "unscramble" it. Because apktool cannot read those classes, I changed only exact bytes in the DEX and avoided a full rebuild.
- **F-08 all-files and overlay permissions.** The app's permission library refuses to request a permission that is missing from the manifest, so removing these two could crash the app when it asks. Neither is granted automatically: you must switch each on in Android Settings. **Don't grant them.** I can build a "strict" variant without them for you to test in an emulator if you want.
- **F-10 downloader package.** The decoded asset files show it is a real downloader feature (site list, page scripts for Instagram/Facebook/X, a bridge to the app). I found no outside server and no upload code. But I could not compare against the official app, so I cannot say if the mod added it, and I cannot prove it is harmless. **Don't log in to Instagram, Facebook or X inside this app.**
- **F-11, F-12, F-13, F-14.** Read-only or standard SDK behavior. Removing them would add crash risk and no real safety. The Google Mobile Ads SDK still downloads and loads code at run time; I cannot review that code.
- **Comparison with the official APK was not done.** The only mirror I found refused automated downloads, and I did not try to get around that. So "what the mod added" in section 5 of `FINDINGS.md` comes from inside this APK only.

## 5. Remaining risk (read this)

1. **~85% of the code is unreadable.** My checks cover known dangerous APIs and strings. They do not cover logic hidden through reflection, string building, or native code in the other libraries.
2. **The other 13 native libraries were checked only by strings.** They look like standard media libraries (dav1d, x264, FFmpeg/OpenSSL, mmkv, mp3lame), but I did not disassemble them.
3. **The mod author already controlled the first code to run.** I removed it, but I cannot see what else they changed inside the scrambled classes.
4. **Ad and tracking SDKs** (Google Ads, InMobi, Firebase) still run and still send device and ad-ID data. That is normal for the real app too.
5. **The app can still read a lot if you let it.** Photos and files if you grant storage; clipboard writes; internet access.
6. **No official signature.** Play Protect cannot vouch for it, and it will not update itself. Any future change needs your key.

## 6. What to test in an Android emulator (and what could crash)

I could not run the app. Use an emulator (Android 11 and 14 images, with Google APIs), with no Google account and no personal files. Watch the network with a tool like PCAPdroid or mitmproxy.

**Must-pass**
1. **Installs and starts.** Install the APK fresh. If it does not install, check that the old modded build is uninstalled first (different signature).
2. **Cold start does not crash.** The startup code was patched at the very first instruction; a crash here would be in `XPlayerApplication → y90 → bin.ghost.yrf`. Check `adb logcat` for `UnsatisfiedLinkError`, `ClassNotFoundException`, `VerifyError`.
3. **Play a local video** and a video over **HTTPS**.
4. **No hidden dialog or outbound call at start.** Confirm no request to `pastebin.com` or `t.me`.

**Things that may break or behave differently**
- **`http://` videos, DLNA/UPnP TV casting, and `http://` sites in the built-in browser** — by design (Change 2).
- **"File Protection"** — the device-admin request will not work (Change 3).
- **HTTPS to servers with bad certificates** — these now fail (Change 5).
- **In-app purchase / VIP / premium state** — the mod may have patched it, and the app no longer reports the original certificate. Some SDKs (Google Play Billing, Firebase) read the app signature, so billing, login, or remote-config features may behave differently.
- **The downloader and browser** — check they open without crashing, but do not sign in to anything.
- **A `VerifyError` or `ClassFormatError`** on first launch would mean one of my byte patches is wrong. The risk is small (each patch is a straight `return-void`, `nop`, or same-target `goto`, and the DEX re-parses), but it is the first thing to look for. Send me the `logcat` and I will fix it.

## 7. Not verified (summary)

- Anything at run time; anything that arrives from a server after install.
- The scrambled 85% of the code beyond the API/string scan.
- The machine code of all native libraries.
- Whether the downloader module is in the official app.
- Behavior on a real device.

## 8. Optional strict build

`XPlayer_strict_signed.apk` (in the same `final/` folder) also removes `MANAGE_EXTERNAL_STORAGE` and `SYSTEM_ALERT_WINDOW`. It is untested. See the end of `CHANGES.md`. Test it first in an emulator: open the app on a fresh install, go through the permission screen, play a local file, and check `adb logcat` for `IllegalStateException` / "You must add ... permission" messages from the permission library.

## 9. About the "100% safe" goal

This goal cannot be met honestly. Static analysis cannot prove an app is free of hidden behavior, and this app's code is deliberately scrambled. The right target is "removed everything I could find and could safely remove, and said clearly what I could not check". That is what this report does.
