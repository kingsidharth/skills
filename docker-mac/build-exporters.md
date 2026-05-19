# Build exporters

By default `docker build` produces an image in the local store. Exporters let you redirect output — to a tar file, a local directory, an OCI layout, or multiple registries at once.

Controlled via `--output` (short: `-o`). The same goal can be expressed with `--load`, `--push`, or explicit `type=...`.

## Exporters at a glance

| Type | Use when |
|---|---|
| `image` / `registry` | Normal: load to local store or push to registry |
| `local` | Extract a filesystem directory (e.g. compiled binaries) |
| `tar` | Same as `local` but as a single tar stream |
| `oci` | Standard OCI image layout directory |
| `docker` | Legacy Docker tar format (single-arch) |

## Convenience flags

```bash
docker buildx build --load   # = --output type=docker
docker buildx build --push   # = --output type=registry
```

`--load` works only with single-platform builds and the default image store. `--push` is required for multi-platform.

## Local / tar — extracting binaries

Useful when the Dockerfile's real purpose is building an artifact you want to grab out.

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/myapp .

FROM scratch AS artifact
COPY --from=build /out/myapp /
```

```bash
# Write the artifact stage to ./bin as files
docker buildx build --target artifact --output type=local,dest=./bin .

# Or as a single tar
docker buildx build --target artifact --output type=tar,dest=./out.tar .
```

The stage you target with `--output type=local` must contain *only* the files you want exported — the entire filesystem of that stage is emitted. That's why the pattern above uses `FROM scratch` for the export stage.

Cross-compile for multiple platforms into per-arch subdirs:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --target artifact \
  --output type=local,dest=./bin .
# Result: ./bin/linux_amd64/myapp, ./bin/linux_arm64/myapp
```

## OCI layout

Standard-compliant directory, consumable by other OCI tools (skopeo, crane, cosign):

```bash
docker buildx build --output type=oci,dest=./image.tar -t myorg/app:v1 .
```

Push from an OCI layout later with `skopeo copy oci:./layout:v1 docker://registry/image:v1`.

## Docker tar (legacy)

Single-arch Docker-format tar. Consumable by `docker load`.

```bash
docker buildx build --output type=docker,dest=./image.tar -t myorg/app:v1 .
docker load -i ./image.tar
```

Note: multi-platform results can't use `type=docker` — use `type=oci` instead.

## Push to multiple destinations

```bash
docker buildx build \
  -t myorg/app:v1 \
  -t ghcr.io/myorg/app:v1 \
  --push .
```

BuildKit pushes to both registries from one build — layers are uploaded once per registry, not rebuilt.

## CI pattern: build once, test, push

```bash
# Build once to local
docker buildx build --target test -t app:test --load .
docker run --rm app:test                     # run tests

# Promote same image to registry
docker buildx build --target runtime -t myorg/app:$SHA --push .
```

Cache from the first call is reused in the second (if you also pass `--cache-to` / `--cache-from`).

## Compose equivalent

In `compose.yaml`:

```yaml
services:
  api:
    build:
      context: .
      target: runtime
      platforms:
        - linux/amd64
        - linux/arm64
      tags:
        - myorg/api:v1
        - ghcr.io/myorg/api:v1
```

Then:

```bash
docker buildx bake -f compose.yaml --push
```
