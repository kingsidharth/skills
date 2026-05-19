# Advanced Features

## Resources

Resources are **application-driven** data sources (unlike tools, which are model-controlled). Your app decides when to fetch and inject them as context.

```ts
// List available resources
const resources = await client.listResources();

// Read a specific resource
const data = await client.readResource({ uri: 'file:///example/document.txt' });

// List resource templates (dynamic URI patterns)
const templates = await client.listResourceTemplates();
```

Typical use: fetch a resource, include its content in the prompt messages array.

## Prompts (Experimental)

Server-exposed prompt templates with optional arguments:

```ts
// List available prompts
const prompts = await client.experimental_listPrompts();

// Get a prompt with arguments
const prompt = await client.experimental_getPrompt({
  name: 'code_review',
  arguments: { code: 'function add(a, b) { return a + b; }' },
});
```

Returns messages array ready for use in `generateText`/`streamText`.

## Elicitation

Mechanism for servers to request additional input from the client mid-tool-execution (e.g., confirmation, form data).

### Enable capability at client creation:

```ts
const client = await createMCPClient({
  transport: { type: 'sse', url: 'https://your-server.com/sse' },
  capabilities: { elicitation: {} },
});
```

### Register a handler:

```ts
import { ElicitationRequestSchema } from '@ai-sdk/mcp';

client.onElicitationRequest(ElicitationRequestSchema, async (request) => {
  // request.params.message — describes what's needed
  // request.params.requestedSchema — JSON Schema for expected input

  const userInput = await collectInput(request.params.requestedSchema);

  return {
    action: 'accept',   // 'accept' | 'decline' | 'cancel'
    content: userInput,  // required when action is 'accept'
  };
});
```

### Response actions:

| Action | Meaning | `content` required? |
|---|---|---|
| `accept` | User provided the data | Yes |
| `decline` | User chose not to provide | No |
| `cancel` | User cancelled the operation | No |

The MCP client surfaces requests — your application handles the UX (CLI readline, web form, dialog, etc.).

## OAuth Authentication

Both HTTP and SSE transports support OAuth via `authProvider`:

```ts
const client = await createMCPClient({
  transport: {
    type: 'http',
    url: 'https://server.com/mcp',
    authProvider: myOAuthClientProvider,  // handles PKCE, token refresh, etc.
  },
});
```

Implement the OAuth client provider interface per your auth flow. The SDK handles token injection into requests.
