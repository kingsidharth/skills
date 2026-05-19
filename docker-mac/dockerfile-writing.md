# Writing Dockerfiles

Goals, in priority order: small, secure, fast to rebuild, reproducible.

## Instruction reference (compact)

| Instruction | Purpose |
|---|---|
| `FROM` | Base image. First instruction (after optional `ARG`) |
| `ARG` | Build-time variable |
| `ENV` | Runtime env var (persisted in image) |
| `WORKDIR` | `cd` into path (creates if missing) |
| `COPY` | Copy from build context into image |
| `ADD` | Like COPY but supports URLs and tar extraction — prefer COPY |
| `RUN` | Execute command in a new layer |
| `USER` | Switch user for subsequent instructions + runtime |
| `EXPOSE` | Document intended listening ports (doesn't publish) |
| `VOLUME` | Mark path as external volume mount point |
| `CMD` | Default command (overridable at run time) |
| `ENTRYPOINT` | Fixed executable prefix (CMD becomes args) |
| `HEALTHCHECK` | Container health probe |
| `LABEL` | Metadata (OCI conventions: `org.opencontainers.image.*`) |
| `STOPSIGNAL` | Custom shutdown signal |

## Base image selection

| Pick | When |
|---|---|
| `<lang>:<version>-slim` | sensible default — Debian-based, smaller than full |
| `<lang>:<version>-alpine` | smallest, but musl libc can break native deps |
| `distroless/<lang>` (Google) | production runtime only — no shell, no package manager |
| `scratch` | pure static binaries (Go, Rust with static linking) |
| Docker Hardened Images | enterprise: CVEs patched, SBOMs, FIPS variants |

Pin the version: `node:22.9-alpine`, not `node:alpine`. Rebuilds stay reproducible and CVE surface is knowable.

## Layer ordering (cache-friendly)

Dependencies change rarely; source changes constantly. Put deps first so source edits don't bust the dep install layer.

Generic pattern:

1. `FROM`
2. `WORKDIR`
3. Install OS deps (rarely changes)
4. Copy dep manifest (`package.json`, `requirements.txt`, `Cargo.toml`, `go.mod`)
5. Install language deps
6. Copy source
7. Build (if needed)
8. `USER`, `EXPOSE`, `CMD`/`ENTRYPOINT`

Details and cache invalidation rules: [build-cache.md](build-cache.md).

## `.dockerignore`

Ships alongside the Dockerfile; excludes patterns from the build context. Every file listed here is a file *not* uploaded, *not* cache-invalidating, *not* leaking into your image.

Minimum for any repo:

```
.git
.gitignore
.env
.env.*
node_modules
__pycache__
*.pyc
.venv
target/
dist/
build/
.DS_Store
*.md
Dockerfile*
docker-compose*.yml
.github/
.vscode/
.idea/
```

Language-specific: add `node_modules`, `target/`, `.venv`, etc. — whatever is rebuilt inside the image anyway.

## `COPY` specificity

Copying the whole context invalidates cache on any file change. Copy only what the next step actually needs:

```dockerfile
# Bad — any source change busts dep install
COPY . .
RUN npm install

# Good — deps layer only invalidates when manifest changes
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
```

## `RUN` — chain + clean in one layer

Each `RUN` creates a layer. Delete temp files in the *same* `RUN` or they stick around in the layer history.

```dockerfile
# Bad — apt cache is in a layer below, still ships
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# Good — one layer, no cache left behind
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*
```

## Run as non-root

Default `USER` is root. Containers run as root unless told otherwise — a real security issue.

```dockerfile
RUN useradd -m -u 1001 app
USER app
```

For `COPY` to land files owned by `app`:

```dockerfile
COPY --chown=app:app . /app
```

Compose Watch's `sync` action requires the container `USER` can write to the target path.

## `CMD` vs `ENTRYPOINT`

| Pattern | Use |
|---|---|
| `CMD ["node", "server.js"]` only | Simple apps — overridable |
| `ENTRYPOINT ["node"]` + `CMD ["server.js"]` | `docker run img other.js` still runs node |
| `ENTRYPOINT ["./docker-entrypoint.sh"]` + `CMD [...]` | Init script that eventually `exec`s the command |

Always use the **exec form** (JSON array), not shell form:

```dockerfile
CMD ["node", "server.js"]          # PID 1 is node, signals work
CMD node server.js                 # PID 1 is sh, signals don't propagate
```

## HEALTHCHECK

Image-level health probe. Compose consumes this for `depends_on: condition: service_healthy`.

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

## Labels (OCI)

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.revision="${GIT_SHA}"
LABEL org.opencontainers.image.version="1.2.3"
```

GitHub Container Registry uses `image.source` to link packages to the repo.

## Build checks

Modern BuildKit runs lint-style checks on your Dockerfile. See warnings in build output; override severity in `# check=error=true` directive at file top.

## Skeleton

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine AS base
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app

FROM base AS deps
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM base AS build
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM base AS runtime
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

Multi-stage details: [multi-stage-builds.md](multi-stage-builds.md).
