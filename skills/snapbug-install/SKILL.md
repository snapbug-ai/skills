---
name: snapbug-install
description: Installs the Snapbug debugging and bug-reporting SDK into an Android, iOS, Flutter or React Native app. It adds the dependencies with the release no-op twin, the one-line start call and the network hook, then runs a build to verify. Use when the user asks to add, install, integrate, set up or connect Snapbug in their app.
---

# Install the Snapbug SDK

Snapbug streams an app's network calls, logs, crashes, database and analytics into a Snapbug inspector, and lets testers file bug reports (screenshot or video, then annotations) from a floating bubble. This skill wires the SDK into the user's app so they write no code themselves.

## Rules that are easy to get wrong

1. **Debug only.** The live SDK goes into debug builds. Release builds get the **no-op twin**, which has the same API and does nothing. Never leave the live SDK in a release or App Store build. Never guard the call site with `BuildConfig.DEBUG` either: the swap happens at the dependency level.
2. **One artifact.** On Android, `ai.snapbug:snapbug` is all-in-one. It already includes the OkHttp, Ktor, gRPC and Room integrations. Do not add per-plugin artifacts (`snapbug-okhttp-interceptor`, `snapbug-analytics`, …). In release they land next to their no-op twins and the build fails with `Duplicate class`.
3. **Android start takes an `Activity`.** `Snapbug.start(this)` goes in the launcher `Activity.onCreate()`, never in `Application.onCreate()`, because the overlay needs an Activity. Repeated calls (rotation, re-creation) are safe.
4. **One line.** All plugins are on by default. Do not add plugin configuration the user did not ask for.
5. **Versions.** Use exactly the versions written in the reference files. They are the published ones. Do not invent a newer version.
6. **Match the project's style.** Use the version catalog (`gradle/libs.versions.toml`) if the project has one, Groovy or Kotlin DSL to match the existing `build.gradle(.kts)`, and the project's own import order and formatting.

## Workflow

### 1. Detect the platform

Look at the repository root (and `android/` and `ios/` for cross-platform apps):

| Signal | Platform | Reference |
| --- | --- | --- |
| `pubspec.yaml` depending on `flutter` | Flutter | [references/flutter.md](references/flutter.md) |
| `package.json` depending on `react-native` | React Native | [references/react-native.md](references/react-native.md) |
| `package.json` depending on `expo` | Expo | read the Expo section of [references/react-native.md](references/react-native.md) first |
| `build.gradle(.kts)` applying `com.android.application` | Android (native) | [references/android.md](references/android.md) |
| `*.xcodeproj` / `*.xcworkspace` with a Swift app target | iOS (native) | [references/ios.md](references/ios.md) |
| Kotlin Multiplatform app with Compose Multiplatform on iOS | Android and KMP-iOS | [references/android.md](references/android.md), then the KMP section of [references/ios.md](references/ios.md) |

Flutter and React Native wrap the native SDK. Use only their reference: do not also apply the Android or iOS reference to the same app.

If the repository holds several apps, ask the user which one to set up.

If it is a web app (a `package.json` without `react-native` or `expo`, or a plain website), no SDK is installed. Snapbug inspects web pages through the Snapbug Chrome extension. It is not in the Chrome Web Store yet, so point the user to https://snapbug.ai/docs/chrome-extension, change nothing in the project, and stop.

If nothing matches, say that Snapbug supports Android, iOS, Flutter and React Native apps, and web pages through the Chrome extension, and stop.

### 2. Read the reference and apply it

Read the whole reference file for the platform before editing. Apply the steps in order. For network capture, hook every HTTP client the app actually builds (search for `OkHttpClient.Builder`, `HttpClient(`, `Dio(`, `fetch`). Skip any step whose client the app does not use.

### 3. Verify

Run the build command from the reference: a debug build plus a release build, where it is cheap. The release build proves the no-op swap. Fix any errors you introduced. If the build cannot run in this environment (no SDK, no Xcode), say so plainly rather than claiming success.

### 4. Tell the user how to connect

Summarize the files you changed, then give these steps:

1. Install a Snapbug inspector: the desktop app from https://snapbug.ai/download, or the CLI with `brew install snapbug-ai/tap/snapbug && snapbug serve`.
2. Run the debug build on a device or emulator. A floating Snapbug bubble appears. On Android it asks for the "display over other apps" permission the first time.
3. Open the bubble's menu and connect: enter the 6-digit code shown in the inspector, or scan its QR code.
4. To file a bug report, open the bubble's menu (shaking the device opens it too).

Docs: https://snapbug.ai/docs/getting-started
