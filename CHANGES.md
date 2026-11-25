# CHANGES.md — every change made to Xplayer.apk (Phase 2: FIX)

Original: `Xplayer.apk`, SHA-256 `08fc8cca89c1315954a9d9dd82218dc901ef58e334e1b2b7f778f8dc41988847` (never edited).
Result: `XPlayer_cleaned_signed.apk`, SHA-256 `899ae2784d490cfd2600273caaa8ccc93e45b7a47f5e1cfec57d9d0431080a10`.

Method: the code was **not** rebuilt from smali. apktool and jadx cannot read ~85% of the classes (Smob obfuscation), so a normal rebuild would have deleted them. Instead:
1. `apktool d -s` (code left untouched), edit manifest and resources, `apktool b`.
2. DEX bytecode changed by a small script that overwrites exact bytes (it refuses to run if the old bytes do not match), then recomputes the DEX checksum and SHA-1.
3. `zipalign`, then `apksigner` with a new key.

Each change was done one at a time and checked before the next one.

---

## Change 1 — F-01 / F-02: remove the signature-spoofing hook and the hidden native library

**Before:** `bin.ghost.yrf.<clinit>` (the Application base class) ran first at every start: it loaded `libyrf.so`, bypassed Android's hidden-API protection, and called `killPM(...)` to fake the app certificate. `libyrf.so` (both ABIs) also held a Pastebin address (`https://pastebin.com/raw/0NEmugw4`) and a Telegram link (`t.me/youarefinished_mods`) for a remote update/credits dialog.

**After:** `<clinit>` returns at once. `libyrf.so` (2 files) and `classes3.dex` (a duplicate of the `bin.ghost` classes) are deleted.

DEX bytes (classes.dex), at file offset 5,442,484:
```
before: 1a 04      const-string v4, "yrf"        (start of loadLibrary/killPM sequence)
after : 0e 00      return-void
```
The rest of the old method is dead code after `return-void`. Other bytes changed: only the DEX header checksum (offset 8) and SHA-1 (offset 12).

Files removed from the APK:
```
- lib/arm64-v8a/libyrf.so
- lib/armeabi-v7a/libyrf.so
- classes3.dex
- META-INF/YOUAREFINISHED.RSA / .SF / MANIFEST.MF  (old signature; replaced in Change 7)
```
Check: re-parsed the new `classes.dex`: `bin.ghost.yrf.<clinit>` first instruction is `return-void`; 13,149 classes still parse. No `libyrf`, `classes3`, `pastebin` or `youarefinished` string remains in any DEX, lib or resource.

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
