# Apps and Deployment Reference

## Core Concepts

**Modal App** = Container for functions/classes deployed together
**Ephemeral App** = `modal run` - temporary, exits when script ends
**Deployed App** = `modal deploy` - persistent until stopped

## Basic App Structure

```python
import modal

# Create app
app = modal.App("my-app-name")

# Define function
@app.function()
def my_function():
    return "Hello from cloud"

# Local entrypoint (runs on your machine)
@app.local_entrypoint()
def main():
    result = my_function.remote()  # Calls cloud function
    print(result)
```

## Deployment

```bash
# Ephemeral (dev/testing)
modal run script.py

# Deployed (production)
modal deploy script.py

# Stop deployed app
modal app stop my-app-name
```

## Multi-File Projects

**Structure**:
```
my_project/
├── __init__.py          # Import all modules
├── llm.py              # LLM functions
├── web.py              # Web endpoints
└── deploy.py           # Combines all
```

**__init__.py**:
```python
from .llm import *
from .web import *
```

**deploy.py**:
```python
import modal
from . import llm, web

app = modal.App("combined")
app.include(llm.app)
app.include(web.app)
```

**Deploy**:
```bash
modal deploy -m my_project.deploy
```

## App Configuration

```python
app = modal.App(
    "my-app",
    secrets=[modal.Secret.from_name("api-keys")],  # Default secrets for all functions
)

# Tagging for billing/organization
app.set_tags({
    "environment": "production",
    "team": "ml"
})
```

## Function Invocation Patterns

```python
# Synchronous call
result = my_function.remote(args)

# Map (parallel)
results = my_function.map(inputs)

# Spawn (fire and forget)
for item in items:
    my_function.spawn(item)

# Generator (streaming results)
for result in my_function.map(inputs):
    process(result)
```

## Invoking Deployed Functions

**From Python**:
```python
import modal

# Lookup deployed function
func = modal.Function.lookup("my-app", "my-function")
result = func.remote(args)
```

**From TypeScript/JavaScript**:
```typescript
import { Modal } from "@modalhq/libmodal";

const modal = new Modal({
  tokenId: process.env.MODAL_TOKEN_ID,
  tokenSecret: process.env.MODAL_TOKEN_SECRET
});

const result = await modal.function.call("my-app", "my-function", {arg: value});
```

**From HTTP**:
```bash
curl -X POST https://yourname--my-app-my-function.modal.run \
  -H "Content-Type: application/json" \
  -d '{"arg": "value"}'
```

## Environments

Separate dev/staging/prod:

```bash
# Create environments
modal environment create dev
modal environment create prod

# Deploy to specific environment
modal deploy --env prod script.py

# Set default
modal config set-environment prod
```

## Best Practices

1. **Name apps consistently**: `team-service-purpose`
2. **Use environments**: Separate dev/prod
3. **Tag deployments**: For billing/tracking
4. **Structure multi-file**: Use __init__.py and deploy.py
5. **Version control**: Deployments create versions, can rollback

