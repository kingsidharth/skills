# Extensions & Ecosystem

## Inspect Scout

**Transcript analysis for AI agents.** Detect issues in agent transcripts using LLM-based or pattern-based scanners.

```bash
pip install inspect-scout
```

### Capabilities

- **LLM Scanner** — use an LLM to detect issues (misconfigured environments, refusals, eval awareness)
- **Grep Scanner** — regex/pattern-based detection
- **Custom Scanners** — extend with your own detection logic
- **Transcripts Database** — store, query, and publish transcripts from Inspect, LangSmith, Logfire, W&B Weave, Claude Code, etc.
- **Scout View** — visual explorer in VS Code extension
- **Validation** — benchmark scanner accuracy against human labels

### Usage

```python
from inspect_scout import scan, llm_scanner

results = scan(
    transcripts="./logs/",
    scanners=[llm_scanner("Check for eval awareness")],
)
```

Docs: [inspect_scout](https://meridianlabs-ai.github.io/inspect_scout/)

---

## Inspect Flow

**Workflow orchestration for systematic evaluations.**

```bash
pip install inspect-flow
```

### What it Solves

- Declarative configuration of complex eval matrices
- Global log reuse (only run what's new/changed)
- Parameter sweeping across tasks × models × hyperparams

### Usage

```python
from inspect_flow import FlowSpec, FlowTask, FlowModel

spec = FlowSpec(
    tasks=[
        FlowTask(task="mmlu.py", params={"subset": "stem"}),
        FlowTask(task="gsm8k.py"),
    ],
    models=[
        FlowModel(model="openai/gpt-4o"),
        FlowModel(model="anthropic/claude-sonnet-4-0"),
    ],
)
spec.run()
```

Docs: [inspect_flow](https://meridianlabs-ai.github.io/inspect_flow/)

---

## Inspect Sandboxes — Daytona

Alternative sandbox provider using [Daytona](https://www.daytona.io/) for remote dev environments instead of Docker.

```bash
pip install inspect-sandboxes
```

```python
Task(..., sandbox="daytona")
```

Docs: [inspect_sandboxes](https://meridianlabs-ai.github.io/inspect_sandboxes/daytona.html)

---

## inspect_costs_plugin

Track and report API costs across eval runs.

```bash
pip install inspect-costs-plugin
```

Adds cost tracking to eval logs. Useful for budgeting large-scale benchmark runs.

GitHub: [inspect_costs_plugin](https://github.com/jasongwartz/inspect_costs_plugin/)

---

## Inspect SWE

Production-grade SWE agent integrations (Claude Code, Codex CLI) for software engineering benchmarks.

Available via Meridian Labs: [inspect_swe](https://meridianlabs-ai.github.io/inspect_swe/)
