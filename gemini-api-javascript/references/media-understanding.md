# Media Understanding

Gemini models can analyze images, video, audio, and documents as input.

## Image Understanding

### Inline base64

```ts
import { GoogleGenAI } from "@google/genai";
import * as fs from "node:fs";

const ai = new GoogleGenAI({});
const imageData = fs.readFileSync("photo.jpg").toString("base64");

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [
    { text: "What's in this image?" },
    { inlineData: { mimeType: "image/jpeg", data: imageData } },
  ],
});
console.log(response.text);
```

### From URL (fetch + base64)

```ts
const imageUrl = "https://example.com/photo.jpg";
const imageResponse = await fetch(imageUrl);
const arrayBuffer = await imageResponse.arrayBuffer();
const base64Data = Buffer.from(arrayBuffer).toString("base64");

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [
    { inlineData: { mimeType: "image/jpeg", data: base64Data } },
    { text: "Describe this image in detail" },
  ],
});
```

### Using Files API for large files

```ts
// Upload
const uploadedFile = await ai.files.upload({
  file: "large-image.png",
  config: { mimeType: "image/png" },
});

// Wait for processing
let file = await ai.files.get({ name: uploadedFile.name });
while (file.state === "PROCESSING") {
  await new Promise((r) => setTimeout(r, 2000));
  file = await ai.files.get({ name: uploadedFile.name });
}

// Use in generation
const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [
    { fileData: { fileUri: file.uri, mimeType: file.mimeType } },
    { text: "Analyze this image" },
  ],
});
```

## Video Understanding

Upload video via Files API (inline base64 not practical for video):

```ts
const uploadedVideo = await ai.files.upload({
  file: "video.mp4",
  config: { mimeType: "video/mp4" },
});

// Wait for ACTIVE state
let file = await ai.files.get({ name: uploadedVideo.name });
while (file.state === "PROCESSING") {
  await new Promise((r) => setTimeout(r, 5000));
  file = await ai.files.get({ name: uploadedVideo.name });
}

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [
    { fileData: { fileUri: file.uri, mimeType: "video/mp4" } },
    { text: "Summarize this video and describe key moments" },
  ],
});
```

## Audio Understanding

```ts
const uploadedAudio = await ai.files.upload({
  file: "podcast.mp3",
  config: { mimeType: "audio/mp3" },
});

// Wait for processing...

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [
    { fileData: { fileUri: uploadedAudio.uri, mimeType: "audio/mp3" } },
    { text: "Transcribe this audio and summarize the key points" },
  ],
});
```

## Document Processing (PDF)

```ts
const uploadedPdf = await ai.files.upload({
  file: "report.pdf",
  config: { mimeType: "application/pdf" },
});

// Wait for processing...

const response = await ai.models.generateContent({
  model: "gemini-3-flash-preview",
  contents: [
    { fileData: { fileUri: uploadedPdf.uri, mimeType: "application/pdf" } },
    { text: "Extract all tables and key findings from this document" },
  ],
});
```

## Media Resolution Control (Gemini 3)

Control token usage per image/video frame with `mediaResolution`:

```ts
// Requires v1alpha API version
const ai = new GoogleGenAI({ apiVersion: "v1alpha" });

const response = await ai.models.generateContent({
  model: "gemini-3-pro-preview",
  contents: [
    {
      parts: [
        { text: "What is in this image?" },
        {
          inlineData: { mimeType: "image/jpeg", data: base64Data },
          mediaResolution: { level: "media_resolution_high" },
        },
      ],
    },
  ],
});
```

Levels: `media_resolution_low` (280 tokens/image), `media_resolution_medium` (560), `media_resolution_high` (1120), `media_resolution_ultra_high` (per-part only).

Recommendations: `high` for images, `medium` for PDFs, `low`/`medium` for video (70 tokens/frame), `high` for text-heavy video (280 tokens/frame).

## Files API Management

```ts
// List uploaded files
const files = await ai.files.list();

// Delete a file
await ai.files.delete({ name: "files/abc123" });
```

Files are automatically deleted after 48 hours. Supported MIME types include: image/jpeg, image/png, image/gif, image/webp, video/mp4, video/mpeg, video/mov, video/avi, video/webm, audio/mp3, audio/wav, audio/aac, audio/flac, audio/ogg, application/pdf, text/plain, text/csv, text/html.
