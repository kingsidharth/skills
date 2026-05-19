# Agent Design Principles

## Identity

- Agent must be clearly distinguishable from a human at all times
- Name should lead with utility/function, not just a brand
- Signal clearly when acting on a user's behalf

## Messaging behavior

- Reply in-thread when @mentioned in channels
- Private info only via DMs, private channels, or ephemeral messages
- Batch updates into single messages to avoid notification overload
- Summarize rather than narrate; provide links for supporting detail

## Data boundaries

- Agent must not access information the invoking user can't access themselves
- If user can't open a file/canvas/record, agent can't use it for context
- Huddle transcripts/summaries follow the same sharing model
- Slack's DM/private/public channel model must be respected

## Bounded autonomy

Balance capability with safety:
- Set clear boundaries on what the agent can do without asking
- Start narrow, expand access as trust is earned
- Strong defaults with flexibility over time
- When a decision exceeds authority boundaries, pause for human input

## What agents are vs aren't

**Agents are**: autonomous within boundaries, goal-oriented, tool-using, context-maintaining.

**Agents are not just**: bots (predetermined responses), workflows (fixed step sequences), assistants (reactive Q&A only), or magic (limited by tools given and goal quality).

## Core principles

1. **User control**: every action/decision accessible to user; real-world actions require confirmation; design for failure
2. **Non-disruptive**: available where work happens, don't pull users out of flow
3. **Safety first**: guardrails, permissions, human-in-the-loop as engineering requirements

## Distribution

- **Internal**: company-specific use cases, private systems, no broad distribution needed
- **Marketplace**: generalizable problems, multi-org support, meets review/security requirements
