# Snapbug on Android (native app)

Requirements: `minSdk` 23 or higher, a recent Kotlin Gradle plugin (2.3 or later is known to work), and an app module applying `com.android.application`.

Expect the debug APK to grow by tens of megabytes (the inspector transport and the overlay UI). The release APK is unaffected. Mention this to the user.

## 1. Dependencies

Edit the **app module** (the one applying `com.android.application`), not the root build file.

If the project has `gradle/libs.versions.toml`, merge these entries into its existing `[versions]` and `[libraries]` tables. Do not add a second table header: that is invalid TOML.

```toml
[versions]
snapbug = "0.1.1"

[libraries]
snapbug = { module = "ai.snapbug:snapbug", version.ref = "snapbug" }
snapbug-no-op = { module = "ai.snapbug:snapbug-no-op", version.ref = "snapbug" }
```

```kotlin
dependencies {
    debugImplementation(libs.snapbug)
    releaseImplementation(libs.snapbug.no.op)
}
```

Gradle turns `-` into `.` in catalog accessors, so `snapbug-no-op` becomes `libs.snapbug.no.op`, not `libs.snapbug.noOp`.

Without a catalog, Kotlin DSL (`build.gradle.kts`):

```kotlin
dependencies {
    debugImplementation("ai.snapbug:snapbug:0.1.1")
    releaseImplementation("ai.snapbug:snapbug-no-op:0.1.1")
}
```

Groovy (`build.gradle`):

```groovy
dependencies {
    debugImplementation 'ai.snapbug:snapbug:0.1.1'
    releaseImplementation 'ai.snapbug:snapbug-no-op:0.1.1'
}
```

Maven Central also lists a `1.0.0`. It is a stray upload, not a newer release. Never use it, and do not "upgrade" to it.

The artifacts come from Maven Central. Make sure `mavenCentral()` is present in `dependencyResolutionManagement.repositories` (in `settings.gradle(.kts)`) or in the module's `repositories`.

If the app has build types other than `debug` and `release` (for example `staging`, `qa`), add the matching line for each one: `stagingImplementation(...)` with the live artifact for internal builds that testers use, and the no-op one for builds that go to end users. If you are unsure which is which, ask the user.

## 2. Start the SDK

Find the launcher Activity: the one with the `android.intent.action.MAIN` + `android.intent.category.LAUNCHER` intent filter in `AndroidManifest.xml`. In its `onCreate`, right after `super.onCreate(savedInstanceState)`:

```kotlin
import io.snapbug.sdk.Snapbug
import io.snapbug.sdk.start

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Snapbug.start(this)
        // ... existing code
    }
}
```

`import io.snapbug.sdk.start` is the extension function that provides `Snapbug.start`. It is required, even though some IDEs show it as unused.

In a Java Activity, call `SnapbugStartKt.start(Snapbug.Companion, this, config -> kotlin.Unit.INSTANCE);`. If the project has any Kotlin at all, a one-line Kotlin helper is cleaner. If the project is Java-only, use the Java form.

If the app has a splash or router Activity that finishes immediately, put the call in the first Activity that stays on screen, and tell the user which one you chose.

This one line enables every plugin (network, logs, crashes, database, analytics, device info, deep links, files, preferences) and the floating bubble. Leave the default configuration alone unless the user asked for something.

## 3. Network capture

The SDK cannot see HTTP traffic on its own: hook each client the app builds. The classes are already in the all-in-one artifact, so do **not** add more dependencies.

OkHttp (also covers Retrofit, Coil and Glide when they use the app's `OkHttpClient`):

```kotlin
import io.snapbug.sdk.okhttp.SnapbugOkhttpInterceptor

val client = OkHttpClient.Builder()
    .addInterceptor(authInterceptor)              // existing interceptors stay first
    .addInterceptor(SnapbugOkhttpInterceptor())   // Snapbug goes after them
    // ... rest of the existing configuration
    .build()
```

Add it as the **last** `addInterceptor` call, so it sees the final request headers (auth and others). Use `addInterceptor`, not `addNetworkInterceptor`. In a DI module (Hilt, Koin, Dagger) that provides the `OkHttpClient`, edit that provider.

Ktor client:

```kotlin
import io.snapbug.sdk.ktor.SnapbugKtorPlugin

val client = HttpClient(OkHttp) {
    install(SnapbugKtorPlugin)
}
```

## 4. Room database (only if the app uses Room)

Database files are discovered automatically. Registering the Room instance makes queries go through Room's own connection, which avoids WAL locking issues. Where the database is built:

```kotlin
import io.snapbug.sdk.database.room.snapbugRegisterDatabase

val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
snapbugRegisterDatabase("app.db", db)
```

For Room 3 (`androidx.room3`), import from `io.snapbug.sdk.database.room3` instead.

## 5. Workaround for SDK 0.1.1: missing network security config

<!-- Maintainers: delete this section (and its twins in flutter.md and react-native.md) once
     the published SDK is 0.1.2 or newer — the core AAR ships the resource from then on. -->

The published SDK 0.1.1 points the app at `@xml/network_security_config` but does not ship that file. A debug build then fails with `resource xml/network_security_config ... not found`.

- If the app's `AndroidManifest.xml` already sets its own `android:networkSecurityConfig`, skip this section. The app's value wins.
- Otherwise, create `app/src/debug/res/xml/network_security_config.xml` (in the debug source set only, so release is untouched) with exactly this content:

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

This only allows plain-text traffic to the local inspector. Everything else keeps the platform defaults.

## 6. Verify

```bash
./gradlew :app:assembleDebug :app:assembleRelease
```

Replace `:app` with the real module path. Both must succeed. A `Duplicate class` error in release means a per-plugin Snapbug artifact was added: remove it and keep only the `snapbug` / `snapbug-no-op` pair. `incompatible Kotlin metadata version` (or a similar Kotlin version error) means the app's Kotlin is older than the SDK needs. Tell the user to upgrade the Kotlin Gradle plugin (2.3 or later is known to work) instead of downgrading Snapbug.

If release fails only on signing, `:app:compileReleaseKotlin` is enough to prove the no-op swap compiles.

## Not covered here

- **Compose overlay instead of the system bubble.** `Snapbug.start(this) { overlay = OverlayMode.FAB }` needs `Snapbug.Overlay()` in the Compose tree. Only do this if the user asks.
- **Release builds that should still send bug reports** to a cloud inbox (`snapbug-lite`). This needs an ingest key from the user. See https://snapbug.ai/docs/getting-started.
