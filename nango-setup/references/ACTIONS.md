# Nango Actions

## Overview

Actions are one-off Functions triggered on demand. Use them for write operations (send a message, create an issue, update a record) or real-time reads that don't need caching.

## Action file scaffold

```typescript
import { createAction } from 'nango';
import * as z from 'zod';

const SendMessageInput = z.object({
  channel: z.string(),
  text: z.string(),
});

const SendMessageOutput = z.object({
  ok: z.boolean(),
  ts: z.string(),
  channel: z.string(),
});

export default createAction({
  description: 'Send a message to a Slack channel',
  version: '1.0.0',
  endpoint: { method: 'POST', path: '/slack/send-message', group: 'Messages' },
  input: SendMessageInput,
  output: SendMessageOutput,

  exec: async (nango, input) => {
    const response = await nango.post({
      endpoint: '/api/chat.postMessage',
      data: {
        channel: input.channel,
        text: input.text,
      },
    });

    return {
      ok: response.data.ok,
      ts: response.data.ts,
      channel: response.data.channel,
    };
  },
});
```

## File structure

```
nango-integrations/
├── index.ts
├── slack/
│   ├── syncs/
│   │   └── slack-messages.ts
│   └── actions/
│       └── slack-send-message.ts    # actions go in actions/ folder
```

Register in `index.ts`:
```typescript
import './slack/actions/slack-send-message';
```

## Triggering actions (from your app)

```typescript
// Node SDK
const result = await nango.triggerAction(
  'slack',                    // providerConfigKey
  'user-123',                 // connectionId
  'slack-send-message',       // action name
  { channel: 'C01234567', text: 'Hello!' }  // input
);

console.log(result); // { ok: true, ts: '1234567890.123456', channel: 'C01234567' }
```

```bash
# cURL
curl -X POST \
  -H "Authorization: Bearer $NANGO_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{"channel":"C01234567","text":"Hello!"}' \
  "https://api.nango.dev/action/trigger?provider_config_key=slack&connection_id=user-123&action_name=slack-send-message"
```

## Actions vs Proxy

| | Actions | Proxy |
|---|---|---|
| Code runs | On Nango's infra | On your infra |
| Logic | Custom TypeScript with validation | Raw HTTP pass-through |
| Input/output schema | Zod-validated | Unstructured |
| Logging | Built-in Nango logs | Your own logging |
| Best for | Complex write operations, multi-step flows | Simple single API calls |

Use Proxy for simple pass-through calls. Use Actions when you need input validation, multi-step logic, or want operations logged in Nango's dashboard.
