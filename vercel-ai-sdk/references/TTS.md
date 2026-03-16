# Text-to-Speech (TTS)

## Core API

```ts
import { generateSpeech } from 'ai';

const { audio } = await generateSpeech({
  model: provider.speech('model-id'),
  text: 'Hello, world!',
  voice: 'nova',
  providerOptions: { /* model-specific options */ },
});

// audio.uint8Array — raw bytes
// audio.base64     — base64 string
// audio.mimeType   — e.g. 'audio/mpeg'
```

## ElevenLabs

```ts
import { ElevenLabs } from '@ai-sdk/elevenlabs';
// pnpm add @ai-sdk/elevenlabs
// Env: ELEVENLABS_API_KEY

const elevenlabs = new ElevenLabs();

const { audio } = await generateSpeech({
  model: elevenlabs.speech('eleven_turbo_v2_5'), // or eleven_monolingual_v1, eleven_multilingual_v2
  text: 'Hello from ElevenLabs!',
  voice: 'pNInz6obpgDQGcFmaJgB', // voice ID from ElevenLabs voice library
  providerOptions: {
    elevenlabs: {
      stability: 0.5,           // 0-1, higher = more stable
      similarity_boost: 0.75,   // 0-1, higher = more similar to original
      style: 0.0,               // 0-1, style exaggeration
      use_speaker_boost: true,
      language_code: 'en',      // only for turbo v2.5 + flash v2.5
    },
  },
});
```

## fal (fast image/audio inference)

```ts
import { fal } from '@ai-sdk/fal';
// pnpm add @ai-sdk/fal
// Env: FAL_API_KEY

// fal.ai TTS models — check https://fal.ai/explore/search?q=tts for available models
const { audio } = await generateSpeech({
  model: fal.speech('fal-ai/kokoro'),
  text: 'Hello from fal!',
  voice: 'af_nicole',
  providerOptions: {
    fal: { speed: 1.0 },
  },
});
```

## OpenAI TTS

```ts
import { openai } from '@ai-sdk/openai';
// Env: OPENAI_API_KEY

const { audio } = await generateSpeech({
  model: openai.speech('tts-1'),        // or tts-1-hd for higher quality
  text: 'Hello from OpenAI!',
  voice: 'nova',                         // alloy | ash | coral | echo | fable | onyx | nova | sage | shimmer
  providerOptions: {
    openai: {
      speed: 1.0,                        // 0.25 - 4.0
      instructions: 'Speak slowly.',    // not supported by tts-1/tts-1-hd
    },
  },
});
```

## Play audio in browser

```ts
// Backend route
app.post('/api/speech', async (req, res) => {
  const { text } = req.body;
  const { audio } = await generateSpeech({
    model: elevenlabs.speech('eleven_turbo_v2_5'),
    text,
    voice: 'pNInz6obpgDQGcFmaJgB',
  });
  res.set('Content-Type', audio.mimeType);
  res.send(Buffer.from(audio.uint8Array));
});
```

```tsx
// Frontend
async function playText(text: string) {
  const res = await fetch('/api/speech', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ text }),
  });
  const blob = await res.blob();
  const url = URL.createObjectURL(blob);
  const audio = new Audio(url);
  audio.play();
}
```

## Choosing a TTS provider

| Provider | Quality | Speed | Notes |
|---|---|---|---|
| ElevenLabs | ⭐⭐⭐⭐⭐ | Medium | Best voices, most control |
| fal (Kokoro) | ⭐⭐⭐⭐ | Fast | Open-source model, cheap |
| OpenAI tts-1-hd | ⭐⭐⭐⭐ | Medium | Good quality, easy setup |
| OpenAI tts-1 | ⭐⭐⭐ | Fast | Fastest, cost-optimized |
