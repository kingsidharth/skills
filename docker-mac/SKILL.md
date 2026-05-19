---
name: docker
description: Docker on Mac with OrbStack. Containers, Dockerfiles, images, Compose, BuildKit, volumes, networking. Use when working with Dockerfiles, compose.yaml, docker/docker compose/buildx commands, or containerizing any service.
---

# Docker

Mac-first via OrbStack. CLI is 100% Docker-compatible — commands here apply to Docker Desktop too unless marked `[OrbStack]`.

## Quick start

```bash
# Verify
docker version && docker compose version && docker buildx version

# Run
docker run -d -p 8080:80 nginx
docker ps

# Build
docker build -t app:dev .
docker compose up -d --build
```

## Routing

### Setup & platform
- **[orbstack-mac.md](references/orbstack-mac.md)** — OrbStack install, domains (`*.orb.local`), HTTPS, Rosetta for amd64, contexts, migrating from Docker Desktop
- **[cli-cheatsheet.md](references/cli-cheatsheet.md)** — flat command reference

### Containers
- **[containers-basics.md](references/containers-basics.md)** — `run`, `exec`, `logs`, `ps`, `stop`, `rm`, overriding CMD/ENTRYPOINT, resource limits
- **[ports-and-networking.md](references/ports-and-networking.md)** — `-p`, published vs exposed ports, user-defined bridge networks, DNS between containers

### Images
- **[images-and-tagging.md](references/images-and-tagging.md)** — `build`, tag structure, `push`, registries
- **[dockerfile-writing.md](references/dockerfile-writing.md)** — instructions, base image selection, `.dockerignore`, layer ordering
- **[multi-stage-builds.md](references/multi-stage-builds.md)** — build/runtime separation, `--target`, `COPY --from`

### Build performance
- **[build-cache.md](references/build-cache.md)** — invalidation rules, cache mounts, registry/inline/GHA backends
- **[buildkit-and-buildx.md](references/buildkit-and-buildx.md)** — drivers, `buildkitd.toml`, multi-platform, `buildx bake`
- **[build-exporters.md](references/build-exporters.md)** — `--output` local/tar for CI artifacts, OCI layouts

### Storage
- **[volumes-and-mounts.md](references/volumes-and-mounts.md)** — named volumes vs bind mounts vs tmpfs, `--mount` syntax, pre-seeding

### Compose
- **[compose-basics.md](references/compose-basics.md)** — `compose.yaml` shape, `up`/`down`/`logs`, build context, project naming
- **[compose-watch.md](references/compose-watch.md)** — live reload via `develop.watch` (sync / rebuild / sync+restart)
- **[compose-dependencies.md](references/compose-dependencies.md)** — `depends_on` with `condition: service_healthy` + healthchecks
- **[compose-profiles.md](references/compose-profiles.md)** — optional services via profiles
- **[compose-secrets-env.md](references/compose-secrets-env.md)** — `secrets`, env vars, precedence, `_FILE` convention
- **[compose-gpu.md](references/compose-gpu.md)** — `deploy.resources.reservations.devices` for NVIDIA
- **[compose-production.md](references/compose-production.md)** — overrides, restart policies, resource caps, logging

### Language patterns
- **[language-guides.md](references/language-guides.md)** — Bun, Python (uv), Rust

## Core principles

- **Build context is everything copied to the daemon.** Keep it small with `.dockerignore`.
- **Order Dockerfile from least to most frequently changing.** Deps before source.
- **One concern per container.** Compose for multi-service.
- **Named volumes for data, bind mounts for code in dev.**
- **On Mac, prefer OrbStack.** Faster, less RAM, free Rosetta for amd64 images.

## Version pinning

This skill targets: Docker Engine ≥28, Compose ≥2.30, Buildx ≥0.24, BuildKit ≥0.18. `compose.yaml` (no `version:` field) is current — legacy `docker-compose.yml` still works.
