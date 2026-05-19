# Forge: Makers, Publishers & Distribution

## Makers

| Maker | Platform | Output |
|-------|----------|--------|
| `maker-squirrel` | Windows | `.exe` installer |
| `maker-dmg` | macOS | `.dmg` disk image |
| `maker-pkg` | macOS | `.pkg` (App Store compatible) |
| `maker-deb` | Linux | `.deb` |
| `maker-rpm` | Linux | `.rpm` |
| `maker-flatpak` | Linux | Flatpak |
| `maker-appx` | Windows | `.appx` (Microsoft Store) |
| `maker-wix` | Windows | `.msi` (WiX) |
| `maker-zip` | All | `.zip` |

```ts
import { MakerSquirrel } from '@electron-forge/maker-squirrel';
import { MakerDMG } from '@electron-forge/maker-dmg';
import { MakerDeb } from '@electron-forge/maker-deb';

makers: [
  new MakerSquirrel({ authors: 'My Company' }, ['win32']),
  new MakerDMG({}, ['darwin']),
  new MakerDeb({}, ['linux']),
],
```

## Publishers

```ts
import { PublisherGitHub } from '@electron-forge/publisher-github';

publishers: [
  new PublisherGitHub({
    repository: { owner: 'myorg', name: 'myapp' },
    draft: true,
    prerelease: false,
    generateReleaseNotes: true,
  }),
],
```

Also available: `publisher-s3`, `publisher-snapcraft`, `publisher-nucleus`.

## Code Signing

### macOS

```ts
packagerConfig: {
  osxSign: {
    identity: 'Developer ID Application: My Company (TEAMID)',
    hardenedRuntime: true,
    entitlements: 'entitlements.plist',
    'entitlements-inherit': 'entitlements.plist',
  },
  osxNotarize: {
    appleId: process.env.APPLE_ID!,
    appleIdPassword: process.env.APPLE_PASSWORD!,
    teamId: process.env.APPLE_TEAM_ID!,
  },
},
```

### Windows

```ts
new MakerSquirrel({
  certificateFile: process.env.WIN_CERT_FILE,
  certificatePassword: process.env.WIN_CERT_PASSWORD,
}),
```

## Auto-Updates

### Open-source apps (GitHub Releases)

```ts
import { updateElectronApp } from 'update-electron-app';
updateElectronApp(); // checks update.electronjs.org automatically
```

### Custom update server

```ts
import { autoUpdater } from 'electron';
autoUpdater.setFeedURL({ url: 'https://my-server.com/updates/latest' });
autoUpdater.checkForUpdates();
autoUpdater.on('update-downloaded', () => autoUpdater.quitAndInstall());
```

## CI/CD (GitHub Actions)

```yaml
jobs:
  build:
    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run make
      - run: npm run publish
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Cross-compilation is limited — use CI matrix builds for per-platform artifacts.

## Writing Custom Makers/Publishers

```ts
import { MakerBase } from '@electron-forge/maker-base';

class MakerCustom extends MakerBase<Config> {
  name = 'custom';
  defaultPlatforms = ['darwin'];
  isSupportedOnCurrentPlatform() { return process.platform === 'darwin'; }
  async make(opts) { return [outputPath]; }
}
```

> **Ref:** [Makers](https://www.electronforge.io/config/makers) · [Publishers](https://www.electronforge.io/config/publishers) · [Auto Updater](https://www.electronjs.org/docs/latest/api/auto-updater) · [Writing Plugins](https://www.electronforge.io/advanced/extending-electron-forge/writing-plugins)
