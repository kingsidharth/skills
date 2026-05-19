# Models and Providers

## Model strings

Format: `provider:model-name`

| Provider | String prefix | Extra | Env var |
|----------|--------------|-------|---------|
| OpenAI | `openai:` | `pydantic-ai[openai]` | `OPENAI_API_KEY` |
| Anthropic | `anthropic:` | `pydantic-ai[anthropic]` | `ANTHROPIC_API_KEY` |
| Google | `google-gla:` | `pydantic-ai[google]` | `GOOGLE_API_KEY` |
| Groq | `groq:` | `pydantic-ai[groq]` | `GROQ_API_KEY` |
| Mistral | `mistral:` | `pydantic-ai[mistral]` | `MISTRAL_API_KEY` |
| Cohere | `cohere:` | `pydantic-ai[cohere]` | `CO_API_KEY` |
| Bedrock | `bedrock:` | `pydantic-ai[bedrock]` | AWS creds |
| HuggingFace | `huggingface:` | `pydantic-ai[huggingface]` | `HF_TOKEN` |
| Ollama | `ollama:` | `pydantic-ai[openai]` | — |
| OpenRouter | `openrouter:` | `pydantic-ai[openai]` | `OPENROUTER_API_KEY` |
| xAI | `xai:` | `pydantic-ai[openai]` | `XAI_API_KEY` |
| Outlines | `outlines:` | `pydantic-ai[outlines]` | — |

Examples: `'openai:gpt-5.2'`, `'anthropic:claude-sonnet-4-6'`, `'google-gla:gemini-2.5-flash'`.

## Setting model at run time

```python
agent = Agent()  # no default model
result = agent.run_sync('...', model='anthropic:claude-sonnet-4-6')
```

## Model instances

For custom configuration (base URL, API key, HTTP client):
```python
from pydantic_ai.models.openai import OpenAIModel

model = OpenAIModel('gpt-5.2', api_key='...', base_url='https://...')
agent = Agent(model)
```

## Ollama (via OpenAI-compatible)

```python
from pydantic_ai.models.openai import OpenAIModel

model = OpenAIModel('llama3.2', provider=OpenAIProvider(base_url='http://localhost:11434/v1'))
agent = Agent(model)
```

## Fallback models

```python
from pydantic_ai.models.fallback import FallbackModel

model = FallbackModel('anthropic:claude-sonnet-4-6', 'openai:gpt-5.2')
agent = Agent(model)  # tries Anthropic first, falls back to OpenAI
```

## Model settings

```python
agent = Agent('...', model_settings={'temperature': 0.7, 'max_tokens': 2000, 'top_p': 0.9})
```

Run-time settings override agent defaults.

## Test models

For unit tests without API calls:
```python
from pydantic_ai.models.test import TestModel

result = agent.run_sync('...', model=TestModel())
```

`FunctionModel` for custom mock logic:
```python
from pydantic_ai.models.function import FunctionModel

def mock_fn(messages, settings):
    return ModelResponse(parts=[TextPart(content='mocked')])

result = agent.run_sync('...', model=FunctionModel(mock_fn))
```

## Pydantic AI Gateway

Optional proxy that routes requests through a unified endpoint. Prefix model strings with `gateway/`:
```python
agent = Agent('gateway/anthropic:claude-sonnet-4-6')
```

Configure via `PYDANTIC_AI_GATEWAY_URL` env var.
