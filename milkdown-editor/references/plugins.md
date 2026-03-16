# Plugin Reference

Milkdown is plugin-driven — almost all functionality (including the schema itself) is provided by plugins. This reference covers using official plugins, plugin architecture, and building custom plugins including the slash command example.

## Using Plugins

Register plugins with `.use()` on the editor builder. Plugins can be single items or arrays:

```typescript
import { commonmark } from '@milkdown/kit/preset/commonmark';
import { gfm } from '@milkdown/kit/preset/gfm';
import { history } from '@milkdown/kit/plugin/history';
import { clipboard } from '@milkdown/kit/plugin/clipboard';

const editor = await Editor.make()
  .config((ctx) => { ctx.set(rootCtx, '#editor'); })
  .use(commonmark)       // Preset (array of plugins)
  .use(gfm)              // Another preset
  .use(history)          // Individual plugin
  .use(clipboard)        // Individual plugin
  .create();
```

Remove plugins dynamically (before or after creation):

```typescript
editor.remove(gfm);
await editor.create();  // Recreates without GFM
```

## Official Plugins — Detail

### Presets

**`preset-commonmark`** — the baseline. Provides paragraph, heading (h1–h6), blockquote, code block, code inline, emphasis, strong, link, image, horizontal rule, hard break, ordered list, bullet list, list item. Registers corresponding input rules (e.g., `#` → heading, `>` → blockquote, ``` → code block) and keyboard shortcuts.

**`preset-gfm`** — extends CommonMark. Adds strikethrough, tables, task lists, and autolinks. Install separately or via Kit: `@milkdown/kit/preset/gfm`.

Both presets are arrays of plugins internally, so `.use(commonmark)` registers ~20+ individual plugins at once.

### Behavior Plugins

**`plugin-history`** — Undo/redo via ProseMirror history. Keybinds: `Mod-z` (undo), `Mod-y` / `Mod-Shift-z` (redo).

**`plugin-clipboard`** — Markdown-aware copy/paste. When you copy text, the clipboard contains both the HTML representation and the original markdown. Pasting markdown into the editor parses it correctly.

**`plugin-cursor`** — Adds drop cursor (visual indicator when dragging) and gap cursor (allows clicking between block-level nodes). 

**`plugin-indent`** — Tab/Shift-Tab for indentation in code blocks and lists.

**`plugin-trailing`** — Ensures there's always a trailing paragraph at the end of the document (prevents the cursor from getting stuck after the last block node).

### UI Plugins

**`plugin-listener`** — Event system. See [core-api.md](core-api.md) for full usage.

**`plugin-tooltip`** — Floating toolbar that appears on text selection. You provide the UI component; Milkdown handles positioning.

**`plugin-slash`** — Slash command menu triggered by typing `/`. You provide the menu UI; Milkdown handles filtering and positioning.

**`plugin-block`** — Block-level manipulation. Shows a handle on hover to drag blocks or trigger a menu. You provide the UI.

**`plugin-upload`** — File upload handling for drag-and-drop or paste. Configure via `uploadConfig` to define how files are uploaded and inserted.

### Content Plugins

**`plugin-collab`** — Y.js integration for real-time collaborative editing. Requires `yjs` and a Y.js provider (e.g., `y-websocket`).

**`plugin-prism`** — Syntax highlighting for code blocks using Refractor (Prism.js compatible).

**`plugin-math`** — LaTeX rendering via KaTeX. Supports both inline (`$...$`) and block (`$$...$$`) math.

**`plugin-emoji`** — Emoji support with twemoji rendering.

## Components

Milkdown provides pre-built components (`@milkdown/kit/component/...`) that render as ProseMirror NodeViews. These use Vue 3 internally but work with any framework:

- **`code-block`** — Syntax-highlighted code with language selector. Can integrate CodeMirror 6.
- **`image-block`** — Image display with upload, resize, caption.
- **`image-inline`** — Inline image handling.
- **`table-block`** — Interactive table editing (add/remove rows/cols, resize).
- **`link-tooltip`** — Floating tooltip for link editing.
- **`list-item-block`** — Drag-and-drop reorderable list items.

## Plugin Architecture

A Milkdown plugin is a function that receives `Ctx` and uses it to register schema, commands, input rules, keymaps, or ProseMirror plugins.

### Composable Utilities

The `@milkdown/kit/utils` package provides factory functions for creating plugins:

```typescript
import {
  $view,           // Create ProseMirror NodeView
  $command,        // Register a command
  $inputRule,      // Add an input rule
  $node,           // Define a node schema + parser/serializer
  $mark,           // Define a mark schema + parser/serializer
  $remark,         // Add a remark plugin to the pipeline
  $prose,          // Add a raw ProseMirror plugin
  $ctx,            // Create a context slice
} from '@milkdown/kit/utils';
```

### Creating a Custom Plugin

Here's a minimal example — a plugin that adds a custom ProseMirror plugin:

```typescript
import { $prose } from '@milkdown/kit/utils';
import { Plugin, PluginKey } from '@milkdown/kit/prose/state';

const myPluginKey = new PluginKey('my-plugin');

const myPlugin = $prose((ctx) => {
  return new Plugin({
    key: myPluginKey,
    props: {
      handleKeyDown(view, event) {
        if (event.key === 'F1') {
          console.log('F1 pressed in editor');
          return true; // consumed
        }
        return false;
      },
    },
  });
});

// Use it:
editor.use(myPlugin);
```

### Custom Node Example

```typescript
import { $node } from '@milkdown/kit/utils';

const customBlock = $node('customBlock', () => ({
  // ProseMirror NodeSpec
  content: 'text*',
  group: 'block',
  defining: true,
  parseDOM: [{ tag: 'div.custom-block' }],
  toDOM() { return ['div', { class: 'custom-block' }, 0]; },

  // Remark parse/serialize spec
  parseMarkdown: {
    match: (node) => node.type === 'containerDirective' && node.name === 'custom',
    runner: (state, node, type) => {
      state.openNode(type);
      state.next(node.children);
      state.closeNode();
    },
  },
  toMarkdown: {
    match: (node) => node.type.name === 'customBlock',
    runner: (state, node) => {
      state.openNode('containerDirective', undefined, { name: 'custom' });
      state.next(node.content);
      state.closeNode();
    },
  },
}));
```

### Custom Command Example

```typescript
import { $command } from '@milkdown/kit/utils';
import { createCmdKey } from '@milkdown/kit/core';

const InsertHR = createCmdKey('InsertHR');

const insertHRCommand = $command('InsertHR', (ctx) => () => {
  return (state, dispatch) => {
    if (!dispatch) return true;
    const { schema, tr } = state;
    const node = schema.nodes.hr.create();
    dispatch(tr.replaceSelectionWith(node));
    return true;
  };
});
```

## Slash Plugin Example

The slash plugin provides the trigger (`/`) and positioning. You provide the menu UI. Here's a complete React example:

```typescript
import { slashFactory } from '@milkdown/kit/plugin/slash';
import { usePluginViewFactory } from '@prosemirror-adapter/react';

// 1. Create a slash instance
const slash = slashFactory('my-slash');

// 2. Create the menu component
function SlashMenu({ ctx }) {
  const commands = [
    { label: 'Heading 1', command: () => { /* call heading command */ } },
    { label: 'Bullet List', command: () => { /* call list command */ } },
    { label: 'Code Block', command: () => { /* call code block command */ } },
  ];

  return (
    <div className="slash-menu">
      {commands.map((item) => (
        <button key={item.label} onClick={item.command}>
          {item.label}
        </button>
      ))}
    </div>
  );
}

// 3. Wire it up (see framework-integration.md for full React setup)
```

For vanilla TypeScript, you'd use `$view` or handle DOM creation directly in the plugin.

### Slash Plugin Configuration

```typescript
import { slash, slashFactory } from '@milkdown/kit/plugin/slash';

// Use the default slash plugin
editor.use(slash);

// Or create a configured instance:
const mySlash = slashFactory('my-slash');
editor.use(mySlash);
```

The slash plugin watches for `/` at the start of a new line and provides positioning data for your floating menu.

## Input Rules

Input rules transform typed patterns into ProseMirror operations. The commonmark preset already includes standard markdown input rules (`#` → heading, `>` → blockquote, etc.).

Add custom input rules:

```typescript
import { $inputRule } from '@milkdown/kit/utils';
import { InputRule } from '@milkdown/kit/prose/inputrules';

const hrInputRule = $inputRule((ctx) => {
  return new InputRule(/^---$/, (state, match, start, end) => {
    const { schema, tr } = state;
    return tr.replaceWith(start - 1, end, schema.nodes.hr.create());
  });
});

editor.use(hrInputRule);
```

## Inline Sync Plugin

Part of `preset-commonmark` (enabled by default). It handles real-time markdown syntax rendering as users type. Unlike simple input rules, it re-parses the current line through the full markdown pipeline, handling edge cases like `asd**'ef**` correctly.

Configure it:

```typescript
import { inlineSyncConfigCtx } from '@milkdown/kit/preset/commonmark';

editor.config((ctx) => {
  ctx.update(inlineSyncConfigCtx, (prev) => ({
    ...prev,
    // Control which changes trigger sync
    shouldSyncNode: ({ prevNode, newNode }) => {
      return true; // or custom logic
    },
  }));
});
```

## Node Attributes

Customize HTML attributes for built-in nodes:

```typescript
import {
  blockquoteAttr,
  inlineCodeAttr,
  headingAttr,
  paragraphAttr,
} from '@milkdown/kit/preset/commonmark';

editor.config((ctx) => {
  ctx.set(blockquoteAttr.key, () => ({
    class: 'border-l-4 border-blue-500 pl-4',
  }));

  ctx.set(inlineCodeAttr.key, () => ({
    class: 'font-mono text-sm bg-gray-100 px-1 rounded',
  }));

  ctx.set(headingAttr.key, (node) => ({
    id: node.textContent.toLowerCase().replace(/\s+/g, '-'),
    class: `heading-${node.attrs.level}`,
  }));
});
```
