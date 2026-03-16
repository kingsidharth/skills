# Framework Integration

Milkdown provides official integration packages for React, Vue, and Svelte. Each adapter maps Milkdown's lifecycle to the framework's component model. You can use either the granular Kit API or the batteries-included Crepe editor with any framework.

## React Integration

Install: `npm i @milkdown/react`

### Kit + React (Full Control)

```tsx
import { Editor, rootCtx, defaultValueCtx } from '@milkdown/kit/core';
import { commonmark } from '@milkdown/kit/preset/commonmark';
import { history } from '@milkdown/kit/plugin/history';
import { listener, listenerCtx } from '@milkdown/kit/plugin/listener';
import { Milkdown, MilkdownProvider, useEditor, useInstance } from '@milkdown/react';

function EditorComponent({ defaultValue, onChange }: {
  defaultValue: string;
  onChange: (md: string) => void;
}) {
  useEditor((root) => {
    return Editor.make()
      .config((ctx) => {
        ctx.set(rootCtx, root);
        ctx.set(defaultValueCtx, defaultValue);
        ctx.get(listenerCtx).markdownUpdated((ctx, markdown, prev) => {
          if (markdown !== prev) onChange(markdown);
        });
      })
      .use(commonmark)
      .use(history)
      .use(listener);
  }, []);   // deps array — empty = create once

  return <Milkdown />;
}

// Wrap with MilkdownProvider at the top level
function App() {
  const [content, setContent] = useState('# Hello');

  return (
    <MilkdownProvider>
      <EditorComponent defaultValue={content} onChange={setContent} />
    </MilkdownProvider>
  );
}
```

**Key React patterns:**

- `MilkdownProvider` must wrap any component that uses `useEditor` or `useInstance`. It provides editor context.
- `useEditor(factory, deps)` creates the editor. The factory receives `root` (DOM element). Return an `Editor` instance. The `deps` array works like `useEffect` deps — changing it recreates the editor.
- `<Milkdown />` renders the actual editor DOM. Place it where you want the editor to appear.
- `useInstance()` returns `[loading, getInstance]` for imperative access after creation.

### Crepe + React

```tsx
import { Crepe } from '@milkdown/crepe';
import { Milkdown, MilkdownProvider, useEditor } from '@milkdown/react';
import '@milkdown/crepe/theme/common/style.css';
import '@milkdown/crepe/theme/frame.css';

function CrepeEditor({ defaultValue, onChange }: {
  defaultValue: string;
  onChange: (md: string) => void;
}) {
  useEditor((root) => {
    const crepe = new Crepe({
      root,
      defaultValue,
      features: {
        [Crepe.Feature.CodeMirror]: true,
        [Crepe.Feature.BlockEdit]: true,
      },
      featureConfigs: {
        [Crepe.Feature.Placeholder]: { text: 'Start writing...' },
      },
    });

    // Listen for changes via Crepe's .on() API
    crepe.on((listener) => {
      listener.markdownUpdated((ctx, markdown, prev) => {
        if (markdown !== prev) onChange(markdown);
      });
    });

    return crepe;  // useEditor accepts both Editor and Crepe instances
  }, []);

  return <Milkdown />;
}

function App() {
  const [md, setMd] = useState('# Hello Crepe');
  return (
    <MilkdownProvider>
      <CrepeEditor defaultValue={md} onChange={setMd} />
    </MilkdownProvider>
  );
}
```

### Crepe + React (Vanilla useEffect Pattern)

Alternative approach without `@milkdown/react` — useful when you want full control:

```tsx
import { Crepe } from '@milkdown/crepe';
import '@milkdown/crepe/theme/common/style.css';
import '@milkdown/crepe/theme/frame.css';
import { useRef, useLayoutEffect } from 'react';

function CrepeEditor({ value }: { value: string }) {
  const divRef = useRef<HTMLDivElement>(null);

  useLayoutEffect(() => {
    if (!divRef.current) return;

    const crepe = new Crepe({
      root: divRef.current,
      defaultValue: value,
    });
    crepe.create();

    return () => { crepe.destroy(); };
  }, [value]);

  return <div ref={divRef} />;
}
```

### useInstance — Imperative Access

```tsx
import { useInstance } from '@milkdown/react';
import { insert } from '@milkdown/kit/utils';

function Toolbar() {
  const [loading, getInstance] = useInstance();

  const insertImage = () => {
    if (loading) return;
    getInstance().action(insert('![alt](url)'));
  };

  return <button onClick={insertImage} disabled={loading}>Insert Image</button>;
}
```

`useInstance` must be inside a `MilkdownProvider` and used alongside an `<EditorComponent>` that calls `useEditor`.

### React Strict Mode and Multiple Instances

React 18's Strict Mode calls effects twice in development. This can create duplicate editor instances. Solutions:

1. Use `useEditor` from `@milkdown/react` — it handles cleanup automatically.
2. If using raw `useEffect`, always return a cleanup function that calls `editor.destroy()`.
3. Use a ref to track the instance and guard against double creation.

```tsx
// Pattern to avoid duplicate editors
const createdRef = useRef(false);

useEffect(() => {
  if (createdRef.current) return;
  createdRef.current = true;

  const editor = await Editor.make().use(commonmark).create();

  return () => {
    editor.destroy();
    createdRef.current = false;
  };
}, []);
```

### Building a Toolbar in React

Milkdown is headless — you build your own toolbar. Use commands and `callCommand`:

```tsx
import { callCommand } from '@milkdown/kit/utils';
import {
  toggleBoldCommand,
  toggleItalicCommand,
  wrapInHeadingCommand,
  wrapInBulletListCommand,
} from '@milkdown/kit/preset/commonmark';
import { useInstance } from '@milkdown/react';

function Toolbar() {
  const [loading, get] = useInstance();

  const call = (command, payload?) => {
    if (loading) return;
    get().action(callCommand(command, payload));
  };

  return (
    <div className="toolbar">
      <button onClick={() => call(toggleBoldCommand.key)}>B</button>
      <button onClick={() => call(toggleItalicCommand.key)}>I</button>
      <button onClick={() => call(wrapInHeadingCommand.key, 1)}>H1</button>
      <button onClick={() => call(wrapInHeadingCommand.key, 2)}>H2</button>
      <button onClick={() => call(wrapInBulletListCommand.key)}>List</button>
    </div>
  );
}
```

## Vue Integration

Install: `npm i @milkdown/vue` (for Vue 3)

```vue
<script setup>
import { Editor, rootCtx, defaultValueCtx } from '@milkdown/kit/core';
import { commonmark } from '@milkdown/kit/preset/commonmark';
import { VueEditor, useEditor } from '@milkdown/vue';
import { ref } from 'vue';

const editorRef = ref(null);

const editor = useEditor((root) =>
  Editor.make()
    .config((ctx) => {
      ctx.set(rootCtx, root);
      ctx.set(defaultValueCtx, '# Hello Vue');
    })
    .use(commonmark)
);
</script>

<template>
  <VueEditor :editor="editor" :editorRef="editorRef" />
</template>
```

The Vue integration wraps the editor in a `Ref` for reactivity. `onUnmounted` handles cleanup automatically.

## Svelte Integration

No official `@milkdown/svelte` package — use vanilla initialization in `onMount`:

```svelte
<script>
  import { onMount, onDestroy } from 'svelte';
  import { Editor, rootCtx, defaultValueCtx } from '@milkdown/core';
  import { commonmark } from '@milkdown/preset-commonmark';

  let editorElement;
  let editor;

  onMount(async () => {
    editor = await Editor.make()
      .config((ctx) => {
        ctx.set(rootCtx, editorElement);
        ctx.set(defaultValueCtx, '# Hello Svelte');
      })
      .use(commonmark)
      .create();
  });

  onDestroy(() => {
    editor?.destroy();
  });
</script>

<div bind:this={editorElement}></div>
```

## Vanilla TypeScript / JavaScript

No framework wrapper needed:

```typescript
import { Editor, rootCtx, defaultValueCtx } from '@milkdown/core';
import { commonmark } from '@milkdown/preset-commonmark';

const editor = await Editor.make()
  .config((ctx) => {
    ctx.set(rootCtx, '#editor');   // CSS selector or DOM element
    ctx.set(defaultValueCtx, '# Hello Milkdown');
  })
  .use(commonmark)
  .create();
```

The editor mounts to `document.body` by default if no `rootCtx` is set.

## Next.js Integration

Milkdown requires browser APIs. In Next.js, dynamically import your editor component:

```tsx
// components/Editor.tsx — client-only
'use client';

import { MilkdownProvider, useEditor, Milkdown } from '@milkdown/react';
// ... editor setup ...

export default function EditorWrapper() {
  return (
    <MilkdownProvider>
      <EditorInner />
    </MilkdownProvider>
  );
}
```

```tsx
// page.tsx
import dynamic from 'next/dynamic';

const Editor = dynamic(() => import('@/components/Editor'), { ssr: false });

export default function Page() {
  return <Editor />;
}
```

Or use the `'use client'` directive (App Router) to ensure the component only runs on the client.

## Collaborative Editing (Y.js)

```typescript
import { collab, collabServiceCtx } from '@milkdown/plugin-collab';
import * as Y from 'yjs';
import { WebsocketProvider } from 'y-websocket';

const doc = new Y.Doc();
const wsProvider = new WebsocketProvider('ws://localhost:1234', 'my-room', doc);

editor
  .config((ctx) => {
    ctx.set(collabServiceCtx, {
      doc,
      awareness: wsProvider.awareness,
    });
  })
  .use(collab);
```

The collab plugin replaces the standard editor state with Y.js-backed state, enabling real-time multi-user editing.

## Image Upload Pattern

```typescript
import { Crepe } from '@milkdown/crepe';

const crepe = new Crepe({
  root: '#editor',
  featureConfigs: {
    [Crepe.Feature.ImageBlock]: {
      onUpload: async (file: File) => {
        const formData = new FormData();
        formData.append('file', file);
        const res = await fetch('/api/upload', { method: 'POST', body: formData });
        const data = await res.json();
        return data.url;  // Return the URL to embed in the document
      },
    },
  },
});
```
