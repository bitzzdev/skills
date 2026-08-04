# GitHub Actions CI build workflow

This project builds exclusively through GitHub Actions — never locally, never in the sandbox.
Every scaffolded app must include a debug-build workflow at `.github/workflows/build.yml`.
Release-signing is a separate, opt-in workflow — only add it if the user actually asks for signed
release builds, since it requires them to configure signing secrets in their own repo.

## Base debug-build workflow (required default, no secrets needed)

```yaml
name: Build

on:
  push:
    branches: [ "**" ]
  pull_request:
    branches: [ "**" ]
  workflow_dispatch: {}   # allows manually triggering a build from the Actions tab

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          # Match whatever JDK version was confirmed in SKILL.md Step 1 for the AGP version in
          # use — keep this in sync with build.gradle.kts's compileOptions/jvmTarget.
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4
        # Handles Gradle dependency + wrapper caching automatically.

      - name: Make gradlew executable
        run: chmod +x ./gradlew

      - name: Run lint
        run: ./gradlew lint --no-daemon

      - name: Run unit tests
        run: ./gradlew testDebugUnitTest --no-daemon

      - name: Build debug APK
        run: ./gradlew assembleDebug --no-daemon

      - name: Upload debug APK
        uses: actions/upload-artifact@v4
        with:
          name: app-debug
          path: app/build/outputs/apk/debug/*.apk
          if-no-files-found: error
```

Notes:
- `assembleDebug` produces a debug-signed APK using Android's built-in debug keystore — this
  needs **no secrets or signing configuration** and is why it's the required default CI build,
  unlike a release build (see below).
- The GitHub-hosted `ubuntu-latest` runner does not include the Android SDK pre-configured for
  Gradle's `android` plugin in every image variant — if a run fails specifically on SDK/license
  issues, add an explicit SDK setup step (search `android-actions/setup-android` or the current
  equivalent GitHub Action, since the Android SDK setup action landscape has shifted over time;
  confirm the current recommended action rather than assuming an old one is still maintained).
  Many recent `ubuntu-latest` images do include a working Android SDK out of the box for basic
  Gradle Android builds — check whether the base workflow above succeeds first before adding an
  extra SDK setup step unnecessarily.
- Lint and unit tests are included by default since they're fast, need no secrets, and catch real
  issues — drop them only if the user explicitly wants a minimal build-only workflow.

## Optional: release-signing workflow (only add if the user asks for signed release builds)

Signing a release build requires secrets the user must configure themselves in their repo's
Settings → Secrets and variables → Actions: typically a base64-encoded keystore file, the
keystore password, key alias, and key password. Never invent or generate a keystore/signing
credentials on the user's behalf inside this sandbox — signing keys are sensitive, long-lived
secrets the user must own and back up themselves.

```yaml
name: Release Build

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch: {}

permissions:
  contents: write   # needed if also creating a GitHub Release from this workflow

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      - name: Decode keystore
        run: echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > keystore.jks

      - name: Make gradlew executable
        run: chmod +x ./gradlew

      - name: Build signed release APK/AAB
        run: ./gradlew bundleRelease assembleRelease --no-daemon
        env:
          KEYSTORE_PATH: keystore.jks
          KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
          KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}

      - name: Upload release artifacts
        uses: actions/upload-artifact@v4
        with:
          name: app-release
          path: |
            app/build/outputs/apk/release/*.apk
            app/build/outputs/bundle/release/*.aab
          if-no-files-found: error
```

This assumes `app/build.gradle.kts` has a `signingConfigs { release { ... } }` block reading those
same environment variables — set that up explicitly if wiring up release signing, and explain to
the user exactly which secrets they need to add and how to generate/export a keystore, rather
than assuming they already have one configured.

## Checking the build result

After pushing, don't declare the app "done" until the workflow has actually been checked:
- If the GitHub CLI (`gh`) is available and authenticated, use `gh run list --branch <branch>
  --limit 1` and `gh run view <run-id> --log` (or `gh run watch <run-id>`) to check status and
  pull failure logs directly, so a broken build can be diagnosed and fixed without waiting on the
  user to check manually.
- If `gh` isn't available, tell the user to check the repository's **Actions** tab, and that the
  debug APK (once the workflow succeeds) is downloadable from that run's **Artifacts** section.
- If the workflow fails, read the actual failure log rather than guessing — Gradle/AGP build
  failures are usually specific about which dependency, SDK component, or config mismatch failed,
  which often points back to a Step 1 research step worth re-checking (e.g. a Compose BOM version
  incompatible with the Kotlin version in use, or a missing Android SDK component on the runner).
