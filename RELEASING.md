# Releasing Stepsy

Releases are cut by pushing a version tag. GitHub Actions builds and signs both flavors and
drafts the release; publishing it stays a manual click.

## One-time setup

The release workflow signs the APKs on the runner, so the signing key has to be available as
repository secrets (*Settings → Secrets and variables → Actions*):

| Secret | Value |
| --- | --- |
| `KEYSTORE_BASE64` | `base64 -w0 keystore.jks` — the whole keystore, base64 encoded |
| `KEYSTORE_PASSWORD` | keystore password |
| `KEY_ALIAS` | alias of the key inside the keystore |
| `KEY_PASSWORD` | password of that key |

> Every release must be signed with the same key. Android refuses to install an update whose
> signature changed, so a lost keystore means every user has to uninstall and reinstall,
> losing their step history unless they exported a backup first.

To create a keystore if you do not have one yet:

```bash
keytool -genkey -v -keystore keystore.jks -alias stepsy -keyalg RSA -keysize 4096 -validity 10000
```

Keep it out of the repository (`*.jks` is already in `.gitignore`) and back it up somewhere safe.

## Cutting a release

1. **Bump the version** in `app/build.gradle`:

   ```groovy
   def appVersionCode = 21
   def appVersionName = "1.6.3"
   ```

   `versionCode` must increase by at least one for every published build; `versionName` is what
   users see and what the tag has to match.

2. **Write the changelog** at `fastlane/metadata/android/en-US/changelogs/<versionCode>.txt`
   (so, `21.txt` for the example above). F-Droid and IzzyOnDroid read this file, and the
   workflow reuses it as the GitHub release notes — the `#new` / `#changes` / `#fixed` markers
   become markdown headings.

3. **Commit and tag**:

   ```bash
   git commit -am "Release 1.6.3"
   git tag v1.6.3
   git push origin master
   git push origin v1.6.3
   ```

4. **Review and publish.** The workflow drafts a release with both APKs, a `SHA256SUMS.txt`, and
   the changelog. Check it under *Releases*, then hit Publish.

## What the pipeline does

`.github/workflows/release.yml`, triggered by a `v*` tag:

1. **verify** — parses `appVersionName` / `appVersionCode` out of `app/build.gradle`, refuses to
   continue if the tag disagrees with the version, if the changelog for that `versionCode` is
   missing, or if a signing secret is unset. All of this runs before any build, so a mistake
   costs seconds rather than a full Gradle run.
2. **build** — assembles `assembleFossRelease` and `assembleFullRelease`, verifies with
   `apksigner` that both APKs really carry a signature, records SHA-256 checksums, and deletes
   the decoded keystore from the runner.
3. **release** — drafts the GitHub release with both APKs attached. A version name containing a
   hyphen (`1.7.0-rc1`) is marked as a pre-release automatically.

Running the workflow manually (*Actions → Release → Run workflow*) executes **verify** and
**build** only. Use it to rehearse signing without publishing anything.

## Artifact names

`base.archivesName` in `app/build.gradle` names the build outputs, so no renaming happens in CI:

```
app/build/outputs/apk/foss/release/stepsy-v1.6.2-foss-release.apk
app/build/outputs/apk/full/release/stepsy-v1.6.2-full-release.apk
```

Published assets drop the redundant build type: `stepsy-v1.6.2-foss.apk` and
`stepsy-v1.6.2-full.apk`.

## Building a release locally

Signing is driven entirely by the environment. Without a keystore the release build still
assembles — it is simply left unsigned — so contributors and F-Droid can build without secrets:

```bash
KEYSTORE_FILE=/path/to/keystore.jks \
KEYSTORE_PASSWORD=... KEY_ALIAS=... KEY_PASSWORD=... \
  ./gradlew assembleFossRelease assembleFullRelease
```

`KEYSTORE_FILE` is optional and defaults to `app/keystore.jks`.

## After publishing

- **IzzyOnDroid** tracks GitHub releases and picks up the `foss` APK on its own.
- **F-Droid** builds from source; the new `versionCode` and its changelog file are what its
  updater watches.
- **Obtainium** users following the GitHub repository get whichever flavor they picked.
