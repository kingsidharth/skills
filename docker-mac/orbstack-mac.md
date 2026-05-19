# OrbStack on Mac

Drop-in Docker Desktop replacement. Fast, low RAM, native Swift GUI, automatic container domains with HTTPS. Mac only.

## Install

```bash
brew install --cask orbstack
# or download from https://orbstack.dev
```

On first launch, OrbStack offers to migrate Docker Desktop data (containers, volumes, images). Optional — re-run later via `File > Migrate Docker Data` or `orb migrate docker`.

## Docker context

OrbStack creates a context named `orbstack` and auto-switches to it. Run side-by-side with Docker Desktop by flipping contexts:

```bash
docker context ls
docker context use orbstack       # OrbStack
docker context use desktop-linux  # Docker Desktop
```

If `/var/run/docker.sock` compatibility matters for third-party tools, grant admin access on first launch — OrbStack symlinks it automatically.

## Container domains

Every container gets a DNS name at `<name>.orb.local` with zero config. Compose services are grouped by project: `<service>.<project>.orb.local`.

- Web services work without port numbers — OrbStack auto-detects the listening port.
- Visit `orb.local` in a browser for a dashboard of running containers.
- Multiple ports on one container → set label `dev.orbstack.http-port=8080` to pick the one exposed via domain.
- Custom domains: label `dev.orbstack.domains=myapp.local,*.myapp.local`.

## Automatic HTTPS

Type `https://orb.local` or `https://<service>.<project>.orb.local` and OrbStack provisions certificates on the fly via its root CA (installed to macOS keychain on first use). Root CA is also injected into containers so `orb.local` domains work between containers without disabling TLS verification. Disable per-container with label `dev.orbstack.add-ca-certificates=false`.

## Rosetta for amd64

On Apple Silicon, OrbStack uses Rosetta to run `linux/amd64` images with near-native speed. Enabled by default. Run/build cross-platform:

```bash
docker run --rm --platform linux/amd64 alpine
docker build --platform linux/amd64,linux/arm64 -t app .
```

## Host ↔ container access

| From | To | Address |
|---|---|---|
| Container | Mac | `host.docker.internal` |
| Mac | Container | `localhost:<forwarded-port>` or `<name>.orb.local` |
| Linux machine | Mac | `host.orb.internal` |
| Linux machine | Docker container port | `docker.orb.internal:<port>` |

## SSH agent forwarding

```bash
docker run -it --rm \
  -v /run/host-services/ssh-auth.sock:/agent.sock \
  -e SSH_AUTH_SOCK=/agent.sock \
  alpine
```

Works with 1Password's agent (unlike `$SSH_AUTH_SOCK` passthrough).

## Volumes on Mac

Read-write view of all container data at `~/OrbStack` — inspect, debug, edit files directly. Takes no disk space (virtualized view). Actual data lives in `~/Library/Group Containers/HUAQ24HBR6.dev.orbstack/data`.

## Daemon config

Edit `daemon.json` for registry mirrors, insecure registries, etc:

```bash
orb config docker        # opens editor
orb restart docker       # apply changes
orb logs docker          # view engine logs
```

Config file: `~/.orbstack/config/docker.json`.

## Credential store

Uses `osxkeychain` (not Docker Desktop's `desktop` store). After switching from Docker Desktop, `docker login` again for registries.

## When OrbStack ≠ Docker Desktop behavior

- No Docker Scout, no Docker Extensions, no Hardened Images integration.
- No built-in Podman (create a Linux machine and install it manually).
- Graphical apps in containers need XQuartz + `DISPLAY=host.docker.internal:0`.
- No Windows/Linux version — Mac only.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `orb.local` domain doesn't load web service | Container uses non-standard port — add `dev.orbstack.http-port=<port>` label |
| Third-party tool fails to find docker socket | Grant admin access, or alias `/var/run/docker.sock → ~/.orbstack/run/docker.sock` |
| IP range conflicts with VPN | Change container `bip` in daemon.json (e.g. `198.19.192.1/23`) |
| Firefox ≤119 rejects HTTPS cert | Import root CA via Firefox settings (system keychain ignored) |
