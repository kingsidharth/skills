# Live API

Real-time, low-latency voice and video interactions over WebSocket.

Model: `gemini-2.5-flash-native-audio-preview-09-2025` (or other Live-compatible models)

## Architecture

Two approaches:
- **Server-to-server**: Backend connects to Live API, client sends audio to your server
- **Client-to-server**: Frontend connects directly via WebSocket (use ephemeral tokens for security)

## Basic Audio Stream (Node.js Server)

```ts
import { GoogleGenAI, Modality } from "@google/genai";
import mic from "mic";
import Speaker from "speaker";

const ai = new GoogleGenAI({});

const model = "gemini-2.5-flash-native-audio-preview-09-2025";
const config = {
  responseModalities: [Modality.AUDIO],
  systemInstruction: "You are a helpful AI assistant.",
};

async function main() {
  const session = await ai.live.connect({ model, config });

  // Set up microphone input
  const micInstance = mic({ rate: "16000", channels: "1", bitwidth: "16" });
  const micStream = micInstance.getAudioStream();

  micStream.on("data", (data) => {
    session.sendRealtimeInput({
      audio: { data: data.toString("base64"), mimeType: "audio/pcm" },
    });
  });
  micInstance.start();

  // Set up speaker output
  const speaker = new Speaker({ channels: 1, bitDepth: 16, sampleRate: 24000 });

  // Receive and play audio
  for await (const response of session.receive()) {
    if (response.serverContent?.modelTurn) {
      for (const part of response.serverContent.modelTurn.parts) {
        if (part.inlineData?.data) {
          speaker.write(Buffer.from(part.inlineData.data, "base64"));
        }
      }
    }
  }
}

main();
```

## Text Input/Output Mode

```ts
const session = await ai.live.connect({
  model: "gemini-2.5-flash-native-audio-preview-09-2025",
  config: {
    responseModalities: [Modality.TEXT],
  },
});

await session.send({ text: "Hello, what can you help me with?" });

for await (const response of session.receive()) {
  if (response.serverContent?.modelTurn) {
    for (const part of response.serverContent.modelTurn.parts) {
      if (part.text) process.stdout.write(part.text);
    }
  }
}
```

## Configuration Options

```ts
config: {
  responseModalities: [Modality.AUDIO],   // AUDIO or TEXT
  systemInstruction: "You are helpful.",
  speechConfig: {
    voiceConfig: {
      prebuiltVoiceConfig: { voiceName: "Puck" },
    },
  },
  tools: [{ functionDeclarations: [...] }],  // Function calling supported
}
```

## Key Features

- **Voice Activity Detection (VAD)**: Auto-detects when user starts/stops speaking
- **Interruptions**: User can interrupt model mid-response
- **Tool use**: Function calling works within live sessions
- **Session management**: Resume sessions, handle long conversations
- **Ephemeral tokens**: Secure client-side auth without exposing API keys

## Ephemeral Tokens (Client-side)

For browser-based apps, generate short-lived tokens server-side:

```ts
// Server-side: generate token
const token = await ai.live.createEphemeralToken({
  model: "gemini-2.5-flash-native-audio-preview-09-2025",
  config: { responseModalities: [Modality.AUDIO] },
});
// Send token.token to client

// Client-side: connect with token
const ai = new GoogleGenAI({ apiKey: ephemeralToken });
const session = await ai.live.connect({ model, config });
```

## Audio Format

- Input: 16-bit PCM, 16kHz, mono
- Output: 16-bit PCM, 24kHz, mono

## Limitations

- Session duration limits apply
- Not all models support Live API
- Different from TTS (which is for batch audio generation, not real-time)
