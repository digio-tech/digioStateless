# digioStateless

[![GitHub Release](https://img.shields.io/github/v/release/digio-tech/digioStateless?logo=github&label=release)](https://github.com/digio-tech/digioStateless/releases)
[![JitPack](https://jitpack.io/v/digio-tech/digioStateless.svg)](https://jitpack.io/#digio-tech/digioStateless)
[![Release Notes](https://img.shields.io/badge/release%20notes-changelog-informational)](#release-notes)
![minSdk](https://img.shields.io/badge/minSdk-21-blue)
![Kotlin](https://img.shields.io/badge/Kotlin-1.9%2B-7F52FF?logo=kotlin&logoColor=white)
![AGP](https://img.shields.io/badge/AGP-8.0%2B-3DDC84?logo=android&logoColor=white)

Android SDK for Digio's stateless KYC journey.
- **minSdk:** 21 &nbsp;&nbsp; **compileSdk:** 35+ &nbsp;&nbsp; **Kotlin:** 1.9+ &nbsp;&nbsp; **AGP:** 8.0+

## Installation

Add JitPack to your root `settings.gradle` / `settings.gradle.kts`:

```groovy
dependencyResolutionManagement {
    repositories {
        maven { url = uri("https://jitpack.io") }
    }
}
```

Add the Digio BOM + modules to your app module:

```groovy
dependencies {
    implementation(platform("com.github.digio-tech:digio-bom:<version>"))
    implementation("com.github.digio-tech:digioStateless")
    implementation("com.github.digio-tech:sdk_ml_camera")
    implementation("com.github.digio-tech:digio_permissions")
    implementation("com.github.digio-tech:api_client")
    implementation("com.github.digio-tech:image_processor")
}
```


## License

See [LICENSE](./LICENSE).
