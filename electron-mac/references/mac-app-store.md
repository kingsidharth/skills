# Mac App Store

MAS distribution is a separate target from Developer ID. Different certs, stricter sandbox, separate entitlements file, in-app purchases possible, no auto-updater needed (App Store handles updates).

Decide before starting: does your app *need* to be on MAS? It's a marketing channel and a trust signal, but the sandbox blocks many things Electron apps casually do (arbitrary file access, network servers, child processes outside the bundle).

## Certificates

Different from Developer ID:

- **3rd Party Mac Developer Application** — signs the .app
- **3rd Party Mac Developer Installer** — signs the .pkg installer for submission
- **Apple Distribution** (newer, replaces both above) — works for both

Generate via developer.apple.com → Certificates → +. Add to login keychain.

## electron-builder MAS target

```yaml
# electron-builder.yml
mac:
  target:
    - target: mas
      arch: [universal]

mas:
  category: public.app-category.productivity
  hardenedRuntime: false        # MAS uses sandbox instead
  entitlements: build/entitlements.mas.plist
  entitlementsInherit: build/entitlements.mas.inherit.plist
  provisioningProfile: build/embedded.provisionprofile
```

Build:

```bash
bunx electron-builder --mac mas
```

Output: `dist/mas-universal/YourApp-1.0.0.pkg` — upload via Transporter.app.

For testing on local Macs before submission (without going through the store):

```yaml
mac:
  target: mas-dev
mas:
  type: development
```

`mas-dev` builds use a development provisioning profile and run on Macs whose UDID is registered in your developer account.

## MAS entitlements

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
  <key>com.apple.security.app-sandbox</key><true/>
  <key>com.apple.security.application-groups</key>
  <array><string>TEAMID.com.yourorg.yourapp</string></array>

  <!-- Network -->
  <key>com.apple.security.network.client</key><true/>

  <!-- File access -->
  <key>com.apple.security.files.user-selected.read-write</key><true/>
  <key>com.apple.security.files.bookmarks.app-scope</key><true/>

  <!-- Hardware -->
  <key>com.apple.security.device.microphone</key><true/>
  <key>com.apple.security.device.camera</key><true/>

  <!-- Required for Electron under sandbox -->
  <key>com.apple.security.cs.allow-jit</key><true/>
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
</dict>
</plist>
```

`app-sandbox` is mandatory for MAS. Once set, your app is in a sandbox container at `~/Library/Containers/com.yourorg.yourapp/`.

## What the sandbox breaks

- **Arbitrary file paths**: `fs.readFile('/Users/x/Documents/foo.txt')` fails. User must pick the file via dialog. After picking, you have read/write to that *one* file.
- **Persistent file access across launches**: re-prompt every launch unless you save a security-scoped bookmark. See `app.getPath('userData')` for app-internal data — that's always allowed.
- **Network servers** on listening ports require `network.server` entitlement; some ports are still blocked.
- **`child_process.spawn`** outside your bundle is blocked.
- **`shell.openExternal`** to file:// URLs needs the user to have already granted access.
- **`updateElectronApp` / `electron-updater`** — disable. The App Store handles updates. Detect MAS via `process.mas`:

```ts
if (!process.mas) initUpdater()
```

## Detect MAS at runtime

```ts
if (process.mas) {
  // we're in MAS build
}
```

Set automatically by Electron when running from a MAS-built bundle.

## App Store rules likely to bite Electron apps

- **No external installers/updaters** — strip all auto-update code paths.
- **No arbitrary code execution** — Apple sometimes flags `eval`, `new Function`, dynamic `import()` of external URLs. Keep CSP tight.
- **Privacy strings** — every `NS*UsageDescription` must clearly explain user benefit. Vague strings ("for app function") are rejected.
- **Background modes** — running with no UI requires explicit declaration; menubar-only apps need `LSUIElement: true`.
- **Hardware access during review** — Apple's reviewers may not grant camera/mic prompts in review. Provide a path through the app that doesn't require hardware.
- **Login items** — must use `SMAppService` (newer macOS) or `app.setLoginItemSettings()`; don't drop launchd plists.

## Login items (launch at login)

```ts
app.setLoginItemSettings({
  openAtLogin: true,
  openAsHidden: true,    // launch into menubar without window
})
```

For MAS in macOS 13+, use the newer SMAppService API via a tiny native helper if needed. `app.setLoginItemSettings` works for most cases.

## Pricing and IAP

In-app purchases require StoreKit, which Electron does not expose. Options:

1. Native module wrapping StoreKit (`node-mas-purchase` etc.) — works, dated.
2. NAPI-RS Swift bridge — modern, write the StoreKit calls in Swift, expose via N-API.
3. Free with optional outside-the-store subscription — App Store rules forbid steering to external payment for digital goods, with narrow "reader app" exceptions.

For most teams, charging via Stripe outside the store is simpler than fighting StoreKit in Electron. Then the MAS version is free, and paid features unlock via account login.

## Submission flow

1. Build: `bunx electron-builder --mac mas`
2. Validate: open `Transporter.app`, drag the .pkg, run validation
3. Upload via Transporter
4. App Store Connect → My Apps → your app → submit a new version with build number from upload
5. Wait for review (1-3 days typical)

If rejected, the message is usually specific. Common rejections: missing privacy strings, attempts to access non-sandboxed resources, missing entitlement justifications, inconsistent metadata.

## Maintaining two builds

Many apps ship both MAS and Developer ID DMG (so power users get fresh updates, App Store users get the trust signal). Set up two electron-builder configs or use `--mac dmg mas` to build both in one CI run. Keep entitlement files separate.
