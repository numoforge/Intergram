# Building Intergram

Intergram inherits Telegram for Android's build requirements. They are strict and
bleeding-edge: an older Android Studio will not even sync the project.

## 1. Requirements

| Component | Version | Where it is pinned |
|---|---|---|
| JDK | 21 minimum, 25 used by CI | `build.gradle` (`JavaVersion.VERSION_21`), `.github/workflows/build.yml` |
| Gradle | 9.8.2 | `gradle/wrapper/gradle-wrapper.properties` |
| Android Gradle Plugin | 9.5.1 | root `build.gradle` |
| compileSdk / buildTools | 37 / 37.0.0 | root `build.gradle` |
| NDK | 27.3.13750724 | root `build.gradle` |
| CMake | 3.22.1 | `TMessagesProj/build.gradle` |
| minSdk / targetSdk | 23 / 36 | root `build.gradle` |

Practical machine sizing: the native build compiles ~5,000 C/C++ files across four ABIs.
Upstream CI uses a 32 vCPU runner with a 56 GB Gradle heap and a 10 GB ccache. A 4-core
machine can build a single ABI, slowly, with ccache enabled.

## 2. Checkout

```bash
git clone --recursive --shallow-submodules https://github.com/numoforge/Intergram.git
cd Intergram
```

Submodules are mandatory. `--shallow-submodules` keeps the download reasonable; without
submodules the CMake step fails immediately.

## 3. Secrets and local configuration

Two files are git-ignored and must be created locally. **Never commit them.**

### `local.properties` (repository root)

```properties
sdk.dir=/path/to/android-sdk
apiId=12345
apiHash=0123456789abcdef0123456789abcdef
sentryDsn=
helperBotId=0
helperBotUsername=
mapsApiKey=
```

- `apiId` / `apiHash` — Intergram's own credentials from <https://my.telegram.org> (API
  development tools). Without them the build succeeds but login fails; upstream credentials must
  never be used.
- `sentryDsn`, `helperBot*` — upstream services. Leave empty; Intergram does not ship their
  credentials.
- `mapsApiKey` — only needed for the Maps SDK; empty is fine.

### `TMessagesProj_App/google-services.json`

Required by the `com.google.gms.google-services` plugin. Create an Intergram Firebase project
with two Android apps, `app.intergram.android` and `app.intergram.android.beta`, enable Firebase
Cloud Messaging, and download the file.

For CI-only compilation the workflow generates a structural placeholder when the
`GOOGLE_SERVICES_JSON_B64` secret is absent. The APK then builds but receives no push
notifications — that is expected and logged as a workflow warning.

### Signing (release only)

`TMessagesProj_App/build.gradle` reads `storeFile`, `storePassword`, `keyAlias`, `keyPassword`
from `local.properties`. Without them, `assembleRelease` still works and is signed with the
debug key — fine for testing, unusable for distribution. Release signing is deferred (P5).

## 4. Build

```bash
./gradlew assembleDebug        # all ABIs + universal (slow, several GB)
./gradlew assembleDebug -Pandroid.injected.build.abi=arm64-v8a   # one ABI (recommended)
./gradlew bundlePlay           # Play bundle (release)
```

Enable ccache for repeat native builds — `TMessagesProj/build.gradle` detects it on `PATH`:

```bash
export CCACHE_MAXSIZE=5G
```

## 5. CI

`.github/workflows/build.yml` runs on `workflow_dispatch` (ABI and variant selectable) and on
pushes to `main` that touch anything outside `docs/` and `*.md`.

What it does:

1. frees ~20 GB on the runner (removing preinstalled SDKs the build does not use);
2. checks out with recursive submodules;
3. installs JDK 25, Android SDK platform 37, build-tools 37.0.0, NDK 27.3.13750724, CMake 3.22.1;
4. writes `local.properties` from optional secrets (`TELEGRAM_API_ID`, `TELEGRAM_API_HASH`,
   `SENTRY_DSN`);
5. writes `google-services.json` from `GOOGLE_SERVICES_JSON_B64` or a documented placeholder;
6. builds one ABI and uploads the APK as a workflow artifact (14 day retention).

Optional repository secrets:

| Secret | Effect when absent |
|---|---|
| `TELEGRAM_API_ID`, `TELEGRAM_API_HASH` | build succeeds, login impossible |
| `GOOGLE_SERVICES_JSON_B64` | placeholder Firebase config, no push notifications |
| `SENTRY_DSN` | crash reporting disabled |

## 6. Troubleshooting

| Symptom | Cause |
|---|---|
| `No space left on device` | four-ABI build on a 14 GB runner. Build one ABI; the workflow already does. |
| `File google-services.json is missing` | the Firebase plugin is applied to `TMessagesProj_App`; the file must exist there. |
| `API_ID_INVALID` at login | `apiId`/`apiHash` are empty or do not belong to Intergram's package. |
| CMake errors about missing headers | submodules were not initialised. |
| Gradle sync fails in Android Studio | the IDE is older than AGP 9.5.1 / Gradle 9.8.2. |
| Sentry upload task asks for credentials | `sentryUploadSourceBundle*` tasks need `sentry.properties`; they are skipped for `debug` builds, which is what CI produces. |
