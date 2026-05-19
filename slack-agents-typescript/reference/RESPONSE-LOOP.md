# Response Loop

The agent response cycle: receive input → set status → reason/call tools → stream output → collect feedback.

## Status indicator

Call `assistant.threads.setStatus` immediately when processing begins. Pass empty string to clear.

```typescript
// Bolt utility
await setStatus({
  status: 'Thinking...',
  loading_messages: [
    'Untangling the internet cables…',
    'Consulting the office goldfish…',
  ],
});
```

Raw API call shape:
```json
{
  "status": "is working on your request...",
  "channel_id": "D324567865",
  "thread_ts": "1724264405.531769"
}
```

## Text streaming

Three API methods: `chat.startStream`, `chat.appendStream`, `chat.stopStream`. Shows LLM output token-by-token instead of as a single block.

Caveats: Block Kit elements only allowed in `stopStream` (not start/append). Unfurling disabled in streaming messages.

### Bolt JS utility

```typescript
const assistant = new Assistant({
  userMessage: async ({ message, sayStream, setTitle, setStatus }) => {
    await setTitle(message.text);
    await setStatus({ status: 'Thinking...' });

    const stream = sayStream();
    await stream.append({ markdown_text: "Here's my response..." });
    await stream.append({ markdown_text: "And here's more..." });
    await stream.stop(); // can include blocks here
  },
});
```

### Raw API streaming

```typescript
// 1. Start
const stream = await client.chat.startStream({
  channel,
  thread_ts: threadTs,
  task_display_mode: 'plan', // or 'timeline'
  chunks: [{ type: 'markdown_text', markdown_text: 'Let me help!' }],
});

// 2. Append (progressive)
await client.chat.appendStream({
  channel,
  message_ts: stream.ts,
  thread_ts: threadTs,
  chunks: [{ type: 'markdown_text', markdown_text: 'Found results...' }],
});

// 3. Stop (can include blocks)
await client.chat.stopStream({
  channel,
  message_ts: stream.ts,
  thread_ts: threadTs,
  chunks: [{ type: 'markdown_text', markdown_text: 'Done!' }],
});
```

## Task display modes

### Task cards

Individual steps shown via `task_update` chunks in `appendStream`. States: `in_progress`, `completed`, `error`.

```json
{
  "type": "task_update",
  "task": {
    "task_id": "task_1",
    "title": "Fetching weather data",
    "status": "complete",
    "output": {
      "type": "rich_text",
      "elements": [{ "type": "rich_text_section", "elements": [{ "type": "text", "text": "Found data" }] }]
    },
    "sources": [{ "type": "url", "url": "https://example.com", "text": "Source" }]
  }
}
```

### Plan display

Groups tasks together using the plan block. States: `pending`, `in_progress`, `completed`, `error`. Set `task_display_mode: 'plan'` in `startStream`.

## Thread titles

Set via `assistant.threads.setTitle` or Bolt's `setTitle` utility. Update title to match conversation content — improves History tab browsability.

```typescript
await setTitle(message.text); // typically set on first user message
```

## Feedback

Use `context_actions` block with `feedback_buttons` element:

```json
{
  "type": "context_actions",
  "elements": [{
    "type": "feedback_buttons",
    "action_id": "feedback",
    "positive_button": { "text": { "type": "plain_text", "text": "👍" }, "value": "good" },
    "negative_button": { "text": { "type": "plain_text", "text": "👎" }, "value": "bad" }
  }]
}
```

Handle via `block_actions` payload. Consider opening a modal on negative feedback to collect details.

Also available: `icon_button` (e.g., trash icon for delete), `reaction_added` events.

## Messaging guidelines

### Formatting

- Use Slack `mrkdwn` (not standard markdown) for plain-text messages — syntax differs
- Or use the **Markdown Block** (`type: 'markdown'`) which accepts standard markdown and translates correctly
- Set section block's `expand: true` to avoid "see more" truncation on long messages
- Rate limit: call `chat.update` at most once per 3 seconds

### Content disclaimers

Add via `context` block:
```json
{ "type": "context", "elements": [{ "type": "mrkdwn", "text": "This tool uses AI. Some information may be inaccurate." }] }
```

### Citations

Use link formatting inline, list references in a `context` block at message end. Suppress unfurls when citing multiple sources.

### Notifications

With Agents & AI Apps enabled, every DM is a thread. To notify a user:
1. `chat.postMessage` → capture `ts` from response
2. `assistant.threads.setTitle` with that `ts` as `thread_ts`

Notifications appear in the **History** tab and the **Activity** side rail.

### Error handling

On failure: save progress, explain where stuck, offer options (provide info / skip step / take over manually). As last resort, clear status so app isn't stuck "thinking" forever.
