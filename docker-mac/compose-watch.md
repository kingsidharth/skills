# Compose Watch

Live-reload for containerized dev, without the downsides of bind mounts. Ships in Compose ≥ 2.22.

Bind mounts cross the host↔VM FS boundary, cause permission drift, and sync `node_modules` you don't want synced. Watch copies individual files on change, ignores what you tell it to ignore, and triggers rebuilds when needed.

## Three actions

| Action | What happens on change |
|---|---|
| `sync` | Copy changed file into container. Best for HMR frameworks |
| `rebuild` | Rebuild the image, recreate the container. For `package.json`, `requirements.txt`, Dockerfile |
| `sync+restart` | Copy file, restart container process. For config files (`nginx.conf`, env changes) |

## Minimal example

```yaml
services:
  web:
    build: .
    command: npm run dev
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          ignore:
            - node_modules/

        - action: rebuild
          path: package.json
```

Run:

```bash
docker compose up --watch
# or, to separate logs:
docker compose watch
```

## Real-world per-language templates

### Node.js / Vite

```yaml
services:
  web:
    build: .
    command: npm run dev -- --host 0.0.0.0
    ports:
      - "5173:5173"
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          ignore: [node_modules/]
        - action: sync
          path: ./public
          target: /app/public
        - action: rebuild
          path: package.json
        - action: rebuild
          path: vite.config.ts
```

### Python / uvicorn with reload

```yaml
services:
  api:
    build: .
    command: uvicorn app:app --host 0.0.0.0 --port 8000 --reload
    develop:
      watch:
        - action: sync
          path: ./app
          target: /app/app
          ignore: [__pycache__/, "*.pyc"]
        - action: rebuild
          path: requirements.txt
        - action: rebuild
          path: pyproject.toml
```

### Go

```yaml
services:
  api:
    build:
      context: .
      target: dev     # stage with `air` or `reflex` for live rebuild
    develop:
      watch:
        - action: sync+restart
          path: .
          target: /src
          ignore: [vendor/, tmp/, "*.test"]
        - action: rebuild
          path: go.mod
```

### Nginx proxy config

```yaml
services:
  nginx:
    image: nginx:alpine
    volumes:
      - ./public:/usr/share/nginx/html:ro
    develop:
      watch:
        - action: sync+restart
          path: ./nginx.conf
          target: /etc/nginx/conf.d/default.conf
```

## Rules

- Paths are **relative to the project directory** (except `ignore`, which is relative to `path`).
- Directories watched recursively. No glob patterns in `path`.
- `.dockerignore` rules apply automatically; `.git` and IDE backup files are ignored automatically.
- Image must have `stat`, `mkdir`, `rmdir` (Alpine fine, distroless won't work).
- Container `USER` must be able to write to the target path. Use `COPY --chown` so initial files are owned by that user.

## `initial_sync`

Ensures files under `path` are in sync *before* the watch session starts. Useful when the container's image is behind your working tree.

```yaml
watch:
  - action: sync
    path: ./src
    target: /app/src
    initial_sync: true
```

## When Watch is the wrong tool

- **Pre-built images** (`image:` only, no `build:`) — Watch requires a `build:` context.
- **Code that requires a full rebuild to run** (not most interpreted languages; compiled languages without live-reload tooling) — just use `rebuild` action for everything, or skip Watch.
- **Mass file changes** (checkout a different branch) — many individual file syncs can be slower than a rebuild.

## Bind mount vs Watch — when each wins

| Situation | Pick |
|---|---|
| Your framework has HMR and watches the FS | Watch (`sync`) |
| You need two-way sync (container writes files you need on host) | Bind mount |
| You're on Mac, codebase is large, perf matters | Watch |
| You want zero config and it's a tiny project | Bind mount |
| You need `node_modules` to stay Linux-native | Watch (the anonymous-volume trick) or both |
