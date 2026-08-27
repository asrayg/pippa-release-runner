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

This deploys Pippa Web, Nuro Web, Clinician Hub, Dashboard, Legal, and Pixel
Studio in parallel.

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

These workflows upload builds to App Store Connect. They do not automatically
submit a release for App Review.

## Security

- The public repository contains release workflows only, not application code.
- `PIPPA_MONO_DEPLOY_KEY` has read-only access to the private source repository.
- Vercel and App Store Connect credentials are encrypted GitHub secrets.
- Workflows run only when manually dispatched; pull requests cannot deploy.
- Workflow permissions are restricted to read-only repository contents.
