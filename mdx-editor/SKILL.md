---
name: mdxeditor-react
description: "React WYSIWYG markdown/MDX editor (@mdxeditor/editor). Use when building rich-text editors with markdown output, CMS editors, or any contentEditable markdown UI in React. Covers plugins, toolbar, theming, Tailwind Typography, code blocks, images, tables, JSX components, diff/source mode, and AI autocomplete."
---

# MDXEditor — React Markdown WYSIWYG Skill

MDXEditor is an open-source React component for WYSIWYG markdown/MDX editing. Built on Lexical (Meta's editor framework) with a plugin architecture that keeps bundle size minimal — only include what you need.

- **Package**: `@mdxeditor/editor` (v3.x, ~200k weekly npm downloads)
- **Peer deps**: React 18+
- **No SSR**: Client-side only — use `dynamic()` with `{ ssr: false }` in Next.js
- **Docs**: https://mdxeditor.dev/editor/docs/getting-started

## When to Read Reference Files

This SKILL.md provides the routing and quick-start patterns. For detailed plugin APIs, toolbar components, theming, and advanced features, read the appropriate reference:

| Scenario | Reference File |
|---|---|
| Installing, framework setup (Vite/Next.js/Remix), basic usage, ref methods | `references/getting-started.md` |
| All plugins: formatting, tables, images, code blocks, links, lists, JSX, etc. | `references/plugins.md` |
| Toolbar components, custom toolbar buttons, ConditionalContents, primitives | `references/toolbar.md` |
| Theming (CSS variables, dark mode), content styling, Tailwind Typography | `references/theming-and-styling.md` |
| AI autocomplete: ghost text, inline suggestions, streaming AI content | `references/ai-autocomplete.md` |
| Extending the editor: Gurx state, custom plugins, Lexical integration | `references/extending.md` |

## Quick Start (Vite + React)

```bash
npm install @mdxeditor/editor
```

```tsx
import { MDXEditor, headingsPlugin, listsPlugin, quotePlugin,
  thematicBreakPlugin, markdownShortcutPlugin } from '@mdxeditor/editor'
import '@mdxeditor/editor/style.css'

function App() {
  return (
    <MDXEditor
      markdown="# Hello world"
      onChange={console.log}
      plugins={[
        headingsPlugin(),
        listsPlugin(),
        quotePlugin(),
        thematicBreakPlugin(),
        markdownShortcutPlugin()
      ]}
    />
  )
}
```

## Decision Tree

**What kind of editor do you need?**

1. **Simple markdown textarea replacement** → Use basic plugins only: `headingsPlugin`, `listsPlugin`, `quotePlugin`, `thematicBreakPlugin`, `markdownShortcutPlugin`. See `references/plugins.md`.

2. **Full-featured CMS editor** → Add toolbar, images, tables, code blocks, links, diff/source mode, front-matter. See `references/plugins.md` + `references/toolbar.md`.

3. **Styled editor matching your app design** → Use `contentEditableClassName="prose"` with Tailwind Typography, plus CSS variable theming for the toolbar/UI. See `references/theming-and-styling.md`.

4. **MDX with custom JSX components** → Use `jsxPlugin` with component descriptors. See `references/plugins.md` (JSX section).

5. **AI-powered autocomplete / ghost text** → Build a custom Lexical plugin via MDXEditor's extension system. See `references/ai-autocomplete.md`.

6. **Custom editor behavior / new plugins** → Understand Gurx state management and Lexical node model. See `references/extending.md`.

## Key Architecture Concepts

**Plugin system**: Every feature is a plugin. The base editor only supports bold, italic, underline, and inline code. Everything else (headings, lists, tables, images, code blocks, etc.) requires adding the corresponding plugin.

**Gurx state management**: MDXEditor uses Gurx (a graph-based reactive state system) instead of React context. State nodes are Cells (stateful) and Signals (stateless). Access them via `useCellValue()`, `useCellValues()`, and `usePublisher()` hooks re-exported from `@mdxeditor/editor`.

**Lexical under the hood**: The rich-text editing surface is Lexical. Custom editor behavior requires understanding Lexical's node model (TextNode, ElementNode, DecoratorNode) and its command/listener system.

**Markdown ↔ Lexical pipeline**: On init, markdown is parsed to MDAST (via remark/unified), then import visitors convert MDAST nodes to Lexical nodes. On export, export visitors do the reverse. Custom plugins can register their own visitors via `addImportVisitor$` and `addExportVisitor$`.

## Common Prop Reference

| Prop | Type | Description |
|---|---|---|
| `markdown` | `string` | Initial markdown value (like `defaultValue` — use `setMarkdown` ref method for updates) |
| `onChange` | `(md: string) => void` | Fires continuously as user types |
| `plugins` | `RealmPlugin[]` | Array of plugins to enable |
| `readOnly` | `boolean` | Disables editing |
| `contentEditableClassName` | `string` | CSS class for the editable content area (use `"prose"` for Tailwind Typography) |
| `className` | `string` | CSS class for the root editor element |
| `ref` | `Ref<MDXEditorMethods>` | Access `getMarkdown()`, `setMarkdown()`, `insertMarkdown()`, `focus()` |
| `onError` | `(payload) => void` | Error handler for markdown processing errors |
| `toMarkdownOptions` | `object` | Options passed to `mdast-util-to-markdown` (bullet style, list indent, etc.) |

**Full-featured example** with all plugins and toolbar: See `references/getting-started.md` → "Full-Featured Example" section.
