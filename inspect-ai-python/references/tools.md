# Tools

## Overview

Tools let models call Python functions during evaluation. Declared with `@tool`, tools require type annotations and docstrings.

## Custom Tools

```python
from inspect_ai.tool import tool

@tool
def add():
    async def execute(x: int, y: int):
        """Add two numbers.
        Args:
            x: First number.
            y: Second number.
        Returns:
            Sum of x and y.
        """
        return x + y
    return execute
```

Register tools with `use_tools()`:

```python
from inspect_ai.solver import generate, use_tools

Task(
    dataset=...,
    solver=[use_tools([add()]), generate()],
    scorer=match(),
)
```

## Standard Tools

| Tool | Import | Purpose |
|---|---|---|
| `bash()` | `inspect_ai.tool` | Execute shell commands |
| `python()` | `inspect_ai.tool` | Execute Python code |
| `bash_session()` | `inspect_ai.tool` | Stateful shell session |
| `text_editor()` | `inspect_ai.tool` | View/create/edit files |
| `computer()` | `inspect_ai.tool` | Desktop interaction via screenshots |
| `web_search()` | `inspect_ai.tool` | Search the web |
| `web_browser()` | `inspect_ai.tool` | Headless Chromium browser |
| `code_execution()` | `inspect_ai.tool` | Provider-sandboxed Python |
| `think()` | `inspect_ai.tool` | Extra reasoning step |
| `update_plan()` | `inspect_ai.tool` | Track steps/progress |
| `memory()` | `inspect_ai.tool` | Store/retrieve info across turns |
| `skill()` | `inspect_ai.tool` | Inject specialized knowledge |

## MCP Tools

Integrate any [MCP server](https://modelcontextprotocol.io/):

```python
from inspect_ai.tool import mcp_tools

tools = mcp_tools("npx -y @modelcontextprotocol/server-filesystem /path")
# or
tools = mcp_tools("http://localhost:8080/sse")
```

Use in a task: `solver=[use_tools(tools), generate()]`

## Sandboxing

Execute tool code in isolated Docker containers:

```python
Task(
    dataset=...,
    solver=[use_tools([bash()]), generate()],
    sandbox="docker",  # or ("docker", "compose.yaml")
    scorer=includes(),
)
```

### sandbox() Function

Inside tools, access the sandbox:

```python
from inspect_ai.util import sandbox

@tool
def list_files():
    async def execute(dir: str):
        """List files in a directory.
        Args:
            dir: Directory path.
        """
        result = await sandbox().exec(["ls", dir])
        if result.success:
            return result.stdout
        else:
            raise ToolError(result.stderr)
    return execute
```

### Per-sample Files and Setup

```python
Sample(
    input="Find the flag",
    target="CTF{found}",
    files={"/shared/flag.txt": "flag.txt"},
    setup="setup.sh",
    sandbox=("docker", "custom-compose.yaml"),
)
```

### Docker Configuration

Provide a `compose.yaml` alongside your task:

```yaml
services:
  default:
    build: .
    init: true
    command: tail -f /dev/null
```

Multiple environments: name services and prefix file paths (`victim:/path`).

### Resource Management

- `--max-sandboxes` — limit concurrent containers
- `--max-subprocesses` — limit subprocess concurrency
- `--max-samples` — limit concurrent sample execution

## Tool Approval

Require human approval before executing tool calls:

```python
from inspect_ai.approval import human_approval

Task(..., approval=human_approval())
```

Or define policies in YAML for auto-approve/deny patterns.
