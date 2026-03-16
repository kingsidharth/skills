# Distribution Reference

## Table of Contents
- Building for Production
- Platform Distribution Matrix
- macOS Distribution
- Windows Distribution
- Linux Distribution
- Android Distribution (Google Play)
- iOS Distribution (App Store)
- Code Signing
- CI/CD Pipelines
- Auto-Updates (Updater Plugin)
- Versioning

---

## Building for Production

```bash
pnpm tauri build              # Desktop (builds + bundles)
pnpm tauri android build      # Android APK/AAB
pnpm tauri ios build          # iOS IPA
```

### Split Build and Bundle

For more control (e.g., different bundle configs for App Store vs direct download):

```bash
# Build without bundling
pnpm tauri build --no-bundle

# Bundle specific formats
pnpm tauri bundle --bundles app,dmg

# Bundle with alternate config (e.g., App Store)
pnpm tauri bundle --bundles app --config src-tauri/tauri.appstore.conf.json

# Skip code signing (for local dev / CI without certs)
pnpm tauri build --no-sign
```

### Bundle Format Options

Use `--bundles` to specify formats (comma-separated):

| Format | Flag | Platform |
|---|---|---|
| macOS App Bundle | `app` | macOS |
| DMG | `dmg` | macOS |
| NSIS Installer | `nsis` | Windows |
| MSI Installer | `msi` | Windows |
| AppImage | `appimage` | Linux |
| Debian Package | `deb` | Linux |
| RPM Package | `rpm` | Linux |

---

## Platform Distribution Matrix

| Platform | Formats | Code Signing | Store |
|---|---|---|---|
| **macOS** | `.app` bundle, `.dmg` | Required (signing + notarization) | App Store |
| **Windows** | NSIS, MSI | Recommended | Microsoft Store |
| **Linux** | AppImage, `.deb`, `.rpm`, Snap, AUR | Optional | Snapcraft, AUR |
| **Android** | APK, AAB | Required | Google Play |
| **iOS** | IPA | Required | App Store |

---

## macOS Distribution

### App Bundle (.app)

The standard macOS application format. Produced by default when building on macOS.

Configure in `tauri.conf.json`:
```json
{
  "bundle": {
    "macOS": {
      "minimumSystemVersion": "10.13",
      "frameworks": [],
      "exceptionDomain": "",
      "signingIdentity": null,
      "entitlements": null
    }
  }
}
```

Full reference: https://v2.tauri.app/distribute/macos-application-bundle/

### DMG (Apple Disk Image)

For direct distribution outside the App Store. Requires both code signing and notarization.

```bash
pnpm tauri bundle --bundles app,dmg
```

Full reference: https://v2.tauri.app/distribute/dmg/

### App Store

Requires Apple Developer Program membership, App Sandbox entitlements, and a separate build configuration.

```bash
pnpm tauri bundle --bundles app --config src-tauri/tauri.appstore.conf.json
```

Full reference: https://v2.tauri.app/distribute/app-store/

### macOS Code Signing & Notarization

Required for all macOS distribution methods. Notarization is Apple's malware check — required for apps distributed outside the App Store.

Environment variables:
- `APPLE_SIGNING_IDENTITY` — your signing identity
- `APPLE_ID` — your Apple ID
- `APPLE_PASSWORD` — app-specific password
- `APPLE_TEAM_ID` — your team identifier

Full reference: https://v2.tauri.app/distribute/sign/macos/

---

## Windows Distribution

### NSIS Installer (Recommended)

The recommended Windows installer format. Produces a `.exe` setup file.

```bash
pnpm tauri bundle --bundles nsis
```

Configure in `tauri.conf.json`:
```json
{
  "bundle": {
    "windows": {
      "nsis": {
        "displayLanguageSelector": false,
        "installerIcon": "icons/icon.ico",
        "installMode": "currentUser"
      }
    }
  }
}
```

Full reference: https://v2.tauri.app/distribute/windows-installer/

### MSI Installer

Alternative format. Requires the VBSCRIPT Windows feature to be enabled.

```bash
pnpm tauri bundle --bundles msi
```

### Microsoft Store

Full reference: https://v2.tauri.app/distribute/microsoft-store/

### Windows Code Signing

Optional but strongly recommended. Prevents "Unknown Publisher" warnings.

Full reference: https://v2.tauri.app/distribute/sign/windows/

---

## Linux Distribution

### AppImage (Most Portable)

Self-contained format that runs on most Linux distributions. Bundles webkitgtk (~80–90MB compressed).

```bash
pnpm tauri bundle --bundles appimage
```

Full reference: https://v2.tauri.app/distribute/appimage/

### Debian Package (.deb)

For Debian/Ubuntu-based distributions.

```bash
pnpm tauri bundle --bundles deb
```

Configure dependencies in `tauri.conf.json`:
```json
{
  "bundle": {
    "linux": {
      "deb": {
        "depends": ["libwebkit2gtk-4.1-0"],
        "section": "utility"
      }
    }
  }
}
```

Full reference: https://v2.tauri.app/distribute/debian/

### RPM Package

For Red Hat/Fedora-based distributions.

Full reference: https://v2.tauri.app/distribute/rpm/

### Snapcraft

Distribute via the Snap Store.

Full reference: https://v2.tauri.app/distribute/snapcraft/

### Arch User Repository (AUR)

Full reference: https://v2.tauri.app/distribute/aur/

### Linux Code Signing

Full reference: https://v2.tauri.app/distribute/sign/linux/

---

## Android Distribution (Google Play)

Build Android App Bundles (AAB) for Play Store submission:

```bash
pnpm tauri android build
```

Requires:
- Android Studio with SDK/NDK installed
- `JAVA_HOME`, `ANDROID_HOME`, `NDK_HOME` environment variables
- Android signing keystore

### Android Code Signing

Generate a keystore and configure signing in your project:

Full reference: https://v2.tauri.app/distribute/google-play/ and https://v2.tauri.app/distribute/sign/android/

---

## iOS Distribution (App Store)

Build for iOS:

```bash
pnpm tauri ios build
```

Requires:
- Apple Developer Program membership
- Xcode with iOS SDK
- Code signing certificates and provisioning profiles

Full reference: https://v2.tauri.app/distribute/app-store/ and https://v2.tauri.app/distribute/sign/ios/

---

## Code Signing — All Platforms

| Platform | Guide | Required? |
|---|---|---|
| macOS | https://v2.tauri.app/distribute/sign/macos/ | Yes (signing + notarization) |
| Windows | https://v2.tauri.app/distribute/sign/windows/ | Recommended |
| Linux | https://v2.tauri.app/distribute/sign/linux/ | Optional |
| Android | https://v2.tauri.app/distribute/sign/android/ | Yes (for Play Store) |
| iOS | https://v2.tauri.app/distribute/sign/ios/ | Yes (always) |

---

## CI/CD Pipelines

### GitHub Actions

Tauri provides official GitHub Actions workflows for cross-platform builds.

Full reference: https://v2.tauri.app/distribute/pipelines/github/

### CrabNebula Cloud

A cloud service purpose-built for Tauri app distribution.

Full reference: https://v2.tauri.app/distribute/pipelines/crabnebula-cloud/

### General CI Tips

- Build on the target platform (macOS runner for macOS, Windows runner for Windows, Linux runner for Linux)
- Cache `src-tauri/target/` and `node_modules/` for faster builds
- For bun users: build frontend on Linux/macOS runner, transfer to Windows for Tauri build
- Use `--no-sign` flag for CI builds that don't need signing (e.g., PR checks)
- Set signing secrets as CI environment variables, never commit them

---

## Auto-Updates (Updater Plugin)

The Updater plugin provides in-app auto-update for desktop apps. See `references/plugins.md` for detailed setup.

Key workflow:
1. Generate signing keys: `pnpm tauri signer generate -w ~/.tauri/myapp.key`
2. Set `TAURI_SIGNING_PRIVATE_KEY` environment variable for builds
3. Configure endpoints in `tauri.conf.json` under `plugins.updater`
4. Build produces signed update artifacts (`.tar.gz` on macOS/Linux, installers on Windows)
5. Host the artifacts at your configured endpoint
6. The app checks the endpoint and applies updates

---

## Versioning

Set in `tauri.conf.json` (preferred):

```json
{
  "version": "1.2.3"
}
```

Falls back to `src-tauri/Cargo.toml`:

```toml
[package]
version = "1.2.3"
```

Some platforms have specific version format requirements — check individual distribution docs. For example, iOS requires semver without pre-release tags, and Android uses `versionCode` (integer) in addition to `versionName`.
