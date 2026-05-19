# Prompts

Prompts are reusable message templates with typed arguments. Clients display them as slash-commands or quick-actions.

## Basic prompt

```typescript
server.registerPrompt(
  "review-code",
  {
    title: "Code Review",
    description: "Review code for best practices and issues",
    argsSchema: { code: z.string(), language: z.string().optional() },
  },
  ({ code, language }) => ({
    messages: [{
      role: "user",
      content: {
        type: "text",
        text: `Review this ${language ?? ""} code for issues:\n\n${code}`,
      },
    }],
  })
);
```

## Prompt with auto-completion

```typescript
import { completable } from "@modelcontextprotocol/sdk/server/completable.js";

server.registerPrompt(
  "team-greeting",
  {
    title: "Team Greeting",
    argsSchema: {
      department: completable(z.string(), (value) =>
        ["engineering", "sales", "marketing"].filter(d => d.startsWith(value))
      ),
      name: completable(z.string(), (value, context) => {
        const dept = context?.arguments?.["department"];
        const names = dept === "engineering" ? ["Alice", "Bob"] : ["Carol", "Dave"];
        return names.filter(n => n.startsWith(value));
      }),
    },
  },
  ({ department, name }) => ({
    messages: [{
      role: "user",
      content: { type: "text", text: `Write a greeting for ${name} in ${department}` },
    }],
  })
);
```

Completions let clients offer typeahead suggestions as the user fills in prompt arguments.
