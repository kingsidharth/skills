# Language guides — Bun, Python, Rust

Production-grade Dockerfile templates for the three runtimes used most here. All three follow the same principles: multi-stage, cache-friendly layer order, non-root runtime, small final image.

## Bun (TypeScript/JS)

Bun is its own runtime + package manager. Official images are at `oven/bun`. Use `distroless` or `alpine` for runtime.

```dockerfile
# syntax=docker/dockerfile:1
FROM oven/bun:1-alpine AS deps
WORKDIR /app
COPY package.json bun.lockb ./
RUN --mount=type=cache,target=/root/.bun/install/cache \
    bun install --frozen-lockfile --production

FROM oven/bun:1-alpine AS build
WORKDIR /app
COPY package.json bun.lockb ./
RUN --mount=type=cache,target=/root/.bun/install/cache \
    bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM oven/bun:1-alpine AS runtime
WORKDIR /app
RUN addgroup -S app && adduser -S app -G app
COPY --from=deps --chown=app:app /app/node_modules ./node_modules
COPY --from=build --chown=app:app /app/dist ./dist
COPY --chown=app:app package.json ./
USER app
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["bun", "run", "dist/index.js"]
```

**Single-binary option** — Bun can bundle to a self-contained executable:

```dockerfile
FROM oven/bun:1-alpine AS build
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun build --compile --target=bun-linux-x64 src/index.ts --outfile server

FROM gcr.io/distroless/base-debian12
COPY --from=build /app/server /server
CMD ["/server"]
```

Tiny image, no runtime dependency on Bun being present.

### Compose for Bun dev

```yaml
services:
  api:
    build:
      context: .
      target: build
    command: bun --hot src/index.ts
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://app:secret@db/app
    depends_on:
      db: {condition: service_healthy}
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          ignore: [node_modules/]
        - action: rebuild
          path: package.json
        - action: rebuild
          path: bun.lockb
```

## Python

Official images: `python:<version>-slim` for most cases, `python:<version>-alpine` if musl is ok. Use `uv` for fast, reproducible dep resolution.

### With uv

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy
RUN pip install --no-cache-dir uv

FROM base AS deps
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-install-project --no-dev

FROM base AS runtime
WORKDIR /app
RUN useradd -m -u 1001 app
COPY --from=deps --chown=app:app /app/.venv /app/.venv
ENV PATH="/app/.venv/bin:$PATH"
COPY --chown=app:app . .
USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### With pip (classic)

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS deps
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt ./
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
RUN useradd -m -u 1001 app
COPY --from=deps /install /usr/local
COPY --chown=app:app . .
USER app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Compose for Python dev

```yaml
services:
  api:
    build:
      context: .
      target: runtime
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgres://app:secret@db/app
    depends_on:
      db: {condition: service_healthy}
    develop:
      watch:
        - action: sync
          path: ./app
          target: /app/app
          ignore: [__pycache__/, "*.pyc", ".venv/]
        - action: rebuild
          path: pyproject.toml
        - action: rebuild
          path: uv.lock
```

### Common gotchas

- **Native deps** (numpy, pillow, psycopg2) — stick to `-slim`, not `-alpine`. musl + wheels is a pain.
- **`PYTHONUNBUFFERED=1`** — without it, `print()` output buffers and `docker logs` shows nothing until crash.
- **Don't install dev deps in runtime stage** — `uv sync --no-dev` or `pip install -r requirements.txt` (not requirements-dev.txt).

## Rust

Compiled binary → scratch/distroless. Tiny images (MB, not hundreds of MB).

```dockerfile
# syntax=docker/dockerfile:1
FROM rust:1.82-slim AS build
WORKDIR /src

# Cache deps separately from source
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/src/target \
    cargo build --release && \
    rm -rf src

# Actual build
COPY . .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/src/target \
    cargo build --release && \
    cp target/release/myapp /myapp

FROM gcr.io/distroless/cc-debian12
COPY --from=build /myapp /myapp
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

**Fully static + scratch** (requires musl target):

```dockerfile
FROM rust:1.82-alpine AS build
RUN apk add --no-cache musl-dev
WORKDIR /src
COPY . .
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/src/target \
    cargo build --release --target x86_64-unknown-linux-musl && \
    cp target/x86_64-unknown-linux-musl/release/myapp /myapp

FROM scratch
COPY --from=build /myapp /myapp
ENTRYPOINT ["/myapp"]
```

Result is often under 10 MB. No shell, no package manager, nothing to exploit.

### With cargo-chef (faster dep caching)

For big projects where even the cache mount isn't enough:

```dockerfile
FROM rust:1.82-slim AS chef
RUN cargo install cargo-chef
WORKDIR /src

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS build
COPY --from=planner /src/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release

FROM gcr.io/distroless/cc-debian12
COPY --from=build /src/target/release/myapp /myapp
ENTRYPOINT ["/myapp"]
```

### Compose for Rust dev

Rust's compile time makes bind-mount + `cargo watch` workflow reasonable for dev, *but* the target directory on a Mac bind mount is painfully slow. Use a named volume for `target/`:

```yaml
services:
  api:
    build:
      context: .
      target: build
    command: cargo watch -x run
    volumes:
      - ./:/src
      - target:/src/target        # named volume keeps target/ in Linux-native FS
      - cargo_cache:/usr/local/cargo/registry
    ports: ["8080:8080"]

volumes:
  target:
  cargo_cache:
```

## Shared patterns across all three

| Pattern | Bun | Python | Rust |
|---|---|---|---|
| Deps cache mount path | `/root/.bun/install/cache` | `/root/.cache/pip` or `/root/.cache/uv` | `/usr/local/cargo/registry` + `target/` |
| Lockfile to copy first | `bun.lockb` | `uv.lock` / `requirements.txt` | `Cargo.lock` |
| Non-root user in runtime | `app:app` (addgroup + adduser) | `app` (useradd) | not strictly needed for scratch/distroless but fine to set |
| Runtime base | `oven/bun:1-alpine` or distroless | `python:3.12-slim` | `gcr.io/distroless/cc-debian12` or `scratch` |
| Port convention | 3000 | 8000 | 8080 |
