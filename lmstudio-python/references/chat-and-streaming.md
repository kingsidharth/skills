# Chat Completions & Streaming

## Obtain a model handle

```python
import lmstudio as lms
model = lms.llm()                         # any loaded model
model = lms.llm("qwen2.5-7b-instruct")   # specific model (JIT loads if needed)
```

## Single response

```python
result = model.respond("prompt")
result = model.respond(chat)              # lms.Chat object
result = model.respond(chat, config={"temperature": 0.6, "maxTokens": 50})
```

## Streaming

```python
for fragment in model.respond_stream("prompt"):
    print(fragment.content, end="", flush=True)
```

## Chat context management

```python
chat = lms.Chat("You are a helpful assistant.")  # system prompt
chat.add_user_message("Hello")
result = model.respond(chat)
```

Append assistant replies automatically:
```python
stream = model.respond_stream(chat, on_message=chat.append)
for fragment in stream:
    print(fragment.content, end="", flush=True)
```

Build from history:
```python
chat = lms.Chat.from_history({
    "messages": [
        {"role": "user", "content": "Hi"},
        {"role": "assistant", "content": "Hello!"},
    ]
})
```

## Prediction stats

```python
print(result.model_info.display_name)
print(result.stats.predicted_tokens_count)
print(result.stats.time_to_first_token_sec)
print(result.stats.stop_reason)
```

For streaming, use `stream.wait_for_result()` or `stream.result()` (non-blocking, raises if not ready).

## Progress callbacks

Available on both `.respond()` and `.respond_stream()`:
- `on_prompt_processing_progress` — `(progress: float) -> None`, 0.0–1.0
- `on_first_token` — no args, fires after prompt processing
- `on_prediction_fragment` — same fragments as stream iteration
- `on_message` — full assistant message on completion

## Multi-turn chatbot pattern

```python
model = lms.llm()
chat = lms.Chat("You are a task focused AI assistant")
while True:
    user_input = input("You: ")
    if not user_input:
        break
    chat.add_user_message(user_input)
    stream = model.respond_stream(chat, on_message=chat.append)
    print("Bot: ", end="", flush=True)
    for fragment in stream:
        print(fragment.content, end="", flush=True)
    print()
```

## Sync API timeout (SDK ≥ 1.5.0)

Default 60s inactivity timeout. Adjust:
```python
lms.set_sync_api_timeout(120)   # seconds
lms.set_sync_api_timeout(None)  # disable
current = lms.get_sync_api_timeout()
```

Async API: use `asyncio.wait_for()` or `anyio.move_on_after()` instead.
