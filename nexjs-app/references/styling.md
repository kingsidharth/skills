# Styling

## Pick one primary approach

- **Tailwind** for most styling. Handles 90% of design patterns with utility classes.
- **CSS Modules** for component-specific styles when utilities are awkward.
- **Global CSS** only for truly global concerns (Tailwind base, resets).
- **CSS-in-JS** only when required by a design system you're adopting.

## Tailwind (v4 default)

Install:

```bash
bun add -D tailwindcss @tailwindcss/postcss
```

`postcss.config.mjs`:

```js
export default {
  plugins: { '@tailwindcss/postcss': {} },
}
```

`app/globals.css`:

```css
@import 'tailwindcss';
```

Import once in `app/layout.tsx`:

```tsx
import './globals.css'
```

For older-browser support, use Tailwind v3 setup instead (different PostCSS config).

## CSS Modules

Filename must end in `.module.css`. Class names are locally scoped:

```css
/* app/blog/blog.module.css */
.blog { padding: 24px; }
```

```tsx
import styles from './blog.module.css'
export default function Page() {
  return <main className={styles.blog} />
}
```

## Global CSS

Import in the root layout:

```tsx
import './global.css'
```

You *can* import global CSS from any file in `app/`, but stylesheets are currently not unmounted on route change — overlapping globals between routes can conflict. Keep globals minimal.

## External stylesheets

Import package CSS anywhere:

```tsx
import 'bootstrap/dist/css/bootstrap.css'
```

In React 19+, `<link rel="stylesheet" href="..." />` inline is also supported.

## Ordering

CSS order follows **import order in your code**. If `<BaseButton>` is imported before `page.module.css`, its styles come first. For predictability:
- Keep imports in a single entry file per route
- Import globals in the root
- Disable auto-sort-imports rules (ESLint's `sort-imports`, prettier-plugin-sort-imports)

Control chunking with `cssChunking` in `next.config.ts` if default chunking causes ordering issues.

## Dev vs Prod

- Dev (`next dev`): Fast Refresh applies CSS changes instantly; JS is required.
- Prod (`next build`): CSS is concatenated, minified, code-split. Works without JS.
- CSS order can differ between dev and build — verify with `next build`.

## CSS-in-JS

Supported in **Client Components only**. Configure a **style registry** + `useServerInsertedHTML` for SSR.

Supported libraries: ant-design, chakra-ui, @fluentui/react-components, kuma-ui, @mui/material, @mui/joy, pandacss, styled-jsx, styled-components, stylex, tamagui, tss-react, vanilla-extract. Emotion support is in progress.

Skip CSS-in-JS on Server Components — use Tailwind or CSS Modules instead.

### styled-jsx pattern

```tsx
// app/registry.tsx
'use client'
import { useState } from 'react'
import { useServerInsertedHTML } from 'next/navigation'
import { StyleRegistry, createStyleRegistry } from 'styled-jsx'

export default function StyledJsxRegistry({ children }: { children: React.ReactNode }) {
  const [registry] = useState(() => createStyleRegistry())
  useServerInsertedHTML(() => {
    const styles = registry.styles()
    registry.flush()
    return <>{styles}</>
  })
  return <StyleRegistry registry={registry}>{children}</StyleRegistry>
}
```

Wrap in root layout:

```tsx
import StyledJsxRegistry from './registry'
export default function RootLayout({ children }: LayoutProps<'/'>) {
  return <html><body><StyledJsxRegistry>{children}</StyledJsxRegistry></body></html>
}
```

### styled-components pattern

Enable the compiler:

```js
// next.config.js
module.exports = { compiler: { styledComponents: true } }
```

Create a similar registry using `ServerStyleSheet` + `StyleSheetManager`. Same principle — collect on server, flush to `<head>` via `useServerInsertedHTML`.
