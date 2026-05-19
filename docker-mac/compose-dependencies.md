# Compose dependencies & startup order

`depends_on` controls start order and — with healthchecks — waits for services to be *actually ready*, not just "process started".

## Short form (container started only)

```yaml
services:
  api:
    depends_on:
      - db
      - redis
```

Only guarantees: `db` and `redis` have **started** before `api`. The DB process might still be initializing. Apps that connect immediately will see "connection refused" and crash.

Use this for containers that truly start ready (e.g., `redis` is usually ready instantly) or when your app has its own retry logic.

## Long form with conditions

```yaml
services:
  api:
    depends_on:
      db:
        condition: service_healthy
        restart: true            # restart api when db restarts
      redis:
        condition: service_started
      migrate:
        condition: service_completed_successfully
```

| Condition | Meaning |
|---|---|
| `service_started` | Container running (same as short form) |
| `service_healthy` | Healthcheck passing |
| `service_completed_successfully` | Container exited 0 (one-shot job) |

`restart: true` is a nice touch: when the dep is restarted via `docker compose restart`, its dependents get restarted too.

`required: false` (Compose ≥ 2.20) downgrades a missing dependency to a warning instead of an error.

## Healthcheck

For `service_healthy` to mean anything, the dep needs a working healthcheck. Two places to put it:

### In the image (Dockerfile)

```dockerfile
HEALTHCHECK --interval=10s --timeout=5s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

### In the Compose file (overrides image's)

```yaml
services:
  db:
    image: postgres:18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 10s
```

Fields:

| Field | Purpose |
|---|---|
| `test` | Command to run. Prefix with `CMD-SHELL` (string) or `CMD` (list) |
| `interval` | How often to run |
| `timeout` | Each check's max duration |
| `retries` | Consecutive failures before "unhealthy" |
| `start_period` | Grace period at startup where failures don't count |
| `start_interval` | Faster interval during `start_period` (Compose ≥ 2.20) |

## Canonical healthchecks by service

```yaml
# PostgreSQL
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
  interval: 5s
  timeout: 5s
  retries: 5
  start_period: 10s

# MySQL / MariaDB
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 5s
  timeout: 3s
  retries: 5
  start_period: 10s

# Redis
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 3s
  retries: 5

# MongoDB
healthcheck:
  test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
  interval: 10s
  timeout: 5s
  retries: 5

# RabbitMQ
healthcheck:
  test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
  interval: 10s
  timeout: 5s
  retries: 5

# Elasticsearch
healthcheck:
  test: ["CMD-SHELL", "curl -sf http://localhost:9200/_cluster/health | grep -q '\"status\":\"\\(yellow\\|green\\)\"'"]
  interval: 10s
  timeout: 5s
  retries: 10
  start_period: 30s

# HTTP service
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://localhost:3000/health || exit 1"]
  interval: 10s
  timeout: 5s
  retries: 3
  start_period: 15s
```

## Migration pattern — run once, then start app

```yaml
services:
  db:
    image: postgres:18
    healthcheck: {...}
    volumes: [pgdata:/var/lib/postgresql/data]

  migrate:
    build: .
    command: ["npm", "run", "migrate"]
    environment:
      DATABASE_URL: postgres://app:secret@db/app
    depends_on:
      db:
        condition: service_healthy
    restart: "no"       # explicitly one-shot

  api:
    build: .
    depends_on:
      migrate:
        condition: service_completed_successfully
      db:
        condition: service_healthy

volumes:
  pgdata:
```

`migrate` runs once, exits 0, and then `api` starts. If migration fails, `api` never starts.

## Disabling a healthcheck

```yaml
services:
  app:
    healthcheck:
      disable: true

    # or:
    healthcheck:
      test: ["NONE"]
```

Necessary if the base image has one that's wrong for your use.

## Important caveats

- **Healthchecks don't auto-restart unhealthy containers.** You need `restart:` policy for that, and even then, restart is based on exit status, not health. Enable `autoheal` via a sidecar if you need this.
- **`depends_on` doesn't cascade across `docker compose` invocations.** `docker compose up api` *does* start its deps. `docker compose restart api` does *not* restart its deps.
- **Circular deps** error out. Break the loop with a healthcheck instead.

## Viewing health state

```bash
docker compose ps                                    # shows health column
docker inspect --format='{{json .State.Health}}' <container> | jq
docker events --filter event=health_status           # watch transitions
```
