---
name: gemini-api-javascript
description: "Google Gemini API integration using the @google/genai SDK for JavaScript/TypeScript. Covers text generation, image generation (Nano Banana), video generation (Veo), speech generation (TTS), image/video/audio/document understanding, thinking, structured outputs, function calling, embeddings, Live API (real-time streaming), batch processing, and safety settings. Use this skill whenever the user mentions Gemini, Google AI, @google/genai, Nano Banana, Veo, GoogleGenAI, or wants to integrate with Google's generative AI models in a JS/TS project — even if they just say 'call Gemini' or 'use Google AI'."
---

# Gemini API — JavaScript/TypeScript

## Requirements

- **SDK**: `@google/genai` (GA, replaces deprecated `@google/generative-ai`)
- **Runtime**: Node.js v18+
- **Install**: `npm install @google/genai`
- **Auth**: Set `GEMINI_API_KEY` env var, or pass `apiKey` to constructor

## Quick Start

```ts
import { GoogleGenAI } from "@google/genai";
const ai = new GoogleGenAI({});                       // reads GEMINI_API_KEY
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: "Explain quantum computing briefly",
});
console.log(response.text);
```

## Model Selection

| Model ID | Best For | Context | Notes |
|---|---|---|---|
| `gemini-3-pro-preview` | Complex reasoning, coding, multimodal | 1M in / 64k out | Default thinking_level: high |
| `gemini-3-flash-preview` | Fast reasoning, general-purpose | 1M in / 64k out | Supports minimal/low/medium/high thinking |
| `gemini-3-pro-image-preview` | High-quality image generation (Nano Banana Pro) | 65k in / 32k out | Requires thought signatures for editing |
| `gemini-3.1-flash-image-preview` | Fast image generation (Nano Banana 2) | — | 512px–4K output, image search grounding |
| `gemini-2.5-flash` | General-purpose (2.5 series) | 1M in / 65k out | Stable GA model |
| `gemini-2.5-flash-preview-tts` | Text-to-speech | 32k tokens | Audio-only output, 30 voices |
| `veo-3.1-generate-preview` | Video generation | — | 8s clips, 720p/1080p/4K, native audio |
| `gemini-embedding-001` | Text embeddings | 8192 tokens | 3072 dimensions, configurable |

## Structure

```
gemini-api-javascript/
├── SKILL.md                          ← You are here (routing hub)
└── references/
    ├── generation.md                 ← Text, streaming, chat, system instructions
    ├── image-generation.md           ← Nano Banana: text→image, editing, multi-turn
    ├── media-understanding.md        ← Image, video, audio, document input processing
    ├── video-generation.md           ← Veo: text→video, image→video, extension
    ├── speech-generation.md          ← TTS: single/multi-speaker, voice control
    ├── thinking.md                   ← Thinking levels, thought summaries, signatures
    ├── structured-outputs.md         ← JSON schema, Zod, streaming structured data
    ├── function-calling.md           ← Tool declarations, auto/manual modes, parallel
    ├── embeddings.md                 ← Text embeddings, batch, task types
    ├── live-api.md                   ← Real-time WebSocket streaming, audio/video
    ├── batch-api.md                  ← Async bulk processing, JSONL files
    └── safety-and-config.md          ← Safety settings, media resolution, token counting
```

## Routing Guide

Determine which reference file to read based on the task:

**"Generate text / chat / stream"** → Read [generation.md](references/generation.md)

**"Generate an image" or "Edit an image"** → Read [image-generation.md](references/image-generation.md)

**"Analyze an image/video/audio/PDF"** → Read [media-understanding.md](references/media-understanding.md)

**"Generate a video"** → Read [video-generation.md](references/video-generation.md)

**"Text-to-speech" or "Generate audio from text"** → Read [speech-generation.md](references/speech-generation.md)

**"Use thinking / reasoning"** → Read [thinking.md](references/thinking.md)

**"Return JSON / structured data"** → Read [structured-outputs.md](references/structured-outputs.md)

**"Call external functions / tools"** → Read [function-calling.md](references/function-calling.md)

**"Embeddings / semantic search"** → Read [embeddings.md](references/embeddings.md)

**"Real-time audio/video streaming"** → Read [live-api.md](references/live-api.md)

**"Process many requests in bulk"** → Read [batch-api.md](references/batch-api.md)

**"Safety filters / token counting"** → Read [safety-and-config.md](references/safety-and-config.md)

## Key SDK Patterns

The `@google/genai` SDK uses a consistent pattern across all operations:

```ts
import { GoogleGenAI } from "@google/genai";
const ai = new GoogleGenAI({});  // or { apiKey: "..." }

// All methods hang off ai.models, ai.chats, ai.files, ai.batches, ai.operations
```

Config is always passed via a `config` object using **camelCase** property names (JS convention), mapping to the REST API's snake_case fields. For example: `responseMimeType`, `thinkingConfig`, `speechConfig`, `imageConfig`.

## Critical: Gemini 3 Thought Signatures

Gemini 3 models produce `thoughtSignature` fields on response parts. These encrypted tokens must be passed back in multi-turn conversations to maintain reasoning context.

- **Function calling**: Strictly enforced — missing signatures cause 400 errors
- **Image generation/editing**: Strictly enforced for conversational editing
- **Text/chat**: Not strictly enforced but improves quality

The SDK handles signatures automatically when using `ai.chats.create()`. For manual history management, preserve `thoughtSignature` on every part that has one.

If migrating conversation history from another model, use the dummy value: `"context_engineering_is_the_way_to_go"`

## Gemini 3 Key Defaults

- **Temperature**: Keep at default `1.0` — lower values may cause looping
- **Thinking**: Defaults to `high` (dynamic) — use `thinkingLevel: "low"` for speed
- **Media resolution**: Higher defaults for images than 2.5 series; use `mediaResolution` parameter for control
