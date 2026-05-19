# Compose secrets & env vars

Two different concerns that people often conflate:

- **Env vars** — configuration that's fine to see in logs, CI output, `docker inspect`.
- **Secrets** — credentials, private keys, tokens. These must not leak to logs, image layers, or environment.

Rule of thumb: if leaking it would be an incident, it's a secret.

## Env var mechanics

Compose distinguishes two env contexts:

1. **Host-side interpolation** — `${VAR}` in the YAML, substituted when Compose parses the file. Source: shell env + `.env` file next to `compose.yaml`.
2. **Container-side environment** — `environment:` block of a service. These become env vars *inside the container*.

```yaml
services:
  api:
    image: myapp:${TAG:-latest}         # host-side — needed to resolve the image name
    environment:
      NODE_ENV: production               # literal
      LOG_LEVEL: ${LOG_LEVEL:-info}      # interpolated from shell/.env
      DATABASE_URL: ${DATABASE_URL}      # passed through
    env_file:
      - .env.shared                      # merged into container env
      - .env.api                         # later file wins on conflicts
```

### `.env` vs `env_file`

| File | Read by | Available to |
|---|---|---|
| `.env` (next to `compose.yaml`) | Compose itself | `${VAR}` interpolation in YAML |
| `env_file:` entries | Service's container env | App running in the container |

One `.env` can do both if you reference the vars on both sides. They're **not** automatically the same.

### Interpolation syntax

```yaml
${VAR}                    # error if unset
${VAR:-default}           # default if unset or empty
${VAR-default}            # default if unset (empty is ok)
${VAR:?error message}     # fail with message if unset or empty
${VAR:+replacement}       # replacement if set
```

### Precedence (highest wins)

1. Shell env
2. `docker compose run/exec -e VAR=value`
3. `environment:` block in Compose file
4. `env_file:` entries
5. Image's `ENV` directive

## `env_file` advanced

```yaml
env_file:
  - path: .env.shared
    required: false              # don't error if missing
  - path: .env.prod
```

## Secrets — the right way

Compose's top-level `secrets:` block. Files are mounted read-only at `/run/secrets/<name>` inside the container. The secret never becomes an env var; app code reads from the file.

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

  api:
    build: .
    environment:
      DATABASE_URL_FILE: /run/secrets/database_url
    secrets:
      - database_url

secrets:
  db_password:
    file: ./secrets/db_password.txt

  database_url:
    environment: DATABASE_URL     # pull from shell env into a secret
```

Sources for a secret:

| Source | Syntax |
|---|---|
| File on disk | `file: ./path/to/secret.txt` |
| Env var (at parse time) | `environment: VAR_NAME` |
| External (Swarm/managed) | `external: true` + `name: secret-name` |

### The `_FILE` convention

Official images (postgres, mysql, redis, wordpress, etc.) support `*_FILE` env vars: instead of `POSTGRES_PASSWORD=secret`, set `POSTGRES_PASSWORD_FILE=/run/secrets/db_password`. The image's entrypoint reads the file.

Always prefer `_FILE` over plain env vars for credentials when supported.

## Build-time secrets

Don't `ARG NPM_TOKEN=...` — it ends up in image history and gets printed in build logs.

Use BuildKit build secrets:

Dockerfile:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=secret,id=npm_token,target=/root/.npmrc \
    npm ci
```

Compose:

```yaml
services:
  api:
    build:
      context: .
      secrets:
        - npm_token

secrets:
  npm_token:
    environment: NPM_TOKEN
```

## `.gitignore` essentials

```
.env
.env.*
!.env.example
secrets/
!secrets/.gitkeep
```

Commit `.env.example` with placeholder values so new devs know what to fill in.

## Environment layering pattern

```
.env                 # committed defaults (non-secret)
.env.local           # gitignored, dev overrides + dev secrets
.env.production      # not on dev machines; CI/prod only
```

Compose:

```yaml
services:
  api:
    env_file:
      - .env
      - path: .env.local
        required: false
```

## Don't do these

| Anti-pattern | Why |
|---|---|
| `ENV DB_PASSWORD=...` in Dockerfile | Baked into image forever |
| `ARG` for secrets in build | Shows in `docker history`, leaks to logs |
| Secrets via plain `environment:` | Visible in `docker inspect`, sometimes in child processes |
| Committing `.env` | It's a secret file |
| `.env` with DB passwords on a dev laptop | OK-ish for dev; never use the same values in prod |

## Debugging

```bash
docker compose config             # merged, interpolated final config
docker compose exec api env       # what env the container actually has
docker compose exec api ls /run/secrets/   # which secrets are mounted
```

`docker compose config` will flag missing interpolation variables with a warning.
