# SDK Size & Partner Integration Guide

Digio Android SDK — stateless + camera path. Last updated: 2026-07-14.

Everything below assumes partners consume the SDK via Jitpack (the shipped distribution).

Captures:
- Detailed size breakdown — every Digio AAR and every required transitive with measured bytes
- Honest APK-impact estimates by partner profile
- Minimum requirements
- The full deps block partners need to add
- Recommended minimal partner `proguard-rules.pro`

---

## TL;DR

| Partner profile | Digio APK contribution (compressed) |
|---|---|
| Modern Kotlin app with existing androidx | **~1.2 – 1.7 MB** |
| App already using CameraX / ML Kit | **~500 – 800 KB** |
| Still-selfie only (drops camera-video + barcode) | **~400 – 600 KB additional saving** |
| Java-only legacy app | **~4 – 6 MB** (unavoidable — includes Kotlin runtime) |
| Any of the above with suboptimal proguard config | +2–3 MB extra (fixable — see anti-patterns below) |

---

## Detailed Size Breakdown

All sizes measured 2026-07-14 from `:module:assembleRelease` output (release AAR after R8 + repackaging). Column 1 is what gets downloaded; column 2 is a realistic estimate of what ends up in the partner's release APK after R8 + DEX compression, assuming the partner already has kotlin stdlib + androidx foundation (most modern apps).

### Digio's own artifacts

| Artifact                               | AAR download | APK contribution |
|----------------------------------------|---|---|
| `digio-api-client`                     | 32 KB | ~40–60 KB |
| `digio_stateless`                      | 62 KB | ~80–120 KB |
| `digio_kyc_ml_camera` / digioCameraKit | 300 KB | ~200–300 KB |
| `digio_permissions`                    | 21 KB | ~25–35 KB |
| `digio_image_process`                  | 98 KB | ~80–120 KB |
| **Digio total**                        | **~513 KB** | **~425–635 KB** |

### Supporting libraries

The Digio SDK relies on standard Android + Google + third-party libraries partners are typically already using (Kotlin stdlib, coroutines-core, appcompat, material, CameraX, ML Kit face detection, OkHttp, Retrofit, Gson). Full list is in the "Required Dependencies" section below.

**Coil is no longer required** — Partners save ~200–300 KB by dropping Coil.

**Lifecycle-KTX libs are no longer required** — cameraKit and digioStateless were refactored (2026-07-13) to use self-managed `CoroutineScope`s instead of `viewModelScope` / `lifecycleScope`. Partners can drop `fragment-ktx` / `activity-ktx` / `lifecycle-*-ktx` unless they use those elsewhere.

**camera-video is now truly optional** — cameraKit's (2026-07-14). Still-selfie partners can drop `androidx.camera:camera-video` (~200–400 KB) with zero crash risk across every Android version cameraKit supports (API 21+).

**play-services-mlkit-barcode-scanning is optional** — only needed if the partner uses QR code flows. Still-selfie partners can drop it (~500 KB – 1 MB after R8).

**Bottom line for a modern Kotlin partner still-selfie app:** the net APK size increase from adding Digio's stateless + camera SDK is ~**1.2 – 1.7 MB compressed** — down from ~1.5–2 MB before the 2026-07-13/14 optimizations.

---

## Partner APK Impact by Profile

### Category A — Modern Kotlin Android app
Most Fintech / Banking / KYC-adjacent apps fall here.

- **Already has:** Kotlin stdlib, coroutines-core, appcompat, material, activity, fragment, lifecycle, OkHttp, Retrofit, Gson
- **Missing (typically):** CameraX, ML Kit face detection, image_processor
- **Digio contribution: ~1.2 – 1.7 MB compressed** (post 2026-07-13/14 optimizations; ~300 KB less than previous versions since Coil is no longer needed)

### Category B — Modern app already using biometric / camera
Already integrates CameraX (e.g., QR scanning) or Firebase ML.

- **Already has:** everything from Cat A + CameraX + likely some ML Kit
- **Digio contribution: ~500 – 800 KB compressed** (mostly just Digio's own AARs)

### Category B+ — Still-selfie-only partner
Selfie flow only, no video capture, no QR code scanning, no geo-tagging.

- **Can drop:** `camera-video`, `mlkit-barcode-scanning`, `play-services-location` — cameraKit's CameraController + QRCodeAnalyzer wrap these in reflection / try-catch and fail gracefully with `onError` callbacks instead of crashing on missing classes.
- **Additional saving on top of Cat A/B: ~700 KB – 1.5 MB**

### Category C — Legacy Java-only app
Older enterprise / banking apps without Kotlin.

- **Missing:** Kotlin stdlib, coroutines, all KTX libs, modern androidx
- **Digio contribution: ~4 – 6 MB compressed** — unavoidable
  - Kotlin runtime alone: ~1.5–2 MB
  - Kotlin coroutines: ~500 KB
  - All the KTX + modern androidx: ~1 MB
  - CameraX + ML Kit chain: ~1–1.5 MB
  - Digio's own AARs: ~500 KB
- No SDK-side workaround — our AARs are compiled Kotlin bytecode

### Category D — Broken partner proguard-rules.pro
Applies on top of any category above. **This is the biggest hidden cost.**

**Symptoms** — partner's proguard-rules.pro contains any of these anti-patterns:
- `-keepnames class ** { *; }` — disables obfuscation globally
- `-keeppackagenames` — blocks package flattening
- `-optimizations !method/inlining/*` — blocks method inlining
- `-keep public class * extends androidx.**` or `-keep public class androidx.** { public *; }` — forces retention of every androidx transitive
- `-dontskipnonpubliclibraryclasses` — ProGuard-era, R8 ignores


**Effect:** globally disables R8 shrinking. Not just Digio's classes — the partner's own code too. Adds 2–3 MB.

**Fix:** replace with the minimal template below.

---

## Minimum Requirements

| Requirement | Version |
|---|---|
| AGP | **8.0+** (uses R8 full-mode by default) |
| Kotlin | **1.9+** (our AARs compiled against 1.9.20) |
| JVM target | **11** |
| compileSdk | **35+** |
| minSdk | **21** (both digioStateless and digioCameraKit) |
| Java-only allowed? | Yes, but adds Kotlin runtime (~2 MB) overhead |

---

## Required Dependencies

Partners consume via Jitpack. Add the Digio BOM + module coordinates + framework transitive deps:

```gradle
dependencies {
    // Digio SDK — pin versions via BOM
    implementation(platform("com.github.digio-tech:digio-bom:<version>"))
    implementation("com.github.digio-tech:digioStateless")
    implementation("com.github.digio-tech:sdk_ml_camera")
    implementation("com.github.digio-tech:digio_permissions")
    implementation("com.github.digio-tech:api_client")
    implementation("com.github.digio-tech:image_processor")

    // UI
    implementation("androidx.appcompat:appcompat")
    implementation("com.google.android.material:material")
    implementation("androidx.constraintlayout:constraintlayout")

    // Camera + ML Kit — core CameraX artifacts (always required)
    implementation("androidx.camera:camera-core")
    implementation("androidx.camera:camera-camera2")
    implementation("androidx.camera:camera-lifecycle")
    implementation("androidx.camera:camera-view")
    implementation("com.google.android.gms:play-services-mlkit-face-detection")

    // OPTIONAL — drop these if the partner flow doesn't need them:
    implementation("androidx.camera:camera-video")                                    // video capture / motion selfie
    implementation("com.google.android.gms:play-services-mlkit-barcode-scanning")     // QR code scanning
    implementation("com.google.android.gms:play-services-location")                   // geo-tagging (hasGeoTagging=true)

    // HTTP — required for KYC network calls
    implementation("com.squareup.okhttp3:okhttp")
    implementation("com.squareup.retrofit2:retrofit")
    implementation("com.squareup.retrofit2:converter-gson")
}
```

The BOM aligns all versions — partners don't need to pin the transitive versions themselves.

### Notes on removed requirements (2026-07-13/14)

- **`io.coil-kt:coil` no longer required.** cameraKit now uses `DigioImageLoader` — a platform-only image loader (HttpURLConnection + BitmapFactory + TransitionDrawable + LruCache). Partners save ~200–300 KB.
- **KTX libs (`fragment-ktx` / `activity-ktx` / `lifecycle-*-ktx`) no longer required by Digio.** Both digioStateless and cameraKit were refactored to use self-managed `CoroutineScope`s. Partners can keep or drop these based on their own app code. Saves ~50 KB if dropped.
- **`androidx.core:core-ktx` no longer explicitly required.** AppCompat brings core transitively; core-ktx only adds Kotlin extensions our code doesn't consume.
- **`androidx.camera:camera-video` is now optional.** cameraKit's `CameraController` loads `VideoController` via `Class.forName` (safe on API 21+); still-selfie partners can drop this. Motion-selfie configs downgrade gracefully to still capture if the artifact is missing.
- **`play-services-mlkit-barcode-scanning` is optional.** `QRCodeAnalyzer` instantiation is wrapped in try/catch — partners without QR flows can drop this artifact.

---

## Minimal Partner `proguard-rules.pro`

All framework-specific R8 rules (Retrofit, ML Kit, Digio public API) travel inside the AARs' `consumer-rules.pro` and are applied automatically by the partner's R8. Partner's own proguard only needs attribute keeps + rules for their own app code.

```proguard
# Attributes required for reflection / stack traces / generics
-keepattributes SourceFile,LineNumberTable
-keepattributes *Annotation*
-keepattributes Signature
-keepattributes InnerClasses, EnclosingMethod
-keepattributes Exceptions
-keepattributes RuntimeVisibleAnnotations, RuntimeVisibleParameterAnnotations
-keepattributes AnnotationDefault

# @Parcelize / kotlin-parcelize
-dontwarn kotlinx.parcelize.**
-keep class kotlinx.parcelize.** { *; }

# Warning suppressions for optional runtime deps
-dontwarn okhttp3.**
-dontwarn java.lang.invoke.StringConcatFactory
-dontwarn javax.annotation.**
-dontwarn org.conscrypt.**
-dontwarn kotlin.jvm.internal.SourceDebugExtension
-dontwarn kotlin.Unit
```

**Do NOT include these anti-patterns** (if you're inheriting an older proguard template that has them, remove them):
- `-keepnames class ** { *; }`
- `-keeppackagenames`
- `-optimizations !method/inlining/*`
- `-keep public class * extends androidx.**` / `-keep public class androidx.** { public *; }`
- `-dontskipnonpubliclibraryclasses`
