# Gradle setup templates

All version placeholders (`<PLACEHOLDER>`) must be filled from research done in SKILL.md Step 1
— never from memorized numbers, since these move frequently and have compatibility constraints
between each other.

## gradle/libs.versions.toml (version catalog)

```toml
[versions]
agp = "<AGP_VERSION>"
kotlin = "<KOTLIN_VERSION>"
composeBom = "<COMPOSE_BOM_VERSION>"
coreKtx = "<CORE_KTX_VERSION>"
lifecycleRuntimeKtx = "<LIFECYCLE_VERSION>"
activityCompose = "<ACTIVITY_COMPOSE_VERSION>"
navigationCompose = "<NAV_COMPOSE_VERSION>"
room = "<ROOM_VERSION>"
hilt = "<HILT_VERSION>"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycleRuntimeKtx" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "activityCompose" }
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-ui-graphics = { group = "androidx.compose.ui", name = "ui-graphics" }
androidx-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
# Compose compiler is a Kotlin-versioned Gradle plugin since Kotlin 2.0 — do NOT use the
# older composeOptions { kotlinCompilerExtensionVersion = "..." } pattern for Kotlin 2.0+.
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
kotlin-ksp = { id = "com.google.devtools.ksp", version = "<KSP_VERSION_MATCHING_KOTLIN>" }
hilt-android-gradle = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
```

Note on KSP: its version string is tied to the exact Kotlin version (format like
`<kotlin_version>-<ksp_version>`) — search `ksp version <kotlin_version>` to find the matching
release rather than guessing, since a mismatched KSP/Kotlin pairing fails immediately at sync.

## Project-level build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.compose.compiler) apply false
    alias(libs.plugins.kotlin.ksp) apply false
    alias(libs.plugins.hilt.android.gradle) apply false
}
```

## app/build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.compose.compiler)
    alias(libs.plugins.kotlin.ksp)
    alias(libs.plugins.hilt.android.gradle)
}

android {
    namespace = "com.example.app"
    compileSdk = <COMPILE_SDK_INT>

    defaultConfig {
        applicationId = "com.example.app"
        minSdk = <MIN_SDK_INT>
        targetSdk = <TARGET_SDK_INT>
        versionCode = 1
        versionName = "1.0"
    }

    buildTypes {
        release {
            isMinifyEnabled = true
            proguardFiles(getDefaultProguardFile("proguard-android-optimize.txt"), "proguard-rules.pro")
        }
    }

    buildFeatures {
        compose = true
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

// Compose compiler options (Kotlin 2.0+ Gradle-plugin style), NOT the legacy
// composeOptions { kotlinCompilerExtensionVersion = ... } block:
composeCompiler {
    // reportsDestination / stabilityConfigurationFile etc are optional; check current docs
    // if the project needs Compose compiler metrics or stability configuration.
}

dependencies {
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.androidx.activity.compose)

    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.ui)
    implementation(libs.androidx.ui.graphics)
    implementation(libs.androidx.ui.tooling.preview)
    implementation(libs.androidx.material3)
    implementation(libs.androidx.navigation.compose)

    implementation(libs.androidx.room.runtime)
    implementation(libs.androidx.room.ktx)
    ksp(libs.androidx.room.compiler)

    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)
}
```

## Notes

- Prefer `platform(libs.androidx.compose.bom)` (the Compose BOM) over pinning each
  `androidx.compose.*` artifact's version individually — the BOM guarantees mutually compatible
  versions across all Compose libraries.
- `minSdk`/`targetSdk`/`compileSdk` should reflect what was found in Step 1's research, not
  memorized numbers — Play Store's minimum targetSdk requirement changes roughly yearly.
- Java 17 is the current common baseline for `sourceCompatibility`/`jvmTarget`, but confirm
  against the AGP version's documented supported JDK range if unsure, since this has shifted
  across AGP releases too.
