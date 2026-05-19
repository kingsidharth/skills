# Workflow Integration

Add AI capabilities as custom workflow steps in Slack's Workflow Builder.

## Process

1. Create a Slack app with a custom function definition in the app manifest
2. Implement the function logic in Bolt JS
3. Deploy the app
4. Users add the custom step to workflows in Workflow Builder

## Manifest: define custom function

```json
{
  "functions": {
    "code_assist": {
      "title": "Code Assist",
      "description": "Get an answer about a code related question",
      "input_parameters": {
        "message_id": {
          "type": "string",
          "title": "Message ID",
          "description": "The message the question was asked in.",
          "is_required": true
        },
        "channel_id": {
          "type": "slack#/types/channel_id",
          "title": "Channel ID",
          "description": "The channel the question was asked in",
          "is_required": true
        }
      },
      "output_parameters": {
        "message": {
          "type": "string",
          "title": "Answer",
          "description": "The response from the LLM",
          "is_required": true
        }
      }
    }
  }
}
```

## Implementation

```typescript
app.function('code_assist', async ({ client, inputs, complete, fail, logger }) => {
  try {
    const { channel_id, message_id } = inputs;

    const result = await client.conversations.history({
      channel: channel_id,
      oldest: message_id,
      limit: 1,
      inclusive: true,
    });

    const userQuestion = result.messages[0].text;

    // Call your LLM
    const llmResponse = await callLLM(userQuestion);

    await complete({ outputs: { message: llmResponse } });
  } catch (error) {
    logger.error(error);
    fail({ error: `Failed: ${error}` });
  }
});
```

Key criteria:
- Always call `complete()` with outputs or `fail()` with error
- Handle `not_in_channel` errors by joining then retrying
- Convert standard markdown to Slack mrkdwn in outputs
