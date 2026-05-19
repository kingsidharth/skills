# Storage

Pick by data shape:

| Shape | Tool |
|---|---|
| Tiny KV (settings, preferences, last-window-state) | `electron-store` or hand-rolled JSON |
| Structured app data, queryable | `better-sqlite3` |
| Cache, expiring data | LevelDB (`level`) or `lru-cache` in memory + JSON snapshot |
| Large binary blobs | files in `userData/blobs/<hash>` |
| Cloud-syncable docs | CRDT (Automerge / Yjs) on top of files |

## electron-store

```ts
import Store from 'electron-store'

const store = new Store<{
  windowBounds: { x: number; y: number; width: number; height: number }
  theme: 'light' | 'dark' | 'system'
}>({
  defaults: {
    windowBounds: { x: 0, y: 0, width: 1200, height: 800 },
    theme: 'system',
  },
})

store.get('theme')
store.set('theme', 'dark')
store.onDidChange('theme', (newVal) => { /* react */ })
```

Whole file is read and written on each set, so don't put 100MB of data here. Settings, preferences, last-known UI state — that's the use case. File lives at `userData/config.json`.

ESM-only since v9. If your main process is CJS, pin to v8 or migrate to ESM.

## better-sqlite3 — recommended SQL

```ts
import Database from 'better-sqlite3'

const db = new Database(path.join(app.getPath('userData'), 'data.sqlite'))
db.pragma('journal_mode = WAL')
db.pragma('synchronous = NORMAL')

db.exec(`
  CREATE TABLE IF NOT EXISTS notes (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    body TEXT,
    updated_at INTEGER NOT NULL
  );
  CREATE INDEX IF NOT EXISTS idx_notes_updated ON notes(updated_at);
`)

const insertNote = db.prepare('INSERT INTO notes (id, title, body, updated_at) VALUES (?, ?, ?, ?)')
const getNote = db.prepare('SELECT * FROM notes WHERE id = ?')
```

`better-sqlite3` is synchronous, single-threaded, and fast. For small queries (< 50ms) called from main, sync is fine. For long queries, run in a `UtilityProcess` to avoid blocking IPC handlers.

It's a native module — runs through `@electron/rebuild`. See `native-modules.md`.

WAL mode improves concurrent read/write — strongly recommended.

## node-sqlite3 vs better-sqlite3

Use `better-sqlite3`. The async `sqlite3` package is slower (callback-based), the API is awkward, and its native binding is less reliable across Electron versions. `better-sqlite3` is the de facto standard.

## Migrations

Track schema version in a `pragma user_version`:

```ts
const v = db.pragma('user_version', { simple: true }) as number
const migrations: Record<number, () => void> = {
  1: () => db.exec('CREATE TABLE notes (id TEXT PRIMARY KEY)'),
  2: () => db.exec('ALTER TABLE notes ADD COLUMN body TEXT'),
}
for (let i = v + 1; i in migrations; i++) {
  db.transaction(() => {
    migrations[i]()
    db.pragma(`user_version = ${i}`)
  })()
}
```

Always wrap migrations in transactions. Test each migration end-to-end before shipping — there's no rollback for users.

## userData layout

```
~/Library/Application Support/MyApp/
├── config.json            # electron-store
├── data.sqlite            # main DB
├── data.sqlite-wal        # WAL
├── data.sqlite-shm
├── blobs/                 # CAS-style file storage
│   ├── ab/
│   │   └── abc123…
└── logs/                  # if app.getPath('logs') points here
```

Don't pollute the user's `~` or `~/Documents`. Keep everything inside `userData` unless the user explicitly chose another location.

## Encryption at rest

For sensitive data, use `safeStorage`:

```ts
import { safeStorage } from 'electron'

if (safeStorage.isEncryptionAvailable()) {
  const cipher = safeStorage.encryptString('secret')      // Buffer
  fs.writeFileSync(path, cipher)
  const plain = safeStorage.decryptString(fs.readFileSync(path))
}
```

On macOS, `safeStorage` is backed by Keychain — encryption keys never leave the OS. The app must be signed for it to work properly across launches; in dev (unsigned), encryption is best-effort and may not survive reboots.

For database-level encryption, use SQLCipher (paid build of better-sqlite3) — overkill for most apps.

## Renderer access pattern

Don't open the database from the renderer. Open in main, expose CRUD via IPC:

```ts
contextBridge.exposeInMainWorld('db', {
  notes: {
    list: () => ipcRenderer.invoke('notes:list'),
    save: (n: Note) => ipcRenderer.invoke('notes:save', n),
    delete: (id: string) => ipcRenderer.invoke('notes:delete', id),
  },
})
```

For real-time updates across windows, use `webContents.send` to broadcast change events.

## State shape across windows

Single source of truth in main; renderers subscribe. For complex UIs, run a small store in main (Zustand, custom event emitter) and replicate to renderers via IPC. Don't sync via shared file — multiple writers race.
