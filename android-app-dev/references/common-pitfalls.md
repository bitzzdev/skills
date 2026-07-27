# Common pitfalls in Android app development

Check this list before telling the user the app/feature is done.

1. **Legacy Compose compiler config on Kotlin 2.0+.** Writing
   `composeOptions { kotlinCompilerExtensionVersion = "..." }` for a project on Kotlin 2.0 or
   later is the old pattern — since Kotlin 2.0, the Compose compiler is a Kotlin-versioned Gradle
   plugin (`org.jetbrains.kotlin.plugin.compose`) applied per-module, configured via a
   `composeCompiler { }` block. Mixing the two conventions, or using the legacy one on a modern
   Kotlin version, causes build configuration errors or silently-wrong compiler versions.

2. **Mismatched KSP/Kotlin version pairing.** KSP's version string is tied to the exact Kotlin
   version it was built against (format like `<kotlin_version>-<ksp_build>`). Pinning an
   unrelated KSP version against a different Kotlin version fails at sync time — always search
   for the exact matching KSP release for the Kotlin version in use.

3. **Pinning individual Compose library versions instead of using the BOM.** Manually version-
   pinning each `androidx.compose.*` artifact risks incompatible combinations. Use
   `implementation(platform(libs.androidx.compose.bom))` and let unversioned Compose library
   declarations resolve through it.

4. **Missing or incorrectly scoped permissions.** Runtime-dangerous permissions (camera,
   location, storage, notifications on API 33+, etc.) need both a `<uses-permission>` manifest
   entry AND a runtime permission request via `ActivityResultContracts.RequestPermission` (or
   `RequestMultiplePermissions`) in Compose/Activity code — a manifest entry alone doesn't grant
   the permission on API 23+. Also check for permissions that were split or renamed across API
   levels (e.g. granular media permissions introduced in API 33) if targeting a wide SDK range.

5. **`collectAsState()` instead of `collectAsStateWithLifecycle()`.** Plain `collectAsState()`
   keeps collecting a Flow even while the composable isn't visible/active (e.g. app backgrounded),
   wasting resources and occasionally causing crashes from state updates on a torn-down screen.
   Prefer the lifecycle-aware variant (`androidx.lifecycle:lifecycle-runtime-compose`) for
   ViewModel `StateFlow` collection in UI.

6. **Recomposition performance pitfalls.** Passing lambdas/objects that aren't stable (not
   marked `@Immutable`/`@Stable`, or newly allocated on every recomposition without `remember`)
   as parameters to composables can trigger excessive recomposition. Common fixes: hoist
   lambdas with `remember { { ... } }` where they capture changing state, mark data classes used
   as Compose parameters `@Immutable` where genuinely immutable, and avoid unnecessary nested
   state reads high up in a large composable tree (read state as close as possible to where it's
   actually used, to scope recomposition narrowly).

7. **Room migration mistakes.** Bumping a `@Database` version number without providing a
   `Migration` (or without allowing destructive migration deliberately via
   `fallbackToDestructiveMigration()` — acceptable only for early development, not for a real app
   with existing user data) causes crashes on upgrade for existing installs. Always write an
   explicit `Migration` for schema changes once the app has real users, and enable Room's schema
   export (`exportSchema = true` + a schema location) to keep migration history verifiable.

8. **R8/ProGuard breaking release builds.** Enabling `isMinifyEnabled = true` for release builds
   without appropriate `proguard-rules.pro` keep-rules can strip classes referenced only via
   reflection (common with some serialization libraries, Room, Retrofit/Moshi/Gson model classes).
   If a release build crashes with `ClassNotFoundException` or similar only in the minified
   release variant (not debug), check for missing keep rules for the library involved — most
   libraries publish their own recommended consumer ProGuard rules, but double-check if using
   reflection-heavy code directly.

9. **Edge-to-edge / window insets handling.** Recent Android versions increasingly enable
   edge-to-edge display by default, meaning content can be drawn under system bars unless insets
   are explicitly handled (`Modifier.windowInsetsPadding`, `enableEdgeToEdge()`, etc.) — verify
   current guidance for the target SDK, since default behavior and the recommended insets APIs
   have changed across recent Android/Compose versions.

10. **Assuming an architecture/DI choice without checking project scale.** Scaffolding full
    Hilt DI + a domain/use-case layer + comprehensive test suites for a simple one-screen
    prototype adds unrequested complexity. Match the architectural investment to what the user
    actually asked for and the app's real scope (see `architecture-guide.md`).

11. **Not verifying at all.** If Gradle and the Android SDK are available, actually attempt
    `./gradlew assembleDebug` (or at least `./gradlew tasks`) rather than only generating files
    and declaring success. If the Android SDK isn't available in the sandbox (common), say so
    explicitly and tell the user to open the project in Android Studio to fully verify the build.
