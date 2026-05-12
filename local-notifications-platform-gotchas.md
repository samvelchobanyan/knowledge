---
type: pitfall
severity: required
status: stable
applies_to: flutter-projects-with-local-notifications
keywords: [flutter-local-notifications, android, ios, core-library-desugaring, unusernotificationcenter, doze, vendor-battery, gradle, manifest]
related: [[local-notifications-stack]], [[adr-006-flutter-local-notifications-plus-timezone]], [[adr-007-inexact-alarm-as-default]]
---

# Local notifications: platform gotchas

Three known traps when integrating `flutter_local_notifications` into a project. Each can manifest as "notifications silently don't work" with no obvious error in the dart layer; all three live at the platform-config layer where most Flutter developers don't look first.

## Symptom

- **Build fails on Android** with: `Dependency ':flutter_local_notifications' requires core library desugaring to be enabled for :app.`
- **Notifications show on iOS in background but not foreground.** The user has the app open, the scheduled time arrives, nothing visible appears (no banner, no sound). Lock-screen alerts work; foreground alerts don't.
- **Notifications work intermittently on specific Android vendor devices** (Xiaomi, Huawei, Oppo, OnePlus) — fire reliably for a day or two after install, then stop. Reinstalling fixes it temporarily.

## Root cause

Three distinct mechanisms, all at the platform layer:

### 1. Android core library desugaring (FLN 17+)

`flutter_local_notifications` 17.x depends on Java 8+ time APIs (`java.time.*`) that are not available on older Android API levels by default. Gradle's "core library desugaring" feature backfills them — but it must be explicitly enabled in `android/app/build.gradle.kts`. Without it, the build fails at the AAR-metadata check before even compiling.

### 2. iOS UNUserNotificationCenter delegate

On iOS, foreground notifications are silently dropped by default unless the `UNUserNotificationCenter` delegate is registered in `AppDelegate.swift`. The `flutter_local_notifications` plugin's documentation covers this; the registration tells iOS that the app wants to be asked whether to show notifications while it's in the foreground.

Without the registration, the plugin schedules and delivers correctly, but iOS hides the visible alert when the app is active. The notification appears in the queue (visible if the user backgrounds the app), but the user sees nothing in real time.

### 3. Vendor battery optimization

Some Android device manufacturers (Xiaomi MIUI, Huawei EMUI, Oppo ColorOS, OnePlus OxygenOS) ship aggressive background-app killers that override standard Android `Doze` behavior. Apps not in the OEM's "protected" list have their scheduled alarms cancelled when the OS decides the device is "idle" — often within hours of app last use. This is **not solvable at the code level**; the OS overrides the AlarmManager registration.

## Fix

### 1. Android core library desugaring

In `android/app/build.gradle.kts`:

```kotlin
android {
    compileOptions {
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }
    // ...
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")
}
```

Run `flutter clean && flutter pub get` after this change — gradle caches the AAR-metadata check.

Also ensure `AndroidManifest.xml` registers the two FLN receivers and the `RECEIVE_BOOT_COMPLETED` permission so scheduled notifications survive reboots:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>

<application ...>
    <!-- ... -->
    <receiver android:exported="false"
        android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationReceiver" />
    <receiver android:exported="false"
        android:name="com.dexterous.flutterlocalnotifications.ScheduledNotificationBootReceiver">
        <intent-filter>
            <action android:name="android.intent.action.BOOT_COMPLETED" />
            <action android:name="android.intent.action.MY_PACKAGE_REPLACED" />
        </intent-filter>
    </receiver>
</application>
```

### 2. iOS UNUserNotificationCenter delegate

Register the delegate in `ios/Runner/AppDelegate.swift`:

```swift
import Flutter
import UIKit
import UserNotifications

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    if #available(iOS 10.0, *) {
      UNUserNotificationCenter.current().delegate = self as? UNUserNotificationCenterDelegate
    }
    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

**Current state in go_sport:** the delegate is **not** registered. This is known technical debt; if foreground notifications don't appear while the app is in foreground, this is the first place to check.

### 3. Vendor battery optimization

There is no code-level fix. The mitigations are:

- **Surface in-app** — when a user enables their first reminder, show a one-time tip explaining that on some Android devices they may need to add the app to a "protected" or "auto-start" list in system settings. Link to the OEM-specific instructions if practical (Don't Kill My App and similar resources document the per-vendor settings).
- **Don't rely on local scheduling as the sole notification path** for safety-critical features. For features where missed notifications materially hurt the user, FCM (server push) is more reliable on these devices — the system gives push messages preferential treatment over scheduled alarms.

## Where this is likely to occur

- Any project upgrading `flutter_local_notifications` from <17 to 17+ (the desugaring requirement is new)
- Apps integrating local notifications for the first time and testing only on stock Android / Pixel (the vendor-battery issue is invisible there)
- Apps tested only with the app in background during QA (the iOS foreground-delegate issue surfaces only when the app is active at fire time)

## How to detect early

- **Android desugaring**: the build error is loud and explicit; this won't surprise you, but it does surprise developers who skip reading the plugin's setup instructions
- **iOS foreground notifications**: schedule a test notification 30 seconds out, leave the app open in the simulator or device, watch what happens. No banner = delegate not registered.
- **Vendor battery**: harder to catch in QA. Watch crash analytics / user feedback for "I didn't get my reminder" reports clustered on specific device brands.
