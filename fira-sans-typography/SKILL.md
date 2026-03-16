---
name: fira-sans-typography
description: Typography system using Fira Sans. Apply when building web UIs, design systems, or any artifact requiring consistent type. Covers font loading, typescale, spacing rules, and HTML element mapping.
---

# Fira Sans Typography

## Font Loading

**Source:** [Mozilla Fira Sans](https://github.com/mozilla/Fira) — use `.otf` or `.woff2` files from the repo for self-hosting.

**Priority order:** Self-hosted (own CDN/server/GitHub) → Google Fonts fallback

### Self-hosted (preferred — full weight control + OpenType features)

Download `.otf` or `.woff2` files from the [GitHub repo](https://github.com/mozilla/Fira/tree/master/otf) and serve from your own CDN or server.

```css
@font-face {
  font-family: 'Fira Sans';
  src: url('/fonts/FiraSans-Regular.otf') format('opentype');
  font-weight: 400;
  font-display: swap;
}
@font-face {
  font-family: 'Fira Sans';
  src: url('/fonts/FiraSans-Medium.otf') format('opentype');
  font-weight: 500;
  font-display: swap;
}
@font-face {
  font-family: 'Fira Sans';
  src: url('/fonts/FiraSans-SemiBold.otf') format('opentype');
  font-weight: 600;
  font-display: swap;
}
```

Load 400 and 500 by default. Add 300 (light) for lead text if needed, or 600 (semibold) for strong emphasis. Avoid loading more than 3 weights.

### Google Fonts (fallback — limited OpenType control)

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Fira+Sans:wght@400;500&display=swap" rel="stylesheet">
```

**Fallback stack:** `'Fira Sans', ui-sans-serif, system-ui, -apple-system, sans-serif`

## Typescale & Element Mapping

Scale is defined in `rem` with `calc()` for fluid adjustments. Base `1rem = 16px` assumed. Mobile-first — larger screens adjust up.

```css
:root {
  --font-sans: 'Fira Sans', ui-sans-serif, system-ui, -apple-system, sans-serif;

  /* Scale (rem) */
  --text-xs:   0.75rem;              /* 12px — labels, captions */
  --text-sm:   0.875rem;             /* 14px — secondary text */
  --text-base: 1rem;                 /* 16px — body copy */
  --text-md:   1.125rem;             /* 18px — lead text, h5 */
  --text-lg:   1.25rem;              /* 20px — h4 */
  --text-xl:   1.5rem;               /* 24px — h3 */
  --text-2xl:  2rem;                 /* 32px — h1 mobile */
  --text-3xl:  2.5rem;               /* 40px — h2 desktop */
  --text-4xl:  2.75rem;              /* 44px — h1 desktop */

  /* Weights */
  --weight-light:   300;
  --weight-regular: 400;
  --weight-medium:  500;
  --weight-semibold: 600;
}

/* Base — mobile first */
body {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  font-weight: var(--weight-regular);
  line-height: 1.5em;
  letter-spacing: 0;
}

/* Responsive body — wider = more leading */
@media (min-width: 640px)  { body { line-height: 1.55em; } }
@media (min-width: 1024px) { body { line-height: 1.6em; } }

/* Headings — mobile first, scale up at breakpoints */
h1 {
  font-size: var(--text-2xl);        /* 32px mobile */
  font-weight: var(--weight-medium);
  line-height: 1.2em;
  letter-spacing: -0.0175em;
}
@media (min-width: 768px) {
  h1 { font-size: var(--text-4xl); letter-spacing: -0.0175em; } /* 44px desktop */
}

h2 {
  font-size: 1.75rem;                /* 28px mobile */
  font-weight: var(--weight-medium);
  line-height: 1.2em;
  letter-spacing: 0;
}
@media (min-width: 768px) {
  h2 { font-size: var(--text-3xl); } /* 40px desktop */
}

h3 {
  font-size: var(--text-lg);         /* 20px mobile */
  font-weight: var(--weight-medium);
  line-height: 1.3em;
  letter-spacing: 0.0035em;
}
@media (min-width: 768px) {
  h3 { font-size: var(--text-xl); }  /* 24px desktop */
}

h4 {
  font-size: var(--text-md);         /* 18px */
  font-weight: var(--weight-medium);
  line-height: 1.35em;
  letter-spacing: 0;
}

h5 {
  font-size: var(--text-base);       /* 16px */
  font-weight: var(--weight-medium);
  line-height: 1.4em;
  letter-spacing: 0;
}

h6 {
  font-size: var(--text-sm);         /* 14px */
  font-weight: var(--weight-medium);
  line-height: 1.4em;
  letter-spacing: 0;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

/* Body */
p {
  font-size: var(--text-base);
  line-height: 1.5em;
  letter-spacing: 0;
}

.lead {
  font-size: var(--text-md);         /* 18px mobile */
  font-weight: var(--weight-light);
  line-height: 1.5em;
}
@media (min-width: 768px) {
  .lead { font-size: var(--text-lg); } /* 20px desktop */
}

small {
  font-size: var(--text-sm);
  line-height: 1.4em;
}

figcaption {
  font-size: var(--text-xs);
  line-height: 1.4em;
  opacity: 0.7;
}

/* UI controls */
button, .btn {
  font-family: var(--font-sans);
  font-size: var(--text-base);       /* 16px */
  font-weight: var(--weight-medium);
  line-height: 1.6em;
  letter-spacing: 0;
}

input, select, textarea {
  font-family: var(--font-sans);
  font-size: var(--text-base);
  font-weight: var(--weight-regular);
  line-height: 1.4em;
  letter-spacing: 0;
}

label {
  font-size: var(--text-sm);         /* 14px desktop */
  font-weight: var(--weight-regular);
  line-height: 1.2em;
  letter-spacing: 0.042em;
  text-transform: uppercase;
}
@media (max-width: 767px) {
  label { font-size: var(--text-xs); } /* 12px mobile */
}

/* Overline / Preheader */
.overline, .preheader {
  font-size: var(--text-base);       /* 16px desktop */
  font-weight: var(--weight-regular);
  line-height: 1.6em;
  letter-spacing: 0.109em;
  text-transform: uppercase;
}
@media (max-width: 767px) {
  .overline, .preheader { font-size: var(--text-sm); } /* 14px mobile */
}

code, pre {
  font-family: 'Fira Mono', ui-monospace, monospace;
  font-size: var(--text-sm);
}
```

## Rules (quick reference)

| Concern | Rule |
|---|---|
| Body range | 16px base; never below on mobile for core reading text |
| Minimum size | 12px (`0.75rem`) — for labels/captions only |
| Letter-spacing | Zero for most text; slight negative (-0.0175em) for large headlines (44px); positive (0.04–0.11em) for uppercase labels/overlines |
| Line-height unit | Use `em` — tracks element's own font-size |
| Line-height range | `1.2em` (single-line headings/UI) → `1.6em` (paragraphs on wide viewports) |
| All caps | Use sparingly; when needed, add `letter-spacing: 0.04–0.11em` depending on size |
| Default weights | Load 400 and 500 only; add 300 for lead text or 600 for strong emphasis if design requires |
| Weight mixing | Regular (400) for body; medium (500) for headings/buttons; light (300) for lead paragraphs |
| Headlines | Medium (500) by default; avoid mixing medium + semibold at same hierarchy level |
| Mobile-first | Base = mobile (16px body); override up at `640px`, `768px`, `1024px` |
| Fira Sans character | Humanist sans with excellent legibility; slightly wider letterforms than geometric sans — comfortable at smaller sizes |

For full desktop + mobile scale table with exact values → [TYPESCALE.md](references/TYPESCALE.md)
For OpenType features and numeric styles → [OPENTYPE.md](references/OPENTYPE.md)
For Tailwind configuration and examples → [TAILWIND.md](references/TAILWIND.md)
