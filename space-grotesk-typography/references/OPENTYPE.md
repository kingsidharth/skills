# OpenType Features

Requires self-hosted `.otf` files from the [GitHub repo](https://github.com/floriankarsten/space-grotesk). **Not available via Google Fonts.**

## Stylistic Sets

Space Grotesk ships four stylistic alternates. Sets 1–4 produce a cleaner, more neutral character that aids legibility at small sizes — ideal for body text and regular weight at ≤20px. At larger sizes and headlines, leave them off to let Space Grotesk's natural flair and personality show through.

| Set | Alternate | When to enable |
|-----|-----------|----------------|
| `ss01` | Alt `a` (single-storey) | Body copy, regular weight, ≤20px — improves legibility |
| `ss02` | Alt `g` (single-storey) | Same as above |
| `ss03` | Alt `y` | Same as above |
| `ss04` | Alt capital `D` | Same as above |

```css
/* Recommended defaults for body text */
body {
  font-feature-settings: "ss01" 1, "ss02" 1, "ss03" 1, "ss04" 1;
}

/* Restore Space Grotesk's character and flair at larger sizes */
.display, h1, h2 {
  font-feature-settings: "ss01" 0, "ss02" 0, "ss03" 0, "ss04" 0;
}
```

CSS class example:
```css
.text-neutral  { font-feature-settings: "ss01" 1, "ss02" 1, "ss03" 1, "ss04" 1; }
.text-stylised { font-feature-settings: "ss01" 0, "ss02" 0, "ss03" 0, "ss04" 0; }
```

## Numeric Features

### Slashed Zero

Prevents confusion between `0` and `O`/`o`. Use in analytics, data tables, headlines with numbers, and small sizes.

```css
.slashed-zero { font-feature-settings: "zero" 1; }
```

Tailwind (requires custom plugin or arbitrary value):
```html
<span class="[font-feature-settings:'zero'_1]">10:00</span>
```

### Tabular Figures

Numbers align to equal widths — essential for tables, dashboards, and mobile where digits shift can cause reflow. Recommend enabling broadly.

```css
.tabular    { font-variant-numeric: tabular-nums; }
/* or */
.tabular    { font-feature-settings: "tnum" 1; }
```

Tailwind: `class="tabular-nums"`

### Oldstyle Figures

Numbers sit on the baseline like lowercase letters — stylistic. Use in headlines and editorial contexts. Use sparingly.

```css
.oldstyle   { font-variant-numeric: oldstyle-nums; }
/* or */
.oldstyle   { font-feature-settings: "onum" 1; }
```

Tailwind: `class="oldstyle-nums"`

### Fractions

Renders `1/2` as a proper stacked fraction. Use across the board for amounts, tables, measurements — improves legibility significantly.

```css
.fractions  { font-variant-numeric: diagonal-fractions; }
/* or */
.fractions  { font-feature-settings: "frac" 1; }
```

## Combining Features

```css
/* Analytics / data table — recommended combination */
.data {
  font-feature-settings: "tnum" 1, "zero" 1, "frac" 1;
}

/* Body copy — neutral, readable */
.body-copy {
  font-feature-settings: "ss01" 1, "ss02" 1, "ss03" 1, "ss04" 1, "tnum" 1, "frac" 1;
}

/* Editorial headline */
.headline {
  font-feature-settings: "onum" 1;
}
```

## Checking Feature Support

```js
document.fonts.ready.then(() => {
  const loaded = [...document.fonts].some(f =>
    f.family.includes('Space Grotesk') && f.status === 'loaded'
  );
  if (!loaded) console.warn('Space Grotesk not loaded — OpenType features unavailable');
});
```
