---
name: shared-ci-cd
description: Add a private repo to the shared self-hosted GitHub Actions platform (Takida Linux runners plus a Mac mini Apple runner). Use when wiring a new project's CI, deploys, signing or store uploads, or when a self-hosted job is stuck or refused.
---

# Shared CI/CD: add a project

GitHub-hosted Actions isn't available on this account, and runners on a personal account
register per repo. Everything runs on the shared platform in
[`kingsidharth/hetzner_server_setup/ci`](https://github.com/kingsidharth/hetzner_server_setup/tree/production/ci);
its README has the full design. Never put secret values in a repo, a log or a PR.

## What runs where

| Host | Runs | Labels |
| --- | --- | --- |
| Takida (Linux, always on) | PR checks, web and Convex deploys, Android AAB, Windows cross-build | `[takida, node]`, `[takida, node-release]`, `[takida, android]`, `[takida, windows-cross]` |
| Mac mini (16 GB, keep light) | macOS sign and notarize, iOS/iPadOS TestFlight | `[apple]` |

- Takida: a fresh container per job, under a dedicated rootless Docker user (`ci-runner`).
  Images are `ci-node`, `ci-android` and `ci-windows`. Containers reach only the public internet.
- Mac: one launchd runner per repo, labels exactly `apple,macos-arm64`. A host-wide lock means
  only one Apple job runs at a time.
- **Trust tiers.** Every runner is one of two:
  - `check`: anything, including PRs. Never gets host secrets.
  - `release`: only `push`, `workflow_dispatch` or `release` events, on `refs/heads/main`
    or `refs/tags/v[0-9]*`. A runner hook enforces this, so a workflow can't bypass it.
    These runners may mount secrets. All Mac runners are release tier.
- There are no default labels, so a bare `runs-on: self-hosted` is never picked up.

## Add a repo

1. **The repo must be private.** Set Settings > Actions > "Run workflows from fork pull requests" off.
2. **Add runner lines** on Takida, in `/etc/ci-platform/runners.conf` (root, 0640). The columns are
   `instance repo image trust cpus memory labels secrets`. Typically:
   ```
   myapp-node      owner/myapp  node     check    4  8g   takida,node            -
   myapp-deploy    owner/myapp  node     release  2  4g   takida,node-release    -
   myapp-android   owner/myapp  android  release  8  16g  takida,android         myapp
   myapp-windows   owner/myapp  windows  release  4  8g   takida,windows-cross   -
   ```
   Only `release` lines may list secret scopes.
3. **Register each instance** from a machine where `gh` is logged in as the owner. The token
   travels on stdin and is never printed:
   ```bash
   gh api -X POST repos/OWNER/REPO/actions/runners/registration-token --jq .token \
     | ssh root@<takida-host> ci-platform register INSTANCE
   ```
   `ci-platform list` shows the state. `ci-platform sync` prints this command for every
   unregistered instance.
4. **Host secrets** (only for things that can't be GitHub secrets, such as the Android keystore)
   go in `/etc/ci-secrets/<scope>/` on Takida (root 0700). Jobs see them read-only at
   `/run/secrets/ci/<scope>/`.
5. **Mac runner** (Apple apps only), run on the Mac mini:
   `bash ci/mac/install-runner.sh OWNER/REPO`. Re-run it to update.
6. **Windows signing** (once Azure is set up): run `az login`, then
   `bash ci/azure/signing.sh trust OWNER/REPO`. This also makes the repo's `release`
   environment admit only `v*` tags. Then set the `AZURE_*` variables below.
7. **GitHub environments**: `development` (branch `main`), and `production` plus `release`
   (tags `v*`). Required reviewers aren't available on this plan, so the production gate is
   "publish the drafted GitHub release".
8. **Workflows**: copy morgwaith's `.github/workflows` and `.github/scripts`. Its `docs/ci.md`
   documents them. Steps must skip with a `::warning` naming any missing secret, never fail.

## Secret and variable names (names only)

- Environment `development`: `CONVEX_DEPLOY_KEY_DEV`
- Environment `production`: `CONVEX_DEPLOY_KEY_PROD`
- Environment `release`:
  - Apple, for notarization and TestFlight: `APPLE_API_KEY_P8` (the full `.p8` text), `APPLE_API_KEY_ID`, `APPLE_API_ISSUER`
  - B2: `B2_KEY_ID`, `B2_APPLICATION_KEY`
  - Play: `PLAY_SERVICE_ACCOUNT_JSON`
- Repo variables:
  - Build config: `CONVEX_URL`, `CONVEX_SITE_URL`, `SITE_URL`, `WORKOS_CLIENT_ID`
  - B2: `B2_BUCKET`, `B2_ENDPOINT`, `B2_CDN_BASE`
  - Windows signing: `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_SIGNING_ENDPOINT`,
    `AZURE_SIGNING_ACCOUNT`, `AZURE_SIGNING_PROFILE`, `AZURE_SIGNING_PUBLISHER`
  - Releases: `MIN_SUPPORTED_VERSION`, `PLAY_RELEASE_STATUS`
- Host files: `/etc/ci-secrets/<scope>/android/{upload.jks,keystore.properties}`, plus an
  optional `/etc/ci-secrets/<scope>/play/service-account.json`.
- Optional: put a fine-grained token (Administration: write, selected repos) in
  `/etc/ci-platform/github-token` on Takida. Every runner then becomes single-use (JIT).

Create Convex keys without displaying them:
`bunx convex deployment token create NAME --prod | gh secret set CONVEX_DEPLOY_KEY_PROD --env production -R OWNER/REPO`.

## Gotchas

1. **Private repos only.** A self-hosted runner on a public repo runs anyone's fork PR.
2. **Release tier means `main` or `v*` tags only.** A release-label job on a PR or another
   branch fails at "Set up runner" with "Release-tier runner refuses …". That's intended.
   To test one, merge it, then `gh workflow run X.yml --ref main`. On this plan anyone with
   write access can push a `v*` tag, so keep write access tight.
3. **Google download stall.** From Takida, some `dl.google.com` IPv4 addresses accept TCP but
   never send data. The host uses IPv6, but containers are IPv4-only. Anything that downloads
   from there at build time (AGP fetching SDK packages, `sdkmanager`) can hang with 0% CPU.
   Bake SDK packages into `ci-android` instead, and give jobs a `timeout-minutes`.
4. **The Mac runner shares the owner's account.** It runs as the owner, in the login session,
   with the unlocked keychain: launchd's `SessionCreate` is removed so `codesign` works, since
   otherwise you get `errSecInternalComponent`. Release-tier job code can therefore read the
   keychain and home directory. Also, anything on an external volume (work dirs, or caches
   such as `~/.npm` and `~/Library/Caches/electron` symlinked to `/Volumes/...`) hangs a
   launchd job in `open()`. The runner keeps its own caches under `~/actions-runner/_cache.noindex/`.
   For electron-builder, pass `-c.electronDownload.cache=$ELECTRON_CACHE`.
