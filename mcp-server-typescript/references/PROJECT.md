# Project Setup

## Structure

```
{service}-mcp-server/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          # Entry point, transport setup
│   ├── server.ts         # McpServer instance, tool/resource registration
│   ├── types.ts          # TypeScript type definitions
│   ├── constants.ts      # API_BASE_URL, CHARACTER_LIMIT, etc.
│   ├── tools/            # Tool handlers (one file per domain)
│   ├── services/         # API clients, shared utilities
│   └── schemas/          # Zod validation schemas
└── dist/                 # Build output (entry: dist/index.js)
```

## package.json

```json
{
  "name": "{service}-mcp-server",
  "version": "1.0.0",
  "description": "MCP server for {Service}",
  "type": "module",
  "main": "dist/index.js",
  "bin": { "{service}-mcp": "dist/index.js" },
  "scripts": {
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "inspect": "npx @modelcontextprotocol/inspector node dist/index.js"
  },
  "engines": { "node": ">=18" },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.11.0",
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "tsx": "^4.19.0",
    "typescript": "^5.7.0"
  }
}
```

Add `express` and `@types/express` if using Streamable HTTP. Add `axios` if making HTTP API calls.

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## Naming

- **Package**: `{service}-mcp-server` (e.g., `github-mcp-server`)
- **Tools**: `{service}_{action}_{resource}` in snake_case (e.g., `slack_send_message`)
- **Zod schemas**: PascalCase with `Schema` suffix (e.g., `UserSearchInputSchema`)

## Quality checklist

### Design
- [ ] Tools named with service prefix to avoid collision
- [ ] Tool descriptions include parameter semantics, return shape, error cases
- [ ] Annotations set correctly (`readOnlyHint`, `destructiveHint`, etc.)
- [ ] Common operations extracted into reusable functions (API client, error handler, formatters)
- [ ] Pagination supported where applicable
- [ ] `CHARACTER_LIMIT` enforced on large responses

### TypeScript
- [ ] `strict: true` in tsconfig
- [ ] No `any` types — use `unknown` or proper types
- [ ] Zod schemas with constraints and `.describe()` on fields
- [ ] All async functions typed with `Promise<T>`
- [ ] Error handling uses type guards (`instanceof AxiosError`, `z.ZodError`)

### Build & test
- [ ] `npm run build` succeeds cleanly
- [ ] `dist/index.js` exists and runs
- [ ] Tested with MCP Inspector (`npx @modelcontextprotocol/inspector`)
- [ ] All imports resolve correctly
- [ ] Environment variables validated at startup with clear error messages

### Transport & auth
- [ ] Transport appropriate for deployment (stdio for local, HTTP for remote)
- [ ] Auth level appropriate for exposure (env var → bearer → OAuth)
- [ ] CORS and DNS rebinding protection for HTTP servers
- [ ] `127.0.0.1` binding for local servers (not `0.0.0.0`)
