---
name: android-app-dev
description: Use whenever the user asks to build, scaffold, edit, or debug a native Android app written in Kotlin (or Java), using Jetpack Compose or the legacy View/XML system, Gradle, and Android Studio project conventions. Trigger on phrases like "make an android app", "build an app for android", "add a screen/activity/fragment", "fix my build.gradle", "add a Room database", or any request touching AndroidManifest.xml, build.gradle(.kts), libs.versions.toml, or files under app/src/main/java or app/src/main/kotlin. Always use this skill when the user names a specific recent Android API level, Kotlin version, Compose version, or Android Gradle Plugin (AGP) version, since these change frequently and the agent's memorized defaults may be stale — verify current versions and current recommended patterns (e.g. the Compose compiler now ships as a Kotlin-versioned Gradle plugin, not a separate kotlinCompilerExtensionVersion) via web search before writing version-sensitive Gradle config.
---

# Android App Development

## Scope

Native Android apps: Kotlin (default, preferred over Java for new code unless the user's
project is already Java) with Gradle as the build system and Android Studio project
conventions. Covers both modern Jetpack Compose UI and legacy View/XML-based UI, architecture
(ViewModel, Room, Hilt/Koin, coroutines/Flow), and the Gradle/version-catalog layer. This is a
different domain from the `minecraft-fabric-mod` skill (Java + Fabric/Loom for Minecraft) — don't
conflate Android's Gradle/Kotlin ecosystem with that one even though both use Gradle.

## Core principle: don't guess fast-moving version numbers or recently-changed conventions

Android tooling changes in ways that break naive pattern-matching from older training data:

- **AGP (Android Gradle Plugin), Kotlin, and Compose versions** all move independently and have
  compatibility constraints between them.
- **The Compose compiler setup changed**: since Kotlin 2.0, the Compose compiler ships as a
  Kotlin-versioned Gradle plugin (`org.jetbrains.kotlin.plugin.compose`), configured via a
  `composeCompiler { }` block — the older `composeOptions { kotlinCompilerExtensionVersion = "..." }`
  pattern from pre-2.0 Kotlin is legacy. Don't write the old pattern for a new project without
  checking which convention applies to the Kotlin version in use.
- **Compose BOM version, minSdk/targetSdk/compileSdk defaults, and current Jetpack library
  versions** (Room, Hilt, Navigation, Lifecycle, etc.) all get bumped regularly, and using a
  memorized version can produce dependency-resolution failures or miss newer, non-deprecated APIs.
- **Recommended architecture guidance itself has evolved** (e.g. Google's official architecture
  guidance, recommended navigation approach, recommended DI approach) — don't assume a pattern
  learned from older tutorials is still what Google currently recommends without checking.

**Therefore: before writing or editing `build.gradle(.kts)`, `libs.versions.toml`, or
`AndroidManifest.xml` for anything version-sensitive, search first rather than relying on
memorized numbers.** This applies even if the agent feels confident.

## Step 1 — Research (do this for any new project or version-sensitive change)

Search for, in order of importance:

1. **Current stable Kotlin version** — search `kotlin latest stable version`.
2. **Current stable AGP version and its Gradle compatibility** — search `android gradle plugin latest version` and check https://developer.android.com/build/releases/gradle-plugin for the AGP-to-Gradle-version compatibility table.
3. **Current Compose BOM version** — search `jetpack compose bom latest version` or check https://developer.android.com/develop/ui/compose/bom/bom-mapping — the BOM pins compatible versions of all `androidx.compose.*` artifacts together, so prefer using it over pinning individual Compose library versions by hand.
4. **Compose compiler setup for the Kotlin version in use** — check https://developer.android.com/develop/ui/compose/setup-compose-dependencies-and-compiler for the current recommended setup (Gradle plugin approach for Kotlin 2.0+, described above).
5. **Current recommended minSdk/targetSdk/compileSdk** — search `android compileSdk targetSdk latest` — Google Play has minimum targetSdk requirements that bump roughly yearly; check https://developer.android.com/google/play/requirements/target-sdk for the current requirement if the app is meant for Play Store distribution.
6. **Version of any specific Jetpack library the task touches** (Room, Hilt, Navigation Compose, Lifecycle, WorkManager, etc.) — search `androidx <library> latest version` or check https://developer.android.com/jetpack/androidx/versions.

Useful authoritative sources to fetch/search directly:
- `https://developer.android.com/develop` — main dev guide hub
- `https://developer.android.com/jetpack/androidx/versions` — current versions of all AndroidX libraries
- `https://developer.android.com/build/releases/gradle-plugin` — AGP release notes + Gradle compatibility
- `https://developer.android.com/develop/ui/compose/bom/bom-mapping` — Compose BOM-to-library version mapping
- `https://kotlinlang.org/docs/releases.html` — Kotlin release notes
- `https://developer.android.com/topic/architecture` — Google's current official architecture guidance

If research tools are unavailable, tell the user explicitly which version numbers are unverified
and should be double-checked in Android Studio before building.

## Step 2 — Confirm findings before generating files

Briefly state the concrete versions found (Kotlin, AGP, Compose BOM, compileSdk/targetSdk/minSdk,
and any specific library versions relevant to the task) before writing Gradle config. If numbers
seem inconsistent (e.g. a Compose BOM newer than the Kotlin version could plausibly support),
search again.

## Step 3 — Project structure

Standard modern Android project layout (single-module, Compose-based — the common default unless
the user's existing project says otherwise):

```
app/
├── build.gradle.kts
├── src/main/
│   ├── AndroidManifest.xml
│   ├── kotlin/com/example/app/
│   │   ├── MainActivity.kt
│   │   ├── ui/
│   │   │   ├── theme/          (Color.kt, Theme.kt, Type.kt)
│   │   │   └── screens/        (one file/folder per screen + its ViewModel)
│   │   ├── data/                (repositories, Room entities/DAOs, network sources)
│   │   └── di/                  (Hilt/Koin modules, if used)
│   └── res/
│       ├── values/ (strings.xml, themes.xml)
│       ├── drawable/
│       └── mipmap-*/ (launcher icons)
build.gradle.kts            (project-level)
settings.gradle.kts
gradle/libs.versions.toml   (version catalog — current standard convention)
```

Use a Gradle **version catalog** (`gradle/libs.versions.toml`) for dependency versions rather
than hardcoding version strings inline in each module's `build.gradle.kts` — this is the current
standard convention and what Android Studio's own project templates generate.

Read `references/gradle-setup.md` for annotated `build.gradle.kts` / `libs.versions.toml`
templates (placeholders to fill from Step 1's research, not to invent).

Read `references/architecture-guide.md` for the current recommended app architecture (UI layer /
domain / data layer, ViewModel + StateFlow/UiState pattern, unidirectional data flow) and when
Hilt vs. Koin vs. manual DI make sense.

Read `references/compose-patterns.md` for common Compose UI patterns: navigation
(Navigation-Compose), state hoisting, common layout recipes, and Material 3 theming.

Read `references/common-pitfalls.md` before finishing — covers mistakes that most often break
freshly generated or edited Android projects (missing permissions, Compose recomposition
pitfalls, Room migration mistakes, ProGuard/R8 issues, wrong Compose compiler config, etc).

## Step 4 — Verify

If Gradle and network access are available in the environment, try `./gradlew tasks` or
`./gradlew assembleDebug` to confirm the project resolves and compiles. Since building a full APK
often needs the Android SDK installed (frequently not available in a sandboxed environment),
clearly tell the user which parts are unverified and that they should open the project in Android
Studio (which will prompt to install any missing SDK components) to fully confirm the build.

## When editing an existing app rather than starting fresh

Check the existing project's `libs.versions.toml`/`build.gradle.kts` for versions and conventions
already in use (Compose vs. Views, Hilt vs. Koin vs. none, existing architecture pattern) and
match them, rather than imposing different conventions — unless the user is explicitly asking to
migrate or upgrade something, in which case do the full research step for the new target.

## Java vs. Kotlin

Default to Kotlin for any new code. If the user's existing project is Java-based, or they
explicitly ask for Java, write idiomatic modern Java instead rather than pushing an
unrequested migration — but mention that Kotlin is Google's recommended default for new Android
development if it seems relevant.
