# Bot API Types

## Type packages overview

| Package | Source | Generation | Use case |
|---------|--------|-----------|----------|
| `@grammyjs/types` | Bundled with `grammy` | Handwritten, JSDoc-rich | Default with grammY |
| `@gramio/types` | Standalone (`bun add @gramio/types`) | Auto-generated from spec | Framework-agnostic raw API |
| `@telegraf/types` | Fork of typegram | Handwritten | Legacy, Telegraf only |

**Recommendation**: use grammY's built-in types (they come from `@grammyjs/types`). Only use `@gramio/types` if building a custom wrapper without grammY.

## You don't need custom types

Both `@grammyjs/types` and `@gramio/types` are updated within days of new Bot API releases (currently at Bot API 9.5). They cover 100% of the API surface. No custom type definitions are needed.

## Using types from grammY

All Bot API types are re-exported from `grammy`:

```ts
import type { Message, Update, Chat, User, PhotoSize } from "grammy/types";
```

Or access them through the context:

```ts
bot.on("message", (ctx) => {
  // ctx.message is fully typed as Message
  const chat: Chat = ctx.chat;
  const from: User | undefined = ctx.from;
});
```

### Type narrowing with filter queries

grammY's filter queries automatically narrow types:

```ts
bot.on("message:text", (ctx) => {
  // ctx.message.text is `string` (guaranteed present)
  const text: string = ctx.message.text;
});

bot.on("message:photo", (ctx) => {
  // ctx.message.photo is `PhotoSize[]` (guaranteed present)
  const photos: PhotoSize[] = ctx.message.photo;
});

bot.on("callback_query:data", (ctx) => {
  // ctx.callbackQuery.data is `string` (guaranteed present)
  const data: string = ctx.callbackQuery.data;
});
```

### Update variant types

The `Update` type is a large union. Access specific variants:

```ts
import type { Update } from "grammy/types";

type MessageUpdate = Update.MessageUpdate;
type CallbackQueryUpdate = Update.CallbackQueryUpdate;
type InlineQueryUpdate = Update.InlineQueryUpdate;
```

### Method parameter and return types

```ts
import type { Other } from "grammy";

// `Other<"sendMessage">` gives you the optional params for sendMessage
// (excludes chat_id and text which are positional)
const opts: Other<"sendMessage"> = {
  parse_mode: "HTML",
  reply_markup: { inline_keyboard: [[]] },
};
```

## Using @gramio/types standalone

For building raw API wrappers without grammY:

```bash
bun add @gramio/types
```

```ts
import type {
  APIMethods,
  APIMethodParams,
  APIMethodReturn,
  TelegramMessage,
  TelegramUpdate,
  TelegramUser,
  TelegramChat,
  TelegramAPIResponse,
} from "@gramio/types";

// Type-safe API call wrapper
async function callApi<M extends keyof APIMethods>(
  token: string,
  method: M,
  params: APIMethodParams<M>
): Promise<APIMethodReturn<M>> {
  const res = await fetch(`https://api.telegram.org/bot${token}/${method}`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(params),
  });
  const data = (await res.json()) as TelegramAPIResponse;
  if (!data.ok) throw new Error(`API error in ${method}: ${data.description}`);
  return data.result as APIMethodReturn<M>;
}

// Usage — fully typed
const me = await callApi(token, "getMe", {});
// ^? TelegramUser

const msg = await callApi(token, "sendMessage", { chat_id: 123, text: "hi" });
// ^? TelegramMessage
```

### @gramio/types naming convention

All object types are prefixed with `Telegram`:

- `TelegramMessage`, `TelegramUser`, `TelegramChat`, `TelegramUpdate`
- `TelegramPhotoSize`, `TelegramDocument`, `TelegramInlineKeyboardMarkup`

Method params use `Params` suffix: `SendMessageParams`, `GetFileParams`, etc.

### Subpath imports

```ts
import type { TelegramMessage } from "@gramio/types/objects";
import type { SendMessageParams } from "@gramio/types/params";
import type { APIMethods } from "@gramio/types/methods";
```

## Declaration merging for custom Bot API servers

If using a local Bot API server with custom methods:

```ts
declare module "@gramio/types" {
  export interface APIMethods {
    myCustomMethod: (params: { chat_id: string }) => Promise<boolean>;
  }
}
```

## InputFile type customization

Both type packages parameterize `InputFile` to let frameworks define their own upload types. grammY uses this to support `Buffer`, `ReadableStream`, file paths, and URLs in its `InputFile` class. When using raw types, you define your own:

```ts
import type * as Telegram from "@grammyjs/types";

type MyInputFile = string | Buffer | ReadableStream;
type MyAPIMethods = Telegram.ApiMethods<MyInputFile>;
```
