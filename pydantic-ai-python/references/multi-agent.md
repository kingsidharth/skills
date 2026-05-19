# Multi-Agent Patterns

## Agent delegation

An agent calls another agent from within a tool. The parent retains control after the delegate finishes.

```python
from pydantic_ai import Agent, RunContext

joke_generator = Agent('openai:gpt-5.2', output_type=list[str])

@joke_generator.tool
async def get_jokes(ctx: RunContext, topic: str) -> list[str]:
    """Generate jokes about a topic."""
    return [f'Joke about {topic}']

joke_selector = Agent('anthropic:claude-sonnet-4-6', deps_type=str)

@joke_selector.tool
async def joke_factory(ctx: RunContext[str], topic: str) -> list[str]:
    """Get candidate jokes, then the parent picks the best."""
    r = await joke_generator.run(
        f'Generate 5 jokes about {topic}',
        usage=ctx.usage,  # share usage tracking
    )
    return r.output
```

Key pattern: pass `ctx.usage` to delegate so usage counts toward parent's total.

Delegates can use different models. Usage-based cost tracking still works via `UsageLimits`.

## Programmatic handoff

Multiple agents called in sequence by application code. No agent calls another — your code decides routing.

```python
flight_agent = Agent('...', output_type=FlightResult)
seat_agent = Agent('...', output_type=SeatPreference)

flight = await flight_agent.run('Find flights to Paris')
seat = await seat_agent.run(
    f'User booked {flight.output}. What seat?',
    message_history=flight.all_messages(),
)
```

Agents don't need the same deps type in this pattern.

## Output functions as handoff

Use `@agent.output_function` to hand off to another agent as the final action (no return to parent):

```python
@triage_agent.output_function
async def route_to_billing(ctx: RunContext, query: str) -> BillingResult:
    return (await billing_agent.run(query)).output
```

## Multi-model delegation

Delegate agents can use cheaper/faster models for subtasks:
```python
summarizer = Agent('groq:llama-3-70b')  # fast, cheap
analyzer = Agent('anthropic:claude-sonnet-4-6')  # capable

@analyzer.tool
async def get_summary(ctx: RunContext, doc: str) -> str:
    return (await summarizer.run(f'Summarize: {doc}')).output
```
