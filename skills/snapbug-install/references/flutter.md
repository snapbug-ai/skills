# Snapbug in a Flutter app

Requirements: Flutter 3.3 or later, Dart 3.5.4 or later, Android `minSdk` 23 or higher, iOS 15 or later (Apple-silicon simulators only).

The plugin brings the native SDK with it. **Do not edit `MainActivity` or `AppDelegate`, and do not add `ai.snapbug:*` Gradle dependencies.** On Android, the plugin already wires the debug SDK and the release no-op twin.

## 1. Dependency

```yaml
dependencies:
  snapbug_flutter: ^0.1.1
```

Run `flutter pub get`.

If `android/app/build.gradle(.kts)` sets `minSdk` (or `minSdkVersion`) below 23, raise it to 23 and tell the user. If it uses `flutter.minSdkVersion`, leave it as it is. Set an explicit `23` only if the build fails on the min SDK.

## 2. iOS release twin

CocoaPods has no automatic debug/release swap. In `ios/Podfile`, inside `target 'Runner' do`, add:

```ruby
pod 'Snapbug',       '~> 0.1.1', :configurations => ['Debug']
pod 'Snapbug-no-op', '~> 0.1.1', :configurations => ['Release']
```

Flutter's `Profile` configuration is built like a release build. If the user ships profile builds to testers, add `'Profile'` to the first list. Otherwise add it to the second list.

Then run `cd ios && pod install`.

## 3. Start the SDK

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

## 4. Network capture (Dio)

If the app uses Dio, add the interceptor wherever each `Dio` instance is created:

```dart
dio.interceptors.add(SnapbugDioInterceptor());
```

Add it after the app's own interceptors (auth and others), so it sees the final request. For `package:http` or other clients there is no automatic hook: skip this step and tell the user that network capture needs Dio.

## 5. Verify

```bash
flutter analyze
flutter build apk --debug
flutter build apk --release
flutter build ios --simulator --debug   # macOS with Xcode only
```

The release APK build proves the no-op swap on Android. If release signing is not configured, `flutter build apk --release` still builds with debug keys by default. If it fails on signing anyway, report that and keep the debug result.
