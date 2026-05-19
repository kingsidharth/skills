# Code signing and notarization

Two distinct steps for macOS distribution:

1. **Code signing** — your Developer ID certificate proves the binary came from you and hasn't been modified.
2. **Notarization** — Apple scans the signed binary and issues a stapled ticket. Required for Gatekeeper to allow the app on macOS 10.15+.

Both are mandatory for distribution outside the Mac App Store. Skipping either gives users a "cannot be opened because Apple cannot check it for malicious software" error.

## Prerequisites

- Apple Developer Program membership ($99/year)
- Xcode + Command Line Tools (for `notarytool` and `codesign`)
- "Developer ID Application" certificate from developer.apple.com → Certificates → +
- App-specific password for your Apple ID (appleid.apple.com → Sign-In and Security → App-Specific Passwords)
- Your Apple Team ID (developer.apple.com → Membership)

## Local signing setup

Import the .p12 cert into Keychain Access (login keychain). electron-builder finds the right identity automatically when present.

```bash
export APPLE_ID="you@example.com"
export APPLE_APP_SPECIFIC_PASSWORD="abcd-efgh-ijkl-mnop"
export APPLE_TEAM_ID="ABCDE12345"

bun run dist
```

That's it for local — electron-builder signs, your `afterSign` script notarizes.

## CI signing

The cert lives as a base64 secret. Decode at build time:

```yaml
# .github/workflows/release.yml
- name: Setup signing
  env:
    CSC_CONTENT: ${{ secrets.MAC_CERTS_P12_BASE64 }}
    CSC_KEY_PASSWORD: ${{ secrets.MAC_CERTS_PASSWORD }}
  run: |
    echo "$CSC_CONTENT" | base64 --decode > /tmp/cert.p12
    echo "CSC_LINK=/tmp/cert.p12" >> $GITHUB_ENV

- name: Build and publish
  env:
    APPLE_ID: ${{ secrets.APPLE_ID }}
    APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
    APPLE_TEAM_ID: ${{ secrets.APPLE_TEAM_ID }}
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: bun run dist --publish always
```

Generate the base64:

```bash
base64 -i ~/Downloads/cert.p12 | pbcopy
```

## Notarization script

electron-builder's built-in notarize is unreliable. Run it explicitly via `afterSign`:

```js
// scripts/notarize.cjs
const { notarize } = require('@electron/notarize')

exports.default = async function notarizing(context) {
  const { electronPlatformName, appOutDir, packager } = context
  if (electronPlatformName !== 'darwin') return
  if (process.env.SKIP_NOTARIZE === 'true') return

  const appName = packager.appInfo.productFilename
  const appPath = `${appOutDir}/${appName}.app`

  console.log(`Notarizing ${appPath}…`)
  await notarize({
    tool: 'notarytool',
    appPath,
    appBundleId: packager.appInfo.id,
    appleId: process.env.APPLE_ID,
    appleIdPassword: process.env.APPLE_APP_SPECIFIC_PASSWORD,
    teamId: process.env.APPLE_TEAM_ID,
  })
  console.log('Notarized')
}
```

```yaml
# electron-builder.yml
afterSign: scripts/notarize.cjs
mac:
  notarize: false   # critical — we do it ourselves
```

`tool: 'notarytool'` is mandatory — `altool` was deprecated and stops accepting submissions Nov 2023.

## Notarization timing

Submission → Apple → response: typically 2-15 minutes, sometimes 30+. `@electron/notarize` polls until done. CI jobs need a generous timeout (45+ min for safety).

If notarization is consistently slow, check:
- App size (>200 MB submissions take longer)
- Network — large uploads from CI are bandwidth-bound
- Apple's status — sometimes the service is degraded

## Stapling

After notarization, the ticket is hosted at Apple. To work offline, "staple" the ticket onto the bundle:

`@electron/notarize` does this automatically (`staple: true` is default). Verify:

```bash
xcrun stapler validate dist/mac/YourApp.app
```

DMG itself doesn't need stapling — Gatekeeper checks the .app inside.

## DMG signing

Don't sign the DMG. Counterintuitive, but Apple's recommendation: signed-but-unnotarized DMGs trip Gatekeeper. Configure:

```yaml
dmg:
  sign: false
```

The .app inside is signed and notarized; that's what Gatekeeper checks.

(electron-builder versions ≥ 20.43.0 default this to false.)

## Keychain profile (alternative auth)

Instead of env vars, store credentials in the macOS keychain:

```bash
xcrun notarytool store-credentials "myapp-notary" \
  --apple-id "you@example.com" \
  --team-id "ABCDE12345" \
  --password "app-specific-password"
```

Then in the notarize script:

```js
await notarize({
  tool: 'notarytool',
  appPath,
  keychainProfile: 'myapp-notary',
})
```

Cleaner locally; less useful in CI (where there's no persistent keychain).

## App Store Connect API key

For automation-heavy setups, an API key avoids password rotation:

```js
await notarize({
  tool: 'notarytool',
  appPath,
  appleApiKey: '/path/to/AuthKey_XXX.p8',
  appleApiKeyId: 'XXXXXXXXXX',
  appleApiIssuer: 'aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee',
})
```

Best for CI. Store the .p8 as a secret.

## Verifying

After build, check signature and notarization:

```bash
codesign --verify --verbose=4 dist/mac/YourApp.app
spctl --assess --verbose=4 dist/mac/YourApp.app
xcrun stapler validate dist/mac/YourApp.app
```

`spctl` should report "accepted" with "Notarized Developer ID". If you see "rejected", read the error — common causes: missing entitlement, unsigned helper binary, library validation.

## Common errors

**"is not signed at all"** — your cert isn't in Keychain or `CSC_LINK` is wrong.

**"resource fork, Finder information, or similar detritus not allowed"** — extended attributes on files. Strip with `xattr -cr build/`.

**"The signature does not include a secure timestamp"** — happens offline; signing requires network access to Apple's timestamp server.

**Crash on launch with "Library not loaded ... not valid for use in process: mapping process and mapped file (non-platform) have different Team IDs"** — missing `com.apple.security.cs.disable-library-validation` entitlement.

**Crash on launch in V8** — missing `com.apple.security.cs.allow-jit` entitlement.

**Notarization rejected with "package is invalid"** — usually a helper binary that wasn't signed. electron-builder signs all helpers automatically; if you've added scripts/binaries via `extraResources`, they need their own signing pass.

**"You must first sign the relevant contracts online"** — log into App Store Connect, accept the latest agreement.
