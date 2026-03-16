# Files and Message Attachments

## How files work in Telegram

Files on Telegram servers are identified by `file_id` — a long opaque string. Bots receive `file_id` values (not raw file data) in incoming messages.

Key rules:

- `file_id` is **bot-specific** — cannot be shared between different bots
- `file_unique_id` is stable across bots — use for deduplication, but cannot download with it
- A file can have multiple `file_id` values over time; compare using `file_unique_id`
- Reuse `file_id` to re-send the same file without re-uploading (very efficient)

## Size limits

| Operation | Standard Bot API | Local Bot API Server |
|-----------|-----------------|---------------------|
| Download (getFile) | 20 MB | Unlimited |
| Upload | 50 MB | 2,000 MB |
| Photo via URL | 5 MB | 5 MB |

For files >50 MB, you must run a [local Bot API server](https://github.com/tdlib/telegram-bot-api).

## Receiving files

### Listen for specific file types

```ts
// Photos
bot.on("message:photo", async (ctx) => {
  // photos come as array of PhotoSize (different resolutions)
  const photo = ctx.message.photo;
  const largest = photo[photo.length - 1]; // last = highest resolution
  console.log(`Photo: ${largest.file_id}, ${largest.width}x${largest.height}`);
});

// Documents (any file)
bot.on("message:document", async (ctx) => {
  const doc = ctx.message.document;
  console.log(`${doc.file_name} (${doc.mime_type}, ${doc.file_size} bytes)`);
});

// Voice messages
bot.on("message:voice", (ctx) => {
  const voice = ctx.message.voice;
  console.log(`Voice: ${voice.duration}s, ${voice.file_size} bytes`);
});

// Video
bot.on("message:video", (ctx) => { /* ctx.message.video */ });

// Audio
bot.on("message:audio", (ctx) => { /* ctx.message.audio */ });

// Sticker
bot.on("message:sticker", (ctx) => { /* ctx.message.sticker */ });

// Animation (GIF)
bot.on("message:animation", (ctx) => { /* ctx.message.animation */ });

// Video note (round video)
bot.on("message:video_note", (ctx) => { /* ctx.message.video_note */ });
```

### Catch-all for any file

```ts
// :media = photo | video | animation | audio | document | voice | video_note | sticker
bot.on("message:media", (ctx) => { /* any media */ });

// :file = same set (alias)
bot.on("message:file", (ctx) => { /* any file */ });
```

### Download a file

```ts
bot.on("message:document", async (ctx) => {
  // getFile returns File object with file_path
  const file = await ctx.getFile(); // auto-uses the file from current message
  
  // Construct download URL
  const url = `https://api.telegram.org/file/bot${Bun.env.BOT_TOKEN}/${file.file_path}`;
  
  // Download with fetch
  const response = await fetch(url);
  const buffer = await response.arrayBuffer();
  await Bun.write(`./downloads/${ctx.message.document.file_name}`, buffer);
});
```

### Using the files plugin (convenience)

```bash
bun add @grammyjs/files
```

```ts
import { hydrateFiles } from "@grammyjs/files";

bot.api.config.use(hydrateFiles(Bun.env.BOT_TOKEN!));

bot.on("message:document", async (ctx) => {
  const file = await ctx.getFile();
  
  // Direct download URL
  const url = file.getUrl(); // https://api.telegram.org/file/bot.../...
  
  // Or download directly
  const path = await file.download(); // saves to temp file, returns path
  
  // Or download to specific path
  await file.download("./my-file.pdf");
});
```

## Sending files

Three methods to send files:

### 1. By file_id (most efficient — no re-upload)

```ts
await ctx.replyWithPhoto(existingFileId);
await ctx.replyWithDocument(existingFileId);
```

### 2. By URL (Telegram downloads it)

```ts
await ctx.replyWithPhoto("https://example.com/image.jpg");
```

### 3. By upload (InputFile)

```ts
import { InputFile } from "grammy";

// From local path
await ctx.replyWithDocument(new InputFile("/path/to/file.pdf"));

// From Buffer / Uint8Array
const data = new Uint8Array([0x50, 0x4b]); // example bytes
await ctx.replyWithDocument(new InputFile(data, "archive.zip"));

// From URL (streams through your server — use sparingly)
await ctx.replyWithPhoto(new InputFile(new URL("https://example.com/img.png")));

// From ReadableStream
const stream = Bun.file("./large-video.mp4").stream();
await ctx.replyWithVideo(new InputFile(stream, "video.mp4"));
```

### With caption and parse mode

```ts
await ctx.replyWithPhoto(new InputFile("./chart.png"), {
  caption: "Here's your <b>weekly report</b>",
  parse_mode: "HTML",
});
```

### Send media group (album)

```ts
import { InputMediaBuilder } from "grammy";

await ctx.replyWithMediaGroup([
  InputMediaBuilder.photo(new InputFile("./photo1.jpg"), { caption: "First" }),
  InputMediaBuilder.photo(new InputFile("./photo2.jpg")),
  InputMediaBuilder.photo(existingFileId),
]);
```

## File type methods

| Type | Send method | Filter query |
|------|------------|--------------|
| Photo | `replyWithPhoto` | `message:photo` |
| Document | `replyWithDocument` | `message:document` |
| Video | `replyWithVideo` | `message:video` |
| Audio | `replyWithAudio` | `message:audio` |
| Voice | `replyWithVoice` | `message:voice` |
| Video note | `replyWithVideoNote` | `message:video_note` |
| Animation | `replyWithAnimation` | `message:animation` |
| Sticker | `replyWithSticker` | `message:sticker` |

## Storing file references

Always store `file_id` and `file_unique_id` together:

```ts
interface StoredFile {
  fileId: string;       // for re-sending
  fileUniqueId: string; // for deduplication
  fileName?: string;
  mimeType?: string;
  fileSize?: number;
}
```

`file_id` can change over time for the same file. `file_unique_id` stays stable — use it as the DB unique key.
