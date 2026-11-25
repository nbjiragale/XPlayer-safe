# CHANGES.md — every change made to Xplayer.apk (Phase 2: FIX)

Original: `Xplayer.apk`, SHA-256 `08fc8cca89c1315954a9d9dd82218dc901ef58e334e1b2b7f778f8dc41988847` (never edited).
Result: `XPlayer_cleaned_signed.apk`, SHA-256 `03ed706b8fb4be77f2224b5f518f2f3f526ca8c85158410d55850b4697137128`.

Method: the code was **not** rebuilt from smali. apktool and jadx cannot read ~85% of the classes (Smob obfuscation), so a normal rebuild would have deleted them. Instead:
1. `apktool d -s` (code left untouched), edit manifest and resources, `apktool b`.
2. DEX bytecode changed by a small script that overwrites exact bytes (it refuses to run if the old bytes do not match), then recomputes the DEX checksum and SHA-1.
3. `zipalign`, then `apksigner` with a new key.

Each change was done one at a time and checked before the next one.

---

## Change 1 — F-01 / F-02: remove the unknown native library and the remote-controlled parts, but keep the small Java signature hook the app needs

**Correction (later in Phase 2).** My first version of this change turned the whole startup method into an immediate `return`. That was wrong. The *real* app protects itself: it reads its own signing certificate and shuts the process down if it is not the developer's (14 inlined checks, class family `ez4` to `rz4`, plus `libitcore.so`, which refuses to load with any other signature). A re-signed app without the hook would silently close itself, and the string decoder would not load. The mod's Java hook `killPM` is what makes those checks pass. So the fix keeps only that hook. The earlier two APKs I sent are **superseded**.

**Before:** `bin.ghost.yrf.<clinit>` (the Application base class) ran first at every start and did four things:
1. `System.loadLibrary("yrf")` (unknown native library; also held `https://pastebin.com/raw/0NEmugw4` and `t.me/youarefinished_mods`)
2. `gho.z(true)` (native helper)
3. `killPM(pkg, <original cert>)` (pure Java: makes `PackageInfo` report the original certificate **inside this process only**)
4. `killOpen(...)` (file-open redirect; had empty arguments, so it already did nothing)

**After:** only step 3 remains. Exact DEX bytes (classes.dex, file offsets):
```
5442490 : 71 10 f5 b3 00 00  -> 00 00 00 00 00 00   nop  System.loadLibrary("yrf")
5442504 : 71 10 c1 22 00 00  -> 00 00 00 00 00 00   nop  gho.z(true)
5442550 : 71 40 dc 22 31 20  -> 00 00 00 00 00 00   nop  killOpen(...)
```
(The 4-byte `const-string` at 5442484 is unchanged from the original. Everything else changed in the header only: checksum and SHA-1.)

Files removed from the APK:
```
- lib/arm64-v8a/libyrf.so
- lib/armeabi-v7a/libyrf.so
- classes3.dex            (duplicate copy of the 19 bin.ghost classes; the live copy is in classes.dex)
- META-INF/YOUAREFINISHED.RSA / .SF / MANIFEST.MF  (old signature; replaced in Change 7)
```
Check: re-parsed the final `classes.dex`: the only real call left in `bin.ghost.yrf.<clinit>` is `killPM`; 13,149 classes still parse. No `libyrf`, `classes3`, `pastebin` or `youarefinished` string remains anywhere in the APK.

**Why keeping `killPM` is a trade-off:** it still lies about the certificate to code *inside this app*, and it uses Java reflection and `Unsafe` to do it. It needs to work on your Android version. If it fails, the app will close itself or crash at start (see REPORT.md, emulator tests). It does not touch other apps.

---

## Change 2 — F-05: turn off cleartext (plain HTTP) traffic
```diff
--- a/res/xml/network_security_config.xml	2026-11-25 18:01:21.117696503 +0000
+++ b/res/xml/network_security_config.xml	2026-11-25 18:01:21.119539797 +0000
@@ -1,4 +1,4 @@
 <?xml version="1.0" encoding="utf-8"?>
 <network-security-config>
-    <base-config cleartextTrafficPermitted="true" />
+    <base-config cleartextTrafficPermitted="false" />
 </network-security-config>
```
Check: compiled `res/xml/network_security_config.xml` shows `cleartextTrafficPermitted=0x0`.
**Side effect:** `http://` links, and DLNA/UPnP casting to a TV over your home network (plain HTTP), will likely stop working. HTTPS is unaffected.

---

## Change 3 — F-07: remove the device-admin receiver
```diff
--- a/AndroidManifest.xml	2026-11-25 18:02:00.452399779 +0000
+++ b/AndroidManifest.xml	2026-11-25 18:02:00.478163976 +0000
@@ -2043,14 +2043,6 @@
         <activity android:configChanges="keyboard|keyboardHidden|screenLayout|screenSize|smallestScreenSize|uiMode" android:name="com.inshot.xplayer.activities.AddWidgetActivity" android:theme="@style/NormalTheme"/>
         <activity android:name="com.inshot.xplayer.activities.IAPActivity" android:screenOrientation="userPortrait"/>
         <activity android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|screenSize|smallestScreenSize|uiMode" android:launchMode="singleTop" android:name="com.inshot.xplayer.activities.FileProtectionActivity" android:theme="@style/DarkTheme"/>
-        <receiver android:exported="true" android:name="com.inshot.xplayer.receiver.PlayerDeviceAdminReceiver" android:permission="android.permission.BIND_DEVICE_ADMIN">
-            <meta-data android:name="android.app.device_admin" android:resource="@xml/device_admin"/>
-            <intent-filter>
-                <action android:name="android.app.action.DEVICE_ADMIN_ENABLED"/>
-                <action android:name="android.app.action.DEVICE_ADMIN_DISABLE_REQUESTED"/>
-                <action android:name="android.app.action.DEVICE_ADMIN_DISABLED"/>
-            </intent-filter>
-        </receiver>
         <meta-data android:name="android.webkit.WebView.MetricsOptOut" android:value="true"/>
         <meta-data android:name="android.max_aspect" android:value="2.1"/>
         <activity android:alwaysRetainTaskState="true" android:configChanges="keyboard|keyboardHidden|orientation|screenSize" android:label="@string/app_name" android:launchMode="singleTop" android:name="a.m.z.activity.MainActivity" android:theme="@style/Theme.LightTheme" android:windowSoftInputMode="adjustPan"/>
```
`res/xml/device_admin.xml` was left in place (unused). The resource table still lists it, and deleting it can break the build.
Check: compiled manifest has no `PlayerDeviceAdminReceiver` and no `DEVICE_ADMIN` action.
**Side effect:** the app's opt-in "File Protection" feature (`FileProtectionActivity`) can no longer make the app a device admin. The code only asks the system "is it active?", so I expect no crash. Not tested.

---

## Change 4 — F-08 / F-09: remove `READ_CLIPBOARD`, un-export two receivers
```diff
--- a/AndroidManifest.xml	2026-11-25 18:02:51.196994789 +0000
+++ b/AndroidManifest.xml	2026-11-25 18:02:51.223514172 +0000
@@ -8,7 +8,6 @@
     <uses-permission android:name="android.permission.VIBRATE"/>
     <uses-permission android:name="com.google.android.providers.gsf.permission.READ_GSERVICES"/>
     <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
-    <uses-permission android:name="android.permission.READ_CLIPBOARD"/>
     <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
     <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
     <uses-permission android:name="android.permission.USE_FINGERPRINT"/>
@@ -1929,7 +1928,7 @@
         <service android:enabled="true" android:exported="false" android:name="com.inshot.xplayer.service.NotifyService"/>
         <service android:enabled="true" android:exported="false" android:foregroundServiceType="mediaPlayback" android:name="com.inshot.xplayer.service.PlayerService"/>
         <service android:enabled="true" android:exported="false" android:foregroundServiceType="mediaPlayback" android:name="com.inshot.xplayer.service.PipPlayerService"/>
-        <receiver android:exported="true" android:name="com.inshot.xplayer.receiver.BootReceiver">
+        <receiver android:exported="false" android:name="com.inshot.xplayer.receiver.BootReceiver">
             <intent-filter>
                 <action android:name="android.intent.action.BOOT_COMPLETED"/>
                 <category android:name="android.intent.category.HOME"/>
@@ -1975,7 +1974,7 @@
         <activity android:configChanges="keyboard|keyboardHidden|orientation|screenLayout|screenSize|smallestScreenSize|uiMode" android:launchMode="singleTop" android:name="com.inshot.xplayer.activities.VipTermsConditionsActivity" android:screenOrie
         <activity android:launchMode="singleTask" android:name="com.inshot.cast.xcast.ControlActivity" android:screenOrientation="userPortrait"/>
         <activity android:configChanges="keyboardHidden|orientation|screenLayout|screenSize|smallestScreenSize" android:name="com.inshot.xplayer.activities.AdActivity" android:theme="@style/SplashTheme"/>
-        <receiver android:enabled="true" android:exported="true" android:name="com.inshot.xplayer.receiver.ChangeMusicCovertReceiver">
+        <receiver android:enabled="true" android:exported="false" android:name="com.inshot.xplayer.receiver.ChangeMusicCovertReceiver">
             <intent-filter>
                 <action android:name="xp.change.music.covert"/>
             </intent-filter>
```
Check: compiled manifest: no `READ_CLIPBOARD`; `BootReceiver` and `ChangeMusicCovertReceiver` have `exported=false`.

---

## Change 5 — F-06: stop the app from accepting any HTTPS certificate

**Before:** two OkHttp clients (`bw6.t`, `x53.j`) were built with a trust-everything certificate checker `w53` (empty `checkServerTrusted`), and `x53.j` also installed an allow-all hostname verifier `v53(1)`. A third place (`qe0.<init>`) installed the same checker when config flag 128 was set.

**After:** those calls are removed, so OkHttp uses Android's normal certificate and hostname checks. Exact byte patches (file offset: before → after):
```
classes.dex  5550524 : 6e 30 6f f6 20 01  -> 00 00 00 00 00 00   bw6.t  : invoke y53.d (sslSocketFactory with w53) -> 3 nops
classes.dex 14587270 : 6e 30 6f f6 31 02  -> 00 00 00 00 00 00   x53.j  : invoke y53.d (sslSocketFactory with w53) -> 3 nops
classes.dex 14587296 : 5b 12 16 a5        -> 00 00 00 00         x53.j  : iput hostnameVerifier = allow-all -> 2 nops
classes2.dex 5610464 : 38 0b 15 00        -> 28 15 00 00         qe0.<init>: if-eqz v11,+0x15 -> goto +0x15 (same target; never builds the trust-all array)
```
Check: re-parsed both DEX files: none of those calls remain; the `goto` lands on byte 0x1ce, the same place the skipped branch used.
**Side effect:** a server with a self-signed or wrong certificate that used to "work" will now fail. That is the intended behavior.
The unused `w53` object is still created in three places but is no longer installed anywhere.

---

## Change 6 — nothing else changed in code or resources
`classes2.dex` changed only by Change 5 (one 4-byte patch plus header). Resource names: 13,108 before and after, none missing or extra (the rebuild folds a few `-v21`-style folders into their base folder because minSdk is 24).

---

## Change 7 — new signature
- Aligned with `zipalign -p 4`, signed with `apksigner` (schemes v2 + v3).
- New key: RSA 4096, SHA-256, `CN=XPlayer-safe local build`. Certificate SHA-256 `c304a3767607d153d040c4530a236184073e6723f059cceb4b49e1117475ead9`.
- Keystore password is in a **separate** file (see REPORT.md). It is not in the repository.
- `apksigner verify`: v2 true, v3 true.

---

## Optional variant — "strict" build (removes the two biggest permissions)

Same as the main build, plus (`XPlayer_strict_signed.apk`, SHA-256 `8a58e9872019c07b0e49311985da597290c595ab424999d7237ac0afc572fe94`, same signing key):
```diff
-    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
-    <uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE"/>
```
`classes.dex` is byte-identical to the main build.

**Why I expect it to work (code read, not run):**
- The app's own storage gates accept either all-files access **or** the media permissions: `ob5.t()` and `ob5.y()` return true if `READ_MEDIA_VIDEO` (and friends) *or* `isExternalStorageManager()` is granted. So the permission screen does not trap you without all-files access.
- The overlay permission is only *read* by the app (`Settings.canDrawOverlays`), and returns false when not granted. The floating/pop-up player is simply off.
- The permission library's strict storage check (`l85.g`) should not fire: `ScopedStorage=true` and `requestLegacyExternalStorage=true` are already set, and the media permissions stay declared. I found no place in the app's own code that asks the library for all-files or overlay access (the all-files screen is opened with a direct Settings intent, `ob5.P`). I did not trace every caller of the library, so this stays unverified.

**Not tested at run time.** Prefer the strict build if it opens and plays files; use the main build if it does not.
