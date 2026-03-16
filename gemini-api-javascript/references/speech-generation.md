# Speech Generation (TTS)

Gemini TTS transforms text into single or multi-speaker audio with natural language style control.

Models: `gemini-2.5-flash-preview-tts`, `gemini-2.5-pro-preview-tts`

TTS models accept **text-only input** and produce **audio-only output** (PCM 24kHz mono).

## Single Speaker

```ts
import { GoogleGenAI } from "@google/genai";
import wav from "wav";

const ai = new GoogleGenAI({});

const response = await ai.models.generateContent({
  model: "gemini-2.5-flash-preview-tts",
  contents: [{ parts: [{ text: "Say cheerfully: Have a wonderful day!" }] }],
  config: {
    responseModalities: ["AUDIO"],
    speechConfig: {
      voiceConfig: {
        prebuiltVoiceConfig: { voiceName: "Kore" },
      },
    },
  },
});

const data = response.candidates?.[0]?.content?.parts?.[0]?.inlineData?.data;
const audioBuffer = Buffer.from(data, "base64");

// Save as WAV (24kHz, 16-bit, mono)
function saveWav(filename, pcmData) {
  return new Promise((resolve, reject) => {
    const writer = new wav.FileWriter(filename, {
      channels: 1, sampleRate: 24000, bitDepth: 16,
    });
    writer.on("finish", resolve);
    writer.on("error", reject);
    writer.write(pcmData);
    writer.end();
  });
}
await saveWav("output.wav", audioBuffer);
```

## Multi-Speaker (up to 2)

Speaker names in the prompt must match `speaker` field in config:

```ts
const prompt = `TTS the following conversation between Joe and Jane:
Joe: How's it going today Jane?
Jane: Not too bad, how about you?`;

const response = await ai.models.generateContent({
  model: "gemini-2.5-flash-preview-tts",
  contents: [{ parts: [{ text: prompt }] }],
  config: {
    responseModalities: ["AUDIO"],
    speechConfig: {
      multiSpeakerVoiceConfig: {
        speakerVoiceConfigs: [
          { speaker: "Joe", voiceConfig: { prebuiltVoiceConfig: { voiceName: "Kore" } } },
          { speaker: "Jane", voiceConfig: { prebuiltVoiceConfig: { voiceName: "Puck" } } },
        ],
      },
    },
  },
});
```

## Controllable Style

Use natural language in the prompt to control style, accent, pace, and tone:

```
Say in a spooky whisper:
"By the pricking of my thumbs... Something wicked this way comes"
```

Advanced prompt structure for fine-grained control:

```
# AUDIO PROFILE: Jaz R.
## "The Morning Hype"

### DIRECTOR'S NOTES
Style: Enthusiastic, infectious energy, vocal smile
Pacing: Fast and bouncy, no dead air
Accent: Brixton, London

### TRANSCRIPT
Yes, massive vibes in the studio! You are locked in...
```

## Voice Options (30 total)

| Voice | Style | Voice | Style |
|---|---|---|---|
| Zephyr | Bright | Puck | Upbeat |
| Charon | Informative | Kore | Firm |
| Fenrir | Excitable | Leda | Youthful |
| Aoede | Breezy | Enceladus | Breathy |
| Achernar | Soft | Gacrux | Mature |
| Achird | Friendly | Sulafat | Warm |
| Sadachbia | Lively | Algieba | Smooth |

Full list of 30 voices available. Preview at [AI Studio](https://aistudio.google.com/generate-speech).

## Two-step: Generate transcript then TTS

```ts
const transcript = await ai.models.generateContent({
  model: "gemini-2.5-flash",
  contents: "Generate a 100-word podcast clip about reptiles. Hosts: Dr. Anya and Liam.",
});

const audio = await ai.models.generateContent({
  model: "gemini-2.5-flash-preview-tts",
  contents: transcript.text,
  config: {
    responseModalities: ["AUDIO"],
    speechConfig: {
      multiSpeakerVoiceConfig: {
        speakerVoiceConfigs: [
          { speaker: "Dr. Anya", voiceConfig: { prebuiltVoiceConfig: { voiceName: "Kore" } } },
          { speaker: "Liam", voiceConfig: { prebuiltVoiceConfig: { voiceName: "Puck" } } },
        ],
      },
    },
  },
});
```

## Limitations

- Text-only input, audio-only output
- 32k token context window
- Auto-detects language (70+ languages supported)
- Different from Live API (which handles interactive real-time audio)
