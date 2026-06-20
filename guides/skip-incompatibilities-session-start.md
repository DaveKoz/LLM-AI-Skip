---
description: Load Skip incompatibilities context at the start of a new session — reads the doc, surfaces rules, and primes memory for any new fixes found this session
---

> ⚠️ **STANDING RULE — No Automatic Builds**
>
> **Do NOT run `swift build`, `skip build`, `gradle build`, or any other automatic build commands.** The user builds manually through Xcode. Build verification is done by the user, not by the agent. Perform manual code review and syntax checking instead.
>
> ⚠️ **STANDING RULE — Auto-Document All Fixes**
>
> **Every single bug fix, compiler error fix, or API workaround MUST be documented in the same response.**
> Do NOT wait for the user to ask "document this." Automatically append the new rule to:
> 1. `docs/skip-incompatibilities.md` (numbered rule, ❌/✅ examples)
> 2. `.devin/workflows/skip-incompatibilities-session-start.md` (quick-reference pattern)
> 3. `.windsurf/workflows/skip-cross-platform-architecture.md` (if architectural)
> 4. Memory entry `7ed0a826-aa35-466e-80e9-bd5aa57e9377` (Pre-Build Checklist)
>
> **Decision Guide:** Specific API ban → session-start.md; Architectural pattern → architecture.md; Both → both. Always append to skip-incompatibilities.md regardless.

## Skip Incompatibilities — Session Start Workflow

Run this at the beginning of any session that involves fixing Skip transpiler errors on a Skip Tools project.

> **Note:** This workflow file is portable and may be copied between projects. However, the `docs/skip-incompatibilities.md` data file is project-specific. When you copy this workflow to a new project, ensure you also copy the current `docs/skip-incompatibilities.md` file, or manually sync any new rules discovered in this session to both files.

### Step 1 — Read the incompatibilities doc and architecture patterns

Read the full contents of `docs/skip-incompatibilities.md` from the project root.

```
read_file: docs/skip-incompatibilities.md
```

Also read the cross-platform architecture workflow for native library integration patterns:

```
read_file: .windsurf/workflows/skip-cross-platform-architecture.md
```

Summarize the key rules and architecture patterns in memory so they are available throughout the session without re-reading the files.

### Step 2 — Update memory with current rules

After reading the doc, update the agent memory entry titled **"Skip Tools — SwiftUI/Swift Incompatibilities Reference"** with the latest content from the file. If the memory entry does not exist yet, create it with the tag `skip_tools`.

The memory must always reflect the full, current rule set from the doc. Do not merge or append — overwrite with the freshest version.

### Step 3 — Announce the active rule set

Briefly list the incompatibility categories that are currently documented (e.g. Access Modifiers, Colors, ForEach, UserDefaults, Firestore, etc.) so the user knows what has been loaded.

### Step 4 — Fix errors as they are reported

When the user pastes a compiler error:

1. Identify the incompatibility category from the known rule set.
2. Apply the documented fix pattern.
3. If the error represents a **new, previously undocumented** incompatibility:
   a. Fix the code.
   b. Append a new section to `docs/skip-incompatibilities.md` **AND** update `.windsurf/workflows/skip-incompatibilities-session-start.md` (this file) with the new rule so it stays in sync when copied to other projects.
   c. If the fix involves a **cross-platform dependency** (native library, Android API via `#if SKIP`, `ContentComposer`, `FileProvider`, etc.), also add the pattern to `.windsurf/workflows/skip-cross-platform-architecture.md` using the template at the bottom of that file.
   
   Use this template for incompatibilities:

```
### <Short title>
**Problem:** <One sentence describing what fails and why.>  
**Fix:** <One sentence describing the correct pattern.>

```swift
// ❌
<before code>

// ✅
<after code>
` `` `
```

   c. Update the memory entry to include the new rule.
   d. Commit both the code fix and the doc update together:
      ```
      git add -A
      git commit -m "fix(<scope>): <description>; document new Skip incompatibility"
      git push origin main
      ```

### Known Patterns — Google Play Photo/Video Permissions Ban

**Problem:** Google Play rejects apps that declare `READ_MEDIA_IMAGES` or `READ_MEDIA_VIDEO` unless persistent, ongoing access to shared photo/video storage is the core purpose of the app. Profile photo pickers and one-time image uploads do NOT qualify. Declaring these permissions causes a policy violation and app rejection.

**Fix:** Remove `READ_MEDIA_IMAGES` and `READ_MEDIA_VIDEO` from `AndroidManifest.xml` entirely. Use the Android photo picker (system intent — no permission required). Scope `READ_EXTERNAL_STORAGE` to Android 12 and below with `maxSdkVersion="32"`, and `WRITE_EXTERNAL_STORAGE` to Android 9 and below with `maxSdkVersion="28"`.

```xml
<!-- ❌ Rejected by Google Play for one-time/profile photo use -->
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

<!-- ✅ Correct — photo picker needs no permissions; scope legacy storage permissions -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" android:maxSdkVersion="32" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="28" />
```

The Android photo picker (returned as `content://` URI) is handled in Skip via `putFileAsync(from:)` or JNI `ContentResolver` — see Content URI Reading rule.

---

### Known Patterns — Firestore Field Deletion (`NSNull` crashes on Android)

**Problem:** Using `NSNull()` in a `[String: Any]` dict passed to `updateData()` causes a fatal crash on Android. `SkipBridge` cannot convert `NSNull` (an Objective-C type) to a Java object and calls `assertionFailure`, crashing the app immediately. The stack trace shows `libSkipBridge.so → AnyBridgingV.toJavaObject → assertionFailure`.

**Fix:** Replace `NSNull()` with `FieldValue.delete()` to remove a Firestore field. `FieldValue.delete()` is fully supported by SkipFirebaseFirestore.

```swift
// ❌ Crashes on Android — NSNull cannot be bridged to Java
let data: [String: Any] = [
    "billingInterval": NSNull()
]
try await ref.updateData(data)

// ✅ Correct — FieldValue.delete() is bridgeable
let data: [String: Any] = [
    "billingInterval": FieldValue.delete()
]
try await ref.updateData(data)
```

**Note:** This also applies to `setData()`. Any `NSNull()` value anywhere in a `[String: Any]` dict passed to a Firestore call will crash on Android.

### Known Patterns — Firestore / Firebase Imports

**`doc.data(as: T.self)` and `ref.setData(from:)`** are unavailable — Skip's Firestore SDK does not include `FirebaseFirestoreSwift`. Use `doc.data()` and decode from `[String: Any]` manually.

```swift
// ❌
let item = try? doc.data(as: MyModel.self)

// ✅
let d = doc.data()
var dWithId = d
dWithId["id"] = doc.documentID
guard let id = dWithId["id"] as? String else { return nil }
// Decode manually from [String: Any]
```

**Firebase Functions Import** — Use platform-conditional imports with `@preconcurrency`:

```swift
#if os(Android)
@preconcurrency import SkipFirebaseFunctions
#else
@preconcurrency import FirebaseFunctions
#endif

// Use Functions.functions().httpsCallable() for Cloud Function calls
```

```swift
// ❌
let item = try? doc.data(as: MyModel.self)

// ✅
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

### Known Patterns — Layout

**`.safeAreaInset(edge:content:)`** is unavailable in skip-fuse-ui.

```swift
// ❌
.safeAreaInset(edge: .bottom) {
    bottomBar
}

// ✅
.overlay(alignment: .bottom) {
    bottomBar
        .background(.ultraThinMaterial)
}
```

### Known Patterns — SwipeActions

**`.swipeActions`** is fully unavailable in skip-fuse-ui — all overloads including the no-argument form. Use `.contextMenu` instead.

```swift
// ❌ (all forms unavailable)
.swipeActions(edge: .trailing) { ... }
.swipeActions() { ... }

// ✅
.contextMenu {
    Button(role: .destructive) { ... } label: { Label("Delete", systemImage: "trash") }
}
```

### Known Patterns — Picker Styles

**`.pickerStyle(.wheel)`** is unavailable in Skip. Use `.pickerStyle(.menu)` instead.

```swift
// ❌ ERROR — .wheel unavailable
Picker("Hour", selection: $hour) {
    ForEach(0..<24) { h in Text("\(h)").tag(h) }
}
.pickerStyle(.wheel)

// ✅ CORRECT — .menu works cross-platform
Picker("Hour", selection: $hour) {
    ForEach(0..<24) { h in Text("\(h)").tag(h) }
}
.pickerStyle(.menu)
```

Supported picker styles in Skip: `.automatic`, `.menu`, `.segmented`, `.navigationLink` (iOS/Android only).

### Known Patterns — Skip Fuse Native Mode (`#if SKIP` vs `#if os(Android)`)

**In Skip Fuse mode (`skip.yml`: `mode: 'native'`), `#if SKIP` is FALSE.** The transpiled Kotlin code is not executed — Swift code is compiled natively for Android. This means:

| Directive | Skip Fuse Behavior | Skip Lite Behavior |
|-----------|------------------|------------------|
| `#if os(Android)` | ✅ Native Swift (no Java interop) | ✅ Transpiled to Kotlin |
| `#if SKIP` | ❌ **FALSE** — code excluded | ✅ Transpiled to Kotlin |

**Problem:** Code in `#if SKIP` blocks (like `shareQRImage()` using `ProcessInfo.processInfo.androidContext`) is **invisible** in Skip Fuse builds, causing silent failures or fall-through to wrong implementations.

**Fix:** Use `#if os(Android)` for Android-specific SwiftUI code, and use `ComposeView` + `ContentComposer` to bridge to Android APIs.

```swift
// ❌ This won't execute in Skip Fuse native mode
#if SKIP
func shareQRImage() {
    let ctx = ProcessInfo.processInfo.androidContext  // Invisible!
    // ...
}
#endif

// ✅ Correct pattern for Skip Fuse
#if os(Android)
// SwiftUI view with Compose bridge
func shareQRImageAndroid(content: String, label: String) {
    QRShareState.shared.share(content: content, label: label)
}
#endif

#if SKIP
// Transpiled to Kotlin — can call Java APIs
import androidx.compose.ui.platform.ComposeView

@Observable final class QRShareState: @unchecked Sendable {
    static let shared = QRShareState()
    var trigger: Int = 0
    // ...
}

/// ContentComposer uses primitive `let` properties — NOT @Binding with custom class
/// Skip cannot bridge @Binding with custom structs, use primitives only
struct QRShareComposer: ContentComposer {
    let content: String
    let trigger: Int

    init(content: String, trigger: Int) {
        self.content = content
        self.trigger = trigger
    }

    @Composable func Compose(context: ComposeContext) {
        LaunchedEffect(trigger) {
            performShare()  // Calls Java APIs here
        }
    }
}
#endif
```

**Key Rules:**
1. Use `#if os(Android)` for Android-specific SwiftUI body code
2. Use `#if SKIP` only for ContentComposer definitions and their Kotlin imports
3. In Skip Fuse, Android API access requires `ComposeView` + `ContentComposer` bridge
4. State sharing between native Swift and Compose uses `@Observable` + primitive properties
5. **`@Observable` state classes MUST be outside all `#if` blocks** — placing them inside prevents type bridging
6. **`ContentComposer` must use primitive `let` properties** — Skip cannot bridge `@Binding` with custom structs (error: "does not appear to be a bridged type")

### Known Patterns — @State private / @Environment private

**Problem:** `@State private var` and `@Environment(...) private var` cannot be bridged to Android — "Private state property cannot be bridged to Android."

**Fix:** Remove `private` from ALL `@State` and `@Environment` properties in every View struct.

```swift
// ❌
@State private var viewModel = MyViewModel()
@Environment(\.dismiss) private var dismiss

// ✅
@State var viewModel = MyViewModel()
@Environment(\.dismiss) var dismiss
```

---

### Known Patterns — @FocusState Crash on Android

**Problem:** `@FocusState` + `.focused()` crashes Android: "offset(0) is out of bounds [0, 0)".

**Fix:** Remove `@FocusState` and `.focused()` entirely. Do not use focus management on Android.

```swift
// ❌ CRASH
@FocusState var focused: Field?
TextField("Email", text: $email).focused($focused, equals: .email)

// ✅
TextField("Email", text: $email)
```

---

### Known Patterns — ShapeStyle `.accent` Not Available

**Problem:** `.foregroundStyle(.accent)` — "Type 'ShapeStyle' has no member 'accent'".

**Fix:** Use `Color.accentColor` instead.

```swift
// ❌
.foregroundStyle(.accent)

// ✅
.foregroundStyle(Color.accentColor)
```

---

### Known Patterns — Color System Backgrounds Not Available

**Problem:** `Color(.systemBackground)`, `Color(.secondarySystemBackground)`, etc. are UIKit-only — "reference to member cannot be resolved".

**Fix:** Use `Color.gray.opacity()` instead.

```swift
// ❌
.background(Color(.systemBackground))
.background(Color(.secondarySystemBackground))

// ✅
.background(Color.white)
.background(Color.gray.opacity(0.1))
```

---

### Known Patterns — `Color(.systemGray*)` Not Available

**Problem:** `Color(.systemGray6)` etc. are UIKit system colors — "reference to member cannot be resolved without a contextual type".

**Fix:** Use `Color.gray.opacity()` instead.

```swift
// ❌
.background(Color(.systemGray6))

// ✅
.background(Color.gray.opacity(0.1))
```

---

### Known Patterns — `.autocapitalization` Not Available

**Problem:** `.autocapitalization(.none)` is a UIKit API — "value of type 'some View' has no member 'autocapitalization'".

**Fix:** Use `.textInputAutocapitalization(.never)` instead.

```swift
// ❌
TextField("Email", text: $email).autocapitalization(.none)

// ✅
TextField("Email", text: $email).textInputAutocapitalization(.never)
```

---

### Known Patterns — SF Symbol Platform Branching

**Problem:** Some SF Symbols are reserved or unavailable on Android (e.g., `message` is iOS-only). Android requires the symbol to exist as a `.symbolset` in `Module.xcassets`.

**Fix:** Use `#if os(Android)` to provide an alternative symbol name for Android.

```swift
// ❌ — "message" may not render on Android
Image(systemName: "message")

// ✅ — Platform-specific symbol names
#if os(Android)
Image(systemName: "bubble")
#else
Image(systemName: "message")
#endif
```

**Also remember:** `Color("name")` from `.xcassets` fails on Android — define static `Color` extension properties with `#if os(Android)` branch instead.

---

### Known Patterns — Lazy Stacks and Grids Not Available

**Problem:** `LazyVStack`, `LazyHStack`, `LazyVGrid`, `LazyHGrid` are not bridged to Android.

**Fix:** Replace with standard `VStack`/`HStack`. For grids, chunk into `VStack` of `HStack` rows manually.

```swift
// ❌
LazyVGrid(columns: [...]) { ... }

// ✅
VStack(spacing: 10) {
    HStack(spacing: 10) { /* row 1 items */ }
    HStack(spacing: 10) { /* row 2 items */ }
}
```

---

### Known Patterns — `Layout` Protocol Not Available

**Problem:** The `Layout` protocol (iOS 16+) and its associated types (`ProposedViewSize`, `Subviews`, `sizeThatFits(proposal:subviews:cache:)`, `placeSubviews(in:proposal:subviews:cache:)`) are not bridged to Android. Using a custom `Layout` conforming struct produces: `static method 'buildExpression' requires that 'FlowLayout' conform to 'View'`.

**Fix:** Replace with `HStack`, `VStack`, `ScrollView(.horizontal)` with `HStack`, or platform-specific Compose alternatives.

```swift
// ❌
struct FlowLayout: Layout { ... }
FlowLayout(spacing: 6) { TagView(...) }

// ✅
HStack(spacing: 6) { TagView(...) }
```

---

### Known Patterns — NavigationStack Large Title on Android

**Problem:** `.navigationBarTitleDisplayMode(.large)` causes ~15% wasted whitespace at top on Android.

**Fix:** Use `.inline` on Android. The root cause of extreme padding is a double-wrapped `NavigationStack` (see pattern below).

```swift
#if os(Android)
.navigationBarTitleDisplayMode(.inline)
#else
.navigationBarTitleDisplayMode(.large)
#endif
```

---

### Known Patterns — Double NavigationStack in TabView

**Problem:** Wrapping a tab's content in `NavigationStack` in both `ContentView` AND in the child view causes ~15% wasted whitespace on Android.

**Fix:** Only ONE `NavigationStack` per tab — either in `ContentView` OR the child view, never both.

```swift
// ❌ — double wrap
TabView { NavigationStack { MyView() } }  // wrap 1
struct MyView: View { var body: some View { NavigationStack { ... } } }  // wrap 2

// ✅ — single wrap in ContentView; child uses .navigationTitle directly
TabView { NavigationStack { MyView() } }
struct MyView: View { var body: some View { VStack { ... }.navigationTitle("Title") } }
```

---

### Known Patterns — Bold Inline Title on Android

**Problem:** `.navigationBarTitleDisplayMode(.inline)` on Android produces a small title that doesn't match iOS large title prominence.

**Fix:** On Android use `ToolbarItem(placement: .principal)` with `.font(.title).fontWeight(.bold)`.

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

### Known Patterns — `.topBarTrailing` and `navigationBarTitleDisplayMode` Unavailable on macOS

**Problem:** `.topBarTrailing` toolbar placement and `.navigationBarTitleDisplayMode` are unavailable on macOS, causing build errors when `swift build` targets the macOS platform.

**Fix:** Use `.automatic` for toolbar placement; wrap `navigationBarTitleDisplayMode` in `#if os(iOS)`.

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

### Known Patterns — Button Closure Properties Require `@MainActor @Sendable`

**Problem:** Swift 6 strict concurrency warns "converting non-Sendable function value to '@MainActor @Sendable () -> Void' may introduce data races" when a View struct stores a plain `() -> Void` closure that is passed to a `Button`.

**Fix:** Declare the closure property as `@MainActor @Sendable () -> Void`.

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

### Known Patterns — `@Observable` Requires `import Observation` Before `import SkipFuse`

**Problem:** Using `@Observable` causes "unknown attribute 'Observable'" if `import Observation` is missing or placed after `import SkipFuse`. The macro is defined in the `Observation` framework and must be imported first.

**Fix:** Always import `Observation` before `SkipFuse` in any file using `@Observable`.

```swift
// ❌ ERROR
import Foundation
import SkipFuse

@Observable class MyViewModel { ... }

// ✅ Correct
import Foundation
import Observation
import SkipFuse

@Observable class MyViewModel { ... }
```

---

### Known Patterns — `.fixedSize` Not Available on Text

**Problem:** `.fixedSize(horizontal:vertical:)` is unavailable on `Text` in Skip — "value of type 'Text' has no member 'fixedSize'".

**Fix:** Use `.lineLimit(nil)` instead, which allows unlimited lines on both platforms.

```swift
// ❌
Text(description).fixedSize(horizontal: false, vertical: true)

// ✅
Text(description).lineLimit(nil)
```

---

### Known Patterns — Compiler Type-Checking Timeouts

**"The compiler is unable to type-check this expression in reasonable time"** — long `View.body` properties, large `Form` blocks, or chains of 10+ `.onChange` modifiers overwhelm the Swift type checker (compounded by Skip's transpiler). Break up into `@ViewBuilder` computed properties and `private func` helpers.

```swift
// ❌ — too much in one body
var body: some View {
    Form { /* 50+ lines */ }
    .onChange(of: a) { ... }
    // ... 15 more onChange
    .toolbar { formToolbar }
}

// ✅ — decompose
var body: some View { formContent }

@ViewBuilder var baseForm: some View {
    Form { conditionsSection; broodSection }
    .toolbar { formToolbar }
    .onAppear { populateFields() }
}

@ViewBuilder var formContent: some View {
    baseForm
        .onChange(of: a) { _, _ in saveDraft() }
        .onChange(of: b) { _, _ in saveDraft() }
}
```

**Rule of thumb:** >8 modifiers or >30 lines of view DSL in one property → extract further.

### Known Patterns — Closures

**Closures stored for later use** should use `@escaping`, but **only in function parameter position**. Skip does not support `@escaping` on stored properties or typealiases (Kotlin lambdas are naturally escaping).

```swift
// ❌ In function parameter - missing @escaping
func setup(onSelect: (Coordinates?) -> Void)

// ✅ In function parameter - with @escaping
func setup(onSelect: @escaping (Coordinates?) -> Void)

// ❌ On stored property - not supported by Skip
struct MyView: View {
    let onSelect: @escaping (Coordinates?) -> Void  // Don't do this
}

// ✅ On stored property - omit @escaping, Skip handles it
struct MyView: View {
    let onSelect: (Coordinates?) -> Void  // OK - Kotlin lambda is escaping
}
```

**Trailing closures in View inits** may cause escaping issues. Extract to local variable:

```swift
// ❌ Trailing closure - may cause Skip bridge errors
.sheet(isPresented: $show) {
    MapPickerView(...) { coord in
        ...
    }
}

// ✅ Extract to local variable first
.sheet(isPresented: $show) {
    let onSelect: (Coordinates?) -> Void = { coord in
        ...
    }
    MapPickerView(..., onSelect: onSelect)
}

// ✅ RECOMMENDED: Use Binding instead of closures
// Define view with binding:
struct MapPickerView: View {
    @Binding var selectedCoordinate: Coordinates?
}

// Use in parent:
.sheet(isPresented: $show) {
    let binding = Binding<Coordinates?>(
        get: { currentCoordinate },
        set: { currentCoordinate = $0 }
    )
    MapPickerView(selectedCoordinate: binding)
}
```

### Known Patterns — Deep Link Navigation from QR Codes

**Components needed:**
1. **DeepLinkHandler** — `@Observable` singleton to parse URLs and route navigation
2. **onOpenURL** — Handle incoming URLs in `ApiLogRootView`
3. **NavigationPath** — Enable programmatic navigation with `NavigationStack(path: $path)`
4. **List Views** — Must have `.navigationDestination(for: Entity.self)` modifier

**Key Pattern:**
```swift
// 1. Parse URL and set pending destination
.onOpenURL { url in
    let dest = DeepLinkHandler.shared.handleURL(url)
    DeepLinkHandler.shared.setPendingDestination(dest)
}

// 2. React to pending destination in HomeView
.onChange(of: deepLinkHandler.isProcessingDeepLink) { _, isProcessing in
    if isProcessing, let dest = deepLinkHandler.pendingDestination {
        handleDeepLink(dest)
    }
}

// 3. Programmatically navigate
private func handleDeepLink(_ dest: DeepLinkDestination) {
    switch dest {
    case .hive(let id, _):
        tab = .hives
        if let hive = findHive(byId: id) { hivesPath.append(hive) }
    }
    deepLinkHandler.clearPendingDestination()
}
```

**Critical:** Verify entity exists before pushing to navigation path — prevents crashes if data isn't loaded yet.

### Known Patterns — NavigationStack navigationDestination Placement

**`.navigationDestination(for:destination:)` must be inside NavigationStack content, not on NavigationStack itself.**

```swift
// ❌ Wrong - causes "misplaced navigationDestination" warning
NavigationStack(path: $hivesPath) {
    HiveListView()
}
.navigationDestination(for: Hive.self) { hive in  // Ignored!
    HiveDetailView(hive: hive)
}

// ✅ Correct - modifier on the view inside NavigationStack
NavigationStack(path: $hivesPath) {
    HiveListView()
        .navigationDestination(for: Hive.self) { hive in
            HiveDetailView(hive: hive)
        }
}
```

### Known Patterns — Navigation Title Display

**Navigation title may appear in wrong position without `.navigationBarTitleDisplayMode()`.**

```swift
// ❌ Title may overlap content or appear misplaced
HiveDetailView(hive: hive)
    .navigationTitle(hive.name)

// ✅ Title appears correctly in navigation bar
HiveDetailView(hive: hive)
    .navigationTitle(hive.name)
    .navigationBarTitleDisplayMode(.inline)
```

**Also remove `.toolbar(.hidden, for: .navigationBar)`** if you want the navigation bar visible on detail views:
```swift
// ❌ Hides navigation bar for entire stack
NavigationStack(path: $hivesPath) {
    HiveListView()
        .toolbar(.hidden, for: .navigationBar)  // Don't do this
}

// ✅ Let detail views show navigation bar
NavigationStack(path: $hivesPath) {
    HiveListView()
        .navigationDestination(for: Hive.self) { hive in
            HiveDetailView(hive: hive)
                .navigationTitle(hive.name)
                .navigationBarTitleDisplayMode(.inline)
        }
}
```

### Known Patterns — Logging

Skip Tools projects use a **shared global `logger`** declared once (e.g. in `FirestoreHelpers.swift` or `ApiLogApp.swift`). Do **not** use `print()` or import `OSLog` directly in individual files.

**Exception:** In `#if SKIP` blocks (transpiled to Kotlin), the Swift `logger` is inaccessible due to different visibility boundaries. Use Android's `android.util.Log` directly:

```swift
// ❌ In #if SKIP block — inaccessible
logger.info("Something happened")

// ✅ In #if SKIP block — use Android Log
import android.util.Log
Log.d("Tag", "Debug message")
Log.e("Tag", "Error: \(error)")
```

```swift
// ✅ Declared once per module (already exists — do not redeclare)
let logger: Logger = Logger(subsystem: "com.floatingaxeheadministries.apilog", category: "ApiLog")

// ✅ Use in any file — logger is a global, no import needed
logger.info("Something happened")
logger.error("Something failed: \(error.localizedDescription)")

// ❌ Do not use
print("Something happened")
import OSLog  // in individual files — already handled by the shared declaration
```

### Known Patterns — ContentComposer (Android-only Compose bridge)

**`@Binding` on `ContentComposer` struct** is not supported — Skip can't synthesize `_propertyName` backing storage on a Kotlin class. Use plain `let` closure properties instead (same as `MapComposer` pattern).

```swift
// ❌
struct MyComposer: ContentComposer {
    @Binding var result: MyResult
}

// ✅
struct MyComposer: ContentComposer {
    let onResult: (String, String) -> Void
    let onCancel: () -> Void

    init(onResult: @escaping (String, String) -> Void, onCancel: @escaping () -> Void) {
        self.onResult = onResult
        self.onCancel = onCancel
    }
}
```

**Custom Swift structs are not bridgeable across the Compose boundary.** Decompose into primitive `let` closure parameters or individual `@State` vars on the SwiftUI side.

**`LocalContext.current` inside a launcher callback** — it's a `@Composable` call and cannot appear in non-composable lambdas. Capture it at the top of `Compose()` before creating the launcher.

```swift
// ❌
let launcher = rememberLauncherForActivityResult(...) { uri in
    let ctx = androidx.compose.ui.platform.LocalContext.current  // crash
}

// ✅
@Composable override func Compose(context: ComposeContext) {
    let ctx = androidx.compose.ui.platform.LocalContext.current  // captured first
    let launcher = rememberLauncherForActivityResult(...) { uri in
        let cr = ctx.contentResolver  // safe
    }
}
```

**Swift array literal → wrong Kotlin array type for `ContentResolver.query`.** `[item]` transpiles to `skip.lib.Array`; use `kotlin.arrayOfNulls<String>(n)` + index assignment.

```swift
// ❌
contentResolver.query(uri, nil, "COL = ?", [contactId], nil)

// ✅
let selectionArgs: kotlin.Array<String?> = kotlin.arrayOfNulls<String>(1)
selectionArgs[0] = contactId
contentResolver.query(uri, nil, "COL = ?", selectionArgs, nil)
```

**`launcher.launch(null)` → type mismatches.** `ActivityResultContracts.PickContact` expects `java.lang.Void?`. Skip maps Swift `Void` to `kotlin.Unit` (not `java.lang.Void`), so none of `null`, `kotlin.Unit`, or `nil as Void?` work. Use `nil as java.lang.Void?`.

```swift
// ❌
launcher.launch(null)                // null_ — unresolved
launcher.launch(kotlin.Unit)         // Unit vs Void?
launcher.launch(nil as Void?)        // Unit? vs Void?

// ✅
launcher.launch(nil as java.lang.Void?)
```

### Known Patterns — SkipSQL / SQLite

**Empty SQLite transactions cause `SQLError error 1`** — SkipSQL on iOS doesn't like empty transactions (BEGIN/COMMIT with no operations). When Firestore sync returns 0 documents, `saveBatch([])` runs an empty transaction and crashes.

```swift
// ❌
public func saveBatch(_ items: [LocalTask]) throws {
    try context.exec(sql: "BEGIN TRANSACTION", parameters: [])
    do {
        for item in items { try save(item) }  // empty when items.isEmpty
        try context.exec(sql: "COMMIT", parameters: [])
    } catch {
        try context.exec(sql: "ROLLBACK", parameters: [])
        throw error
    }
}

// ✅
public func saveBatch(_ items: [LocalTask]) throws {
    guard !items.isEmpty else { return }  // Skip transaction for empty arrays
    try context.exec(sql: "BEGIN TRANSACTION", parameters: [])
    do {
        for item in items { try save(item) }
        try context.exec(sql: "COMMIT", parameters: [])
    } catch {
        try context.exec(sql: "ROLLBACK", parameters: [])
        throw error
    }
}
```

**Actor-isolated DAO methods require `await`** — DAO actors in SkipSQL require `await` for all method calls, even synchronous-looking ones like `fetchAll`. Missing `await` causes "Actor-isolated instance method cannot be called from outside of the actor" error.

```swift
// ❌
let result = try dao.fetchAll(userId: userId)

// ✅
let result = try await dao.fetchAll(userId: userId)
```

### Known Patterns — Firebase Cloud Functions v2 IAM (Android data missing / crash on write)

**Problem:** Firebase Functions v2 runs on Cloud Run. Even with `invoker: 'public'` in the `onCall` config, newly deployed functions are **not publicly reachable** until the `allUsers` Cloud Run IAM binding is manually applied. On Android, all Firestore reads and writes route through callable Cloud Functions. Without the IAM binding, every call returns 401/403 — the Android app shows **no data after sign-in** and **crashes on any write**.

**Symptoms:** No data on Android after sign-in; crash on create. iOS unaffected (uses direct Firestore SDK). `firebase functions:list` may not show the new functions even when deployed.

**Fix:** After every `firebase deploy --only functions`, grant `roles/run.invoker` to `allUsers` for each new callable function:

```bash
# Get exact Cloud Run service names (lowercase, no separators)
gcloud run services list --region=us-central1 --project=<project-id> --format="value(metadata.name)"

# Grant invoker to each new function (repeat for all new callables)
gcloud run services add-iam-policy-binding <servicename> \
  --member="allUsers" --role="roles/run.invoker" \
  --region=us-central1 --project=<project-id>
```

**Note:** Cloud Run service names are the export name in all-lowercase, no separators (e.g. `exports.beekeeperGetCollection` → `beekeepergetcollection`). Firebase Auth via `requireAuth()` still enforces user auth inside the function.

**Checklist after any new callable function:**
- [ ] `firebase deploy --only functions`
- [ ] `gcloud run services add-iam-policy-binding <servicename> --member="allUsers" --role="roles/run.invoker" --region=us-central1 --project=<project-id>`

### Known Patterns — Async / Concurrency

**Unbounded async calls cause app lockup** — Async network calls without timeouts can hang indefinitely, locking up Xcode and the app. Always add timeouts using `withTaskGroup` for async operations that could hang.

```swift
// ❌ - Can hang forever
let config = await AppUpdateService.shared.fetchVersionConfig()

// ✅ - 10-second timeout prevents hanging
let config = await withTaskGroup(of: AppVersionConfig?.self) { group in
    group.addTask {
        return await AppUpdateService.shared.fetchVersionConfig()
    }
    group.addTask {
        // Timeout after 10 seconds
        try? await Task.sleep(nanoseconds: 10_000_000_000)
        return nil as AppVersionConfig?
    }
    
    // Return first completed task (either config or timeout)
    let firstResult = await group.next()!
    group.cancelAll()
    return firstResult
}
```

---

### Known Patterns — `.contentShape` Not Available

**Problem:** Using `.contentShape()` on views causes "value of type 'some View' has no member 'contentShape'" error. This modifier is not available in Skip.

**Fix:** Remove `.contentShape()` modifiers. For button tap areas, use `.frame()` to provide sufficient tap area instead.

```swift
// ❌ ERROR — contentShape not available in Skip
Button(action: { toggle() }) {
    Image(systemName: "eye.fill")
        .frame(width: 44, height: 44)
        .contentShape(Rectangle())
}

// ✅ Correct — frame provides sufficient tap area
Button(action: { toggle() }) {
    Image(systemName: "eye.fill")
        .frame(width: 44, height: 44)
}
```

---

### Known Patterns — Nested Buttons Block Navigation

**Problem:** Creating a reusable view component that contains its own `Button` with an empty action, then wrapping it in a `NavigationLink` or another `Button`, causes tap interception issues. The inner Button's empty action blocks the outer navigation/action from firing.

**Fix:** Make reusable view components plain views without their own Button wrapper. Apply the Button/NavigationLink at the call site.

```swift
// ❌ WRONG — Inner Button intercepts taps, NavigationLink never fires
struct QuickActionButton: View {
    let icon: String
    let title: String
    
    var body: some View {
        Button(action: {}) {  // Empty action blocks taps!
            VStack {
                Image(systemName: icon)
                Text(title)
            }
        }
    }
}

// Usage — this won't navigate!
NavigationLink(destination: EventsTabView()) {
    QuickActionButton(icon: "calendar", title: "Events")
}

// ✅ CORRECT — Plain view component, Button applied at call site
struct QuickActionButton: View {
    let icon: String
    let title: String
    
    var body: some View {
        VStack {
            Image(systemName: icon)
            Text(title)
        }
        .padding()
        .background(Color.white)
        .cornerRadius(12)
    }
}

// Usage — NavigationLink wrapper handles the tap
NavigationLink(destination: EventsTabView()) {
    QuickActionButton(icon: "calendar", title: "Events")
}

// Or for actions:
Button { showDonation = true } label: {
    QuickActionButton(icon: "heart", title: "Give")
}
```

---

### Known Patterns — Swift 6 Data Race: Sending Non-Sendable Values into `async let`

**Problem:** In Swift 6 strict concurrency, using a `@MainActor`-isolated non-`Sendable` value (e.g. `Timestamp`, `[String: Any]`) directly inside `async let` task initializers causes `"sending 'x' risks causing data races"` errors. The value is captured by a concurrently-executing child task before it is safe to share.

**Fix:** Build all queries/values synchronously on the `@MainActor` before any suspension point, then `await` each result sequentially.

```swift
// ❌ ERROR — startTs (Timestamp) sent into concurrent async let tasks
let startTs = Timestamp(date: cutoff)
async let eventsSnap = db.collection("events").whereField("date", isGreaterThanOrEqualTo: startTs).getDocuments()
async let sermonsSnap = db.collection("sermons").whereField("date", isGreaterThanOrEqualTo: startTs).getDocuments()
let (ev, se) = try await (eventsSnap, sermonsSnap)

// ✅ Build queries synchronously first, then await sequentially
let startTs = Timestamp(date: cutoff)
let eventsQuery = db.collection("events").whereField("date", isGreaterThanOrEqualTo: startTs)
let sermonsQuery = db.collection("sermons").whereField("date", isGreaterThanOrEqualTo: startTs)
let ev = try await eventsQuery.getDocuments()
let se = try await sermonsQuery.getDocuments()
```

---

### Known Patterns — ScrollView Blocks Payment Sheet Presentation

**Problem:** Stripe PaymentSheet button nested inside a ScrollView can fail to present on iOS. The ScrollView's gesture handling can interfere with the PaymentSheet's presentation controller.

**Fix:** Move the PaymentSheet button outside the ScrollView, placing it in a fixed bottom area.

```swift
// ❌ WRONG — PaymentSheet button inside ScrollView may not present
ScrollView {
    VStack {
        // ... form content ...
        SimpleStripePaymentButton(...)
    }
}

// ✅ CORRECT — Payment button outside ScrollView
VStack(spacing: 0) {
    ScrollView {
        VStack {
            // ... form content ...
        }
    }
    
    // Fixed bottom area for payment
    SimpleStripePaymentButton(...)
        .padding()
}
```

---

### Known Patterns — Use SkipFuseUI Instead of SwiftUI

**Problem:** Using `import SwiftUI` causes build errors on Android since Skip Tools projects use `SkipFuseUI` as the cross-platform UI framework. `import SwiftUI` is iOS-only and won't transpile to Android.

**Fix:** Replace all `import SwiftUI` with `import SkipFuseUI` in all View files.

```swift
// ❌ ERROR — SwiftUI is not available in Skip cross-platform builds
import SwiftUI

struct MyView: View {
    var body: some View {
        Text("Hello")
    }
}

// ✅ CORRECT — Use SkipFuseUI for cross-platform compatibility
import SkipFuseUI

struct MyView: View {
    var body: some View {
        Text("Hello")
    }
}
```

**Quick fix command:**
```bash
cd mobile-apps/member/Sources/ChurchCompass
sed -i '' 's/import SwiftUI/import SkipFuseUI/g' *.swift
```

---

### Known Patterns — @Observable UI Refresh Issues

**Problem:** When using `@Observable` classes with SkipFuseUI, state changes may not trigger immediate UI updates. This is particularly noticeable when conditionally showing UI elements based on `@Observable` properties. The UI requires a second interaction (tap, scroll, etc.) to refresh and show the updated state.

**Fix:** Use a `refreshID` state variable combined with `.id()` modifier and `.onChange()` to force immediate view updates when `@Observable` state changes.

```swift
// ❌ PROBLEMATIC — UI doesn't refresh when showPaymentSheet changes
struct MyView: View {
    @State var viewModel: MyViewModel  // @Observable class
    
    var body: some View {
        VStack {
            if viewModel.showPaymentSheet {
                PaymentButton()  // Doesn't appear immediately
            }
        }
    }
}

// ✅ CORRECT — Force UI refresh on state change
struct MyView: View {
    @State var viewModel: MyViewModel  // @Observable class
    @State var refreshID = UUID()      // Forces view rebuild
    
    var body: some View {
        VStack {
            if viewModel.showPaymentSheet {
                PaymentButton()
            }
        }
        .id(refreshID)  // Apply unique ID to force rebuild
        .onChange(of: viewModel.showPaymentSheet) { _, _ in
            refreshID = UUID()  // Update ID when state changes
        }
    }
}
```

**Key Points:**
- `@Observable` macro requires `import Observation` before `import SkipFuseUI`
- State changes in `@Observable` classes don't always trigger SwiftUI view updates in Skip
- Use `.id(refreshID)` on the container view to force complete rebuild
- Use `.onChange(of:)` to detect state changes and update the refresh ID

---

### Known Patterns — Shadow on Interactive Cards Blocks Taps on Android

**Problem:** Adding `.shadow()` to an interactive card container (a `VStack` or `RoundedRectangle` that contains `Button` or `NavigationLink`) blocks all gesture handling below the shadow layer on Android. Buttons inside the card become untappable.

**Fix:** Remove `.shadow()` from interactive card containers. Use border or background differentiation instead.

```swift
// ❌ WRONG — shadow blocks button taps inside the card on Android
VStack { Button { ... } label: { ... } }
    .background(Color.white)
    .cornerRadius(12)
    .shadow(color: .black.opacity(0.1), radius: 4, x: 0, y: 2)  // ← blocks taps

// ✅ CORRECT — no shadow on interactive containers
VStack { Button { ... } label: { ... } }
    .background(Color.white)
    .cornerRadius(12)
    .overlay(RoundedRectangle(cornerRadius: 12).stroke(Color.gray.opacity(0.2)))
```

---

### Known Patterns — Toggle in Forms Blocks Gestures Below on Android

**Problem:** A `Toggle` placed inside a `Form` or `VStack` with other interactive controls (Buttons, TextFields) below it causes Android to block all gesture handling for everything beneath the Toggle.

**Fix:** Avoid placing `Toggle` in forms with buttons below it. If needed, restructure layout so Toggle is at the bottom, or use a `Button`-style boolean row instead.

```swift
// ❌ WRONG — buttons below Toggle are untappable on Android
Form {
    Toggle("Remember me", isOn: $rememberMe)
    Button("Sign In") { signIn() }   // ← unreachable on Android
}

// ✅ CORRECT — Toggle at bottom, or replace with a tap-row
Form {
    Button("Sign In") { signIn() }
    Toggle("Remember me", isOn: $rememberMe)  // Toggle last
}
```

---

### Known Patterns — Firestore Timestamps Decode as `Double` on Android

**Problem:** On Android, Firestore timestamps arrive as `Double` (epoch milliseconds), not as `Timestamp` objects. Casting directly to `Timestamp` returns `nil`, silently dropping all events/documents.

**Fix:** Branch on platform to decode timestamps:

```swift
// ❌ WRONG — fails silently on Android (returns nil)
guard let startTimestamp = data["startDate"] as? Timestamp else { return nil }
self.startDate = startTimestamp.dateValue()

// ✅ CORRECT — handle both Timestamp (iOS) and Double (Android)
func dateFromFirestore(_ value: Any?) -> Date? {
    if let ts = value as? Timestamp { return ts.dateValue() }
    if let ms = value as? Double { return Date(timeIntervalSince1970: ms / 1000.0) }
    return nil
}

guard let startDate = dateFromFirestore(data["startDate"]) else { return nil }
self.startDate = startDate

// Or inline with #if os(Android):
#if os(Android)
if let ms = data["startDate"] as? Double {
    self.startDate = Date(timeIntervalSince1970: ms / 1000.0)
} else { return nil }
#else
guard let ts = data["startDate"] as? Timestamp else { return nil }
self.startDate = ts.dateValue()
#endif
```

---

### Known Patterns — Firestore Integer Fields Decode as `Double` on Android

**Problem:** On Android, Firestore integer fields (e.g., `memberCount`, `paidMonths`) decode as `Double` (e.g., `6.0`), not as `Int`. Using `data["field"] as? Int` returns `nil` on Android, silently producing `0`.

**Fix:** Always decode with a `Double` fallback:

```swift
// ❌ WRONG — returns 0 on Android
self.memberCount = data["memberCount"] as? Int ?? 0

// ✅ CORRECT — handles Int (iOS) and Double (Android)
if let intVal = data["memberCount"] as? Int {
    self.memberCount = intVal
} else if let doubleVal = data["memberCount"] as? Double {
    self.memberCount = Int(doubleVal)
} else {
    self.memberCount = 0
}
```

---

### Known Patterns — `[String: Any]` Must Not Cross Async Suspension Points

**Problem:** Swift 6 strict concurrency: `[String: Any]` is not `Sendable`. Passing it as an argument to an `async` function produces:
```
Sending value of non-Sendable type '[String : Any]' risks causing data races
Sending 'payload' risks causing data races
```
`nonisolated(unsafe)` on a local copy does **not** suppress the warning at the call site.

**Fix:** Serialize to `Data` (Sendable) synchronously **before** the first `await`:

```swift
// ❌ WRONG — [String: Any] passed as argument across async boundary
private func send(_ name: String, payload: [String: Any]) async throws {
    let idToken = try await user.getIDToken()   // ← first await
    try await upload(payload: payload, ...)     // ← Sendable warning here
}

// ✅ CORRECT — serialize synchronously before any await
private func send(_ name: String, payload: [String: Any]) async throws {
    let bodyData = try JSONSerialization.data(withJSONObject: payload)  // ← before any await
    let idToken = try await user.getIDToken()
    try await upload(bodyData: bodyData, ...)   // Data is Sendable ✓
}
```

---

### Known Patterns — Firebase Functions: Use Direct HTTP, Not `httpsCallable.call()`

**Problem:** `httpsCallable.call(payload)` where `payload` is `[String: Any]` causes a Sendable warning because `call(_:)` takes `Any?` (non-Sendable). `nonisolated(unsafe)` does NOT suppress this.

**Fix:** Use direct HTTP with ID token on both platforms. Serialize payload to `Data` before any `await`:

```swift
// Only FirebaseAuth needed — no FirebaseFunctions import
#if os(Android)
@preconcurrency import SkipFirebaseAuth
#else
@preconcurrency import FirebaseAuth
#endif

private func invokeCallable(_ name: String, payload: [String: Any]) async throws -> [String: Any] {
    guard let user = Auth.auth().currentUser else { throw ... }
    let bodyData = try JSONSerialization.data(withJSONObject: ["data": payload])  // before await
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

private func invokeHTTP(functionName: String, bodyData: Data, idToken: String) async throws -> [String: Any] {
    var request = URLRequest(url: URL(string: "https://us-central1-PROJECT.cloudfunctions.net")!.appendingPathComponent(functionName))
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.setValue("Bearer \(idToken)", forHTTPHeaderField: "Authorization")
    request.httpBody = bodyData
    let (data, _) = try await URLSession.shared.data(for: request)
    guard let json = try JSONSerialization.jsonObject(with: data) as? [String: Any] else { throw ... }
    return json["result"] as? [String: Any] ?? json
}
```

---

### Known Patterns — `LaunchedEffect` Requires Explicit Import in `#if SKIP`

**Problem:** Using `LaunchedEffect` inside a `ContentComposer`'s `@Composable func Compose()` causes "Unresolved reference 'LaunchedEffect'" even though it's a standard Compose runtime API.

**Fix:** Explicitly import `LaunchedEffect` from `androidx.compose.runtime` inside the `#if SKIP` block:

```swift
// ❌ ERROR — LaunchedEffect unresolved
#if SKIP
import android.content.Intent

struct MyComposer: ContentComposer {
    @Composable func Compose(context: ComposeContext) {
        LaunchedEffect(trigger) { ... }  // ← Unresolved reference
    }
}
#endif

// ✅ CORRECT — import LaunchedEffect explicitly
#if SKIP
import android.content.Intent
import androidx.compose.runtime.LaunchedEffect  // ← required

struct MyComposer: ContentComposer {
    @Composable func Compose(context: ComposeContext) {
        LaunchedEffect(trigger) { ... }  // ✓
    }
}
#endif
```

---

### Known Patterns — Sendable Closure Capture (Swift 6)

**Problem:** Swift 6 strict concurrency produces "Main actor-isolated property 'x' can not be referenced from a Sendable closure" when accessing `@MainActor`-isolated properties (like `@State` vars) from within Sendable closures like KVO observers (`observe(_:options:changeHandler:)`) or async callbacks.

**Fix:** Capture the MainActor-isolated value into a local `let` **before** the Sendable closure. The local `let` is not actor-isolated and can be safely captured by the closure.

```swift
// ❌ ERROR — MainActor-isolated property referenced inside Sendable closure
let statusObs = item.observe(\.status) { item, _ in
    DispatchQueue.main.async {
        let rate = playbackRate  // ERROR: Main actor-isolated property in Sendable closure
        p.rate = rate
    }
}

// ✅ CORRECT — Capture before the Sendable closure
let rate = playbackRate  // Local let, not actor-isolated
let statusObs = item.observe(\.status) { item, _ in
    DispatchQueue.main.async {
        p.rate = rate  // OK: captures local 'let', not MainActor property
    }
}
```

**Applies to:** KVO observers, `async let` task initializers, `@Sendable` closures, and any closure that crosses actor boundaries.

---

### Known Patterns — Swift 6 Firebase Closure Pattern (CRITICAL)

**Problem:** Swift 6 strict concurrency produces "Passing closure as a 'sending' parameter risks causing data races" and "Sending 'self' risks causing data races" when using Firebase closure-based APIs from `@Observable` ViewModels.

**The Definitive Fix:** Add `@MainActor` annotation to all `@Observable` ViewModels:

```swift
// ❌ WRONG — Missing @MainActor causes warnings
@Observable
final class MyViewModel {
    func fetch() {
        db.collection("users").getDocument { [weak self] _, _ in
            Task { @MainActor in
                self?.isLoading = false  // ⚠️ Warning
            }
        }
    }
}

// ✅ CORRECT — Add @MainActor annotation
@MainActor
@Observable
final class MyViewModel {
    func fetch() {
        db.collection("users").getDocument { [weak self] _, _ in
            guard let self = self else { return }
            Task {
                await MainActor.run {
                    self.isLoading = false  // ✅ No warning
                }
            }
        }
    }
}
```

**Key Rules:**
1. **Always add `@MainActor`** to `@Observable` ViewModels that update UI state
2. Use `guard let self = self else { return }` directly in the closure
3. Use `Task { await MainActor.run { } }` to dispatch back to main thread
4. Use `self` directly (not `self?`) after the guard statement
5. **Never use deinit** — use `stopListening()` called from `.onDisappear`:

```swift
@MainActor
@Observable
final class MyViewModel {
    private var listener: ListenerRegistration? = nil
    
    func stopListening() {
        listener?.remove()
        listener = nil
    }
}

struct MyView: View {
    @State var viewModel = MyViewModel()
    
    var body: some View {
        List { ... }
        .onAppear { viewModel.fetch() }
        .onDisappear { viewModel.stopListening() }  // ✅ Correct cleanup
    }
}

---

### Known Patterns — Swift 6 Firebase Closure in Views (CRITICAL)

**Problem:** When a **SwiftUI View** (not a ViewModel) directly uses Firebase closure-based APIs and mutates `@State` properties, Swift 6 produces "Main actor-isolated property can not be mutated from a Sendable closure".

**The Definitive Fix:** Extract values locally in the closure, then dispatch to MainActor:

```swift
// ❌ WRONG — Mutating @State directly from Firebase closure
struct MyView: View {
    @State var showName = true
    
    private func loadSettings() {
        db.collection("users").document(uid).getDocument { snapshot, error in
            guard let data = snapshot?.data() else { return }
            showName = data["showName"] as? Bool ?? true  // ⚠️ Error
        }
    }
}

// ✅ CORRECT — Extract values locally, dispatch to MainActor
struct MyView: View {
    @State var showName = true
    
    private func loadSettings() {
        db.collection("users").document(uid).getDocument { snapshot, error in
            guard let data = snapshot?.data() else { return }
            
            let showNameValue = data["showName"] as? Bool ?? true
            
            Task { @MainActor in
                showName = showNameValue
            }
        }
    }
}
```

**Key Rules:**
1. **Never mutate `@State` directly** inside a Firebase closure
2. **Extract all values** into local `let` constants inside the closure
3. **Use `Task { @MainActor in }`** to dispatch back to MainActor for @State mutation
4. **Prefer moving Firebase logic to a `@MainActor @Observable` ViewModel** — views should not directly call Firebase

---

### Known Patterns — Android content:// URI Data Reading (CRITICAL)

**Problem:** `Data(contentsOf: url)` crashes on Android with `NSCocoaErrorDomain` error 256 when `url.scheme == "content"` (photo picker URIs). Skip Foundation does not bridge to Android's `ContentResolver`.

**Fix (no compression needed):** Use `putFileAsync(from: url)` directly — Firebase Storage handles content:// URIs natively.

**Fix (compression needed — PREFERRED):** Write a Kotlin helper class in the Android project, call it via a single JNI call that returns a `String` file path, upload the temp file with `putFileAsync`.

```swift
// ❌ WRONG — crashes on Android photo picker URIs
let data = try Data(contentsOf: url)

// ✅ CORRECT (no compression) — putFileAsync handles content:// natively
_ = try await imageRef.putFileAsync(from: url)

// ✅ CORRECT (with compression) — Kotlin helper + putFileAsync
// In ImageCompression.swift (Android section):
func compressAndroidContentURI(_ url: URL) throws -> String {
    let helperClass = try AnyDynamicObject(forStaticsOfClassName: "com.example.ImageCompressorHelper")
    let filePath: String? = try helperClass.compress(url.absoluteString)
    guard let filePath else { throw ImageCompressError.cannotCompress }
    return filePath
}

// In upload ViewModel:
#if os(Android)
let filePath = try compressAndroidContentURI(url)
let fileURL = URL(fileURLWithPath: filePath)
defer { try? FileManager.default.removeItem(at: fileURL) }
let metadata = StorageMetadata(); metadata.contentType = "image/jpeg"
_ = try await imageRef.putFileAsync(from: fileURL, metadata: metadata)
return try await imageRef.downloadURL()
#endif
```

**Key Rules:**
1. Any `URL` from an Android image picker is a `content://` URI
2. `Data(contentsOf:)` only works with `file://` paths on Android
3. Firebase Storage's `putFileAsync(from:)` natively handles `content://` URIs
4. For compression: write a Kotlin `companion object` with `@JvmStatic` methods, call via `AnyDynamicObject`, return `String?` (file path) — `String` return has NO overload ambiguity
5. The Kotlin helper must be initialized with `Context` at app startup in `AndroidAppMain.onCreate()`
6. **See:** `skip-cross-platform-architecture.md` → "Android ContentResolver & JNI Bridging" → Pattern 3 for full Kotlin helper source
7. **See:** `docs/image-upload-cross-platform.md` for full architecture and framework design

---

### Known Patterns — Logging from Swift on Android

**Problem:** `print()` in Swift does NOT appear in `adb logcat`. When debugging Android-specific code, print statements are silently dropped.

**Fix:** Use `OSLog.Logger` from `swift-android-native`'s `AndroidLogging` module. `Logger.log()` calls `__android_log_write` which appears in `adb logcat` at INFO level.

```swift
// ❌ WRONG — print() is invisible in adb logcat on Android
print("[DEBUG] got contentResolver")

// ✅ CORRECT — Logger.log() appears in adb logcat under tag "Subsystem/Category"
import OSLog
private let logger = Logger(subsystem: "MyApp", category: "ImageUpload")
logger.log("📸 got contentResolver")   // visible: adb logcat | grep "MyApp/ImageUpload"
```

**Key Rules:**
1. `logger.log()` → `ANDROID_LOG_INFO` — always visible in logcat
2. `logger.debug()` → `ANDROID_LOG_DEBUG` — may be filtered in some configurations; prefer `.log()`
3. The logcat tag is `subsystem/category` — e.g., `MyApp/ImageUpload`
4. Emoji prefixes help grep: `adb logcat | grep "📸"`
5. `print()` goes to stdout which is NOT captured by logcat in native Android builds

---

### Quick Reference — Additional Skip Unavailable APIs (#41–45)

| API | Replacement |
|-----|-------------|
| `RelativeDateTimeFormatter` | `DateFormatter` with `.dateStyle` + `.timeStyle` |
| `getDocuments { closure }` | `try await .getDocuments()` — closure form hits wrong overload |
| `DispatchGroup` | Sequential `async/await` loop inside `Task { }` |
| `.listStyle(.insetGrouped)` | `.listStyle(.plain)` |
| `.searchable(placement: .navigationBarDrawer(...))` | `.searchable(text:prompt:)` — omit `placement:` |
| `systemImage:` any new icon | Check `Module.xcassets` — if missing, create `.symbolset/Contents.json` + add to `docs/sf-symbols-tracker.md` |
| `SkipDevice.LocationProvider` (iOS) | Call `PermissionManager.requestLocationPermission()` FIRST; then call `fetchCurrentLocation()` inside `Task { @MainActor in }` — NOT inside `withTaskGroup` child task (CLLocationManager requires main thread run loop); use `@preconcurrency import SkipDevice` |
| Firestore field mismatch | Verify read field names match write path — `primaryChurchId` was written but `churchId` was read |
| `Data(contentsOf: contentURI)` on Android | Branch on `url.scheme == "content"` — use `putFileAsync(from:)` or read via `ContentResolver` through JNI |

**Firestore async/await rule:** In `@MainActor` ViewModels, ALWAYS use `try await` form for all Firestore queries. Never use completion handler form — Skip's overload resolution picks the wrong `FirestoreSource` overload.

```swift
// ❌ WRONG in any @MainActor ViewModel
db.collection("items").getDocuments { snapshot, error in ... }

// ✅ CORRECT
Task {
    do {
        let snapshot = try await db.collection("items").getDocuments()
        items = snapshot.documents.compactMap { Item(from: $0) }
    } catch { }
}
```

---

### Known Patterns — Stale SQLite Cache with External URLs (Sermons, Thumbnails)

**Problem:** A SQLite cache with checkpoint-based staleness (`maxAge`) never refreshes if the data was written before a new field existed, or if `maxAge` is generous. For collections like `sermons` that store `videoUrl` and `thumbnail`, the UI silently shows no thumbnails and broken links.

**Fix:** Fetch directly from Firestore for small metadata collections that contain external URLs. Network payload is tiny (~KBs) and URLs must stay current.

```swift
// ❌ WRONG — stale cache holds empty videoUrl/thumbnail
let cached = try await SermonRepository.shared.getAll(churchId: churchId)
sermons = cached.map { ChurchSermon(from: $0) }

// ✅ CORRECT — source of truth from Firestore
let db = Firestore.firestore()
let batch = try await fetchChurchCollection(db: db, churchId: churchId, collection: "sermons")
sermons = batch.docs.compactMap { doc -> ChurchSermon? in
    guard let data = doc.data() else { return nil }
    return ChurchSermon(id: doc.documentID, data: data)
}.sorted { $0.date > $1.date }
```

**Key Rules:**
1. Direct Firestore fetch for URL-bearing collections
2. SQLite cache reserved for large/heavy or offline-critical data only
3. Thumbnails: prefer stored `thumbnail` field, then derive from YouTube ID

---

### Known Patterns — iOS LocationProvider / CLLocationManager Must Run on @MainActor

**Problem:** `SkipDevice.LocationProvider.fetchCurrentLocation()` wraps `CLLocationManager` on iOS. `CLLocationManager` dispatches delegate callbacks on the **main thread's run loop**. Calling `fetchCurrentLocation()` from a `withTaskGroup` child task or a `nonisolated` function executes it on a background thread with no run loop — the delegate callback fires but is never received. Result: the call hangs indefinitely until a timeout fires.

**Symptoms in console:**
```
fetchCurrentLocation        ← SkipDevice's own log — function entered
[Location] Task: timeout fired   ← 8+ seconds later, no location ever returned
```

**Fix:** Call `fetchCurrentLocation()` inside `Task { @MainActor in }`, which runs on the main actor's always-active run loop. Use a separate cancellation `Task` for the timeout instead of `withTaskGroup`.

```swift
// ❌ WRONG — hangs; CLLocationManager delegate never fires on background thread
group.addTask {
    let provider = LocationProvider()
    let location = try await provider.fetchCurrentLocation()  // ← hangs forever
}

// ❌ ALSO WRONG — nonisolated static helper runs on background thread
nonisolated private static func fetchWithTimeout() async -> RaceResult {
    return await withTaskGroup(...) { group in
        group.addTask { try await provider.fetchCurrentLocation() }  // ← hangs
    }
}

// ✅ CORRECT — @MainActor preserves main thread run loop
// (in a @MainActor ViewModel func)
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
    // success: location.latitude, location.longitude
} catch is CancellationError {
    locationError = "Location timed out."
} catch {
    locationError = "Error: \(error.localizedDescription)"
}
```

**Import:** `LocationEvent` from SkipDevice is not `Sendable` — add `@preconcurrency`:
```swift
@preconcurrency import SkipDevice
```

**Dependency:** Verified working version is `skip-device 0.5.0`:
```swift
.package(url: "https://source.skip.tools/skip-device.git", from: "0.5.0")
```

---

### Known Patterns — `.lineLimit(ClosedRange)` Not Available in Skip

**Problem:** SwiftUI's `.lineLimit(_:)` accepts a `ClosedRange<Int>` (e.g., `.lineLimit(2...4)`) to set a dynamic line count range. Skip's `SkipFuseUI` does not bridge this overload — only the single `Int` overload is available.

**Error:**
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

---

### Known Patterns — `TextField(axis:)` Not Available in Skip

**Problem:** SwiftUI's `TextField` initializer with the `axis:` parameter (e.g., `TextField("Notes", text: $formNotes, axis: .vertical)`) is not available in Skip. Only the plain `TextField(_:text:)` overload is bridged to Android.

**Error:**
```
'init(_:text:axis:)' is unavailable
```

**Fix:** Remove the `axis:` parameter. Use `TextField(_:text:)` instead.

```swift
// ❌ ERROR — axis parameter not available in Skip
TextField("Notes", text: $formNotes, axis: .vertical)
    .lineLimit(4)

// ✅ CORRECT — plain TextField
TextField("Notes", text: $formNotes)
    .lineLimit(4)
```

**Applies to:** Any `TextField` using `axis: .vertical` or `axis: .horizontal`.

---

### Known Patterns — `UIApplication.applicationIconBadgeNumber` Deprecated in iOS 17

**Problem:** `UIApplication.shared.applicationIconBadgeNumber = 0` was deprecated in iOS 17.0. Using it from a `/* SKIP @bridge */` method (nonisolated) also triggers MainActor isolation warnings.

**Errors:**
```
'applicationIconBadgeNumber' was deprecated in iOS 17.0
Main actor-isolated class property 'shared' can not be mutated from a nonisolated context
Main actor-isolated property 'applicationIconBadgeNumber' can not be mutated from a nonisolated context
```

**Fix:** Replace with `UNUserNotificationCenter.current().setBadgeCount()` wrapped in `Task { @MainActor in }`.

```swift
// ❌ ERROR — deprecated + MainActor isolation
#if !os(Android)
UIApplication.shared.applicationIconBadgeNumber = 0
#endif

// ✅ CORRECT
#if !os(Android)
Task { @MainActor in
    UNUserNotificationCenter.current().setBadgeCount(0)
}
#endif
```

**Import required:** `import UserNotifications` in the `#if !os(Android)` block.

**Applies to:** Any `@bridge` lifecycle method that mutates UIKit state.

---

### Known Patterns — Swift 6 Sendable Closure Properties on View Structs

**Problem:** Swift 6 strict concurrency warns:
```
converting non-Sendable function value to '@MainActor @Sendable () -> Void' may introduce data races
```

This happens when a `View` struct has a `let` property holding a closure (e.g., `let onJoin: () -> Void`) and passes it to a `Button(action:)` or other SwiftUI control. Swift 6 infers the closure must cross actor boundaries because `View.body` is `@MainActor`, but the bare `() -> Void` type is not `Sendable`.

**Fix:** Annotate the property type with `@MainActor @Sendable`.

```swift
// ❌ WRONG — plain () -> Void on struct properties
struct ChurchSearchCard: View {
    let onJoin: () -> Void
    var body: some View {
        Button(action: onJoin) { ... }  // warning
    }
}

// ✅ CORRECT — annotate the property type
struct ChurchSearchCard: View {
    let onJoin: @MainActor @Sendable () -> Void
    var body: some View {
        Button(action: onJoin) { ... }
    }
}
```

**When to apply:** Any `View` struct that holds a closure property used as a `Button` action, `onTapGesture`, or passed to another control. Common names: `onJoin`, `onTap`, `onSelect`, `onDelete`, `onConfirm`, `action`.

**Quick grep to find violations:**
```bash
grep -r "let (on[A-Z]|action|onTap|onSelect|onDelete|onConfirm|onDismiss): \(\) -> Void" mobile-apps/compass/Sources/
```

---

### Known Patterns — Firestore Dictionary: `nonisolated(unsafe) let`

**Problem:** Using `var` for a `[String: Any]` Firestore dictionary that is never mutated triggers "variable was never mutated; consider changing to 'let' constant". Separately copying it to `nonisolated(unsafe) let` is redundant.

**Fix:** Declare the dictionary as `nonisolated(unsafe) let` directly:

```swift
// ❌
var updateData: [String: Any] = ["status": "removed", "leftAt": FieldValue.serverTimestamp()]
nonisolated(unsafe) let finalUpdate = updateData

// ✅
nonisolated(unsafe) let updateData: [String: Any] = [
    "status": "removed",
    "leftAt": FieldValue.serverTimestamp()
]
try await ref.updateData(updateData)
```

---

### Known Patterns — Button Tap Area Collapses on Android (style inside `label`)

**Problem:** `Button("Title") { }` with `.frame(maxWidth: .infinity).padding().background().cornerRadius()` applied to the Button has a tap area that collapses to the **text bounds** on Android — taps on the colored pill do nothing, so it needs several taps and feels broken (Google Play functionality rejection risk).

**Fix:** Use `Button { } label: { }` and move ALL layout/visual modifiers INSIDE the label. Do NOT use `.contentShape()` (banned).

```swift
// ❌
Button("Sign In") { Task { await signIn() } }
    .frame(maxWidth: .infinity).padding().background(Color.accentColor).cornerRadius(14)

// ✅
Button {
    Task { await signIn() }
} label: {
    Text("Sign In")
        .frame(maxWidth: .infinity).padding().background(Color.accentColor)
        .foregroundColor(.white).cornerRadius(14)
}
```

---

### Known Patterns — Firestore Reference Region Isolation (#SendingRisksDataRace)

**Problem:** Calling an `async` Firebase method on a `DocumentReference`/`Query`/`StorageReference` from a `@MainActor` ViewModel produces `sending '...' risks causing data races [#SendingRisksDataRace]`. Region isolation (SE-0414) ties a non-Sendable Firebase ref to the actor region when reached via (1) a `self.` property, (2) a local captured from an outer `@MainActor` scope, or (3) `doc.reference` from an awaited snapshot.

**Key distinction (Rule #67, updated):**
- **`DocumentReference`** IS `Sendable` — plain `let ref = db.collection(...).document(id)` is fine; `nonisolated(unsafe)` on it triggers "unnecessary" warning.
- **`Query`** IS also `Sendable` (confirmed in current SkipFirebaseFirestore) — plain `let q = db.collection(...).whereField(...)` is fine; `nonisolated(unsafe)` on it also triggers "unnecessary" warning.
- **Deeply-chained inline subcollection refs** (`db.collection.document.collection.document` used directly in `try await`) STILL trigger `#SendingRisksDataRace` — always assign to a `let` constant first.

```swift
// ✅ DocumentReference — Sendable, no annotation needed
let ref = db.collection("foo").document(id)
try await ref.setData(data, merge: true)

// ✅ Query — also Sendable, no annotation needed
let q = db.collection("foo")
    .whereField("status", isEqualTo: "pending")
let snap = try await q.getDocuments()

// ✅ Deep subcollection chain — assign to let first to avoid inline data race warning
let memberRef = db.collection("a").document(id).collection("b").document(id2)
try await memberRef.updateData([...])

// ✅ Alternative — nonisolated handle property disconnects both from actor region
private nonisolated var db: Firestore { Firestore.firestore() }

// ✅ Alternative — fresh local Firestore handle inside the async scope
Task {
    let db = Firestore.firestore()
    let snap = try await db.collection("users").whereField("churchId", isEqualTo: cid).getDocuments()
}

// ✅ Alternative — rebuild ref from doc.documentID (not doc.reference) after await
let col = Firestore.firestore().collection("churches").document(cid).collection("signups")
for doc in snap.documents { try? await col.document(doc.documentID).delete() }
```

---

### Known Patterns — iOS 18 TabView Sidebar Breaks Navigation on iPad

**Problem:** iOS 18 changed the default `TabView` style on iPad to a sidebar/top-bar layout. This breaks `NavigationStack` inside tabs — tabs appear at the top of the screen, taps on tabs hang, and `NavigationLink` quick actions stop responding entirely. Affects Skip Fuse apps because the iOS binary runs natively on iPad.

**Fix:** Extract tabs into a `@ViewBuilder` property and apply `.tabViewStyle(.tabBarOnly)` conditionally on iOS 18+. Wrap in `#if !os(Android)` so Skip never sees the iOS 18 API.

```swift
// ❌ — works on iPhone but hangs on iPad running iOS 18+
var body: some View {
    TabView(selection: $selectedTab) { ... }
}

// ✅ — forces traditional bottom tab bar on iPad
var body: some View {
    #if !os(Android)
    if #available(iOS 18.0, *) {
        tabContent.tabViewStyle(.tabBarOnly)
    } else {
        tabContent
    }
    #else
    tabContent
    #endif
}

@ViewBuilder
var tabContent: some View {
    TabView(selection: $selectedTab) {
        // ... tabs with NavigationStack
    }
}
```

**Note:** `.tabBarOnly` is iOS 18+ only — the `if #available` guard is required. The `#if !os(Android)` wrapper prevents Skip from transpiling the iOS 18 API check.

---

### Known Patterns — Firebase Callable Response: All Numbers Are `Double` on Android

**Problem:** Firebase callable function responses decode all JSON numbers as `Double` on Android, not `Int`. Any `as? Int` cast silently returns `nil` and falls through to the `?? 0` default, causing every numeric field to show as zero. iOS returns `Int` natively so the bug is Android-only.

**Fix:** Use a helper that tries `as? Int` first, then falls back to `Int(as? Double ?? 0)`. Apply to all numeric fields decoded from callable response payloads.

```swift
// ❌ — zero on Android, correct on iOS
let total = dict["totalAllTime"] as? Int ?? 0
let count = dict["donorCount"] as? Int ?? 0

// ✅ — correct on both platforms
private func givingInt(_ dict: [String: Any], _ key: String) -> Int {
    (dict[key] as? Int) ?? Int(dict[key] as? Double ?? 0)
}

let total = givingInt(dict, "totalAllTime")
let count = givingInt(dict, "donorCount")
```

**Scope:** Applies to any `[String: Any]` response decoded from `Functions.functions().httpsCallable()` or `invokeAdminCallable()`. Does NOT affect Firestore `document.data()` fields (those have a separate pattern). Does NOT affect `String`, `Bool`, or `Double` fields — only `Int` casts are affected.

---

### Known Patterns — Combine / `@Published` / `.onReceive` Not Supported on Android (Rule #66)

**Problem:** Skip Fuse only bridges the Swift **Observation macro** (`@Observable`) to Jetpack Compose. Combine — `ObservableObject`, `@Published`, `PassthroughSubject`, `CurrentValueSubject`, `AnyCancellable` — and the SwiftUI `.onReceive(_:)` modifier that accepts a Combine `Publisher` are **not transpiled** and cause Android build failures.

```swift
// ❌ WRONG — Combine is not bridged to Android
import Combine
class MyViewModel: ObservableObject { @Published var value = "" }
.onReceive(NotificationCenter.default.publisher(for: myName)) { _ in ... }
```

```swift
// ✅ CORRECT — @Observable only; NotificationCenter async sequence for notifications
import Observation
@MainActor @Observable class MyViewModel { var value = "" }

.task {
    for await notification in NotificationCenter.default.notifications(named: myName) {
        // handle
    }
}
```

**Rule:** Never use Combine anywhere in Skip Fuse cross-platform code. Only `@Observable` (import Observation BEFORE SkipFuseUI) is bridged. Use `.task { for await }` with `NotificationCenter.default.notifications(named:)` instead of `.onReceive(publisher:)`.

---

### Known Patterns — `List` with Multiple `ForEach` Sections: Duplicate Key Crash (Rule #65)

**Problem:** Compose's `LazyColumn` (what `List` maps to) has a **flat key namespace** — unlike SwiftUI where each `Section` scopes its `ForEach` keys independently. If any two items across ALL sections share the same ID, Compose throws `IllegalArgumentException: Key "Optional("...")" was already used`. The `Optional(...)` wrapper is Skip's key serialization format.

**Real trigger:** In `MessagesView`, `messageRequest` documents use `uid1_uid2` as their Firestore document ID (same as `conversation.id`). During Firestore propagation after accepting a request, both a `pendingRequest` and a `conversation` have the same ID → crash.

```swift
// ❌ CRASH — same ID can appear in both ForEach blocks
List {
    Section { ForEach(viewModel.pendingRequests) { req in ... } }  // key = req.id
    Section { ForEach(viewModel.conversations) { conv in ... } }   // key = conv.id ← collision!
}
```

**Fix:** Add a type-prefixed `listId` computed property to each struct and use `id: \.listId` in ForEach:

```swift
// ✅ CORRECT — type prefix ensures global uniqueness
struct MessageRequest: Identifiable {
    var listId: String { "req_\(id)" }
}
struct Conversation: Identifiable {
    var listId: String { "conv_\(id)" }
}

List {
    Section { ForEach(viewModel.pendingRequests, id: \.listId) { ... } }
    Section { ForEach(viewModel.conversations, id: \.listId) { ... } }
}
```

**Applies to:** Any `List` with multiple `ForEach` blocks or sections on Android. Always add type-prefixed `listId` when the same `List` renders items from different model types or collections.

---

### Known Patterns — `TextEditor` with Flexible `minHeight`/`maxHeight` Crashes Android (Rule #64)

**Problem:** `TextEditor(text:)` with `.frame(minHeight: X, maxHeight: Y)` inside a `VStack` causes a `StackOverflowError` on Android. Compose's layout pass enters an infinite `forceMeasureTheSubtree → remeasure → measure` loop because the flexible height range is ambiguous in the parent `Column` context. The crash occurs at draw time and kills the screen.

```swift
// ❌ CRASH on Android — ambiguous flexible height constraints
TextEditor(text: $messageText)
    .frame(minHeight: 36, maxHeight: 100)
```

**Fix:** Replace with `TextField` at a fixed height. `TextEditor` + flexible min/max is banned on Android.

```swift
// ✅ CORRECT — fixed height, no measure ambiguity
TextField("Message…", text: $messageText)
    .frame(height: 36)
```

**Applies to:** Any `TextEditor` or component using `.frame(minHeight:maxHeight:)` inside a `VStack`/`Column`. Use `.frame(height:)` (fixed) instead. Also note: `TextField(..., axis: .vertical)` is separately banned (Rule #39).

---

### Known Patterns — `.monospacedDigit()` Is Unavailable in Skip (Rule #68)

**Problem:** `.monospacedDigit()` on `Font` is not bridged to Jetpack Compose. Produces `'monospacedDigit' is unavailable` on Android.

```swift
// ❌ WRONG
.font(.caption.monospacedDigit())

// ✅ CORRECT
.font(.caption)
.frame(width: 24, alignment: .trailing)
```

**Rule:** Never use `.monospacedDigit()` in cross-platform Skip code. Use `.frame(width:alignment:)` for tabular alignment.

---

### Known Patterns — Firestore `.order(by:)` Excludes Documents Missing the Field on Android (Rule #69)

**Problem:** A Firestore query with `.order(by: "fieldName")` only returns documents that contain `fieldName`. On iOS, the local SDK cache often includes all documents regardless. On Android, the SDK fetches from the server and correctly excludes docs without the field — silently returning an empty or partial result set.

This caused `loadMemberships()` to return zero results on Android, keeping `isOnWorshipTeam = false` and hiding both the Schedule content and the Worship tab.

```swift
// ❌ WRONG — excludes any membership doc missing "joinedAt" on Android
let snap = try await ref.order(by: "joinedAt", descending: true).getDocuments()

// ✅ CORRECT — fetch all, sort in Swift
let snap = try await ref.getDocuments()
let loaded = snap.documents.compactMap { MyModel(from: $0) }
    .sorted { $0.joinedAt > $1.joinedAt }
```

**Rule:** Never use `.order(by:)` unless every document is guaranteed to have that field. Always fetch unordered and sort in Swift.

---

### Known Patterns — `logger.debug()` Invisible on Android; Use `logger.info()` (Rule #72)

**Problem:** `Logger.debug()` maps to `Log.d()` on Android, which is **filtered by default** — nothing appears in `adb logcat`. Additionally, in Swift 6 strict concurrency, interpolating `self`-owned properties inside the `OSLogMessage` autoclosure requires capturing them in a local `let` first.

```swift
// ❌ WRONG — debug level suppressed on Android; also self-capture error for properties
logger.debug("[trace] isOnWorshipTeam=\(isOnWorshipTeam)")

// ✅ CORRECT — info level always visible; capture self-owned props in locals
let onTeam = isOnWorshipTeam
logger.info("[trace] isOnWorshipTeam=\(onTeam)")
```

**ADB logcat command** (tag = `category:` value in `Logger(subsystem:category:)`):
```bash
adb logcat -s WorshipCompass
```

**Rule:** Always use `logger.info()` (or `.warning()`/`.error()`) for Android-visible logs. Only use `logger.debug()` for iOS-only diagnostics.

---

### Known Patterns — `@Observable` Cross-View Property Changes Don't Recompose Child Views (Rule #73)

**Problem:** An `@Observable @MainActor` ViewModel owned by a parent as `@State var vm = ViewModel()` does NOT automatically trigger Compose recomposition in child views that receive `vm` as a plain `var` parameter. iOS uses `withObservationTracking` (per-property, anywhere in the tree). Skip's Compose bridge does NOT replicate this — `vm.someProperty = newValue` silently updates the Kotlin object but never invalidates any child composable that was passed `vm`. Child views freeze at their initial composition state.

```swift
// ❌ WRONG — child view frozen; isOnWorshipTeam always reads as false
ContentView:     CompassScheduleView(churchId: vm.worshipTeamChurchId ?? "")
ChildView:       var vm: CompassChurchViewModel  // plain var — no Compose subscription

// ✅ CORRECT — shadow key state as @State primitives; copy after async load
ContentView:
    @State var worshipChurchId: String = ""
    @State var isMembershipLoading: Bool = true
    @State var onWorshipTeam: Bool = false
    .task {
        await vm.loadMemberships()
        worshipChurchId = vm.worshipTeamChurchId ?? ""  // @State change triggers recompose
        onWorshipTeam = vm.isOnWorshipTeam
        isMembershipLoading = false
    }
    body: CompassScheduleView(churchId: worshipChurchId)
          WorshipHubView(vm: vm, isLoading: isMembershipLoading, isOnWorshipTeam: onWorshipTeam)

ChildView:
    var isLoading: Bool        // primitive param — change forces recomposition
    var isOnWorshipTeam: Bool  // primitive param — change forces recomposition
```

**Rule:** For any state that gates a UI branch in a child view, copy the ViewModel value to a `@State` primitive in the owning view after async load and pass it as an explicit parameter. Do NOT rely on `@Observable` cross-view propagation on Android.

---

### Known Patterns — `List` + `Section` + Nested `ForEach` Items Invisible on Android (Rule #74)

**Problem:** Skip maps `List` → Compose `LazyColumn`. `Section { ForEach(...) }` inside `List` renders the header but **the ForEach items never appear** on Android. The nested ForEach isn't properly expanded into LazyColumn items.

```swift
// ❌ WRONG — headers show, rows invisible on Android
List {
    Section {
        ForEach(items) { item in RowView(item: item) }
    } header: { Text("Header") }
}

// ✅ CORRECT — flat ScrollView + VStack + NavigationLink
ScrollView {
    VStack(spacing: 0) {
        Text("Header")
        Divider()
        ForEach(items, id: \.id) { item in
            NavigationLink(destination: DetailView(item: item)) {
                RowView(item: item)
            }
            Divider()
        }
    }
}
```

**Rule:** Never nest `ForEach` inside `Section` inside `List` on Android. Use `ScrollView` + `VStack` with manual `Text` dividers.

---

### Known Patterns — `Button(action: {})` Is Completely Untappable (Rule #75)

**Problem:** A `Button` with an empty action closure renders but **never responds to taps** on iOS or Android. The runtime optimizes away the gesture.

```swift
// ❌ WRONG — looks interactive, zero response
Button(action: {}) { RowView(item: item) }

// ✅ CORRECT — NavigationLink for routing, real closure for actions
NavigationLink(destination: DetailView(item: item)) { RowView(item: item) }
```

**Rule:** Never ship a `Button` with an empty action. Use `NavigationLink`, `.sheet`, or a real `@State`-mutating closure.

---

### Known Patterns — Conditional `TabView` Tabs Break Compose Pager on Android (Rule #70)

**Problem:** Putting `if condition { SomeView().tabItem {...}.tag(...) }` inside a `TabView` changes the number of tabs at runtime. iOS handles this via tag-based selection. Android/Compose uses an index-based pager — inserting a new tab at position N shifts all subsequent tabs, causing the wrong tab to appear selected (e.g., jumping to "More" when "Worship" is inserted).

```swift
// ❌ WRONG — tab count changes after data loads, pager jumps
TabView(selection: $tab) {
    ScheduleView().tabItem { ... }.tag(.schedule)
    if vm.isOnWorshipTeam {           // inserts tab at runtime
        WorshipView().tabItem { ... }.tag(.worship)
    }
    MoreView().tabItem { ... }.tag(.more)
}

// ✅ CORRECT — fixed tab count, conditional content INSIDE the tab body
TabView(selection: $tab) {
    ScheduleView().tabItem { ... }.tag(.schedule)
    WorshipView(vm: vm).tabItem { ... }.tag(.worship)   // always present
    MoreView().tabItem { ... }.tag(.more)
}
// Inside WorshipView.body: if !vm.isOnWorshipTeam { placeholder } else { content }
```

**Rule:** Never conditionally show/hide tabs. Always render a fixed set of tabs; branch content inside each tab's `body`.

---

### Known Patterns — `@AppStorage` for Tab Selection Persists Across Login Sessions (Rule #71)

**Problem:** `@AppStorage("tab") var tab = ContentTab.schedule` persists the selected tab in `UserDefaults`. After logout/login the previous tab is restored — if the user was on "More", the next login opens to "More".

```swift
// ❌ WRONG — persists across auth sessions
@AppStorage("tab") var tab = ContentTab.schedule

// ✅ CORRECT — always resets to default on login
@State var tab = ContentTab.schedule
```

Because `ContentView` is re-instantiated on every login, `@State` correctly starts at the default. Reserve `@AppStorage` for settings that should intentionally survive logout.

---

### Step 5 — End-of-session commit check

Before ending the session, confirm that:
- All new incompatibilities discovered this session are appended to `Docs/skip-incompatibilities.md`.
- The memory entry is up to date.
- All changes are committed and pushed.
