---
name: apple-releases
description: Build and release iOS, iPadOS, macOS, watchOS, tvOS, or visionOS apps. Use for Xcode signing, archives, TestFlight, App Store Connect beta review, public links, and notarization.
---

# Apple releases

Verify the bundle ID, version, build number, signing, environment, and archive before upload. Confirm App Store Connect processing and beta review state before sharing a public TestFlight link or enforcing an upgrade.

For Noteasy on this Mac, run `~/Sites/apple_setup/scripts/app-store-connect status` or `groups` for read-only TestFlight checks. API keys and local setup belong in `~/Sites/apple_setup/`; never commit them. Use `xcodebuild` and Apple's upload tools for archives; use the App Store Connect API for beta status and groups.
