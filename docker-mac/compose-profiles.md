# Compose profiles

Turn sets of services on/off by name. Services with no `profiles:` attribute always run. Services with a profile run only when that profile is activated.

Use for: optional tools (debuggers, migrations, seeders), environment-specific services (test runner, admin UI), things you don't want up every time.

## Declare

```yaml
services:
  api:
    build: .
    # no profiles → always runs

  db:
    image: postgres:18
    # no profiles → always runs

  pgadmin:
    image: dpage/pgadmin4
    profiles: [debug]
    depends_on: [db]

  seed:
    build: .
    command: npm run seed
    profiles: [seed]
    depends_on:
      db:
        condition: service_healthy

  playwright:
    image: mcr.microsoft.com/playwright
    profiles: [test, ci]
    depends_on: [api]
```

## Activate

```bash
docker compose up                              # api, db
docker compose --profile debug up              # api, db, pgadmin
docker compose --profile seed up               # api, db, seed
docker compose --profile debug --profile seed up   # everything in those profiles
docker compose --profile "*" up                # all profiles
```

Via env var:

```bash
COMPOSE_PROFILES=debug,seed docker compose up
```

## Naming conventions

Common profile names:

| Profile | Contents |
|---|---|
| `debug` | pgadmin, redis-commander, mailhog, etc. |
| `seed` | one-shot data loaders |
| `test` | test runners, mocked external services |
| `monitoring` | prometheus, grafana, loki |
| `ci` | CI-only helpers |

## Behavior with dependencies

A non-profile service that `depends_on:` a profiled service will pull the profiled one in automatically:

```yaml
services:
  api:
    depends_on: [pgadmin]         # pgadmin is `profiles: [debug]`

  pgadmin:
    profiles: [debug]
```

`docker compose up api` → starts both. The depends-on relationship implicitly activates the profile.

Reverse direction (profiled service depends on always-on service) just works — the always-on service is already running.

## Multiple profiles on one service

```yaml
services:
  metrics:
    profiles: [monitoring, debug]
```

Runs if *either* `monitoring` or `debug` is active (OR, not AND).

## Run a single profiled service

```bash
docker compose run --rm seed
```

`run` respects profiles — if the service has `profiles: [seed]`, you don't need to activate; it's implicit for that invocation.

## Typical setup

```yaml
services:
  api:
    build: .

  db:
    image: postgres:18
    healthcheck: {...}

  # Dev convenience — activate with --profile debug
  pgadmin:
    image: dpage/pgadmin4
    profiles: [debug]
    ports: ["5050:80"]

  mailhog:
    image: mailhog/mailhog
    profiles: [debug]
    ports: ["8025:8025"]

  # One-off jobs — activate with --profile seed / migrate
  migrate:
    build: .
    command: npm run migrate
    profiles: [migrate]
    depends_on:
      db: {condition: service_healthy}

  seed:
    build: .
    command: npm run seed
    profiles: [seed]
    depends_on:
      db: {condition: service_healthy}

  # Load tests — CI only
  k6:
    image: grafana/k6
    profiles: [loadtest]
    volumes: [./loadtest:/scripts]
    command: run /scripts/main.js
```

Daily flow:

```bash
docker compose up -d                           # api + db
docker compose --profile debug up -d pgadmin   # add admin UI
docker compose run --rm migrate                # one-off migration
docker compose --profile loadtest up k6        # run load test
```
