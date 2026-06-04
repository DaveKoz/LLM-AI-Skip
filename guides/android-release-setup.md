---
description: Complete Android release setup for Skip Tools projects — edge-to-edge, Crashlytics, native debug symbols, and Play Console compliance
tags: [android, skip-tools, firebase, crashlytics, google-play, release, edge-to-edge, sdk-35, sdk-36]
---

# Android Release Setup for Skip Tools Projects

One-stop configuration guide for Google Play Store submission compliance on Android 15+ (SDK 35) and Android 16+ (SDK 36). Copy this checklist into every new Skip project.

---

## Quick Checklist

| # | Item | Status |
|---|------|--------|
| 1 | Edge-to-edge display configured | [ ] |
| 2 | Custom theme replaces raw `Theme.AppCompat` | [ ] |
| 3 | No `android:screenOrientation` restrictions | [ ] |
| 4 | No `android:resizeableActivity="false"` | [ ] |
| 5 | Firebase Crashlytics integrated | [ ] |
| 6 | Native debug symbols auto-generated on release build | [ ] |
| 7 | Debug symbols uploaded to Play Console | [ ] |
| 8 | R8/ProGuard keep rules for Crashlytics + Firebase | [ ] |

---

## 1. Edge-to-Edge Display (Android 15+, SDK 35)

### Problem
Apps targeting SDK 35 display **edge-to-edge by default**. System bars are transparent, content draws behind them. Play Console warns if not handled.

### Files to Modify

**`Android/app/src/main/res/values/themes.xml`** (create if missing)

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="Theme.YourApp" parent="Theme.AppCompat.DayNight.NoActionBar">
    </style>
</resources>
```

**`Android/app/src/main/AndroidManifest.xml`**

```xml
<activity
    android:name=".MainActivity"
    android:theme="@style/Theme.YourApp"
    android:configChanges="orientation|screenSize|screenLayout|keyboardHidden|mnc|colorMode|density|fontScale|fontWeightAdjustment|keyboard|layoutDirection|locale|mcc|navigation|smallestScreenSize|touchscreen|uiMode"
    ... >
```

> **IMPORTANT:** Do NOT use `enableEdgeToEdge()`. Even the latest `androidx.activity` (1.13.0) calls the deprecated `setStatusBarColor`/`setNavigationBarColor` and `LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES` internally, which Google Play's deprecated-API report flags (stack frames `androidx.activity.*`). Use `WindowCompat.setDecorFitsSystemWindows(window, false)` instead — it produces the same edge-to-edge result without the flagged calls.

**`Android/app/src/main/kotlin/Main.kt`** — Theme-aware composable using `WindowCompat`:

```kotlin
import androidx.core.view.WindowCompat

@Composable
internal fun SyncSystemBarsWithTheme() {
    val dark = MaterialTheme.colorScheme.background.luminance() < 0.5f
    val activity = LocalContext.current as? ComponentActivity
    DisposableEffect(dark) {
        activity?.window?.let { window ->
            // Edge-to-edge without deprecated setStatusBarColor/setNavigationBarColor.
            WindowCompat.setDecorFitsSystemWindows(window, false)
            val controller = WindowCompat.getInsetsController(window, window.decorView)
            // Light background => dark icons; dark background => light icons
            controller.isAppearanceLightStatusBars = !dark
            controller.isAppearanceLightNavigationBars = !dark
        }
        onDispose { }
    }
}
```

**`Android/app/src/main/kotlin/YourScanActivity.kt`** — For non-Compose activities:

```kotlin
import androidx.core.view.WindowCompat

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    WindowCompat.setDecorFitsSystemWindows(window, false)
    setContentView(R.layout.activity_scanner)
}
```

---

## 2. Orientation & Resizability (Android 16+, SDK 36)

### Problem
Android 16 ignores `screenOrientation` and `resizeableActivity` on large screens (foldables, tablets).

### Fix

Remove ALL `android:screenOrientation="portrait"` from every `<activity>` in `AndroidManifest.xml`.

**Before:**
```xml
<activity
    android:name=".ScannerActivity"
    android:exported="false"
    android:screenOrientation="portrait" />
```

**After:**
```xml
<activity
    android:name=".ScannerActivity"
    android:exported="false"
    android:configChanges="orientation|screenSize|screenLayout|smallestScreenSize" />
```

| Activity Type | Required `configChanges` |
|---------------|--------------------------|
| Main / Compose | `orientation|screenSize|screenLayout|keyboardHidden|mnc|colorMode|density|fontScale|fontWeightAdjustment|keyboard|layoutDirection|locale|mcc|navigation|smallestScreenSize|touchscreen|uiMode` |
| Camera / Scanner | `orientation|screenSize|screenLayout|smallestScreenSize` |
| Transparent trampoline | No `configChanges` needed |

---

## 3. Firebase Crashlytics Integration

### 3A. `Package.swift` — Add Dependency

```swift
.target(name: "YourApp", dependencies: [
    "YourModule",
    .product(name: "SkipFirebaseMessaging", package: "skip-firebase"),
    .product(name: "SkipFirebaseAnalytics", package: "skip-firebase"),
    .product(name: "SkipFirebaseCrashlytics", package: "skip-firebase"),
    // ... other deps
], plugins: [.plugin(name: "skipstone", package: "skip")]),
```

### 3B. Swift App Startup — Initialize Crashlytics

```swift
// YourApp.swift or App delegate
#if os(Android)
import SkipFirebaseCore
import SkipFirebaseMessaging
import SkipFirebaseCrashlytics
#else
import FirebaseCore
import FirebaseMessaging
import FirebaseCrashlytics
#endif

public func onInit() {
    FirebaseApp.configure()
    Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)
    // ... rest of init
}
```

### 3C. `Android/app/build.gradle.kts` — Add Plugin + Config

```kotlin
plugins {
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.android.application)
    id("skip-build-plugin")
    id("com.google.gms.google-services") version "4.4.2" apply true
    id("com.google.firebase.crashlytics") version "3.0.2" apply true
}

// Configure Crashlytics native symbol upload (reflection because
// CrashlyticsExtension type isn't on the Kotlin DSL classpath at parse time)
plugins.withId("com.google.firebase.crashlytics") {
    afterEvaluate {
        val ext = extensions.findByName("crashlytics") ?: return@afterEvaluate
        ext.javaClass.getMethod("setNativeSymbolUploadEnabled", Boolean::class.javaPrimitiveType)
            .invoke(ext, true)
    }
}
```

### 3D. `Android/app/build.gradle.kts` — Firebase Dependencies

```kotlin
dependencies {
    implementation(platform("com.google.firebase:firebase-bom:33.12.0"))
    implementation("com.google.firebase:firebase-crashlytics-ndk")
    // ... other deps
}
```

### 3E. `Android/app/proguard-rules.pro` — Keep Rules

```proguard
# Crashlytics — keep line numbers and exception types for readable traces
-keepattributes SourceFile,LineNumberTable
-keep public class * extends java.lang.Exception

# Keep Crashlytics + Google Play Services classes from R8 stripping
-keep class com.google.firebase.crashlytics.** { *; }
-keep class com.google.android.gms.common.** { *; }
-keep class com.google.android.gms.tasks.** { *; }
-dontwarn com.google.android.gms.common.GoogleApiAvailability

# Firebase Firestore (existing — required for Skip JNI bridge)
-keep class com.google.firebase.firestore.FieldValue {
    public static com.google.firebase.firestore.FieldValue serverTimestamp();
    public static com.google.firebase.firestore.FieldValue delete();
    public static com.google.firebase.firestore.FieldValue arrayUnion(...);
    public static com.google.firebase.firestore.FieldValue arrayRemove(...);
    public static com.google.firebase.firestore.FieldValue increment(...);
    *;
}
-keep class com.google.firebase.firestore.** { *; }
-keep class com.google.firebase.Timestamp { *; }
```

---

## 4. Native Debug Symbols (Play Console Requirement)

### Problem
Google Play Console warns: *"This App Bundle contains native code, and you've not uploaded debug symbols."*

### Fix

#### Step 1 — Preserve unstripped `.so` files in `build.gradle.kts`

```kotlin
android {
    packaging {
        jniLibs {
            keepDebugSymbols.add("**/*.so")
            useLegacyPackaging = true
        }
    }
}
```

#### Step 2 — Add the symbol packaging task

AGP's `ndk.debugSymbolLevel` only works for NDK-compiled C/C++ code. Skip's Swift `.so` files are compiled externally by the Swift compiler, so we must package them manually.

Add to `app/build.gradle.kts` **outside** the `android { }` block:

```kotlin
// Package unstripped native debug symbols for Google Play Console.
// Skip's Swift .so files are compiled externally, so AGP can't extract symbols automatically.
// Output: app/build/outputs/native-debug-symbols/release/native-debug-symbols.zip
tasks.register<Zip>("packageNativeDebugSymbols") {
    description = "Packages unstripped .so files into a zip for Google Play Console"
    group = "build"

    // Run after all native libs are merged together
    dependsOn("mergeReleaseNativeLibs")

    val validAbis = listOf("arm64-v8a", "armeabi-v7a", "x86", "x86_64")
    val libBaseDir = layout.buildDirectory
        .dir("intermediates/merged_native_libs/release/mergeReleaseNativeLibs/out/lib")
        .get().asFile

    var foundAny = false
    for (abi in validAbis) {
        val abiDir = libBaseDir.resolve(abi)
        if (abiDir.isDirectory) {
            val soFiles = abiDir.listFiles { f -> f.isFile && f.extension == "so" }
            if (!soFiles.isNullOrEmpty()) {
                foundAny = true
                from(abiDir) {
                    into(abi)
                    include("*.so")
                }
            }
        }
    }

    onlyIf { foundAny }
    duplicatesStrategy = DuplicatesStrategy.INCLUDE
    archiveFileName.set("native-debug-symbols.zip")
    destinationDirectory.set(layout.buildDirectory.dir("outputs/native-debug-symbols/release"))
}

// Auto-run after any release build path
// mergeReleaseNativeLibs = triggered by skip export
// assembleRelease = triggered by Xcode builds
// bundleRelease = triggered by direct Gradle builds
afterEvaluate {
    tasks.findByName("mergeReleaseNativeLibs")?.let { mergeTask ->
        mergeTask.finalizedBy("packageNativeDebugSymbols")
    }
    tasks.findByName("assembleRelease")?.let { assembleTask ->
        assembleTask.finalizedBy("packageNativeDebugSymbols")
    }
    tasks.findByName("bundleRelease")?.let { bundleTask ->
        bundleTask.finalizedBy("packageNativeDebugSymbols")
    }
}
```

**Output path** (auto-generated on every release build):
```
app/build/outputs/native-debug-symbols/release/native-debug-symbols.zip
```

#### Step 3 — Upload to Play Console

After every release build from Xcode or Gradle, find:

```
Android/app/build/outputs/native-debug-symbols/release/native-debug-symbols.zip
```

Upload to:
1. [Google Play Console](https://play.google.com/console) → Your app → Release → App bundles
2. Find your release → **App bundle explorer**
3. **Downloads** tab → **Native debug symbols** → Upload zip

---

## 5. Pre-Build Verification

Run these from your project root before every Play Store upload:

```bash
# Check orientation restrictions
grep -r "screenOrientation" Android/app/src/main/AndroidManifest.xml
# Expected: no output

# Check theme exists
test -f Android/app/src/main/res/values/themes.xml && echo "Theme OK" || echo "Theme MISSING"

# Check custom debug symbol packaging task exists
grep -q "packageNativeDebugSymbols" Android/app/build.gradle.kts && echo "Symbols task OK" || echo "Symbols task MISSING"

# Check Crashlytics plugin
grep -q "firebase.crashlytics" Android/app/build.gradle.kts && echo "Crashlytics plugin OK" || echo "Crashlytics plugin MISSING"

# Check ProGuard keep rules
grep -q "crashlytics" Android/app/proguard-rules.pro && echo "ProGuard rules OK" || echo "ProGuard rules MISSING"
```

---

## 6. Troubleshooting

### `ANDROID_HOME` not set when running Gradle directly

```bash
export ANDROID_HOME=$HOME/Library/Android/sdk
```

### `gradlew` not found in Skip projects

Skip projects don't include a Gradle wrapper. Use system `gradle`:

```bash
gradle bundleRelease
# or
gradle assembleRelease
```

### `SDK location not found` error

Either export `ANDROID_HOME` or create `local.properties`:

```bash
echo "sdk.dir=$HOME/Library/Android/sdk" > Android/local.properties
```

### Play Console still shows "debug symbols not uploaded"

The zip is only generated on **release** builds. Debug builds don't produce it. Make sure you're building Release scheme in Xcode.

---

## 7. Files Changed Summary

| File | What Changed |
|------|-------------|
| `Package.swift` | Added `SkipFirebaseCrashlytics` dependency |
| `Sources/*/App.swift` | Added `Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)` in `onInit()` |
| `Android/app/build.gradle.kts` | Added Crashlytics plugin, Firebase BOM, `packageNativeDebugSymbols` task, `crashlytics { nativeSymbolUploadEnabled }` config |
| `Android/app/proguard-rules.pro` | Added Crashlytics + GMS keep rules |
| `Android/app/src/main/res/values/themes.xml` | Created custom `Theme.YourApp` |
| `Android/app/src/main/AndroidManifest.xml` | Removed `screenOrientation`, applied custom theme, added `configChanges` |
| `Android/app/src/main/kotlin/Main.kt` | `SyncSystemBarsWithTheme()` uses `WindowCompat.setDecorFitsSystemWindows` (not `enableEdgeToEdge()`) |
| `Android/app/src/main/kotlin/YourScanActivity.kt` | Uses `WindowCompat.setDecorFitsSystemWindows` for edge-to-edge |

---

## Change Log

| Date | Change | Notes |
|------|--------|-------|
| 2026-06-03 | Initial setup | Edge-to-edge, orientation, Crashlytics, native symbols |
