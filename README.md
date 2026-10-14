# 📱 Mobile CI/CD

### From commit to store: a practical Android & Flutter release guide.

![Flutter](https://img.shields.io/badge/Flutter-02569B?style=for-the-badge&logo=flutter&logoColor=white)
![Android](https://img.shields.io/badge/Android-3DDC84?style=for-the-badge&logo=android&logoColor=white)
![iOS](https://img.shields.io/badge/iOS-000000?style=for-the-badge&logo=apple&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)

</div>

---

## 🔄 Lifecycle

<div align="left<img width="1202" height="261" alt="Screenshot 2026-10-14 at 2 47 33 PM" src="https://github.com/user-attachments/assets/d33c4c4c-6863-4979-99d7-e5d85341b5c1" />


`Develop` → `Pull request` → `Analyze & test` → `Build` → `Sign` → `Internal testing / TestFlight` → `Review` → `Staged release` → `Monitor`

</div>

---

## 📚 Contents

| | | |
|---|---|---|
| 🧭 [The big picture](#-the-big-picture) | 🛠️ [Prepare your project](#-prepare-your-project) | 🏗️ [Build types, without the confusion](#-build-types-without-the-confusion) |
| 🎨 [One codebase. Multiple flavors.](#-one-codebase-multiple-flavors) | 💻 [Develop locally, integrate often](#-develop-locally-integrate-often) | 🧪 [Test in layers](#-test-in-layers) |
| ⚙️ [Automate with GitHub Actions](#-automate-with-github-actions) | 🐳 [Reproducible builds with Docker](#-reproducible-builds-with-docker) | 🔐 [Sign, version, and protect](#-sign-version-and-protect) |
| 🤖 [Publish to Google Play](#-publish-to-google-play) | 🍎 [Publish to the App Store](#-publish-to-the-app-store) | 🚀 [Release with confidence](#-release-with-confidence) |

---

## 🧭 The big picture

A reliable release is not a single command. It is a repeatable journey from a local change to a signed, tested app in your users’ hands.

- 🔁 **Continuous integration (CI):** every pull request is automatically analyzed, tested, and built. Problems surface before code is merged.
- 📦 **Continuous delivery (CD):** a passing build becomes a release candidate that can be promoted with approval. Continuous deployment removes that manual gate where appropriate.
- 🚦 Mobile releases still depend on signing, store review, and rollout controls. A green pipeline does not mean an app is publicly available.

---

## 🛠️ Prepare your project

Start with a project that builds locally. Automation should reproduce a known-good process, not hide a broken one.

- 🧰 Install a pinned Flutter SDK (including Dart), the Android SDK, and the JDK required by your Android Gradle Plugin. Commit `pubspec.lock` for a Flutter application and the Gradle wrapper for Android.
- 🍏 For iOS, use macOS with a supported Xcode version, command-line tools, and CocoaPods where required by your project.
- 🪪 Create Google Play Console and Apple Developer / App Store Connect accounts. Register unique application IDs and bundle IDs for each app you intend to publish.
- 🛡️ Use protected branches, pull-request review, separate development/staging/production environments, and a documented versioning convention.

```bash
flutter doctor -v
flutter pub get
flutter analyze
flutter test
flutter run
```

---

## 🏗️ Build types, without the confusion

A build type describes how your app is compiled. An environment describes where it connects. These are different decisions.

| Mode | Purpose |
|---|---|
| 🐞 **Debug** | Fast development, debugging enabled, and a debug signing key on Android. Never distribute a debug build as a production release. |
| 📊 **Profile** (Flutter) | Performance measurement on real devices, with some profiling tools retained. It is not your store distribution build. |
| 🚀 **Release** | Optimized compilation, debugging disabled, and production signing. Android shrinking and obfuscation with R8 are configuration-dependent—not automatic for every project. |

- Native Android typically starts with debug and release. Flutter exposes debug, profile, and release build modes. Keep custom Gradle build types compatible with Flutter’s tooling.

```bash
flutter run --debug
flutter run --profile
flutter build appbundle --release
flutter build ipa --release
```

> [!NOTE]
> Use a physical device to evaluate profile/release performance. Simulator and emulator behavior is not a substitute for real-device testing.

---

## 🎨 One codebase. Multiple flavors.

A product flavor is a version of your app: development, staging, production—or a different brand. A build variant combines a product flavor with a build type.

| Flavor + Type | Variant | Use |
|---|---|---|
| `dev` + `debug` | **devDebug** | Local development against a development API. |
| `staging` + `release` | **stagingRelease** | Production-like QA, signed for controlled distribution. |
| `prod` + `release` | **prodRelease** | The store-ready production application. |

- Give non-production Android flavors an `applicationIdSuffix` and a distinct app name. On iOS, use shared Xcode schemes and matching Debug/Profile/Release configurations with separate bundle IDs.
- Multiple flavor dimensions can combine environment and brand. Every combination adds build and testing cost; only create variants that you actually need.
- Dart defines configure Dart code; they do not create native flavors. Anything bundled in the app, including a Dart define, is readable by users and must not contain secrets.

```kotlin
// android/app/build.gradle.kts — inside android { ... }
flavorDimensions += "environment"
productFlavors {
    create("dev") {
        dimension = "environment"
        applicationIdSuffix = ".dev"
        resValue("string", "app_name", "My App Dev")
    }
    create("staging") {
        dimension = "environment"
        applicationIdSuffix = ".staging"
    }
    create("prod") {
        dimension = "environment"
    }
}

// Reference @string/app_name in AndroidManifest.xml.
```

> [!NOTE]
> For iOS, configure matching shared schemes in Xcode before using `--flavor`. Android Gradle flavors do not automatically configure iOS.

---

## 💻 Develop locally, integrate often

Keep changes small and your feedback loop short. Each pull request should tell reviewers what changed and how it was tested.

- 🌿 Create a short-lived feature branch from `main`. Run your development flavor, then format, analyze, and test before pushing.
- 🔍 Open a pull request. Require CI checks and review before merging; do not let a release job run on untrusted pull-request code.
- 🧾 Keep environment configuration explicit. Define `API_BASE_URL` through `String.fromEnvironment` in Dart; validate missing configuration early.
- 🏷️ Build release candidates from an immutable commit or tag. Record the commit SHA, SDK versions, flavor, version, and artifact checksum.

```bash
git switch -c feature/sign-in
flutter run --flavor dev --target lib/main_dev.dart
dart format --output=none --set-exit-if-changed .
flutter analyze
flutter test
git push -u origin feature/sign-in
```

---

## 🧪 Test in layers

Fast tests run on every change. Slower, device-dependent tests protect the release boundary.

- 🧹 **Static checks:** formatting, `flutter analyze`, Android lint, and dependency/security checks.
- 🔬 **Unit tests:** isolated business logic and services. **Widget tests:** UI states, validation, and interactions.
- 🔗 **Integration tests:** critical journeys such as login and checkout on emulators or physical devices. Flutter’s `integration_test` package is a common starting point.
- 🤖 **Native Android:** `./gradlew test` runs local unit tests; `./gradlew connectedAndroidTest` needs a running emulator or device.
- ✅ **Release verification:** install signed candidates and test deep links, notifications, offline behavior, accessibility, upgrades, and environment selection.
- 📈 Track coverage as a signal, not a guarantee. Favor meaningful assertions over an arbitrary percentage.

```bash
flutter test --coverage
# Run with an emulator/device available:
flutter test integration_test/app_test.dart --flavor staging

# Native Android, from android/ (task names depend on variants):
./gradlew test lint
./gradlew connectedStagingDebugAndroidTest
```

---

## ⚙️ Automate with GitHub Actions

A workflow is a YAML file in `.github/workflows/`. Events trigger workflows; jobs run on machines called runners; steps execute commands or reusable actions.

- 🔐 Use `pull_request` for verification and trusted tags or manual dispatch for releases. Set minimal token permissions and protect production with environment approval.
- 🖥️ Linux runners handle Android builds. iOS builds require macOS runners with Xcode. Build each flavor explicitly rather than guessing from the branch name.
- 📌 Pin the Flutter and Java versions to your project’s tested toolchain. Cache dependencies, but never treat a cache as your source of truth.
- 📤 Upload artifacts for QA and traceability. Signing and store upload are separate release stages; the example below intentionally builds an unsigned QA APK.
- 🧷 For stronger supply-chain protection, pin third-party actions to reviewed full commit SHAs. Major-version tags below are readable examples, not immutable pins.

```yaml
# .github/workflows/ci.yml
name: Mobile CI
on:
  pull_request:
  push:
    branches: [main]
permissions:
  contents: read
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  verify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17' # Match your AGP requirements
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.35.4' # Example: pin your tested SDK
          channel: stable
          cache: true
      - run: flutter pub get
      - run: dart format --output=none --set-exit-if-changed .
      - run: flutter analyze
      - run: flutter test --coverage
      - run: flutter build apk --debug --flavor staging --target lib/main_staging.dart
      - uses: actions/upload-artifact@v4
        with:
          name: staging-debug-${{ github.sha }}
          path: build/app/outputs/flutter-apk/*staging*debug.apk
          if-no-files-found: error
```

> [!NOTE]
> Adapt flavor names, entrypoints, and SDK versions to your project. A release workflow must restore signing credentials securely and require approval before publishing.

---

## 🐳 Reproducible builds with Docker

Docker packages a Linux build environment so local development and CI can use the same tools. It is useful for Android—not a replacement for a Mac.

- 🧱 Build a reviewed base image containing pinned Flutter, JDK, Android SDK command-line tools, platforms, and build-tools. Pin its digest and record the tool versions.
- 📝 Accept Android SDK licenses as part of image setup when permitted. Keep dependency caches outside the immutable image for faster builds.
- 🚫 Never `COPY` signing keys or service account credentials into an image. Inject them temporarily at runtime, restrict file permissions, and delete them afterward.
- 📵 Run emulators in suitably configured hosts/device farms; an ordinary container does not guarantee virtualization or USB access.
- 🍎 Linux Docker containers cannot build or sign iOS apps. Use a macOS runner with Xcode for the iOS stage.

```dockerfile
# Dockerfile — use your team's vetted mobile toolchain image
ARG MOBILE_CI_IMAGE
FROM ${MOBILE_CI_IMAGE}
WORKDIR /workspace
COPY pubspec.yaml pubspec.lock ./
RUN flutter pub get
COPY . .
CMD ["flutter", "test"]

# Pass a real, digest-pinned image containing Flutter + Android SDK:
docker build --build-arg MOBILE_CI_IMAGE=your-image@sha256:YOUR_DIGEST -t mobile-ci .
docker run --rm mobile-ci
```

> [!NOTE]
> This is a template, not a ready-made toolchain image. Add `.git`, `build/`, `.dart_tool/`, keystores, and credential files to `.dockerignore`.

---

## 🔐 Sign, version, and protect

Signing establishes who produced the app. Versioning tells stores and devices whether a build is newer. Both belong in your release plan.

- 🤖 **Android:** use a private upload keystore and configure the release `signingConfig`. With Play App Signing, Google holds the app signing key; your upload key authenticates uploads. Back up keys securely.
- 🍎 **iOS:** configure an Apple distribution certificate and matching provisioning profile, or a controlled automatic-signing workflow. An App Store Connect API key authorizes uploads; it does not replace code signing.
- 🗝️ Store passwords, keystores, certificates, and API credentials in GitHub environment secrets or an approved secret manager. Never commit them or print them in logs.
- 🔢 Flutter `pubspec.yaml` uses `version: 1.2.0+42`. The name maps to the user-visible version, and the number to Android `versionCode` / iOS build number.
- ⬆️ Increase Android `versionCode` for every new upload and follow App Store Connect’s build-number requirements. Use a monotonic release counter, not an uncoordinated per-workflow counter.
- 🧹 On trusted release jobs, decode credentials into temporary files, sign, upload, then clean up. Do not expose production secrets to fork pull requests.

```bash
# After release signing has been configured:
flutter build appbundle --release --flavor prod \
  --target lib/main_prod.dart --build-name 1.2.0 --build-number 42

# On macOS, with Xcode signing and prod scheme configured:
flutter build ipa --release --flavor prod \
  --target lib/main_prod.dart --build-name 1.2.0 --build-number 42
```

---

## 🤖 Publish to Google Play

Use an Android App Bundle (`.aab`) for a new Google Play app. An APK is useful for direct installation and QA; an AAB lets Play generate device-specific APKs.

- 🏪 Create your app in Play Console. Complete your store listing, screenshots, content rating, target audience, ads declarations, privacy policy, and Data safety details.
- ✍️ Enroll in Play App Signing and build a production AAB with your upload key. Confirm the application ID, `versionCode`, and current target API requirements.
- 🧪 Start with internal testing. For automated uploads, enable the Google Play Developer API and grant a service account only the required Play Console permissions.
- 📤 Upload using Play Console or a configured tool such as `fastlane supply`. New apps may need initial manual setup/upload before API-based releases work.
- 📈 Promote the tested version to closed/open testing as appropriate, then production. Some new personal accounts must meet testing requirements before requesting production access; check your account’s Console guidance.
- 🚦 Submit required changes for review. Use managed publishing when you want to control the go-live time after approval. Start a staged production rollout, monitor health, then expand.

```bash
# With a configured Fastfile and protected release credentials:
# supply accepts the AAB path and track; configure these explicitly.
bundle exec fastlane supply \
  --aab path/to/prod-release.aab \
  --track internal \
  --package_name com.example.app \
  --json_key path/to/temporary-service-account.json
```

> [!NOTE]
> Store requirements change. Confirm target API, testing eligibility, and policy declarations in Play Console before each release.

---

## 🍎 Publish to the App Store

iOS distribution passes through Xcode signing, App Store Connect processing, TestFlight, and App Review. Uploading is only the first milestone.

- 🆔 Register the bundle ID and create the app record in App Store Connect. Set up certificates, capabilities, provisioning, and the production Xcode scheme.
- 📦 Archive and export a signed App Store distribution IPA on macOS. Upload via Xcode, Transporter, or a configured `fastlane pilot` workflow with scoped credentials.
- ⏳ Wait for processing and resolve export-compliance questions. Distribute to internal TestFlight testers; external testing can require Beta App Review.
- 🖼️ Prepare screenshots, description, age rating, privacy disclosures, support/privacy URLs, and review notes. Include a working review account when sign-in is required.
- 📝 Select the processed build, submit the app version for review, and address feedback. Choose manual, automatic, or scheduled release as appropriate; phased release can gradually deliver automatic updates.
- 🕐 Account for both Apple review time and your own release approval. Do not promise that an upload will become public immediately.

```bash
# After configuring App Store Connect auth and signing:
# Run on macOS; the IPA path depends on your export.
bundle exec fastlane pilot upload \
  --ipa path/to/MyApp.ipa \
  --skip_waiting_for_build_processing true

# This uploads to TestFlight; it does not submit App Review.
```

---

## 🚀 Release with confidence

Promote deliberately, observe real users, and be ready to pause. Publishing is the start of the operational lifecycle—not the end.

| Stage | Checklist |
|---|---|
| 🏗️ **Before build** | Reviewed commit, passing checks, correct flavor/API endpoint, approved dependencies, updated version and changelog. |
| 📤 **Before upload** | Signed artifact, verified package/bundle ID, real-device QA, symbols/mapping files retained for crash diagnosis, no embedded secrets. |
| 🌍 **Before publishing** | Complete store metadata/privacy disclosures, approved release owner, production monitoring and support ready. |
| 📡 **After publishing** | Monitor crashes, Android ANRs, startup time, API errors, and user feedback by release version. Expand rollout only when health is acceptable. |

- 🛑 If a regression appears: halt a staged/phased rollout and use server-side feature flags where available. Users already updated are not automatically downgraded; ship a fixed build with a higher version.
- 🗂️ Keep an audit trail of approval, artifact checksum, commit SHA, version, upload time, and store status. Feed lessons back into tests and the next development cycle.

> [!IMPORTANT]
> Build once per platform and flavor, then promote that exact artifact between compatible store tracks. Do not rebuild the same version during promotion.

---

## 🗺️ Pipeline diagram

<img width="1184" height="248" alt="CICD Pipeline diagram" src="https://github.com/user-attachments/assets/aac098e3-b5e5-4521-8388-ef6696f9a9a7" />

---

## 📖 Official references

- [Flutter flavors](https://docs.flutter.dev/deployment/flavors)
- [Flutter Android deployment](https://docs.flutter.dev/deployment/android)
- [Flutter iOS deployment](https://docs.flutter.dev/deployment/ios)
- [Android build variants](https://developer.android.com/build/build-variants)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Google Play Console help](https://support.google.com/googleplay/android-developer/)
- [App Store Connect help](https://developer.apple.com/help/app-store-connect/)

<div align="center">

<sub>Examples are educational templates. Configure identifiers, credentials, versions, and toolchains for your own project.</sub>

</div>
