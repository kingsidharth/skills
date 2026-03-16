---
name: milkdown-editor
description: "Build WYSIWYG markdown editors with Milkdown, a plugin-driven framework built on ProseMirror and Remark. Use this skill whenever the user wants to create, customize, or integrate a rich-text markdown editor using Milkdown — whether via the batteries-included Crepe editor or the granular Kit/Core API. Also trigger when the user mentions Milkdown plugins, slash commands, toolbar creation, editor listeners, ProseMirror node views in a Milkdown context, @milkdown/react or @milkdown/crepe, or building WYSIWYG markdown editing into React/Vue/Svelte/vanilla apps. Covers getting started, plugin usage, React integration, keyboard shortcuts, styling, and the slash-plugin example. Even if the user just says 'markdown editor' or 'WYSIWYG editor' alongside mentions of ProseMirror or Remark, consider whether Milkdown is the right tool."
---

# Milkdown — WYSIWYG Markdown Editor Framework

Milkdown is a plugin-driven WYSIWYG markdown editor built on **ProseMirror** (rich-text editing) and **Remark** (markdown parsing/serialization). It provides bidirectional transformation between markdown and a rendered document, inspired by Typora.

Latest version: **7.18.x** · License: MIT · https://milkdown.dev

## When to Read Reference Files

This SKILL.md covers architecture, quick-start, and decision routing. For deeper guidance, read the relevant reference:

| Scenario | Reference File |
|---|---|
| Core API, Ctx system, editor lifecycle, actions, listeners | [references/core-api.md](references/core-api.md) |
| Plugin system, creating custom plugins, official plugin list | [references/plugins.md](references/plugins.md) |
| React integration (Kit + Crepe), Vue, Svelte patterns | [references/framework-integration.md](references/framework-integration.md) |
| Styling, theming, CSS customization, headless approach | [references/styling.md](references/styling.md) |

## Architecture Overview

Milkdown has a strict layered dependency hierarchy:

```
Layer 4 — Application Bundles
  @milkdown/crepe  (batteries-included, builder API, themes)
  @milkdown/kit    (re-exports everything, tree-shakeable)

Layer 3 — Plugins & Presets
  preset-commonmark, preset-gfm
  plugin-history, plugin-clipboard, plugin-listener, plugin-slash, ...
  components (code-block, image-block, table-block, link-tooltip, list-item-block)

Layer 2 — Core
  @milkdown/core        (Editor class, plugin lifecycle, internal plugins)
  @milkdown/transformer  (Markdown ↔ ProseMirror via Remark/unified)

Layer 1 — Foundation
  @milkdown/ctx   (dependency injection: Slices + Timers)
  @milkdown/prose (ProseMirror wrapper, consistent imports)
```

**Data flow**: Markdown string → remark-parse → MDAST → transformer → ProseMirror doc → EditorView → DOM. On edit: ProseMirror doc → transformer → MDAST → remark-stringify → Markdown string.

## Choose Your Approach

Milkdown supports three usage levels. Pick based on how much control you need:

### 1. Crepe (Easiest — batteries included)

Use `@milkdown/crepe` when you want a working editor fast with sensible defaults. Includes block editing, slash commands, CodeMirror code blocks, image upload, tables, LaTeX, placeholders, and theming out of the box.

```bash
npm i @milkdown/crepe
```

```typescript
import { Crepe } from '@milkdown/crepe';
import '@milkdown/crepe/theme/common/style.css';
import '@milkdown/crepe/theme/frame.css';   // or nord.css

const crepe = new Crepe({
  root: document.getElementById('editor'),
  defaultValue: '# Hello Milkdown',
  features: {
    [Crepe.Feature.CodeMirror]: true,
    [Crepe.Feature.BlockEdit]: true,
    [Crepe.Feature.Placeholder]: true,
  },
  featureConfigs: {
    [Crepe.Feature.Placeholder]: {
      text: 'Start typing...',
    },
  },
});

await crepe.create();

// Listen for changes
crepe.on((listener) => {
  listener.markdownUpdated((ctx, markdown, prevMarkdown) => {
    console.log(markdown);
  });
});

// Cleanup
await crepe.destroy();
```

Crepe features you can toggle: `CodeMirror`, `BlockEdit`, `Cursor`, `ImageBlock`, `LinkTooltip`, `ListItem`, `Placeholder`, `Table`, `Latex`, `Toolbar`.

### 2. Kit (Customizable — curated re-exports)

Use `@milkdown/kit` when you want to pick and choose plugins but still prefer a single dependency. It re-exports core, presets, plugins, and components under structured paths.

```bash
npm i @milkdown/kit
```

```typescript
import { Editor, rootCtx, defaultValueCtx } from '@milkdown/kit/core';
import { commonmark } from '@milkdown/kit/preset/commonmark';
import { history } from '@milkdown/kit/plugin/history';
import { listener, listenerCtx } from '@milkdown/kit/plugin/listener';

const editor = await Editor.make()
  .config((ctx) => {
    ctx.set(rootCtx, '#editor');
    ctx.set(defaultValueCtx, '# Hello Milkdown');
  })
  .use(commonmark)
  .use(history)
  .use(listener)
  .create();
```

Kit export paths: `@milkdown/kit/core`, `@milkdown/kit/ctx`, `@milkdown/kit/prose/...`, `@milkdown/kit/preset/commonmark`, `@milkdown/kit/preset/gfm`, `@milkdown/kit/plugin/history`, `@milkdown/kit/plugin/listener`, `@milkdown/kit/plugin/clipboard`, `@milkdown/kit/plugin/cursor`, `@milkdown/kit/plugin/slash`, `@milkdown/kit/plugin/tooltip`, `@milkdown/kit/plugin/block`, `@milkdown/kit/plugin/indent`, `@milkdown/kit/plugin/trailing`, `@milkdown/kit/plugin/upload`, `@milkdown/kit/component/...`.

### 3. Granular (Maximum control)

Install individual packages when you need precise dependency management:

```bash
npm i @milkdown/core @milkdown/prose @milkdown/ctx @milkdown/transformer
npm i @milkdown/preset-commonmark
npm i @milkdown/plugin-history @milkdown/plugin-listener
```

Same API as Kit, just different import paths: `import { Editor } from '@milkdown/core'`.

## Core Concepts (Quick Reference)

### Editor Lifecycle

```typescript
const editor = Editor.make()   // Create builder
  .config((ctx) => { ... })    // Configure context slices
  .use(pluginA)                // Register plugins
  .use(pluginB);

await editor.create();         // Initialize (async)
await editor.destroy();        // Cleanup
await editor.create();         // Can recreate after destroy
```

**Status**: `Idle` → `OnCreate` → `Created` → `OnDestroyed` → `Destroyed`. Monitor with `editor.onStatusChange(cb)`.

### Context System (Ctx)

Ctx is the dependency injection container. Plugins communicate through typed **Slices** (key-value stores) and **Timers** (execution ordering).

```typescript
editor.config((ctx) => {
  ctx.set(rootCtx, '#editor');                    // Set a slice
  ctx.set(defaultValueCtx, '# Hello');            // Set default content
  ctx.update(editorViewOptionsCtx, (prev) => ({   // Update a slice
    ...prev,
    editable: () => !readOnly,
  }));
});
```

### Listening for Changes

Use the listener plugin to get markdown or doc updates:

```typescript
import { listener, listenerCtx } from '@milkdown/kit/plugin/listener';

editor.config((ctx) => {
  ctx.get(listenerCtx).markdownUpdated((ctx, markdown, prevMarkdown) => {
    if (markdown !== prevMarkdown) saveContent(markdown);
  });
});
editor.use(listener);
```

Also available: `.updated((ctx, doc, prevDoc) => { ... })` for ProseMirror doc JSON.

### Actions (Programmatic Control)

Execute operations against the editor after it's created:

```typescript
import { editorViewCtx, serializerCtx, parserCtx } from '@milkdown/kit/core';
import { insert, replaceAll } from '@milkdown/kit/utils';

// Get current markdown
const markdown = editor.action((ctx) => {
  const view = ctx.get(editorViewCtx);
  const serializer = ctx.get(serializerCtx);
  return serializer(view.state.doc);
});

// Replace all content
editor.action(replaceAll('# New Content'));

// Insert at cursor
editor.action(insert('**bold text**'));
```

### Default Values

Three supported types — markdown string, HTML DOM, or ProseMirror JSON:

```typescript
// Markdown (most common)
ctx.set(defaultValueCtx, '# Hello');

// HTML
ctx.set(defaultValueCtx, { type: 'html', dom: document.querySelector('#source') });

// JSON (from listener output)
ctx.set(defaultValueCtx, { type: 'json', value: savedDoc });
```

### Readonly Mode

```typescript
import { editorViewOptionsCtx } from '@milkdown/kit/core';

editor.config((ctx) => {
  ctx.update(editorViewOptionsCtx, (prev) => ({
    ...prev,
    editable: () => false,
  }));
});
```

## Official Plugins

| Package | Import via Kit | Purpose |
|---|---|---|
| `preset-commonmark` | `@milkdown/kit/preset/commonmark` | CommonMark syntax (paragraphs, headings, lists, code, etc.) |
| `preset-gfm` | `@milkdown/kit/preset/gfm` | GitHub Flavored Markdown (tables, strikethrough, task lists, autolinks) |
| `plugin-history` | `@milkdown/kit/plugin/history` | Undo/redo |
| `plugin-clipboard` | `@milkdown/kit/plugin/clipboard` | Markdown-aware copy/paste |
| `plugin-listener` | `@milkdown/kit/plugin/listener` | Event listeners (markdownUpdated, updated, mounted, etc.) |
| `plugin-cursor` | `@milkdown/kit/plugin/cursor` | Drop cursor and gap cursor |
| `plugin-slash` | `@milkdown/kit/plugin/slash` | Slash command menu |
| `plugin-tooltip` | `@milkdown/kit/plugin/tooltip` | Selection tooltip |
| `plugin-block` | `@milkdown/kit/plugin/block` | Block-level drag/manipulation |
| `plugin-indent` | `@milkdown/kit/plugin/indent` | Tab indentation |
| `plugin-trailing` | `@milkdown/kit/plugin/trailing` | Trailing paragraph node |
| `plugin-upload` | `@milkdown/kit/plugin/upload` | File upload handling |
| `plugin-collab` | (separate install) | Y.js collaborative editing |
| `plugin-prism` | (separate install) | Prism.js code highlighting |
| `plugin-math` | (separate install) | LaTeX math via KaTeX |
| `plugin-emoji` | (separate install) | Emoji support with twemoji |

## Keyboard Shortcuts (CommonMark Preset)

The commonmark preset registers these keybindings by default:

| Shortcut | Action |
|---|---|
| `Mod-b` | Toggle bold |
| `Mod-i` | Toggle italic |
| `Mod-e` | Toggle inline code |
| `Mod-Shift-s` | Toggle strikethrough (GFM) |
| `Mod-Alt-1` through `Mod-Alt-6` | Heading levels 1–6 |
| `Mod-Shift-7` | Toggle ordered list |
| `Mod-Shift-8` | Toggle bullet list |
| `Mod-Shift-9` | Toggle task list (GFM) |
| `Mod-]` | Sink list item |
| `Mod-[` | Lift list item |
| `Enter` | Split list item / new paragraph |
| `Tab` | Indent in code block / list |
| `Shift-Tab` | Dedent in code block / list |
| `Mod-z` | Undo (with history plugin) |
| `Mod-y` / `Mod-Shift-z` | Redo (with history plugin) |

`Mod` = Cmd on macOS, Ctrl on Windows/Linux.

## Common Patterns

### Getting markdown out of the editor

```typescript
// Via listener (reactive)
ctx.get(listenerCtx).markdownUpdated((ctx, md) => { save(md); });

// Via action (imperative)
const md = editor.action((ctx) => {
  const view = ctx.get(editorViewCtx);
  return ctx.get(serializerCtx)(view.state.doc);
});
```

### Programmatically replacing content

```typescript
import { replaceAll } from '@milkdown/kit/utils';
editor.action(replaceAll('# New Document\n\nContent here.'));
```

### Adding commands

```typescript
import { commandsCtx } from '@milkdown/kit/core';
import { createCmdKey } from '@milkdown/kit/core';

const MyCommand = createCmdKey<string>('myCommand');

// In a plugin or config:
ctx.get(commandsCtx).create(MyCommand, () => (state, dispatch) => {
  // ProseMirror command logic
  return true;
});

// Execute:
import { callCommand } from '@milkdown/kit/utils';
editor.action(callCommand(MyCommand, 'payload'));
```

For framework integration (React, Vue, Svelte), plugin development, slash commands, and styling — see the reference files linked above.
