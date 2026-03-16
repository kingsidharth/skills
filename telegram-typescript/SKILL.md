---
name: telegrambot-typescript
description: Build Telegram bots in TypeScript using grammY framework on Bun runtime. Use this skill whenever the user mentions Telegram bots, Telegram Bot API, grammY, chat bots on Telegram, webhook-based bots, Telegram file handling, inline keyboards, or wants to create any kind of Telegram bot. Also trigger for questions about Telegram Bot API types, @grammyjs/types, @gramio/types, Telegram webhook security, cloudflared tunnels for bots, or Bun-based bot deployments.
---

# Telegram Bot (TypeScript)

Build Telegram bots with **grammY** framework on **Bun** runtime. grammY is the most popular TypeScript Telegram bot framework — type-safe, plugin-rich, and Bun-native.

## Quick start

```bash
mkdir my-bot && cd my-bot
bun init -y
bun add grammy
```

```ts
import { Bot } from "grammy";

const bot = new Bot(Bun.env.BOT_TOKEN!);

bot.command("start", (ctx) => ctx.reply("Hello!"));
bot.on("message:text", (ctx) => ctx.reply(`Echo: ${ctx.message.text}`));

bot.start();
```

```bash
BOT_TOKEN=123456:ABC-DEF bun run bot.ts
```

## Architecture decision: grammY vs alternatives

| Framework | Types | Bun | Plugins | Maintained |
|-----------|-------|-----|---------|------------|
| **grammY** | Excellent (built-in) | ✅ Native | Rich ecosystem | ✅ Active |
| GramIO | Auto-generated | ✅ | Growing | ✅ Active |
| Telegraf | Legacy (@telegraf/types) | Partial | Mature | Slower |

**Use grammY** for all new TypeScript bots. It has the best type inference, widest plugin ecosystem, and first-class Bun support.

## Types landscape

grammY bundles `@grammyjs/types` — handwritten, complete Bot API types with JSDoc. These are automatically available when you import from `grammy`.

For **framework-agnostic** type usage (e.g., raw API wrappers), use `@gramio/types` — auto-generated from the Telegram Bot API spec on every release:

```ts
import type { APIMethods, APIMethodParams, TelegramMessage } from "@gramio/types";
```

**You do NOT need custom types.** Both packages track the Bot API spec closely and are updated within days of Telegram releases. See [TYPES.md](references/TYPES.md) for details.

## Receiving updates: long polling vs webhooks

| | Long polling | Webhooks |
|---|---|---|
| **Use for** | Dev, VPS, always-on servers | Serverless, edge, production tunnels |
| **Setup** | `bot.start()` | HTTP server + `webhookCallback()` |
| **Latency** | ~30s timeout cycle | Instant push from Telegram |
| **Scaling** | Single process controls rate | Telegram pushes; you must respond fast |

For production behind **cloudflared** or any reverse proxy, use webhooks. See [WEBHOOKS.md](references/WEBHOOKS.md).

## Key topics

- **Webhook setup with Bun + cloudflared**: See [WEBHOOKS.md](references/WEBHOOKS.md)
- **Security practices** (secret_token, IP allowlisting, token safety): See [SECURITY.md](references/SECURITY.md)
- **Files and message attachments** (upload, download, file_id reuse, limits): See [FILES.md](references/FILES.md)
- **Bot API types** (@grammyjs/types vs @gramio/types, type utilities): See [TYPES.md](references/TYPES.md)
- **Core patterns** (commands, keyboards, middleware, sessions, error handling): See [PATTERNS.md](references/PATTERNS.md)
