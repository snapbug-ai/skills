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

Both lines are required. With only the first one, the Release configuration has no `Snapbug` module and fails to compile. If the project has extra configurations (for example `Staging`), add each one to the matching list.

Then run `pod install` and keep using the `.xcworkspace`.

## Swift Package Manager

Add the package `https://github.com/snapbug-ai/snapbug-ios` and its `Snapbug` library product to the app target. With XcodeGen or Tuist, edit their manifest instead of `project.pbxproj`. Editing `project.pbxproj` by hand is a last resort: if you cannot add the package safely, tell the user to do it from Xcode (**File → Add Package Dependencies…**) and continue with the code steps.

SPM has no per-configuration dependencies, so guard the call site yourself: wrap `import Snapbug` and `Snapbug.start()` in `#if DEBUG … #endif`.

## Start the SDK

SwiftUI: in the `@main` App struct's `init()`.

```swift
import Snapbug

@main
struct MyApp: App {
    init() {
        Snapbug.start()
    }

    var body: some Scene {
        WindowGroup { ContentView() }
    }
}
```

UIKit: in `application(_:didFinishLaunchingWithOptions:)` of the `AppDelegate`, before `return true`.

```swift
import Snapbug

func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    Snapbug.start()
    return true
}
```

With CocoaPods, do not wrap the call in `#if DEBUG`: the Release build links the no-op pod, and the call does nothing there.

## Info.plist keys (required)

Add these two keys to the app's Info.plist, unless they are already present. iOS kills the app outright when it touches the camera or the microphone without a usage string, and that looks like a crash in the SDK.

```xml
<key>NSMicrophoneUsageDescription</key>
<string>The microphone is used to narrate bug report recordings.</string>
<key>NSCameraUsageDescription</key>
<string>The camera is used to scan the Snapbug pairing QR code.</string>
```

If the app already has one of these keys, keep its existing text.

## Network capture

Swift `URLSession` traffic is not captured automatically. Only a Kotlin Multiplatform Ktor client can be hooked: `install(SnapbugKtorPlugin)`, with `import io.snapbug.sdk.ktor.SnapbugKtorPlugin`. For a pure Swift app, skip network capture and tell the user so.

## Verify

```bash
xcodebuild -workspace App.xcworkspace -scheme App -configuration Debug -destination 'generic/platform=iOS Simulator' build
xcodebuild -workspace App.xcworkspace -scheme App -configuration Release -destination 'generic/platform=iOS Simulator' build
```

Use `-project App.xcodeproj` instead of `-workspace` for SPM projects without a workspace. Use the real scheme name (`xcodebuild -list`). Without Xcode in this environment, say that the build was not verified.

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
