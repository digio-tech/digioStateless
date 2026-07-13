# SDK Size & Partner Integration Guide

Digio Android SDK — stateless + camera path. Last updated: 2026-07-10.

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
| Modern Kotlin app with existing androidx | **~1.5 – 2 MB** |
| App already using CameraX / ML Kit | **~0.7 – 1 MB** |
| Java-only legacy app | **~4 – 6 MB** (unavoidable — includes Kotlin runtime) |
| Any of the above with suboptimal proguard config | +2–3 MB extra (fixable — see anti-patterns below) |

---

## Detailed Size Breakdown

All sizes measured 2026-07-10 from Gradle cache (Jitpack download sizes). Column 1 is what gets downloaded; column 2 is a realistic estimate of what ends up in the partner's release APK after R8 + DEX compression, assuming the partner already has kotlin stdlib + androidx foundation (most modern apps).

### Digio's own artifacts

| Artifact                                       | AAR download | APK contribution |
|------------------------------------------------|---|---|
| `digio-api-client`                     | 32 KB | ~40–60 KB |
| `digio_stateless`                      | 62 KB | ~80–120 KB |
| `digio_kyc_ml_camera` (digioCameraKit)  | 283 KB | ~200–300 KB |
| `digio_permissions`                            | (small) | ~30 KB |
| `image_processor`                              | (small) | ~50–100 KB |
| **Digio total**                                | **~400–500 KB** | **~400–600 KB** |

### Supporting libraries

The Digio SDK relies on standard Android + Google + third-party libraries partners are typically already using (Kotlin stdlib, coroutines, appcompat, material, CameraX, ML Kit face detection, OkHttp, Retrofit, Gson, Coil). Full list is in the "Required Dependencies" section below.

**Bottom line for a modern Kotlin partner app:** the net APK size increase from adding Digio's stateless + camera SDK is ~**1.5 – 2 MB compressed** — measured on a newly-created modern Kotlin integration test app.

---

## Partner APK Impact by Profile

### Category A — Modern Kotlin Android app
Most Fintech / Banking / KYC-adjacent apps fall here.

- **Already has:** Kotlin stdlib, coroutines, appcompat, material, activity, fragment, lifecycle, OkHttp, Retrofit, Gson, core-ktx
- **Missing (typically):** CameraX, ML Kit face detection, Coil, image_processor
- **Digio contribution: ~1.5 – 2 MB compressed**

### Category B — Modern app already using biometric / camera
Already integrates CameraX (e.g., QR scanning) or Firebase ML.

- **Already has:** everything from Cat A + CameraX + likely some ML Kit
- **Digio contribution: ~700 KB – 1 MB compressed** (mostly just Digio's own AARs)

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
- Broad keeps for libraries the partner doesn't depend on (spongycastle, ecs.rdlibrary, two_way_sdk, simpleframework.xml, com.emudhra, etc.)

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

    // Kotlin + KTX (fragment-ktx transitively brings activity-ktx + lifecycle-*-ktx)
    implementation("androidx.core:core-ktx")
    implementation("androidx.fragment:fragment-ktx")

    // UI
    implementation("androidx.appcompat:appcompat")
    implementation("com.google.android.material:material")
    implementation("androidx.constraintlayout:constraintlayout")

    // Camera + ML Kit
    implementation("androidx.camera:camera-core")
    implementation("androidx.camera:camera-camera2")
    implementation("androidx.camera:camera-lifecycle")
    implementation("androidx.camera:camera-view")
    implementation("androidx.camera:camera-video")
    implementation("com.google.android.gms:play-services-mlkit-face-detection")
    implementation("com.google.android.gms:play-services-location")

    // Image loading
    implementation("io.coil-kt:coil:2.6.0")

    // HTTP
    implementation("com.squareup.okhttp3:okhttp")
    implementation("com.squareup.retrofit2:retrofit")
    implementation("com.squareup.retrofit2:converter-gson")
}
```

The BOM aligns all versions — partners don't need to pin the transitive versions themselves.

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

That's it. ~20 lines. Nothing framework-specific.

**Do NOT include these anti-patterns** (if you're inheriting an older proguard template that has them, remove them):
- `-keepnames class ** { *; }`
- `-keeppackagenames`
- `-optimizations !method/inlining/*`
- `-keep public class * extends androidx.**` / `-keep public class androidx.** { public *; }`
- `-dontskipnonpubliclibraryclasses`
