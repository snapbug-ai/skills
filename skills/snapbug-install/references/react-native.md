# Snapbug in a React Native app

Requirements: React Native 0.73 or later, React 18 or later, Android `minSdk` 24 or higher plus the toolchain floor below, iOS 15 or later (Apple-silicon simulators only).

The package brings the native SDK with it. **Do not edit `MainActivity`, `MainApplication` or `AppDelegate`, and do not add `ai.snapbug:*` Gradle dependencies.** On Android, the module already wires the debug SDK and the release no-op twin.

## Expo

Stop and tell the user if `package.json` depends on `expo` and the project has no `android/` or `ios/` folder (managed workflow), or is run in Expo Go. The SDK is a native module: Expo Go cannot load it, and the managed workflow needs `npx expo prebuild` first. Expo support in the published package is not verified yet. Offer to continue on a prebuilt project at the user's own risk, and do not claim it works until a debug build runs.

## 1. Dependency

Install with the project's package manager (look for `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb` or `package-lock.json`):

```bash
npm install snapbug-react-native
```

The package has its own version number, separate from the native SDK it brings, so a package version that differs from the pod versions below is expected. Let the package manager choose the version.

If `android/build.gradle` sets `minSdkVersion` below 24, raise it to 24 and tell the user.

## Android toolchain floor

The SDK's Android artifacts need a recent build toolchain. Older project templates (for example Flutter 3.24 and React Native 0.73) ship below it and fail to build until it is raised:

| What | Minimum | Where |
| --- | --- | --- |
| `compileSdk` | 36 | `android/app/build.gradle(.kts)` |
| Android Gradle plugin | 8.6.0 | `android/settings.gradle(.kts)` (`com.android.application`) or `android/build.gradle` |
| Gradle wrapper | 8.7 | `android/gradle/wrapper/gradle-wrapper.properties` |
| Kotlin Gradle plugin | 2.2 | `org.jetbrains.kotlin.android` next to AGP, or the `kotlinVersion` / `kotlin_version` property |

Raise only what is below the minimum, and never lower anything. Typical errors when a floor is missed: `requires … compileSdk 36`, `requires Android Gradle plugin 8.6.0`, `incompatible version of Kotlin`. These upgrades touch the user's toolchain, so list each one explicitly in your final summary.

## Workaround for SDK 0.1.1: missing network security config

<!-- Maintainers: delete this section (and its twins in android.md and the other wrapper file) once
     the published SDK is 0.1.2 or newer — the core AAR ships the resource from then on. -->

Every `snapbug-react-native` release so far brings SDK 0.1.1. That SDK points the app at `@xml/network_security_config` but does not ship the file. A debug build then fails with `resource xml/network_security_config ... not found`. Unless `android/app/src/main/AndroidManifest.xml` already sets its own `android:networkSecurityConfig`, create `android/app/src/debug/res/xml/network_security_config.xml` with exactly this content:

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


## 2. iOS release twin

CocoaPods has no automatic debug/release swap. In `ios/Podfile`, make sure the platform is at least iOS 15. Templates from React Native 0.76 on use `platform :ios, min_ios_version_supported`, which is already 15.1, so leave that line alone. Older templates need `platform :ios, '15.0'`. Then, inside the app target (for example right after `config = use_native_modules!`), add:

```ruby
pod 'Snapbug',       '~> 0.1.1', :configurations => ['Debug']
pod 'Snapbug-no-op', '~> 0.1.1', :configurations => ['Release']
```

Then run `cd ios && bundle exec pod install` (or plain `pod install` if the project has no `Gemfile`). The first run also creates or updates Xcode project files. That is normal, and the user should commit them.

Both pods provide a module named `Snapbug`, so the warning `'Snapbug' is missing a dependency on 'Snapbug-no-op'` is expected and harmless.

## 3. Start the SDK

In the app's JS entry file (`index.js` or `index.ts`, the file named by `main` in `package.json`, which calls `AppRegistry.registerComponent`), before the registration:

```javascript
import { Snapbug, patchFetch } from 'snapbug-react-native';

patchFetch();      // captures every fetch() call
Snapbug.start();
```

`patchFetch()` must run before any module makes a request. Keep it at the top of the entry file, after the imports. Only `fetch` is patched. axios uses `XMLHttpRequest` by default, so unless the app sets `adapter: 'fetch'`, tell the user that axios calls will not appear in the network tab. If the app does not use axios, say nothing about it.

In TypeScript entry files the same code applies unchanged.

## 4. Verify

```bash
cd android && ./gradlew assembleDebug assembleRelease && cd ..
cd ios && xcodebuild -workspace <App>.xcworkspace -scheme <App> -configuration Debug -sdk iphonesimulator ARCHS=arm64 CODE_SIGNING_ALLOWED=NO build   # macOS with Xcode only
```

Use the app's real workspace and scheme names (`xcodebuild -list`, not the `Pods-…` scheme).

Then run `npm run lint` if that script exists. In a TypeScript project, also run `npx tsc --noEmit`.
