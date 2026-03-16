# Fira Sans Typescale Reference

Complete desktop and mobile type scale with exact values derived from design tokens and optimized for screen readability.

## Desktop Scale (≥768px)

| Token | Size | Line-height | Letter-spacing | Weight | Element | Usage |
|---|---|---|---|---|---|---|
| `display` / `h1` | 44px (2.75rem) | 52.8px (1.2em) | -0.77px (-0.0175em) | 500 | `<h1>` | Hero headings, page titles |
| `h2` | 40px (2.5rem) | 48px (1.2em) | 0 | 500 | `<h2>` | Section headings |
| `h3` | 24px (1.5rem) | 31.2px (1.3em) | 0.084px (0.0035em) | 500 | `<h3>` | Card titles, subsections |
| `h4` | 20px (1.25rem) | 27px (1.35em) | 0 | 500 | `<h4>` | Smaller subsections |
| `h5` | 18px (1.125rem) | 25.2px (1.4em) | 0 | 500 | `<h5>` | Minor headings |
| `h6` / `overline` | 16px (1rem) | 25.6px (1.6em) | 1.75px (0.109em) | 400 | `.overline` | Preheaders (uppercase) |
| `lead` | 20px (1.25rem) | 30px (1.5em) | 0 | 300 | `.lead`, `<p class="lead">` | Lead paragraphs, intro text |
| `body-lg` | 18px (1.125rem) | 24.3px (1.35em) | 0 | 400 | — | Sub-headings, emphasized text |
| `body` | 16px (1rem) | 24px (1.5em) | 0 | 400 | `<p>` | Standard body copy |
| `body-sm` | 14px (0.875rem) | 19.6px (1.4em) | 0 | 400 | `<small>` | Secondary text, descriptions |
| `caption` | 12px (0.75rem) | 16.8px (1.4em) | 0 | 400 | `<figcaption>` | Image captions, footnotes |
| `button` / `cta` | 16px (1rem) | 25.6px (1.6em) | 0 | 500 | `<button>`, `.btn` | Buttons, CTAs |
| `label` | 14px (0.875rem) | 16.8px (1.2em) | 0.588px (0.042em) | 400 | `<label>` | Form labels (uppercase) |
| `numeric` | 24px (1.5rem) | 31.2px (1.3em) | 0.084px (0.0035em) | 500 | `.numeric` | Stats, data points |

## Mobile Scale (<768px)

| Token | Size | Line-height | Letter-spacing | Weight | Element | Usage |
|---|---|---|---|---|---|---|
| `display` / `h1` | 32px (2rem) | 38.4px (1.2em) | -0.56px (-0.0175em) | 500 | `<h1>` | Hero headings, page titles |
| `h2` | 28px (1.75rem) | 33.6px (1.2em) | 0 | 500 | `<h2>` | Section headings |
| `h3` | 20px (1.25rem) | 26px (1.3em) | 0.07px (0.0035em) | 500 | `<h3>` | Card titles, subsections |
| `h4` | 18px (1.125rem) | 24.3px (1.35em) | 0 | 500 | `<h4>` | Smaller subsections |
| `h5` | 16px (1rem) | 22.4px (1.4em) | 0 | 500 | `<h5>` | Minor headings |
| `h6` / `overline` | 14px (0.875rem) | 22.4px (1.6em) | 1.75px (0.125em) | 400 | `.overline` | Preheaders (uppercase) |
| `lead` | 18px (1.125rem) | 27px (1.5em) | 0 | 300 | `.lead` | Lead paragraphs, intro text |
| `body-lg` | 16px (1rem) | 21.6px (1.35em) | 0 | 400 | — | Sub-headings, emphasized text |
| `body` | 16px (1rem) | 24px (1.5em) | 0 | 400 | `<p>` | Standard body copy |
| `body-sm` | 14px (0.875rem) | 19.6px (1.4em) | 0 | 400 | `<small>` | Secondary text, descriptions |
| `caption` | 12px (0.75rem) | 16.8px (1.4em) | 0 | 400 | `<figcaption>` | Image captions, footnotes |
| `button` / `cta` | 16px (1rem) | 25.6px (1.6em) | 0 | 500 | `<button>`, `.btn` | Buttons, CTAs |
| `label` | 12px (0.75rem) | 14.4px (1.2em) | 0.504px (0.042em) | 400 | `<label>` | Form labels (uppercase) |
| `numeric` | 16px (1rem) | 20.8px (1.3em) | 0.056px (0.0035em) | 500 | `.numeric` | Stats, data points |

## Fluid Type (Advanced)

For smooth scaling between mobile and desktop, use `clamp()`:

```css
/* H1 — scales from 32px (mobile) to 44px (desktop) */
h1 {
  font-size: clamp(2rem, 1.5rem + 2vw, 2.75rem);
  line-height: 1.2em;
  letter-spacing: -0.0175em;
  font-weight: 500;
}

/* H2 — scales from 28px to 40px */
h2 {
  font-size: clamp(1.75rem, 1.25rem + 2vw, 2.5rem);
  line-height: 1.2em;
  font-weight: 500;
}

/* Body — stays 16px but line-height adapts */
p {
  font-size: 1rem;
  line-height: clamp(1.5em, 1.4em + 0.5vw, 1.6em);
}
```

## Semantic HTML Mapping

| HTML Element | Desktop | Mobile | Notes |
|---|---|---|---|
| `<h1>` | 44px / 500 | 32px / 500 | Negative letter-spacing for optical balance |
| `<h2>` | 40px / 500 | 28px / 500 | Zero letter-spacing |
| `<h3>` | 24px / 500 | 20px / 500 | Subtle positive spacing (0.0035em) |
| `<h4>` | 20px / 500 | 18px / 500 | — |
| `<h5>` | 18px / 500 | 16px / 500 | — |
| `<h6>` | 16px / 400 | 14px / 400 | Uppercase with extra spacing |
| `<p>` | 16px / 400 | 16px / 400 | Default body text |
| `<small>` | 14px / 400 | 14px / 400 | De-emphasized text |
| `<figcaption>` | 12px / 400 | 12px / 400 | Minimum readable size |
| `<button>` | 16px / 500 | 16px / 500 | Consistent across breakpoints |
| `<label>` | 14px / 400 | 12px / 400 | Uppercase, extra spacing |

## Usage Guidelines

### When to use each token

**Display / H1:**
- Hero sections
- Landing page main headings
- Article titles on dedicated pages

**H2:**
- Major section dividers
- Content block headings
- Feature highlights

**H3:**
- Card titles
- List section headings
- Sidebar headings

**H4–H6:**
- Sub-subsections
- Tertiary headings
- Overlines (H6 as uppercase preheader)

**Lead:**
- First paragraph of articles
- Intro text blocks
- Hero subheadings (when lighter weight is desired)

**Body:**
- All standard paragraph content
- Default reading text
- Never go below 16px on mobile for core reading

**Body-sm:**
- Descriptions under cards
- Metadata (dates, authors)
- Secondary information

**Caption:**
- Image captions
- Footnotes
- Disclaimers
- Absolute minimum readable size

**Button / CTA:**
- Primary and secondary buttons
- Call-to-action links
- Form submit buttons

**Label:**
- Form field labels
- Data table headers
- Categorical tags

**Numeric:**
- Statistics displays
- Data highlights
- Metric callouts

## Line-height Best Practices

| Use case | Recommended line-height | Reasoning |
|---|---|---|
| Large headlines (>32px) | 1.1–1.2em | Tight leading for visual impact |
| Headings (20–32px) | 1.2–1.3em | Balance between density and readability |
| Body copy (16–18px) | 1.5–1.6em | Generous leading for comfortable reading |
| Small text (12–14px) | 1.4em | Tighter to prevent excessive whitespace |
| Buttons / UI | 1.2–1.6em | Depends on padding; tighter for compact buttons |

## Letter-spacing Rules

1. **Large headings (≥32px):** Slight negative tracking (-0.0175em) — compensates for optical spacing at large sizes
2. **Medium headings (20–28px):** Zero or very subtle positive (0–0.0035em)
3. **Body text (16–18px):** Zero — Fira Sans is well-spaced by default
4. **Small text (12–14px):** Zero for normal text; positive (0.04–0.125em) for uppercase labels
5. **Uppercase text:** Always add positive spacing (minimum 0.04em, up to 0.125em for small sizes)

## Responsive Breakpoints

Use these breakpoints to match the scale:

```css
/* Mobile: base styles */
/* Tablet: 640px */
@media (min-width: 640px) { /* Adjust body line-height */ }

/* Desktop: 768px */
@media (min-width: 768px) { /* Scale up h1, h2, h3, lead */ }

/* Wide: 1024px */
@media (min-width: 1024px) { /* Further line-height adjustments */ }
```

## Converting to rem

All sizes assume `1rem = 16px` (browser default).

| px | rem | em equivalent (if font-size = 16px) |
|---|---|---|
| 12px | 0.75rem | 0.75em |
| 14px | 0.875rem | 0.875em |
| 16px | 1rem | 1em |
| 18px | 1.125rem | 1.125em |
| 20px | 1.25rem | 1.25em |
| 24px | 1.5rem | 1.5em |
| 28px | 1.75rem | 1.75em |
| 32px | 2rem | 2em |
| 40px | 2.5rem | 2.5em |
| 44px | 2.75rem | 2.75em |

**Letter-spacing conversion:**
- Figma letter-spacing (px) ÷ font-size (px) = em value
- Example: 0.84px on 24px text = 0.84 ÷ 24 = 0.035em (round to 0.0035em)

**Line-height conversion:**
- Figma line-height (px) ÷ font-size (px) = em value
- Example: 52.8px on 44px text = 52.8 ÷ 44 = 1.2em
