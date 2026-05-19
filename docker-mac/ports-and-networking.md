# Ports & networking

Containers run in their own network namespace. To reach them from the host or from each other, you either **publish a port** (host↔container) or put them on a **user-defined network** (container↔container with DNS).

## Publishing ports — `-p`

```bash
docker run -d -p <HOST>:<CONTAINER> <image>
docker run -d -p 8080:80 nginx           # host 8080 → container 80
docker run -d -p 127.0.0.1:8080:80 nginx # bind to loopback only
docker run -d -p 80 nginx                # host port is ephemeral (random)
docker run -d -P nginx                   # publish all EXPOSEd ports to random host ports
```

Verify:

```bash
docker port <container>
```

**Default binds to all interfaces.** A published port is reachable from anywhere that can reach your machine. Bind to `127.0.0.1:<port>:<port>` for anything sensitive, or stay off `-p` entirely when only other containers need access.

`EXPOSE` in a Dockerfile is **documentation only** — it doesn't publish anything. `-P` uses it to pick which ports to map.

## User-defined bridge networks

The default `bridge` network exists but has poor DNS and no isolation. Always create your own.

```bash
docker network create app-net
docker run -d --name db --network app-net postgres:18
docker run -d --name api --network app-net -p 3000:3000 myapi
# `api` reaches postgres via `db:5432` — DNS works by container name
```

Features of user-defined bridges (vs default bridge):

| Capability | default | user-defined |
|---|---|---|
| DNS by container name | no | yes |
| DNS aliases (`--network-alias`) | no | yes |
| Isolation from other networks | no | yes |
| Attach/detach running container | no | yes |

Containers on the *same* user-defined network talk on **container ports directly** — no `-p` needed between them. Only use `-p` for the host.

## Compose networks

Compose creates one default network per project (named `<project>_default`). Every service joins it automatically and reaches others by service name.

```yaml
services:
  api:
    build: .
    ports:
      - "3000:3000"    # only api is exposed to host
  db:
    image: postgres:18
    # no ports: — reachable only from other services in this project
```

Explicit networks for multi-tier apps:

```yaml
services:
  web:
    networks: [frontend]
  api:
    networks: [frontend, backend]
  db:
    networks: [backend]

networks:
  frontend:
  backend:
    internal: true   # no external connectivity (no NAT out)
```

`internal: true` is a common and easy-to-miss security win — databases never need outbound internet.

Connect to an existing external network:

```yaml
networks:
  default:
    name: shared-net
    external: true
```

## Host networking

Container shares the host's network namespace — no port mapping, no isolation.

```bash
docker run --network host nginx
```

On Mac (both Docker Desktop and OrbStack), `--network host` is partially supported — containers bind to a gateway interface, not literally the Mac's interfaces.

## host.docker.internal

Container → host, from inside the container:

```
host.docker.internal
```

Works on Mac/Windows out of the box. On Linux you need `--add-host=host.docker.internal:host-gateway`.

OrbStack also provides `host.orb.internal` (same thing) and `docker.orb.internal` (reach Docker containers from Linux machines).

## OrbStack container domains

Zero-config DNS: `<container>.orb.local` and `<service>.<project>.orb.local` for Compose. Web services work without port numbers. See [orbstack-mac.md](orbstack-mac.md).

## Network drivers

| Driver | Use |
|---|---|
| `bridge` (default) | Standard container↔container + host NAT |
| `host` | Share host namespace — no isolation |
| `none` | No network at all |
| `overlay` | Multi-host (Swarm) |
| `macvlan` / `ipvlan` | Container gets a real LAN IP |

99% of dev uses `bridge`.

## Published ports — security note

A published port is reachable from anywhere that can reach your machine (LAN, sometimes internet). Never publish:

- Databases (`postgres`, `mysql`, `redis`, `mongo`) without auth
- Internal admin UIs
- Any service you wouldn't put directly on the internet

Keep these inside the Compose network. If you need to connect from a host tool (psql, DB GUI), bind to loopback: `-p 127.0.0.1:5432:5432`.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `bind: address already in use` | Host port already taken |
| Container can't reach another by name | Not on the same user-defined network, or typo'd service name |
| Can ping container from host but TCP refused | Service not listening, or listening on `127.0.0.1` inside container (bind to `0.0.0.0`) |
| Works on Linux, broken on Mac | `--network host`, or missing `host.docker.internal` |
| Published port works from Mac, not from phone on same WiFi | Mac firewall blocking; check System Settings |
