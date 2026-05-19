# Agentic Tools & `.act()`

## Defining a tool

```ts
import { tool } from "@lmstudio/sdk";
import { z } from "zod";

const addTool = tool({
  name: "add",
  description: "Returns the sum of two numbers.",
  parameters: { a: z.number(), b: z.number() },
  implementation: ({ a, b }) => a + b,
});
```

- `name`, `description`, and `parameters` are all sent to the model — wording affects generation quality
- `implementation` is a regular (optionally async) function

## Running an agent with `.act()`

```ts
await model.act(
  "Create a file with your understanding of life.",
  [createFileTool],
);
```

`.act()` lets the model autonomously call tools in a loop until it decides it's done. Tools can have external effects (file I/O, API calls, etc.), turning the LLM into a local agent.

## Tool with side effects example

```ts
const createFileTool = tool({
  name: "createFile",
  description: "Create a file with the given name and content.",
  parameters: { name: z.string(), content: z.string() },
  implementation: async ({ name, content }) => {
    if (existsSync(name)) return "Error: File already exists.";
    await writeFile(name, content, "utf-8");
    return "File created.";
  },
});
```
