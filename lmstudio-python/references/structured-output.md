# Structured Output

Enforce JSON-conformant responses via `response_format` on `.respond()`.

## Pydantic schema (recommended)

```python
from pydantic import BaseModel

class Book(BaseModel):
    title: str
    author: str
    year: int

result = model.respond("Tell me about The Hobbit", response_format=Book)
book = result.parsed  # typed dict: {title, author, year}
```

`lmstudio.BaseModel` (a `msgspec.Struct` subclass) also works.

Any class implementing `lmstudio.ModelSchema` protocol is accepted:
```python
@runtime_checkable
class ModelSchema(Protocol):
    @classmethod
    def model_json_schema(cls) -> DictSchema: ...
```

## Raw JSON schema

```python
schema = {
    "type": "object",
    "properties": {
        "title": {"type": "string"},
        "author": {"type": "string"},
        "year": {"type": "integer"},
    },
    "required": ["title", "author", "year"],
}
result = model.respond("Tell me about The Hobbit", response_format=schema)
book = result.parsed
```

## Streaming with structured output

Works the same — `response_format` accepted on `.respond_stream()`. Final `result.parsed` available after stream completes.

## Notes

- Zod schemas not supported (Python SDK)
- `structured` in `config` dict also works but `response_format` is preferred
- `result.parsed` returns a `dict` for structured responses, a `str` for unstructured
