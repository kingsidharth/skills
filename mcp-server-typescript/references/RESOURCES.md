# Resources

Resources expose read-only data via URI-based access. Clients can list, read, and subscribe to changes.

## Static resource

```typescript
server.registerResource(
  "config",
  "config://app",
  { title: "App Config", description: "Current application configuration", mimeType: "application/json" },
  async (uri) => ({
    contents: [{ uri: uri.href, mimeType: "application/json", text: JSON.stringify(getConfig()) }],
  })
);
```

## Dynamic resource (URI template)

```typescript
import { ResourceTemplate } from "@modelcontextprotocol/sdk/server/mcp.js";

server.registerResource(
  "user-profile",
  new ResourceTemplate("users://{userId}/profile", { list: undefined }),
  { title: "User Profile", description: "User profile by ID", mimeType: "application/json" },
  async (uri, { userId }) => ({
    contents: [{
      uri: uri.href,
      mimeType: "application/json",
      text: JSON.stringify(await getUser(userId)),
    }],
  })
);
```

## When to use resources vs tools

- **Resources**: data access with simple URI parameters, relatively static, no side effects
- **Tools**: complex operations requiring validation, business logic, or side effects
