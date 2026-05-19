# Governance & Trust

## Stakeholder concerns

**Admins**: clear AI/data usage docs, audit trails via Audit Logs API, policy controls, modular capabilities.

**Compliance**: explicit fallback behaviors, timeouts/retry logic, rate-limit monitoring, traceable sourced responses, consistency testing.

**Developers**: Block Kit for UI, Real-Time Search API for context, Conversations API for threads, assistant API methods for thread management, Slack CLI for scaffolding, Bolt framework for implementation.

**End users**: transparency (status indicators, task cards), control surfaces (buttons, actions, slash commands), contextual UI (threads, App Home, modals).

## Observability metrics

Track per response:

| Key | Description |
|---|---|
| `total_latency_ms` | End-to-end time including tool calls |
| `outcome` | `success` / `partial` / `failure` |
| `user_id` | User in interaction |
| `agent_id` | Which agent/handler produced this |
| `tools_called` | Array of tool names invoked |
| `model` | Model name + version |
| `retry_attempts` | Count of retries |
| `total_tokens` | Input + output combined |
| `token_efficiency` | Output/input ratio (low = over-prompting) |
| `error_type` | `llm_error` / `tool_error` / `validation_error` / `timeout` |

## Human-in-the-loop

### Transparency
- `assistant.threads.setStatus` for visible progress
- Streaming with task cards for orchestration visibility
- Clear agent identity — never masquerade as human

### Control
- App Home for pause/resume/stop controls
- Block Kit actions for confirmations and next steps
- Slash commands for state inspection: `/agent logs`, `/agent state`, `/agent settings`

### Progressive trust

Start with confirmations for every new capability. As user selects "Always allow" for specific action classes, remove friction.

```json
{
  "type": "actions",
  "elements": [
    { "type": "button", "text": { "type": "plain_text", "text": "Always allow" }, "action_id": "always_allow", "style": "primary" },
    { "type": "button", "text": { "type": "plain_text", "text": "Allow once" }, "action_id": "allow_once" },
    { "type": "button", "text": { "type": "plain_text", "text": "Deny" }, "action_id": "deny", "style": "danger" }
  ]
}
```

## Data retention

Do not store Slack data. Store metadata only, pull data in real time when needed.

## Access restrictions

Workspace guests cannot access apps with Agents & AI Apps enabled.

## Security

- Prompt injection is a real risk — see security docs
- Only directory-published or internal apps may use MCP
- Consider marking search scopes as optional to reduce install abandonment
