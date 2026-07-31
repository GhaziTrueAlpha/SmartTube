# Handoff: SmartTube "cast wake from sleep" bug — still unresolved

## Project
- Repo: `D:\Coding Space\Smarttube\SmartTube` (SmartTube — unofficial YouTube client for Android TV)
- Package under test: `org.smarttube.stable` (the `ststable` build flavor)
- Build output: `smarttubetv/build/outputs/apk/ststable/debug/`
- TV: Android 14 (API 34), target/compile SDK 34 in the app, but `SharedModules/constants.gradle` sets `targetSdkVersion = 27` for most modules (legacy). USB + wireless ADB debugging always on.
- Git submodules `SharedModules` and `MediaServiceCore` are used; they were not initialized in this dev environment and had to be `git submodule update --init --recursive`'d to build.

## The bug (as originally reported by the user, in their words)
When casting a YouTube video from the phone's YouTube app to this TV (using SmartTube's own reimplementation of YouTube's "Lounge API" remote-control/cast protocol — NOT Google's real Cast Framework/DIAL):
1. If the TV is asleep (screen off) and a cast command arrives, the TV does **not** wake itself up automatically. Only manually pressing power on the TV causes the already-queued video to start.
2. Within ~1-3 seconds of the screen going off, the wake *does* work — but any longer than that, it stops working entirely.
3. Some time after the TV sleeps, SmartTube disappears from the YouTube app's cast device list. When the TV is later turned on **manually**, SmartTube shows as "already closed" (its process was killed), yet it still (confusingly) reappears in the cast list once the TV is awake — even before the app is manually reopened.
4. User's actual goal: cast should wake the TV reliably regardless of how long it's been asleep, and the TV should remain visible in the phone's cast list *even while asleep*, indefinitely (background service must survive).

## Root causes diagnosed, in order of discovery

### 1. `Utils.turnScreenOn(Context)` was a no-op from background
`ViewManager.movePlayerToForeground()` called `Utils.turnScreenOn(mContext)` to wake the screen, but that method only works `if (context instanceof Activity)`. `ViewManager.mContext` is always the Application context, so the call silently did nothing — the video state updated in the background but the screen never lit up.

### 2. `RemoteControlService` held no wake lock
The foreground service that keeps polling YouTube's Lounge API (so the TV stays listed as a cast target) had no `PowerManager.WakeLock`, so the CPU could deep-sleep and kill the background long-poll connection shortly after screen-off.

### 3. Background Activity Launch (BAL) restrictions (Android 10+, tightened in 14)
A plain `startActivity(FLAG_ACTIVITY_NEW_TASK)` call from a background/Service context is only allowed by Android for a short grace period (a few seconds) after the app was last visible. This exactly explains the "works within 1-3 seconds, fails after" symptom — it's not a timing/wakelock issue, it's Android silently blocking the activity launch once the grace window expires.

### 4. `ForegroundServiceStartNotAllowedException` (Android 12+, relevant on 14)
The existing code already had a comment anticipating this:
```java
try {
    startForeground(NOTIFICATION_ID, createNotification());
} catch (Exception e) {
    // ForegroundServiceStartNotAllowedException: Service.startForeground() not allowed due to mAllowStartForeground false (Android 14)
    e.printStackTrace();
}
```
This exception is silently swallowed. When it fires, `RemoteControlService` fails to actually become a protected foreground service, so Android kills it quickly afterward — explaining both the "app already closed" symptom and the cast-list disappearance.

### 5. Missing runtime permission grant
The manifest already declared `<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />` (a documented exemption from both BAL and FGS-start restrictions), but **the app never actually requested it at runtime** — declaring a special permission in the manifest does not grant it; the user must approve it via a settings screen (`Settings.ACTION_MANAGE_OVERLAY_PERMISSION`). This call was completely missing from the codebase.

## Fixes implemented (uncommitted, 6 files modified)

Run `git status --short` / `git diff` in the repo root to see the exact diffs. Summary:

1. **`common/src/main/AndroidManifest.xml`**
   - Added `WAKE_LOCK`, `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`, `USE_FULL_SCREEN_INTENT` permissions.

2. **`common/src/main/java/.../common/utils/Utils.java`** (main logic, ~160 new lines)
   - `wakeUpScreen(Context)` — new method using a `PowerManager.WakeLock` (`SCREEN_BRIGHT_WAKE_LOCK | ACQUIRE_CAUSES_WAKEUP | ON_AFTER_RELEASE`) that works from any context (unlike the old `turnScreenOn`).
   - `wakeUpAndOpenActivity(Context, Class<? extends Activity>)` — posts a **full-screen-intent notification** (`NotificationCompat.Builder.setFullScreenIntent(pendingIntent, true)`, `setCategory(CATEGORY_CALL)`, `IMPORTANCE_HIGH` channel) to launch the target Activity. This is one of Android's few background-activity-start paths that is **not** subject to the BAL grace-period restriction — the same mechanism incoming-call/alarm apps use.
   - `requestIgnoreBatteryOptimizations(Context)` — checks `PowerManager.isIgnoringBatteryOptimizations()` and if not exempted, fires `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`.
   - `requestOverlayPermission(Context)` — checks `Settings.canDrawOverlays()` and if not granted, fires `ACTION_MANAGE_OVERLAY_PERMISSION` (this is the fix for root cause #5).
   - `startRemoteControlWorkRequest`: `PeriodicWorkRequest` interval reduced from 30 min to 15 min (WorkManager's minimum) so the background service recovers faster if killed.

3. **`common/src/main/java/.../common/app/views/ViewManager.java`**
   - `movePlayerToForeground()` now calls `Utils.wakeUpAndOpenActivity(mContext, activityClass)` (resolving the target Activity class from `mViewMapping.get(PlaybackView.class)`) instead of the broken `turnScreenOn`, then still calls the normal `startView(PlaybackView.class)` afterward for state tracking.

4. **`common/src/main/java/.../common/misc/RemoteControlService.java`**
   - Acquires a `PowerManager.PARTIAL_WAKE_LOCK` in `onCreate()` (helpers `acquireWakeLock()`/`releaseWakeLock()`), released in a new `onDestroy()` override.

5. **`common/src/main/java/.../common/app/presenters/SplashPresenter.java`**
   - `runOnceTasks()` now calls `Utils.requestIgnoreBatteryOptimizations(getContext())` and `Utils.requestOverlayPermission(getContext())` once per app process lifetime, at first launch.

6. **`smarttubetv/src/main/java/.../tv/ui/playback/PlaybackActivity.java`**
   - `onResume()` also calls `Utils.turnScreenOn(this)` (now with a real Activity context) to dismiss the keyguard / set screen-on window flags.

## Build/compile verification done
- `:common` module (contains all the changed Java files) was confirmed to **compile successfully** (`BUILD SUCCESSFUL`) via manual `gradlew :common:compileStbetaDebugJavaWithJavac` after setting up JDK 17 (Gradle 7.5 can't run on JDK 21+), Android SDK, and initializing submodules. No compile errors in any of the 6 changed files.
- A separate, **pre-existing, unrelated** Windows-only bug in the old `exoplayer-amzn-2.10.6` submodule's `javadoc_library.gradle` (an `android.sdkDirectory` access throwing `IOException` during AGP configuration) blocks a *full* end-to-end Gradle build of the whole app in this dev environment specifically. This was confirmed to reproduce identically on a clean `master` checkout with **none** of the above changes — it is not caused by this work. It was worked around locally (temporarily neutering that gradle file) purely to get `:common` to compile-check, then reverted. **This does not block the user's own Android Studio build** — they've successfully built and installed APKs multiple times already (see below).

## User's own testing (their machine, real TV, real builds — this is the important part)

The user built and installed the APK themselves via Android Studio (this works fine for them; the build issue above is dev-environment-specific to the assistant's sandbox, not the user).

- **SYSTEM_ALERT_WINDOW was confirmed granted** via ADB: `adb shell appops get org.smarttube.stable SYSTEM_ALERT_WINDOW` → `SYSTEM_ALERT_WINDOW: allow`.
- User tested "does the TV's network/CPU survive sleep at all" via `adb shell input keyevent 224` (KEYCODE_WAKEUP) while the TV was asleep — **it worked, TV woke instantly**. This proves the TV's SoC/network is NOT in a hardware deep-suspend during "sleep" — it's a software-level Android sleep, meaning a software fix should be possible in principle.
- User captured a `logcat` around a sleep/wake/cast cycle. **Key finding from that log**: multiple times, the log showed:
  ```
  ActivityTaskManager: Background activity start for org.smarttube.stable allowed because SYSTEM_ALERT_WINDOW permission is granted.
  ActivityTaskManager: START ... PlaybackActivity ... (BAL_ALLOW_SAW_PERMISSION) result code=0
  ```
  — with `isSleeping=true` in the TaskInfo at the time. **This proves the SYSTEM_ALERT_WINDOW-based background activity launch fix DOES work** — Android itself logged that it allowed the launch specifically because of the granted SAW permission, while the device was asleep.
  - Also saw multiple `Background started FGS: Allowed [... code:PROC_STATE_TOP ...]` entries for `RemoteControlService` with no `ForegroundServiceStartNotAllowedException` visible anywhere in the captured log.
- **However, the log was contaminated**: it also contained several `Force stopping org.smarttube.stable ... installPackageLI` / `PackageManager: Update package org.smarttube.stable code path from ... to ...` entries — i.e., the user was reinstalling new APK builds (`adb install -r`) *during* the capture window, which force-kills the running process as a normal side effect of reinstalling. This makes it impossible to tell, from that log alone, whether there's a *genuine* OS/battery-manager kill happening during real idle sleep, versus all the "app died" events being artifacts of redeploying builds mid-test.

**A clean re-test was requested but not yet completed/reported**: install once, don't touch ADB/reinstall again, sleep the TV, wait 5-10+ minutes untouched, then cast from phone and observe + capture a *clean* `adb logcat -c` → wait → `adb logcat -d` cycle with no reinstalls in between.

## Current status / what's unresolved

Despite the fixes above (which have direct evidence of working for the BAL/activity-launch part in the user's own captured log), the user's most recent message says: **"sab kar liya (permissions, testing) — still it doesn't wake up SmartTube."** This is the last message before requesting this handoff — i.e., **the fix is not fully working end-to-end from the user's perspective**, despite the log evidence above suggesting the activity-launch mechanism itself is functioning at least some of the time.

Possible explanations not yet ruled out:
1. The APK the user was actually testing at the time of the "still doesn't work" report may not have been the latest build with all 6 fixes (given the log shows evidence of multiple reinstalls/version churn during testing — `versionName=32.10` was seen in one snapshot).
2. There could be a *separate*, not-yet-diagnosed failure specifically in the Lounge-API/cast-command-receiving path itself (i.e., maybe the wake code is correct but the trigger — the actual incoming cast command from the phone — never arrives/fires `movePlayerToForeground()` after the connection has been idle a while, independent of all the Activity-launch/FGS fixes). This has **not been directly diagnosed** — all investigation so far focused on "can we launch/wake the Activity," not on "does the incoming Lounge API bind/command actually reach the app after N minutes asleep." **This is the most likely next investigation area.**
3. OEM-specific background killing beyond stock AOSP Doze/App-Standby/BAL — not fully ruled out. TV brand/model was never identified (asked twice, user didn't answer either time — worth asking again, or extracting via `adb shell getprop ro.product.manufacturer` / `ro.product.model`).
4. `cached_apps_freezer` (Android 14's new process-freezing feature) was suggested as a possible contributor (`adb shell settings put global cached_apps_freezer disabled`) but whether the user actually ran this ADB command is unconfirmed.
5. The `RemoteControlWorker` (WorkManager `PeriodicWorkRequest`, now 15-min interval) that's supposed to restart `RemoteControlService` if it dies — its actual behavior/logs during a long sleep were never directly inspected.

## Suggested next steps for whoever picks this up

1. **Get the TV brand/model** (`adb shell getprop ro.product.manufacturer` / `ro.product.model`) — still unknown, asked twice, never answered. Also note: the katniss package-cache log dump in the captured logcat lists installed apps including `com.realtek.tv`, `com.realtek.leanbacklauncher.partnercustomizer`, `com.apps.tvmanager`, `com.apps.ota`, `com.mk.tv.timeclock` — **this strongly suggests a Realtek-chipset generic/white-label Android TV** (not a major-brand TV), which often ships with a custom lightweight OEM shell with its own background-app management quirks (`com.apps.tvmanager` in particular is suspicious and worth investigating — it appeared in the log as an actively-managed foreground process around the same time; may be an OEM "TV manager"/power-management app worth checking `adb shell dumpsys package com.apps.tvmanager` and potentially testing with it force-stopped or its background restriction disabled).
2. **Get a clean logcat** (no reinstalls during capture) covering: app install once → wait → sleep TV → wait 5-10+ min untouched → cast from phone → note exact wall-clock time of the cast attempt → stop capture. Search for what happens to `org.smarttube.stable` process/service in the window between sleep and the cast attempt, and specifically whether `RemoteControlService`'s Lounge API long-poll/bind connection is still alive right before the cast command should arrive (may need to add temporary `Log.d` instrumentation in `RemoteControlService`/`RemoteController`/wherever the Lounge API bind response handling is, to confirm whether the incoming cast command is even received server-side, vs received-but-not-acted-on).
3. **Investigate `com.apps.tvmanager`** and similar bundled OEM apps as possible sources of aggressive background-process killing outside AOSP's normal Doze/Standby/BAL/FGS system (this class of OEM app is not addressable via any standard Android permission or wake lock — would require either disabling it via ADB `pm disable-user`, or documenting it as a required manual TV-settings change for the user).
4. Confirm whether `cached_apps_freezer` was actually disabled via ADB, and whether that changes anything.
5. Directly check whether the *cast command itself* (the network trigger from Google's Lounge API backend to the TV) is being received at all after long idle periods — this is the biggest unverified assumption. If the incoming bind/command never arrives (e.g., because the long-poll HTTP connection itself silently died despite the wake lock, or because of a network-level idle timeout on the TV's WiFi/router), then all the Activity-launch and FGS fixes are moot — the trigger never fires in the first place. This needs to be isolated from the "can we successfully launch/wake once triggered" question, which the current fixes have already been shown (in the user's own log) to handle correctly.

## Environment notes for whoever continues this
- Windows machine, PowerShell/CMD (user prefers CMD syntax for adb commands — no `grep`, use `findstr`).
- ADB (USB + wireless) always available and enabled on the TV.
- No commits have been made yet — all 6 files are uncommitted working-tree modifications. Decide whether/when to commit once the fix is fully confirmed working.
