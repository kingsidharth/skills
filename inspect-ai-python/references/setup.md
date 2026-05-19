# Setup

## Installation

```bash
pip install inspect-ai
```

Optional: Install the [VS Code Extension](https://inspect.aisi.org.uk/vscode.html) for authoring/debugging support and log viewing.

## API Keys

Each model provider requires its own package + API key:

```bash
# Pick one (or more)
pip install openai        && export OPENAI_API_KEY=...
pip install anthropic     && export ANTHROPIC_API_KEY=...
pip install google-genai  && export GOOGLE_API_KEY=...
pip install mistralai     && export MISTRAL_API_KEY=...
```

Model strings follow `provider/model-name` format:
`openai/gpt-4o`, `anthropic/claude-sonnet-4-0`, `google/gemini-2.5-pro`, `hf/meta-llama/Llama-2-7b-chat-hf`

## .env Files

Inspect auto-reads `.env` from the working directory (searches parents). Common settings:

```env
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
INSPECT_EVAL_MODEL=anthropic/claude-sonnet-4-0
INSPECT_LOG_DIR=./logs
INSPECT_LOG_LEVEL=warning
INSPECT_EVAL_MAX_RETRIES=5
INSPECT_EVAL_MAX_CONNECTIONS=20
```

All CLI options map to `INSPECT_EVAL_*` env vars. Never commit `.env` to version control.

## Running Your First Eval

```bash
inspect eval my_eval.py --model openai/gpt-4o
```

Or set `INSPECT_EVAL_MODEL` and omit `--model`.

From Python:

```python
from inspect_ai import eval
logs = eval("my_eval.py", model="openai/gpt-4o")
```

## Log Viewer

```bash
inspect view                    # opens web UI for latest log
inspect view --log-dir ./logs   # browse a directory of logs
```
