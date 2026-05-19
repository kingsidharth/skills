# Gemini API Skill

**CRITICAL RULES - READ FIRST:**
1. **ALWAYS fetch official documentation** - Do NOT rely on memory. Use `web_fetch` to retrieve current docs from ai.google.dev
2. **Follow official patterns** - All code examples must match Google's official SDK patterns
3. **Check model availability** - Different models support different features (see MODEL_CAPABILITIES.md)
4. **Use progressive disclosure** - Start with quickstart patterns, link to specialized references
5. **Multiple language support** - Provide examples in Python, JavaScript/TypeScript, and REST when relevant

## Overview

The Gemini API provides access to Google's most capable multimodal AI models with support for text, images, video, audio, and code. This skill covers the complete API including advanced features like function calling, structured outputs, batch processing, and long context understanding.

**Current date context:** February 2026 - Gemini 3 series and 2.5 series models are the latest available.

## Quick Navigation

**For general implementation:**
- New to Gemini? → Start with QUICKSTART.md
- Building a chatbot/agent? → FUNCTION_CALLING.md + STRUCTURED_OUTPUT.md  
- Need predictable JSON? → STRUCTURED_OUTPUT.md
- Processing large volumes? → BATCH_API.md
- Working with long documents? → LONG_CONTEXT.md

**For specific capabilities:**
- Image generation → IMAGE_GENERATION.md (Nano Banana)
- Video generation → VIDEO_GENERATION.md (Veo)
- Video understanding → VIDEO_UNDERSTANDING.md
- Image understanding → Built into core models (see QUICKSTART.md)
- YouTube video analysis → LONG_CONTEXT.md + VIDEO_UNDERSTANDING.md

**For optimization:**
- Reduce costs → BATCH_API.md (50% discount) + CONTEXT_CACHING.md
- Handle rate limits → BATCH_API.md
- Manage safety → SAFETY_SETTINGS.md

## Core Concepts

### SDK Installation

**Python:**
```bash
pip install google-genai
```

**JavaScript/TypeScript:**
```bash
npm install @google/genai
```

### Authentication

Get API key from: https://aistudio.google.com/apikey

**Python:**
```python
from google import genai

client = genai.Client(api_key="YOUR_API_KEY")
# Or set GOOGLE_API_KEY environment variable
client = genai.Client()  # Reads from env
```

**JavaScript:**
```javascript
import { GoogleGenAI } from '@google/genai';

const ai = new GoogleGenAI({ apiKey: 'YOUR_API_KEY' });
// Or set GOOGLE_API_KEY environment variable
const ai = new GoogleGenAI({});  // Reads from env
```

### Basic Text Generation

**Python:**
```python
from google import genai

client = genai.Client()

response = client.models.generate_content(
    model="gemini-3-flash-preview",
    contents="Explain quantum computing in simple terms"
)

print(response.text)
```

**JavaScript:**
```javascript
import { GoogleGenAI } from '@google/genai';

const ai = new GoogleGenAI({});

const response = await ai.models.generateContent({
    model: 'gemini-3-flash-preview',
    contents: 'Explain quantum computing in simple terms'
});

console.log(response.text);
```

### Streaming Responses

**Python:**
```python
response_stream = client.models.generate_content_stream(
    model="gemini-3-flash-preview",
    contents="Write a story about a robot"
)

for chunk in response_stream:
    print(chunk.text, end='')
```

**JavaScript:**
```javascript
const responseStream = await ai.models.generateContentStream({
    model: 'gemini-3-flash-preview',
    contents: 'Write a story about a robot'
});

for await (const chunk of responseStream) {
    console.log(chunk.text);
}
```

## Model Selection Guide

| Model | Best For | Context Window | Key Features |
|-------|----------|----------------|--------------|
| gemini-3-pro | Complex reasoning, multimodal understanding | 2M tokens | Best-in-class reasoning, thinking mode |
| gemini-3-flash | Fast, cost-effective tasks | 1M tokens | Excellent balance of speed/quality |
| gemini-2.5-pro | Production workloads | 2M tokens | Stable, reliable performance |
| gemini-2.5-flash | High-volume tasks | 1M tokens | Cost-effective at scale |

See MODEL_CAPABILITIES.md for complete feature matrix.

## Reference Files

### Core API Patterns
- **QUICKSTART.md** - Complete getting started guide with all basic patterns
- **FUNCTION_CALLING.md** - Build agents with external tool integration
- **STRUCTURED_OUTPUT.md** - Guarantee JSON schema compliance
- **MULTIMODAL_INPUT.md** - Working with images, audio, video, documents

### Advanced Features
- **BATCH_API.md** - Process thousands of requests at 50% cost
- **LONG_CONTEXT.md** - Handle millions of tokens efficiently
- **CONTEXT_CACHING.md** - Reduce costs for repeated context
- **THINKING_MODE.md** - Gemini 3's reasoning capabilities

### Specialized Capabilities
- **IMAGE_GENERATION.md** - Nano Banana for image creation/editing
- **VIDEO_GENERATION.md** - Veo for video synthesis
- **VIDEO_UNDERSTANDING.md** - Analyze video content
- **LIVE_API.md** - Real-time voice/audio interactions

### Configuration & Management
- **SAFETY_SETTINGS.md** - Content filtering configuration
- **MODEL_CAPABILITIES.md** - Complete feature support matrix
- **TROUBLESHOOTING.md** - Common issues and solutions
- **BEST_PRACTICES.md** - Production deployment guidelines

## Common Patterns

### Chat Conversations

**Python:**
```python
from google.genai import types

chat = client.chats.create(
    model="gemini-3-flash-preview",
    config=types.GenerateContentConfig(
        temperature=0.7,
        max_output_tokens=1000
    )
)

response = chat.send_message("Hello!")
print(response.text)

response = chat.send_message("What did I just say?")
print(response.text)  # Model remembers conversation
```

**JavaScript:**
```javascript
const chat = ai.chats.create({
    model: 'gemini-3-flash-preview',
    config: {
        temperature: 0.7,
        maxOutputTokens: 1000
    }
});

let response = await chat.sendMessage({ message: 'Hello!' });
console.log(response.text);

response = await chat.sendMessage({ message: 'What did I just say?' });
console.log(response.text);  // Model remembers conversation
```

### Multimodal Input (Image + Text)

**Python:**
```python
import base64

# Read image file
with open("image.jpg", "rb") as f:
    image_data = base64.b64encode(f.read()).decode()

response = client.models.generate_content(
    model="gemini-3-flash-preview",
    contents=[
        types.Part(text="What's in this image?"),
        types.Part(
            inline_data=types.Blob(
                mime_type="image/jpeg",
                data=image_data
            )
        )
    ]
)

print(response.text)
```

**JavaScript:**
```javascript
import fs from 'fs';

const imageData = fs.readFileSync('image.jpg').toString('base64');

const response = await ai.models.generateContent({
    model: 'gemini-3-flash-preview',
    contents: [
        { text: "What's in this image?" },
        {
            inlineData: {
                mimeType: 'image/jpeg',
                data: imageData
            }
        }
    ]
});

console.log(response.text);
```

## Key Differences from Other APIs

### vs OpenAI
- **Function calling:** Similar concept, different schema format (OpenAPI 3.0 subset)
- **Structured output:** Native JSON schema support with property ordering
- **Thinking mode:** Gemini 3 exposes reasoning process transparently
- **Multimodal:** Images/video/audio supported natively in all models
- **Context caching:** Built-in, not requiring separate endpoints

### vs Anthropic
- **Tool use:** Gemini calls it "function calling," same core concept
- **Streaming:** Different event structure for streaming responses
- **System instructions:** Set via `system_instruction` parameter, not separate role
- **Artifacts:** No direct equivalent; use structured output for typed data

## Important Notes

### Thought Signatures (Gemini 3)
Gemini 3 models use internal thinking processes. When using function calling or multi-turn conversations:
- **SDKs handle automatically** - No manual management needed
- **Manual API calls** - Must preserve `thought_signature` in conversation history
- See THINKING_MODE.md for details

### Temperature Settings
- **Gemini 3 models:** Keep temperature at default (1.0) for best results
- **Earlier models:** Can use temperature 0-2 for determinism control
- Lowering temperature on Gemini 3 can cause loops or degraded performance

### Rate Limits
- **Free tier:** 15 requests/minute, 1 million tokens/minute  
- **Paid tier:** 1000 requests/minute, 4 million tokens/minute
- **Batch API:** Higher limits, separate quota pool
- See official docs for current limits: https://ai.google.dev/gemini-api/docs/rate-limits

## Use Case Decision Tree

**Q: Do you need immediate responses?**
- Yes → Use standard API (`generate_content`)
- No, can wait 24hrs → Use BATCH_API.md (50% cost savings)

**Q: Do you need to call external tools/APIs?**  
- Yes → FUNCTION_CALLING.md
- No → Continue

**Q: Do you need responses in specific JSON format?**
- Yes → STRUCTURED_OUTPUT.md  
- No → Continue

**Q: Are you processing large documents (100+ pages)?**
- Yes → LONG_CONTEXT.md + CONTEXT_CACHING.md
- No → Continue

**Q: Do you need to generate images?**
- Yes → IMAGE_GENERATION.md (Nano Banana)
- No → Continue

**Q: Do you need to generate videos?**
- Yes → VIDEO_GENERATION.md (Veo)
- No → Use standard text generation

## Resources

### Official Documentation
- Main docs: https://ai.google.dev/gemini-api/docs
- API reference: https://ai.google.dev/api
- Cookbook (examples): https://github.com/google-gemini/cookbook
- Google AI Studio: https://aistudio.google.com

### SDK Documentation  
- Python SDK: https://googleapis.github.io/python-genai
- JavaScript SDK: https://github.com/googleapis/sdk-platform-nodejs

### Community
- Discussion forum: https://discuss.ai.google.dev/c/gemini-api
- Status page: https://aistudio.google.com/status

## Next Steps

1. **Start here:** Read QUICKSTART.md for comprehensive setup
2. **Choose your path:**
   - Chatbot/Agent → FUNCTION_CALLING.md
   - Data extraction → STRUCTURED_OUTPUT.md  
   - Bulk processing → BATCH_API.md
   - Document analysis → LONG_CONTEXT.md
3. **Optimize:** Review BEST_PRACTICES.md before production deployment

## Version Information

- **Skill created:** February 2026
- **Latest model series:** Gemini 3 (3-pro, 3-flash)
- **Stable models:** Gemini 2.5 (2.5-pro, 2.5-flash)
- **SDK versions:** google-genai (Python), @google/genai (JavaScript)

Always verify current model availability and features at: https://ai.google.dev/gemini-api/docs/models
