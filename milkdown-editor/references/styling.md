# Styling & Theming

Milkdown v7 is **headless by default** — it renders semantic HTML with no built-in styles. You control all visual presentation through CSS. The Crepe editor is the exception, shipping pre-built themes.

## Headless Approach (Kit / Core)

When using `@milkdown/kit` or `@milkdown/core` directly, the editor renders into a `.milkdown` container with standard ProseMirror markup. You need to provide all styling yourself.

### Basic Structure

```html
<!-- What Milkdown renders -->
<div data-milkdown-root="true">
  <div class="milkdown">
    <div class="editor" contenteditable="true">
      <h1>Heading</h1>
      <p>Paragraph text with <strong>bold</strong> and <em>italic</em>.</p>
      <ul>
        <li>List item</li>
      </ul>
      <pre><code class="language-javascript">const x = 1;</code></pre>
    </div>
  </div>
</div>
```

### Minimal CSS Starting Point

```css
/* ProseMirror base styles (important) */
.ProseMirror {
  outline: none;
  min-height: 200px;
  padding: 1rem;
}

.ProseMirror:focus {
  outline: none;
}

/* Basic typography */
.milkdown h1 { font-size: 2em; font-weight: bold; margin: 0.67em 0; }
.milkdown h2 { font-size: 1.5em; font-weight: bold; margin: 0.83em 0; }
.milkdown h3 { font-size: 1.25em; font-weight: bold; margin: 1em 0; }
.milkdown p { margin: 1em 0; line-height: 1.6; }

/* Lists */
.milkdown ul { list-style: disc; padding-left: 2em; }
.milkdown ol { list-style: decimal; padding-left: 2em; }
.milkdown li { margin: 0.25em 0; }

/* Code */
.milkdown code {
  font-family: 'SFMono-Regular', Consolas, monospace;
  background: #f5f5f5;
  padding: 0.2em 0.4em;
  border-radius: 3px;
  font-size: 0.9em;
}
.milkdown pre {
  background: #f5f5f5;
  padding: 1em;
  border-radius: 4px;
  overflow-x: auto;
}
.milkdown pre code {
  background: none;
  padding: 0;
}

/* Blockquote */
.milkdown blockquote {
  border-left: 4px solid #ddd;
  padding-left: 1em;
  margin-left: 0;
  color: #666;
}

/* Horizontal rule */
.milkdown hr {
  border: none;
  border-top: 1px solid #ddd;
  margin: 2em 0;
}

/* Links */
.milkdown a { color: #0366d6; text-decoration: underline; }

/* Images */
.milkdown img { max-width: 100%; height: auto; }

/* Tables (if using GFM) */
.milkdown table {
  border-collapse: collapse;
  width: 100%;
  margin: 1em 0;
}
.milkdown th, .milkdown td {
  border: 1px solid #ddd;
  padding: 0.5em;
  text-align: left;
}
.milkdown th { background: #f5f5f5; font-weight: bold; }

/* Task lists (GFM) */
.milkdown li[data-checked] { list-style: none; }
.milkdown li[data-checked]::before { content: '☐ '; }
.milkdown li[data-checked="true"]::before { content: '☑ '; }
```

### ProseMirror CSS Requirements

Import ProseMirror's base CSS to avoid layout issues:

```typescript
// If using @milkdown/prose or @milkdown/kit:
import '@milkdown/prose/view/style/prosemirror.css';
// Also useful:
import '@milkdown/prose/tables/style/tables.css';  // If using GFM tables
```

Or include the essential rules directly:

```css
/* ProseMirror gap cursor */
.ProseMirror-gapcursor {
  display: none;
  pointer-events: none;
  position: absolute;
}
.ProseMirror-gapcursor:after {
  content: '';
  display: block;
  position: absolute;
  top: -2px;
  width: 20px;
  border-top: 1px solid black;
  animation: ProseMirror-cursor-blink 1.1s steps(2, start) infinite;
}

/* ProseMirror selected node */
.ProseMirror-selectednode {
  outline: 2px solid #8cf;
}
```

### Using Tailwind CSS / Utility Classes

Apply classes via `editorViewOptionsCtx` and node attribute hooks:

```typescript
import { editorViewOptionsCtx } from '@milkdown/kit/core';
import { blockquoteAttr, inlineCodeAttr, headingAttr } from '@milkdown/kit/preset/commonmark';

editor.config((ctx) => {
  // Add classes to the editor container
  ctx.update(editorViewOptionsCtx, (prev) => ({
    ...prev,
    attributes: {
      class: 'prose prose-lg max-w-none focus:outline-none',
    },
  }));

  // Style individual node types
  ctx.set(blockquoteAttr.key, () => ({
    class: 'border-l-4 border-blue-400 pl-4 italic text-gray-600',
  }));

  ctx.set(inlineCodeAttr.key, () => ({
    class: 'font-mono text-sm bg-gray-100 px-1 py-0.5 rounded',
  }));

  ctx.set(headingAttr.key, (node) => ({
    class: `heading-level-${node.attrs.level}`,
  }));
});
```

If you're using Tailwind's `@tailwindcss/typography` plugin, adding `prose` to the editor container gives you sensible defaults for all elements.

## Crepe Themes

Crepe ships with pre-built themes. Always import the common styles plus one theme:

```typescript
// Required base styles
import '@milkdown/crepe/theme/common/style.css';

// Pick ONE theme:
import '@milkdown/crepe/theme/frame.css';   // Light, minimal frame
// OR
import '@milkdown/crepe/theme/nord.css';    // Nord color scheme
```

### Customizing Crepe

Override Crepe's CSS variables to adjust colors without replacing the theme:

```css
/* Override Crepe theme variables */
.milkdown {
  --crepe-color-primary: #3b82f6;
  --crepe-color-on-primary: #ffffff;
  --crepe-color-surface: #ffffff;
  --crepe-color-on-surface: #1f2937;
  --crepe-color-outline: #d1d5db;
}
```

### Crepe Feature-Specific Styles

Some Crepe features have their own CSS. They're included automatically when you import the theme, but if doing selective feature imports:

```typescript
// Individual feature styles (usually not needed separately)
import '@milkdown/crepe/theme/common/style.css';
import '@milkdown/crepe/theme/frame.css';
```

## Common Styling Patterns

### Placeholder Text

```css
/* Show placeholder when editor is empty */
.milkdown .ProseMirror p.is-editor-empty:first-child::before {
  content: attr(data-placeholder);
  float: left;
  color: #adb5bd;
  pointer-events: none;
  height: 0;
}
```

Or use Crepe's built-in placeholder feature:

```typescript
const crepe = new Crepe({
  featureConfigs: {
    [Crepe.Feature.Placeholder]: { text: 'Start writing...' },
  },
});
```

### Focus Ring

```css
.milkdown {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  transition: border-color 0.2s;
}
.milkdown:focus-within {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}
```

### Fixed Height with Scroll

```css
.milkdown {
  height: 400px;
  overflow-y: auto;
}
.milkdown .ProseMirror {
  min-height: 100%;
}
```

### Dark Mode

```css
@media (prefers-color-scheme: dark) {
  .milkdown {
    background: #1a1a2e;
    color: #e0e0e0;
  }
  .milkdown code { background: #2a2a3e; }
  .milkdown pre { background: #2a2a3e; }
  .milkdown blockquote { border-left-color: #4a4a5e; color: #a0a0b0; }
  .milkdown a { color: #58a6ff; }
  .milkdown hr { border-top-color: #3a3a4e; }
  .milkdown th { background: #2a2a3e; }
  .milkdown th, .milkdown td { border-color: #3a3a4e; }
}
```

## FAQ

### Why does my editor have no styles?

Milkdown v7 is headless. You must provide CSS. Either use Crepe (which includes themes) or write your own styles targeting `.milkdown` and standard HTML elements.

### My content overflows / looks broken

Import ProseMirror's base styles: `import '@milkdown/prose/view/style/prosemirror.css'`. Without them, contenteditable behavior can be unpredictable.

### Can I use CSS-in-JS?

Yes. Target the `.milkdown` container and its children. For node-level customization, use the `*Attr` hooks (e.g., `blockquoteAttr.key`) to inject className or style attributes.

### How do I scope styles to avoid conflicts?

Prefix all selectors with `.milkdown`:
```css
.milkdown h1 { /* only affects the editor */ }
```

Or use CSS Modules / Shadow DOM if your framework supports it.
