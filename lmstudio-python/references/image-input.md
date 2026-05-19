# Image Input (VLMs)

*SDK ≥ 1.1.0*

Vision-Language Models accept images alongside text. Supported formats: JPEG, PNG, WebP.

## Workflow

```python
import lmstudio as lms

model = lms.llm("qwen2-vl-2b-instruct")
image = lms.prepare_image("/path/to/image.jpg")

chat = lms.Chat()
chat.add_user_message("Describe this image", images=[image])
result = model.respond(chat)
```

## Input types for `prepare_image()`

- File path (`str` or `Path`)
- Raw `bytes` (image data directly — no disk write needed)
- Binary IO objects

Binary filesystem paths are not supported (interpreted as malformed image data).

## Getting a VLM

```bash
lms get qwen2-vl-2b-instruct
```
