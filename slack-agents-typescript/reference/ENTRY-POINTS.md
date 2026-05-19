# Entry Points & Interaction Surfaces

## Entry points

### Agent container (split pane)

Requires **Agents & AI Apps** enabled in app settings. The `Assistant` class wraps three events:

- `threadStarted` → `assistant_thread_started`: user opens container. Set suggested prompts, send welcome.
- `threadContextChanged` → `assistant_thread_context_changed`: user navigates to different channel while container open. Update grounding context.
- `userMessage` → `message.im`: user sends message. Core response loop.

```typescript
const assistant = new Assistant({
  threadStarted: async ({ say, setSuggestedPrompts, saveThreadContext }) => {
    await say({ text: 'Hi! How can I help?' });
    await saveThreadContext();
    await setSuggestedPrompts({
      prompts: [
        { title: 'Summarize channel', message: 'Summarize the last week of #general' },
        { title: 'Draft a message', message: 'Help me write a project update' },
      ],
    });
  },
  threadContextChanged: async ({ saveThreadContext }) => {
    await saveThreadContext();
  },
  userMessage: async ({ message, say, setStatus }) => {
    await setStatus({ status: 'Thinking...' });
    await say({ text: 'Here is your answer.' });
  },
});
app.use(assistant);
```

Dynamic prompts based on context (channel, user profile, connected data) are preferred over static prompts — keeps repeat usage feeling fresh.

### Channel @mentions

`app_mention` event. Always reply in-thread using `thread_ts ?? event.ts`:

```typescript
app.event('app_mention', async ({ event, client }) => {
  const threadTs = event.thread_ts ?? event.ts;
  await client.chat.postMessage({
    channel: event.channel,
    thread_ts: threadTs,
    text: 'On it!',
  });
});
```

### Direct messages

`message.im` event. Filter `channel_type: 'im'` to distinguish from group DMs:

```typescript
app.message(async ({ message, say }) => {
  if (message.channel_type !== 'im') return;
  await say({ text: 'Got your message!' });
});
```

## Interaction surfaces

### App Home

Persistent surface for workflow visibility, settings, recovery paths. Publish on `app_home_opened`:

```typescript
app.event('app_home_opened', async ({ event, client }) => {
  await client.views.publish({
    user_id: event.user,
    view: { type: 'home', blocks: [/* Block Kit blocks */] },
  });
});
```

Enable **Home Tab** in app settings under **App Home** → **Features**.

### Modals

Temporary overlays for structured input. Open with `views.open`, handle with `app.view()`:

```typescript
await client.views.open({
  trigger_id,
  view: {
    type: 'modal',
    callback_id: 'my_modal',
    title: { type: 'plain_text', text: 'Title' },
    submit: { type: 'plain_text', text: 'Submit' },
    blocks: [],
  },
});

app.view('my_modal', async ({ view, ack }) => {
  await ack();
  const values = view.state.values;
});
```

Pre-fill with `initial_value` / `initial_options` to reduce user effort.

### Slash commands

Text-invoked actions with inline arguments. Not supported in the split-pane container (threads don't support slash commands). Always `ack()` immediately:

```typescript
app.command('/myapp', async ({ command, ack, client }) => {
  await ack();
  const [subcommand] = command.text.trim().split(/\s+/);
  // route by subcommand
});
```

### Message shortcuts

Triggered from message context menu. Receives full message payload — ideal for "act on this message" (summarize, create ticket, draft reply):

```typescript
app.shortcut({ callback_id: 'summarize', type: 'message_action' },
  async ({ shortcut, ack, client }) => {
    await ack();
    await client.views.open({ trigger_id: shortcut.trigger_id, view: {/* modal */} });
  }
);
```

### Global shortcuts

From compose box `+` button. Quick-launch for agent workflows not tied to a message:

```typescript
app.shortcut({ callback_id: 'new_task', type: 'shortcut' },
  async ({ shortcut, ack, client }) => {
    await ack();
    await client.views.open({ trigger_id: shortcut.trigger_id, view: {/* modal */} });
  }
);
```

### Unfurls

Rich link previews. Subscribe to `link_shared`, call `chat.unfurl`:

```typescript
app.event('link_shared', async ({ event, client }) => {
  await client.chat.unfurl({
    channel: event.channel,
    ts: event.message_ts,
    unfurls: { [event.links[0].url]: { blocks: [] } },
  });
});
```

### Ephemeral messages

Temp messages visible only to one user. Use for acknowledgments, inline errors, confirmations. Channel/user source varies by handler context:

```typescript
await client.chat.postEphemeral({
  channel: event.channel,   // or command.channel_id, body.channel.id
  user: event.user,         // or command.user_id, body.user.id
  text: 'Working on it...',
});
```
