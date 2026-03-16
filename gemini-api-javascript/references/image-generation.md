# Image Generation (Nano Banana)

"Nano Banana" is the name for Gemini's native image generation. Three models available:

| Model | ID | Strengths |
|---|---|---|
| Nano Banana 2 | `gemini-3.1-flash-image-preview` | Fast, high-volume, 512px–4K, image search grounding |
| Nano Banana Pro | `gemini-3-pro-image-preview` | Professional quality, thinking-powered, 1K–4K |
| Nano Banana (2.5) | `gemini-2.5-flash-image` | Speed-optimized for high-volume |

## Text to Image

```ts
import { GoogleGenAI } from "@google/genai";
import * as fs from "node:fs";

const ai = new GoogleGenAI({});

const response = await ai.models.generateContent({
  model: "gemini-3.1-flash-image-preview",
  contents: "A futuristic cityscape at sunset, cyberpunk style",
});

for (const part of response.candidates[0].content.parts) {
  if (part.inlineData) {
    const buffer = Buffer.from(part.inlineData.data, "base64");
    fs.writeFileSync("output.png", buffer);
  } else if (part.text) {
    console.log(part.text);
  }
}
```

## Image Editing (Image + Text → Image)

```ts
const imageData = fs.readFileSync("input.png").toString("base64");

const response = await ai.models.generateContent({
  model: "gemini-3.1-flash-image-preview",
  contents: [
    { text: "Make this photo look like a watercolor painting" },
    { inlineData: { mimeType: "image/png", data: imageData } },
  ],
});
```

## Multi-turn Editing (Chat)

Use `ai.chats.create()` for conversational editing — the SDK handles thought signatures automatically:

```ts
const chat = ai.chats.create({
  model: "gemini-3.1-flash-image-preview",
  config: {
    responseModalities: ["TEXT", "IMAGE"],
    tools: [{ googleSearch: {} }],
  },
});

// Generate initial image
let response = await chat.sendMessage({
  message: "Create a vibrant infographic about photosynthesis",
});
// Save the image from response...

// Edit in follow-up turn
response = await chat.sendMessage({
  message: "Update this infographic to be in Spanish",
  config: {
    imageConfig: { aspectRatio: "16:9", imageSize: "2K" },
  },
});
```

## Aspect Ratios and Image Size

Set via `imageConfig`:

```ts
config: {
  imageConfig: {
    aspectRatio: "16:9",  // 1:1, 1:4, 1:8, 2:3, 3:2, 3:4, 4:1, 4:3, 4:5, 5:4, 8:1, 9:16, 16:9, 21:9
    imageSize: "2K",      // "512px", "1K", "2K", "4K"
  },
}
```

- `512px` only available on Nano Banana 2 (`gemini-3.1-flash-image-preview`)
- `1:4`, `4:1`, `1:8`, `8:1` ratios only on Nano Banana 2

## Multiple Reference Images

Gemini 3 image models accept up to 14 reference images (mix of objects and characters):

```ts
const response = await ai.models.generateContent({
  model: "gemini-3.1-flash-image-preview",
  contents: [
    { text: "An office group photo of these people, making funny faces." },
    { inlineData: { mimeType: "image/jpeg", data: person1Base64 } },
    { inlineData: { mimeType: "image/jpeg", data: person2Base64 } },
    { inlineData: { mimeType: "image/jpeg", data: person3Base64 } },
  ],
  config: {
    responseModalities: ["TEXT", "IMAGE"],
    imageConfig: { aspectRatio: "5:4", imageSize: "2K" },
  },
});
```

## Google Search Grounding for Images

Enable search grounding to generate images based on real-time data:

```ts
const response = await ai.models.generateContent({
  model: "gemini-3-pro-image-preview",
  contents: "Generate a visualization of the current weather in Tokyo.",
  config: {
    tools: [{ googleSearch: {} }],
    imageConfig: { aspectRatio: "16:9", imageSize: "4K" },
  },
});
```

## Key Notes

- All generated images include SynthID watermarks
- Nano Banana Pro uses "thinking" to reason through complex prompts
- For multi-turn editing, thought signatures are strictly enforced — use chat API
- `responseModalities: ["TEXT", "IMAGE"]` is needed for explicit image output control
