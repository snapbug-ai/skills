# Snapbug in a React Native app

Requirements: React Native 0.73 or later, React 18 or later, Android `minSdk` 24 or higher, iOS 15 or later (Apple-silicon simulators only).

The package brings the native SDK with it. **Do not edit `MainActivity`, `MainApplication` or `AppDelegate`, and do not add `ai.snapbug:*` Gradle dependencies.** On Android, the module already wires the debug SDK and the release no-op twin.

## Expo

Stop and tell the user if `package.json` depends on `expo` and the project has no `android/` or `ios/` folder (managed workflow), or is run in Expo Go. The SDK is a native module: Expo Go cannot load it, and the managed workflow needs `npx expo prebuild` first. Expo support in the published package is not verified yet. Offer to continue on a prebuilt project at the user's own risk, and do not claim it works until a debug build runs.

## 1. Dependency

Install with the project's package manager (look for `yarn.lock`, `pnpm-lock.yaml`, `bun.lockb` or `package-lock.json`):

```bash
npm install snapbug-react-native
```

If `android/build.gradle` sets `minSdkVersion` below 24, raise it to 24 and tell the user.

## 2. iOS release twin

CocoaPods has no automatic debug/release swap. In `ios/Podfile`, inside the app target, add:

```ruby
pod 'Snapbug',       '~> 0.1.1', :configurations => ['Debug']
pod 'Snapbug-no-op', '~> 0.1.1', :configurations => ['Release']
```

Then run `cd ios && pod install`.

## 3. Start the SDK

In the app's JS entry file (`index.js`, or the file that calls `AppRegistry.registerComponent`), before the registration:

```javascript
import { Snapbug, patchFetch } from 'snapbug-react-native';

patchFetch();      // captures every fetch() call
Snapbug.start();
```

`patchFetch()` must run before any module makes a request. Keep it at the top of the entry file, after the imports. Only `fetch` is patched. axios uses `XMLHttpRequest` by default, so unless the app sets `adapter: 'fetch'`, tell the user that axios calls will not appear in the network tab.

In TypeScript entry files the same code applies unchanged.

## 4. Verify

```bash
cd android && ./gradlew assembleDebug assembleRelease && cd ..
cd ios && xcodebuild -workspace *.xcworkspace -scheme <AppScheme> -configuration Debug -destination 'generic/platform=iOS Simulator' build   # macOS with Xcode only
```

Also run the project's typecheck or lint script if one exists (`npm run tsc`, `npm run lint`).
