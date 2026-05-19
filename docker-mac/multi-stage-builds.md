# Multi-stage builds

One Dockerfile, multiple `FROM` statements. Each stage is independent — use a heavy image for building, a tiny one for running, and copy only the artifact across.

Always the right choice. The saving is typically 50–95% of image size.

## Shape

```dockerfile
FROM <builder-image> AS build
# install build tools, compile, bundle
...

FROM <runtime-image> AS runtime
COPY --from=build /path/to/artifact /app/
CMD ["..."]
```

The final stage (last one in the file) is the default target when you run `docker build`. Earlier stages exist only to produce artifacts unless you explicitly target them.

## Target a specific stage

```bash
docker build --target build -t app:build-stage .   # stop at `build`
docker build -t app:latest .                        # default: last stage
```

Handy for CI — run tests against the `build` stage (has tooling), ship the final one (doesn't).

## `COPY --from=<stage>`

```dockerfile
FROM node:22 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
```

Can also copy from an external image:

```dockerfile
COPY --from=alpine:3.20 /etc/ssl/certs /etc/ssl/certs
```

## Common patterns

**Compiled languages (Go, Rust, C/C++):**

```dockerfile
FROM rust:1.82 AS build
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/src/target \
    cargo build --release && \
    cp target/release/myapp /myapp

FROM gcr.io/distroless/cc-debian12
COPY --from=build /myapp /myapp
ENTRYPOINT ["/myapp"]
```

Static Rust binary → `FROM scratch` for an image measured in MB not hundreds of MB.

**Interpreted languages (Node, Python):**

Split deps and build output — the runtime stage doesn't need dev dependencies or build toolchain.

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
USER node
CMD ["node", "dist/server.js"]
```

**Monorepo with shared base:**

```dockerfile
FROM node:22-alpine AS base
WORKDIR /app
RUN npm install -g pnpm

FROM base AS deps
COPY pnpm-workspace.yaml package.json pnpm-lock.yaml ./
COPY apps/api/package.json apps/api/
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile

FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm --filter api build

FROM node:22-alpine AS runtime
COPY --from=build /app/apps/api/dist ./
CMD ["node", "server.js"]
```

## Parallel stages

Independent stages build in parallel automatically under BuildKit:

```dockerfile
FROM node:22 AS frontend
COPY frontend/ .
RUN npm ci && npm run build

FROM golang:1.23 AS backend
COPY backend/ .
RUN go build -o server .

FROM alpine
COPY --from=frontend /app/dist /srv/static
COPY --from=backend /app/server /srv/server
CMD ["/srv/server"]
```

`frontend` and `backend` build concurrently.

## Named intermediate for cache

Useful when you want to keep a heavy build stage around for dev/testing without pushing it:

```bash
docker build --target build -t app:build .
docker build --target runtime -t app:latest .
```

Second build reuses layers from the first.

## Tips

- Put the runtime stage *last* in the file; it's the default target.
- Each stage gets its own cache; editing stage 3 doesn't invalidate stage 1.
- Don't over-split — 2–4 stages is normal. 10 stages is a code smell.
- `COPY --from=<stage> --chown=user:group` to set ownership on copied files.
- For the smallest possible runtime: `scratch` (static binary only), then `distroless`, then `alpine`, then `<lang>-slim`.

## Size comparison example

Single-stage Java build (JDK + Maven + app) vs multi-stage (JRE only):

| Approach | Size |
|---|---|
| Single stage (full JDK + Maven) | ~880 MB |
| Multi-stage (JRE only) | ~428 MB |
| Multi-stage + jlink custom JRE | ~150 MB |
| Multi-stage + distroless | ~250 MB |

Numbers vary, but the pattern is always the same: separation of build and runtime wins.
