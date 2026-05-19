# IPC Security & Safe Storage

## IPC as Security Boundary

Treat every IPC message from renderer like an untrusted HTTP request. Validate inputs, check permissions, sanitize data.

```ts
// ❌ NEVER — exposes entire IPC system
contextBridge.exposeInMainWorld('electron', { ipcRenderer });

// ✅ Expose specific, validated APIs only
contextBridge.exposeInMainWorld('api', {
  saveFile: (name: string, content: string) => {
    if (typeof name !== 'string' || name.includes('..')) throw new Error('Invalid');
    return ipcRenderer.invoke('save-file', name, content);
  },
});
```

In `ipcMain.handle`:
- Validate argument types and ranges
- Reject path traversal (`..`, absolute paths)
- Use an allowlist of channel names
- Don't pass callbacks directly through IPC (leaks `IpcRendererEvent` with `.sender`)

## Navigation & Window Hardening

Lock navigation to known origins:

```ts
win.webContents.on('will-navigate', (event, url) => {
  if (!allowedOrigins.includes(new URL(url).origin)) event.preventDefault();
});

win.webContents.setWindowOpenHandler(({ url }) => {
  if (isSafeUrl(url)) shell.openExternal(url);
  return { action: 'deny' };
});
```

Disable `webviewTag` unless explicitly needed. Audit `will-attach-webview` events.

## Permission Handling

```ts
win.webContents.session.setPermissionRequestHandler(
  (_wc, permission, callback) => {
    const allowed = ['clipboard-read', 'notifications'];
    callback(allowed.includes(permission));
  }
);
```

## Safe Storage

`safeStorage` encrypts strings using the OS keychain (macOS Keychain, Windows DPAPI, Linux libsecret/kwallet):

```ts
import { safeStorage } from 'electron';

if (safeStorage.isEncryptionAvailable()) {
  const encrypted = safeStorage.encryptString('api-key-value');
  fs.writeFileSync(keyPath, encrypted);
  const decrypted = safeStorage.decryptString(fs.readFileSync(keyPath));
}
```

Never store secrets in plaintext JSON files or `electron-store` without encryption.

## Avoid file:// Protocol

Register a custom protocol with appropriate privileges:

```ts
protocol.registerSchemesAsPrivileged([{
  scheme: 'app',
  privileges: { secure: true, standard: true, supportFetchAPI: true },
}]);
```

## Auditing Tools

- **Electronegativity**: static analysis for Electron security anti-patterns. Checks `NODE_INTEGRATION_JS_CHECK`, `CONTEXT_ISOLATION_JS_CHECK`, `CSP_GLOBAL_CHECK`
- Run `npm audit` regularly for dependency vulnerabilities
- Enable ASAR integrity verification

## Security Checklist

- [ ] `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`
- [ ] CSP defined and strict
- [ ] IPC handlers validate all inputs
- [ ] No raw `ipcRenderer` exposed to renderer
- [ ] Navigation restricted to known origins
- [ ] Permission handler configured
- [ ] `safeStorage` for secrets
- [ ] Dependencies audited
- [ ] ASAR integrity verification enabled
- [ ] Code signing configured

> **Ref:** [Security Tutorial](https://www.electronjs.org/docs/latest/tutorial/security) · [safeStorage](https://www.electronjs.org/docs/latest/api/safe-storage)
