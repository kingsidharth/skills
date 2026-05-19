# Agents

## Overview

Agents combine planning, memory, and tool usage for complex, multi-turn tasks. The `Agent` protocol provides a flexible interface usable as a solver, standalone operation, delegated sub-agent, or tool.

## Agent Protocol

```python
from inspect_ai.agent import Agent, AgentState, agent

@agent
def my_agent() -> Agent:
    async def execute(state: AgentState) -> AgentState:
        """Agent description for tool registration."""
        messages, output = await get_model().generate_loop(
            state.messages, tools=[bash(), python()]
        )
        return state.completed(output, messages)
    return execute
```

Use as solver: `Task(..., solver=my_agent())`

## ReAct Agent

Built-in agent implementing the Reason + Act pattern:

```python
from inspect_ai.agent import react

Task(
    dataset=...,
    solver=react(tools=[bash(), python()]),
    sandbox="docker",
    scorer=includes(),
)
```

Options: `message_limit`, `token_limit`, `max_tool_output`, custom system prompt.

## Deep Agent

Extended agent with planning, memory, and skill tools built in:

```python
from inspect_ai.agent import deep_agent

Task(
    dataset=...,
    solver=deep_agent(tools=[bash(), python(), web_browser()]),
    sandbox="docker",
    scorer=...,
)
```

## Multi-Agent

Compose multiple agents. Use `handoff()` for delegation:

```python
from inspect_ai.agent import react, handoff

coder = react(tools=[bash(), python()], name="coder")
researcher = react(tools=[web_search()], name="researcher")

orchestrator = react(
    tools=[handoff(coder), handoff(researcher)],
    name="orchestrator",
)
```

Each agent runs with its own message history. The orchestrator delegates via tool calls.

## Agent Bridge

Run external agents (Claude Code, Codex CLI, Gemini CLI) as Inspect solvers:

```python
from inspect_ai.agent import agent_bridge

Task(
    dataset=...,
    solver=agent_bridge("claude-code"),  # or "codex-cli", "gemini-cli"
    sandbox="docker",
    scorer=...,
)
```

See [Inspect SWE](https://meridianlabs-ai.github.io/inspect_swe/) for production SWE agent integrations.

## Human Agent

For human baselining of computing tasks:

```python
from inspect_ai.agent import human_agent

Task(
    dataset=...,
    solver=human_agent(),
    sandbox="docker",
    scorer=...,
)
```

Provides a human with the same tools/sandbox as an AI agent for comparison baselines.

## Key Patterns

- Always set `message_limit` or `token_limit` for agent evals to prevent runaway costs
- Use `sandbox="docker"` for any agent with code execution tools
- Use `--max-samples` to control parallelism for resource-intensive agents
- Agent evals benefit from `epochs` to reduce variance
