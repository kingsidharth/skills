---
name: slack-agents-typescript
description: Build AI agents for Slack using Bolt for JavaScript/TypeScript. Use when building Slack bots with AI capabilities, assistant containers, agent streaming, suggested prompts, feedback buttons, App Home, modals, slash commands, @mentions, context management, or the Slack MCP Server. Also trigger for Slack CLI agent scaffolding, text streaming APIs, task cards, plan display, workspace search via assistant.search.context, or deploying Slack agents. Even if the user just says 'Slack bot with AI' or 'chat agent in Slack', use this skill.
---

# Slack Agents (TypeScript / Bolt JS)

Build AI-powered agents that live natively inside Slack: split-pane containers, streamed responses, suggested prompts, feedback loops, workspace search, and multi-surface interactions.

## Quick orientation

Slack agents are Bolt JS apps with the **Agents & AI Apps** feature enabled. The `Assistant` class provides a three-callback lifecycle (thread started → context changed → user message) that powers the split-pane container experience. Outside the container, agents respond to `app_mention`, `message.im`, slash commands, shortcuts, and App Home.

The response loop: receive input → set status → call tools/LLM → stream output → collect feedback → repeat.

## When to read reference files

| Topic | File | Read when… |
|---|---|---|
| Assistant lifecycle, entry points, surfaces | [ENTRY-POINTS.md](references/ENTRY-POINTS.md) | Wiring up the container, @mentions, DMs, App Home, modals, slash commands, shortcuts, unfurls |
| Response loop: streaming, status, feedback, threads | [RESPONSE-LOOP.md](references/RESPONSE-LOOP.md) | Implementing `setStatus`, `startStream`/`appendStream`/`stopStream`, task cards, plan display, feedback buttons, thread titles, markdown blocks, notifications |
| Context management & workspace search | [CONTEXT-MANAGEMENT.md](references/CONTEXT-MANAGEMENT.md) | Using `assistant.search.context`, `conversations.replies`, structured state objects, token budgets, progressive summarization |
| Governance, trust, human-in-the-loop | [GOVERNANCE.md](references/GOVERNANCE.md) | Audit trails, observability metrics, progressive trust, confirmation flows, admin controls, data retention rules |
| Slack MCP Server | [MCP-SERVER.md](references/MCP-SERVER.md) | Connecting agents to Slack data via MCP, OAuth scopes, rate limits, transport protocol |
| Workflow integration & custom steps | [WORKFLOW-INTEGRATION.md](references/WORKFLOW-INTEGRATION.md) | Adding AI as a custom workflow step in Workflow Builder |
| Agent design principles | [AGENT-DESIGN.md](references/AGENT-DESIGN.md) | Data boundaries, bounded autonomy, messaging best practices, naming |

## Setup

```bash
# Install Slack CLI
curl -fsSL https://downloads.slack-edge.com/slack-cli/install.sh | bash
slack login

# Scaffold from template
slack create agent   # support agent (Casey) with Claude/OpenAI SDK
# Or start blank:
# git clone https://github.com/slack-samples/bolt-js-starter-agent
```

Enable **Agents & AI Apps** in app settings → adds `assistant:write` scope. Subscribe to `assistant_thread_started`, `assistant_thread_context_changed`, `message.im`.

## Minimal agent skeleton

```typescript
import { App, Assistant } from '@slack/bolt';

const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET,
});

const assistant = new Assistant({
  threadStarted: async ({ say, setSuggestedPrompts }) => {
    await say({ text: 'How can I help?' });
    await setSuggestedPrompts({
      prompts: [
        { title: 'Summarize channel', message: 'Summarize #general' },
      ],
    });
  },
  threadContextChanged: async ({ saveThreadContext }) => {
    await saveThreadContext();
  },
  userMessage: async ({ message, say, setStatus, setTitle, sayStream }) => {
    await setTitle(message.text);
    await setStatus({ status: 'Thinking...' });
    // call LLM, then stream response:
    const stream = sayStream();
    await stream.append({ markdown_text: 'Here is the answer...' });
    await stream.stop();
  },
});

app.use(assistant);
```

## Key APIs at a glance

| Method | Purpose |
|---|---|
| `assistant.threads.setSuggestedPrompts` | Show clickable prompts in container |
| `assistant.threads.setStatus` | Loading indicator (empty string clears) |
| `assistant.threads.setTitle` | Name the thread in History tab |
| `chat.startStream` / `appendStream` / `stopStream` | Token-by-token text streaming |
| `assistant.search.context` | Semantic workspace search (requires `action_token`) |
| `conversations.replies` | Drill into a thread |
| `chat.postEphemeral` | Temporary user-only messages |

## Deployment options

- `slack run` — local dev via Slack CLI (Socket Mode)
- AWS Lambda, Heroku, Vercel — see Bolt JS deployment guides
- Requires paid Slack plan (or free Developer Program sandbox)
