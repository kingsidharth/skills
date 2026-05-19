# Debugging

## Main Process Debugging

Attach Node.js inspector to the main process:

```bash
# Launch with inspector
electron --inspect=5858 .

# Break on first line
electron --inspect-brk=5858 .
```

Connect via `chrome://inspect` in Chrome, or attach from VS Code:

```json
// .vscode/launch.json
{
  "type": "node",
  "request": "launch",
  "name": "Debug Main Process",
  "runtimeExecutable": "${workspaceFolder}/node_modules/.bin/electron",
  "args": ["."],
  "runtimeArgs": ["--inspect-brk=5858"]
}
```

## Renderer Debugging

Standard Chrome DevTools — open via:
- `win.webContents.openDevTools()` programmatically
- Keyboard shortcut (Cmd+Option+I / Ctrl+Shift+I)
- Disable DevTools in production: don't call `openDevTools()` and optionally use CSP

## REPL

Interactive main process REPL:

```bash
# Start Electron with REPL access
electron --interactive .
```

Or use the `ELECTRON_RUN_AS_NODE` environment variable to run scripts in Electron's Node.js context without launching the app.

## DevTools Extensions

Load React DevTools, Vue DevTools, etc.:

```ts
import { session } from 'electron';
import installExtension, { REACT_DEVELOPER_TOOLS } from 'electron-devtools-installer';

app.whenReady().then(async () => {
  await installExtension(REACT_DEVELOPER_TOOLS);
});
```

Or manually load from path:

```ts
await session.defaultSession.loadExtension(
  path.join(os.homedir(), '.config/google-chrome/Default/Extensions/...')
);
```

## Debugging Native Addons (macOS)

When native Node addons crash (segfaults in `better-sqlite3`, `nodegit`, etc.):

1. **Build in debug mode:**
   ```bash
   npx @electron/rebuild --debug -f -w better-sqlite3
   ```

2. **Create Xcode project:**
   ```bash
   cd node_modules/better-sqlite3
   node-gyp configure --debug --target=33.0.0 --arch=arm64 \
     --dist-url=https://electronjs.org/headers -- -f xcode
   node-gyp rebuild --debug --target=33.0.0 --arch=arm64 \
     --dist-url=https://electronjs.org/headers
   ```

3. **Open in Xcode:** `open build/binding.xcodeproj`

4. **Configure scheme:** set Electron.app as executable, pass your app's main.js as argument

5. **Run Without Building** (Xcode would rebuild for system Node, not Electron's Node)

6. Set breakpoints, inspect variables, catch segfaults with full stack context

## Automated Testing

### Spectron Successor: Playwright / WebDriverIO

Electron supports WebDriver protocol. Use Playwright or WebDriverIO for E2E tests:

```ts
// Using Playwright
import { _electron as electron } from 'playwright';

const app = await electron.launch({ args: ['.'] });
const window = await app.firstWindow();
await window.click('button#submit');
expect(await window.title()).toBe('My App');
await app.close();
```

### Unit Testing Main Process

Test IPC handlers in isolation by mocking Electron APIs:

```ts
// Jest example
jest.mock('electron', () => ({
  ipcMain: { handle: jest.fn() },
  app: { getPath: jest.fn(() => '/tmp') },
}));
```

### Testing Preload Scripts

Preload runs in a special context. Test the exported API surface, not the preload internals:

```ts
// Test the shape of exposed API
const api = require('./preload');
expect(api).toHaveProperty('getVersion');
expect(typeof api.getVersion).toBe('function');
```

## Common Debug Techniques

- **White screen on launch:** check DevTools console for errors, verify preload path is correct
- **IPC not working:** ensure channel names match exactly, check `contextIsolation` is enabled
- **Native module crashes:** rebuild with `@electron/rebuild` matching your Electron version
- **Memory leaks:** take heap snapshots over time, diff to find growing objects
- **Slow startup:** use `--trace-warnings` and `contentTracing` to identify bottlenecks

> **Ref:** [Debugging Main Process](https://www.electronjs.org/docs/latest/tutorial/debugging-main-process) · [Application Debugging](https://www.electronjs.org/docs/latest/tutorial/application-debugging) · [Automated Testing](https://www.electronjs.org/docs/latest/tutorial/automated-testing) · [DevTools Extension](https://www.electronjs.org/docs/latest/tutorial/devtools-extension) · [REPL](https://www.electronjs.org/docs/latest/tutorial/repl) · [Debugging Native Addons](https://felixrieseberg.com/debugging-native-node-js-addons-with-electron/)
