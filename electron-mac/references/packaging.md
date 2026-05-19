# Packaging (electron-builder)

## Minimal config

```yaml
# electron-builder.yml
appId: com.yourorg.yourapp
productName: YourApp
copyright: Copyright © 2026 YourOrg

directories:
  output: dist
  buildResources: build

asar: true
asarUnpack:
  - "**/*.node"
  - "node_modules/sharp/**"
  - "node_modules/better-sqlite3/build/**"

files:
  - "out/**/*"                 # Next.js static export
  - "dist/main/**/*"           # tsup main process
  - "dist/preload/**/*"        # tsup preload
  - "package.json"
  - "!**/node_modules/**/{*.md,*.markdown,*.ts,*.map,test,tests,docs}"

mac:
  category: public.app-category.productivity
  target:
    - target: dmg
      arch: [universal]
    - target: zip              # for auto-update
      arch: [universal]
  hardenedRuntime: true
  gatekeeperAssess: false
  entitlements: build/entitlements.mac.plist
  entitlementsInherit: build/entitlements.mac.plist
  notarize: false              # we run notarize manually in afterSign
  extendInfo:
    NSMicrophoneUsageDescription: "YourApp records audio for…"
    NSCameraUsageDescription: "YourApp captures video for…"

dmg:
  sign: false                  # don't sign the DMG; only the .app inside
  contents:
    - x: 130
      y: 220
    - x: 410
      y: 220
      type: link
      path: /Applications

afterSign: scripts/notarize.cjs

publish:
  - provider: github
    owner: yourorg
    repo: yourapp
```

## Universal vs separate arches

`arch: [universal]` produces one binary that runs on Apple silicon and Intel via Apple's `lipo`. Pros: one DMG. Cons: ~2x size, slower build.

`arch: [arm64, x64]` produces two DMGs. Pros: smaller individual downloads. Cons: download page needs to detect arch or list both.

Most apps ship universal. Only split if download size matters more than UX simplicity.

## ASAR

`asar: true` (default) packs your app into a single archive. Faster file enumeration, slightly faster startup, mild obfuscation. Files in `asarUnpack` patterns extract to `app.asar.unpacked/` next to the archive.

What needs unpacking:
- All `.node` native binaries
- Anything you `child_process.fork()` or `utilityProcess.fork()` — they need a real file path
- Any file accessed via OS APIs that don't grok ASAR's virtual fs (image-rendering tools, ffmpeg with file paths)
- Sharp, better-sqlite3, ffmpeg, sentry-native — these have known ASAR issues

If a third-party module fails inside the packaged app but works in dev, asar-unpack is the first thing to try.

## ASAR integrity

Newer Electron supports ASAR integrity — your binary refuses to launch if the asar has been tampered with. Enabled via:

```yaml
electronUpdaterCompatibility: ">= 4.0.0"
asarIntegrity: true   # may already be default in your electron-builder version
```

Helps against trivial post-distribution modification. Doesn't replace code signing.

## File filtering

Default `files` glob includes everything. Trim to ship only what's needed:

- Don't include `src/**` — that's TypeScript source.
- Don't include `node_modules/.cache`, test files, docs.
- electron-builder auto-excludes `devDependencies` from `node_modules`. Make sure runtime deps are in `dependencies`.

Check what shipped: `dist/mac/YourApp.app/Contents/Resources/app.asar` — extract with `npx asar extract app.asar /tmp/check`.

## Universal-build native modules

Each native module needs both arches available. electron-builder runs `install-app-deps` per arch before universal'ing. On an M-series Mac, building x64 native modules requires:

```bash
arch -x86_64 zsh
brew install node           # x64 node, optional
bun install                 # bun under rosetta gets x64 modules
bunx electron-builder install-app-deps --arch x64
exit
bunx electron-builder install-app-deps --arch arm64
bun run dist
```

In CI, GitHub's `macos-latest` (arm64) handles both arches with Rosetta available. Add a setup step:

```yaml
- run: softwareupdate --install-rosetta --agree-to-license || true
```

## DMG layout

`dmg.contents` positions icons in the disk image window. The `link` entry creates the typical Applications shortcut. The default 540×380 window with the app on the left and `/Applications` on the right is what users expect — don't get creative.

Custom DMG background:

```yaml
dmg:
  background: build/dmg-background.png  # 540x380 px @1x, 1080x760 @2x
  window:
    width: 540
    height: 380
  iconSize: 96
```

## Icons

`build/icon.icns` for macOS. Generate from a 1024×1024 PNG with `iconutil` or `electron-icon-builder`:

```bash
bunx electron-icon-builder --input=icon.png --output=build/icons
mv build/icons/icons/mac/icon.icns build/icon.icns
```

`.icns` must include sizes from 16×16 up to 1024×1024 to look right at every Finder/Dock zoom level.

## Build commands

```bash
# package only (no installer)
bunx electron-builder --mac --dir

# DMG + ZIP, signed if certs present
bunx electron-builder --mac

# Universal
bunx electron-builder --mac --universal

# Publish to GitHub
GH_TOKEN=xxx bunx electron-builder --mac --publish always
```

`--dir` skips DMG creation — useful for fast iteration on packaging issues.

## Build hooks

`afterSign` script runs after code signing, before DMG packaging — the right hook for notarization. `afterPack` runs after files are copied but before signing. `beforeBuild` runs before any work — use to inject env-derived config.

## Output

```
dist/
├── builder-effective-config.yaml   # merged config; debug w/ this
├── YourApp-1.0.0-universal.dmg
├── YourApp-1.0.0-universal.zip
├── YourApp-1.0.0-universal-mac.zip.blockmap
├── latest-mac.yml                  # auto-update manifest
└── mac-universal/
    └── YourApp.app                 # the actual bundle
```

`builder-effective-config.yaml` is invaluable for debugging — shows the final config after defaults+merging.
