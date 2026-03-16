# Plugins Reference

Every MDXEditor feature beyond basic bold/italic/underline/inline-code requires a plugin. Plugins are passed as an array to the `plugins` prop.

## Plugin Overview Table

| Plugin | Import | Purpose | Toolbar Component |
|---|---|---|---|
| `headingsPlugin()` | `headingsPlugin` | H1–H6 headings | `BlockTypeSelect` |
| `listsPlugin()` | `listsPlugin` | Ordered, unordered, check lists (nested) | `ListsToggle` |
| `quotePlugin()` | `quotePlugin` | Block quotes | `BlockTypeSelect` |
| `thematicBreakPlugin()` | `thematicBreakPlugin` | Horizontal rules (`---`) | `InsertThematicBreak` |
| `markdownShortcutPlugin()` | `markdownShortcutPlugin` | Keyboard shortcuts (Notion-style) | — |
| `linkPlugin()` | `linkPlugin` | Markdown links | — |
| `linkDialogPlugin()` | `linkDialogPlugin` | Floating link edit popover + Ctrl/Cmd+K | `CreateLink` |
| `imagePlugin()` | `imagePlugin` | Images (URL, paste, drop, resize) | `InsertImage` |
| `tablePlugin()` | `tablePlugin` | GFM markdown tables | `InsertTable` |
| `codeBlockPlugin()` | `codeBlockPlugin` | Fenced code blocks (base — no editor UI) | `InsertCodeBlock` |
| `codeMirrorPlugin()` | `codeMirrorPlugin` | CodeMirror editor for code blocks | `ChangeCodeMirrorLanguage` |
| `sandpackPlugin()` | `sandpackPlugin` | Sandpack live code blocks | `InsertSandpack`, `ShowSandpackInfo` |
| `frontmatterPlugin()` | `frontmatterPlugin` | YAML front-matter key-value editor | `InsertFrontmatter` |
| `diffSourcePlugin()` | `diffSourcePlugin` | Rich-text / source / diff toggle | `DiffSourceToggleWrapper` |
| `toolbarPlugin()` | `toolbarPlugin` | Toolbar container | — |
| `directivesPlugin()` | `directivesPlugin` | Generic directive support | — |
| `jsxPlugin()` | `jsxPlugin` | JSX/MDX component editing | Custom buttons |
| `searchPlugin()` | `searchPlugin` | Find and replace in rich-text mode | Custom UI via `useEditorSearch` hook |

**Important**: `markdownShortcutPlugin()` must come **after** the plugins whose blocks it triggers (headings, lists, links, quotes).

## Basic Formatting Plugins

### Headings

```tsx
import { headingsPlugin } from '@mdxeditor/editor'
// Enables H1–H6. Pair with BlockTypeSelect in toolbar.
plugins={[headingsPlugin()]}
```

### Lists

```tsx
import { listsPlugin } from '@mdxeditor/editor'
// Supports ordered, unordered, and checkbox lists with nesting.
plugins={[listsPlugin()]}
```

### Quotes

```tsx
import { quotePlugin } from '@mdxeditor/editor'
// Enables > blockquote syntax.
plugins={[quotePlugin()]}
```

### Thematic Break

```tsx
import { thematicBreakPlugin } from '@mdxeditor/editor'
// Enables --- horizontal rules.
plugins={[thematicBreakPlugin()]}
```

### Markdown Shortcuts

```tsx
import { markdownShortcutPlugin } from '@mdxeditor/editor'
// Notion-style shortcuts:
// # → heading, * or - → list, > → quote
// Ctrl+B bold, Ctrl+I italic, Ctrl+U underline
// Cmd+K → link dialog
// ` → inline code
// ```lang + space → code block
plugins={[markdownShortcutPlugin()]}
```

## Links

```tsx
import { linkPlugin, linkDialogPlugin } from '@mdxeditor/editor'

plugins={[
  linkPlugin(),
  linkDialogPlugin({
    // Optional: pre-configured URL suggestions for the link dialog
    linkAutocompleteSuggestions: [
      'https://example.com',
      'https://docs.example.com'
    ]
  })
]}
```

The link dialog floats as a popover when cursor is inside a link. Supports Ctrl+K / Cmd+K shortcut to open.

## Images

```tsx
import { imagePlugin } from '@mdxeditor/editor'

plugins={[
  imagePlugin({
    // Handler for pasted/dropped images — upload and return URL
    imageUploadHandler: async (file: File) => {
      const formData = new FormData()
      formData.append('image', file)
      const response = await fetch('/uploads/new', {
        method: 'POST',
        body: formData
      })
      const json = await response.json()
      return json.url
    },
    // Optional: URL autocomplete suggestions
    imageAutocompleteSuggestions: [
      'https://picsum.photos/200/300'
    ]
  })
]}
```

**Image resizing**: If a user resizes an image, it serializes as an HTML `<img>` tag with `width`/`height` attributes instead of markdown `![]()` syntax.

## Tables

```tsx
import { tablePlugin } from '@mdxeditor/editor'
// Supports GFM markdown tables with inline editing.
// Users can add/remove rows and columns, change column alignment.
// Each cell supports markdown formatting (bold, links, images, etc.).
// Note: HTML tables are NOT supported.
plugins={[tablePlugin()]}
```

## Code Blocks

Three-layer system: `codeBlockPlugin` (base) + `codeMirrorPlugin` (syntax highlighting) + `sandpackPlugin` (live execution).

### Basic Code Blocks with CodeMirror

```tsx
import { codeBlockPlugin, codeMirrorPlugin } from '@mdxeditor/editor'

plugins={[
  codeBlockPlugin({ defaultCodeBlockLanguage: 'js' }),
  codeMirrorPlugin({
    codeBlockLanguages: {
      js: 'JavaScript',
      ts: 'TypeScript',
      tsx: 'TypeScript (React)',
      jsx: 'JavaScript (React)',
      css: 'CSS',
      html: 'HTML',
      json: 'JSON',
      bash: 'Bash',
      python: 'Python',
      sql: 'SQL',
      '': 'Plain text'
    }
  })
]}
```

### Sandpack Live Code Blocks

```tsx
import { sandpackPlugin, SandpackConfig } from '@mdxeditor/editor'

const sandpackConfig: SandpackConfig = {
  defaultPreset: 'react',
  presets: [
    {
      label: 'React',
      name: 'react',
      meta: 'live react',
      sandpackTemplate: 'react',
      sandpackTheme: 'light',
      snippetFileName: '/App.js',
      snippetLanguage: 'jsx',
      initialSnippetContent: `export default function App() {
  return <h1>Hello CodeSandbox</h1>
}`
    }
  ]
}

plugins={[
  codeBlockPlugin({ defaultCodeBlockLanguage: 'js' }),
  sandpackPlugin({ sandpackConfig }),
  codeMirrorPlugin({ codeBlockLanguages: { js: 'JavaScript', css: 'CSS' } })
]}
```

### Custom Code Block Editor

```tsx
import { codeBlockPlugin, CodeBlockEditorDescriptor, useCodeBlockEditorContext } from '@mdxeditor/editor'

const PlainTextEditor: CodeBlockEditorDescriptor = {
  match: (language, meta) => true,  // match all
  priority: 0,                       // lowest priority = fallback
  Editor: (props) => {
    const cb = useCodeBlockEditorContext()
    return (
      <div onKeyDown={(e) => e.nativeEvent.stopImmediatePropagation()}>
        <textarea
          rows={3} cols={20}
          defaultValue={props.code}
          onChange={(e) => cb.setCode(e.target.value)}
        />
      </div>
    )
  }
}

plugins={[
  codeBlockPlugin({
    codeBlockEditorDescriptors: [PlainTextEditor]
  })
]}
```

## Front-matter

```tsx
import { frontmatterPlugin } from '@mdxeditor/editor'
// Renders a key-value form editor for YAML front-matter.
// Users can add/remove rows.
plugins={[frontmatterPlugin()]}
```

Input markdown:
```markdown
---
slug: hello-world
title: My Post
---

Content here
```

## Diff / Source Mode

```tsx
import { diffSourcePlugin } from '@mdxeditor/editor'

plugins={[
  diffSourcePlugin({
    diffMarkdown: 'An older version of the markdown',  // for diff view
    viewMode: 'rich-text',   // 'rich-text' | 'source' | 'diff'
    readOnlyDiff: false,     // make only diff view read-only
  })
]}
```

Use `DiffSourceToggleWrapper` in the toolbar to let users switch between views. Wrap the rich-text toolbar contents as children:

```tsx
toolbarPlugin({
  toolbarContents: () => (
    <DiffSourceToggleWrapper>
      <UndoRedo />
      <BoldItalicUnderlineToggles />
    </DiffSourceToggleWrapper>
  )
})
```

The diff-source plugin also serves as an error recovery escape hatch — if markdown processing fails, it suggests switching to source mode.

## JSX / MDX Components

```tsx
import { jsxPlugin, GenericJsxEditor, JsxComponentDescriptor } from '@mdxeditor/editor'

const jsxComponentDescriptors: JsxComponentDescriptor[] = [
  {
    name: 'MyComponent',
    kind: 'text',    // 'text' = inline, 'flow' = block
    source: './external',  // import path (for the import statement in output)
    props: [
      { name: 'title', type: 'string' },      // renders as "title"
      { name: 'onClick', type: 'expression' }  // renders as {onClick}
    ],
    hasChildren: true,
    Editor: GenericJsxEditor  // built-in property editor
  },
  {
    name: 'BlockWidget',
    kind: 'flow',
    source: './widgets',
    props: [],
    hasChildren: true,
    Editor: GenericJsxEditor
  }
]

plugins={[
  jsxPlugin({ jsxComponentDescriptors })
]}
```

**Property types**: `'string'` produces `prop="value"`, `'expression'` produces `prop={value}`.

**Custom JSX editors**: Use `NestedLexicalEditor` to allow markdown editing inside JSX component children.

## Search and Replace

```tsx
import { searchPlugin } from '@mdxeditor/editor'

plugins={[searchPlugin()]}
```

Build custom search UI with the `useEditorSearch` hook:

```tsx
import { useEditorSearch } from '@mdxeditor/editor'

function SearchBar() {
  const { search, setSearch, next, prev, total, cursor, replace, replaceAll } = useEditorSearch()

  return (
    <div>
      <input
        value={search ?? ''}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search..."
      />
      <span>{total > 0 ? `${cursor} / ${total}` : '0 / 0'}</span>
      <button onClick={prev} disabled={total === 0}>Prev</button>
      <button onClick={next} disabled={total === 0}>Next</button>
    </div>
  )
}
```

**Required CSS** for search highlights:

```css
::highlight(MdxSearch) {
  background: yellow;
}
::highlight(MdxFocusSearch) {
  background: fuchsia;
}
```

**Notes**:
- Uses `CSS.highlights` API (widely supported since July 2024)
- Supports regex and wildcard searches across newlines
- Does NOT search inside embedded CodeMirror editors
- Searches across formatting (bold, italic, etc.)
