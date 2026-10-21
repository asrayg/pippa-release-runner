# Pippa release runner

This public repository provides manual production-release pipelines for the
private `asrayg/pippa-mono` repository. Application source stays private. The
pipelines use a read-only deploy key and encrypted GitHub secrets.

## Requirements

Install and authenticate the GitHub CLI once:

```bash
brew install gh
gh auth login
```

All commands below can be run from any directory.

## Deploy websites

The website workflow can deploy every production site in parallel or only one
site. It always checks out the latest `main` branch from `pippa-mono`.

### Deploy every website

```bash
gh workflow run deploy-all-websites.yml \
  --repo asrayg/pippa-release-runner \
  --field site=all
```

This deploys Pippa Web, Nuro Web, Clinician Hub, Dashboard, Legal, Pixel
Studio, and the companion app sites in parallel.

### Deploy one website

Replace the value after `site=` with one of the values in the table.

| Website | Command value |
| --- | --- |
| Pippa Web | `pippa-web` |
| Nuro Web | `nuro-web` |
| Clinician Hub | `clinician-hub` |
| Dashboard | `dashboard` |
| Legal | `legal` |
| Pixel Studio | `pixel-studio` |
| Companion app sites (Kipi, Soli, Momo, Lumi, Rumi, Miya, Nomi, join) | `companion-web` |

Examples:

```bash
# Dashboard only
gh workflow run deploy-all-websites.yml \
  --repo asrayg/pippa-release-runner \
  --field site=dashboard

# Pippa marketing website only
gh workflow run deploy-all-websites.yml \
  --repo asrayg/pippa-release-runner \
  --field site=pippa-web

# Clinician Hub only
gh workflow run deploy-all-websites.yml \
  --repo asrayg/pippa-release-runner \
  --field site=clinician-hub
```

### Run from the GitHub website

1. Open **Actions** in this repository.
2. Select **Deploy websites to production**.
3. Select **Run workflow**.
4. Choose `all` or one website from the dropdown.
5. Select the green **Run workflow** button.

### Watch the deployment

List recent website releases:

```bash
gh run list \
  --repo asrayg/pippa-release-runner \
  --workflow deploy-all-websites.yml \
  --limit 10
```

Open the newest release in your browser:

```bash
gh run view \
  --repo asrayg/pippa-release-runner \
  --workflow deploy-all-websites.yml \
  --web
```

If a deployment fails, print its failed step logs:

```bash
gh run view \
  --repo asrayg/pippa-release-runner \
  --workflow deploy-all-websites.yml \
  --log-failed
```

## Upload iOS apps

Upload Pippa to App Store Connect:

```bash
gh workflow run pippa-ios.yml --repo asrayg/pippa-release-runner
```

Upload Pippa For Clinicians to App Store Connect:

```bash
gh workflow run dashboard-shell-ios.yml --repo asrayg/pippa-release-runner
```

Upload Nuro to App Store Connect:

```bash
gh workflow run nuro-ios.yml --repo asrayg/pippa-release-runner
```

These workflows upload builds to App Store Connect. They do not automatically
submit a release for App Review.

## Upload Android apps

Upload Nuro to Google Play. The version code must be higher than every code
already uploaded for `com.nuro.app`, and the release lands as a draft on the
chosen track until you promote it in the Play Console:

```bash
gh workflow run nuro-android.yml \
  --repo asrayg/pippa-release-runner \
  --field version_name=1.0.0 \
  --field version_code=21 \
  --field track=internal \
  --field status=draft
```

This needs five secrets that the iOS pipelines do not use, on a `play-store`
environment:

| Secret | What it is |
| --- | --- |
| `NURO_ANDROID_KEYSTORE_BASE64` | The upload keystore, base64 encoded (`base64 -i nuro-upload.jks`) |
| `NURO_ANDROID_KEYSTORE_PASSWORD` | Keystore password |
| `NURO_ANDROID_KEY_ALIAS` | Key alias inside the keystore |
| `NURO_ANDROID_KEY_PASSWORD` | Key password |
| `NURO_PLAY_SERVICE_ACCOUNT_JSON` | Google Play service-account JSON, with Release Manager access to the app |

Google will not let the API create an app listing, and it rejects API uploads
for a package that has never had a release. **The very first build for
`com.nuro.app` has to be uploaded by hand in the Play Console**; this workflow
takes over from the second one.

The marketing version is pinned inside each workflow (`MARKETING_VERSION`);
bump it there when the app's version changes. The build number is the GitHub
run id, so it always increases and never collides with a previous upload.

Watch an upload:

```bash
gh run list --repo asrayg/pippa-release-runner --workflow nuro-ios.yml --limit 5
gh run view --repo asrayg/pippa-release-runner --workflow nuro-ios.yml --log-failed
```

## Companion apps (Soli, Kipi, Momo, Lumi, Rumi, Miya, Nomi)

Two workflows cover the seven Pippa companion apps. Each takes `app`, either
one app or `all` (the seven build in parallel, and one failing does not stop
the others). Bundle ids, package names and Xcode schemes are read from
`contracts/product-registry/release.json` on `main`. The Nuro-side structured
apps (Vela, Aven, Onda, Sero, Talo, Elio, Aster) are on hold and not included.

### iOS

```bash
gh workflow run companion-ios.yml --repo asrayg/pippa-release-runner \
  --field app=all --field marketing_version=1.0.0
```

- Signs with the same `BUILD_CERT_P12` certificate and App Store Connect key as
  Pippa and Nuro. Nothing new is needed to start.
- The App Store profile is made on every run through the App Store Connect API
  (named `<App> App Store (release runner)`, replacing the previous one). The
  run also switches on the bundle id's capabilities first: Push Notifications
  for all seven, plus HealthKit for Kipi. To pin a hand-made profile instead,
  set a `<APP>_APP_STORE_PROFILE` secret (base64), for example
  `KIPI_APP_STORE_PROFILE`.
- The archive is refused if it lost `aps-environment` (or HealthKit for Kipi).
- Build number is the GitHub run id. Bump `marketing_version` once a version is
  approved, or App Store Connect closes the train.

### Android

```bash
gh workflow run companion-android.yml --repo asrayg/pippa-release-runner \
  --field app=all --field version_name=1.0.0 \
  --field track=internal --field status=draft
```

- `version_code` is optional; it defaults to 1000 + the run number, which
  always goes up.
- Google rejects API uploads for a package with no release yet. For each app's
  first release, run with `--field upload=false`, download the signed `.aab`
  from the run's artifacts, and upload it by hand in the Play Console. Every
  later run can upload itself.
- The run checks that the bundle is signed, R8 ran, and the R8 keep rules held
  (`scripts/check-r8-keeps.mjs`).

| Secret | What it is |
| --- | --- |
| `FAMILY_GOOGLE_SERVICES_JSON` | One `google-services.json` with a Firebase client for every companion package (set). Push silently breaks without it, so the run fails instead. |
| `FAMILY_ANDROID_KEYSTORE_BASE64` | The shared upload keystore for the seven apps, base64 encoded |
| `FAMILY_ANDROID_KEYSTORE_PASSWORD` | Keystore password |
| `FAMILY_ANDROID_KEY_ALIAS` | Key alias inside the keystore |
| `FAMILY_ANDROID_KEY_PASSWORD` | Key password |
| `FAMILY_PLAY_SERVICE_ACCOUNT_JSON` | Optional. Play service account with Release Manager on the seven apps. Falls back to `PIPPA_PLAY_SERVICE_ACCOUNT_JSON`, which works once that account is granted the seven apps. |

Make the upload keystore once, keep a copy somewhere safe, and store it:

```bash
keytool -genkeypair -v -keystore family-upload.jks -alias family-upload \
  -keyalg RSA -keysize 4096 -validity 10000
base64 -i family-upload.jks | gh secret set FAMILY_ANDROID_KEYSTORE_BASE64 --repo asrayg/pippa-release-runner
gh secret set FAMILY_ANDROID_KEYSTORE_PASSWORD --repo asrayg/pippa-release-runner
gh secret set FAMILY_ANDROID_KEY_ALIAS --repo asrayg/pippa-release-runner --body family-upload
gh secret set FAMILY_ANDROID_KEY_PASSWORD --repo asrayg/pippa-release-runner
```

## Security

- The public repository contains release workflows only, not application code.
- `PIPPA_MONO_DEPLOY_KEY` has read-only access to the private source repository.
- Vercel and App Store Connect credentials are encrypted GitHub secrets.
- Workflows run only when manually dispatched; pull requests cannot deploy.
- Workflow permissions are restricted to read-only repository contents.
