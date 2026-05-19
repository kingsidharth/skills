# Compose in production

Compose is fine for small production deployments (single host, handful of services) — think self-hosted tools, internal apps, edge boxes. For anything requiring multi-host, rolling updates, autoscaling, or HA, use Kubernetes / Nomad / Swarm.

This doc is about **hardening a Compose file for prod**, not arguing for/against Compose at scale.

## Base + prod override pattern

Keep `compose.yaml` clean and dev-friendly. Layer prod concerns on top.

```
compose.yaml           # everything needed in dev
compose.override.yaml  # dev-only (auto-loaded)
compose.prod.yaml      # prod-only (explicit)
```

Deploy:

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

## Prod-appropriate settings

### Pin image versions — never `latest`

```yaml
# bad (dev maybe, prod never)
image: postgres:latest

# good
image: postgres:18.1

# best
image: postgres@sha256:abc123...   # digest pin, immutable
```

### Restart policy

```yaml
services:
  api:
    restart: unless-stopped    # survives host reboot, manual stop is respected
```

`always` works but also restarts after explicit stops — annoying for maintenance.

### Resource limits

Prevent one service from starving the rest of the box:

```yaml
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 256M
```

`deploy.resources.limits` is honored by Compose (despite living under `deploy:` which is mainly Swarm-flavored). `reservations` is a soft floor.

### Logging driver + rotation

The default json-file driver grows unbounded. At minimum set rotation:

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"
```

For remote log aggregation:

```yaml
logging:
  driver: loki
  options:
    loki-url: http://loki:3100/loki/api/v1/push
    loki-retries: "5"
```

Other drivers: `syslog`, `journald`, `fluentd`, `gelf`, `awslogs`, `splunk`.

### Healthchecks on everything user-facing

See [compose-dependencies.md](compose-dependencies.md). Required for:

- `depends_on: condition: service_healthy` to mean anything
- External orchestrators / load balancers that probe the container
- Reverse proxies that mark backends up/down

### Drop capabilities / add read-only

```yaml
services:
  api:
    read_only: true                  # rootfs read-only
    tmpfs:
      - /tmp
      - /run
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE            # only if binding <1024
    security_opt:
      - no-new-privileges:true
    user: "1001:1001"                # non-root
```

### Secrets, not env vars

All credentials as secrets with `_FILE` variant. See [compose-secrets-env.md](compose-secrets-env.md).

### Private network for DB

```yaml
services:
  api:
    networks: [frontend, backend]
    ports: ["443:443"]

  db:
    networks: [backend]
    # NO ports: — never expose to host

networks:
  frontend:
  backend:
    internal: true      # no outbound internet either
```

## Prod `compose.prod.yaml` example

```yaml
services:
  api:
    image: myorg/api@sha256:abc...
    restart: unless-stopped
    read_only: true
    tmpfs: ["/tmp"]
    cap_drop: [ALL]
    user: "1001:1001"
    deploy:
      resources:
        limits: {cpus: '2', memory: 1G}
    logging:
      driver: json-file
      options: {max-size: "10m", max-file: "5"}
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    environment:
      NODE_ENV: production
      DATABASE_URL_FILE: /run/secrets/database_url
    secrets:
      - database_url

  db:
    image: postgres@sha256:def...
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [backend]
    logging:
      driver: json-file
      options: {max-size: "50m", max-file: "5"}

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
    depends_on:
      api:
        condition: service_healthy
    networks: [frontend]

networks:
  frontend:
  backend:
    internal: true

volumes:
  pgdata:
  caddy_data:

secrets:
  db_password:
    file: /etc/myapp/secrets/db_password
  database_url:
    file: /etc/myapp/secrets/database_url
```

## Deployment workflow

Build elsewhere (CI), push to registry, pull + up on host:

```bash
# On host:
docker compose -f compose.yaml -f compose.prod.yaml pull
docker compose -f compose.yaml -f compose.prod.yaml up -d
docker compose -f compose.yaml -f compose.prod.yaml ps
```

Rolling-ish update for a single service (zero downtime requires a proxy, see below):

```bash
docker compose pull api
docker compose up -d api
```

## Zero-downtime with a reverse proxy

Caddy, Traefik, or Nginx in front. Use healthchecks + `depends_on: service_healthy`. For true zero-downtime on a single host: run two replicas of the app and roll them one at a time, or use a proxy with its own health-based draining (Caddy/Traefik do this automatically for services they reverse-proxy).

## Backup & restore

Volumes are dirs — backup with a throwaway container:

```bash
docker run --rm \
  -v pgdata:/src:ro \
  -v /backup:/out \
  alpine tar czf /out/pgdata-$(date +%F).tar.gz -C /src .
```

For Postgres specifically, prefer `pg_dump` over raw volume backups — it's consistent across versions.

## Observability sidecars

Typical stack to run alongside the app on a small host:

```yaml
services:
  prometheus:
    image: prom/prometheus
    profiles: [monitoring]
    volumes: [./prometheus.yml:/etc/prometheus/prometheus.yml:ro]

  grafana:
    image: grafana/grafana
    profiles: [monitoring]
    ports: ["3001:3000"]

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    profiles: [monitoring]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
```

## When to outgrow Compose

- You need multi-host (across VMs).
- You need auto-scaling.
- You need rolling updates with automated rollback.
- You have 15+ services with complex dependency management.

At that point: Kubernetes (or Nomad, or Swarm if you want to stay close to Compose). Compose stays useful for dev even after you move prod elsewhere — it's the fastest local equivalent.
