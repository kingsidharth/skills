# Containers — running, inspecting, controlling

A container is an isolated process with its own filesystem, network namespace, and resource limits. Ephemeral by default — delete a container, lose its changes (unless persisted via volumes).

## Lifecycle

```
image ──► run ──► running ──► stop ──► stopped ──► rm ──► gone
                    │                     │
                    └── exec/logs ────────┘
```

## Run

```bash
docker run [opts] <image> [cmd] [args]
```

Most common shape for services:

```bash
docker run -d --name api -p 3000:3000 --restart unless-stopped myapp:latest
```

For one-off work (interactive shell):

```bash
docker run --rm -it alpine sh
```

Key flags: see [cli-cheatsheet.md](cli-cheatsheet.md).

## Inspect running state

```bash
docker ps                          # running only
docker ps -a                       # all, including exited
docker logs -f --tail 100 <n>   # follow logs
docker stats                       # live resource usage
docker inspect <n>              # everything as JSON
docker top <n>                  # processes inside container
docker port <n>                 # published ports
```

Grab one field from inspect:

```bash
docker inspect -f '{{.State.Status}}' <n>
docker inspect -f '{{.NetworkSettings.IPAddress}}' <n>
```

## Execute inside a running container

```bash
docker exec -it <n> sh          # or bash if image has it
docker exec <n> env             # one-shot command
docker exec -u root <n> sh      # as root
```

OrbStack: use the built-in Debug Shell (available in the GUI) — works even on distroless containers because it injects utilities without needing them in the image.

## Overriding image defaults

Image defaults come from the Dockerfile's `CMD`, `ENTRYPOINT`, `ENV`, `EXPOSE`, `USER`, `WORKDIR`. Override at run time:

| Override | Flag | Example |
|---|---|---|
| Command | positional args after image | `docker run nginx nginx -v` |
| Entrypoint | `--entrypoint` | `docker run --entrypoint sh alpine` |
| Env var | `-e` or `--env-file` | `-e LOG_LEVEL=debug` |
| Working dir | `-w` | `-w /app` |
| User | `-u` | `-u 1000:1000` |
| Port | `-p` | `-p 8080:80` |

`.env` file is the clean way to set many variables:

```bash
docker run --env-file .env myapp
```

## Resource limits

```bash
docker run --memory 512m --cpus 0.5 myapp
```

No limit = container can use all host resources. On shared dev machines with many containers, cap memory at minimum. Soft vs hard limits: `--memory-reservation` (soft) + `--memory` (hard).

Watch in real time with `docker stats`.

## Stop, remove, restart

```bash
docker stop <n>              # SIGTERM, then SIGKILL after 10s (--time N to change)
docker kill <n>              # immediate SIGKILL
docker restart <n>
docker rm <n>                # must be stopped first
docker rm -f <n>             # force-stop + remove
docker container prune          # remove all stopped
```

Restart policies (survive daemon restart):

| Policy | Behavior |
|---|---|
| `no` (default) | never restart |
| `on-failure[:N]` | restart on non-zero exit, up to N retries |
| `always` | always restart, even if stopped manually |
| `unless-stopped` | always restart unless explicitly stopped |

```bash
docker run --restart unless-stopped ...
```

## Copy files in/out

```bash
docker cp ./local.txt <n>:/app/    # host → container
docker cp <n>:/app/log.txt ./      # container → host
```

For ongoing sync, use bind mounts (dev) or Compose Watch — not `cp`.

## Logs

Docker captures stdout/stderr. Don't write logs to files inside the container — the host can't see them and they inflate the container layer.

```bash
docker logs <n>
docker logs -f --since 10m --tail 50 <n>
docker logs --timestamps <n>
```

## Common gotchas

| Problem | Cause | Fix |
|---|---|---|
| Container exits immediately | No foreground process; `CMD` ran and finished | Run a long-lived process (`nginx -g 'daemon off;'`), not the default background-forking mode |
| "bind: address already in use" | Host port already bound | Change host port or kill existing process |
| Changes inside container vanish on restart | Filesystem is ephemeral | Use a volume for that path |
| `exec` fails with "executable file not found" | Image has no shell (distroless/scratch) | Use OrbStack Debug Shell, or pick a debug image tag |
| amd64 image runs slow on M-series Mac | Emulated | On OrbStack this uses Rosetta (fast). Otherwise use qemu |
