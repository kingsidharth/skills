# Batch API

Process large volumes of requests asynchronously at 50% of standard cost. Target turnaround: 24 hours (often faster).

## Inline Requests (Small Batches)

```ts
import { GoogleGenAI } from "@google/genai";

const ai = new GoogleGenAI({});

const inlinedRequests = [
  { contents: [{ parts: [{ text: "Tell me a one-sentence joke." }], role: "user" }] },
  { contents: [{ parts: [{ text: "Why is the sky blue?" }], role: "user" }] },
];

const batchJob = await ai.batches.create({
  model: "gemini-3-flash-preview",
  src: inlinedRequests,
  config: { displayName: "my-batch-job" },
});

console.log(`Created batch job: ${batchJob.name}`);
```

## File-based Requests (Large Batches)

Create a JSONL file where each line is `{ key, request }`:

```jsonl
{"key": "req-1", "request": {"contents": [{"parts": [{"text": "Describe photosynthesis."}]}]}}
{"key": "req-2", "request": {"contents": [{"parts": [{"text": "What's in a Margherita pizza?"}]}]}}
```

Upload and create batch:

```ts
import * as fs from "node:fs";

// Upload JSONL file
const uploadedFile = await ai.files.upload({
  file: "my-batch-requests.jsonl",
  config: { displayName: "my-batch-requests", mimeType: "jsonl" },
});

// Create batch job from file
const batchJob = await ai.batches.create({
  model: "gemini-3-flash-preview",
  src: uploadedFile,
  config: { displayName: "file-based-batch" },
});
```

## Check Status and Get Results

```ts
// Poll for completion
let job = await ai.batches.get({ name: batchJob.name });
while (job.state !== "JOB_STATE_SUCCEEDED" && job.state !== "JOB_STATE_FAILED") {
  await new Promise((r) => setTimeout(r, 30000));
  job = await ai.batches.get({ name: batchJob.name });
  console.log(`Status: ${job.state}`);
}

// For inline requests: results in job.dest.inlineResponses
// For file requests: results in output JSONL file
```

## List and Delete Batch Jobs

```ts
const jobs = await ai.batches.list();
await ai.batches.delete({ name: "batches/abc123" });
```

## Key Notes

- Max input file size: 2GB
- Max 20MB for inline requests
- Each request in JSONL can have its own system instructions, tools, and generation config
- Supports multimodal inputs (reference uploaded files in JSONL)
- 50% cost reduction vs standard API
- Output preserves `key` for matching responses to requests
