# Embeddings

Generate text embeddings for semantic search, classification, and clustering.

## Single Text

```ts
import { GoogleGenAI } from "@google/genai";

const ai = new GoogleGenAI({});

const response = await ai.models.embedContent({
  model: "gemini-embedding-001",
  contents: "What is the meaning of life?",
});

console.log(response.embeddings); // Array of embedding objects
```

## Batch Embedding

```ts
const response = await ai.models.embedContent({
  model: "gemini-embedding-001",
  contents: [
    "What is the meaning of life?",
    "What is the purpose of existence?",
    "How do I bake a cake?",
  ],
});

for (const embedding of response.embeddings) {
  console.log(embedding.values.length); // 3072 dimensions
}
```

## Task Types

Specify `taskType` to optimize embeddings for your use case:

```ts
const response = await ai.models.embedContent({
  model: "gemini-embedding-001",
  contents: "Machine learning fundamentals",
  config: {
    taskType: "RETRIEVAL_DOCUMENT",  // Optimize for document retrieval
  },
});
```

| Task Type | Use Case |
|---|---|
| `RETRIEVAL_QUERY` | Search query embedding |
| `RETRIEVAL_DOCUMENT` | Document to be searched |
| `SEMANTIC_SIMILARITY` | Comparing text similarity |
| `CLASSIFICATION` | Text classification |
| `CLUSTERING` | Grouping similar texts |
| `QUESTION_ANSWERING` | Answer a question |
| `FACT_VERIFICATION` | Verify facts |
| `CODE_RETRIEVAL_QUERY` | Code search query |

## Output Dimensionality

Reduce dimensions for storage efficiency:

```ts
const response = await ai.models.embedContent({
  model: "gemini-embedding-001",
  contents: "Some text",
  config: {
    outputDimensionality: 768,  // Reduce from default 3072
  },
});
```

## Model

`gemini-embedding-001`: Default 3072 dimensions, 8192 token input limit. Supports all task types.
