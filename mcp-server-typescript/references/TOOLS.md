# Tools

Tools let LLMs invoke server-side functions with validated inputs and structured outputs.

## Registration

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "my-server", version: "1.0.0" });

server.registerTool(
  "search_users",
  {
    title: "Search Users",
    description: "Find users by name or email. Returns paginated results.",
    inputSchema: {
      query: z.string().min(2).describe("Search term"),
      limit: z.number().int().min(1).max(100).default(20),
      offset: z.number().int().min(0).default(0),
    },
    outputSchema: {
      total: z.number(),
      users: z.array(z.object({ id: z.string(), name: z.string() })),
      has_more: z.boolean(),
    },
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true,
    },
  },
  async ({ query, limit, offset }) => {
    const data = await api.searchUsers(query, limit, offset);
    const output = {
      total: data.total,
      users: data.users.map(u => ({ id: u.id, name: u.name })),
      has_more: data.total > offset + data.users.length,
    };
    return {
      content: [{ type: "text", text: JSON.stringify(output, null, 2) }],
      structuredContent: output,
    };
  }
);
```

## Key conventions

- **Name**: `snake_case`, prefixed with service name (`github_create_issue`, not `create_issue`)
- **inputSchema**: Zod object with `.describe()` on each field. Add `.strict()` to forbid extra fields.
- **outputSchema**: optional but recommended — enables `structuredContent` in response
- **structuredContent**: return alongside `content` for machine-parseable output. Use a `type` alias, not an `interface`, for the output shape (interfaces lack implicit index signatures).
- **Annotations**: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` — hints for clients, not security guarantees
- **Description**: include what the tool does, parameter semantics, return schema shape, and error cases. The description is the *only* thing the LLM sees to decide which tool to call.

## Error handling

Return errors inside the result, not as protocol-level exceptions:

```typescript
async (params) => {
  try {
    const result = await doWork(params);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  } catch (error) {
    return {
      isError: true,
      content: [{
        type: "text",
        text: `Error: ${error instanceof Error ? error.message : String(error)}. Try narrowing your query.`
      }],
    };
  }
}
```

## Pagination

For list endpoints, always support `limit`/`offset` and return `has_more` + `next_offset`:

```typescript
const output = {
  total: data.total,
  count: items.length,
  offset: params.offset,
  items,
  has_more: data.total > params.offset + items.length,
  ...(data.total > params.offset + items.length
    ? { next_offset: params.offset + items.length }
    : {}),
};
```

## Character limits

Define a `CHARACTER_LIMIT` constant (e.g., 25000). If a response exceeds it, truncate items and add a `truncation_message` telling the caller to paginate or filter.

## Sampling (server-initiated LLM calls)

A tool handler can request an LLM completion from the connected client:

```typescript
async ({ text }) => {
  const response = await server.server.createMessage({
    messages: [{ role: "user", content: { type: "text", text: `Summarize: ${text}` } }],
    maxTokens: 500,
  });
  return {
    content: [{
      type: "text",
      text: response.content.type === "text" ? response.content.text : "Failed",
    }],
  };
}
```

Requires client to declare the `sampling` capability.
