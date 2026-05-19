# Ecosystem Libraries

Pydantic's model system extends into CLI tools, dependency injection, and data stores via third-party libraries.

## CLI — clipstick

Generates CLI interfaces directly from Pydantic models. No decorators or DSL — the model *is* the CLI spec.

```python
# cli.py
from pydantic import BaseModel, Field
from clipstick import parse

class MyArgs(BaseModel):
    """My CLI tool description."""
    input_file: str = Field(description="Path to input")
    output_file: str = Field(description="Path to output")
    verbose: bool = Field(default=False, description="Enable verbose logging")

args = parse(MyArgs)
```

Run: `python cli.py input.txt output.txt --verbose`

Supports subcommands via nested models and `Union` types.

**Install:** `pip install clipstick`

## CLI — pydantic-argparse

Wraps `argparse` with Pydantic models. More control over argparse features while keeping models as the schema source.

```python
from pydantic import BaseModel, Field
from pydantic_argparse import ArgumentParser

class Args(BaseModel):
    name: str = Field(description="Your name")
    count: int = Field(default=1, description="Repeat count")

parser = ArgumentParser(model=Args, description="Greeter")
args = parser.parse_typed_args()
```

**Install:** `pip install pydantic-argparse`

### When to use which

- **clipstick** — zero-boilerplate, model-first, no argparse dependency. Good for simple tools.
- **pydantic-argparse** — when you need argparse features (mutually exclusive groups, subparsers with more control, custom formatters).

## Redis — redis-om-python

Redis OM uses Pydantic models as the schema for Redis data:

```python
from redis_om import HashModel, Field

class Product(HashModel):
    name: str = Field(index=True)
    price: float = Field(index=True)
    quantity: int

product = Product(name="Widget", price=9.99, quantity=100)
product.save()

# Query
results = Product.find(Product.price < 20).all()
```

Supports `HashModel` (Redis hashes) and `JsonModel` (RedisJSON). Full-text search via RediSearch indexes.

**Install:** `pip install redis-om`

## Dependency Injection — FastDepends

Pydantic-powered dependency injection. Functions declare dependencies via type hints; FastDepends resolves, validates, and injects them.

```python
from fast_depends import inject, Depends

def get_db() -> Database:
    return Database(url="...")

@inject
async def handler(db: Database = Depends(get_db), user_id: int = 0):
    ...
```

Works standalone or integrates with FastAPI, FastStream, etc. Validates injected values through Pydantic.

**Install:** `pip install fast-depends`
