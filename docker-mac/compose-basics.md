# Compose basics

Compose describes a multi-container app in YAML, then brings it up/down as a unit. Single file, single command, reproducible across machines.

## File

Canonical name: `compose.yaml` (also accepts `docker-compose.yml`, `compose.yml`, etc.). The legacy `version:` field is obsolete — modern Compose ignores it. Omit it.

Skeleton:

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

## Top-level blocks

| Block | Purpose |
|---|---|
| `services` | Your containers — the core |
| `volumes` | Named volumes (shared across services if needed) |
| `networks` | Custom networks (one default is auto-created) |
| `secrets` | Secret files/values injected into services |
| `configs` | Non-secret config files injected into services |

## Core lifecycle commands

```bash
docker compose up -d              # create + start (detached)
docker compose up --build         # rebuild images first
docker compose up --watch         # with file watch (dev)
docker compose down               # stop + remove containers, networks
docker compose down -v            # also remove volumes  (destroys data!)
docker compose ps                 # status
docker compose logs -f [svc]      # tail
docker compose exec <svc> <cmd>   # shell into running service
docker compose restart [svc]
docker compose stop / start       # without recreating
docker compose pull               # pull referenced images
docker compose build [--no-cache] [svc]
docker compose config             # validated merged config (debug)
docker compose config --services  # list service names
```

Run single commands without bringing up a service:

```bash
docker compose run --rm api npm test        # one-shot, remove after
docker compose run --rm --service-ports api # with ports (run doesn't publish by default)
```

## Project name

Compose groups resources under a **project name** (default: current directory name). Container names, networks, and volumes are all prefixed with it.

```bash
docker compose -p myproject up -d
COMPOSE_PROJECT_NAME=myproject docker compose up -d
```

Override per-service container names with `container_name: foo` — loses scalability but fine for dev.

## Build context

```yaml
services:
  api:
    build: .                     # shorthand: context is current dir

  worker:
    build:
      context: ./worker
      dockerfile: Dockerfile.worker
      target: runtime
      args:
        NODE_VERSION: "22"
      platforms:
        - linux/amd64
      cache_from:
        - myorg/worker:cache
      tags:
        - myorg/worker:latest
```

## Image pull policy

```yaml
services:
  db:
    image: postgres:18
    pull_policy: always          # always | missing | never | build
```

## Commands and entrypoint

```yaml
services:
  api:
    image: myapp
    entrypoint: ["/entrypoint.sh"]
    command: ["node", "server.js"]
```

Use list form — string form goes through a shell (breaks signal handling, glob issues).

## Restart policies

```yaml
services:
  api:
    restart: unless-stopped      # no | on-failure | always | unless-stopped
```

For production: `unless-stopped` on user-facing services, `on-failure` on batch/one-shot jobs.

## Include other files

```yaml
include:
  - path: ./infra/compose.yaml
  - path: ./logging/compose.yaml
```

Merges referenced files. Good for splitting infra (db, cache, observability) from app services.

## Override files

`compose.override.yaml` is auto-loaded and merged over `compose.yaml`. Ideal for dev-only tweaks (ports, volumes, bind mounts) without touching the base.

```yaml
# compose.yaml — base
services:
  api:
    image: myorg/api:latest

# compose.override.yaml — dev
services:
  api:
    build: .
    volumes:
      - ./src:/app/src
    ports:
      - "3000:3000"
```

Explicit files (skip auto-loaded override):

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

Later files override earlier ones. Merge semantics: scalars overwrite, lists extend, maps merge.

## Extends (reuse)

```yaml
# common.yaml
services:
  _base:
    image: node:22-alpine
    working_dir: /app
    environment:
      NODE_ENV: development

# compose.yaml
services:
  api:
    extends:
      file: common.yaml
      service: _base
    command: ["node", "api.js"]

  worker:
    extends:
      file: common.yaml
      service: _base
    command: ["node", "worker.js"]
```

## Env vars in Compose

Two distinct things, often confused:

1. **Host-side interpolation** — `${VAR}` in the YAML, replaced when Compose parses. Pulled from the shell env and from `.env` next to `compose.yaml`.
2. **Container-side environment** — `environment:` block in a service, becomes env vars *inside* the container.

```yaml
services:
  api:
    image: myapp:${TAG:-latest}           # host-side: shell or .env
    environment:
      LOG_LEVEL: ${LOG_LEVEL}             # passed through to container
      DATABASE_URL: postgres://...         # literal
    env_file:
      - .env.api                           # read into container env
```

Details and precedence: [compose-secrets-env.md](compose-secrets-env.md).

## Scaling

```bash
docker compose up -d --scale worker=4
```

Requires no `container_name` and no hard-coded host ports (or a port range: `ports: ["3000-3010:3000"]`).

## Common mistakes

| Mistake | Fix |
|---|---|
| Using `version: "3.8"` | Delete it — modern Compose ignores it |
| `container_name` + `--scale N` | Remove `container_name` |
| Publishing DB ports to host by default | Drop `ports:` for the db service |
| Editing `compose.yaml` and nothing rebuilds | Use `--build` or set `pull_policy: build` |
| `docker compose down` lost my data | `down` without `-v` keeps volumes — `-v` is the destructive flag |
