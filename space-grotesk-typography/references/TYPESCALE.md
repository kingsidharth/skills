# Typescale Reference

All values in `rem` (base `1rem = 16px`). Mobile-first — breakpoints adjust scale upward.

## Scale Table

| Token     | Mobile rem | Mobile px | Desktop rem | Desktop px | Line-height | Usage                     |
|-----------|-----------|-----------|------------|------------|-------------|---------------------------|
| `xs`      | 0.6875    | 11        | 0.6875     | 11         | 1.4em       | Absolute minimum          |
| `sm`      | 0.75      | 12        | 0.75       | 12         | 1.4em       | Captions, labels, meta    |
| `base`    | 1         | 16        | 1          | 16         | 1.55–1.65em | Body copy                 |
| `md`      | fluid     | ~17       | fluid      | ~18        | 1.35em      | UI controls, lead text    |
| `lg`      | 1.375     | 22        | 1.375      | 22         | 1.3em       | h3, lead paragraphs       |
| `xl`      | 1.5       | 24        | 1.5        | 24         | 1.25em      | h2 mobile / h3 desktop    |
| `2xl`     | 2         | 32        | 2          | 32         | 1.2em       | h1 mobile / h2 desktop    |
| `3xl`     | 2.5       | 40        | 2.5        | 40         | 1.15em      | h1 desktop                |

## Fluid `md` Step

```css
--text-md: calc(1rem + 0.0625vw);
/* 320px viewport → 16.2px */
/* 1440px viewport → 17.1px */
/* Clamp for safety: */
--text-md: clamp(1.0625rem, calc(1rem + 0.0625vw), 1.125rem);
```

## Responsive Headings (mobile-first)

```css
/* h1: 32px mobile → 40px desktop */
h1 { font-size: 2rem; }
@media (min-width: 768px) { h1 { font-size: 2.5rem; } }

/* h2: 24px mobile → 32px desktop */
h2 { font-size: 1.5rem; }
@media (min-width: 768px) { h2 { font-size: 2rem; } }

/* h3: 22px, stable */
h3 { font-size: 1.375rem; }
```

## Letter-spacing Scale (em)

| Size range     | Letter-spacing |
|----------------|---------------|
| ≤18px          | none (`0`)    |
| 16–18px (UI)   | `-0.006em`    |
| 20–24px        | `-0.01em`     |
| 24–32px        | `-0.015em`    |
| 32–40px        | `-0.02em`     |
| >40px display  | `-0.02em` max |

## Display / Hero Sizes

For marketing pages only — outside the base system:

```css
.display {
  font-size: clamp(2.5rem, 5vw, 4rem);  /* 40–64px fluid */
  font-weight: 500;
  line-height: 1.1em;
  letter-spacing: -0.02em;
}
```

## Spacing Rhythm (4px grid)

```css
h1 { margin-bottom: 1rem; }      /* 16px */
h2 { margin-bottom: 0.75rem; }   /* 12px */
h3 { margin-bottom: 0.5rem; }    /* 8px */
p  { margin-bottom: 1rem; }      /* 16px */
p + p { margin-top: -0.5rem; }   /* tighten consecutive paragraphs */
```
