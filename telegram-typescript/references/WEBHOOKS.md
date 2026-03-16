# Webhooks (Bun + Cloudflared)

## Bun HTTP server with grammY

grammY provides `webhookCallback` with adapters for many frameworks. For Bun's native HTTP server:

```ts
import { Bot, webhookCallback } from "grammy";

const bot = new Bot(Bun.env.BOT_TOKEN!);

bot.command("start", (ctx) => ctx.reply("Hello from webhook!"));
bot.on("message:text", (ctx) => ctx.reply(`Echo: ${ctx.message.text}`));

const handleUpdate = webhookCallback(bot, "std/http");

Bun.serve({
  port: Number(Bun.env.PORT ?? 8443),
  async fetch(req) {
    const url = new URL(req.url);
    if (req.method === "POST" && url.pathname === `/webhook/${Bun.env.WEBHOOK_SECRET}`) {
      return handleUpdate(req);
    }
    return new Response("OK", { status: 200 });
  },
});
```

### Available webhook adapters

The second argument to `webhookCallback()`:

| Adapter | Use with |
|---------|----------|
| `"std/http"` | `Bun.serve`, `Deno.serve` |
| `"bun"` | `Bun.serve` (alternative) |
| `"express"` | Express.js |
| `"hono"` | Hono |
| `"fastify"` | Fastify |
| `"koa"` | Koa |
| `"next-js"` | Next.js API routes |
| `"cloudflare"` | Cloudflare Workers |

### Timeout behavior

`webhookCallback` has a default 10-second timeout. If middleware doesn't finish in time:

- Default (`"throw"`): throws an error — you see it in logs
- `"return"`: ends request early, **dangerous** — causes race conditions with sessions

```ts
// Third arg controls timeout behavior (default: "throw")
const handleUpdate = webhookCallback(bot, "std/http", "throw");
```

**Rule**: keep middleware fast. Offload heavy work (file processing, AI calls) to a queue.

## Setting the webhook

```ts
// At startup or via a setup script:
await bot.api.setWebhook(`https://your-domain.com/webhook/${Bun.env.WEBHOOK_SECRET}`, {
  secret_token: Bun.env.WEBHOOK_SECRET_TOKEN, // verified via header
  allowed_updates: ["message", "callback_query", "inline_query"],
  max_connections: 40,
});
```

To check current webhook status:

```ts
const info = await bot.api.getWebhookInfo();
console.log(info); // url, pending_update_count, last_error_date, etc.
```

To remove webhook (switch back to polling):

```ts
await bot.api.deleteWebhook();
```

## Cloudflared tunnel setup

Cloudflared creates a secure tunnel from Telegram's servers to your local/VPS machine without opening ports or managing SSL certificates.

### Development (quick tunnel)

```bash
# No config needed — generates a random *.trycloudflare.com URL
cloudflared tunnel --url http://localhost:8443
```

Copy the `https://*.trycloudflare.com` URL and set it as your webhook:

```bash
curl "https://api.telegram.org/bot$BOT_TOKEN/setWebhook?url=https://RANDOM.trycloudflare.com/webhook/$WEBHOOK_SECRET"
```

Quick tunnels are ephemeral — URL changes on restart. Fine for dev, not for production.

### Production (named tunnel)

```bash
# One-time: authenticate and create tunnel
cloudflared tunnel login
cloudflared tunnel create my-bot-tunnel
cloudflared tunnel route dns my-bot-tunnel bot.yourdomain.com
```

Config file (`~/.cloudflared/config.yml`):

```yaml
tunnel: <TUNNEL_UUID>
credentials-file: /root/.cloudflared/<TUNNEL_UUID>.json

ingress:
  - hostname: bot.yourdomain.com
    service: http://localhost:8443
  - service: http_status:404
```

Run:

```bash
cloudflared tunnel run my-bot-tunnel
```

Then set webhook to `https://bot.yourdomain.com/webhook/<secret>`.

### Systemd service for cloudflared

```ini
# /etc/systemd/system/cloudflared-bot.service
[Unit]
Description=Cloudflare Tunnel for Telegram Bot
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/cloudflared tunnel run my-bot-tunnel
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now cloudflared-bot
```

## Webhook reply optimization

You can piggyback one API call on the webhook response, saving an HTTP round-trip:

```ts
const bot = new Bot(Bun.env.BOT_TOKEN!, {
  client: {
    // Only use for fire-and-forget methods (no response needed)
    canUseWebhookReply: (method) => method === "sendChatAction",
  },
});
```

**Drawbacks**: no error handling on that call, no response object, no abort support. Use sparingly.

## Health check endpoint

Always expose a health endpoint for monitoring:

```ts
Bun.serve({
  port: 8443,
  async fetch(req) {
    const url = new URL(req.url);
    if (url.pathname === "/health") {
      return Response.json({ status: "ok", timestamp: Date.now() });
    }
    if (req.method === "POST" && url.pathname === `/webhook/${Bun.env.WEBHOOK_SECRET}`) {
      return handleUpdate(req);
    }
    return new Response("Not Found", { status: 404 });
  },
});
```
