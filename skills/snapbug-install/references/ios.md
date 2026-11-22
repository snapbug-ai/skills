# Snapbug on iOS (native Swift app)

Requirements: iOS 15 or later. Simulators must run on Apple silicon (the framework has no `x86_64` simulator slice).

Choose the install path in this order:

1. The app already uses **CocoaPods** (has a `Podfile`): use CocoaPods. This is the only path with a release no-op twin.
2. The app uses **Compose Multiplatform** for its iOS UI: use the Kotlin Multiplatform path at the end of this file. The XCFramework embeds its own Compose runtime and would clash with the app's (duplicate symbols).
3. Otherwise, use **Swift Package Manager**.

## CocoaPods

In the `Podfile`, inside the app target, add both lines:

```ruby
pod 'Snapbug',       '~> 0.1.1', :configurations => ['Debug']
pod 'Snapbug-no-op', '~> 0.1.1', :configurations => ['Release']
```

Both lines are required. With only the first one, the Release configuration has no `Snapbug` module and fails to compile. If the project has extra configurations (for example `Staging`), put a configuration in the live list only if internal testers install builds made from it. Anything that can reach the App Store goes in the no-op list. If you are unsure, ask the user.

Keep the target's existing `use_frameworks!` setting. Then run `pod install` (it needs network access to CocoaPods trunk) and keep using the `.xcworkspace`.

Both pods provide a module named `Snapbug`, and CocoaPods builds both in every configuration. Warnings like `Multiple targets match implicit dependency for linker flags '-framework Snapbug'` and `'Snapbug-no-op' is missing a dependency on 'Snapbug'` are therefore expected and harmless. Do not try to fix them.

## Swift Package Manager

Add the package `https://github.com/snapbug-ai/snapbug-ios` and its `Snapbug` library product to the app target. With XcodeGen or Tuist, edit their manifest instead of `project.pbxproj`. Editing `project.pbxproj` by hand is a last resort: if you cannot add the package safely, tell the user to do it from Xcode (**File → Add Package Dependencies…**) and continue with the code steps.

SPM has no per-configuration dependencies, so guard the call site yourself: wrap `import Snapbug` and `Snapbug.start()` in `#if DEBUG … #endif`.

## Start the SDK

The published SDK 0.1.1 attaches the floating bubble only if a window scene already exists when `Snapbug.start()` runs. Started from `App.init()` or `didFinishLaunching`, the SDK runs, but the bubble never appears and the user cannot connect. Start once the first screen is on screen:

<!-- Maintainers: once the published SDK is 0.1.2 or newer (it waits for the scene itself),
     this can go back to a plain Snapbug.start() in App.init() / didFinishLaunching. -->

SwiftUI: from the root view's `.onAppear`, with a run-once guard. This is verified on the simulator with 0.1.1.

```swift
import Snapbug
import SwiftUI

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onAppear { SnapbugBootstrap.startOnce() }
        }
    }
}

@MainActor
enum SnapbugBootstrap {
    private static var started = false
    static func startOnce() {
        guard !started else { return }
        started = true
        Snapbug.start()
    }
}
```

If the root view already has an `.onAppear`, add the call inside it rather than chaining a second one.

UIKit with a `SceneDelegate`: at the end of `scene(_:willConnectTo:options:)`, after the window is made key and visible, using the same run-once guard. UIKit without scenes (only an `AppDelegate` with a `window`): at the end of `application(_:didFinishLaunchingWithOptions:)`, after `window?.makeKeyAndVisible()`, before `return true`.

With CocoaPods, do not wrap the call in `#if DEBUG`: the Release build links the no-op pod, and the call does nothing there.

## Info.plist keys (required)

Add these two keys to the app's Info.plist, unless they are already present. Where the keys go:

- If the target has `GENERATE_INFOPLIST_FILE = YES` and no Info.plist file of its own, add them as build settings `INFOPLIST_KEY_NSMicrophoneUsageDescription` and `INFOPLIST_KEY_NSCameraUsageDescription`.
- With XcodeGen or Tuist, add them to the manifest (XcodeGen: `info.properties` in `project.yml`). Otherwise the next generate erases them.

 iOS kills the app outright when it touches the camera or the microphone without a usage string, and that looks like a crash in the SDK.

```xml
<key>NSMicrophoneUsageDescription</key>
<string>The microphone is used to narrate bug report recordings.</string>
<key>NSCameraUsageDescription</key>
<string>The camera is used to scan the Snapbug pairing QR code.</string>
```

If the app already has one of these keys, keep its existing text.

## Network capture

Swift `URLSession` traffic is not captured automatically. Only a Kotlin Multiplatform Ktor client can be hooked: `install(SnapbugKtorPlugin)`, with `import io.snapbug.sdk.ktor.SnapbugKtorPlugin`. For a pure Swift app, skip network capture, and say in your final summary that network requests will not appear in the inspector.

## Verify

```bash
xcodebuild -workspace App.xcworkspace -scheme App -configuration Debug -sdk iphonesimulator ARCHS=arm64 CODE_SIGNING_ALLOWED=NO build
xcodebuild -workspace App.xcworkspace -scheme App -configuration Release -sdk iphonesimulator ARCHS=arm64 CODE_SIGNING_ALLOWED=NO build
```

`ARCHS=arm64` is needed because the framework has no Intel simulator slice.

A green Release build does not prove the swap on its own, because both pods are compiled. Also check the built Release app: `Snapbug.framework` inside `<App>.app/Frameworks/` should be about 7 MB (no-op). About 31 MB means the live SDK leaked into Release.

Use `-project App.xcodeproj` instead of `-workspace` for SPM projects without a workspace. Use the app's real scheme name from `xcodebuild -list`, not the `Pods-…` one. Without Xcode in this environment, say that the build was not verified.

## Kotlin Multiplatform path (Compose Multiplatform apps)

Add the Kotlin artifact to `iosMain` of the shared module that builds the iOS framework:

```kotlin
kotlin {
    sourceSets {
        iosMain.dependencies {
            implementation("ai.snapbug:snapbug:0.1.1")
        }
    }
}
```

Kotlin/Native has no debug/release dependency split. Ask the user how they tell internal builds from App Store builds (often a Gradle property passed by the Xcode build phase). Use `ai.snapbug:snapbug-no-op:0.1.1` for App Store builds, chosen by that switch.

Start once, from the function that creates the root `UIViewController` (for example `MainViewController()`):

```kotlin
import io.snapbug.sdk.SnapbugContext
import io.snapbug.sdk.startSnapbug
import io.snapbug.sdk.crashreporter.SnapbugCrashReporter
import io.snapbug.sdk.debugfeedback.DebugFeedbackConfig
import io.snapbug.sdk.debugfeedback.SnapbugDebugFeedbackIos
import io.snapbug.sdk.device.SnapbugDevice
import io.snapbug.sdk.plugins.analytics.SnapbugAnalytics
import io.snapbug.sdk.plugins.debugfeedback.SnapbugDebugFeedback

private var snapbugStarted = false

private fun startSnapbugOnce() {
    if (snapbugStarted) return
    snapbugStarted = true
    startSnapbug(SnapbugContext()) {
        install(SnapbugDebugFeedback)
        install(SnapbugAnalytics)
        install(SnapbugCrashReporter) { catchFatalErrors = true }
        install(SnapbugDevice)
    }
    SnapbugDebugFeedbackIos.start(DebugFeedbackConfig())
}
```

The Info.plist keys above apply here too.
