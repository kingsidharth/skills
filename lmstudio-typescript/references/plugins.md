# Plugins (Beta)

Plugins extend LM Studio via hook functions that execute at specific points during operation. Written in TypeScript, run on Node.js v22.21.1.

## Plugin types

- **Tools Provider** — gives models extra capabilities (API access, calculations)
- **Prompt Preprocessor** — modifies user input before it reaches the model (file handling, context injection)
- **Generator** — custom text generation source replacing the local model (online model adapters)

## Development workflow

```bash
lms dev          # start dev mode (auto-rebuild on change)
```

Plugin appears in LM Studio's plugin list during dev. Installed plugins run automatically.

## Tools Provider pattern

```ts
import { tool, Tool, ToolsProviderController } from "@lmstudio/sdk";
import { z } from "zod";

export async function toolsProvider(ctl: ToolsProviderController) {
  const tools: Tool[] = [];

  tools.push(tool({
    name: "create_file",
    description: "Create a file with the given name and content.",
    parameters: { file_name: z.string(), content: z.string() },
    implementation: async ({ file_name, content }) => {
      const filePath = join(ctl.getWorkingDirectory(), file_name);
      if (existsSync(filePath)) return "Error: File already exists.";
      await writeFile(filePath, content, "utf-8");
      return "File created.";
    },
  }));

  return tools;
}
```

## Custom configuration

Define per-chat or global config fields via `createConfigSchematics()`:

```ts
export const configSchematics = createConfigSchematics()
  .field("folderName", "string", {
    displayName: "Folder Name",
    subtitle: "Where files will be created.",
  }, "default_folder")
  .build();
```

Read config inside tools:

```ts
const folderName = ctl.getPluginConfig(configSchematics).get("folderName");
```

## Distribution

```bash
lms push          # upload to LM Studio Hub
lms clone <name>  # clone from Hub
```
