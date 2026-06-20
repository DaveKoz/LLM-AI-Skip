---
description: Skip cross-platform architecture patterns — documents how to integrate native platform libraries and APIs in Skip Fuse projects
---

## Skip Cross-Platform Architecture

This document captures architectural patterns for integrating platform-specific libraries and APIs in Skip Fuse (and Lite) projects. Each entry describes: the problem, which compilation mode applies, the dependency setup, and working code.

> **Note:** This file is portable — copy it between Skip projects. Update it whenever a new cross-platform dependency pattern is discovered.

---

### Key Concepts — Compilation Modes

| Directive | Behavior in Skip Fuse | Behavior in Skip Lite |
|-----------|----------------------|----------------------|
| `#if os(Android)` | Compiled as **native Swift** by Swift Android compiler. Cannot call Kotlin/Java APIs. | Transpiled to Kotlin (same as `#if SKIP`). |
| `#if SKIP` | Always **transpiled to Kotlin**. Can call any Kotlin/Java API. | Same — transpiled to Kotlin. |
| `#if os(iOS)` | Compiled as native Swift on iOS only. | Excluded from transpilation. |

**Rule:** Use `#if SKIP` when you need to call Kotlin/Java APIs. Use `#if os(Android)` only for platform-specific Swift that does not need JVM interop.

---

### ⚠️ CRITICAL: No Automatic Builds

**Do NOT run `swift build`, `skip build`, `gradle build`, or any other automatic build commands.** The user builds manually through Xcode. Build verification is done by the user, not by the agent. Perform manual code review and syntax checking instead.

### ⚠️ CRITICAL: Pre-Build Checklist

Before building or committing Swift code, verify these five rules:

| Rule | Check | Files to Review |
|------|-------|-----------------|
| **@State Visibility** | NO `@State private` — use `@State var` | ALL View `.swift` files |
| **@FocusState Ban** | NO `@FocusState` or `.focused()` — causes Android crash | ALL View `.swift` files |
| **Hashable for ForEach** | `id: \.self` requires `Hashable` conformance | Data models in ForEach |
| **NSNull Ban** | NO `NSNull()` — use `FieldValue.delete()` | Firestore update/set calls |
| **Dictionary Safety** | Mark `[String: Any]` as `nonisolated(unsafe)` | Firestore data operations |
| **Firestore Ref Region** | `DocumentReference` IS `Sendable` — plain `let ref = ...document(id)` is fine; do NOT add `nonisolated(unsafe)` (triggers "unnecessary" warning). `Query` is NOT `Sendable` — use `nonisolated(unsafe) let q = ...whereField(...)` before the `await`. Alternatives: `private nonisolated var db { Firestore.firestore() }`, fresh local `Firestore.firestore()` in the async scope, or rebuild from `doc.documentID` (not `doc.reference`). | All `@MainActor` ViewModels calling async Firebase |
| **Import SkipFuseUI** | NO `import SwiftUI` — use `import SkipFuseUI` | ALL View `.swift` files |
| **@Observable Refresh** | Use `refreshID` + `.id()` + `.onChange()` for UI updates | Views with `@Observable` view models |
| **Toggle in Forms Ban** | NO `Toggle` in forms with buttons — blocks all gestures below it on Android | Login/form views |
| **Shadow Ban on Cards** | NO `.shadow()` on interactive card containers — blocks button taps on Android | Card/form container views |
| **Button Tap Area** | NO styling a title-initializer `Button("..") { }` with external `.frame/.padding/.background` — tap area collapses to text bounds on Android. Use `Button { } label: { Text(..).frame(maxWidth:.infinity).padding().background()... }` | ALL full-width buttons (login, forms, CTAs) |
| **@MainActor on ViewModels** | All `@Observable` ViewModels MUST have `@MainActor` | ALL ViewModel `.swift` files |
| **NO deinit in ViewModels** | Use `stopListening()` + `.onDisappear` — deinit is nonisolated | ALL ViewModel `.swift` files |
| **Firestore async only** | `try await .getDocuments()` ONLY — closure form hits wrong overload | ALL ViewModel `.swift` files |
| **DispatchGroup Ban** | NO `DispatchGroup` — use sequential `async/await` loop | ALL ViewModel `.swift` files |
| **swipeActions Ban** | NO `.swipeActions` — use `.contextMenu` | ALL List View `.swift` files |
| **pickerStyle wheel Ban** | NO `.pickerStyle(.wheel)` — use `.pickerStyle(.menu)` | ALL Picker views |
| **insetGrouped Ban** | NO `.listStyle(.insetGrouped)` — use `.listStyle(.plain)` | ALL List View `.swift` files |
| **Layout Protocol Ban** | NO custom `Layout` conforming structs — `ProposedViewSize`, `Subviews`, `sizeThatFits/placeSubviews` not bridged | ALL View `.swift` files |
| **navigationBarDrawer Ban** | NO `.searchable(placement: .navigationBarDrawer(...))` — omit `placement:` | ALL searchable views |
| **LocationProvider Permission** | `PermissionManager.requestLocationPermission()` FIRST, then `fetchCurrentLocation()` | Any view using location |
| **Firestore Field Names** | Read field names MUST match write path — verify all reads use same key as writes | All Firestore read/write code |
| **Content URI Reading** | `Data(contentsOf:)` crashes on Android `content://` URIs — branch to `putFileAsync(from:)` or use JNI `ContentResolver` | Image upload / file handling |
| **Google Play Media Permissions Ban** | NO `READ_MEDIA_IMAGES` or `READ_MEDIA_VIDEO` in `AndroidManifest.xml` — Google Play rejects apps that declare these for one-time/profile photo use. Use Android photo picker (no permission needed). Scope `READ_EXTERNAL_STORAGE` with `maxSdkVersion="32"`, `WRITE_EXTERNAL_STORAGE` with `maxSdkVersion="28"`. | `Android/app/src/main/AndroidManifest.xml` |
| **SkipSQL API (0.16.0)** | Use `context.prepare(sql:)`, `stmt.bind(_:at:)`, `stmt.next()`, `stmt.text/real/long(at:)`, `stmt.close()`, `context.exec(sql:parameters:)` | DAO / SQLite `.swift` files |
| **SkipSQLPlus `.plus` config** | Use `SkipSQLPlus` + `configuration: .plus` — `.platform` crashes Android at launch (SIGTRAP) | `LocalDatabase.swift`, `Package.swift` |
| **SQLite Serialization** | ALL SQLite access through `DatabaseSyncQueue.shared.run { }` — no concurrent access | DAO / Repository `.swift` files |
| **Entity Sendable** | DAO entity structs + sync metadata structs MUST conform to `Sendable` (cross actor boundary) | `Local*` entities, `Sync*` types |
| **DB Singleton Sendable** | Non-actor DB manager singleton needs `final class ... : @unchecked Sendable` | `LocalDatabase.swift` |
| **var capture in @Sendable** | Copy a built-up `var` array to `let` BEFORE the `@Sendable run { }` closure | Repository sync methods |
| **Timestamp.seconds Int64** | `Timestamp.seconds` is `Int64` on Android — wrap in `Double(...)` for arithmetic | Firestore timestamp parsing |
| **lineLimit ClosedRange Ban** | NO `.lineLimit(2...4)` — only single `Int` overload supported; use `.lineLimit(4)` | TextField/Text views |
| **TextField axis Ban** | NO `TextField(..., axis: .vertical)` — `axis:` parameter unavailable in Skip; use plain `TextField(..., text:)` | Form views with multi-line notes |
| **TextEditor Flexible Height Ban** | NO `TextEditor` with `.frame(minHeight:maxHeight:)` — triggers Compose infinite measure loop (StackOverflowError); use `TextField` with fixed `.frame(height:)` | Compose input areas (message bars, notes) |
| **List Multi-Section Duplicate Key** | Compose `LazyColumn` has a FLAT key namespace across all sections — if any two `ForEach` items share an ID, crash with `IllegalArgumentException: Key "Optional(...)" was already used`. Add type-prefixed `listId: String { "type_\(id)" }` to structs and use `ForEach(items, id: \.listId)` | Any `List` with multiple `Section`/`ForEach` blocks |
| **Combine / @Published / .onReceive Ban** | NO `import Combine`, NO `ObservableObject`, NO `@Published`, NO `.onReceive(publisher:)` — Combine is NOT bridged to Android. Use `@Observable` (Observation macro) ONLY. For notification reactivity use `.task { for await n in NotificationCenter.default.notifications(named:) { } }` | ALL Swift files compiled for Android |
| **applicationIconBadgeNumber Deprecation** | NO `UIApplication.shared.applicationIconBadgeNumber` — deprecated iOS 17; use `UNUserNotificationCenter.current().setBadgeCount()` | AppDelegate lifecycle methods |
| **Sendable Closure Properties** | Closure properties on `View` structs MUST be `@MainActor @Sendable () -> Void` — not plain `() -> Void` | ALL reusable View `.swift` files with closure `let` properties |
| **MainActor from @bridge** | `@bridge` methods are nonisolated — wrap UIKit mutations in `Task { @MainActor in }` | AppDelegate / bridge methods |
| **monospacedDigit Ban (Rule #68)** | NO `.monospacedDigit()` on Font — not bridged to Android. Use `.font(.caption)` + `.frame(width:alignment:)` for tabular alignment. | All Text/Font code |
| **Firestore order(by:) Excludes Docs (Rule #69)** | NEVER use `.order(by:)` unless every document is guaranteed to have that field. Android fetches from server and excludes docs missing the field; iOS cache may return them silently. Fetch unordered, sort in Swift. | All Firestore collection queries with `.order(by:)` |
| **Conditional TabView Tabs (Rule #70)** | NEVER use `if` conditions to add/remove tabs from `TabView`. Android/Compose uses index-based tab paging — inserting a tab shifts all subsequent indices, causing the wrong tab to be selected. Always render all tabs; branch inside each tab's body. | ALL `TabView` with dynamic content |
| **@AppStorage for Tab Selection (Rule #71)** | Use `@State` (NOT `@AppStorage`) for tab selection in login-gated views. `@AppStorage` persists across logout/login, landing the user on the wrong tab. `@State` resets correctly when the view is re-created after login. | All `ContentView`-style auth-gated tab views |
| **@Observable Cross-View Freeze (Rule #73)** | `@Observable` property changes do NOT trigger Compose recomposition in child views that receive the object as a plain `var`. Shadow key display state as `@State` primitives in the owning view; copy values from ViewModel after async load; pass as explicit primitive params to children. | Any parent view using `@State var vm = ViewModel()` that passes state to child views |
| **List Section ForEach Invisible (Rule #74)** | `List { Section { ForEach(...) } }` renders headers but **items are invisible** on Android's Compose `LazyColumn`. Use `ScrollView` + `VStack` with `Text` dividers + `NavigationLink` instead. | Any grouped list view |
| **Empty Button Action (Rule #75)** | `Button(action: {})` renders but is **completely untappable** on both platforms. Use `NavigationLink` for routing, real closures for actions, or plain `HStack` for non-interactive rows. | Any row or card that needs tap handling |

**Quick grep to find violations:**
```bash
# Find @State private violations
grep -r "@State private" mobile-apps/member/Sources/

# Find @FocusState violations
grep -r "@FocusState\|\.focused(" mobile-apps/member/Sources/

# Find NSNull violations
grep -r "NSNull()" mobile-apps/member/Sources/

# Find import SwiftUI violations
grep -r "import SwiftUI" mobile-apps/member/Sources/

# Find .lineLimit range violations
grep -r "\.lineLimit(.*\.\.\.)" mobile-apps/member/Sources/

# Find TextField axis violations
grep -r "axis: \.vertical" mobile-apps/member/Sources/

# Find deprecated UIApplication badge violations
grep -r "applicationIconBadgeNumber" mobile-apps/member/Sources/

# Find non-Sendable closure properties on View structs
grep -r "let (on[A-Z]|action|onTap|onSelect|onDelete|onConfirm|onDismiss): \(\) -> Void" mobile-apps/compass/Sources/ mobile-apps/member/Sources/ mobile-apps/admin/Sources/
```

---

### Dependency Declaration

To use a Kotlin/Java library in `#if SKIP` blocks, declare it in **two places**:

1. **`Sources/<Module>/Skip/skip.yml`** — required for the Skip transpiler to resolve imports:
   ```yaml
   build:
     contents:
       - block: 'dependencies'
         contents:
           - 'implementation("group:artifact:version")'
   ```

2. **`Android/app/build.gradle.kts`** — required for the Android app build (may be redundant if skip.yml handles it, but ensures the dependency is present at all build stages):
   ```kotlin
   dependencies {
       implementation("group:artifact:version")
   }
   ```

---

### Import Syntax in `#if SKIP`

Use fully qualified Java/Kotlin package names, same as you would in Kotlin:

```swift
#if SKIP
import com.google.zxing.BarcodeFormat
import com.google.zxing.qrcode.QRCodeWriter
import android.content.Intent
import android.graphics.Bitmap
import androidx.core.content.FileProvider
#endif
```

For wildcard imports (all public types in a package):
```swift
#if SKIP
import com.google.maps.android.compose.__
#endif
```

---

### Pattern: Android Media Permissions — Google Play Policy Rejection

**Problem:** Google Play rejects apps that declare `READ_MEDIA_IMAGES` or `READ_MEDIA_VIDEO` unless the app's *core purpose* requires persistent, ongoing access to photo/video files in shared storage. Profile photo pickers and one-time image uploads (e.g., uploading a profile picture, attaching a sermon image) do **not** qualify. Submitting with these permissions results in a policy violation rejection even if the app compiled and ran correctly.

**Google Play rejection message:**
```
Photo and Video Permissions policy: Permission use is not directly related to your app's core purpose.
Your app only requires one-time or infrequent access to media files on the device.
Remove the use of READ_MEDIA_IMAGES/READ_MEDIA_VIDEO permission from all version codes.
If your app requires one-time, or limited use of photo and video file, remove the permissions
and consider using the Android photo picker.
```

**Fix:** Remove `READ_MEDIA_IMAGES` and `READ_MEDIA_VIDEO` entirely. Use the **Android photo picker** (system `ActivityResultContracts.PickVisualMedia` — requires zero permissions). Scope legacy storage permissions to older API levels only.

```xml
<!-- ❌ Rejected by Google Play — one-time photo upload does not justify this permission -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_MEDIA_VIDEO" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

<!-- ✅ Correct — photo picker needs NO permissions; scope legacy permissions by SDK -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="28" />
```

**How image upload works without permissions in Skip:**
1. Android photo picker returns a `content://` URI — no storage permission needed
2. In Swift code: branch on `#if SKIP` to call Kotlin's `ContentResolver` or `putFileAsync(from:)` (see Content URI Reading rule)
3. Upload the resolved `Data` or file path directly to Firebase Storage

**Affected files:** `Android/app/src/main/AndroidManifest.xml` in every app module (`member`, `admin`, `compass`)

---

### Pattern: Firestore Field Deletion — `NSNull()` Crashes on Android

**Problem:** Using `NSNull()` in a `[String: Any]` dict passed to `DocumentReference.updateData()` or `setData()` causes a fatal crash on Android. `SkipBridge` cannot convert `NSNull` (an Objective-C type) to a Java/Kotlin object and calls Swift's `assertionFailure`. This crash fires at startup if any code path calls `updateData` with `NSNull()` — even inside a `do/catch` block.

**Crash signature (logcat):**
```
F DEBUG: #00 libswiftCore.so (_assertionFailure+160)
F DEBUG: #01 libSkipBridge.so (AnyBridgingV.toJavaObject+1864)
F DEBUG: #02 libSkipBridge.so (Dictionary.toJavaObject+1372)
F DEBUG: #03 libSkipFirebaseFirestore.so (DocumentReference.updateData+512)
```

**Fix:** Replace `NSNull()` with `FieldValue.delete()`. `FieldValue` is part of `FirebaseFirestore`/`SkipFirebaseFirestore` and is fully bridgeable.

```swift
// ❌ Crashes on Android at runtime — NSNull is not bridgeable by SkipBridge
nonisolated(unsafe) let data: [String: Any] = [
    "plan": "free",
    "billingInterval": NSNull()   // ← fatal crash
]
try await ref.updateData(data)

// ✅ Correct — FieldValue.delete() removes the field and bridges correctly
nonisolated(unsafe) let data: [String: Any] = [
    "plan": "free",
    "billingInterval": FieldValue.delete()
]
try await ref.updateData(data)
```

**Note:** No additional import needed — `FieldValue` is already available from the platform-conditional Firestore import:
```swift
#if os(Android)
@preconcurrency import SkipFirebaseFirestore
#else
@preconcurrency import FirebaseFirestore
#endif
```

---

### Pattern: Firestore `updateData` Data Race Warnings (Swift 6)

**Problem:** Swift 6 strict concurrency checking warns: "sending 'updateData' risks causing data races" when passing `[String: Any]` dictionaries to Firestore.

**Fix:** Mark the dictionary as `nonisolated(unsafe)`:

```swift
// ❌ Warning: Data race risk
var updateData: [String: Any] = ["field": value]
try await ref.updateData(updateData)

// ✅ Correct: Suppress warning with nonisolated(unsafe)
nonisolated(unsafe) var updateData: [String: Any] = ["field": value]
try await ref.updateData(updateData)
```

**Note:** This also applies to `setData()` calls with `[String: Any]` dictionaries.

---

### Pattern: SwiftUI @State Property Visibility

**Problem:** Using `@State private` with custom types (ViewModels, ObservableObjects, CLLocationManager, etc.) causes a "Private state property cannot be bridged to Android" error. Skip Tools cannot bridge `private` properties to Jetpack Compose.

**Error message:**
```
Private state property 'viewModel' cannot be bridged to Android. Consider making this property internal
```

**Fix:** Remove `private` access modifier from `@State` properties in ALL view structs (including subviews). Use `internal` (default) visibility instead.

```swift
// ❌ Error: Private state property cannot be bridged to Android
struct ChurchDiscoveryView: View {
    @State private var viewModel = ChurchDiscoveryViewModel()  // Error
}

struct NearbyChurchesView: View {
    @State private var showRadiusPicker = false  // Error
}

struct ChurchSearchView: View {
    @State private var selectedState = ""  // Error
    @Environment(\.dismiss) private var dismiss  // Also error
}

// ✅ Correct: Remove private from ALL @State properties in ALL views
struct ChurchDiscoveryView: View {
    @State var viewModel = ChurchDiscoveryViewModel()
}

struct NearbyChurchesView: View {
    @State var showRadiusPicker = false
}

struct ChurchSearchView: View {
    @State var selectedState = ""
    @Environment(\.dismiss) var dismiss  // Remove private here too
}
```

**Rule of thumb:**
- **ALL view structs**: Remove `private` from `@State` and `@Environment` properties
- Applies to main view AND all subviews in the same file
- Also applies to: `@Environment(\.dismiss) private var dismiss` → `@Environment(\.dismiss) var dismiss`

**Why:** Skip transpiles SwiftUI to Jetpack Compose. `private` properties are not accessible to the transpiler, which breaks the Compose code generation for state management. This affects EVERY view struct, not just the main one.

---

### Pattern: Import Statement Placement

**Problem:** Platform-conditional imports at the bottom of the file cause "no such module" errors.

**Error message:**
```
no such module 'SkipCoreLocation'
```

**Fix:** ALL import statements must be at the **TOP** of the file, before any type declarations.

**CRITICAL:** `CoreLocation` is **NOT available** in Skip. Neither `import CoreLocation` nor `import SkipCoreLocation` work. Use **SkipDevice** for location functionality.

**Error:**
```
no such module 'CoreLocation'
no such module 'SkipCoreLocation'
```

**Solution:** Use `SkipDevice` with `LocationProvider`, or create a simple coordinate struct:

```swift
// ❌ ERROR - CoreLocation not available
import CoreLocation

// ❌ ERROR - SkipCoreLocation doesn't exist
import SkipCoreLocation

// ✅ CORRECT - Use SkipDevice
import SkipDevice
let provider = LocationProvider()
let location = try await provider.fetchCurrentLocation()
// Access: location.latitude, location.longitude

// ✅ CORRECT - Or use a simple struct
struct LocationCoordinate {
    let latitude: Double
    let longitude: Double
}
```

```swift
// ❌ Error: Import at bottom causes "no such module"
import Foundation

struct MyModel { ... }

// WRONG - imports must be at top!
// Also wrong: SkipCoreLocation is not a standard package
#if os(Android)
import SkipCoreLocation
#else
import CoreLocation
#endif

// ✅ Correct: All imports at top, use standard CoreLocation
import Foundation
import CoreLocation  // Works on both platforms

struct MyModel { ... }
```

---

### Pattern: Color System Backgrounds Not Available

**Problem:** `Color(.systemBackground)`, `Color(.secondarySystemBackground)`, etc. are UIKit-specific and not available in Skip.

**Fix:** Use standard colors with opacity:

```swift
// ❌ ERROR
.background(Color(.secondarySystemBackground))

// ✅ CORRECT
.background(Color.gray.opacity(0.1))
```

---

### Pattern: SwiftUI `swipeActions` Not Available

**Problem:** `.swipeActions(edge:content:)` for List rows doesn't bridge to Android.

**Fix:** Use `.contextMenu` instead:

```swift
// ❌ ERROR
.swipeActions(edge: .trailing) { ... }

// ✅ CORRECT
.contextMenu { ... }
```

---

### Pattern: SwiftUI `PickerStyle.inline` Not Available

**Problem:** `.pickerStyle(.inline)` is not available in Skip.

**Fix:** Use `.pickerStyle(.segmented)` or `.pickerStyle(.menu)` instead:

```swift
// ❌ ERROR
Picker("Choose", selection: $value) { ... }
    .pickerStyle(.inline)

// ✅ CORRECT
Picker("Choose", selection: $value) { ... }
    .pickerStyle(.segmented)
```

---

### Pattern: SwiftUI ShapeStyle `.accent` Not Available

**Problem:** Using `.accent` as a ShapeStyle (e.g., `.foregroundStyle(.accent)`) causes a "Type 'ShapeStyle' has no member 'accent'" error. The `.accent` static property on ShapeStyle is not available in Skip.

**Error message:**
```
Type 'ShapeStyle' has no member 'accent'
```

**Fix:** Use `Color.accentColor` instead of `.accent`.

```swift
// ❌ Error: Type 'ShapeStyle' has no member 'accent'
Button("Tap Me") {
    .foregroundStyle(.accent)
}

Image(systemName: "checkmark")
    .foregroundStyle(.accent)

// ✅ Correct: Use Color.accentColor
Button("Tap Me") {
    .foregroundStyle(Color.accentColor)
}

Image(systemName: "checkmark")
    .foregroundStyle(Color.accentColor)
```

---

### Pattern: Firebase Functions — Direct HTTP with ID Token

**Problem:** Need to call Firebase Cloud Functions from the app. `httpsCallable.call(payload)` where `payload` is `[String: Any]` causes a Swift 6 Sendable warning (`Sending value of non-Sendable type '[String : Any]' risks causing data races`) because `call(_:)` takes `Any?` (non-`Sendable`). `nonisolated(unsafe)` does NOT suppress this at the call site.

**Solution:** Use direct HTTP with an ID token on **both** iOS and Android. Serialize `[String: Any]` to `Data` synchronously **before** any `await` so it never crosses a concurrency boundary. This is the pattern used in `StripePaymentService` (Coffee House) and `RSVPViewModel` (Church Compass).

```swift
// Imports — only FirebaseAuth needed (no FirebaseFunctions)
#if os(Android)
@preconcurrency import SkipFirebaseAuth
#else
@preconcurrency import FirebaseAuth
#endif

// In your ViewModel/Service:
private func invokeCallable(_ name: String, payload: [String: Any]) async throws -> [String: Any] {
    guard let user = Auth.auth().currentUser else {
        throw NSError(domain: "MyViewModel", code: 401,
                      userInfo: [NSLocalizedDescriptionKey: "User not authenticated."])
    }

    // ✅ Serialize to Data (Sendable) BEFORE any await
    // [String: Any] is consumed synchronously here — never crosses a suspension point
    let bodyData = try JSONSerialization.data(withJSONObject: ["data": payload])

    // Now safe to await — only Data and String cross the boundary
    #if os(Android)
    let idToken = try await user.getIDToken()
    #else
    let idToken: String = try await withCheckedThrowingContinuation { cont in
        user.getIDTokenForcingRefresh(false) { token, error in
            if let token { cont.resume(returning: token) }
            else { cont.resume(throwing: error ?? URLError(.unknown)) }
        }
    }
    #endif

    return try await invokeHTTP(functionName: name, bodyData: bodyData, idToken: idToken)
}

// Helper accepts Data (Sendable), not [String: Any]
private func invokeHTTP(
    functionName: String,
    bodyData: Data,
    idToken: String
) async throws -> [String: Any] {
    let url = URL(string: "https://us-central1-YOUR_PROJECT.cloudfunctions.net")
        .appendingPathComponent(functionName)
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.setValue("Bearer \(idToken)", forHTTPHeaderField: "Authorization")
    request.httpBody = bodyData
    let (data, response) = try await URLSession.shared.data(for: request)
    guard let httpResponse = response as? HTTPURLResponse,
          (200..<300).contains(httpResponse.statusCode) else {
        throw NSError(domain: "MyViewModel", code: -1,
                      userInfo: [NSLocalizedDescriptionKey: "HTTP error calling \(functionName)"])
    }
    guard let json = try JSONSerialization.jsonObject(with: data) as? [String: Any] else {
        throw NSError(domain: "MyViewModel", code: -1,
                      userInfo: [NSLocalizedDescriptionKey: "Invalid JSON response"])
    }
    return json["result"] as? [String: Any] ?? json
}
```

**❌ Avoid — causes Sendable warning:**
```swift
// DO NOT use httpsCallable.call(payload) — call(_:) takes Any? (non-Sendable)
let callable = Functions.functions().httpsCallable("functionName")
let result = try await callable.call(payload)  // ← WARNING

// DO NOT pass [String: Any] as argument to async function after an await
let idToken = try await user.getIDToken()
return try await invokeHTTP(payload: payload, ...)  // ← WARNING — payload crossed boundary
```

**Note:** `SkipFirebaseFunctions` import is no longer needed when using direct HTTP.

---

### Pattern: `[String: Any]` Must Not Cross Async Suspension Points

**Problem:** Swift 6 strict concurrency: `[String: Any]` is not `Sendable` (because `Any` is not `Sendable`). Passing it as an argument to an `async` function — even if declared `nonisolated(unsafe)` locally — causes:

```
Sending value of non-Sendable type '[String : Any]' risks causing data races
Sending 'payload' risks causing data races
```

**Root cause:** The warning fires at the *call site* where the value is passed to an `async throws` function, not where it is declared. `nonisolated(unsafe)` suppresses the declaration warning but **not** the passing-to-async-function warning.

**Fix:** Serialize to `Data` (which IS `Sendable`) synchronously before the first `await`:

```swift
// ❌ WRONG — [String: Any] passed as argument across async boundary
private func send(_ name: String, payload: [String: Any]) async throws {
    let idToken = try await user.getIDToken()   // ← first await
    try await upload(payload: payload, ...)     // ← Sendable warning here
}

// ✅ CORRECT — serialize synchronously before any await
private func send(_ name: String, payload: [String: Any]) async throws {
    // Consumed synchronously — never an async argument
    let bodyData = try JSONSerialization.data(withJSONObject: payload)

    let idToken = try await user.getIDToken()   // only Data crosses here
    try await upload(bodyData: bodyData, ...)   // Data is Sendable ✓
}
```

**Key rules:**
- `[String: Any]` must be fully consumed by `JSONSerialization.data(withJSONObject:)` **before** the first `await`
- After serialization, pass only `Data`, `String`, `Int`, `Bool` (all `Sendable`) across `await`s
- `nonisolated(unsafe)` on a local copy does NOT suppress the warning when that copy is passed to another `async` function
- This applies to Firestore `updateData`/`setData` calls too — prefer `nonisolated(unsafe)` there since Firestore accepts the dict directly without an intermediate async function

---

### Pattern: QR Code Generation & Image Sharing (Android)

**Problem:** Need to generate a QR code bitmap and share it as an image via Android's share sheet. The Swift `QRCodeGenerator` module is compiled as native Swift in Fuse mode and is NOT accessible from `#if SKIP` blocks.

**Solution:** Use ZXing (Java QR library) inside `#if SKIP` + Android `FileProvider` for secure file sharing.

**Dependencies:**

`Sources/ApiLog/Skip/skip.yml`:
```yaml
build:
  contents:
    - block: 'dependencies'
      contents:
        - 'implementation("com.google.zxing:core:3.5.3")'
```

`Android/app/build.gradle.kts`:
```kotlin
dependencies {
    implementation("com.google.zxing:core:3.5.3")
}
```

**Android Manifest** — Add `FileProvider` to `AndroidManifest.xml`:
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

**File paths resource** — `Android/app/src/main/res/xml/file_paths.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="shared_images" path="shared_images/" />
</paths>
```

**Implementation:**
```swift
// Button in View body calls shareQRImage on Android
Button {
    #if SKIP
    shareQRImage(content: qrContent, label: "Hive: \(hive.name)")
    #else
    showShareSheet = true
    #endif
} label: {
    Label("Share / Print Label", systemImage: "square.and.arrow.up")
}

// #if SKIP block — transpiled to Kotlin, can use Java APIs
#if SKIP
import android.content.Intent
import android.graphics.Bitmap
import androidx.core.content.FileProvider
import com.google.zxing.BarcodeFormat
import com.google.zxing.qrcode.QRCodeWriter

func shareQRImage(content: String, label: String) {
    let size = 512
    let writer = QRCodeWriter()
    let bitMatrix = writer.encode(content, BarcodeFormat.QR_CODE, size, size)
    let bitmap = Bitmap.createBitmap(size, size, Bitmap.Config.ARGB_8888)
    for y in 0..<size {
        for x in 0..<size {
            bitmap.setPixel(x, y, bitMatrix.get(x, y) ? android.graphics.Color.BLACK : android.graphics.Color.WHITE)
        }
    }

    let ctx = ProcessInfo.processInfo.androidContext
    let cacheDir = java.io.File(ctx.cacheDir, "shared_images")
    cacheDir.mkdirs()
    let file = java.io.File(cacheDir, "qr_label.png")
    let outStream = java.io.FileOutputStream(file)
    bitmap.compress(Bitmap.CompressFormat.PNG, 100, outStream)
    outStream.flush()
    outStream.close()

    let authority = ctx.packageName + ".fileprovider"
    let uri = FileProvider.getUriForFile(ctx, authority, file)

    let intent = Intent(Intent.ACTION_SEND)
    intent.setType("image/png")
    intent.putExtra(Intent.EXTRA_STREAM, uri)
    intent.putExtra(Intent.EXTRA_TEXT, label)
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    let chooser = Intent.createChooser(intent, "Share QR Label")
    chooser.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
    ctx.startActivity(chooser)
}
#endif
```

**Key Takeaways:**
- Swift-only modules (like `QRCodeGenerator`) are NOT visible in `#if SKIP` — use a Java equivalent
- `ProcessInfo.processInfo.androidContext` provides the Android `Context`
- `FileProvider` + cache directory is the safe way to share files via intents
- Use `FLAG_ACTIVITY_NEW_TASK` on the chooser intent when launching from non-Activity context

---

### Pattern: Google Maps Compose Integration

**Problem:** Display a native Google Map on Android using Jetpack Compose APIs.

**Solution:** Use `ComposeView` + `ContentComposer` pattern.

**Dependencies:**

`Sources/<Module>/Skip/skip.yml`:
```yaml
build:
  contents:
    - block: 'dependencies'
      contents:
        - 'implementation("com.google.maps.android:maps-compose:6.4.1")'
```

**Implementation:**
```swift
struct MapView: View {
    let latitude: Double
    let longitude: Double

    var body: some View {
        #if os(Android)
        ComposeView { MapComposer(latitude: latitude, longitude: longitude) }
        #else
        Map(initialPosition: .region(...))
        #endif
    }
}

#if SKIP
import com.google.maps.android.compose.__
import com.google.android.gms.maps.model.CameraPosition
import com.google.android.gms.maps.model.LatLng

struct MapComposer: ContentComposer {
    let latitude: Double
    let longitude: Double

    @Composable func Compose(context: ComposeContext) {
        GoogleMap(cameraPositionState: rememberCameraPositionState {
            position = CameraPosition.fromLatLngZoom(LatLng(latitude, longitude), Float(12.0))
        })
    }
}
#endif
```

**Key Takeaways:**
- `ComposeView` bridges from SwiftUI to Jetpack Compose
- `ContentComposer` is the struct protocol for Compose content
- `@Composable` annotation on the `Compose` function enables Compose APIs
- The `#if os(Android)` around `ComposeView` is fine because it's a SwiftUI view — no Kotlin APIs needed

---

### Pattern: Android Context Access

**Skip Fuse:**
```swift
#if SKIP
let ctx = ProcessInfo.processInfo.androidContext
#endif
```

**Skip Lite:**
```swift
#if SKIP
let ctx = ProcessInfo.processInfo.androidContext
#endif
```

Both modes use the same API. Returns `android.content.Context`.

---

### Pattern: Skip Fuse Native Mode — Calling Android APIs via Compose Bridge

**Problem:** In Skip Fuse native mode (`mode: 'native'` in `skip.yml`), `#if SKIP` is **false**, so code that directly calls Android APIs via `ProcessInfo.processInfo.androidContext` is excluded. You need to access Android Java APIs from native Swift code.

**Solution:** Use `ComposeView` + `ContentComposer` to bridge from SwiftUI to Jetpack Compose, then call Java APIs from the `@Composable` function.

**Dependencies:** Same as any Java library — add to both `skip.yml` and `build.gradle.kts`.

**Implementation:**

```swift
// MARK: - Native Swift side (#if os(Android))

#if os(Android)
import androidx.compose.ui.platform.ComposeView
import androidx.compose.runtime.Composable

/// Shared state between native Swift and Compose
@Observable final class ShareState: @unchecked Sendable {
    static let shared = ShareState()
    var trigger: Int = 0
    var content: String = ""

    func share(content: String) {
        self.content = content
        self.trigger += 1
    }
}

#if os(Android)
/// Invisible bridge view — add to your view hierarchy
/// Pass primitive values to ContentComposer, NOT @Binding with custom class
/// NOTE: No Android imports needed here — ComposeView is from SkipFuseUI
struct ShareTriggerView: View {
    @State var state = ShareState.shared

    var body: some View {
        ComposeView {
            ShareComposer(
                content: state.content,
                trigger: state.trigger
            )
        }
        .frame(width: 0, height: 0)
        .opacity(0)
    }
}

/// Called from button action
func shareContent(_ content: String) {
    ShareState.shared.share(content: content)
}
#endif

// MARK: - Kotlin-transpiled side (#if SKIP)

#if SKIP
// Android imports go here (transpiled to Kotlin)
import android.content.Intent
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.Composable

/// ContentComposer uses primitive `let` properties, NOT @Binding with custom class
/// Skip cannot bridge @Binding with custom structs — use primitives only
struct ShareComposer: ContentComposer {
    let content: String
    let trigger: Int

    init(content: String, trigger: Int) {
        self.content = content
        self.trigger = trigger
    }

    @Composable func Compose(context: ComposeContext) {
        LaunchedEffect(trigger) {
            if trigger > 0 {
                performShare(content)
            }
        }
    }

    func performShare(_ content: String) {
        let ctx = ProcessInfo.processInfo.androidContext
        let intent = Intent(Intent.ACTION_SEND)
        intent.setType("text/plain")
        intent.putExtra(Intent.EXTRA_TEXT, content)
        let chooser = Intent.createChooser(intent, "Share")
        chooser.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        ctx.startActivity(chooser)
    }
}
#endif
```

**Usage in View:**
```swift
var body: some View {
    VStack {
        // ... content ...

        #if os(Android)
        ShareTriggerView()  // Invisible bridge
        #endif

        Button {
            #if os(Android)
            shareContent("Hello")
            #else
            // iOS share implementation
            #endif
        } label: {
            Text("Share")
        }
    }
}
```

**Key Takeaways:**
- In Skip Fuse native mode, use `#if os(Android)` for SwiftUI, `#if SKIP` for Compose/Kotlin code
- The `@Observable` state class bridges between native Swift and transpiled Compose
- **`@Observable` classes MUST be outside all `#if` blocks** to be bridged to both platforms — if inside `#if os(Android)` or `#if SKIP`, the type won't be bridged and you'll get "does not appear to be a bridged type" errors
- **`ContentComposer` must use primitive `let` properties** — Skip cannot bridge `@Binding` with custom structs. Pass `String`, `Int`, `Bool`, etc. directly
- `ComposeView` is the only way to access Android Java APIs from Skip Fuse
- The bridge view should be invisible (0x0 frame, opacity 0)
- Trigger changes via shared state, react in `LaunchedEffect`
- **`LaunchedEffect` requires explicit import** — add `import androidx.compose.runtime.LaunchedEffect` inside the `#if SKIP` block; it is NOT auto-imported even though it is a standard Compose runtime API

---

### Pattern: Deep Link Handling with QR Codes (Cold Start Compatible)

**Context:** Handle QR code scans from camera app to navigate directly to specific entities (Hive, Apiary, Garden). Must work when app is closed (cold start) or already running.

**QR Code URL Format:**
```swift
// Hive
"apilog://hive/\(hive.id)?code=\(hive.qrCode)&name=\(hive.name)"

// Apiary  
"apilog://apiary/\(apiary.id)?name=\(apiary.name)"

// Garden
"apilog://garden/\(garden.id)?code=\(garden.qrCode)&name=\(garden.name)"
```

**Implementation:**

1. **DeepLinkHandler.swift** — Parses URLs, manages state, persists for cold start:
```swift
@MainActor
@Observable final class DeepLinkHandler: @unchecked Sendable {
    static let shared = DeepLinkHandler()
    private let pendingDeepLinkKey = "pendingDeepLink"
    
    var pendingDestination: DeepLinkDestination?
    var isProcessingDeepLink = false
    
    enum DeepLinkDestination: Hashable {
        case hive(id: String, qrCode: String?)
        case apiary(id: String)
        case garden(id: String, qrCode: String?)
        case billing
        case unknown
    }
    
    private init() {
        // Restore deep link from previous session (cold start)
        if let savedData = UserDefaults.standard.data(forKey: pendingDeepLinkKey),
           let savedURL = try? JSONDecoder().decode(URL.self, from: savedData) {
            let destination = handleURL(savedURL)
            if destination != .unknown {
                setPendingDestination(destination)
            }
            UserDefaults.standard.removeObject(forKey: pendingDeepLinkKey)
        }
    }
    
    func handleURL(_ url: URL) -> DeepLinkDestination {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false) else {
            return .unknown
        }
        let path = url.path.lowercased()
        let queryItems = components.queryItems ?? []
        
        switch components.host?.lowercased() {
        case "hive":
            let id = path.trimmingCharacters(in: CharacterSet(charactersIn: "/"))
            let qrCode = queryItems.first(where: { $0.name == "code" })?.value
            return .hive(id: id, qrCode: qrCode)
        case "apiary":
            let id = path.trimmingCharacters(in: CharacterSet(charactersIn: "/"))
            return .apiary(id: id)
        case "garden":
            let id = path.trimmingCharacters(in: CharacterSet(charactersIn: "/"))
            let qrCode = queryItems.first(where: { $0.name == "code" })?.value
            return .garden(id: id, qrCode: qrCode)
        default:
            return .unknown
        }
    }
    
    func setPendingDestination(_ destination: DeepLinkDestination) {
        pendingDestination = destination
        isProcessingDeepLink = true
    }
    
    func clearPendingDestination() {
        pendingDestination = nil
        isProcessingDeepLink = false
        UserDefaults.standard.removeObject(forKey: pendingDeepLinkKey)
    }
    
    func postDeepLinkNotification(_ destination: DeepLinkDestination, url: URL? = nil) {
        setPendingDestination(destination)
        if let url = url {
            if let data = try? JSONEncoder().encode(url) {
                UserDefaults.standard.set(data, forKey: pendingDeepLinkKey)
            }
        }
        NotificationCenter.default.post(
            name: .deepLinkReceived,
            object: nil,
            userInfo: ["destination": destination]
        )
    }
}

extension Notification.Name {
    static let deepLinkReceived = Notification.Name("deepLinkReceived")
}
```

2. **ApiLogApp.swift** — Handle incoming URLs:
```swift
.onOpenURL { url in
    let destination = DeepLinkHandler.shared.handleURL(url)
    switch destination {
    case .hive, .apiary, .garden:
        DeepLinkHandler.shared.postDeepLinkNotification(destination, url: url)
    case .billing:
        await SubscriptionService.shared.refresh()
    case .unknown:
        logger.warning("Unhandled deep link: \(url)")
    }
}
```

3. **HomeView.swift** — Handle navigation with cold start support:
```swift
struct HomeView: View {
    @State var tab: HomeTab = .home
    @State var hivesPath = NavigationPath()
    @State var apiariesPath = NavigationPath()
    @State var deepLinkHandler = DeepLinkHandler.shared
    @State var workspaceReady = false
    @State var coldStartDeepLink: DeepLinkHandler.DeepLinkDestination?
    
    var body: some View {
        TabView(selection: $tab) {
            NavigationStack(path: $hivesPath) {
                HiveListView(service: sharedService)
                    .navigationDestination(for: Hive.self) { hive in
                        HiveDetailView(hive: hive, service: sharedService)
                    }
            }
            .tabItem { Label("Hives", systemImage: "hexagon.fill") }
            .tag(HomeTab.hives)
            // ... other tabs
        }
        .onChange(of: deepLinkHandler.isProcessingDeepLink) { _, isProcessing in
            if isProcessing { checkAndProcessPendingDeepLink() }
        }
        .task { await handleTask() }
    }
    
    private func handleTask() async {
        await subscriptionService.loadWorkspace()
        workspaceReady = true
        
        // Wait for data to load from Firestore
        var attempts = 0
        while sharedService.apiaries.isEmpty && attempts < 10 {
            try? await Task.sleep(for: .seconds(0.5))
            attempts += 1
        }
        
        // Process any pending deep links
        checkAndProcessPendingDeepLink()
        
        if let coldStartLink = coldStartDeepLink {
            handleDeepLink(coldStartLink)
            coldStartDeepLink = nil
        }
    }
    
    private func checkAndProcessPendingDeepLink() {
        if deepLinkHandler.isProcessingDeepLink,
           let destination = deepLinkHandler.pendingDestination {
            // Only process if we have data loaded
            if workspaceReady && !sharedService.apiaries.isEmpty {
                handleDeepLink(destination)
            } else {
                // Store for processing after workspace loads
                coldStartDeepLink = destination
                deepLinkHandler.clearPendingDestination()
            }
        }
    }
    
    private func handleDeepLink(_ destination: DeepLinkHandler.DeepLinkDestination) {
        switch destination {
        case .hive(let id, _):
            tab = .hives
            if let hive = findHive(byId: id) {
                hivesPath.append(hive)
            }
        case .apiary(let id):
            tab = .apiaries
            if let apiary = sharedService.apiaries.first(where: { $0.id.lowercased() == id.lowercased() }) {
                apiariesPath.append(apiary)
            }
        // ... garden handling
        }
        deepLinkHandler.clearPendingDestination()
    }
    
    private func findHive(byId id: String) -> Hive? {
        for apiary in sharedService.apiaries {
            if let hive = apiary.hives.first(where: { $0.id.lowercased() == id.lowercased() }) {
                return hive
            }
        }
        return nil
    }
}
```

**Critical Implementation Details:**

1. **Cold Start Persistence:** Use `UserDefaults` to store the deep link URL when app is killed mid-navigation. Restore in `DeepLinkHandler.init()`.

2. **Main Actor Isolation:** Mark `DeepLinkHandler` as `@MainActor` to ensure all state updates happen on main thread. This prevents race conditions during cold start.

3. **Data Loading Wait:** Always wait for workspace data to load before navigating. The deep link may arrive before Firestore sync completes.

4. **Case-Insensitive Matching:** UUIDs in URLs may have different casing. Always compare with `.lowercased()`.

5. **NavigationDestination Placement:** Put `.navigationDestination(for:)` inside the `NavigationStack` content, NOT on the `NavigationStack` itself:
```swift
// ❌ Wrong - causes "misplaced navigationDestination" warning
NavigationStack(path: $hivesPath) {
    HiveListView()
}
.navigationDestination(for: Hive.self) { ... }  // Wrong!

// ✅ Correct
NavigationStack(path: $hivesPath) {
    HiveListView()
        .navigationDestination(for: Hive.self) { ... }  // Correct!
}
```

6. **Navigation Title Display:** Add `.navigationBarTitleDisplayMode(.inline)` to ensure title appears in navigation bar:
```swift
HiveDetailView(hive: hive)
    .navigationTitle(hive.name)
    .navigationBarTitleDisplayMode(.inline)
```

**Key Points:**
- Use `NavigationPath` + `NavigationStack(path:)` for programmatic navigation
- Store deep link URL in `UserDefaults` for cold start recovery
- Wait for workspace data to load before processing deep links
- Use `@MainActor` for state consistency
- Always verify entity exists before attempting navigation
- Handle case-insensitive UUID matching

---

### Pattern: QR Code Generation and Sharing (Skip Fuse)

**Context:** QR labels that work on both iOS and Android, with deep-linking and native share functionality.

**Architecture Overview:**
- **iOS:** Use CoreImage for QR generation, UIActivityViewController for sharing
- **Android (Skip Fuse):** Use ZXing library for QR generation, Intent with FileProvider for sharing
- **Deep Links:** Custom URL scheme (`apilog://`) registered in AndroidManifest.xml and handled via onOpenURL

**Dependencies:**

`skip.yml`:
```yaml
dependencies:
  - com.google.zxing:core:3.5.3
```

`Android/app/build.gradle.kts`:
```kotlin
dependencies {
    implementation("com.google.zxing:core:3.5.3")
}
```

`AndroidManifest.xml` — FileProvider for sharing:
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

`res/xml/file_paths.xml`:
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="shared_images" path="shared_images/" />
</paths>
```

**Complete Implementation:**

```swift
// MARK: - QR Label View (Native Swift for both platforms)

struct HiveQRLabelView: View {
    var hive: Hive
    @State var showShareSheet = false
    @Environment(\.dismiss) var dismiss

    // Use apilog:// scheme — must match AndroidManifest.xml intent-filter
    var qrContent: String {
        "apilog://hive/\(hive.id)?code=\(hive.qrCode)&name=\(hive.name.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? hive.name)"
    }

    var body: some View {
        VStack(spacing: AppTheme.largePadding) {
            // ... QR display, labels, etc ...

            #if os(Android)
            // Invisible bridge view — MUST be in view hierarchy before button
            QRShareTriggerView()
                .frame(width: 0, height: 0)
                .opacity(0)
            #endif

            Spacer(minLength: AppTheme.largePadding)

            Button {
                #if os(Android)
                shareQRImageAndroid(content: qrContent, label: "Hive: \(hive.name)")
                #else
                showShareSheet = true
                #endif
            } label: {
                Label("Share / Print Label", systemImage: "square.and.arrow.up")
                    .fontWeight(.semibold)
                    .padding()
                    .frame(maxWidth: .infinity)
                    .background(AppTheme.primaryColor)
                    .foregroundColor(.white)
                    .cornerRadius(AppTheme.cornerRadius)
            }
            .padding(.horizontal, AppTheme.largePadding)
            .padding(.bottom, AppTheme.largePadding)
        }
        .padding(.top, AppTheme.largePadding)
        .padding(.horizontal, AppTheme.largePadding)
        .sheet(isPresented: $showShareSheet) {
            ShareSheetView(items: [qrContent, "Hive: \(hive.name)"], qrContent: qrContent)
        }
    }
}

// MARK: - Shared State (MUST be outside ALL #if blocks)

/// Observable state shared between native Swift and transpiled Compose
/// CRITICAL: Placing this inside #if os(Android) or #if SKIP breaks type bridging
@Observable final class QRShareState: @unchecked Sendable {
    static let shared = QRShareState()
    var content: String = ""
    var label: String = ""
    var trigger: Int = 0

    func share(content: String, label: String) {
        self.content = content
        self.label = label
        self.trigger += 1
    }
}

// MARK: - Native Swift Side (#if os(Android))

#if os(Android)
/// Invisible bridge view — no Android imports needed (ComposeView from SkipFuseUI)
struct QRShareTriggerView: View {
    @State var state = QRShareState.shared

    var body: some View {
        ComposeView {
            // Pass primitive values, NOT @Binding with custom class
            QRShareComposer(
                content: state.content,
                label: state.label,
                trigger: state.trigger
            )
        }
    }
}

/// Called from button action — triggers share via state change
func shareQRImageAndroid(content: String, label: String) {
    QRShareState.shared.share(content: content, label: label)
}
#endif

// MARK: - Kotlin-Transpiled Side (#if SKIP)

#if SKIP
// All Android imports go here — this block is transpiled to Kotlin
import android.content.Intent
import android.graphics.Bitmap
import android.util.Log
import androidx.core.content.FileProvider
import com.google.zxing.BarcodeFormat
import com.google.zxing.qrcode.QRCodeWriter
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.Composable

/// ContentComposer — uses primitive `let` properties only
/// CRITICAL: @Binding with custom structs fails with "does not appear to be a bridged type"
struct QRShareComposer: ContentComposer {
    let content: String
    let label: String
    let trigger: Int

    init(content: String, label: String, trigger: Int) {
        self.content = content
        self.label = label
        self.trigger = trigger
    }

    @Composable func Compose(context: ComposeContext) {
        LaunchedEffect(trigger) {
            if trigger > 0 {
                performShare(content: content, label: label)
            }
        }
    }

    func performShare(content: String, label: String) {
        let ctx = ProcessInfo.processInfo.androidContext
        GlobalScope.launch(Dispatchers.IO) {
            do {
                // Generate QR with ZXing
                let size = 512
                let writer = QRCodeWriter()
                let bitMatrix = writer.encode(content, BarcodeFormat.QR_CODE, size, size)
                let pixels = kotlin.IntArray(size * size)
                for y in 0..<size {
                    for x in 0..<size {
                        pixels[y * size + x] = bitMatrix.get(x, y) 
                            ? android.graphics.Color.BLACK 
                            : android.graphics.Color.WHITE
                    }
                }
                let bitmap = Bitmap.createBitmap(pixels, size, size, Bitmap.Config.ARGB_8888)

                // Write to cache dir (matches file_paths.xml)
                let cacheDir = java.io.File(ctx.cacheDir, "shared_images")
                cacheDir.mkdirs()
                let file = java.io.File(cacheDir, "qr_label.png")
                
                Log.d("QRShare", "Writing QR image to: \(file.absolutePath)")
                let outStream = java.io.FileOutputStream(file)
                bitmap.compress(Bitmap.CompressFormat.PNG, 100, outStream)
                outStream.flush()
                outStream.close()
                Log.d("QRShare", "QR image written, size: \(file.length()) bytes")

                // Create content URI via FileProvider
                let authority = ctx.packageName + ".fileprovider"
                let uri = FileProvider.getUriForFile(ctx, authority, file)
                Log.d("QRShare", "Content URI: \(uri.toString())")

                // Launch share intent
                withContext(Dispatchers.Main) {
                    let intent = Intent(Intent.ACTION_SEND)
                    intent.setType("image/png")
                    intent.putExtra(Intent.EXTRA_STREAM, uri)
                    intent.putExtra(Intent.EXTRA_TEXT, label)
                    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
                    let chooser = Intent.createChooser(intent, "Share QR Label")
                    chooser.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                    ctx.startActivity(chooser)
                    Log.d("QRShare", "Share intent launched")
                }
            } catch {
                Log.e("QRShare", "Failed to share: \(error)")
            }
        }
    }
}
#endif
```

**Critical Rules for QR/Sharing:**

1. **URL Scheme Consistency** — QR codes and AndroidManifest.xml must use the SAME scheme:
   ```swift
   // QR content
   "apilog://hive/\(id)?code=\(code)"
   
   // AndroidManifest.xml
   <data android:scheme="apilog" />
   ```

2. **FileProvider Authority** — Must match package name + ".fileprovider":
   ```kotlin
   val authority = ctx.packageName + ".fileprovider"
   ```

3. **Cache Path** — Must match `file_paths.xml`:
   ```kotlin
   File(ctx.cacheDir, "shared_images")  // matches <cache-path name="shared_images">
   ```

4. **Logging in #if SKIP** — Use `android.util.Log`, NOT Swift `logger`:
   ```swift
   Log.d("QRShare", "message")  // ✓ Works in #if SKIP
   logger.info("message")        // ✗ Inaccessible in #if SKIP
   ```

5. **Order Matters** — `QRShareTriggerView` must be in view hierarchy BEFORE button is tapped

6. **Layout Spacing** — Use `Spacer(minLength:)` and `.padding(.bottom:)` to prevent button cutoff:
   ```swift
   Spacer(minLength: AppTheme.largePadding)
   Button { ... }
   .padding(.bottom, AppTheme.largePadding)
   ```

**Debugging:**
```bash
# View QR share logs
adb logcat -s "QRShare"

# Verify FileProvider is registered
adb shell dumpsys package com.floatingaxeheadministries.apilog | grep -i fileprovider
```

---

### Pattern: Subscription Verification & Expiry Detection (Skip Marketplace)

**Context:** Handle subscription renewals, cancellations, and expirations across iOS (App Store) and Android (Google Play) using Skip Marketplace. When a user cancels or a subscription expires, the app must detect this and update Firestore to synchronize with web app.

**Problem:** Auto-renewal works automatically, but cancellations and expirations are "silent" — the app only knows by checking current entitlements. Without periodic verification, cancelled subscriptions continue showing as "active" in Firestore.

**Solution:** Call `verifySubscriptionStatus()` on app launch to check `Marketplace.fetchEntitlements()` and sync with Firestore.

**Dependencies:**

`Sources/<Module>/Skip/skip.yml`:
```yaml
build:
  contents:
    - block: 'dependencies'
      contents:
        - 'implementation("skip.marketplace:skip-marketplace:0.x.x")'
```

**Implementation:**

1. **IAPService.swift** — Add verification method:
```swift
@Observable final class IAPService: @unchecked Sendable {
    static let shared = IAPService()
    
    /// Checks current subscription status and updates Firestore if expired/cancelled
    /// Call this on app launch to ensure subscription status is accurate
    func verifySubscriptionStatus() async {
        logger.info("[IAPService] Verifying subscription status...")
        let wsId = SubscriptionService.shared.workspaceId
        guard !wsId.isEmpty else { return }
        
        do {
            let entitlements = try await Marketplace.current.fetchEntitlements()
            
            // Check if we have any active subscription entitlements
            let activeSubscriptions = entitlements.filter { transaction in
                guard let productId = transaction.products.first else { return false }
                return IAPProducts.plan(for: productId) != nil
            }
            
            if activeSubscriptions.isEmpty {
                // No active subscriptions found - user has expired or cancelled
                logger.info("[IAPService] No active subscriptions found, downgrading to free")
                await downgradeToFreePlan()
            } else {
                // Has active subscription - verify it's the correct one
                if let transaction = activeSubscriptions.first,
                   let productId = transaction.products.first,
                   let plan = IAPProducts.plan(for: productId),
                   let interval = IAPProducts.interval(for: productId) {
                    logger.info("[IAPService] Active subscription verified: \(productId)")
                    await updateFirestorePlan(plan: plan, interval: interval)
                }
            }
        } catch {
            logger.error("[IAPService] verifySubscriptionStatus error: \(error)")
        }
    }
    
    /// Downgrades user to free plan when subscription expires or is cancelled
    private func downgradeToFreePlan() async {
        let wsId = SubscriptionService.shared.workspaceId
        guard !wsId.isEmpty else { return }
        
        let db = Firestore.firestore()
        let freeConfig = PlanConfigs.free
        
        nonisolated(unsafe) let data: [String: Any] = [
            "plan": SubscriptionPlan.free.rawValue,
            "subscriptionStatus": SubscriptionStatus.free.rawValue,
            "billingInterval": NSNull(), // Clear the billing interval
            "hiveLimit": freeConfig.hiveLimit,
            "userLimit": freeConfig.userLimit,
            "apiaryLimit": freeConfig.apiaryLimit,
            "cancelAtPeriodEnd": false,
            "hasUsedTrial": true
        ]
        
        do {
            try await db.collection("workspaces").document(wsId).updateData(data)
            await SubscriptionService.shared.refresh()
            logger.info("[IAPService] Downgraded to free plan")
        } catch {
            logger.error("[IAPService] downgradeToFreePlan error: \(error)")
        }
    }
    
    func updateFirestorePlan(plan: SubscriptionPlan, interval: BillingInterval?) async {
        let wsId = SubscriptionService.shared.workspaceId
        guard !wsId.isEmpty else { return }
        let db = Firestore.firestore()
        let config = PlanConfigs.config(for: plan)
        
        var data: [String: Any] = [
            "plan": plan.rawValue,
            "subscriptionStatus": plan == .free ? SubscriptionStatus.free.rawValue : "active",
            "hiveLimit": config.hiveLimit,
            "userLimit": config.userLimit,
            "apiaryLimit": config.apiaryLimit,
        ]
        
        if let interval = interval {
            data["billingInterval"] = interval.rawValue
        } else {
            data["billingInterval"] = NSNull()
        }
        
        nonisolated(unsafe) let finalData = data
        
        do {
            try await db.collection("workspaces").document(wsId).updateData(finalData)
            await SubscriptionService.shared.refresh()
        } catch {
            logger.error("[IAPService] updateFirestore error: \(error)")
        }
    }
}
```

2. **HomeView.swift** — Call on app launch:
```swift
private func handleTask() async {
    await subscriptionService.loadWorkspace()
    workspaceReady = true
    
    // Verify subscription status on launch (handles expiry/cancellation)
    await IAPService.shared.verifySubscriptionStatus()
    
    // ... rest of initialization
}
```

3. **PlanConfig.swift** — Map product IDs to plans:
```swift
enum IAPProducts {
    static func plan(for productId: String) -> SubscriptionPlan? {
        switch productId {
        case "com.faam.apilog.babybee.monthly", "com.faam.apilog.babybee.yearly",
             "com.floatingaxeheadministries.apilog.babybee.monthly",
             "com.floatingaxeheadministries.apilog.babybee.yearly":
            return .babyBee
        case "com.faam.apilog.hobby.monthly", "com.faam.apilog.hobby.yearly",
             "com.floatingaxeheadministries.apilog.hobby.monthly",
             "com.floatingaxeheadministries.apilog.hobby.yearly":
            return .hobby
        // ... other plans
        default:
            return nil
        }
    }
    
    static func interval(for productId: String) -> BillingInterval? {
        if productId.contains("monthly") { return .monthly }
        if productId.contains("yearly") { return .yearly }
        return nil
    }
}
```

**How It Works:**

1. **On App Launch:** `verifySubscriptionStatus()` queries `Marketplace.fetchEntitlements()`
2. **Active Subscription Found:** Updates Firestore with current plan/interval
3. **No Active Subscriptions:** Calls `downgradeToFreePlan()` which sets:
   - `plan: "free"`
   - `subscriptionStatus: "free"`
   - `billingInterval: null`
   - Limits back to free tier (3 hives, 1 user)

**Cross-Platform Sync:**

Since Firestore is the source of truth, the web app immediately sees the updated subscription status. No additional work needed for web synchronization.

**Testing Expiry:**

For testing, you can:
1. Cancel subscription in Play Store/App Store
2. Relaunch app → should downgrade to free
3. Or manually set `subscriptionStatus: "free"` in Firestore for testing

**Key Takeaways:**

- Call `verifySubscriptionStatus()` on every app launch for accuracy
- Use `NSNull()` to clear fields in Firestore (not `nil` which may not update)
- `fetchEntitlements()` returns empty array when subscription expired/cancelled
- Firestore updates sync immediately to web app
- Both iOS and Android use same `IAPService` code — Skip Marketplace abstracts platform differences

---

### Pattern: Cross-Platform Billing (Stripe + IAP)

**Context:** Support both web subscriptions (Stripe) and mobile in-app purchases (IAP via Skip Marketplace) with a single Firestore workspace. Users should not be double-billed or confused about which system manages their subscription.

**Problem:**
- User subscribes on web via Stripe → mobile app shows "Upgrade" button (wrong!)
- User subscribes via IAP → can't manage on web
- No way to distinguish billing source in Firestore

**Solution:** Track `billingSource` in Firestore (`stripe`, `iap`, `free`) and show different UI accordingly.

**Implementation:**

1. **SubscriptionService.swift** — Track billing source:
```swift
public var billingSource: BillingSource = .unknown

public enum BillingSource: String {
    case stripe = "stripe"
    case iap = "iap"
    case free = "free"
    case unknown = "unknown"
    
    var displayName: String {
        switch self {
        case .stripe: return "Stripe"
        case .iap: return "In-App Purchase"
        case .free: return "Free Plan"
        case .unknown: return "Unknown"
        }
    }
}

// In fetchWorkspaceData():
if let sourceStr = data["billingSource"] as? String,
   let source = BillingSource(rawValue: sourceStr) {
    billingSource = source
}
```

2. **IAPService.swift** — Mark purchases as IAP:
```swift
func updateFirestorePlan(plan: SubscriptionPlan, interval: BillingInterval?) async {
    var data: [String: Any] = [
        "plan": plan.rawValue,
        "subscriptionStatus": plan == .free ? "free" : "active",
        "billingSource": "iap",  // ← Mark as In-App Purchase
        "hiveLimit": config.hiveLimit,
        // ...
    ]
    // ... update Firestore
}
```

3. **BillingSettingsView.swift** — Show appropriate UI:
```swift
Section("Actions") {
    // Stripe subscribers - link to web portal
    if service.billingSource == .stripe && service.subscription.isPaid {
        Button {
            Task {
                if let url = await service.createPortalURL() {
                    #if os(Android)
                    let ctx = ProcessInfo.processInfo.androidContext
                    let intent = android.content.Intent(android.content.Intent.ACTION_VIEW)
                    intent.setData(android.net.Uri.parse(url))
                    ctx.startActivity(intent)
                    #else
                    await UIApplication.shared.open(URL(string: url)!)
                    #endif
                }
            }
        } label: {
            Label("Manage on Web", systemImage: "creditcard.fill")
        }
        
        Text("Your subscription is managed through Stripe. Use the web portal to update or cancel.")
            .font(.caption)
    }
    
    // Free or IAP subscribers - show upgrade button
    if service.billingSource != .stripe {
        Button { showPlanPicker = true } label: {
            Label(service.subscription.isPaid ? "Change Plan" : "Upgrade", 
                  systemImage: "arrow.up.circle.fill")
        }
    }
}
```

**Web App Integration:**

Your web app should also set `billingSource: "stripe"` when creating subscriptions:
```javascript
// Firebase Function or web app
await db.collection('workspaces').doc(workspaceId).update({
  plan: 'hobby',
  subscriptionStatus: 'active',
  billingSource: 'stripe',  // ← Important!
  billingInterval: 'monthly',
  // ...
});
```

**Key Points:**

- **Stripe subscribers** on mobile see "Manage on Web" button that opens Stripe Customer Portal
- **IAP subscribers** use native upgrade/change plan flow
- **Free users** see "Upgrade" button that shows IAP options
- Firestore `billingSource` field prevents double-billing confusion
- Web app and mobile app sync automatically via Firestore

---

### Pattern: Firestore Document Decoding (`doc.data(as:)` Unavailable)

**Problem:** Skip's Firestore SDK does not include `FirebaseFirestoreSwift`. The `doc.data(as: T.self)` method for automatic Codable decoding is unavailable.

**Fix:** Use `doc.data()` to get `[String: Any]` and decode manually:

```swift
// ❌ ERROR - Not available in Skip
let item = try? doc.data(as: MyModel.self)

// ✅ CORRECT - Decode manually from [String: Any]
let d = doc.data()
var dWithId = d
dWithId["id"] = doc.documentID
guard let id = dWithId["id"] as? String else { return nil }

// Decode date with platform branch:
let date: Date
if let ts = dWithId["date"] as? Double {
    date = Date(timeIntervalSince1970: ts)
} else {
    #if os(Android)
    return nil
    #else
    guard let ts = dWithId["date"] as? Timestamp else { return nil }
    date = ts.dateValue()
    #endif
}
return MyModel(id: id, date: date)
```

---

### Pattern: Layout — `.safeAreaInset` Unavailable

**Problem:** `.safeAreaInset(edge:content:)` is unavailable in skip-fuse-ui.

**Fix:** Use `.overlay(alignment:)` instead:

```swift
// ❌ ERROR - Not available
.safeAreaInset(edge: .bottom) {
    bottomBar
}

// ✅ CORRECT - Use overlay
.overlay(alignment: .bottom) {
    bottomBar
}
```

---

### Pattern: Firebase Authentication

**Problem:** Implementing email/password authentication with Firebase Auth in a cross-platform Skip app requires platform-conditional imports and handling differences in error domains between iOS and Android.

**Solution:** Use platform-conditional imports with `@preconcurrency`, implement an `@MainActor @Observable` ViewModel, and handle auth state changes via `Auth.auth().addStateDidChangeListener`.

**Dependencies:**

Already included in `skip-firebase` package:
- `SkipFirebaseAuth` (Android)
- `FirebaseAuth` (iOS)

**Implementation:**

```swift
// 1. AuthViewModel.swift — Handles authentication logic
import Foundation
import SkipFuseUI
#if os(Android)
@preconcurrency import SkipFirebaseAuth
@preconcurrency import SkipFirebaseFirestore
#else
@preconcurrency import FirebaseAuth
@preconcurrency import FirebaseFirestore
#endif

@MainActor
@Observable
class AuthViewModel {
    var email = ""
    var password = ""
    var errorMessage: String?

    func signIn() async -> Bool {
        errorMessage = nil
        do {
            let result = try await Auth.auth().signIn(withEmail: email, password: password)
            // Enforce email verification
            if !result.user.isEmailVerified {
                let sent = await sendVerification()
                try? Auth.auth().signOut()
                errorMessage = sent ? "Please verify your email." : "Couldn't send verification email."
                return false
            }
            return true
        } catch {
            errorMessage = friendlyAuthErrorMessage(from: error)
            return false
        }
    }

    func signUp() async -> Bool {
        errorMessage = nil
        do {
            _ = try await Auth.auth().createUser(withEmail: email, password: password)
            let sent = await sendVerification()
            try? Auth.auth().signOut()
            errorMessage = sent ? "Verification email sent. Please verify, then log in." : "Couldn't send verification email."
            return true
        } catch {
            errorMessage = "Sign up failed: \(friendlyAuthErrorMessage(from: error))"
            return false
        }
    }

    private func sendVerification() async -> Bool {
        #if os(Android)
        do {
            if let user = Auth.auth().currentUser {
                try await user.sendEmailVerification()
                return true
            }
            return false
        } catch {
            return false
        }
        #else
        Auth.auth().currentUser?.sendEmailVerification { _ in }
        return true
        #endif
    }

    private func friendlyAuthErrorMessage(from error: Error) -> String {
        let nsError = error as NSError
        let errorDescription = error.localizedDescription.lowercased()
        
        // Handle Android JNI errors
        if errorDescription.contains("operation") && errorDescription.contains("cannot be completed") {
            return "Username or password does not match. Please try again."
        }
        
        // Firebase Auth error codes
        if nsError.domain == "FIRAuthErrorDomain" {
            switch nsError.code {
            case 17009, 17011, 17020: // wrongPassword, userNotFound, invalidCredential
                return "Username or password does not match. Please try again."
            default:
                break
            }
        }
        return error.localizedDescription
    }
}
```

```swift
// 2. Root View with Auth State Management
import SwiftUI
import SkipFuseUI
#if os(Android)
@preconcurrency import SkipFirebaseAuth
#else
@preconcurrency import FirebaseAuth
#endif

enum AuthState {
    case undetermined
    case signedOut
    case signedIn(User)
    case guest
}

struct RootView: View {
    @State var authState: AuthState = .undetermined
    @State var authViewModel = AuthViewModel()
    @State var hasAuthListener = false

    var body: some View {
        VStack {
            switch authState {
            case .undetermined:
                ProgressView("Loading...")
            case .signedOut:
                LoginView(viewModel: authViewModel, onLoginSuccess: checkAuthState)
            case .signedIn(_), .guest:
                MainContentView(onLogout: handleLogout)
            }
        }
        .onAppear {
            setupAuthListener()
        }
    }

    private func setupAuthListener() {
        if !hasAuthListener {
            _ = Auth.auth().addStateDidChangeListener { (auth: Auth, user: User?) in
                Task { @MainActor in
                    if let user = user {
                        // Block unverified users
                        if !user.isAnonymous && !user.isEmailVerified {
                            self.authState = .signedOut
                            return
                        }
                        self.authState = user.isAnonymous ? .guest : .signedIn(user)
                    } else {
                        self.authState = .signedOut
                    }
                }
            }
            hasAuthListener = true
        }
    }

    func checkAuthState() {
        if let user = Auth.auth().currentUser {
            if !user.isAnonymous && !user.isEmailVerified {
                try? Auth.auth().signOut()
                self.authState = .signedOut
                return
            }
            self.authState = .signedIn(user)
        } else {
            self.authState = .signedOut
        }
    }

    private func handleLogout() {
        try? Auth.auth().signOut()
        authState = .signedOut
    }
}
```

**Key Takeaways:**

- **Platform-conditional imports** with `@preconcurrency` are required for Firebase Auth
- **Email verification** must be enforced manually — Firebase doesn't block unverified users by default
- **Auth state listener** (`addStateDidChangeListener`) is the reliable way to observe login/logout across platforms
- **Error handling** must account for different error domains: `FIRAuthErrorDomain` vs `AuthErrorDomain`
- **Guest mode** uses anonymous Firebase auth (`user.isAnonymous`) for users who skip login
- **Profile creation** on first sign-in should use `FieldValue.serverTimestamp()` for dates
- **Skip incompatibilities to avoid:**
  - Don't use `NSNull()` in Firestore data — use `FieldValue.delete()` instead
  - Mark `[String: Any]` dictionaries as `nonisolated(unsafe)` for Firestore calls in Swift 6
  - Remove `private` from `@State` properties in all views

---

### Login Screen Implementation

**Problem:** A cross-platform login screen with email/password fields, a password visibility toggle, and action buttons can silently break on Android due to several Skip incompatibilities that are hard to diagnose — buttons become completely unresponsive and the toggle stops working.

**Root Causes Discovered:**
- **`Toggle` blocks all gestures below it** — Skip's `Toggle` component intercepts touch events on Android, making every button below it in the same `VStack` unresponsive
- **`.shadow()` creates invisible blocking layers** — `.shadow(color:radius:x:y:)` generates a hit-testing layer on Android that absorbs button taps
- **Duplicate `import SkipFuseUI`** — causes unexpected view behavior
- **`.overlay` on `TextField`/`SecureField` blocks input** — putting the eye toggle button in an `.overlay` on the text field blocks typing; use `HStack` layout instead for eye icon next to field, OR use the Button-in-overlay pattern with `RoundedBorderTextFieldStyle()` (the Coffee House pattern)
- **`@FocusState` crashes Android** — never use `@FocusState` or `.focused()` in any view

**Working `PasswordField` Pattern (matches Coffee House customer app):**
```swift
import Foundation
import SkipFuseUI

struct PasswordField: View {
    var placeholder: String
    @Binding var text: String
    @State var isVisible: Bool = false

    var body: some View {
        Group {
            if isVisible {
                TextField(placeholder, text: $text)
                    .textContentType(.password)
                    .autocorrectionDisabled(true)
                    .textInputAutocapitalization(.never)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding(.trailing, 36)
                    .overlay(alignment: .trailing) {
                        Button(action: { isVisible.toggle() }) {
                            Image(systemName: "eye.slash")
                                .foregroundColor(.secondary)
                        }
                        .padding(.leading, 10)
                        .padding(.trailing, 12)
                    }
            } else {
                SecureField(placeholder, text: $text)
                    .textContentType(.password)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .padding(.trailing, 36)
                    .overlay(alignment: .trailing) {
                        Button(action: { isVisible.toggle() }) {
                            Image(systemName: "eye")
                                .foregroundColor(.secondary)
                        }
                        .padding(.leading, 10)
                        .padding(.trailing, 12)
                    }
            }
        }
    }
}
```

**Working Login View Rules:**
- Use `RoundedBorderTextFieldStyle()` on all text fields — built-in style that works cross-platform
- Do NOT use `Toggle` — use `@AppStorage` directly without a visible toggle, or replace with a `Button`
- Do NOT use `.shadow()` on card containers — use `.background(Color.white).cornerRadius()` only
- Do NOT use `@FocusState` — remove entirely
- Do NOT use `@State private` — use `@State var`
- Use `@AppStorage` for persisting email across sessions
- Use `Group { if isVisible { ... } else { ... } }` pattern to switch between `TextField` and `SecureField`

**Login View Skeleton:**
```swift
struct LoginView: View {
    @Bindable var viewModel: AuthViewModel
    var onLoginSuccess: () -> Void

    @State var showConfirmSheet = false
    @AppStorage("app_lastEmail") var lastEmail = ""

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 24) {
                    // Branding
                    VStack(spacing: 16) { ... }
                        .padding(.top, 32)

                    // Form — NO .shadow(), NO Toggle
                    VStack(spacing: 16) {
                        TextField("Email", text: $viewModel.email)
                            .textInputAutocapitalization(.never)
                            .keyboardType(.emailAddress)
                            .textFieldStyle(RoundedBorderTextFieldStyle())
                            .autocorrectionDisabled(true)

                        PasswordField(placeholder: "Password", text: $viewModel.password)

                        if let errorMessage = viewModel.errorMessage {
                            Text(errorMessage)
                                .foregroundColor(.red)
                                .font(.caption)
                        }

                        Button("Sign In") {
                            Task {
                                if await viewModel.signIn() { onLoginSuccess() }
                            }
                        }
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color.accentColor)
                        .foregroundColor(.white)
                        .cornerRadius(14)
                    }
                    .padding()
                    .background(Color.white)
                    .cornerRadius(16)
                    // ⚠️ NO .shadow() here
                    .padding(.horizontal)
                }
            }
            #if os(Android)
            .ignoresSafeArea(.keyboard, edges: .bottom)
            #endif
            .onAppear {
                if viewModel.email.isEmpty { viewModel.email = lastEmail }
            }
            .onChange(of: viewModel.email) { _, newValue in
                lastEmail = newValue
            }
        }
    }
}
```

**Key Takeaways:**
- `Toggle` is a gesture killer on Android — avoid in forms with buttons
- `.shadow()` blocks taps — never use on interactive card containers
- `RoundedBorderTextFieldStyle()` is the reliable cross-platform text field style
- `Button(action:)` inside `.overlay(alignment: .trailing)` works for the eye toggle when combined with `RoundedBorderTextFieldStyle()`
- Always check: duplicate imports, `@State private`, `@FocusState`

---

### Template — Adding New Entries

When a new cross-platform pattern is discovered, add it using this template:

```markdown
### Pattern: <Short Title>

**Problem:** <What you need to do cross-platform and why it's non-trivial.>

**Solution:** <Brief description of the approach.>

**Dependencies:**
<skip.yml and/or build.gradle.kts entries>

**Implementation:**
<Code showing the pattern>

**Key Takeaways:**
- <Bullet points with lessons learned>
```

---

### Pattern: Stripe Payment Flow for Android and iOS

**Context:** Church Compass member app — donation payment via Stripe PaymentSheet. Took multiple sessions to get correct. Do not revisit.

---

#### Architecture Overview

| Platform | Payment trigger | Payment sheet presenter |
|----------|----------------|------------------------|
| iOS | `PaymentSheetService.shared.present(with:)` | Native `StripePaymentSheet` via `iOSPaymentSheetPresenter` |
| Android | `SimpleStripePaymentButton` from `SkipStripe` | Android Stripe SDK via registered `ActivityResultLauncher` |

**Critical rule:** `PaymentSheetService.shared.present(with:)` has NO Android implementation. The `#else` branch returns `.failed("Stripe PaymentSheet is not available on this platform yet.")`. Do NOT call it on Android. Android MUST use `SimpleStripePaymentButton`.

---

#### Why Android Requires SimpleStripePaymentButton

The Android Stripe SDK launches the PaymentSheet as an Activity, which requires an `ActivityResultLauncher` registered at composable creation time. It cannot be triggered programmatically from a coroutine/Task. `SimpleStripePaymentButton` (from `SkipStripe`) is a SwiftUI view that wraps this launcher and must be present in the view hierarchy and tapped by the user to launch the sheet.

---

#### Ephemeral Key API Version (CRITICAL)

The Android Stripe SDK requires the ephemeral key to be created with API version **`2025-07-30.basil`** (or newer basil versions). Using an older version (e.g., `2024-06-20`) causes the PaymentSheet to dismiss immediately with no error shown to the user.

- Use `buildAndroidPaymentConfiguration` from `StripePaymentService` — it calls `createStripeEphemeralKeyV1` which accepts an `apiVersion` parameter.
- Do NOT use the ephemeral key returned by `createDonationPaymentIntentV1` directly on Android — it uses an older API version.
- In `firebase-functions/index.js`, the default `apiVersion` in `createStripeEphemeralKeyV1` and the Stripe client initialization must both use `'2025-07-30.basil'`.

```js
// firebase-functions/index.js
const stripe = STRIPE_SECRET_KEY
  ? new Stripe(STRIPE_SECRET_KEY, { apiVersion: '2025-07-30.basil' })
  : null;

// Inside createStripeEphemeralKeyV1:
const apiVersion = data.apiVersion || '2025-07-30.basil';
```

---

#### Complete Android + iOS Payment Flow

**iOS flow:**
1. User taps **Give** → `isSubmitting = true` → "Processing…" spinner
2. `viewModel.submit()` creates a PaymentIntent → sets `viewModel.showPaymentSheet = true`
3. `.onChange(of: viewModel.showPaymentSheet)` fires (iOS only, guarded with `#if os(iOS)`) → sets `pendingPaymentData` + `showRedirectPaymentAlert = true`
4. "Payment Redirect" alert appears → user taps **Continue to Payment**
5. `presentPaymentSheet(...)` → `PaymentSheetService.shared.present(with:)` → Stripe sheet opens
6. Result handled: `.completed` → success, `.canceled` → dismiss, `.failed` → error

**Android flow:**
1. User taps **Give** → `isSubmitting = true` → "Processing…" spinner
2. `viewModel.submit()` creates a PaymentIntent → sets `viewModel.showPaymentSheet = true`
3. Android Task block calls `StripePaymentService.shared.buildAndroidPaymentConfiguration(...)` → fetches a fresh ephemeral key via `createStripeEphemeralKeyV1` with correct API version
4. `androidPaymentConfiguration` set → `SimpleStripePaymentButton("Continue to Payment")` replaces the Give button
5. User taps **Continue to Payment** (`SimpleStripePaymentButton`) → Stripe sheet opens directly
6. Result handled in `handleAndroidPaymentResult(_:)`

**No alert on Android.** The `onChange(of: viewModel.showPaymentSheet)` block MUST be wrapped in `#if os(iOS)` — if not, Android will also trigger the alert + call `presentPaymentSheet` which will always fail.

---

#### DonationView.swift Key Structure

```swift
// Imports
#if os(Android)
@preconcurrency import SkipFirebaseAuth
@preconcurrency import SkipFirebaseFirestore
import SkipStripe              // Required for SimpleStripePaymentButton + StripePaymentResult
#else
@preconcurrency import FirebaseAuth
@preconcurrency import FirebaseFirestore
#endif

struct DonationView: View {
    @State var isSubmitting = false
    @State var isProcessingPayment = false
    @State var showRedirectPaymentAlert = false
    @State var pendingPaymentData: (publishableKey: String, clientSecret: String, customerId: String, ephemeralKeySecret: String)? = nil
#if os(Android)
    @State var androidPaymentConfiguration: StripePaymentConfiguration? = nil
#endif

    // Button section — OUTSIDE ScrollView (required for iOS PaymentSheet)
    VStack(spacing: 12) {
#if os(Android)
        if let configuration = androidPaymentConfiguration {
            SimpleStripePaymentButton(
                configuration: configuration,
                buttonText: "Continue to Payment",
                completion: handleAndroidPaymentResult
            )
        } else {
            giveButton
        }
#else
        giveButton
#endif
    }

    // onChange — iOS ONLY. If this fires on Android it triggers the wrong path.
    .onChange(of: viewModel.showPaymentSheet) { _, newValue in
        guard newValue else { return }
#if os(iOS)
        guard let clientSecret = viewModel.clientSecret,
              let publishableKey = viewModel.publishableKey,
              let customerId = viewModel.customerId,
              let ephemeralKeySecret = viewModel.ephemeralKeySecret else { return }
        pendingPaymentData = (publishableKey, clientSecret, customerId, ephemeralKeySecret)
        showRedirectPaymentAlert = true
#endif
    }

    // Alert — only triggered on iOS (pendingPaymentData set iOS-only above)
    .alert("Payment Redirect", isPresented: $showRedirectPaymentAlert) {
        Button("Continue to Payment") {
            if let data = pendingPaymentData {
                Task {
                    await presentPaymentSheet(
                        publishableKey: data.publishableKey,
                        clientSecret: data.clientSecret,
                        customerId: data.customerId,
                        ephemeralKeySecret: data.ephemeralKeySecret
                    )
                }
            }
            pendingPaymentData = nil
        }
        Button("Cancel", role: .cancel) {
            pendingPaymentData = nil
            viewModel.showPaymentSheet = false
        }
    } message: {
        Text("You may be redirected to another app (like Cash App) to complete your payment.")
    }
}

// Give button Task
Button {
    Task {
        isSubmitting = true
        await viewModel.submit()
#if os(Android)
        if viewModel.showPaymentSheet,
           let customerId = viewModel.customerId,
           let clientSecret = viewModel.clientSecret {
            do {
                let configuration = try await StripePaymentService.shared.buildAndroidPaymentConfiguration(
                    stripeCustomerId: customerId,
                    shopId: viewModel.church.id,
                    amountCents: 0,
                    merchantDisplayName: viewModel.church.churchName,
                    allowsDelayedPaymentMethods: true,
                    existingPaymentIntentClientSecret: clientSecret
                )
                androidPaymentConfiguration = configuration
            } catch {
                viewModel.state = .error(message: error.localizedDescription)
            }
        }
#endif
        isSubmitting = false
    }
}

// Android result handler
#if os(Android)
func handleAndroidPaymentResult(_ result: StripePaymentResult) {
    Task { @MainActor in
        switch result {
        case .completed:
            androidPaymentConfiguration = nil
            viewModel.state = .success(donationId: viewModel.donationId ?? "completed")
            viewModel.showPaymentSheet = false
        case .canceled:
            androidPaymentConfiguration = nil
            viewModel.showPaymentSheet = false
        case .failed(let error):
            androidPaymentConfiguration = nil
            viewModel.state = .error(message: error.localizedDescription)
        }
    }
}
#endif

// iOS presentPaymentSheet — do NOT call on Android
private func presentPaymentSheet(
    publishableKey: String,
    clientSecret: String,
    customerId: String,
    ephemeralKeySecret: String
) async {
    isProcessingPayment = true
    defer { isProcessingPayment = false }
    let customer = PaymentSheetInitData.Customer(id: customerId, ephemeralKey: ephemeralKeySecret)
    let data = PaymentSheetInitData(
        mode: .paymentIntent(clientSecret: clientSecret),
        publishableKey: publishableKey,
        merchantDisplayName: viewModel.church.churchName,
        customer: customer,
        allowsDelayedPaymentMethods: false
    )
    do {
        let result = try await PaymentSheetService.shared.present(with: data)
        switch result {
        case .completed:
            viewModel.state = .success(donationId: viewModel.donationId ?? "")
            viewModel.showPaymentSheet = false
        case .canceled:
            viewModel.showPaymentSheet = false
        case .failed(let error):
            viewModel.state = .error(message: error)
        }
    } catch {
        viewModel.state = .error(message: error.localizedDescription)
    }
}
```

---

#### Stripe Webhook Setup

- Webhook handler function: `stripeWebhookV2` in `packages/firebase-functions/index.js`
- Endpoint URL: `https://us-central1-church-compass-platform.cloudfunctions.net/stripeWebhookV2`
- Events to subscribe: `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_succeeded`
- Payload type: **Snapshot** (handler uses `event.data.object` directly)
- Account type: **Your account** (not Connected accounts)
- After creating the endpoint, copy the signing secret → update `STRIPE_WEBHOOK_SECRET` in `packages/firebase-functions/.env` → redeploy: `firebase deploy --only functions`
- The webhook handles **church platform subscriptions only**, NOT donation payments. Donation payment completion is tracked in-app via `PaymentSheetService` result.
- If multiple Stripe apps share the same account (e.g., Coffee House + Church Compass), create **separate webhook endpoints** per app — do not share or replace each other's endpoints.

---

#### Logger Usage for Stripe Debugging

Use `logger.info` / `logger.error` (never `print`) for Stripe flow tracing:

```swift
logger.info("[DonationView] Submit tapped")
logger.info("[DonationView] Android: configuration ready")
logger.info("[DonationView] Continue to Payment tapped")
logger.info("[DonationView] presentPaymentSheet entered")
logger.error("[DonationView] buildAndroidPaymentConfiguration failed: \(error.localizedDescription)")
```

Pull logs on device:
```bash
adb logcat | grep "ChurchCompass"
```

---

#### Key Takeaways

- **Never call `PaymentSheetService.shared.present` on Android** — it always fails with "not available on this platform"
- **Always guard `onChange` iOS-only paths with `#if os(iOS)`** — `onChange` can fire on Android and trigger the wrong code path
- **Use `buildAndroidPaymentConfiguration`** (not the ephemeral key from the donation function) — correct API version is critical
- **`SimpleStripePaymentButton` must be in the view hierarchy before the user taps** — it registers an Android `ActivityResultLauncher` at composition time
- **`PaymentSheetOutcome` does not conform to `CustomStringConvertible`** — use `String(describing: result)` in logger calls
- **Donation payments are NOT tracked via webhook** — only subscription/billing events go through the webhook

---

### Pattern: `@FocusState` Crashes Android

**Problem:** `@FocusState` + `.focused()` crashes Android with "offset(0) is out of bounds [0, 0)".

**Fix:** Remove `@FocusState` and `.focused()` entirely. Do not use focus management.

```swift
// ❌ CRASH on Android
@FocusState var focused: Field?
TextField("Email", text: $email).focused($focused, equals: .email)

// ✅ Correct
TextField("Email", text: $email)
```

---

### Pattern: `Color(.systemGray*)` and System Background Colors Not Available

**Problem:** `Color(.systemGray6)`, `Color(.systemBackground)`, etc. are UIKit-only — "reference to member cannot be resolved".

**Fix:** Use `Color.gray.opacity()` or `Color.white` instead.

```swift
// ❌
.background(Color(.systemGray6))
.background(Color(.systemBackground))

// ✅
.background(Color.gray.opacity(0.1))
.background(Color.white)
```

---

### Pattern: `.autocapitalization` Not Available

**Problem:** `.autocapitalization(.none)` is a UIKit API — "value of type 'some View' has no member 'autocapitalization'".

**Fix:** Use `.textInputAutocapitalization(.never)`.

```swift
// ❌
TextField("Email", text: $email).autocapitalization(.none)

// ✅
TextField("Email", text: $email).textInputAutocapitalization(.never)
```

---

### Pattern: Lazy Stacks and Grids Not Available

**Problem:** `LazyVStack`, `LazyHStack`, `LazyVGrid`, `LazyHGrid` are not bridged to Android.

**Fix:** Replace with standard `VStack`/`HStack`. For grids, chunk into `VStack` of `HStack` rows.

```swift
// ❌
LazyVGrid(columns: [...]) { ... }

// ✅
VStack(spacing: 10) {
    HStack(spacing: 10) { /* row 1 */ }
    HStack(spacing: 10) { /* row 2 */ }
}
```

---

### Pattern: NavigationStack Large Title Padding / Double NavigationStack

**Problem 1:** `.navigationBarTitleDisplayMode(.large)` causes ~15% wasted whitespace on Android.

**Problem 2:** Wrapping a tab's content in `NavigationStack` in both `ContentView` AND a child view causes extreme padding on Android.

**Fix:** Use `.inline` on Android. Only ONE `NavigationStack` per tab.

```swift
// Large title — platform branch
#if os(Android)
.navigationBarTitleDisplayMode(.inline)
#else
.navigationBarTitleDisplayMode(.large)
#endif

// ❌ Double wrap — extreme padding
TabView { NavigationStack { MyView() } }
struct MyView: View { var body: some View { NavigationStack { ... } } }

// ✅ Single wrap in ContentView; child uses .navigationTitle directly
TabView { NavigationStack { MyView() } }
struct MyView: View { var body: some View { VStack { ... }.navigationTitle("Title") } }
```

**Bold inline title on Android:**
```swift
#if os(Android)
.navigationBarTitleDisplayMode(.inline)
.toolbar {
    ToolbarItem(placement: .principal) {
        Text("Title").font(.title).fontWeight(.bold)
    }
}
#else
.navigationTitle("Title")
.navigationBarTitleDisplayMode(.large)
#endif
```

---

### Pattern: `.topBarTrailing` and `navigationBarTitleDisplayMode` Unavailable on macOS

**Problem:** `.topBarTrailing` and `.navigationBarTitleDisplayMode` are unavailable on macOS, causing build errors.

**Fix:** Use `.automatic`; wrap `navigationBarTitleDisplayMode` in `#if os(iOS)`.

```swift
// ❌
ToolbarItem(placement: .topBarTrailing) { ... }
.navigationBarTitleDisplayMode(.inline)

// ✅
ToolbarItem(placement: .automatic) { ... }
#if os(iOS)
.navigationBarTitleDisplayMode(.inline)
#endif
```

---

### Pattern: Button Closure Properties Require `@MainActor @Sendable`

**Problem:** Swift 6 warns "converting non-Sendable function value to '@MainActor @Sendable () -> Void'" when a View struct stores a plain `() -> Void` closure passed to a `Button`.

**Fix:** Declare the closure as `@MainActor @Sendable () -> Void`.

```swift
// ❌
struct MyRow: View {
    let onTap: () -> Void
}

// ✅
struct MyRow: View {
    let onTap: @MainActor @Sendable () -> Void
}
```

---

### Pattern: `@Observable` Requires `import Observation` Before `import SkipFuseUI`

**Problem:** `@Observable` causes "unknown attribute 'Observable'" if `import Observation` is missing or placed after `import SkipFuseUI`.

**Fix:** Always import `Observation` before `SkipFuseUI`.

```swift
// ❌
import Foundation
import SkipFuseUI
@Observable class MyViewModel { ... }

// ✅
import Foundation
import Observation
import SkipFuseUI
@Observable class MyViewModel { ... }
```

---

### Pattern: `.fixedSize` Not Available on Text

**Problem:** `.fixedSize(horizontal:vertical:)` is unavailable on `Text` in Skip.

**Fix:** Use `.lineLimit(nil)`.

```swift
// ❌
Text(description).fixedSize(horizontal: false, vertical: true)

// ✅
Text(description).lineLimit(nil)
```

---

### Pattern: Compiler Type-Checking Timeouts

**Problem:** "The compiler is unable to type-check this expression in reasonable time" — long `View.body` properties with many modifiers overwhelm the Swift type checker.

**Fix:** Break up into `@ViewBuilder` computed properties and `private func` helpers.

```swift
// ❌ — too much in one body
var body: some View {
    Form { /* 50+ lines */ }
    .onChange(of: a) { ... }
    // ... 15 more onChange
}

// ✅ — decompose
var body: some View { formContent }

@ViewBuilder var formContent: some View {
    baseForm
        .onChange(of: a) { _, _ in save() }
        .onChange(of: b) { _, _ in save() }
}
```

**Rule of thumb:** >8 modifiers or >30 lines of view DSL in one property → extract further.

---

### Pattern: `@escaping` on Closures

**Problem:** `@escaping` is only valid in function parameter position. Skip does not support it on stored properties (Kotlin lambdas are naturally escaping).

```swift
// ❌ On stored property
struct MyView: View {
    let onSelect: @escaping (Item?) -> Void
}

// ✅ On stored property — omit @escaping
struct MyView: View {
    let onSelect: (Item?) -> Void
}

// ✅ On function parameter — @escaping is correct
func setup(onSelect: @escaping (Item?) -> Void)
```

---

### Pattern: `navigationDestination` Must Be Inside `NavigationStack` Content

**Problem:** `.navigationDestination(for:destination:)` placed on the `NavigationStack` itself is ignored.

**Fix:** Place inside the content view inside the `NavigationStack`.

```swift
// ❌ Wrong — destination is ignored
NavigationStack(path: $path) { ListView() }
    .navigationDestination(for: Item.self) { ... }

// ✅ Correct
NavigationStack(path: $path) {
    ListView()
        .navigationDestination(for: Item.self) { item in DetailView(item: item) }
}
```

---

### Pattern: Shadow on Interactive Cards Blocks Taps on Android

**Problem:** `.shadow()` on a container that holds `Button` or `NavigationLink` blocks all gesture handling on Android — buttons become untappable.

**Fix:** Remove `.shadow()` from interactive card containers. Use border/overlay instead.

```swift
// ❌ — shadow blocks taps inside card on Android
VStack { Button { ... } label: { ... } }
    .background(Color.white).cornerRadius(12)
    .shadow(color: .black.opacity(0.1), radius: 4)

// ✅ — use overlay border instead
VStack { Button { ... } label: { ... } }
    .background(Color.white).cornerRadius(12)
    .overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2)))
```

---

### Pattern: Toggle in Forms Blocks Gestures Below on Android

**Problem:** A `Toggle` in a `Form` or `VStack` blocks all gesture handling for controls below it on Android.

**Fix:** Place `Toggle` at the bottom of the form, or restructure to avoid it above buttons.

```swift
// ❌ — Sign In button unreachable on Android
Form {
    Toggle("Remember me", isOn: $rememberMe)
    Button("Sign In") { signIn() }
}

// ✅ — Toggle last
Form {
    Button("Sign In") { signIn() }
    Toggle("Remember me", isOn: $rememberMe)
}
```

---

### Pattern: Firestore Timestamps Decode as `Double` on Android

**Problem:** On Android, Firestore timestamps arrive as `Double` (epoch milliseconds), not `Timestamp`. Direct cast to `Timestamp` returns `nil`, silently dropping documents.

**Fix:** Handle both types:

```swift
// ❌ — fails silently on Android
guard let ts = data["startDate"] as? Timestamp else { return nil }
self.startDate = ts.dateValue()

// ✅ — handle both
func dateFromFirestore(_ value: Any?) -> Date? {
    if let ts = value as? Timestamp { return ts.dateValue() }
    if let ms = value as? Double { return Date(timeIntervalSince1970: ms / 1000.0) }
    return nil
}
self.startDate = dateFromFirestore(data["startDate"]) ?? Date()
```

---

### Pattern: `@Observable` UI Refresh Issues

**Problem:** State changes in `@Observable` classes don't always trigger immediate SwiftUI view updates in Skip. UI requires a second interaction to refresh.

**Fix:** Use a `refreshID` + `.id()` + `.onChange()` to force rebuild:

```swift
// ❌ — UI doesn't refresh immediately
struct MyView: View {
    @State var viewModel: MyViewModel
    var body: some View {
        VStack { if viewModel.showItem { ItemView() } }
    }
}

// ✅ — force rebuild on state change
struct MyView: View {
    @State var viewModel: MyViewModel
    @State var refreshID = UUID()
    var body: some View {
        VStack { if viewModel.showItem { ItemView() } }
            .id(refreshID)
            .onChange(of: viewModel.showItem) { _, _ in refreshID = UUID() }
    }
}
```

---

### Pattern: `import SwiftUI` Ban — Use `import SkipFuseUI`

**Problem:** `import SwiftUI` is iOS-only and causes build errors on Android in Skip projects.

**Fix:** Replace all `import SwiftUI` with `import SkipFuseUI`.

```swift
// ❌
import SwiftUI

// ✅
import SkipFuseUI
```

**Quick fix:**
```bash
sed -i '' 's/import SwiftUI/import SkipFuseUI/g' mobile-apps/member/Sources/ChurchCompass/*.swift
```

---

### Pattern: ScrollView Blocks Stripe PaymentSheet Presentation

**Problem:** A Stripe `PaymentSheet` button nested inside a `ScrollView` can fail to present on iOS due to gesture handler conflicts.

**Fix:** Move the payment button outside the `ScrollView` into a fixed bottom area.

```swift
// ❌ — may not present
ScrollView { VStack { SimpleStripePaymentButton(...) } }

// ✅ — button outside ScrollView
VStack(spacing: 0) {
    ScrollView { VStack { /* form content */ } }
    SimpleStripePaymentButton(...).padding()
}
```

---

### Pattern: Native Audio/Video Playback with SkipAV (iOS + Android)

**Problem:** `AVKit` and `AVFoundation` are iOS-only. Using them directly causes build failures on Android. Wrapping everything in `#if os(iOS)` means Android users get no native player.

**Solution:** Use **`SkipAV`** — a Skip package that wraps `AVKit`/`AVFoundation` on iOS and **ExoPlayer** on Android. The same `AVPlayer`, `AVPlayerItem`, `VideoPlayer`, and `AVQueuePlayer` APIs work cross-platform.

**Verified against:** `skipapp-showcase` and `skipapp-showcase-fuse` (May 2026) — API unchanged.

**Package.swift:**
```swift
dependencies: [
    .package(url: "https://source.skip.tools/skip-av.git", "0.6.2"..<"2.0.0"),
    // ...
],
targets: [
    .target(name: "MyTarget", dependencies: [
        .product(name: "SkipAV", package: "skip-av"),
        // ...
    ])
]
```

**Import pattern** (use `canImport`, not `os(iOS)`):
```swift
#if canImport(SkipAV)
import SkipAV
#else
import AVKit
import AVFoundation  // Fuse projects need this in the #else branch
#endif
```

> **Why `canImport` not `#if os(iOS)`?** On Android in Fuse mode, `SkipAV` IS importable — it provides the AVKit API surface backed by ExoPlayer. Using `#if os(iOS)` would exclude it on Android entirely.

**Cross-platform player — video:**
```swift
struct VideoPlayerView: View {
    @State var player: AVPlayer

    init(url: URL) {
        self.player = AVPlayer(playerItem: AVPlayerItem(url: url))
    }

    var body: some View {
        VideoPlayer(player: player)
            .onAppear { player.play() }
            .onDisappear { player.pause() }
    }
}
```

**Cross-platform player — looping with rate control:**
```swift
struct LoopingPlayerView: View {
    @State var player: AVQueuePlayer
    @State var playerLooper: AVPlayerLooper
    @State var rate = 1.0

    init(url: URL) {
        let item = AVPlayerItem(url: url)
        let qp = AVQueuePlayer(playerItem: item)
        self.player = qp
        self.playerLooper = AVPlayerLooper(player: qp, templateItem: item)
    }

    var body: some View {
        VStack {
            VideoPlayer(player: player)
                .onAppear { player.play(); player.rate = Float(rate) }
            Slider(value: $rate, in: 0.0...2.0, label: { Text("Speed") })
                .onChange(of: rate) { player.rate = Float($0) }
        }
    }
}
```

**iOS-only features — guard with `#if os(iOS)`:**
```swift
#if os(iOS)
import MediaPlayer
import CoreLocation

// Lock screen / Control Center
func setupNowPlaying() {
    MPNowPlayingInfoCenter.default().nowPlayingInfo = [
        MPMediaItemPropertyTitle: title,
        MPNowPlayingInfoPropertyPlaybackRate: 1.0,
    ]
}

// AirPlay picker button
struct AirPlayButton: UIViewRepresentable {
    func makeUIView(context: Context) -> AVRoutePickerView {
        let v = AVRoutePickerView()
        v.tintColor = .white
        return v
    }
    func updateUIView(_ v: AVRoutePickerView, context: Context) {}
}

// Location — SkipDevice.LocationProvider calls requestLocation() without
// checking authorizationStatus first, causing a main-thread warning.
// FIX: Use SkipKit.PermissionManager to request authorization BEFORE
// calling LocationProvider.fetchCurrentLocation().
// This works cross-platform (iOS + Android).
import SkipDevice
import SkipKit

@MainActor
class MyViewModel {
    var locationProvider = LocationProvider()

    func fetchLocation() async {
        // Step 1: Request permission via PermissionManager
        let result = await PermissionManager.requestLocationPermission(precise: true, always: false)
        guard result.isAuthorized ?? false else {
            // Permission denied — show settings prompt
            return
        }

        // Step 2: Now safe to fetch — authorization is already resolved
        do {
            let location = try await locationProvider.fetchCurrentLocation()
            // ... use location.latitude, location.longitude ...
        } catch {
            // Handle timeout or GPS error
        }
    }
}

// Sheet detents
.presentationDetents([.height(360)])
// .navigationBarTitleDisplayMode(.inline)
#endif
```

**Key rules:**
- `AVPlayer`, `VideoPlayer`, `AVPlayerItem`, `AVQueuePlayer`, `AVPlayerLooper`, `CMTime`, `player.rate` — **all work on both platforms** via SkipAV, no guards needed
- `MediaPlayer` (`MPNowPlayingInfoCenter`, `MPRemoteCommandCenter`) — iOS only, guard with `#if os(iOS)`
- `AVRoutePickerView` (AirPlay) — iOS only, guard with `#if os(iOS)` + `UIViewRepresentable`
- `SkipDevice.LocationProvider` — calls `requestLocation()` without checking `authorizationStatus`, causing main-thread warning. **Fix:** Use `SkipKit.PermissionManager.requestLocationPermission()` FIRST, then call `LocationProvider.fetchCurrentLocation()`. Works cross-platform (iOS + Android).
- `.presentationDetents` — iOS only, guard with `#if os(iOS)`
- `.navigationBarTitleDisplayMode(.inline)` — iOS only, guard with `#if os(iOS)`
- `item.observe(\.status)` KVO and `addPeriodicTimeObserver` — work cross-platform via SkipAV
- Supports local files, remote URLs (HTTP/HTTPS), and HLS streams (`.m3u8`) on both platforms

---

### Swift 6 Concurrency — @MainActor ViewModel Pattern (CRITICAL)

All `@Observable` ViewModels in the member app MUST follow this pattern:

```swift
@MainActor          // ← REQUIRED: prevents "Sending 'self' risks causing data races"
@Observable
final class MyViewModel {
    var items: [Item] = []
    private var listener: ListenerRegistration? = nil

    // ✅ NO deinit — deinit runs in nonisolated context
    func stopListening() {
        listener?.remove()
        listener = nil
    }

    // ✅ Always use try await — NEVER closure-based getDocuments
    func fetchItems() {
        Task {
            do {
                let snapshot = try await Firestore.firestore()
                    .collection("items")
                    .getDocuments()
                items = snapshot.documents.compactMap { Item(from: $0) }
            } catch { }
        }
    }
}

struct MyView: View {
    @State var viewModel = MyViewModel()

    var body: some View {
        List { ... }
        .searchable(text: $query, prompt: "Search")  // NO placement: parameter
        .listStyle(.plain)                            // NO .insetGrouped
        .onAppear { viewModel.fetchItems() }
        .onDisappear { viewModel.stopListening() }    // NOT deinit
    }
}
```

**Why `try await` for all Firestore queries:**
Skip's `SkipFirebaseFirestore` overloads `getDocuments` and `getDocument` with a `FirestoreSource` first parameter. Swift's closure-based overload resolves to the wrong signature, producing: "cannot convert value of type '_' to expected argument type 'FirestoreSource'". The `async/await` form has no ambiguity.

**Why no `DispatchGroup`:**
`DispatchGroup` is not available in Skip. Replace batch-fetch patterns with sequential `async/await`:

```swift
// ❌ DispatchGroup — unavailable in Skip
let group = DispatchGroup()
for id in ids { group.enter(); db.document(id).getDocument { defer { group.leave() }; ... } }
group.notify(queue: .main) { self?.items = results }

// ✅ Sequential async/await
Task {
    do {
        var results: [Item] = []
        for id in ids {
            let snap = try await db.document(id).getDocument()
            if let item = Item(from: snap) { results.append(item) }
        }
        items = results
    } catch { }
}
```

---

### SF Symbols Management (Android Requires Manual SVG Registration)

**Problem:** Android does not have the SF Symbols font. Any `systemImage:` in a `Label`, `Image(systemName:)`, or toolbar item that is not manually registered will render as blank/missing on Android.

**Solution:** Every `systemImage:` name must have a corresponding `.symbolset` entry in the module's asset catalog:

```
Sources/<Module>/Resources/Module.xcassets/{symbol-name}.symbolset/
├── Contents.json          ← References the SVG
└── {symbol-name}.svg      ← Exported from SF Symbols Mac app
```

**Contents.json template:**
```json
{
  "info" : { "author" : "xcode", "version" : 1 },
  "symbols" : [{ "filename" : "{symbol-name}.svg", "idiom" : "universal" }]
}
```

**How to export an SVG from SF Symbols app:**
1. Open SF Symbols app on Mac
2. Search for the symbol name
3. Select → File → Export Symbol → SVG
4. Place SVG in the corresponding `.symbolset` folder

**Project-specific tracking file:**
Create `docs/sf-symbols-tracker.md` in each project with:
- Full table of all `systemImage:` names used
- Status: ✅ SVG present / ⚠️ SVG needed
- Which source files use each symbol
- Section for "Symbols Needing SVGs" with export instructions

**Standing rule for code generation:**
When writing any new `systemImage:` code:
1. Check if `{name}.symbolset/` exists in `Module.xcassets`
2. If missing → create `Contents.json` automatically
3. Add row to `docs/sf-symbols-tracker.md` with `⚠️ SVG needed`
4. Inform user which SVGs need to be filled in

**Common symbols to pre-load in new projects:**
```swift
// Navigation & Actions
"arrow.left", "arrow.right", "xmark", "checkmark", "plus", "minus"
"gear", "ellipsis.circle", "magnifyingglass"

// User & Social
"person", "person.fill", "person.circle", "person.2", "person.3"
"envelope", "phone", "location", "mappin"

// Media & Content
"play.fill", "pause.fill", "calendar", "book.closed", "photo"
"heart", "heart.fill", "share", "square.and.arrow.up"

// Status & Feedback
"exclamationmark.triangle", "checkmark.circle.fill", "info.circle"
```

---

### Android ContentResolver & JNI Bridging via AnyDynamicObject

**Problem:** Skip Foundation's `Data(contentsOf:)` cannot read Android `content://` URIs (photo picker results). It produces `NSCocoaErrorDomain` error 256. To access Android platform APIs like `ContentResolver`, `BitmapFactory`, or `Intent` from Swift in Skip Fuse mode, you must use JNI bridging.

**Solution:** Skip provides `AnyDynamicObject` in `SkipBridge` for dynamic JNI calls to Java/Kotlin APIs without writing Kotlin source files.

**Setup:**
```swift
#if os(Android)
import SkipBridge
#endif
```

**Pattern 1 — Reading content:// URIs via ContentResolver (when putFileAsync is insufficient):**
```swift
#if os(Android)
private func readContentURIData(_ url: URL) throws -> Data {
    let context = ProcessInfo.processInfo.androidContext
    let contentResolver: AnyDynamicObject = try context.getContentResolver()!
    let uriClass = try AnyDynamicObject(forStaticsOfClassName: "android.net.Uri")
    let androidUri = try uriClass.parse(url.absoluteString)!
    let inputStream: AnyDynamicObject = try contentResolver.openInputStream(androidUri)!
    defer { try? inputStream.close() }

    // For API 33+: readAllBytes() returns byte[] → bridges to Swift Data
    if let bytes: Data = try? inputStream.readAllBytes() {
        return bytes
    }

    // Fallback: byte-by-byte read (slower, works on all API levels)
    let outputStream = try AnyDynamicObject(className: "java.io.ByteArrayOutputStream")
    while true {
        let byte = try inputStream.read() as Int?
        guard let byte, byte != -1 else { break }
        try outputStream.write(byte)
    }
    return try outputStream.toByteArray()!
}
#endif
```

**Pattern 2 — Calling Android Bitmap APIs for image compression:**
```swift
#if os(Android)
func compressImageForUpload(from imageData: Data, targetBytes: Int = 600_000, maxDimension: Int = 2048) throws -> CompressedImage {
    let bitmapFactory = try AnyDynamicObject(forStaticsOfClassName: "android.graphics.BitmapFactory")
    guard let bitmap: AnyDynamicObject = try bitmapFactory.decodeByteArray(imageData, 0, imageData.count) else {
        throw ImageCompressError.cannotDecode
    }

    let width: Int = try bitmap.getWidth()!
    let height: Int = try bitmap.getHeight()!

    var targetWidth = width
    var targetHeight = height
    if width > maxDimension || height > maxDimension {
        let scale = Double(maxDimension) / Double(max(width, height))
        targetWidth = Int(Double(width) * scale)
        targetHeight = Int(Double(height) * scale)
    }

    let finalBitmap: AnyDynamicObject
    if targetWidth != width || targetHeight != height {
        finalBitmap = try bitmap.createScaledBitmap(targetWidth, targetHeight, true)!
    } else {
        finalBitmap = bitmap
    }

    let compressFormatClass = try AnyDynamicObject(forStaticsOfClassName: "android.graphics.Bitmap$CompressFormat")
    let jpegFormat: AnyDynamicObject = compressFormatClass.JPEG!

    var low = 10
    var high = 95
    var bestData: Data?

    for _ in 0..<7 {
        let quality = (low + high) / 2
        let stream = try AnyDynamicObject(className: "java.io.ByteArrayOutputStream")
        let success: Bool = try finalBitmap.compress(jpegFormat, quality, stream)!
        if !success { break }

        let outputBytes: Data = try stream.toByteArray()!
        bestData = outputBytes

        if outputBytes.count > targetBytes {
            high = quality
        } else {
            low = quality
        }
    }

    if let result = bestData {
        return CompressedImage(data: result, mimeType: "image/jpeg", fileExtension: "jpg")
    }
    throw ImageCompressError.cannotCompress
}
#endif
```

**Pattern 3 — ✅ PREFERRED: Kotlin helper for content URI compression + upload (avoids chained JNI):**

Rather than chaining many `AnyDynamicObject` calls in Swift (each requiring explicit return type annotations), write the entire Android processing pipeline in Kotlin and call it with a single JNI call that returns a `String`. `String` satisfies `JConvertible` but NOT `AnyDynamicObject`, so there is **no overload ambiguity** — no type annotation required.

```kotlin
// Android/app/src/main/kotlin/ImageCompressorHelper.kt
package <your.app.package>

import android.content.Context
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.graphics.Matrix
import android.media.ExifInterface  // API 24+ (built-in, no extra dep; minSdk 33 ✓)
import android.net.Uri
import android.util.Log
import java.io.File
import java.io.FileOutputStream

class ImageCompressorHelper {
    companion object {
        private const val TAG = "ImageCompressorHelper"
        private const val MAX_DIMENSION = 2048
        private const val TARGET_BYTES = 600_000L

        @Volatile private var appContext: Context? = null

        @JvmStatic fun init(context: Context) {
            appContext = context.applicationContext
            Log.i(TAG, "Initialized")
        }

        @JvmStatic fun compress(contentUriString: String): String? {
            val ctx = appContext ?: return null
            return try {
                val uri = Uri.parse(contentUriString)
                // Pass 1: bounds only
                val opts = BitmapFactory.Options().apply { inJustDecodeBounds = true }
                ctx.contentResolver.openInputStream(uri)?.use { BitmapFactory.decodeStream(it, null, opts) }
                var sampleSize = 1
                while (opts.outWidth / sampleSize > MAX_DIMENSION || opts.outHeight / sampleSize > MAX_DIMENSION) sampleSize *= 2
                // Pass 2: decode
                val bitmap = ctx.contentResolver.openInputStream(uri)?.use { stream ->
                    BitmapFactory.decodeStream(stream, null, BitmapFactory.Options().apply { inSampleSize = sampleSize })
                } ?: return null
                // Pass 3: EXIF rotation
                val rotated = ctx.contentResolver.openInputStream(uri)?.use { stream ->
                    val orientation = ExifInterface(stream).getAttributeInt(ExifInterface.TAG_ORIENTATION, ExifInterface.ORIENTATION_NORMAL)
                    rotateBitmap(bitmap, orientation)
                } ?: bitmap
                // Scale
                val scale = MAX_DIMENSION.toFloat() / maxOf(rotated.width, rotated.height)
                val scaled = if (scale < 1f) Bitmap.createScaledBitmap(rotated, (rotated.width * scale).toInt(), (rotated.height * scale).toInt(), true).also { rotated.recycle() } else rotated
                // Compress
                val out = File(ctx.cacheDir, "cc_compressed_${System.currentTimeMillis()}.jpg")
                var quality = 85
                do {
                    FileOutputStream(out).use { scaled.compress(Bitmap.CompressFormat.JPEG, quality, it) }
                    if (out.length() > TARGET_BYTES && quality > 20) quality -= 10 else break
                } while (quality > 20)
                scaled.recycle()
                Log.i(TAG, "Done: ${out.length() / 1024}KB")
                out.absolutePath
            } catch (e: Exception) {
                Log.e(TAG, "Compression failed: ${e.message}", e)
                null
            }
        }

        private fun rotateBitmap(bitmap: Bitmap, orientation: Int): Bitmap {
            val matrix = Matrix()
            when (orientation) {
                ExifInterface.ORIENTATION_ROTATE_90 -> matrix.postRotate(90f)
                ExifInterface.ORIENTATION_ROTATE_180 -> matrix.postRotate(180f)
                ExifInterface.ORIENTATION_ROTATE_270 -> matrix.postRotate(270f)
                else -> return bitmap
            }
            return Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true).also { bitmap.recycle() }
        }
    }
}
```

**Initialize once in `Main.kt` `AndroidAppMain.onCreate()`:**
```kotlin
ProcessInfo.launch(applicationContext)
ImageCompressorHelper.init(applicationContext)   // ← before AppDelegate
AppDelegate.shared.onInit()
```

**Swift side — `ImageCompression.swift`:**
```swift
#if os(Android)
import SkipBridge
import SkipAndroidBridge

func compressAndroidContentURI(_ url: URL) throws -> String {
    let helperClass = try AnyDynamicObject(forStaticsOfClassName: "<your.app.package>.ImageCompressorHelper")
    let filePath: String? = try helperClass.compress(url.absoluteString)
    guard let filePath else { throw ImageCompressError.cannotCompress }
    return filePath
}
#endif
```

**Upload call site:**
```swift
#if os(Android)
let filePath = try compressAndroidContentURI(url)
let fileURL = URL(fileURLWithPath: filePath)
defer { try? FileManager.default.removeItem(at: fileURL) }
let metadata = StorageMetadata(); metadata.contentType = "image/jpeg"
_ = try await imageRef.putFileAsync(from: fileURL, metadata: metadata)
return try await imageRef.downloadURL()
#endif
```

**Why this is better than chained JNI:**
- All EXIF rotation, dimension bounds, quality reduction handled in Kotlin with full Android APIs
- Single JNI call; `String` return type has no overload ambiguity
- Full `android.util.Log` output visible in `adb logcat` under tag `ImageCompressorHelper`
- Errors surface as human-readable messages, not opaque "Swift JNI throwable"

**Logging from Swift:** Use `OSLog.Logger` (from `swift-android-native`/`AndroidLogging`), NOT `print()`. `print()` does NOT appear in `adb logcat`. Use `logger.log("message")` (INFO level) for reliable logcat output.
```swift
import OSLog
private let logger = Logger(subsystem: "MyApp", category: "ImageUpload")
logger.log("📸 compressing: \(url.absoluteString)")
```

**Key Rules:**
1. `AnyDynamicObject(className:)` creates a new Java/Kotlin object via reflection
2. `AnyDynamicObject(forStaticsOfClassName:)` accesses static methods/fields
3. Method calls use `try object.methodName(args)` — returns bridged types or `AnyDynamicObject`
4. `Data` bridges to `kotlin.ByteArray` automatically for arguments and return values
5. Static fields (e.g., `Bitmap.CompressFormat.JPEG`) use dynamic member lookup: `compressFormatClass.JPEG!` with explicit `AnyDynamicObject` type annotation to resolve ambiguity
6. Constructor arguments and method arguments are passed positionally in the `(...)` call
7. Always wrap Android-only JNI code in `#if os(Android)` — `AnyDynamicObject` is unavailable on iOS
8. `String` return types are never ambiguous — only `AnyDynamicObject` returns need explicit type annotations

**When to use `putFileAsync` vs JNI vs Kotlin helper:**
- **Just uploading, no compression?** → `putFileAsync(from: url)` directly — handles content URIs natively
- **Compression needed, simple pipeline?** → Kotlin helper (Pattern 3) + `putFileAsync(from: tempFileURL)`
- **Dynamic JNI to arbitrary Android APIs?** → `AnyDynamicObject` (Patterns 1/2) — but minimize chain length

---

### Pattern: iOS Location — CLLocationManager Requires `@MainActor` / Main Thread Run Loop

**Problem:** `SkipDevice.LocationProvider.fetchCurrentLocation()` wraps `CLLocationManager` on iOS. `CLLocationManager` dispatches delegate callbacks on the **main thread's run loop**. When `fetchCurrentLocation()` is called from a `withTaskGroup` child task or a `nonisolated` context, it executes on a background thread with **no active run loop** — the delegate callback fires but is never received. The call hangs indefinitely (observable in logs as `fetchCurrentLocation` printed but no `didUpdateLocations` following).

**Symptoms:**
```
[Location] Task: starting fetchCurrentLocation
fetchCurrentLocation                        ← SkipDevice's own log
[Location] Task: timeout fired              ← 8+ seconds later, no location returned
```

**Root cause:** `CLLocationManager` delegate events require the calling thread's run loop to be spinning. Swift concurrency task group child tasks and `nonisolated` functions run on the cooperative thread pool, which has no persistent run loop.

**Fix:** Call `fetchCurrentLocation()` inside a `Task { @MainActor in }`. `@MainActor` ensures the call runs on the main actor, where the run loop is always active. Implement the timeout via a separate cancellation `Task` rather than `withTaskGroup`.

```swift
// ❌ WRONG — nonisolated / background thread, CLLocationManager delegate never fires
nonisolated private static func fetchLocationWithTimeout() async -> RaceResult {
    let provider = LocationProvider()
    return await withTaskGroup(of: RaceResult.self, ...) { group in
        group.addTask {
            let location = try await provider.fetchCurrentLocation()  // ← hangs
            ...
        }
    }
}

// ❌ ALSO WRONG — same problem: withTaskGroup child tasks run on background threads
group.addTask {
    let provider = LocationProvider()
    let location = try await provider.fetchCurrentLocation()  // ← hangs
}

// ✅ CORRECT — @MainActor ensures main thread + active run loop
// In a @MainActor func (e.g. requestAndFetchLocation on a @MainActor ViewModel):
let provider = LocationProvider()

let locationTask = Task { @MainActor in
    try await provider.fetchCurrentLocation()
}
let timeoutTask = Task {
    try? await Task.sleep(for: .seconds(10))
    locationTask.cancel()
}

do {
    let location = try await locationTask.value
    timeoutTask.cancel()
    // use location.latitude, location.longitude
} catch is CancellationError {
    locationError = "Location request timed out."
} catch {
    locationError = "Failed: \(error.localizedDescription)"
}
```

**Import note:** `SkipDevice`'s `LocationEvent` type is not `Sendable`. Add `@preconcurrency` to suppress Swift 6 errors:

```swift
@preconcurrency import SkipDevice
```

**Dependency version:** Use `skip-device` `0.5.0` (verified working):
```swift
// Package.swift
.package(url: "https://source.skip.tools/skip-device.git", from: "0.5.0"),
```

**Key Rules:**
1. Always call `PermissionManager.requestLocationPermission()` BEFORE `fetchCurrentLocation()`
2. `LocationProvider()` must be created and used on `@MainActor`
3. Use `Task { @MainActor in }` + a separate cancellation `Task` for timeout — NOT `withTaskGroup`
4. Add `@preconcurrency import SkipDevice` to suppress `Sendable` errors on `LocationEvent`
5. This issue is iOS-specific. Android's `LocationProvider` uses a different underlying mechanism and is unaffected.

---

## Local-First Sync Architecture (SkipSQL + Firestore)

A cross-platform local-first cache layer: the UI reads instantly from a local SQLite database (via `SkipSQL` + `SkipSQLPlus`), while Firestore syncs in the background. SQLite works on both iOS and Android through `SkipSQLPlus`'s **bundled** SQLite — no JNI or `#if SKIP` required.

> ⚠️ **CRITICAL:** Use **`SkipSQLPlus` + `configuration: .plus`**, NOT `configuration: .platform`. The `.platform` config crashes on Android at launch with **signal 5 (SIGTRAP)** in `SQLiteConfiguration.platform`'s lazy initializer because it tries to dynamically load the OS-provided `libsqlite3`. See the dedicated pattern below.

**Layers:**
- **`LocalDatabase`** — singleton that owns the `SQLContext`, runs migrations, and vends DAOs.
- **`DatabaseSyncQueue`** — actor that serializes ALL SQLite access (prevents concurrent-access crashes on Android).
- **DAOs** (`AnnouncementDAO`, `EventDAO`, `SermonDAO`, `CheckpointStore`) — actors performing CRUD against SQLite.
- **Repositories** (`*Repository`) — actors orchestrating local-first reads + background Firestore sync.
- **Entities** (`LocalEvent`, `LocalSermon`, `LocalAnnouncement`) + **sync metadata** (`SyncResult`, `SyncCheckpoint`, `SyncConfiguration`) — plain `Sendable` value types.

**Dependency** — `Package.swift` (declare BOTH products):
```swift
.package(url: "https://source.skip.tools/skip-sql.git", "0.16.0"..<"2.0.0"),
// ...
.product(name: "SkipSQL", package: "skip-sql"),
.product(name: "SkipSQLPlus", package: "skip-sql"),   // ← bundled SQLite, required for Android
```

---

### Pattern: SkipSQL 0.16.0 API

**Problem:** The `SkipSQL` API changed significantly at 0.16.0. Older method names (`context.prepare("...")`, `stmt.bind(1, value)`, `stmt.step()`, `stmt.string(0)`, `stmt.double(0)`, `stmt.integer(0)`, `stmt.reset()`, `context.execute("...")`, `SQLContext(url:)`) no longer compile.

**Fix:** Use the 0.16.0 surface. Initialize with `SQLContext(path:flags:configuration: .plus)`, use labeled `prepare(sql:)` / `exec(sql:parameters:)`, bind with `SQLValue` cases at 1-based indices, and read columns at 0-based indices.

| Old (pre-0.16.0) | New (0.16.0) |
|---|---|
| `SQLContext(url: dbURL)` | `SQLContext(path: dbPath, flags: [.create, .readWrite], configuration: .plus)` |
| `context.execute("...")` | `context.exec(sql: "...")` |
| `context.prepare("...")` | `context.prepare(sql: "...")` |
| `stmt.bind(1, value)` | `stmt.bind(.text(value), at: 1)` / `.real(...)` / `.long(...)` |
| `stmt.step()` | `stmt.next()` |
| `stmt.reset()` | `stmt.close()` |
| `stmt.string(0)` | `stmt.text(at: 0)` (returns `String?`) |
| `stmt.double(0)` | `stmt.real(at: 0)` |
| `stmt.integer(0)` | `stmt.long(at: 0)` |
| Manual `DB_VERSION` table | `context.userVersion` (PRAGMA user_version) |

**Bind values** use the `SQLValue` enum: `.text(String)`, `.real(Double)`, `.long(Int64)`, `.blob(Data)`, `.null`. Bind indices are **1-based**; column read indices are **0-based**.

```swift
import Foundation
import SkipSQL
import SkipSQLPlus

// ✅ Read with prepared statement
public func getAll(churchId: String) throws -> [LocalEvent] {
    let stmt = try context.prepare(sql: """
        SELECT ID, CHURCH_ID, TITLE, START_DATE, IS_RECURRING
        FROM LOCAL_EVENTS WHERE CHURCH_ID = ? ORDER BY START_DATE ASC
    """)
    defer { try? stmt.close() }
    try stmt.bind(.text(churchId), at: 1)        // 1-based bind
    var results: [LocalEvent] = []
    while try stmt.next() {
        results.append(LocalEvent(
            id:          stmt.text(at: 0) ?? "",  // 0-based read
            churchId:    stmt.text(at: 1) ?? "",
            title:       stmt.text(at: 2) ?? "",
            startDate:   Date(timeIntervalSince1970: stmt.real(at: 3)),
            isRecurring: stmt.long(at: 4) != 0
        ))
    }
    return results
}

// ✅ Write with one-shot exec + parameters (no manual statement lifecycle)
public func save(_ item: LocalEvent) throws {
    try context.exec(sql: """
        INSERT OR REPLACE INTO LOCAL_EVENTS (ID, CHURCH_ID, TITLE, START_DATE, IS_RECURRING)
        VALUES (?, ?, ?, ?, ?)
    """, parameters: [
        .text(item.id),
        .text(item.churchId),
        .text(item.title),
        .real(item.startDate.timeIntervalSince1970),
        .long(item.isRecurring ? 1 : 0)
    ])
}

// ✅ Transactions + deletes are plain exec calls
public func deleteAll(churchId: String) throws {
    try context.exec(sql: "DELETE FROM LOCAL_EVENTS WHERE CHURCH_ID = ?", parameters: [.text(churchId)])
}
public func beginTransaction()    throws { try context.exec(sql: "BEGIN") }
public func commitTransaction()   throws { try context.exec(sql: "COMMIT") }
public func rollbackTransaction() throws { try context.exec(sql: "ROLLBACK") }
```

**Migrations** use `userVersion` instead of a hand-rolled version table:
```swift
let ctx = try SQLContext(path: dbPath, flags: [.create, .readWrite], configuration: .plus)
if ctx.userVersion < 1 {
    try ctx.exec(sql: "CREATE TABLE IF NOT EXISTS LOCAL_EVENTS (...)")
    ctx.userVersion = 1
}
```

---

### Pattern: `configuration: .platform` Crashes on Android (SIGTRAP) — Use `.plus`

**Problem:** Opening a `SQLContext` with `configuration: .platform` crashes the app on Android **at launch** with a native tombstone:
```
F libc    : Fatal signal 5 (SIGTRAP), code 1 (TRAP_BRKPT) ... (s.YourApp)
F DEBUG   : #01 ... SkipSQLCore (SQLiteConfiguration.platform initializer closure)
F DEBUG   : #02 ... SkipSQLCore (SQLiteConfiguration.platform one-time init)
F DEBUG   : #03 ... libswiftCore (swift::threading_impl::once_slow)   ← lazy global init traps
F DEBUG   : #05 ... YourApp`LocalDatabase.initialize()
```
The `.platform` configuration loads the **OS-provided** `libsqlite3` dynamically; on Android this fails and the lazy global initializer traps. iOS is unaffected (system SQLite links reliably).

**Fix:** Depend on `SkipSQLPlus` and use **`configuration: .plus`** — a SQLite **bundled** inside the `SkipSQLPlus` dynamic library that works identically on both platforms.

```swift
import SkipSQL
import SkipSQLPlus   // ← required

// ❌ CRASHES on Android at launch (SIGTRAP)
let ctx = try SQLContext(path: dbPath, configuration: .platform)

// ✅ CORRECT — bundled SQLite, cross-platform
let ctx = try SQLContext(path: dbPath, flags: [.create, .readWrite], configuration: .plus)
```

Also declare the `SkipSQLPlus` product in `Package.swift` (see Dependency above) and add `extension SQLContext: @unchecked Sendable {}` so the context can be held by actor-isolated DAOs.

**Debugging note:** This crash is a **native** SIGTRAP, NOT a Java `AndroidRuntime` exception — it won't appear when grepping logcat for `AndroidRuntime`/`Exception`. Capture it with `adb logcat -d | grep -E "F DEBUG|Fatal signal|libc"` and read the `F DEBUG` backtrace frames (look for your app's `.so` + `SkipSQLCore`/`SQLiteConfiguration`).

---

### Pattern: DatabaseSyncQueue — Serialize All SQLite Access

**Problem:** Concurrent SQLite access crashes on Android. Multiple actors/tasks hitting the same `SQLContext` simultaneously corrupts state.

**Fix:** Route every SQLite operation through a single `DatabaseSyncQueue` actor. The `run` method suspends the caller until the work completes and returns its result. The pending-work array must hold `@Sendable` closures so dequeued items can be invoked from a detached `@Sendable` task.

```swift
public actor DatabaseSyncQueue {
    public static let shared = DatabaseSyncQueue()

    // ✅ MUST be @Sendable — dequeued items run in a detached @Sendable task
    private var pending: [@Sendable () async throws -> Void] = []
    private var isProcessing = false

    // Generic result MUST be Sendable; work closure MUST be @Sendable @escaping
    public func run<T: Sendable>(_ work: @Sendable @escaping () async throws -> T) async throws -> T {
        try await withCheckedThrowingContinuation { continuation in
            pending.append {
                do { continuation.resume(returning: try await work()) }
                catch { continuation.resume(throwing: error) }
            }
            if !isProcessing {
                isProcessing = true
                Task.detached { [weak self] in await self?.drain() }
            }
        }
    }

    private func drain() async {
        while !pending.isEmpty {
            let item = pending.removeFirst()
            await Task.detached { @Sendable in try? await item() }.value
        }
        isProcessing = false
    }
}

// Usage from a Repository actor:
let events = try await DatabaseSyncQueue.shared.run {
    let dao = try LocalDatabase.shared.eventDAO()
    return try await dao.getAll(churchId: churchId)
}
```

---

### Pattern: Sendable Conformance for DAO Entities & Sync Metadata

**Problem:** DAO actor methods return value types (`LocalEvent`, `SyncCheckpoint`, etc.) across the actor boundary. Swift 6 strict concurrency requires those types to be `Sendable`:
```
Type 'LocalEvent' does not conform to the 'Sendable' protocol
Static property 'noChange' is not concurrency-safe because non-'Sendable' type 'SyncResult' may have shared mutable state
```

**Fix:** Add `: Sendable` to every entity and sync-metadata struct. These are simple value types (only `String`, `Date`, `Bool`, `Int`, and optionals), so `Sendable` is synthesized automatically — no `@unchecked` needed.

```swift
// ❌ Missing conformance — fails when crossing the DAO actor boundary
public struct LocalEvent { ... }
public struct SyncResult { ... }
public struct SyncCheckpoint { ... }
public struct SyncConfiguration { ... }

// ✅ Add : Sendable to all DAO entities + sync metadata value types
public struct LocalEvent: Sendable { ... }
public struct LocalSermon: Sendable { ... }
public struct LocalAnnouncement: Sendable { ... }
public struct SyncResult: Sendable { ... }
public struct SyncCheckpoint: Sendable { ... }
public struct SyncConfiguration: Sendable { ... }
```

---

### Pattern: `@unchecked Sendable` for the Non-Actor DB Manager Singleton

**Problem:** `LocalDatabase` is a `class` singleton (`static let shared`) with mutable stored properties (`_context`, cached DAOs, `isReady`). Swift 6 flags:
```
Static property 'shared' is not concurrency-safe because non-'Sendable' type 'LocalDatabase' may have shared mutable state
```

**Fix:** Mark the class `final ... : @unchecked Sendable`. This is safe **because** all mutation flows through `DatabaseSyncQueue.shared`, which serializes access. (An actor would also work but complicates the synchronous DAO-vending API; `@unchecked Sendable` + the serial queue is the chosen pattern.)

```swift
// ❌ Plain class singleton — not concurrency-safe
public class LocalDatabase {
    public static let shared = LocalDatabase()
    private var _context: SQLContext?
}

// ✅ final + @unchecked Sendable (access serialized via DatabaseSyncQueue)
public final class LocalDatabase: @unchecked Sendable {
    public static let shared = LocalDatabase()
    private var _context: SQLContext?
}
```

---

### Pattern: Copy `var` to `let` Before a `@Sendable` Closure

**Problem:** A repository builds a results array incrementally (`var locals: [LocalEvent] = []` + `.append(...)`), then passes it into the `@Sendable` `DatabaseSyncQueue.shared.run { }` closure. Swift 6 errors:
```
Capture of 'locals' with non-Sendable type '[LocalEvent]' in a '@Sendable' closure
Reference to captured var 'locals' in concurrently-executing code
```
Even after the element type is `Sendable`, capturing a mutable `var` in a concurrently-executing closure is illegal.

**Fix:** Bind an immutable copy (`let toSave = locals`) right before the closure and reference the `let` inside it.

```swift
// ❌ WRONG — captures the mutable var directly
var locals: [LocalEvent] = []
for doc in batch.docs { locals.append(LocalEvent(...)) }
try await DatabaseSyncQueue.shared.run {
    for item in locals { try await dao.save(item) }   // ← capture-of-var error
}

// ✅ CORRECT — immutable snapshot captured by the @Sendable closure
var locals: [LocalEvent] = []
for doc in batch.docs { locals.append(LocalEvent(...)) }
let toSave = locals                                    // immutable copy
try await DatabaseSyncQueue.shared.run {
    for item in toSave { try await dao.save(item) }    // ✓ captures let
}
```

---

### Pattern: `Timestamp.seconds` Is `Int64` on Android

**Problem:** When parsing a Firestore `Timestamp` on Android (`SkipFirebaseFirestore`), `ts.seconds` is typed `Int64`. Adding it to a fractional `Double` term fails:
```
cannot convert value of type 'Int64' to expected argument type 'Double'
```

**Fix:** Wrap `ts.seconds` in `Double(...)`. (On iOS `FirebaseFirestore`, prefer `ts.dateValue().timeIntervalSince1970` which is already `Double`.)

```swift
public func parseTimestampEpoch(_ data: [String: Any], key: String) -> Double {
    #if os(Android)
    // ❌ ts.seconds is Int64 — Double(...) wrap required
    // if let ts = data[key] as? Timestamp { return ts.seconds + Double(ts.nanoseconds) / 1_000_000_000 }
    // ✅
    if let ts = data[key] as? Timestamp { return Double(ts.seconds) + Double(ts.nanoseconds) / 1_000_000_000 }
    if let d  = data[key] as? Double    { return d }
    #else
    if let ts = data[key] as? Timestamp { return ts.dateValue().timeIntervalSince1970 }
    if let d  = data[key] as? Double    { return d }
    #endif
    return 0
}
```

**Note:** Firestore timestamps also arrive as a raw `Double` (epoch seconds) on some Android decode paths — always try `as? Timestamp` first, then fall back to `as? Double`.

---

### Pattern: Bypass Local Cache for Small Firestore Collections with External URLs

**Problem:** Collections like `sermons` store `videoUrl` (YouTube links) and `thumbnail` fields. A SQLite cache with checkpoint-based staleness (`maxAge`) may never refresh if the sync was written before a field existed or if `maxAge` is generous. The UI shows empty URLs → no thumbnails, broken links, and silent failures.

**Fix:** For small, metadata-only collections that contain external URLs, fetch directly from Firestore. The network payload is tiny (~KBs) and URLs must be the current source of truth. Reserve the SQLite cache for large/heavy data or offline-critical content.

```swift
// ❌ WRONG — stale SQLite cache may hold empty videoUrl/thumbnail forever
func loadSermons(churchId: String) async {
    let cached = try await SermonRepository.shared.getAll(churchId: churchId)
    sermons = cached.map { ChurchSermon(from: $0) }  // stale!

    let result = try await SermonRepository.shared.sync(
        churchId: churchId, strategy: .cacheFirst
    )
    if result.hasChanges { /* ...still reading stale cache */ }
}

// ✅ CORRECT — direct Firestore fetch; source of truth
func loadSermons(churchId: String) async {
    isLoading = true
    do {
        let db = Firestore.firestore()
        let batch = try await fetchChurchCollection(
            db: db, churchId: churchId, collection: "sermons"
        )
        sermons = batch.docs.compactMap { doc -> ChurchSermon? in
            guard let data = doc.data() else { return nil }
            return ChurchSermon(id: doc.documentID, data: data)
        }
        .sorted { $0.date > $1.date }
    } catch {
        // handle error
    }
    isLoading = false
}
```

**Key Rules:**
1. Direct Firestore fetch is the source of truth for URL-bearing collections
2. SQLite cache works for offline-critical data (large text, images, etc.)
3. `fetchChurchCollection()` helper is already cross-platform (iOS direct Firestore SDK + SkipFirebaseFirestore on Android)
4. For thumbnails, prefer the Firestore-stored `thumbnail` field (written by the web app's YouTube sync), then fall back to a derived `img.youtube.com/vi/{id}/hqdefault.jpg`

---

### Pattern: `.lineLimit(ClosedRange)` Not Available in Skip

**Problem:** SwiftUI's `.lineLimit(_:)` has an overload that accepts a `ClosedRange<Int>` (e.g., `.lineLimit(2...4)`) to set a dynamic line count range. Skip's `SkipFuseUI` does not bridge this overload — only the single `Int` overload is available. Attempting to use a range produces:

```
cannot convert value of type 'ClosedRange<Int>' to expected argument type 'Int'
```

**Fix:** Replace `.lineLimit(min...max)` with `.lineLimit(max)` (use the upper bound). For unlimited lines, use `.lineLimit(nil)`.

```swift
// ❌ ERROR — ClosedRange<Int> overload not available in Skip
TextField("Notes", text: $formNotes, axis: .vertical)
    .lineLimit(2...4)

// ✅ CORRECT — use single Int (upper bound)
TextField("Notes", text: $formNotes, axis: .vertical)
    .lineLimit(4)

// ✅ CORRECT — unlimited lines
Text("Long text...")
    .lineLimit(nil)
```

**Applies to:** `Text`, `TextField`, and any view using `.lineLimit(_:)`.

**Affected files (Worship Compass admin views):**
- `AdminWorshipRehearsalsView.swift`
- `AdminWorshipTeamView.swift`
- `AdminWorshipSetListsView.swift`

---

### Pattern: `UIApplication.applicationIconBadgeNumber` Deprecated in iOS 17

**Problem:** `UIApplication.shared.applicationIconBadgeNumber = 0` was deprecated in iOS 17.0. Using it produces:
```
'applicationIconBadgeNumber' was deprecated in iOS 17.0: Use -[UNUserNotificationCenter setBadgeCount:withCompletionHandler:] instead.
```

Additionally, calling it from a `/* SKIP @bridge */` method (which is nonisolated) produces:
```
Main actor-isolated class property 'shared' can not be mutated from a nonisolated context
Main actor-isolated property 'applicationIconBadgeNumber' can not be mutated from a nonisolated context
```

**Fix:** Replace with `UNUserNotificationCenter.current().setBadgeCount()` and wrap in `Task { @MainActor in }` to satisfy both the deprecation and the MainActor isolation.

```swift
// ❌ ERROR — deprecated + MainActor isolation
#if !os(Android)
UIApplication.shared.applicationIconBadgeNumber = 0
#endif

// ✅ CORRECT — modern API + @MainActor from @bridge
#if !os(Android)
Task { @MainActor in
    UNUserNotificationCenter.current().setBadgeCount(0)
}
#endif
```

**Import required:** Add `import UserNotifications` in the `#if !os(Android)` block.

**Applies to:** Any `@bridge` lifecycle method (`onResume`, `onLaunch`, etc.) that needs to mutate UIKit state or clear badge counts.

**Affected file:**
- `ChurchCompassAdminApp.swift`
---

### Pattern: Firebase Cloud Messaging — APNS Token Forwarding on iOS

**Problem:** FCM token retrieval fails silently with:
```
APNS device token not set before retrieving FCM Token for Sender ID '...'
Declining request for FCM Token since no APNS Token specified
[FCMTokenManager] Failed to get FCM token: ... No APNS token specified before fetching FCM Token
```

This happens when `UIApplicationDelegate.application(_:didRegisterForRemoteNotificationsWithDeviceToken:)` is not implemented in the Darwin `Main.swift`. The Firebase Messaging SDK needs the APNS token forwarded to `Messaging.messaging().apnsToken` before it can request an FCM token.

**Skip projects use a custom `AppDelegate` bridge pattern** (`ChurchCompassAdminAppDelegate` / `ChurchCompassAppDelegate`) that lives in the shared SPM module, but the actual `UIApplicationDelegate` methods must be declared in the platform-specific Darwin `Main.swift`.

**Fix:** Add the APNS delegate methods to your Darwin `Main.swift`, and import `FirebaseMessaging`:

```swift
import SwiftUI
import ChurchCompassAdmin   // or ChurchCompass for member app
import FirebaseMessaging    // ← REQUIRED

// ... AppMain and AppMainDelegateBase aliases ...

@MainActor final class AppMainDelegate: NSObject, AppMainDelegateBase {
    // ... existing lifecycle methods ...

    #if canImport(UIKit)
    // ... onInit, onLaunch, onDestroy, onLowMemory ...

    // ✅ REQUIRED — forwards APNS token to Firebase Messaging
    func application(_ application: UIApplication, didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data) {
        Messaging.messaging().apnsToken = deviceToken
    }

    // ✅ RECOMMENDED — logs failure reason instead of silently failing
    func application(_ application: UIApplication, didFailToRegisterForRemoteNotificationsWithError error: Error) {
        print("[APNS] Failed to register: \(error.localizedDescription)")
    }
    #endif
}
```

**Key requirements:**
- **`import FirebaseMessaging`** must be present in Darwin `Main.swift` (not just the shared module)
- Both methods must be inside the `#if canImport(UIKit)` block
- This is required for **both admin and member apps** (each has its own Darwin `Main.swift`)

**FCMTokenManager pattern:**

The shared `FCMTokenManager` typically calls `UIApplication.shared.registerForRemoteNotifications()` and then polls for the APNS token before requesting the FCM token:

```swift
// Inside FCMTokenManager.registerForPushNotifications()
#if !os(Android)
UIApplication.shared.registerForRemoteNotifications()
// Wait for APNS token to be forwarded (via didRegisterForRemoteNotificationsWithDeviceToken above)
for _ in 0..<20 {
    if Messaging.messaging().apnsToken != nil { break }
    try? await Task.sleep(nanoseconds: 500_000_000)
}
// Now safe to request FCM token
let fcmToken = try await withCheckedThrowingContinuation { cont in
    Messaging.messaging().token { token, error in
        if let token { cont.resume(returning: token) }
        else { cont.resume(throwing: error ?? NSError(...)) }
    }
}
#endif
```

**Performance note:**
- On **iOS Simulator**: APNS is unavailable, so `didRegisterForRemoteNotificationsWithDeviceToken` is never called. The 10-second poll times out. Wrap `registerForPushNotifications()` in a non-blocking `Task {}` so login isn't delayed.
- On **real iOS device**: APNS token arrives quickly (usually <1s) after `registerForRemoteNotifications()`, and FCM token retrieval succeeds.

**Affected files:**
- `mobile-apps/admin/Darwin/Sources/Main.swift`
- `mobile-apps/member/Darwin/Sources/Main.swift`

---

### Pattern: `TextField(axis:)` Not Available in Skip

**Problem:** SwiftUI's `TextField` initializer with the `axis:` parameter (e.g., `TextField("Notes", text: $formNotes, axis: .vertical)`) is not available in Skip. Only the plain `TextField(_:text:)` overload is bridged to Android.

**Error:**
```
'init(_:text:axis:)' is unavailable
```

**Fix:** Remove the `axis:` parameter. Use `TextField(_:text:)` instead. The `.lineLimit(_:)` modifier (with a single `Int`) can still be applied afterward, though it has no effect on a single-line `TextField` in Skip.

```swift
// ❌ ERROR — axis parameter not available in Skip
TextField("Notes", text: $formNotes, axis: .vertical)
    .lineLimit(4)

// ✅ CORRECT — plain TextField
TextField("Notes", text: $formNotes)
    .lineLimit(4)
```

**Applies to:** Any `TextField` using `axis: .vertical` or `axis: .horizontal`.

**Affected files (Worship Compass admin views):**
- `AdminWorshipRehearsalsView.swift`
- `AdminWorshipTeamView.swift`
- `AdminWorshipSetListsView.swift`
