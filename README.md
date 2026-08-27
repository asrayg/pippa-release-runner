# Pippa release runner

Manual, public GitHub Actions workflows that build the private `pippa-mono`
repository with Xcode 26 and upload its iOS apps to App Store Connect.

The application source remains private. Releases are triggered manually from
the Actions tab and use a read-only deploy key plus encrypted repository
secrets.

## One-command website release

The `Deploy all websites to production` workflow deploys all six Vercel sites
in parallel:

```bash
gh workflow run deploy-all-websites.yml --repo asrayg/pippa-release-runner
```
