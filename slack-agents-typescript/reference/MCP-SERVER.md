# Slack MCP Server

MCP (Model Context Protocol) gives AI agents a standardized way to discover and use external tools. The Slack MCP Server lets agents search channels, send messages, manage canvases, and read user profiles.

## Transport

JSON-RPC 2.0 over Streamable HTTP. Endpoint: `https://mcp.slack.com/mcp`

No SSE or Dynamic Client Registration support.

## Capabilities

| Category | Tools |
|---|---|
| Search | Messages & files (filter by date/user/type), users (partial name match), channels (name/description) |
| Messages | Send to any conversation, draft messages, read channel history, read threads |
| Canvases | Create/update/read (exports as markdown) |
| Users | Full profile info including custom fields and statuses |

## App identity

MCP clients must be backed by a registered Slack app with a fixed app ID. Only directory-published or internal apps may use MCP.

## Authentication

Confidential OAuth. Uses app's `client_id` and `client_secret`.

Discovery endpoints:
- `https://mcp.slack.com/.well-known/oauth-protected-resource`
- `https://mcp.slack.com/.well-known/oauth-authorization-server`

Authorization: `https://slack.com/oauth/v2_user/authorize`
Token: `https://slack.com/api/oauth.v2.user.access`

PKCE supported for desktop clients.

## Required OAuth scopes (user token)

| Action | Scopes |
|---|---|
| Search messages/channels | `search:read.public`, `search:read.private`, `search:read.mpim`, `search:read.im` |
| Search files | `search:read.files` |
| Search users | `search:read.users` |
| Send message | `chat:write` |
| Read channel/thread | `channels:history`, `groups:history`, `mpim:history`, `im:history` |
| Canvas create/update | `canvases:read`, `canvases:write` |
| User profile | `users:read`, `users:read.email` |

## Rate limits

| Tool | Limit |
|---|---|
| Search messages & files | Special — see method docs |
| Search users | Tier 2 (20+/min) |
| Search channels | Tier 2 (20+/min) |
| Send message | Special — see method docs |
| Read channel | Tier 3 (50+/min) |
| Read thread | Tier 3 (50+/min) |
| Create canvas | Tier 2 (20+/min) |
| Update canvas | Tier 3 (50+/min) |
| Read canvas | Tier 3-4 (50-100+/min) |

## Available partner clients

Claude.ai, Claude Code, Perplexity, Cursor — no coding needed.
