# ACLs and Policy File

The tailnet policy file controls who can access what. It's written in HuJSON (JSON with comments and trailing commas) and edited in the admin console under **Access Controls**.

## Core constructs

### Tag owners

Tags represent machine identities. Define who can assign them:

```jsonc
{
  "tagOwners": {
    "tag:server": ["alice@example.com"],
    "tag:worker": ["tag:server"],      // tag can own other tags
    "tag:ci":     ["autogroup:admin"]
  }
}
```

### Groups

Group users for easier ACL management:

```jsonc
{
  "groups": {
    "group:eng":   ["alice@example.com", "bob@example.com"],
    "group:admin": ["alice@example.com"]
  }
}
```

### ACLs (legacy style)

```jsonc
{
  "acls": [
    // Engineers can access servers on any port
    {"action": "accept", "src": ["group:eng"], "dst": ["tag:server:*"]},

    // Workers can reach the backend API port
    {"action": "accept", "src": ["tag:worker"], "dst": ["tag:server:4000"]},

    // Everyone can access each other (permissive default)
    {"action": "accept", "src": ["*"], "dst": ["*:*"]}
  ]
}
```

### Grants (newer, recommended)

Grants are more flexible and support services, app capabilities, and per-port access:

```jsonc
{
  "grants": [
    {
      "src": ["group:eng"],
      "dst": ["tag:server"],
      "ip": ["*"]
    },
    {
      "src": ["tag:worker"],
      "dst": ["tag:server"],
      "ip": ["*"],
      "app": {
        "tailscale.com/cap/drive": [{"shares": ["*"]}]
      }
    }
  ]
}
```

### SSH policy

```jsonc
{
  "ssh": [
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["tag:server"],
      "users": ["root", "deploy"]
    },
    {
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:server"],
      "users": ["autogroup:nonroot"]
    }
  ]
}
```

### Funnel policy

```jsonc
{
  "nodeAttrs": [
    {
      "target": ["tag:server"],
      "attr": ["funnel"]
    }
  ]
}
```

### Auto-approvers

Auto-approve subnet routes or exit nodes:

```jsonc
{
  "autoApprovers": {
    "routes": {
      "10.0.0.0/24": ["tag:server"]
    },
    "exitNode": ["tag:server"]
  }
}
```

## Common patterns

### Backend + workers setup

```jsonc
{
  "tagOwners": {
    "tag:backend": ["alice@example.com"],
    "tag:worker":  ["alice@example.com"]
  },
  "acls": [
    // Admin can access everything
    {"action": "accept", "src": ["autogroup:admin"], "dst": ["*:*"]},
    // Workers can reach the backend API
    {"action": "accept", "src": ["tag:worker"], "dst": ["tag:backend:4000"]},
    // Backend can reach workers (for health checks, task dispatch)
    {"action": "accept", "src": ["tag:backend"], "dst": ["tag:worker:*"]}
  ],
  "ssh": [
    {"action": "accept", "src": ["autogroup:admin"], "dst": ["tag:backend", "tag:worker"], "users": ["autogroup:nonroot"]}
  ]
}
```

### Tests

Validate ACLs before deploying:

```jsonc
{
  "tests": [
    {
      "src": "tag:worker",
      "accept": ["tag:backend:4000"],
      "deny": ["tag:backend:22"]
    }
  ]
}
```

## Key autogroups

- `autogroup:admin` — Owners and Admins
- `autogroup:member` — all human users
- `autogroup:nonroot` — SSH as any non-root user
- `autogroup:tagged` — all tagged devices
