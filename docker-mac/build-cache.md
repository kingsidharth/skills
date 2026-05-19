# Build cache

Docker caches each instruction's output as a layer. A rebuild reuses cached layers until something invalidates one — and once a layer is invalidated, **every layer after it is too**.

## What invalidates the cache

| Instruction | Invalidates when |
|---|---|
| `FROM` | base image digest changes |
| `RUN` | command string changes character-for-character |
| `COPY` / `ADD` | any source file's content, permissions, or mtime changes |
| `ARG` / `ENV` used by a later `RUN` | value changes |
| `WORKDIR`, `USER`, `EXPOSE`, `LABEL`, etc. | argument changes |

`.dockerignore` matters here: a file the daemon never saw can't invalidate cache.

## The golden rule: order by change frequency

Each layer must be cached independently of layers *below* changing. So: stable things first, volatile things last.

```dockerfile
# Layer 1: base       (changes: monthly)
FROM python:3.12-slim

# Layer 2: OS deps    (changes: rarely)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential && rm -rf /var/lib/apt/lists/*

# Layer 3: app deps   (changes: when you add packages)
COPY requirements.txt .
RUN pip install -r requirements.txt

# Layer 4: source     (changes: every commit)
COPY . .
```

Edit a `.py` file → only layer 4 rebuilds. Edit `requirements.txt` → 3 and 4 rebuild. Edit the Dockerfile's apt line → everything rebuilds.

## Cache mounts (BuildKit)

Keep package-manager caches between builds *without* baking them into the image. Requires BuildKit (default in modern Docker).

```dockerfile
# syntax=docker/dockerfile:1

RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y curl

RUN --mount=type=cache,target=/root/.npm \
    npm ci

RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release
```

Cache persists across builds, across Dockerfile changes, across anything. Survives until pruned with `docker builder prune`.

## Bind mounts at build time

Avoid copying huge files into a layer you're going to delete anyway:

```dockerfile
RUN --mount=type=bind,source=package-lock.json,target=package-lock.json \
    --mount=type=bind,source=package.json,target=package.json \
    --mount=type=cache,target=/root/.npm \
    npm ci
```

## Secrets at build time

Don't `COPY` a secret, don't set it via `ARG` (ends up in image history). Use BuildKit's `--mount=type=secret`:

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc -t app .
# or with Compose:
#   build.secrets: [npmrc]   referencing a top-level secret
```

## Cache backends (CI)

Local Docker cache dies when the CI runner dies. Export/import from remote storage:

| Backend | Flag example |
|---|---|
| Inline (embed in pushed image) | `--cache-to type=inline` |
| Registry (separate cache image) | `--cache-to type=registry,ref=myorg/app:cache --cache-from type=registry,ref=myorg/app:cache` |
| GitHub Actions | `--cache-to type=gha,mode=max --cache-from type=gha` |
| Local dir | `--cache-to type=local,dest=/tmp/cache --cache-from type=local,src=/tmp/cache` |
| S3 | `--cache-to type=s3,region=...,bucket=... --cache-from type=s3,...` |

`mode=max` exports cache for *all* layers including intermediate ones. `mode=min` only exports the final stage — smaller but less useful for multi-stage builds.

## Diagnosing cache misses

When a rebuild is slow, check build output — BuildKit prints `CACHED` for cache hits. First non-CACHED line is the invalidation point.

```bash
docker buildx build --progress=plain -t app .   # full output, not collapsed
```

Also useful:

```bash
docker buildx du              # cache size breakdown
docker buildx prune           # clear BuildKit cache
docker buildx prune --keep-storage 10gb   # cap cache size
```

## Quick diagnostics table

| Symptom | Likely cause |
|---|---|
| Every build reinstalls deps | `COPY . .` before dep install — reorder |
| Build context transfer is slow | missing `.dockerignore`, sending `node_modules`/`.git` |
| `ARG FOO` change doesn't bust cache | `ARG` declared before `FROM` (global) vs after (stage-local) |
| Cache works locally, not in CI | no `--cache-from`/`--cache-to` configured for CI backend |
| Layer size bigger than expected | temp files not cleaned in same `RUN` |
