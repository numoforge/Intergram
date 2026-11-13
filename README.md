<p align="center">
  <img src="docs/assets/intergram-icon.png" width="120" alt="Intergram icon">
</p>

<h1 align="center">Intergram</h1>

<p align="center">A full-featured, independent Telegram client for Android.</p>

Intergram is a standalone open-source Telegram client built on the code of
[Nekogram](https://github.com/Nekogram/Nekogram) (GPL-2.0), which in turn is built on
[Telegram for Android](https://github.com/DrKLO/Telegram). It ships the complete Telegram
feature set — accounts, chats, groups, channels, media, calls, stickers, notifications,
multi-account and privacy settings — with an Intergram identity and its own release pipeline.

Intergram is **not affiliated with, endorsed by, or connected to Telegram Messenger LLP,
Telegram FZ-LLC, or the Nekogram project.** It is an unofficial third-party client.

## Status

| Phase | Scope | State |
|---|---|---|
| P0 | Repository, upstream import `v13.2.2`, sync path, documentation, CI | done |
| P1 | Intergram identity: application id, name, launcher icon, brand resources, links | done |
| P2 | Build verification with Intergram credentials (login, push) | not started |
| P3+ | Product work | not approved |

No Intergram APK has been published yet. Builds are produced by the
[Build Intergram](.github/workflows/build.yml) workflow; its artifacts are the only supported
source of APKs until a signed release exists.

## What is inherited

Everything Telegram's official Android client does, verified against the `v13.2.2` source tree:

- account login, registration, phone authentication and two-step verification
- private chats, groups, supergroups, channels, secret chats
- voice messages, photo/video/file upload and download, resumable transfers
- stickers, GIFs, animated and custom emoji
- contacts, global search, notifications, multiple accounts (up to 4)
- privacy and security settings, passcode, session management
- MTProto 4.x networking, VoIP, widgets, Wear OS companion

Plus the features Nekogram added on top (extra settings, translator, message details,
folder customisation, and more).

## Build and install

Intergram matches Telegram 13.2.2 and requires a current toolchain (JDK 21+, Android SDK 37,
NDK 27.3). See [docs/BUILD.md](docs/BUILD.md) for the full requirement list, the local build
recipe and the CI configuration.

A build that has no Intergram Telegram API credentials still compiles, but it cannot log in.
Intergram never ships or uses upstream credentials.

## Documentation

- [Architecture](docs/ARCHITECTURE.md) — modules, package layout, dependency graph, native layer
- [Build](docs/BUILD.md) — toolchain, secrets, local and CI builds, troubleshooting
- [Upstream](docs/UPSTREAM.md) — version mapping, sync procedure, modification log, licensing

## License and attribution

Intergram is distributed under the **GNU General Public License v2.0** (see [LICENSE](LICENSE)),
the same license as Nekogram and Telegram for Android. If you distribute a modified build, you
must publish the corresponding source under GPL-2.0 and keep the copyright notices intact.

Upstream projects this work derives from:

- [Nekogram/Nekogram](https://github.com/Nekogram/Nekogram) — GPL-2.0, the direct base (`v13.2.2`)
- [Telegram for Android](https://github.com/DrKLO/Telegram) — GPL-2.0, the original client
- the third-party libraries and native submodules listed in `.gitmodules`

Telegram is a trademark of Telegram Messenger LLP. Use of the Telegram API is subject to
Telegram's API Terms of Service; Intergram uses its own API credentials.
