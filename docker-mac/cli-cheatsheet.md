# CLI cheatsheet

Flat reference. Deeper semantics live in the topic files this links to.

## Containers

```bash
docker run [opts] <image> [cmd]   # create + start
docker ps                          # running (-a for all)
docker logs -f <name>              # tail logs
docker exec -it <name> <cmd>       # shell into running container
docker stop <name> && docker rm <name>
docker stats                       # live CPU/mem/net/io
docker inspect <name>              # full JSON
```

Common `run` flags:

| Flag | Effect |
|---|---|
| `-d` | detached |
| `-it` | interactive + TTY |
| `--rm` | remove on exit |
| `-p HOST:CONTAINER` | publish port |
| `-P` | publish all `EXPOSE`d ports to random host ports |
| `-v NAME:/path` | named volume |
| `-v /host:/container` | bind mount (use `--mount` for prod) |
| `-e KEY=VAL` or `--env-file .env` | env |
| `--name <n>` | stable name |
| `--network <net>` | attach to custom network |
| `--memory 512m --cpus 0.5` | resource limits |
| `--restart unless-stopped` | auto-restart policy |
| `--platform linux/amd64` | force arch (Mac: use Rosetta) |

## Images

```bash
docker build -t app:tag .          # build with tag
docker buildx build ...            # BuildKit (default on modern Docker)
docker images                      # list
docker image history <image>       # layer breakdown
docker image tag src dst           # re-tag
docker push <registry>/<image>
docker pull <image>
docker login [registry]
docker rmi <image>
docker image prune [-a]            # remove unused
```

## Volumes

```bash
docker volume create <n>
docker volume ls
docker volume inspect <n>
docker volume rm <n>
docker volume prune
```

## Networks

```bash
docker network create <n>
docker network ls
docker network inspect <n>
docker network connect <net> <container>
docker network rm <n>
```

## System / cleanup

```bash
docker system df                   # disk usage breakdown
docker system prune                # containers + networks + dangling images
docker system prune -a --volumes   # nuclear: everything unused
docker builder prune               # just BuildKit cache
```

## Compose

```bash
docker compose up -d               # start in background
docker compose up --build          # rebuild images first
docker compose up --watch          # with file watch
docker compose down                # stop + remove containers/networks
docker compose down -v             # also remove volumes
docker compose ps
docker compose logs -f [svc]
docker compose exec <svc> <cmd>
docker compose restart [svc]
docker compose build [--no-cache]
docker compose config              # validated merged config
docker compose --profile <p> up    # activate profile
```

## Buildx

```bash
docker buildx ls                   # list builders
docker buildx create --use         # new container-driver builder
docker buildx build --platform linux/amd64,linux/arm64 -t <img> --push .
docker buildx bake [target]        # HCL/JSON-driven multi-target builds
docker buildx imagetools inspect <img>
```

## Registry tag structure

```
[HOST[:PORT]/]PATH[:TAG]

nginx                            → docker.io/library/nginx:latest
myorg/app:v1.2                   → docker.io/myorg/app:v1.2
ghcr.io/org/app:sha-abc          → GitHub Container Registry
registry.example.com:5000/app    → private registry
```

## OrbStack-only

```bash
orb config docker                  # edit daemon.json
orb restart docker
orb logs docker
orb migrate docker                 # import Docker Desktop data
orb create ubuntu                  # Linux machine
```
