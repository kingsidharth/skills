# Theming & Styling Reference

## Content Styling with `contentEditableClassName`

The `contentEditableClassName` prop applies CSS to the editable content area. This is how you style headings, paragraphs, lists, code, etc. inside the editor.

```tsx
<MDXEditor
  markdown="Hello **world**!"
  contentEditableClassName="prose"
/>
```

**Important**: Keep selectors conservative — avoid targeting generic `div` elements. The content area also contains editor UI elements (table controls, frontmatter editor, code block editor) that you don't want to break.

## Tailwind Typography Integration

The `@tailwindcss/typography` plugin provides the `prose` class — perfect for styling MDXEditor content since it targets the exact HTML elements that markdown produces.

### Setup

```bash
npm install -D @tailwindcss/typography
```

**Tailwind CSS v4** (CSS-first config):

```css
/* main.css */
@import "tailwindcss";
@plugin "@tailwindcss/typography";
```

**Tailwind CSS v3** (JS config):

```js
// tailwind.config.js
module.exports = {
  plugins: [require('@tailwindcss/typography')],
}
```

### Basic Usage

```tsx
<MDXEditor
  markdown={markdown}
  contentEditableClassName="prose"
  plugins={[...]}
/>
```

### Prose Modifiers

| Class | Effect |
|---|---|
| `prose` | Base typography styles (default gray) |
| `prose-sm` | Smaller text scale |
| `prose-lg` | Larger text scale |
| `prose-xl` | Extra large text scale |
| `prose-slate` / `prose-stone` / `prose-zinc` | Color themes |
| `prose-invert` | Inverted colors (for dark backgrounds) |
| `dark:prose-invert` | Dark mode auto-inversion |
| `max-w-none` | Remove max-width constraint (editor often needs full width) |

### Recommended Configuration

```tsx
<MDXEditor
  markdown={markdown}
  contentEditableClassName="prose prose-sm sm:prose-base max-w-none dark:prose-invert"
  plugins={[...]}
/>
```

### Customizing Prose Styles

Override typography defaults in `tailwind.config.js`:

```js
module.exports = {
  theme: {
    extend: {
      typography: {
        DEFAULT: {
          css: {
            color: '#333',
            a: {
              color: '#3182ce',
              '&:hover': { color: '#2c5282' },
            },
            h1: { color: '#1e1e1e' },
            'code::before': { content: '""' },
            'code::after': { content: '""' },
          },
        },
      },
    },
  },
  plugins: [require('@tailwindcss/typography')],
}
```

### Element-Level Tailwind Overrides

Use prose modifier utilities for inline overrides:

```tsx
contentEditableClassName="prose
  prose-headings:font-semibold
  prose-headings:text-gray-900
  prose-a:text-blue-600
  prose-a:no-underline
  hover:prose-a:underline
  prose-code:text-pink-600
  prose-code:before:content-none
  prose-code:after:content-none
  prose-img:rounded-lg
  prose-blockquote:border-blue-500
  max-w-none"
```

### Custom `className` Option (Rename `prose`)

If `prose` conflicts with your project:

**Tailwind v4**:
```css
@plugin "@tailwindcss/typography" { className: wysiwyg; }
```

**Tailwind v3**:
```js
// tailwind.config.js — currently not directly supported, use wrapper class
```

Then use `contentEditableClassName="wysiwyg"` instead.

## Editor UI Theming (CSS Variables)

The editor toolbar, dialogs, and popups are themed via CSS variables on the root editor element. Colors follow the [Radix semantic aliasing](https://www.radix-ui.com/colors/docs/overview/aliasing#semantic-aliases) convention.

### Public CSS Classes

| Class | Element |
|---|---|
| `.mdxeditor` | Root element + popup container |
| `.mdxeditor-popup-container` | Popup container (for z-index overrides) |
| `.mdxeditor-toolbar` | Toolbar container |
| `.mdxeditor-root-contenteditable` | The root contentEditable element |
| `.mdxeditor-diff-source-wrapper` | Wrapper when diffSourcePlugin is active |
| `.mdxeditor-rich-text-editor` | Rich text mode root |
| `.mdxeditor-source-editor` | Source mode root |
| `.mdxeditor-diff-editor` | Diff mode root |
| `.mdxeditor-select-content` | Radix Select.Content (for dropdown styling) |

### Dark Mode

Add `dark-theme` class to the editor root to flip to dark mode (uses Radix dark color tokens):

```tsx
<MDXEditor
  className="dark-theme dark-editor"
  markdown={markdown}
  plugins={[...]}
/>
```

### Custom Color Theme

Override the accent and base color scales. Import Radix color tokens or define your own:

```css
@import url('@radix-ui/colors/tomato-dark.css');
@import url('@radix-ui/colors/mauve-dark.css');

.dark-editor {
  /* Accent colors (buttons, links, active states) */
  --accentBase: var(--tomato-1);
  --accentBgSubtle: var(--tomato-2);
  --accentBg: var(--tomato-3);
  --accentBgHover: var(--tomato-4);
  --accentBgActive: var(--tomato-5);
  --accentLine: var(--tomato-6);
  --accentBorder: var(--tomato-7);
  --accentBorderHover: var(--tomato-8);
  --accentSolid: var(--tomato-9);
  --accentSolidHover: var(--tomato-10);
  --accentText: var(--tomato-11);
  --accentTextContrast: var(--tomato-12);

  /* Base colors (backgrounds, text) */
  --baseBase: var(--mauve-1);
  --baseBgSubtle: var(--mauve-2);
  --baseBg: var(--mauve-3);
  --baseBgHover: var(--mauve-4);
  --baseBgActive: var(--mauve-5);
  --baseLine: var(--mauve-6);
  --baseBorder: var(--mauve-7);
  --baseBorderHover: var(--mauve-8);
  --baseSolid: var(--mauve-9);
  --baseSolidHover: var(--mauve-10);
  --baseText: var(--mauve-11);
  --baseTextContrast: var(--mauve-12);

  /* Admonition colors */
  --admonitionTipBg: var(--cyan4);
  --admonitionTipBorder: var(--cyan8);
  --admonitionInfoBg: var(--grass4);
  --admonitionInfoBorder: var(--grass8);
  --admonitionCautionBg: var(--amber4);
  --admonitionCautionBorder: var(--amber8);
  --admonitionDangerBg: var(--red4);
  --admonitionDangerBorder: var(--red8);
  --admonitionNoteBg: var(--mauve-4);
  --admonitionNoteBorder: var(--mauve-8);

  /* Fonts */
  font-family: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  --font-mono: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;

  /* Background */
  color: var(--baseText);
  --basePageBg: black;
  background: var(--basePageBg);
}
```

```tsx
<MDXEditor className="dark-theme dark-editor" markdown={markdown} plugins={[...]} />
```

### Responsive Dark Mode with Tailwind

```tsx
import { useTheme } from 'next-themes'

function Editor({ markdown }) {
  const { resolvedTheme } = useTheme()
  const isDark = resolvedTheme === 'dark'

  return (
    <MDXEditor
      key={resolvedTheme}  // Force re-mount on theme change
      className={isDark ? 'dark-theme dark-editor' : ''}
      contentEditableClassName={`prose max-w-none ${isDark ? 'prose-invert' : ''}`}
      markdown={markdown}
      plugins={[...]}
    />
  )
}
```

## CodeMirror Dark Theme

For code blocks, use `cm6-theme-basic-dark`:

```bash
npm install cm6-theme-basic-dark
```

```tsx
import { basicDark } from 'cm6-theme-basic-dark'
import { codeMirrorPlugin } from '@mdxeditor/editor'

const isDark = resolvedTheme === 'dark'

codeMirrorPlugin({
  codeBlockLanguages: { js: 'JavaScript', css: 'CSS' },
  codeMirrorExtensions: isDark ? [basicDark] : []
})
```
