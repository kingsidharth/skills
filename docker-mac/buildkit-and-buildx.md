# BuildKit & buildx

BuildKit is the modern build backend (parallel stages, cache mounts, secrets, multi-platform). `buildx` is the CLI plugin that drives it. Default in modern Docker — `docker build` and `docker buildx build` use the same engine.

## Builders and drivers

A **builder** is a named BuildKit instance. The **driver** determines where it runs.

| Driver | Where it runs | Multi-platform | Cache export | Typical use |
|---|---|---|---|---|
| `docker` (default) | Inside the Docker daemon | no | inline only | default single-arch local builds |
| `docker-container` | In a container | yes | all backends | local/CI multi-platform |
| `kubernetes` | K8s pod | yes | all backends | CI in K8s clusters |
| `remote` | Remote BuildKit daemon | yes | all backends | shared team builder / Build Cloud |

To unlock multi-platform, cache exports, and bigger cache, switch off the default driver:

```bash
docker buildx create --name mybuilder --driver docker-container --use
docker buildx inspect --bootstrap
```

Manage:

```bash
docker buildx ls
docker buildx use <n>
docker buildx rm <n>
```

## Multi-platform builds

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myorg/app:v1 \
  --push .
```

Notes:

- `--push` is required for multi-arch — the default Docker image store can't hold a multi-arch manifest (unless you enable the containerd image store in Desktop/OrbStack settings).
- To load a single-arch result locally: `--load --platform linux/arm64`.
- QEMU emulation is automatic for cross-arch. On Mac with OrbStack, `linux/amd64` uses Rosetta instead — much faster.

## `buildx bake`

Declarative build orchestration. Define targets in `docker-bake.hcl` (or JSON/YAML), build many at once.

```hcl
group "default" {
  targets = ["api", "worker"]
}

target "api" {
  context = "./api"
  dockerfile = "Dockerfile"
  platforms = ["linux/amd64", "linux/arm64"]
  tags = ["myorg/api:latest"]
}

target "worker" {
  context = "./worker"
  tags = ["myorg/worker:latest"]
  inherits = ["api"]   # share platforms
}
```

```bash
docker buildx bake              # build default group
docker buildx bake api          # single target
docker buildx bake --push
```

Bake can also read a Compose file directly — `docker buildx bake -f compose.yaml` builds all services with `build:` blocks in parallel.

## BuildKit configuration (`buildkitd.toml`)

Tunes registry mirrors, insecure registries, cache size, network, etc. Used by `docker-container` and `remote` drivers.

```toml
debug = false

[worker.oci]
  max-parallelism = 4

[registry."docker.io"]
  mirrors = ["mirror.gcr.io"]

[registry."registry.internal:5000"]
  http = true
  insecure = true

[grpc]
  address = ["unix:///run/buildkit/buildkitd.sock"]
```

Apply at builder creation:

```bash
docker buildx create \
  --name mybuilder \
  --driver docker-container \
  --config ./buildkitd.toml \
  --use
```

Garbage-collection policy is also configured here — set `[worker.oci.gcpolicy]` entries to cap cache size and age.

## Cache size management

```bash
docker buildx du                        # usage summary
docker buildx du --verbose              # per-layer
docker buildx prune                     # all of it
docker buildx prune --keep-storage 20gb # cap
docker buildx prune --filter until=168h # older than 7 days
```

## Frontend directive

The first line of a modern Dockerfile should be:

```dockerfile
# syntax=docker/dockerfile:1
```

This pins the Dockerfile frontend to the latest stable 1.x, unlocking features like `RUN --mount`, heredocs, named contexts, etc. Without it you get an older dialect.

## Build secrets

```dockerfile
RUN --mount=type=secret,id=aws,target=/root/.aws/credentials \
    aws s3 cp s3://bucket/file .
```

```bash
docker buildx build --secret id=aws,src=$HOME/.aws/credentials -t app .
```

Or source from env:

```bash
docker buildx build --secret id=npm_token,env=NPM_TOKEN -t app .
```

## SSH agent forward at build time

```dockerfile
RUN --mount=type=ssh git clone git@github.com:private/repo.git
```

```bash
docker buildx build --ssh default -t app .
```

## Build checks (lint)

BuildKit runs lint checks and emits warnings. See them in build output. Make warnings fatal:

```dockerfile
# syntax=docker/dockerfile:1
# check=error=true
```

Skip specific rules:

```dockerfile
# check=skip=JSONArgsRecommended
```

## Attestations, SBOMs, provenance

Modern BuildKit can attach attestations to pushed images:

```bash
docker buildx build \
  --sbom=true \
  --provenance=mode=max \
  --push -t myorg/app:v1 .
```

Verify with `docker buildx imagetools inspect myorg/app:v1 --format "{{json .SBOM}}"`.

## Named contexts

Treat any resource as a build context:

```bash
docker buildx build \
  --build-context shared=../shared \
  --build-context base=docker-image://myorg/base:v1 \
  -t app .
```

In Dockerfile: `COPY --from=shared /src /shared`.

## Related: [build-exporters.md](build-exporters.md)

For writing build output as tar/local files/OCI layouts instead of images.
