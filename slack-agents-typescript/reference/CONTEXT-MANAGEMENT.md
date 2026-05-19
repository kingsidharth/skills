# Context Management

Pull targeted workspace data, store as structured state, pass to LLM each turn. Avoid re-querying the same data on every exchange.

## Best practices

- **Don't refetch entire threads** — use structured state between turns
- **Progressive summarization** — summarize older material, preserve key decisions
- **Token budgets** — prefer small relevant slices over raw conversational exhaust
- **Drift detection** — confirm goal/constraints before significant actions

## Workspace search

`assistant.search.context` — semantic cross-workspace search across messages, files, channels, canvases. Requires `action_token` from the triggering event payload when using bot token.

```typescript
const result = await client.assistant.search.context({
  query: 'What are the latest decisions on project alpha?',
  action_token: event.action_token,
  content_types: ['messages', 'files', 'channels'],
  channel_types: ['public_channel', 'private_channel'],
  include_context_messages: true,
  limit: 20,
});
const matches = result.results?.messages || [];
// Each match: content, permalink, channel_id, message_ts, context_messages
```

Do NOT use the legacy `search.messages` endpoint.

## Thread context

Drill into a specific thread from search results:

```typescript
const result = await client.conversations.replies({
  channel: match.channel_id,
  ts: match.message_ts,
  limit: 100,
});
// messages[0] = parent, rest = replies
```

## Channel context

Pull surrounding messages from a channel:

```typescript
const result = await client.conversations.history({
  channel: channelId,
  oldest: String((Date.now() / 1000) - 7 * 24 * 60 * 60),
  limit: 100,
});
// Paginate: result.has_more → result.response_metadata.next_cursor
```

## Structured state object

Persist between turns instead of re-injecting raw threads:

```typescript
interface AgentState {
  goal: string;        // user's current objective
  constraints: string; // date range, channel scope, filters
  decisions: string[]; // key decisions this session
  artifacts: Array<{ type: string; text: string }>;  // outputs created
  sources: Array<{ text: string; link: string }>;    // attribution
}
```

On follow-up turns, pass existing state to LLM and update fields — don't re-call `assistant.search.context` unless the goal changed.
