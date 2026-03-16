# Tailwind Usage

Configure Space Grotesk in `tailwind.config.js` first, then apply via utility classes.

## Config

```js
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      fontFamily: {
        sans: ['Space Grotesk', 'ui-sans-serif', 'system-ui', 'sans-serif'],
      },
      fontSize: {
        'xs':   ['0.6875rem', { lineHeight: '1.4em' }],   /* 11px */
        'sm':   ['0.75rem',   { lineHeight: '1.4em' }],   /* 12px */
        'base': ['1rem',      { lineHeight: '1.6em' }],   /* 16px */
        'lg':   ['1.375rem',  { lineHeight: '1.3em' }],   /* 22px */
        'xl':   ['1.5rem',    { lineHeight: '1.25em' }],  /* 24px */
        '2xl':  ['2rem',      { lineHeight: '1.2em' }],   /* 32px */
        '3xl':  ['2.5rem',    { lineHeight: '1.15em' }],  /* 40px */
      },
      letterSpacing: {
        'ui':      '-0.006em',
        'heading': '-0.015em',
        'display': '-0.02em',
      },
    },
  },
}
```

## Headline + Subheadline + Paragraph Example

```html
<article class="font-sans max-w-prose">

  <!-- h1: 32px mobile → 40px desktop, medium weight -->
  <h1 class="text-2xl md:text-3xl font-medium leading-tight tracking-heading mb-4">
    Building Better Interfaces with Space Grotesk
  </h1>

  <!-- Subheadline / lead paragraph: 22px, regular weight, more leading -->
  <p class="text-lg font-normal leading-relaxed text-gray-600 mb-6">
    A proportional sans-serif designed for legibility at small sizes, 
    with just enough character to stand out.
  </p>

  <!-- Body paragraphs: 16px, regular, comfortable leading -->
  <p class="text-base font-normal leading-relaxed mb-4">
    Space Grotesk retains the idiosyncratic details of Space Mono while 
    optimising for improved readability at non-display sizes. The result is 
    a typeface that works across UI contexts without feeling generic.
  </p>

  <p class="text-base font-normal leading-relaxed mb-4">
    Use medium weight (500) for hierarchy cues — subheadings, labels, 
    navigation — and regular (400) for everything else.
  </p>

  <!-- h2: section heading -->
  <h2 class="text-xl md:text-2xl font-medium leading-snug tracking-heading mt-10 mb-3">
    OpenType Features
  </h2>

  <p class="text-base leading-relaxed mb-4">
    Self-hosted OTF files unlock stylistic sets and numeric features not 
    available through Google Fonts.
  </p>

  <!-- Label + UI element -->
  <label class="block text-sm font-medium mb-1 tracking-ui">
    Email address
  </label>
  <input
    type="email"
    class="font-sans text-base font-normal leading-none tracking-ui px-3 py-2 border rounded"
    placeholder="you@example.com"
  />

  <!-- Caption / meta -->
  <p class="text-sm text-gray-400 mt-2">
    We'll never share your email with anyone.
  </p>

</article>
```

## Key Tailwind Classes Mapped to System

| Role | Classes |
|------|---------|
| Body | `text-base font-normal leading-relaxed` |
| Lead / subheadline | `text-lg font-normal leading-relaxed` |
| h3 | `text-lg font-medium leading-snug tracking-heading` |
| h2 | `text-xl md:text-2xl font-medium leading-snug tracking-heading` |
| h1 | `text-2xl md:text-3xl font-medium leading-tight tracking-heading` |
| Label | `text-sm font-medium leading-none tracking-ui` |
| Caption / meta | `text-sm font-normal leading-snug text-gray-400` |
| UI button | `text-base font-normal leading-none tracking-ui` |
| Tabular numbers | `tabular-nums` |
| Oldstyle numbers | `oldstyle-nums` |
| Diagonal fractions | `diagonal-fractions` |

## OpenType via Tailwind Arbitrary Values

```html
<!-- Slashed zero (self-hosted only) -->
<span class="[font-feature-settings:'zero'_1]">100</span>

<!-- Neutral alternates for body copy (self-hosted only) -->
<p class="[font-feature-settings:'ss01'_1,'ss02'_1,'ss03'_1,'ss04'_1]">
  Long-form text benefits from the single-storey alternates.
</p>

<!-- Data table cell — tabular + slashed zero + fractions -->
<td class="tabular-nums [font-feature-settings:'zero'_1,'frac'_1]">
  1/3 of 1000
</td>
```
