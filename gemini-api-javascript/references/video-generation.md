# Video Generation (Veo 3.1)

Veo 3.1 generates 8-second 720p/1080p/4K videos with natively generated audio (dialogue, sound effects).

## Text to Video

Video generation is asynchronous — you submit a request and poll for completion:

```ts
import { GoogleGenAI } from "@google/genai";

const ai = new GoogleGenAI({});

let operation = await ai.models.generateVideos({
  model: "veo-3.1-generate-preview",
  prompt: `A close up of two people staring at a cryptic drawing on a wall, torchlight flickering.
A man murmurs, 'This must be it.' The woman whispers excitedly, 'What did you find?'`,
});

// Poll until done
while (!operation.done) {
  console.log("Waiting for video generation...");
  await new Promise((r) => setTimeout(r, 10000));
  operation = await ai.operations.getVideosOperation({ operation });
}

// Download
ai.files.download({
  file: operation.response.generatedVideos[0].video,
  downloadPath: "output.mp4",
});
```

## Aspect Ratio

```ts
let operation = await ai.models.generateVideos({
  model: "veo-3.1-generate-preview",
  prompt: "A pizza-making montage with upbeat music",
  config: {
    aspectRatio: "9:16",  // "16:9" (default) or "9:16"
  },
});
```

## Resolution

```ts
config: {
  resolution: "4k",  // "720p" (default), "1080p", "4k"
}
```

Higher resolution = higher latency and cost. Video extension is limited to 720p.

## Image to Video

Use a generated or uploaded image as the starting frame:

```ts
// Step 1: Generate an image with Nano Banana
const imageResponse = await ai.models.generateContent({
  model: "gemini-2.5-flash-image",
  prompt: "A calico kitten sleeping in sunshine",
  config: { responseModalities: ["IMAGE"] },
});

// Step 2: Use as first frame for Veo
let operation = await ai.models.generateVideos({
  model: "veo-3.1-generate-preview",
  prompt: "Panning wide shot of a calico kitten sleeping in the sunshine",
  image: {
    imageBytes: imageResponse.generatedImages[0].image.imageBytes,
    mimeType: "image/png",
  },
});
```

## Video Extension

Extend a previously generated Veo video (720p only):

```ts
let operation = await ai.models.generateVideos({
  model: "veo-3.1-generate-preview",
  prompt: "The camera continues to pan revealing a mountain range",
  config: {
    extendedVideo: previousVideoFile,  // From prior generation
  },
});
```

## Frame-specific Generation

Specify first and/or last frames to control the video's start and end:

```ts
let operation = await ai.models.generateVideos({
  model: "veo-3.1-generate-preview",
  prompt: "A sunrise timelapse over a city",
  image: firstFrameImage,   // First frame
  config: {
    lastFrame: lastFrameImage,  // Last frame
  },
});
```

## Prompt Guide

Effective Veo prompts should describe: subject, action, scene/setting, camera movement, style, and audio.

Example: `"A cinematic drone shot of a coastal village at golden hour, waves crashing against rocky shores, seagulls calling in the distance, shot on 35mm film"`
