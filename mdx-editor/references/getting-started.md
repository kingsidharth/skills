# Getting Started Reference

## Installation

```bash
npm install @mdxeditor/editor
```

Import the CSS file — required for the editor to render correctly:

```tsx
import '@mdxeditor/editor/style.css'
```

## Framework Setup

### Vite (Works out of the box)

```tsx
// App.tsx
import { MDXEditor, headingsPlugin } from '@mdxeditor/editor'
import '@mdxeditor/editor/style.css'

function App() {
  return <MDXEditor markdown="# Hello world" plugins={[headingsPlugin()]} />
}
export default App
```

### Next.js (App Router) — Client-Only Required

MDXEditor does not support SSR. Use `dynamic()` with `{ ssr: false }`. **Critical**: Ensure plugins are also initialized client-side only — some plugins cause hydration errors if imported during SSR.

**Step 1**: Create the initialized editor component:

```tsx
'use client'
// InitializedMDXEditor.tsx
import type { ForwardedRef } from 'react'
import {
  headingsPlugin, listsPlugin, quotePlugin,
  thematicBreakPlugin, markdownShortcutPlugin,
  MDXEditor, type MDXEditorMethods, type MDXEditorProps
} from '@mdxeditor/editor'
import '@mdxeditor/editor/style.css'

export default function InitializedMDXEditor({
  editorRef, ...props
}: { editorRef: ForwardedRef<MDXEditorMethods> | null } & MDXEditorProps) {
  return (
    <MDXEditor
      plugins={[
        headingsPlugin(), listsPlugin(), quotePlugin(),
        thematicBreakPlugin(), markdownShortcutPlugin()
      ]}
      {...props}
      ref={editorRef}
    />
  )
}
```

**Step 2**: Create the dynamic wrapper:

```tsx
'use client'
// ForwardRefEditor.tsx
import dynamic from 'next/dynamic'
import { forwardRef } from 'react'
import { type MDXEditorMethods, type MDXEditorProps } from '@mdxeditor/editor'

const Editor = dynamic(() => import('./InitializedMDXEditor'), { ssr: false })

export const ForwardRefEditor = forwardRef<MDXEditorMethods, MDXEditorProps>(
  (props, ref) => <Editor {...props} editorRef={ref} />
)
ForwardRefEditor.displayName = 'ForwardRefEditor'
```

### Next.js (Pages Router)

Requires transpilation config in `next.config.js`:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  transpilePackages: ['@mdxeditor/editor'],
  reactStrictMode: true,
  webpack: (config) => {
    config.experiments = { ...config.experiments, topLevelAwait: true }
    return config
  }
}
module.exports = nextConfig
```

### Remix

Use the `ClientOnly` utility from `remix-utils`:

```tsx
import { ClientOnly } from 'remix-utils/client-only'

export default function EditorRoute() {
  return (
    <ClientOnly fallback={<textarea />}>
      {() => <MyMDXEditor />}
    </ClientOnly>
  )
}
```

## Basic Usage Patterns

### Controlled-ish Value (like textarea defaultValue)

The `markdown` prop is the **initial** value only. To update dynamically, use the ref:

```tsx
const ref = useRef<MDXEditorMethods>(null)

// Set new content programmatically
ref.current?.setMarkdown('# New content')

// Get current content
const md = ref.current?.getMarkdown()

// Insert at cursor position
ref.current?.insertMarkdown('**inserted text**')

// Focus the editor
ref.current?.focus()
```

### Listening for Changes

```tsx
<MDXEditor
  markdown={initialMarkdown}
  onChange={(newMarkdown) => {
    // Fires continuously as user types
    setContent(newMarkdown)
  }}
/>
```

### Configuring Markdown Output

Control how the markdown is serialized via `toMarkdownOptions` (passed to `mdast-util-to-markdown`):

```tsx
<MDXEditor
  markdown={value}
  toMarkdownOptions={{
    bullet: '-',          // or '*' or '+'
    listItemIndent: 'one', // or 'tab' or 'mixed'
    emphasis: '_',         // or '*'
    rule: '-',
    incrementListMarker: false,
  }}
/>
```

### Read-Only Mode

```tsx
<MDXEditor markdown={markdown} readOnly={true} plugins={[...]} />
```

### Error Handling

The `diffSourcePlugin` acts as an escape hatch — if markdown processing fails, it suggests the user switch to source mode to fix issues:

```tsx
<MDXEditor
  markdown={markdown}
  onError={(error) => {
    console.error('MDXEditor error:', error)
  }}
  plugins={[
    diffSourcePlugin({ viewMode: 'rich-text' }),
    // ... other plugins
  ]}
/>
```

## Full-Featured Example

A kitchen-sink editor with all common plugins and a complete toolbar:

```tsx
import {
  MDXEditor, headingsPlugin, listsPlugin, quotePlugin,
  thematicBreakPlugin, markdownShortcutPlugin, linkPlugin,
  linkDialogPlugin, imagePlugin, tablePlugin, frontmatterPlugin,
  codeBlockPlugin, codeMirrorPlugin, diffSourcePlugin,
  toolbarPlugin, BoldItalicUnderlineToggles, UndoRedo,
  BlockTypeSelect, CreateLink, InsertImage, InsertTable,
  InsertThematicBreak, ListsToggle, CodeToggle, InsertCodeBlock,
  InsertFrontmatter, DiffSourceToggleWrapper, Separator
} from '@mdxeditor/editor'
import '@mdxeditor/editor/style.css'

function FullEditor({ markdown, onChange }) {
  return (
    <MDXEditor
      markdown={markdown}
      onChange={onChange}
      contentEditableClassName="prose prose-sm max-w-none"
      plugins={[
        headingsPlugin(),
        listsPlugin(),
        quotePlugin(),
        thematicBreakPlugin(),
        markdownShortcutPlugin(),
        linkPlugin(),
        linkDialogPlugin(),
        imagePlugin({ imageUploadHandler: async (file) => {
          // Upload file and return URL
          return URL.createObjectURL(file)
        }}),
        tablePlugin(),
        frontmatterPlugin(),
        codeBlockPlugin({ defaultCodeBlockLanguage: 'js' }),
        codeMirrorPlugin({
          codeBlockLanguages: {
            js: 'JavaScript', ts: 'TypeScript', tsx: 'TypeScript (React)',
            css: 'CSS', html: 'HTML', json: 'JSON', bash: 'Bash',
            python: 'Python', '': 'Plain text'
          }
        }),
        diffSourcePlugin({ viewMode: 'rich-text' }),
        toolbarPlugin({
          toolbarContents: () => (
            <DiffSourceToggleWrapper>
              <UndoRedo />
              <Separator />
              <BoldItalicUnderlineToggles />
              <CodeToggle />
              <Separator />
              <BlockTypeSelect />
              <Separator />
              <CreateLink />
              <InsertImage />
              <InsertTable />
              <InsertThematicBreak />
              <Separator />
              <ListsToggle />
              <Separator />
              <InsertCodeBlock />
              <InsertFrontmatter />
            </DiffSourceToggleWrapper>
          )
        })
      ]}
    />
  )
}
```
