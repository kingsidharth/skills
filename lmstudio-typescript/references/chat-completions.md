# Chat Completions

## Core pattern

```ts
import { LMStudioClient } from "@lmstudio/sdk";
const client = new LMStudioClient();
const model = await client.llm.model();
```

## Streaming

```ts
for await (const fragment of model.respond("What is life?")) {
  process.stdout.write(fragment.content);
}
```

## Non-streaming

```ts
const result = await model.respond("What is life?").result();
console.info(result.content);
```

## Chat context via `Chat` object

```ts
import { Chat } from "@lmstudio/sdk";

const chat = Chat.from([
  { role: "system", content: "You are a philosopher." },
  { role: "user", content: "What is life?" },
]);

const prediction = model.respond(chat);
```

Or construct incrementally:

```ts
const chat = Chat.empty();
chat.append("user", "Hello");
```

## Multi-turn loop

```ts
const chat = Chat.empty();
while (true) {
  const input = await rl.question("You: ");
  chat.append("user", input);
  const prediction = model.respond(chat, {
    onMessage: (message) => chat.append(message),
  });
  for await (const { content } of prediction) {
    process.stdout.write(content);
  }
}
```

## Prediction stats

```ts
const result = await prediction.result();
result.modelInfo.displayName;
result.stats.predictedTokensCount;
result.stats.timeToFirstTokenSec;
result.stats.stopReason;
```
