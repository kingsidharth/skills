# Security Reference

## Table of Contents
- Security Model Overview
- Capabilities
- Permissions
- Command Scopes
- Content Security Policy (CSP)
- IPC Security (Isolation Pattern)
- Best Practices Checklist

---

## Security Model Overview

Tauri v2's security model is built on the Principle of Least Privilege. Every API access from the frontend must be explicitly granted via capabilities and permissions. The system was independently audited by Radically Open Security.

**What the security model protects against:**
- Minimizes impact of frontend compromise
- Prevents/reduces accidental exposure of system interfaces and data
- Prevents/reduces privilege escalation from frontend to backend

**What it does NOT protect against:**
- Malicious or insecure Rust code you write
- Too-permissive scope configuration
- Incorrect scope checks in command implementations
- Intentional bypasses from Rust code
- 0-day or unpatched WebView vulnerabilities
- Supply chain attacks or compromised developer systems

---

## Capabilities

Capabilities are the top-level security boundary. They map sets of permissions to specific windows/webviews. Defined as JSON or TOML files in `src-tauri/capabilities/`.

### Basic Capability

```json
// src-tauri/capabilities/default.json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "main-capability",
  "description": "Capability for the main window",
  "windows": ["main"],
  "permissions": [
    "core:path:default",
    "core:event:default",
    "core:window:default",
    "core:app:default",
    "core:resources:default",
    "core:menu:default",
    "core:tray:default",
    "core:window:allow-set-title"
  ]
}
```

### Platform-Specific Capabilities

Use the `platforms` array to restrict capabilities to specific targets:

```json
// src-tauri/capabilities/desktop.json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "desktop-capability",
  "windows": ["main"],
  "platforms": ["linux", "macOS", "windows"],
  "permissions": [
    "global-shortcut:allow-register",
    "shell:allow-execute",
    "updater:default"
  ]
}
```

```json
// src-tauri/capabilities/mobile.json
{
  "$schema": "../gen/schemas/mobile-schema.json",
  "identifier": "mobile-capability",
  "windows": ["main"],
  "platforms": ["iOS", "android"],
  "permissions": [
    "nfc:allow-scan",
    "biometric:allow-authenticate",
    "barcode-scanner:allow-scan",
    "haptics:default"
  ]
}
```

### Auto-Enable vs. Explicit Enable

By default, all capability files in `src-tauri/capabilities/` are automatically enabled. To manually control which capabilities are active:

```json
// tauri.conf.json
{
  "app": {
    "security": {
      "capabilities": ["main-capability", "desktop-capability"]
    }
  }
}
```

Once you explicitly list capabilities in `tauri.conf.json`, only those are used.

### Inline Capabilities

You can also define capabilities inline in `tauri.conf.json`:

```json
{
  "app": {
    "security": {
      "capabilities": [
        {
          "identifier": "inline-cap",
          "windows": ["*"],
          "permissions": ["fs:default", "store:default"]
        },
        "my-file-capability"
      ]
    }
  }
}
```

### Multi-Window Security

Windows referenced in multiple capabilities merge all permissions. Keep high-privilege and low-privilege windows in separate capabilities.

Security boundaries are based on window **labels** (not titles). Restrict window creation functionality to higher-privilege windows only.

---

## Remote API Access

By default, Tauri APIs are only accessible to bundled code. To allow remote URLs to call certain commands:

```json
// src-tauri/capabilities/remote.json
{
  "$schema": "../gen/schemas/remote-schema.json",
  "identifier": "remote-capability",
  "windows": ["main"],
  "remote": {
    "urls": ["https://*.yourdomain.com"]
  },
  "platforms": ["iOS", "android"],
  "permissions": [
    "nfc:allow-scan",
    "barcode-scanner:allow-scan"
  ]
}
```

**Caution:** On Linux and Android, Tauri cannot distinguish requests from embedded `<iframe>` elements from the window itself. Use remote access carefully.

---

## Permissions

Permissions are granular allow/deny rules for individual plugin commands. Each plugin defines its own set of permissions.

### Permission Format

```
<plugin>:default              # Enable the plugin's default permission set
<plugin>:allow-<command>      # Allow a specific command
<plugin>:deny-<command>       # Deny a specific command
```

### Example: HTTP Plugin Permissions

```json
{
  "permissions": [
    {
      "identifier": "http:default",
      "allow": [
        { "url": "https://api.example.com/*" },
        { "url": "https://cdn.example.com/*" }
      ],
      "deny": [
        { "url": "https://internal.example.com/*" }
      ]
    }
  ]
}
```

### Example: File System Permissions with Scopes

```json
{
  "permissions": [
    "fs:default",
    {
      "identifier": "fs:allow-read-text-file",
      "allow": [
        { "path": "$APPDATA/**" },
        { "path": "$RESOURCE/**" }
      ]
    },
    {
      "identifier": "fs:allow-write-text-file",
      "allow": [
        { "path": "$APPDATA/**" }
      ]
    }
  ]
}
```

### Core Permissions

Core Tauri functionality also requires permissions:

```
core:path:default
core:event:default
core:window:default
core:app:default
core:resources:default
core:menu:default
core:tray:default
core:window:allow-set-title
core:window:allow-close
core:window:allow-minimize
```

### Restricting Custom Commands

By default, all registered commands are accessible to all windows. To restrict this:

```rust
// src-tauri/build.rs
fn main() {
    tauri_build::try_build(
        tauri_build::Attributes::new()
            .app_manifest(
                tauri_build::AppManifest::new()
                    .commands(&["allowed_command_1", "allowed_command_2"])
            ),
    )
    .unwrap();
}
```

---

## Command Scopes

Scopes provide fine-grained constraints on what a permitted command can actually access (paths, URLs, etc.).

Full reference: https://v2.tauri.app/security/scope/

---

## Content Security Policy (CSP)

Configure CSP headers to restrict what resources the WebView can load:

Full reference: https://v2.tauri.app/security/csp/

---

## IPC Security (Isolation Pattern)

The Isolation pattern adds a secure JavaScript layer between the frontend and Tauri Core:

1. A sandboxed `<iframe>` runs your isolation script
2. All IPC calls from the frontend pass through this iframe
3. Messages are encrypted with SubtleCrypto (new keys per app start)
4. Your isolation script can intercept, verify, and modify messages

### Setup

```json
// tauri.conf.json
{
  "app": {
    "security": {
      "pattern": {
        "use": "isolation",
        "options": {
          "dir": "../dist-isolation"
        }
      }
    }
  }
}
```

### Isolation Script

```html
<!-- dist-isolation/index.html -->
<!DOCTYPE html>
<html>
<head><title>Isolation</title></head>
<body>
  <script src="index.js"></script>
</body>
</html>
```

```javascript
// dist-isolation/index.js
window.__TAURI_ISOLATION_HOOK__ = (payload) => {
  // Validate or modify the IPC payload
  console.log('IPC intercepted:', payload);
  return payload; // Return modified or original payload
};
```

Use the Isolation pattern when your frontend loads untrusted third-party dependencies.

---

## Best Practices Checklist

- [ ] Grant minimum permissions — only what each window needs
- [ ] Never expose secrets in the frontend — keep them in Rust
- [ ] Sanitize all user input in both frontend and Rust commands
- [ ] Use platform-specific capabilities to avoid granting mobile permissions on desktop
- [ ] Use the Isolation pattern when loading third-party frontend code
- [ ] Restrict window creation to privileged windows (security boundaries use labels)
- [ ] Defer business logic and sensitive operations to the Rust Core process
- [ ] Review plugin permissions — use specific `allow-*` instead of blanket `default` when possible
- [ ] Use scoped file system access — don't grant `$HOME/**` unless truly needed
- [ ] Configure CSP headers to restrict script/style sources
- [ ] Keep Tauri and WebView dependencies updated
- [ ] Use `tauri_build::AppManifest::commands()` to restrict which custom commands are accessible
- [ ] For remote API access, limit URLs to specific domains with wildcard patterns
- [ ] On Linux/Android, be extra cautious with remote capabilities (iframe distinction issue)

Full security documentation: https://v2.tauri.app/security/
