# Snapbug in a Flutter app

Requirements: Dart 3.5.4 or later, Android `minSdk` 23 or higher plus the toolchain floor below, iOS 15 or later (Apple-silicon simulators only).

The plugin brings the native SDK with it. **Do not edit `MainActivity` or `AppDelegate`, and do not add `ai.snapbug:*` Gradle dependencies.** On Android, the plugin already wires the debug SDK and the release no-op twin.

## 1. Dependency

```yaml
dependencies:
  snapbug_flutter: ^0.1.1
```

Run `flutter pub get`. It also creates `ios/Podfile` if the project had none yet, so do it before step 2.

In `android/app/build.gradle(.kts)`, make sure the min SDK is at least 23. A literal below 23 becomes `23`. `minSdk = flutter.minSdkVersion` becomes `minSdk = Math.max(23, flutter.minSdkVersion)` in Groovy, or `minSdk = maxOf(23, flutter.minSdkVersion)` in Kotlin DSL, because older Flutter versions default to 21.

## Android toolchain floor

The SDK's Android artifacts need a recent build toolchain. Older project templates (for example Flutter 3.24 and React Native 0.73) ship below it and fail to build until it is raised:

| What | Minimum | Where |
| --- | --- | --- |
| `compileSdk` | 36 | `android/app/build.gradle(.kts)` |
| Android Gradle plugin | 8.6.0 | `android/settings.gradle(.kts)` (`com.android.application`) or `android/build.gradle` |
| Gradle wrapper | 8.7 | `android/gradle/wrapper/gradle-wrapper.properties` |
| Kotlin Gradle plugin | 2.2 | `org.jetbrains.kotlin.android` next to AGP, or the `kotlinVersion` / `kotlin_version` property |

Raise only what is below the minimum, and never lower anything. Typical errors when a floor is missed: `requires … compileSdk 36`, `requires Android Gradle plugin 8.6.0`, `incompatible version of Kotlin`. These upgrades touch the user's toolchain, so list each one explicitly in your final summary.


## 2. iOS release twin

CocoaPods has no automatic debug/release swap. In `ios/Podfile`, set (or uncomment) `platform :ios, '15.0'` at the top. If the Runner target's `IPHONEOS_DEPLOYMENT_TARGET` in `ios/Runner.xcodeproj/project.pbxproj` and `MinimumOSVersion` in `ios/Flutter/AppFrameworkInfo.plist` are below 15.0, raise them to 15.0. Then, inside `target 'Runner' do`, add:

```ruby
pod 'Snapbug',       '~> 0.1.1', :configurations => ['Debug']
pod 'Snapbug-no-op', '~> 0.1.1', :configurations => ['Release']
```

Flutter's `Profile` configuration is built like a release build. Put it in the no-op list by default (`:configurations => ['Release', 'Profile']`). Move it to the live list only if the user says testers get profile builds.

Then run `cd ios && pod install`.

## 3. Workaround for SDK 0.1.1: missing network security config

<!-- Maintainers: delete this section (and its twins in android.md and the other wrapper file) once
     the published SDK is 0.1.2 or newer — the core AAR ships the resource from then on. -->

The published SDK 0.1.1 points the app at `@xml/network_security_config` but does not ship that file. A debug build then fails with `resource xml/network_security_config ... not found`. Unless `android/app/src/main/AndroidManifest.xml` already sets its own `android:networkSecurityConfig`, create `android/app/src/debug/res/xml/network_security_config.xml` with exactly this content:

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- Needed by Snapbug SDK 0.1.1 in debug builds; delete after upgrading Snapbug. -->
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">localhost</domain>
        <domain includeSubdomains="true">127.0.0.1</domain>
    </domain-config>
</network-security-config>
```

## 4. Start the SDK

In `lib/main.dart`, make `main` start Snapbug before `runApp`. Wrap the body in `runZonedGuarded`, so uncaught Dart errors reach the crash reporter and `print()` reaches the Logs tab on iOS:

```dart
import 'dart:async';
import 'package:flutter/widgets.dart';
import 'package:snapbug_flutter/snapbug_flutter.dart';

void main() {
  runZonedGuarded(() async {
    WidgetsFlutterBinding.ensureInitialized();
    await Snapbug.start();
    runApp(const MyApp());
  }, (error, stack) {
    SnapbugCrashReporter.reportError(error, stack);
  }, zoneSpecification: SnapbugLogs.zoneSpecification());
}
```

If `main` already has a `runZonedGuarded` or `ensureInitialized` call, merge into it instead of nesting a second zone. Keep the app's existing error handler, and call `SnapbugCrashReporter.reportError` from it in addition to what it already does. Keep whatever `main` already did before `runApp` (Firebase init and so on), after `ensureInitialized()`.

## 5. Network capture (Dio)

If the app uses Dio, add the interceptor wherever each `Dio` instance is created:

```dart
dio.interceptors.add(SnapbugDioInterceptor());
```

If the client is built with a cascade (`final dio = Dio(...)..interceptors.add(a)`), append `..interceptors.add(SnapbugDioInterceptor())` to it.

Add it after the app's own interceptors (auth and others), so it sees the final request. For `package:http` or other clients there is no automatic hook: skip this step and tell the user that network capture needs Dio.

## 6. Verify

```bash
flutter analyze
flutter build apk --debug
flutter build apk --release
flutter build ios --simulator --debug   # macOS with Xcode only
```

If the iOS build fails with `does not contain arm64 x86_64`, the simulator build is trying an Intel slice that neither Flutter nor Snapbug ship on Apple silicon. Build the arm64 slice directly: `cd ios && xcodebuild -workspace Runner.xcworkspace -scheme Runner -configuration Debug -sdk iphonesimulator ARCHS=arm64 CODE_SIGNING_ALLOWED=NO build`.

The release APK build proves the no-op swap on Android. If release signing is not configured, `flutter build apk --release` still builds with debug keys by default. If it fails on signing anyway, report that and keep the debug result.
