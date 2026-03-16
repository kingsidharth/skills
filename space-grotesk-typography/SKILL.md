---
name: space-grotesk-typography
description: Typography system using Space Grotesk. Apply when building web UIs, design systems, or any artifact requiring consistent type. Covers font loading, typescale, spacing rules, and HTML element mapping.
---

# Space Grotesk Typography

## Font Loading

**Source:** [floriankarsten/space-grotesk](https://github.com/floriankarsten/space-grotesk) — use `.otf` files from the repo for self-hosting.

**Priority order:** Self-hosted (own CDN/server/GitHub) → Google Fonts fallback

### Self-hosted (preferred — enables OpenType features)

Download `.otf` files from the [GitHub repo](https://github.com/floriankarsten/space-grotesk/tree/master/fonts/otf) and serve from your own CDN or server.

```css
@font-face {
  font-family: 'Space Grotesk';
  src: url('/fonts/SpaceGrotesk-Regular.otf') format('opentype');
  font-weight: 400;
  font-display: swap;
}
@font-face {
  font-family: 'Space Grotesk';
  src: url('/fonts/SpaceGrotesk-Medium.otf') format('opentype');
  font-weight: 500;
  font-display: swap;
}
```

Load 400 and 500 only by default. Add 300/700 only if the design explicitly needs them.

### Google Fonts (fallback — no OpenType features)

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500&display=swap" rel="stylesheet">
```

## Typescale & Element Mapping

Scale is defined in `rem` with `calc()` for fluid adjustments. Base `1rem = 16px` assumed. Mobile-first — larger screens adjust up.

```css
:root {
  --font-sans: 'Space Grotesk', ui-sans-serif, system-ui, sans-serif;

  /* Scale (rem) */
  --text-xs:   0.6875rem;            /* 11px — absolute minimum */
  --text-sm:   0.75rem;              /* 12px — captions, labels */
  --text-base: 1rem;                 /* 16px — body copy */
  --text-md:   calc(1rem + 0.0625vw);/* ~17–18px, fluid */
  --text-lg:   1.375rem;             /* 22px — lead text */
  --text-xl:   1.5rem;               /* 24px — h3 */
  --text-2xl:  2rem;                 /* 32px — h2 */
  --text-3xl:  2.5rem;               /* 40px — h1 */

  /* Weights */
  --weight-regular: 400;
  --weight-medium:  500;
}

/* Base — mobile first */
body {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  font-weight: var(--weight-regular);
  line-height: 1.55em;
}

/* Responsive body — wider = more leading */
@media (min-width: 640px)  { body { line-height: 1.6em; } }
@media (min-width: 1024px) { body { line-height: 1.65em; } }

/* Headings — mobile first, scale up at breakpoints */
h1 {
  font-size: var(--text-2xl);        /* 32px mobile */
  font-weight: var(--weight-medium);
  line-height: 1.2em;
  letter-spacing: -0.015em;
}
@media (min-width: 768px) {
  h1 { font-size: var(--text-3xl); letter-spacing: -0.02em; } /* 40px desktop */
}

h2 {
  font-size: var(--text-xl);         /* 24px mobile */
  font-weight: var(--weight-medium);
  line-height: 1.25em;
  letter-spacing: -0.01em;
}
@media (min-width: 768px) {
  h2 { font-size: var(--text-2xl); letter-spacing: -0.015em; } /* 32px desktop */
}

h3 {
  font-size: var(--text-lg);         /* 22px */
  font-weight: var(--weight-medium);
  line-height: 1.3em;
  letter-spacing: -0.005em;
}

h4 {
  font-size: var(--text-md);
  font-weight: var(--weight-medium);
  line-height: 1.35em;
}

h5, h6 {
  font-size: var(--text-base);
  font-weight: var(--weight-medium);
  line-height: 1.4em;
}

/* Body */
p          { font-size: var(--text-base); line-height: 1.6em; }
small      { font-size: var(--text-sm); line-height: 1.4em; }
figcaption { font-size: var(--text-sm); line-height: 1.4em; opacity: 0.7; }

/* UI controls */
button, input, select, textarea {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  font-weight: var(--weight-regular);
  line-height: 1.2em;
  letter-spacing: -0.006em;
}

label {
  font-size: var(--text-sm);
  font-weight: var(--weight-medium);
  line-height: 1.2em;
}

code, pre {
  font-family: 'Space Mono', ui-monospace, monospace;
  font-size: var(--text-sm);
}
```

## Rules (quick reference)

| Concern | Rule |
|---|---|
| Body range | 16–18px effective |
| Minimum size | 11px (`0.6875rem`) — never go below |
| Letter-spacing | None below ~18px; `-0.006em` for 16–18px UI; up to `-0.02em` for headlines >20px |
| Line-height unit | Use `em` — tracks element's own font-size |
| Line-height range | `1.2em` (single-line UI) → `1.65em` (wide-viewport paragraphs) |
| All caps | Avoid; if needed, `letter-spacing: 0.05em` |
| Default weights | Load 400 and 500 only |
| Weight mixing | Regular + medium for UI/body; don't mix medium and bold within the same hierarchy level |
| Headlines | Medium by default; bold only when medium reads as insufficiently strong — not both |
| Mobile-first | Base = mobile; override up at `640px`, `768px`, `1024px` |

For display sizes, full scale table, fluid type → [TYPESCALE.md](references/TYPESCALE.md)
For OpenType features (self-hosted only) → [OPENTYPE.md](references/OPENTYPE.md)
For Tailwind usage with examples → [TAILWIND.md](references/TAILWIND.md)
