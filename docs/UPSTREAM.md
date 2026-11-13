# Upstream relationship

Intergram is a **standalone mirror** of `Nekogram/Nekogram`, not a GitHub fork. The full upstream
history lives in this repository's `main`, with Intergram commits applied on top.

## 1. Version mapping

| Intergram | Base | Telegram version | Upstream commit |
|---|---|---|---|
| (unreleased) | `v13.2.2` | 13.2.2 (`APP_VERSION_CODE 7303`) | `9c424851cf6a06b8e96a64bdee540c5debb38caa` (tree `814a1e4`) |

The import was verified by tree equality: the imported `main` tree is identical to the upstream
`v13.2.2` tree before any Intergram commit is applied.

`APP_VERSION_CODE` / `APP_VERSION_NAME` in `gradle.properties` follow upstream so that Telegram
API version negotiation and the About screen stay meaningful. Intergram's own release numbering
(once releases exist) will be derived from the base version, e.g. `1.0.0+13.2.2`.

## 2. Remotes

```bash
git remote add upstream https://github.com/Nekogram/Nekogram.git
git fetch upstream
```

Intergram is the `origin`. `upstream` is read-only and is never pushed to.

## 3. Sync procedure

```bash
git switch main
git fetch upstream
git merge --no-ff upstream/main        # or: git merge --no-ff upstream/<tag>
# resolve conflicts: branding files are the only ones we own
./gradlew assembleDebug -Pandroid.injected.build.abi=arm64-v8a   # or run CI
git push origin main
```

Rules learned from the initial import:

- **Never push upstream tags into this repository.** Tags point at upstream commits, and those
  commits contain upstream's `release.yml`, which would start a release job (and a Telegram
  upload attempt) against secrets that do not exist here. Branch-only sync avoids it entirely.
- Upstream's CI triggers on pushes to `dev`/`ci`; those branches are not mirrored for the same
  reason.
- Keep the branding delta small and confined to the files listed in §5 — every extra touched file
  is a future conflict.

## 4. What upstream provides that Intergram does not replace

These Nekogram services are still called by inherited code. They work, but they are not
Intergram's:

| Surface | Owner |
|---|---|
| translator backend (`app.nekogram.translator`) | Nekogram (Maven Central artifact) |
| GIF encoder (`app.nekogram.gifski`) | Nekogram |
| helper bot (`helperBotId`, `helperBotUsername`) | Nekogram — Intergram builds with `0`/empty, so the feature is inactive |
| donation screen (`NekoDonateActivity`) | Nekogram — inherited, unreviewed for Intergram (P3 decision) |
| official channel handle (`OfficialChannelUsername`) | set to `intergram`; must be registered by the project before release |

## 5. Modification log

Every Intergram change to upstream code is recorded here.

### P0 — repository establishment

| Change | Files |
|---|---|
| Import of upstream `v13.2.2` including full history (`main` tree identical to upstream `814a1e4`) | all |
| CI replaced with the Intergram workflow; upstream `release.yml` removed (it targets Nekogram's Telegram channel) | `.github/workflows/build.yml`, `.github/workflows/release.yml` |
| Upstream Crowdin project config and funding config removed | `crowdin.yml`, `.github/FUNDING.yml` |
| Documentation added | `README.md`, `docs/*.md`, `docs/assets/intergram-icon.png` |
| Secret hygiene: signing material ignored | `.gitignore` |
| GPL-2.0 license and copyright notices preserved unchanged | `LICENSE` |

### P1 — Intergram identity

| Change | Files |
|---|---|
| Application id `tw.nekomimi.nekogram` → `app.intergram.android` (internal Java package untouched) | `gradle.properties` |
| App name, beta name, settings section, notification, share/invite/update/QR/about strings; brand tokens replaced in 38 locale files (resource names untouched) | `TMessagesProj/src/main/res/values*/strings_neko.xml` |
| Launcher icon (5 densities) + adaptive foreground vector + background colour | `mipmap-*/ic_launcher.png`, `drawable/ic_launcher_foreground.xml`, `values/color_launcher.xml` |
| About line credits upstream instead of the Nekogram author | `AndroidUtilities.java`, `strings_neko.xml` |
| In-app links now point at `github.com/numoforge/Intergram` and its releases page | `BuildVars.java`, `ContactsController.java`, `NekoSettingsActivity.java` |

Not changed on purpose: Java packages, class and resource identifiers, Nekogram feature code,
native layer, third-party submodules, alternate launcher icons (`icon_2..9_launcher`), version
code scheme.

## 6. Licensing obligations

- Intergram remains **GPL-2.0**; `LICENSE` is untouched and every distributed build must ship
  with corresponding source.
- Copyright notices of Telegram, Nekogram and contributors stay in place.
- Telegram API access uses Intergram's own `apiId`/`apiHash`; upstream credentials are never
  reused or published.
- "Telegram" is used descriptively only; Intergram must not present itself as an official client.
