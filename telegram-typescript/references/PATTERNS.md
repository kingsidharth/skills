# Core Patterns

## Commands

```ts
bot.command("start", (ctx) => ctx.reply("Welcome!"));
bot.command("help", (ctx) => ctx.reply("Commands: /start, /help, /settings"));

// With arguments
bot.command("echo", (ctx) => {
  const text = ctx.match; // everything after "/echo "
  if (!text) return ctx.reply("Usage: /echo <text>");
  return ctx.reply(text);
});
```

Register commands with BotFather for autocomplete, or programmatically:

```ts
await bot.api.setMyCommands([
  { command: "start", description: "Start the bot" },
  { command: "help", description: "Show help" },
  { command: "settings", description: "Open settings" },
]);
```

## Keyboards

### Inline keyboard (buttons under messages)

```ts
import { InlineKeyboard } from "grammy";

const keyboard = new InlineKeyboard()
  .text("Option A", "callback:a")
  .text("Option B", "callback:b")
  .row()
  .url("Visit site", "https://example.com");

bot.command("menu", (ctx) => ctx.reply("Choose:", { reply_markup: keyboard }));

// Handle callback
bot.callbackQuery("callback:a", async (ctx) => {
  await ctx.answerCallbackQuery("You chose A!");
  await ctx.editMessageText("You selected Option A.");
});

// Pattern matching
bot.callbackQuery(/^callback:(.+)$/, (ctx) => {
  const choice = ctx.match![1]; // "a" or "b"
  return ctx.answerCallbackQuery(`Chose: ${choice}`);
});
```

### Custom keyboard (reply keyboard)

```ts
import { Keyboard } from "grammy";

const keyboard = new Keyboard()
  .text("Button 1").text("Button 2").row()
  .text("Button 3")
  .resized()     // fit to content
  .oneTime();    // hide after use

bot.command("keyboard", (ctx) =>
  ctx.reply("Choose:", { reply_markup: keyboard })
);

// Remove keyboard
bot.command("remove", (ctx) =>
  ctx.reply("Keyboard removed", { reply_markup: { remove_keyboard: true } })
);
```

## Middleware

grammY uses a middleware stack (like Express/Koa). Each handler receives `ctx` and `next`.

```ts
// Logging middleware
bot.use(async (ctx, next) => {
  const start = Date.now();
  await next(); // call downstream middleware
  const ms = Date.now() - start;
  console.log(`Update ${ctx.update.update_id} processed in ${ms}ms`);
});

// Auth guard
function adminOnly(ctx, next) {
  if (ctx.from?.id !== Number(Bun.env.ADMIN_ID)) {
    return ctx.reply("Unauthorized");
  }
  return next();
}

bot.command("admin", adminOnly, (ctx) => ctx.reply("Admin panel"));
```

### Composer for modular bots

```ts
import { Composer } from "grammy";

const adminModule = new Composer();
adminModule.command("ban", (ctx) => { /* ... */ });
adminModule.command("stats", (ctx) => { /* ... */ });

const userModule = new Composer();
userModule.command("profile", (ctx) => { /* ... */ });

bot.use(adminModule);
bot.use(userModule);
```

## Sessions

Store per-chat or per-user state across updates:

```bash
bun add # sessions are built into grammy
```

```ts
import { session } from "grammy";
import type { Context, SessionFlavor } from "grammy";

interface SessionData {
  count: number;
  lastSeen: number;
}

type MyContext = Context & SessionFlavor<SessionData>;

const bot = new Bot<MyContext>(Bun.env.BOT_TOKEN!);

bot.use(
  session({
    initial: (): SessionData => ({ count: 0, lastSeen: Date.now() }),
  })
);

bot.command("count", (ctx) => {
  ctx.session.count++;
  return ctx.reply(`Count: ${ctx.session.count}`);
});
```

### External session storage

For production, use a database adapter:

```ts
// Redis example (via @grammyjs/storage-free or custom)
import { freeStorage } from "@grammyjs/storage-free";

bot.use(
  session({
    initial: () => ({ count: 0 }),
    storage: freeStorage<SessionData>(bot.token),
  })
);
```

## Error handling

```ts
// Catch all errors
bot.catch((err) => {
  const ctx = err.ctx;
  console.error(`Error handling update ${ctx.update.update_id}:`);
  console.error(err.error);
});

// Or use error boundary middleware
bot.use(async (ctx, next) => {
  try {
    await next();
  } catch (e) {
    if (e instanceof GrammyError) {
      // Error from Telegram API (e.g., message not found)
      console.error("Telegram API error:", e.description);
    } else if (e instanceof HttpError) {
      // Network error
      console.error("Network error:", e);
    } else {
      throw e; // re-throw unknown errors
    }
  }
});
```

```ts
import { GrammyError, HttpError } from "grammy";
```

## Parse modes (HTML and Markdown)

```ts
// HTML (recommended — more predictable)
await ctx.reply("<b>Bold</b> and <i>italic</i> and <code>code</code>", {
  parse_mode: "HTML",
});

// MarkdownV2 (requires escaping special chars: _ * [ ] ( ) ~ ` > # + - = | { } . !)
await ctx.reply("*Bold* and _italic_", { parse_mode: "MarkdownV2" });
```

### parse-mode plugin (set default)

```bash
bun add @grammyjs/parse-mode
```

```ts
import { hydrateReply, parseMode } from "@grammyjs/parse-mode";
import type { ParseModeFlavor } from "@grammyjs/parse-mode";

type MyContext = ParseModeFlavor<Context>;
const bot = new Bot<MyContext>(Bun.env.BOT_TOKEN!);

bot.use(hydrateReply);
bot.api.config.use(parseMode("HTML")); // default parse mode

bot.command("styled", (ctx) =>
  ctx.replyWithHTML("<b>Bold</b> without specifying parse_mode every time")
);
```

## Conversations (multi-step flows)

```bash
bun add @grammyjs/conversations
```

```ts
import { conversations, createConversation } from "@grammyjs/conversations";
import type { Conversation, ConversationFlavor } from "@grammyjs/conversations";

type MyContext = Context & ConversationFlavor;
type MyConversation = Conversation<MyContext>;

async function registration(conversation: MyConversation, ctx: MyContext) {
  await ctx.reply("What's your name?");
  const nameCtx = await conversation.wait();
  const name = nameCtx.message?.text;

  await ctx.reply("What's your email?");
  const emailCtx = await conversation.wait();
  const email = emailCtx.message?.text;

  await ctx.reply(`Registered: ${name} (${email})`);
}

bot.use(conversations());
bot.use(createConversation(registration));

bot.command("register", async (ctx) => {
  await ctx.conversation.enter("registration");
});
```

## Router plugin

Route updates based on custom logic:

```ts
import { Router } from "@grammyjs/router";

const router = new Router<MyContext>((ctx) => {
  // Return a route key
  return ctx.session.step ?? "idle";
});

router.route("idle", (ctx) => ctx.reply("Send /start to begin"));
router.route("waiting_name", (ctx) => { /* handle name input */ });
router.route("waiting_email", (ctx) => { /* handle email input */ });

bot.use(router);
```

## Graceful shutdown

```ts
const bot = new Bot(Bun.env.BOT_TOKEN!);

// For long polling
process.on("SIGINT", () => bot.stop());
process.on("SIGTERM", () => bot.stop());

bot.start();
```

For webhooks, just let the HTTP server close naturally.

## Project structure (recommended)

```
my-bot/
├── src/
│   ├── bot.ts          # Bot instance + middleware stack
│   ├── commands/       # Command handlers
│   │   ├── start.ts
│   │   └── help.ts
│   ├── handlers/       # Non-command handlers (callbacks, inline, etc.)
│   ├── middleware/      # Auth, logging, rate limiting
│   ├── services/       # Business logic, DB access
│   ├── types.ts        # Context type, session interface
│   └── index.ts        # Entry point (start polling or serve webhooks)
├── .env
├── package.json
└── tsconfig.json
```
