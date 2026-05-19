# Firewall Lockdown

Lock down a server so it only accepts connections over Tailscale while retaining full outbound internet access.

## Strategy

1. Install Tailscale, join tailnet
2. SSH in via Tailscale IP (verify connectivity)
3. Configure UFW to deny all incoming except `tailscale0` interface
4. Allow all outgoing (needed for B2 uploads, image downloads, apt, etc.)
5. Optionally disable public SSH entirely

## Step-by-step

### 1. Verify Tailscale connectivity first

```bash
# Get your Tailscale IP
tailscale ip -4
# → 100.x.y.z

# From your local machine, verify SSH over Tailscale works
ssh user@100.x.y.z
```

**Critical**: confirm you can SSH via Tailscale IP before locking down. Otherwise you may lose access.

### 2. Configure UFW

```bash
# Set defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow all traffic on the Tailscale interface
sudo ufw allow in on tailscale0

# Enable UFW
sudo ufw --force enable
```

### 3. Verify

```bash
sudo ufw status verbose
# Should show:
# Default: deny (incoming), allow (outgoing), disabled (routed)
# Anywhere on tailscale0  ALLOW IN  Anywhere
```

### 4. Test access

```bash
# From local machine — this should FAIL (timeout):
ssh user@<public-ip>

# This should SUCCEED:
ssh user@100.x.y.z
# or
ssh user@hostname.tailnet-name.ts.net
```

## Important caveats

### Docker bypasses UFW

Docker manipulates iptables directly, bypassing UFW rules. Published container ports may still be publicly accessible even with UFW configured.

**Mitigations**:
- Bind containers to `127.0.0.1` only: `-p 127.0.0.1:3000:3000`
- Use your cloud provider's firewall (Hetzner Firewall, AWS Security Groups) as the outer layer
- Consider [ufw-docker](https://github.com/chaifeng/ufw-docker) if you need Docker + UFW integration

### Key expiry

Tailscale requires periodic reauthentication by default. If your key expires and you've blocked public SSH, you're locked out.

**Prevent lockout**:
- Disable key expiry for critical servers (admin console → machine page → "Disable key expiry")
- Use your cloud provider's console/VNC access as emergency fallback (Hetzner Console, DigitalOcean Droplet Console)

### IPv6

Check `/etc/default/ufw` has `IPV6=yes`. Otherwise IPv6 traffic bypasses UFW entirely.

## Outbound internet access

UFW `default allow outgoing` means the server can still:
- Download images from the web
- Upload to Backblaze B2
- Pull packages via apt/pip/npm
- Make API calls

Only **incoming** connections from the public internet are blocked.

## Hetzner Firewall (additional layer)

For defense in depth, also configure Hetzner's cloud firewall:
- Allow UDP 41641 inbound (Tailscale direct connections)
- Block everything else inbound
- Allow all outbound

This protects against Docker's iptables bypass since the cloud firewall operates at the network edge.

## Setup script template

```bash
#!/bin/bash
set -euo pipefail

# --- Config ---
TS_AUTHKEY="${TS_AUTHKEY:?Set TS_AUTHKEY}"
TS_HOSTNAME="${TS_HOSTNAME:-worker-$(hostname -s)}"
TS_TAGS="${TS_TAGS:-tag:worker}"

# --- Install Tailscale ---
curl -fsSL https://tailscale.com/install.sh | sh

# --- Join tailnet ---
sudo tailscale up \
  --auth-key="$TS_AUTHKEY" \
  --hostname="$TS_HOSTNAME" \
  --advertise-tags="$TS_TAGS" \
  --ssh

# --- Wait for connection ---
sleep 5
tailscale status

# --- Lock down ---
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw --force enable

echo "Tailscale IP: $(tailscale ip -4)"
echo "DNS name: $(tailscale status --json | jq -r '.Self.DNSName' | sed 's/\.$//')"
echo "Server locked down. SSH only via Tailscale."
```
