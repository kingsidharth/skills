# Server Setup

## Auth keys

Auth keys let you join a device to a tailnet non-interactively. Generate them in the admin console under **Settings → Keys**, or programmatically via an OAuth client.

### Key types

- **One-off**: single use, good for a single cloud server
- **Reusable**: multiple devices can use the same key
- **Ephemeral**: device auto-removed ~30-60 min after going offline (containers, lambdas, short-lived workers)
- **Pre-approved**: skips device approval if your tailnet requires it
- **Tagged**: automatically applies a tag identity to the device

### Usage

```bash
# Basic server join
sudo tailscale up --auth-key=$TS_AUTHKEY --advertise-tags=tag:worker --ssh

# With custom hostname (affects MagicDNS name)
sudo tailscale up --auth-key=$TS_AUTHKEY --hostname=worker-01 --advertise-tags=tag:worker --ssh
```

Auth keys expire (max 90 days). For long-lived automation, use an **OAuth client** with `auth_keys` scope to generate keys on-demand:

```bash
# Generate auth key via OAuth
curl -s -X POST "https://api.tailscale.com/api/v2/oauth/token" \
  -d "client_id=$TS_OAUTH_ID&client_secret=$TS_OAUTH_SECRET&grant_type=client_credentials" \
  | jq -r '.access_token' > /tmp/ts_token

curl -s -X POST "https://api.tailscale.com/api/v2/tailnet/-/keys" \
  -H "Authorization: Bearer $(cat /tmp/ts_token)" \
  -H "Content-Type: application/json" \
  -d '{"capabilities":{"devices":{"create":{"reusable":true,"ephemeral":true,"preauthorized":true,"tags":["tag:worker"]}}}}'
```

**Security**: Store auth keys in a secrets manager. Never hardcode in scripts or commit to git.

## Tags

Tags give devices a machine identity (vs user identity). Required for servers.

```jsonc
// In tailnet policy file
{
  "tagOwners": {
    "tag:server": ["alice@example.com"],
    "tag:worker": ["tag:server"]  // tag can own other tags
  }
}
```

When a device authenticates with a tagged key, it assumes the tag identity. This determines what ACLs apply to it.

## Tailscale SSH

Enable SSH access over Tailscale — no need for traditional SSH key management or open port 22:

```bash
sudo tailscale up --auth-key=$TS_AUTHKEY --advertise-tags=tag:server --ssh
```

Then connect from any tailnet device:

```bash
tailscale ssh user@hostname
# or
ssh user@hostname.tailnet-name.ts.net
```

ACLs control who can SSH:

```jsonc
{
  "ssh": [{
    "action": "accept",
    "src": ["autogroup:admin"],
    "dst": ["tag:server"],
    "users": ["autogroup:nonroot"]
  }]
}
```

## Cloud-init provisioning

Cloud-init runs on first boot. Works with Hetzner, AWS, GCP, Azure, DigitalOcean, etc.

```yaml
#cloud-config
runcmd:
  - ['sh', '-c', 'curl -fsSL https://tailscale.com/install.sh | sh']
  - ['tailscale', 'up', '--auth-key=TS_AUTHKEY_PLACEHOLDER', '--advertise-tags=tag:worker', '--ssh']
```

### Hetzner-specific

Hetzner provides the Cloud config text box in the creation flow. Key points:

- Use `cx22` or higher (needs enough RAM for your workload)
- Allow UDP 41641 inbound in Hetzner Firewall for direct WireGuard connections (avoids DERP relay latency)
- After Tailscale is up, remove all other firewall rules in Hetzner console

Full Hetzner cloud-init example:

```yaml
#cloud-config
users:
  - name: deploy
    groups: users, admin, docker
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - <your-public-key>

packages:
  - ufw
  - curl
  - jq

package_update: true
package_upgrade: true

runcmd:
  # Install Tailscale
  - ['sh', '-c', 'curl -fsSL https://tailscale.com/install.sh | sh']
  - ['tailscale', 'up', '--auth-key=YOUR_AUTH_KEY', '--hostname=worker-hetzner-01', '--advertise-tags=tag:worker', '--ssh']

  # Lock down with UFW (see firewall-lockdown.md)
  - ['ufw', 'default', 'deny', 'incoming']
  - ['ufw', 'default', 'allow', 'outgoing']
  - ['ufw', 'allow', 'in', 'on', 'tailscale0']
  - ['ufw', '--force', 'enable']
```

## Ephemeral nodes

For short-lived workers (batch jobs, CI, containers):

```bash
# State in memory — device removed when process exits
sudo tailscaled --state=mem: &
sudo tailscale up --auth-key=$TS_EPHEMERAL_KEY --hostname=batch-$(date +%s)
# ... do work ...
sudo tailscale logout  # immediately removes from tailnet
```

## Run unattended

For persistent servers, ensure `tailscaled` starts on boot:

```bash
sudo systemctl enable tailscaled
sudo systemctl start tailscaled
```

Disable key expiry for critical infrastructure in the admin console (machine page → "Disable key expiry") to avoid getting locked out.

## Getting the Tailscale IP/hostname at runtime

```bash
# Get this node's Tailscale IPv4
tailscale ip -4
# → 100.64.1.23

# Get this node's MagicDNS FQDN
tailscale status --json | jq -r '.Self.DNSName' | sed 's/\.$//'
# → worker-01.tailnet-name.ts.net

# Get another node's IP
tailscale ip my-server
```
