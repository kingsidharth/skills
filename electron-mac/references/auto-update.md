# Auto-update

Two paths:

1. **`electron-updater`** (from electron-builder ecosystem) — recommended. Works with electron-builder publish targets, supports macOS+Windows+Linux, code-signature validation.
2. **`update-electron-app`** (from Electron team) — minimal wrapper around the built-in `autoUpdater` (Squirrel.Mac). Best when using Electron Forge with GitHub Releases.

Pick `electron-updater` if using `electron-builder` to package. Pick `update-electron-app` if using Electron Forge.

Both require code signing. Squirrel.Mac refuses to apply unsigned updates.

## electron-updater setup

```bash
bun add electron-updater
```

```ts
// main/updater.ts
import { autoUpdater } from 'electron-updater'
import log from 'electron-log'

autoUpdater.logger = log
log.transports.file.level = 'info'

autoUpdater.autoDownload = true
autoUpdater.autoInstallOnAppQuit = true

export function initUpdater() {
  if (!app.isPackaged) return  // skip in dev
  autoUpdater.checkForUpdatesAndNotify()
  setInterval(() => autoUpdater.checkForUpdates(), 60 * 60 * 1000) // hourly
}

autoUpdater.on('update-available', info => mainWin?.webContents.send('update:available', info))
autoUpdater.on('update-downloaded', info => mainWin?.webContents.send('update:ready', info))
autoUpdater.on('error', err => log.error('updater', err))
```

`autoInstallOnAppQuit: true` applies the update silently when the user quits — best UX. For an explicit "Restart now" button, call `autoUpdater.quitAndInstall()`.

## Publish config (electron-builder)

```yaml
# electron-builder.yml
publish:
  - provider: github
    owner: yourorg
    repo: yourapp
```

Or generic HTTPS:

```yaml
publish:
  - provider: generic
    url: https://updates.yourdomain.com/myapp/
    channel: latest
```

Or S3 / Spaces / Bunny / Keygen — all supported. See electron-builder's Publishers docs for the full list.

GitHub Releases is simplest: each release uploads `*.dmg`, `*.zip`, `latest-mac.yml` (and Windows equivalents). The updater fetches `latest-mac.yml` to compare versions.

## Required mac targets

```yaml
mac:
  target:
    - target: dmg
      arch: [universal]   # or [x64, arm64] for separate
    - target: zip         # required for auto-update on macOS
      arch: [universal]
```

`zip` is mandatory — Squirrel.Mac applies updates from a zip archive. Without it, `latest-mac.yml` won't be generated and updates fail silently.

## Update channels

`channel: 'beta'` in publish config emits `beta-mac.yml` instead of `latest-mac.yml`. Useful for staged rollouts:

```ts
autoUpdater.channel = user.optedIntoBeta ? 'beta' : 'latest'
autoUpdater.checkForUpdates()
```

Versions in beta channel: `1.5.0-beta.1`, `1.5.0-beta.2`. Stable: `1.5.0`. Update flow won't auto-promote a beta user to a stable release of the same version unless `allowDowngrade: true`.

## Staged rollouts

`stagingPercentage` in the publish provider lets a release reach only a subset:

```yaml
publish:
  - provider: github
    owner: yourorg
    repo: yourapp
```

Then in `latest-mac.yml`, set `stagingPercentage: 25`. electron-updater hashes the user's machine ID and compares — deterministically rolls out to ~25% of users.

## update-electron-app (Forge)

```ts
import { updateElectronApp } from 'update-electron-app'
updateElectronApp({
  repo: 'yourorg/yourapp',
  updateInterval: '1 hour',
})
```

Open-source apps on GitHub get update.electronjs.org for free. Closed-source needs a paid hosted service or DIY.

## Manual `autoUpdater` (built-in)

Only for advanced setups (custom signing flow, custom server). Skip unless you have specific needs.

```ts
import { autoUpdater } from 'electron'
autoUpdater.setFeedURL({ url: 'https://yourserver/update/darwin/' + app.getVersion() })
autoUpdater.checkForUpdates()
```

Squirrel.Mac expects a JSON response with at minimum `{ url: 'https://.../MyApp-1.2.3-mac.zip' }` when an update is available, or `204 No Content` when up to date.

## UX patterns

**Silent + on-quit**: most users prefer this. Set `autoInstallOnAppQuit: true`, never bother them.

**Notification + restart**: tell the user when the update has downloaded and offer an explicit restart. Don't surface "checking for updates" — that's noise.

**Forced update**: for security releases, refuse to start the app if the version is below a server-defined minimum. Implement this server-side via your own check before showing the main window.

```ts
const minVersion = await fetch('https://api.yourapp.com/min-version').then(r => r.text())
if (semver.lt(app.getVersion(), minVersion)) {
  await autoUpdater.downloadUpdate()
  autoUpdater.quitAndInstall(true, true)
}
```

## Things that break updates

- App is unsigned or has signature mismatch → silent failure.
- Renamed app bundle (`MyApp.app` → `My App.app`) between releases → Squirrel can't find the install location.
- `productName` change → same issue.
- ZIP target removed → no `latest-mac.yml`.
- DMG-only distribution + auto-update → DMG is for install, ZIP is for updates. Ship both.
- Network blocked corporate environments → no updates. Surface a clear "couldn't check for updates" state.
- App in Mac App Store → MAS handles updates; do NOT bundle electron-updater. Detect via `process.mas` and skip.
