# Core API Reference

Detailed reference for Milkdown's core systems: the Editor class, Context (Ctx) dependency injection, actions, macros, and the listener plugin.

## Editor Class

The `Editor` class from `@milkdown/core` (or `@milkdown/kit/core`) is the central orchestrator. It manages plugin lifecycle through a timer-based dependency system.

### Builder Pattern

```typescript
import { Editor, rootCtx, defaultValueCtx } from '@milkdown/kit/core';
import { commonmark } from '@milkdown/kit/preset/commonmark';

const editor = Editor.make()
  .config((ctx) => {
    ctx.set(rootCtx, document.getElementById('editor'));
    ctx.set(defaultValueCtx, '# Hello');
  })
  .config(anotherConfigFn)   // Multiple .config() calls are fine
  .use(commonmark)
  .use(anotherPlugin);

await editor.create();       // Async — initializes everything
```

You can chain `.config()` and `.use()` in any order. Plugins are initialized in dependency order (managed by timers), not registration order.

### Lifecycle Methods

| Method | Returns | Description |
|---|---|---|
| `Editor.make()` | `Editor` | Static factory — creates a new builder |
| `.config(fn)` | `Editor` | Register a config callback `(ctx: Ctx) => void` |
| `.use(plugin)` | `Editor` | Register a plugin or plugin array |
| `.remove(plugin)` | `Editor` | Unregister a plugin |
| `.create()` | `Promise<Editor>` | Initialize the editor (can be called after destroy) |
| `.destroy()` | `Promise<void>` | Tear down ProseMirror view and cleanup |
| `.action(fn)` | Return type of `fn` | Execute `(ctx: Ctx) => T` against the live editor |

### Editor Status

```typescript
import { Editor, EditorStatus } from '@milkdown/kit/core';

const editor = Editor.make().use(commonmark);

// Status: EditorStatus.Idle
await editor.create();
// Status: EditorStatus.Created
await editor.destroy();
// Status: EditorStatus.Destroyed

// Monitor changes:
editor.onStatusChange((status: EditorStatus) => {
  console.log('Editor is now:', status);
});
```

Statuses: `Idle`, `OnCreate`, `Created`, `OnDestroyed`, `Destroyed`.

## Context System (Ctx)

Ctx is the dependency injection container shared across all plugins. It has two subsystems:

### Slices (Key-Value Store)

Typed containers for shared state. Created with a unique key and default value.

```typescript
// Built-in slices from @milkdown/core:
import {
  rootCtx,               // HTMLElement | string — editor mount point
  defaultValueCtx,       // string | { type: 'html'|'json', ... } — initial content
  editorViewCtx,         // ProseMirror EditorView (read after create)
  editorViewOptionsCtx,  // ProseMirror EditorViewConfig
  serializerCtx,         // Doc → Markdown function
  parserCtx,             // Markdown → Doc function
  schemaCtx,             // ProseMirror Schema
  commandsCtx,           // Command manager
} from '@milkdown/kit/core';
```

**Reading**: `const view = ctx.get(editorViewCtx);`

**Writing**: `ctx.set(rootCtx, '#editor');`

**Updating** (merge with previous value): 
```typescript
ctx.update(editorViewOptionsCtx, (prev) => ({
  ...prev,
  editable: () => !readOnly,
  attributes: { class: 'my-editor', spellcheck: 'false' },
}));
```

### Timers (Execution Ordering)

Timers control when plugins can proceed. A plugin can `await ctx.wait(SomeTimer)` to block until another plugin signals completion via `ctx.done(SomeTimer)`. This is how Milkdown ensures schema is ready before the parser runs, etc.

You rarely need to create custom timers unless writing advanced plugins.

## Actions

Actions let you interact with a created editor programmatically. They receive the `Ctx` and can return values.

```typescript
// Read-only action — get markdown
const markdown = editor.action((ctx) => {
  const view = ctx.get(editorViewCtx);
  const serializer = ctx.get(serializerCtx);
  return serializer(view.state.doc);
});
```

### Built-in Macros (from @milkdown/kit/utils)

Macros are pre-built action factories:

```typescript
import { insert, replaceAll, callCommand, getHTML } from '@milkdown/kit/utils';

// Insert markdown at cursor position
editor.action(insert('**bold** and *italic*'));

// Replace all editor content
editor.action(replaceAll('# Fresh Start'));

// Call a registered command
import { toggleBoldCommand } from '@milkdown/kit/preset/commonmark';
editor.action(callCommand(toggleBoldCommand.key));

// Get HTML representation
const html = editor.action(getHTML());
```

### Custom Actions

```typescript
// Focus the editor
editor.action((ctx) => {
  const view = ctx.get(editorViewCtx);
  view.focus();
});

// Get document JSON
editor.action((ctx) => {
  const view = ctx.get(editorViewCtx);
  return view.state.doc.toJSON();
});

// Replace content with parsed markdown
editor.action((ctx) => {
  const view = ctx.get(editorViewCtx);
  const parser = ctx.get(parserCtx);
  const doc = parser('# New content\n\nParagraph here.');
  const tr = view.state.tr.replaceWith(
    0,
    view.state.doc.content.size,
    doc.content
  );
  view.dispatch(tr);
});
```

## Listener Plugin

The listener plugin provides reactive hooks for editor changes. It must be installed and used as a plugin.

```typescript
import { listener, listenerCtx } from '@milkdown/kit/plugin/listener';

Editor.make()
  .config((ctx) => {
    const l = ctx.get(listenerCtx);

    // Fires when markdown changes
    l.markdownUpdated((ctx, markdown, prevMarkdown) => {
      if (markdown !== prevMarkdown) {
        debouncedSave(markdown);
      }
    });

    // Fires when ProseMirror doc changes
    l.updated((ctx, doc, prevDoc) => {
      console.log('Doc JSON:', doc.toJSON());
    });

    // Fires when editor is mounted
    l.mounted((ctx) => {
      console.log('Editor mounted');
    });

    // Fires on every ProseMirror transaction
    l.beforeMount((ctx) => {
      console.log('About to mount');
    });
  })
  .use(listener)
  .use(commonmark);
```

The listener must be registered via `.use(listener)` — just configuring `listenerCtx` without using the plugin won't work.

## Default Value Types

```typescript
import { defaultValueCtx } from '@milkdown/kit/core';

// 1. Markdown string (most common)
ctx.set(defaultValueCtx, '# Title\n\nContent');

// 2. HTML DOM node
ctx.set(defaultValueCtx, {
  type: 'html',
  dom: document.querySelector('#source-html'),
});

// 3. ProseMirror JSON (from previous doc.toJSON())
ctx.set(defaultValueCtx, {
  type: 'json',
  value: { type: 'doc', content: [{ type: 'paragraph', content: [...] }] },
});
```

When restoring content from a database, JSON format avoids reparsing overhead. Get the JSON via the listener's `.updated()` callback or an action.

## Readonly Mode

```typescript
import { editorViewOptionsCtx } from '@milkdown/kit/core';

let isReadonly = false;

editor.config((ctx) => {
  ctx.update(editorViewOptionsCtx, (prev) => ({
    ...prev,
    editable: () => !isReadonly,
  }));
});

// Toggle later — ProseMirror checks editable() on each transaction
isReadonly = true;
```

## EditorView Attributes

Add CSS classes, data attributes, or spellcheck settings:

```typescript
ctx.update(editorViewOptionsCtx, (prev) => ({
  ...prev,
  attributes: {
    class: 'my-milkdown-editor prose',
    spellcheck: 'false',
    'data-testid': 'markdown-editor',
  },
}));
```

## Remark Plugin Configuration

Configure the underlying remark stringify behavior:

```typescript
import { remarkStringifyOptionsCtx } from '@milkdown/kit/core';

editor.config((ctx) => {
  ctx.set(remarkStringifyOptionsCtx, {
    bullet: '-',           // Use - for unordered lists (default: *)
    emphasis: '_',         // Use _ for emphasis (default: *)
    listItemIndent: 'one', // Indent list items by one space
  });
});
```

## rootDOMCtx

Access the root DOM element of the current editor:

```typescript
import { rootDOMCtx } from '@milkdown/kit/core';

editor.action((ctx) => {
  const rootDOM = ctx.get(rootDOMCtx);
  rootDOM.scrollTop = 0;
});
```
