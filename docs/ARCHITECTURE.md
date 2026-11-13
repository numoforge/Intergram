# Intergram architecture

This document describes the codebase Intergram is built on. It was verified against the
imported upstream tree (`Nekogram/Nekogram` `v13.2.2`, commit `9c42485`, tree `814a1e4`) and is
not a design proposal — Intergram keeps this architecture.

## 1. Shape of the project

Intergram is a Gradle multi-project build that produces one APK. The app is **Java**, not Kotlin,
and it is a refinement of Telegram for Android's single-activity architecture.

```
settings.gradle          modules: TMessagesProj, TMessagesProj_App, jlatexmath,
                         baselineprofile, media3 (composite build from a submodule)
gradle.properties        APP_PACKAGE, APP_VERSION_CODE/NAME, TG_APP_VERSION_*
build-logic/             Gradle convention plugins: TL/RPC codegen, string codegen,
                         Lottie metadata, build-type wiring
TMessagesProj/           the client library: ~2,370 Java sources, JNI/C++, all resources
TMessagesProj_App/       thin application shell: applicationId, signing, ABI splits,
                         Firebase/Sentry plugin wiring
TMessagesProj_AppTests/  instrumentation tests (688 Kotlin files) + `tlscheme` input
TMessagesProj_Modules/   `media` (Media3/ExoPlayer fork) — a required git submodule
baselineprofile/         Baseline Profile module used for startup performance
jlatexmath/              LaTeX rendering (submodule)
```

Verified file inventory (14,419 tracked blobs):

| Extension | Count | Notes |
|---|---|---|
| `.java` | 2,380 | 2,368 in `TMessagesProj/src/main/java` |
| `.kt` | 722 | 688 instrumentation tests, 32 build-logic, 2 in the app |
| `.c` / `.cc` / `.h` / `.cpp` | ~4,970 | native: MTProto crypto, media, VoIP, widgets |
| `.xml` | 719 | layouts, 49 locale folders, manifest |
| `.webp` / `.svg` / `.png` | 4,071 / 46 / 392 | emoji, sticker and icon assets |

## 2. Java package layout

```
org/telegram/ui/            1,586 files   screens, cells, custom-drawn views
org/telegram/messenger/       322 files   controllers, loaders, workers, notifications
org/telegram/tgnet/            42 files   MTProto transport, TL serialization
org/telegram/utils/            79 files   helpers (permissions, media, contacts)
org/telegram/SQLite/                      SQLite wrapper over the platform database
tw/nekomimi/nekogram/         ~90 files   Nekogram-specific additions
```

`tw.nekomimi.nekogram` is **kept on purpose**: it is the internal Java package of the upstream
codebase, referenced by ~188 files. Renaming it would break settings migration, backups and every
future upstream merge, so Intergram changes only the *application id* (`app.intergram.android`)
and user-visible branding.

## 3. Runtime architecture

- **Entry point** — `ApplicationLoader` builds the app; `LaunchActivity` is the single task entry
  and owns the global UI (drawer, fragment stack, action bar). Everything else is a `Fragment`.
- **Controllers** — `MessagesController` (data + business rules), `MessagesStorage`/`SQLite`
  (persistence), `ConnectionsManager` (MTProto connections), `ContactsController`,
  `MediaController`, `LocationController` and friends. Controllers are per-account instances,
  held in arrays of four (`UserConfig.MAX_ACCOUNT_COUNT`).
- **Accounts** — `UserConfig` selects the active account; `AccountInstance` bundles per-account
  controllers. Multiple accounts are a first-class part of the architecture, not a wrapper.
- **Network** — `tgnet` implements MTProto 4.x (abridged/intermediate/full transports, DH key
  exchange, session persistence) plus a separate `ConnectionsManager` HTTP path for CDN and
  file transfers.
- **Media** — `FileLoadOperation`/`FileLoader` implement resumable upload and download;
  `ImageLoader` owns bitmap caching and decoding; Media3/ExoPlayer (from the `media` submodule)
  plays audio, video and streams.
- **UI** — most screens are custom `View`s measuring and drawing themselves rather than using
  stock widgets. This is why UI changes are expensive and why Intergram does not rewrite them.
- **Settings** — `SharedConfig` for Telegram settings, `NekoConfig` for the Nekogram layer, both
  persisted to preferences and (optionally) Telegram cloud config.
- **Extras** — `tw.nekomimi.nekogram.Extra` reads `BuildConfig` values (API id/hash, Sentry DSN,
  helper bot) that Gradle injects from `local.properties`.

## 4. Native layer

CMake 3.22.1 builds two targets, `tmessages.49` and `twidget`, with `c++_static` and
`ANDROID_PLATFORM=android-23`. Sources come from 20 git submodules, including:

| Submodule | Purpose |
|---|---|
| `tdlib/td` | TDLib components (TL parsing, some MTProto pieces) |
| `FFmpeg/FFmpeg`, `libvpx`, `dav1d` | media decoding/encoding |
| `TelegramMessenger/tgcalls` + `webrtc` | voice/video calls |
| `google/boringssl` | TLS/crypto |
| `xiph/opus`, `opusfile`, `ogg` | voice messages |
| `wasm-micro-runtime` | widget engine sandbox |
| `TelegramMessenger/twidget`, `widget-sandbox` | home-screen widgets |
| `dkaraush/tlottie`, `libprisma`, `wallet-engine` | Lottie rendering, video effects, TON wallet |

Without initialised submodules the native build cannot start, so any checkout must use
`--recursive`.

## 5. Build graph and outputs

- Root `build.gradle` sets a common Android configuration for all modules: `compileSdk 37`,
  `buildTools 37.0.0`, `NDK 27.3.13750724`, `minSdk 23`, `targetSdk 36`, Java 21 bytecode and
  core-library desugaring.
- `TMessagesProj_App` applies `com.google.gms.google-services`, `io.sentry.android.gradle` and
  `androidx.baselineprofile`, reads `APP_PACKAGE`/`APP_VERSION_*` from `gradle.properties`, and
  produces ABI splits (`armeabi-v7a`, `arm64-v8a`, `x86`, `x86_64` + universal), each with
  `versionCode × 10`.
- `debug` gets the `.beta` application-id suffix and is not minified; `release` shrinks, minifies
  with `TMessagesProj/proguard-rules.pro` and signs with `local.properties` keys when present
  (otherwise the debug key).

## 6. What Intergram changes, and what it never changes

Changed in P0/P1 (see `docs/UPSTREAM.md` for the exact log):

- `APP_PACKAGE` → `app.intergram.android`; app name, launcher icon and colour, brand strings and
  in-app links; repository CI and documentation.

Deliberately unchanged:

- every Java package, class and resource *name* (attribute values were rewritten only where they
  are user-visible text, never resource identifiers);
- Nekogram's feature code, settings layer and services;
- the native layer and third-party submodules.
