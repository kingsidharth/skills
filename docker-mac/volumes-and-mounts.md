# Volumes & mounts

Three ways to persist or share data with a container: **named volumes**, **bind mounts**, **tmpfs mounts**. Pick by use case.

## Which to use

| Type | Lives in | Survives container delete | Editable from host | Use for |
|---|---|---|---|---|
| **Named volume** | Docker-managed area | yes | indirectly | databases, any persistent app data |
| **Bind mount** | Arbitrary host path | yes (it's on host) | yes | dev: mount source code; config files |
| **tmpfs** | RAM only | no | no | secrets/caches that must never hit disk |

Default: **named volume** unless you specifically need host-path access.

## Named volumes

```bash
docker volume create pgdata
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres:18

# if the volume doesn't exist, -v creates it
docker run -d -v pgdata:/var/lib/postgresql/data postgres:18
```

Compose:

```yaml
services:
  db:
    image: postgres:18
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Manage:

```bash
docker volume ls
docker volume inspect pgdata
docker volume rm pgdata            # must be detached
docker volume prune                # all unused
```

Volumes persist across `docker rm`, `docker-compose down`, even daemon restarts. `docker compose down -v` is the only common command that removes them.

On OrbStack, view/edit contents under `~/OrbStack/docker/volumes/<n>/_data/` directly from Mac.

## Bind mounts

Mount a host directory into the container:

```bash
docker run -v $(pwd)/src:/app/src myapp              # short form
docker run --mount type=bind,source=$(pwd)/src,target=/app/src myapp
```

Short form (`-v`) auto-creates missing host paths. Long form (`--mount`) errors if the host path doesn't exist — preferred for production/scripts because it surfaces typos.

Read-only:

```bash
docker run -v $(pwd)/config:/etc/myapp:ro myapp
docker run --mount type=bind,source=$(pwd)/config,target=/etc/myapp,readonly myapp
```

Compose (both forms):

```yaml
services:
  api:
    volumes:
      - ./src:/app/src                    # bind, read-write
      - ./config:/etc/myapp:ro            # bind, read-only
      - type: bind
        source: ./data
        target: /data
        read_only: true
```

### Dev pattern: source bind + named volume for deps

Bind source so edits reflect instantly; use a named volume for `node_modules`/`target`/etc. to avoid cross-platform conflicts and keep host clean:

```yaml
services:
  api:
    build: .
    volumes:
      - ./src:/app/src          # bind: your code
      - /app/node_modules       # anonymous volume: keep container's node_modules
```

This prevents your Mac's `node_modules` from clobbering the container's Linux-compiled native deps.

For large codebases, bind mounts on Mac can be slow (filesystem crossing the VM boundary). OrbStack's VirtioFS is fast; Docker Desktop can enable "Synchronized file shares" for similar perf. Or use **Compose Watch** (see [compose-watch.md](compose-watch.md)) — it's strictly faster than bind mounts for most dev workflows.

## tmpfs mounts

RAM-only storage — gone when the container stops. Use for sensitive scratch data.

```bash
docker run --tmpfs /tmp:size=100M,mode=1777 myapp
docker run --mount type=tmpfs,destination=/tmp,tmpfs-size=100000000 myapp
```

Compose:

```yaml
services:
  app:
    tmpfs:
      - /tmp
      - /run
```

## `VOLUME` in Dockerfile

```dockerfile
VOLUME /var/lib/postgresql/data
```

Declares a mount point that **must** be externally mounted. If the container runs without specifying a volume for that path, Docker creates an anonymous volume. Mostly used by official DB images. Rarely needed in your own Dockerfiles.

## Permissions

Linux-style uid/gid rules apply inside the container. Common trap: container process runs as uid 1001, bind-mounted host dir is owned by your Mac user (uid 501). Mac auto-reconciles via its FS bridge (mostly), but Linux CI doesn't.

Explicit fix:

```bash
docker run -u $(id -u):$(id -g) -v $(pwd):/app myapp
```

Or set ownership in Dockerfile and `COPY --chown`:

```dockerfile
RUN useradd -u 1001 app
COPY --chown=app:app . /app
USER app
```

## Pre-seeding volumes

You have a named volume that needs initial data (sample DB, config, etc.) the first time it's used.

**Option 1 — an init container:**

```yaml
services:
  seed:
    image: alpine
    volumes:
      - data:/data
    command: sh -c "[ -f /data/.seeded ] || (wget -O - https://example.com/seed.tar.gz | tar xz -C /data && touch /data/.seeded)"
  app:
    depends_on:
      seed:
        condition: service_completed_successfully
    volumes:
      - data:/data

volumes:
  data:
```

**Option 2 — bake seed files into the image, copy to volume on first run:**

```dockerfile
COPY seed/ /opt/seed/
```

Entrypoint script:

```bash
if [ ! -f /data/.seeded ]; then
  cp -a /opt/seed/. /data/
  touch /data/.seeded
fi
exec "$@"
```

**Option 3 — `docker run --volume` with an image-backed source** (newer Docker):

```yaml
volumes:
  seeded:
    driver: local
    driver_opts:
      type: image
      device: myorg/seed:v1
```

## Backup a volume

Volumes are just directories — use a temporary container:

```bash
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .
```

Restore:

```bash
docker run --rm \
  -v pgdata:/data \
  -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/pgdata-2026-04-18.tar.gz"
```

## `-v` vs `--mount` cheat

| Feature | `-v` / `--volume` | `--mount` |
|---|---|---|
| Auto-create missing host path | yes | no (errors — safer) |
| Syntax | terse, positional | verbose, key=value |
| Can specify all options | most | all |
| Recommended for | quick CLI, dev | scripts, production |

Same thing, different ergonomics.
