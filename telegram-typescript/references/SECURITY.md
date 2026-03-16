# Security Practices

## Bot token safety

The bot token (`123456:ABC-DEF...`) is a **full credential**. Anyone with it controls your bot completely.

- Store in environment variables, never in code
- Use `.env` files locally (add to `.gitignore`)
- Rotate via BotFather `/revoke` if compromised
- In Bun: access via `Bun.env.BOT_TOKEN`

```ts
// .env
BOT_TOKEN=123456789:AAHdqTcvCH1vGWJxfSeofSAs0K5PALDsaw
WEBHOOK_SECRET_TOKEN=my-random-secret-string-32-chars
```

## Webhook authentication

### 1. Secret token (primary)

When setting the webhook, provide a `secret_token`. Telegram includes it in every webhook request as the `X-Telegram-Bot-Api-Secret-Token` header.

```ts
await bot.api.setWebhook("https://bot.example.com/webhook/path", {
  secret_token: Bun.env.WEBHOOK_SECRET_TOKEN,
});
```

grammY's `webhookCallback` validates this automatically if you configure it:

```ts
// grammY handles secret_token verification internally when using webhookCallback
// Just make sure to pass the same secret_token in setWebhook
```

For raw Bun HTTP servers without grammY's callback:

```ts
Bun.serve({
  async fetch(req) {
    const secretHeader = req.headers.get("x-telegram-bot-api-secret-token");
    if (secretHeader !== Bun.env.WEBHOOK_SECRET_TOKEN) {
      return new Response("Forbidden", { status: 403 });
    }
    // Process update...
  },
});
```

**Use constant-time comparison** to prevent timing attacks:

```ts
import { timingSafeEqual } from "crypto";

function safeCompare(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  return timingSafeEqual(Buffer.from(a), Buffer.from(b));
}
```

### 2. Secret path segment (defense in depth)

Include a random path segment in the webhook URL itself:

```ts
const webhookPath = `/webhook/${crypto.randomUUID()}`;
await bot.api.setWebhook(`https://bot.example.com${webhookPath}`, {
  secret_token: Bun.env.WEBHOOK_SECRET_TOKEN,
});
```

This provides two layers: an attacker needs both the correct URL path and the correct header.

### 3. Telegram IP allowlisting

Telegram sends webhooks from these CIDR blocks:

- `149.154.160.0/20`
- `91.108.4.0/22`

Use firewall rules or middleware to restrict:

```ts
import { isIPv4 } from "net";

const TELEGRAM_CIDRS = ["149.154.160.0/20", "91.108.4.0/22"];

function ipInCidr(ip: string, cidr: string): boolean {
  const [range, bits] = cidr.split("/");
  const mask = ~(2 ** (32 - Number(bits)) - 1);
  const ipNum = ip.split(".").reduce((a, o) => (a << 8) + Number(o), 0);
  const rangeNum = range.split(".").reduce((a, o) => (a << 8) + Number(o), 0);
  return (ipNum & mask) === (rangeNum & mask);
}

function isTelegramIP(ip: string): boolean {
  return TELEGRAM_CIDRS.some((cidr) => ipInCidr(ip, cidr));
}
```

**Note**: when behind cloudflared or a reverse proxy, the direct IP will be the proxy's. Use `CF-Connecting-IP` or `X-Forwarded-For` headers instead, and trust them only from your proxy.

## Input validation

### Sanitize user input

Never trust any data from Telegram updates. Users can send arbitrary text, malicious filenames, and crafted callback data.

```ts
bot.on("message:text", (ctx) => {
  const text = ctx.message.text;
  // Always validate/sanitize before using in:
  // - Database queries (use parameterized queries)
  // - Shell commands (never interpolate)
  // - HTML responses (escape)
  // - File paths (reject path traversal)
});
```

### Validate callback data

Callback query data is a string you set, but a modified client could send anything:

```ts
bot.callbackQuery(/^vote:(.+)$/, (ctx) => {
  const choice = ctx.match![1];
  // Validate `choice` is one of your expected values
  if (!["yes", "no"].includes(choice)) {
    return ctx.answerCallbackQuery("Invalid choice");
  }
});
```

## Rate limiting

Use the grammY `ratelimiter` plugin to prevent abuse:

```bash
bun add @grammyjs/ratelimiter
```

```ts
import { limit } from "@grammyjs/ratelimiter";

bot.use(
  limit({
    timeFrame: 2000, // ms
    limit: 3, // max updates per timeFrame per user
    onLimitExceeded: (ctx) => ctx.reply("Slow down!"),
  })
);
```

## Flood control (outgoing)

Telegram limits outgoing API calls (~30 msg/sec to different chats, 1 msg/sec per chat). Use:

```bash
bun add @grammyjs/transformer-throttler
```

```ts
import { apiThrottler } from "@grammyjs/transformer-throttler";

bot.api.config.use(apiThrottler());
```

## Auto-retry on 429

```bash
bun add @grammyjs/auto-retry
```

```ts
import { autoRetry } from "@grammyjs/auto-retry";

bot.api.config.use(autoRetry());
```

## Permissions and privacy

- Use `/setprivacy` via BotFather to control whether your bot receives all messages in groups or only commands
- Request only the `allowed_updates` you need in `setWebhook` to reduce attack surface
- Don't log sensitive user data (phone numbers, locations) unless required
