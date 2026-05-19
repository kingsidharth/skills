# Models

## Provider Selection

Model string format: `provider/model-name`

| Category | Providers |
|---|---|
| Lab APIs | OpenAI, Anthropic, Google, Grok, Mistral, DeepSeek, Perplexity |
| Cloud | AWS Bedrock, AWS SageMaker, Azure AI |
| Hosted Open | Groq, Together AI, Fireworks AI, Cloudflare, HF Inference, SambaNova |
| Local Open | Hugging Face, vLLM, Ollama, llama-cpp-python, SGLang, TransformerLens, nnterp |

OpenAI-compatible endpoints work via `openai/model-name` with a custom `base_url`.

```bash
inspect eval task.py --model openai/gpt-4o
inspect eval task.py --model anthropic/claude-sonnet-4-0
inspect eval task.py --model google/gemini-2.5-pro
inspect eval task.py --model hf/meta-llama/Llama-2-7b-chat-hf
```

## GenerateConfig

Control model generation parameters:

```python
from inspect_ai.model import GenerateConfig

Task(..., config=GenerateConfig(
    temperature=0.7,
    max_tokens=1024,
    top_p=0.9,
    stop_seqs=["ANSWER:"],
))
```

CLI: `inspect eval task.py --temperature 0.5 --max-tokens 512`

## Model API (Programmatic)

```python
from inspect_ai.model import get_model

model = get_model("openai/gpt-4o")
response = await model.generate("What is 2+2?")
```

## Model Roles

Assign different models to different roles:

- **eval model** — the model being evaluated (default)
- **grader model** — used by model-graded scorers
- **agent model** — used inside agent solvers

```bash
inspect eval task.py --model openai/gpt-4o --model-roles grader=anthropic/claude-sonnet-4-0
```

## Caching

Cache model responses to avoid redundant API calls:

```bash
inspect eval task.py --cache     # enable caching
```

Cache is per-model, per-input. Useful for development/debugging.

## Batch Mode

Run evals using provider batch APIs (lower cost, higher latency):

```bash
inspect eval task.py --model openai/gpt-4o --batch
```

Supported: OpenAI, Anthropic, Google. Logs are written when the batch completes.

## Compaction

For long conversations, compaction summarizes earlier messages to fit context windows:

```python
from inspect_ai.model import compact

Task(..., solver=[..., compact()])
```

## Multimodal

Include images in samples:

```python
from inspect_ai.dataset import Sample
from inspect_ai.model import ChatMessageUser, ContentImage

Sample(input=[
    ChatMessageUser(content=[
        ContentText(text="Describe this image"),
        ContentImage(image="image.png"),
    ])
])
```

## Reasoning Models

Models with extended thinking (o1, o3, Claude with thinking):

```python
Task(..., config=GenerateConfig(reasoning_effort="high"))
```

## Structured Output

Force JSON schema output:

```python
from inspect_ai.model import GenerateConfig
from pydantic import BaseModel

class Answer(BaseModel):
    reasoning: str
    answer: str

Task(..., config=GenerateConfig(response_format=Answer))
```
