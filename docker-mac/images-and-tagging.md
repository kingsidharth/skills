# Images — build, tag, publish

An image is a read-only filesystem snapshot + config (env, cmd, entrypoint, ports). Tags are human-readable pointers; the real identity is the SHA256 digest.

## Tag structure

```
[HOST[:PORT]/]PATH[:TAG]
```

| Part | Default if omitted | Notes |
|---|---|---|
| `HOST[:PORT]` | `docker.io` | Registry hostname |
| `PATH` | on Hub, `library/<name>` | namespace/repository |
| `TAG` | `latest` | version/variant marker |

Examples:

| Written | Resolves to |
|---|---|
| `nginx` | `docker.io/library/nginx:latest` |
| `myorg/app:v1.2` | `docker.io/myorg/app:v1.2` |
| `ghcr.io/org/app:sha-abc` | GitHub Container Registry |
| `registry.example.com:5000/app` | private registry on port 5000 |

`latest` is a *tag*, not a special keyword — never rely on it for production. Pin to versions or digests.

## Build

```bash
docker build -t myapp:dev .
docker build -t myapp:dev -t myapp:latest .    # multiple tags
docker build --target builder -t myapp:build . # stop at specific multi-stage target
docker build --platform linux/amd64 -t myapp . # force arch
docker build --no-cache -t myapp .             # skip cache
```

The trailing `.` is the **build context** — the directory uploaded to the daemon. `.dockerignore` controls what ships. Big contexts = slow builds; see [dockerfile-writing.md](dockerfile-writing.md).

## Tag an already-built image

```bash
docker image tag myapp:dev myorg/myapp:v1.0
docker image tag myapp:dev ghcr.io/myorg/myapp:$(git rev-parse --short HEAD)
```

## Push to registry

```bash
docker login                                   # Docker Hub
docker login ghcr.io                           # GitHub (use PAT)
docker login registry.example.com

docker push myorg/myapp:v1.0
```

Authentication failures (`requested access to the resource is denied`) almost always mean the tag doesn't match the logged-in namespace.

## Pin to digest (production)

Tags are mutable; digests aren't.

```bash
docker pull nginx@sha256:abc123...
# In Dockerfile:
FROM nginx@sha256:abc123...
```

Get the digest of a local image:

```bash
docker inspect --format='{{index .RepoDigests 0}}' nginx
```

## Inspect

```bash
docker image ls
docker image inspect myapp:dev
docker image history myapp:dev        # layer-by-layer breakdown with sizes
docker image tree myapp:dev           # (with docker-scout / dive) visualize layers
```

`history` is your first stop when an image is larger than expected — it shows which instruction added how much.

## Clean up

```bash
docker image prune                    # dangling only (untagged)
docker image prune -a                 # all unused
docker rmi <image>
docker system df                      # see what's eating disk
```

## Registries worth knowing

| Registry | Use when |
|---|---|
| Docker Hub (`docker.io`) | public images, open source |
| GHCR (`ghcr.io`) | tied to GitHub repo, free for public |
| ECR (`<id>.dkr.ecr.<region>.amazonaws.com`) | AWS-native |
| GCR/Artifact Registry | GCP-native |
| Private (Harbor, etc.) | air-gapped / regulated |

Docker Hub has rate limits for anonymous pulls. Log in or use a mirror if you hit them.

## Multi-platform

Single command builds for multiple architectures via Buildx (requires the `docker-container` driver):

```bash
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 -t myorg/app:v1 --push .
```

`--push` is required for multi-platform — local Docker image store can't hold multi-arch manifests (unless containerd image store is enabled). See [buildkit-and-buildx.md](buildkit-and-buildx.md).
