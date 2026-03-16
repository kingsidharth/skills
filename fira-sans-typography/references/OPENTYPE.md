# Fira Sans OpenType Features

Fira Sans includes several OpenType features useful for professional typography, particularly for numeric formatting and alternative glyphs. These features are **only accessible when self-hosting** (OTF/WOFF2 files) — Google Fonts provides limited OpenType control.

## Available Features

### 1. Tabular Figures (`tnum`)

**Purpose:** Align numbers vertically in tables and data displays

**Use when:**
- Financial tables
- Data dashboards
- Pricing tables
- Analytics displays
- Any columnar numeric data

**CSS:**
```css
.table-numbers {
  font-variant-numeric: tabular-nums;
  /* or */
  font-feature-settings: 'tnum' 1;
}
```

**Tailwind (arbitrary value):**
```html
<div class="[font-variant-numeric:tabular-nums]">
  <div>$1,234.56</div>
  <div>$   89.00</div>
  <div>$9,876.54</div>
</div>
```

**Example:**
```html
<!-- Without tnum (proportional — misaligned) -->
<table>
  <tr><td>Revenue</td><td>$1,234.56</td></tr>
  <tr><td>Cost</td><td>$89.00</td></tr>
  <tr><td>Profit</td><td>$9,876.54</td></tr>
</table>

<!-- With tnum (tabular — aligned) -->
<table class="[font-variant-numeric:tabular-nums]">
  <tr><td>Revenue</td><td>$1,234.56</td></tr>
  <tr><td>Cost</td><td>$   89.00</td></tr>
  <tr><td>Profit</td><td>$9,876.54</td></tr>
</table>
```

### 2. Proportional Figures (`pnum`)

**Purpose:** Default numeric style — better for running text

**Use when:**
- Numbers embedded in paragraphs
- Dates and times in prose
- Addresses

**CSS:**
```css
p {
  font-variant-numeric: proportional-nums; /* Default, usually not needed */
}
```

This is the default in Fira Sans, so you typically don't need to set it unless overriding a parent's `tnum`.

### 3. Oldstyle Figures (`onum`)

**Purpose:** Lowercase numbers that blend into text (some ascenders/descenders)

**Use when:**
- Editorial content
- Long-form articles
- Book-style typography
- When you want numbers to be less visually dominant

**CSS:**
```css
article p {
  font-variant-numeric: oldstyle-nums;
  /* or */
  font-feature-settings: 'onum' 1;
}
```

**Tailwind:**
```html
<p class="[font-variant-numeric:oldstyle-nums]">
  Founded in 1984, the company grew to over 5,000 employees by 2020.
</p>
```

### 4. Lining Figures (`lnum`)

**Purpose:** Uppercase-height numbers (default in most digital contexts)

**Use when:**
- Headlines
- All-caps text
- UI elements
- Data displays

**CSS:**
```css
h1 {
  font-variant-numeric: lining-nums; /* Usually default */
}
```

This is typically the default, so you rarely need to specify it.

### 5. Slashed Zero (`zero`)

**Purpose:** Distinguish zero (0) from uppercase O

**Use when:**
- Code snippets
- Serial numbers
- Technical documentation
- Any context where 0/O confusion is possible

**CSS:**
```css
code, .monospace, .serial {
  font-feature-settings: 'zero' 1;
}
```

**Tailwind:**
```html
<code class="[font-feature-settings:'zero'_1]">
  Order #00123456
</code>
```

### 6. Fractions (`frac`)

**Purpose:** Automatically format 1/2, 3/4, etc. as proper typographic fractions

**CSS:**
```css
.recipe, .measurements {
  font-feature-settings: 'frac' 1;
}
```

**Note:** This requires the fraction to be typed with a slash (e.g., `1/2`). Not all font weights may support full fraction sets.

## Combining Features

You can enable multiple features simultaneously:

```css
/* Tabular + slashed zero for data tables with IDs */
.data-table {
  font-variant-numeric: tabular-nums;
  font-feature-settings: 'tnum' 1, 'zero' 1;
}

/* Oldstyle numbers in editorial content */
article p {
  font-variant-numeric: oldstyle-nums proportional-nums;
}
```

**Tailwind (multiple features):**
```html
<div class="[font-feature-settings:'tnum'_1,'zero'_1]">
  <!-- Tabular numbers with slashed zero -->
</div>
```

## Practical Use Cases

### Data Dashboards

```css
.dashboard-metric {
  font-size: 24px;
  font-weight: 500;
  font-variant-numeric: tabular-nums;
  font-feature-settings: 'tnum' 1;
}
```

### Analytics Tables

```css
.analytics-table {
  font-variant-numeric: tabular-nums;
  letter-spacing: 0.0035em; /* Match h3 spacing for visual consistency */
}

.analytics-table td {
  text-align: right; /* Align numbers to the right */
}
```

### Editorial Numbers

```css
article {
  font-variant-numeric: oldstyle-nums;
}

article h1, article h2 {
  font-variant-numeric: lining-nums; /* Headlines stay uppercase-height */
}
```

### Code / Technical

```css
code, pre, .technical {
  font-family: 'Fira Mono', monospace;
  font-feature-settings: 'zero' 1; /* Always use slashed zero */
}
```

### Mixed Content (Reset Pattern)

```css
/* Global: proportional lining (default) */
body {
  font-variant-numeric: lining-nums proportional-nums;
}

/* Tables: switch to tabular */
table {
  font-variant-numeric: tabular-nums;
}

/* Articles: switch to oldstyle */
article p {
  font-variant-numeric: oldstyle-nums;
}

/* Code: slashed zero */
code {
  font-feature-settings: 'zero' 1;
}
```

## Feature Support Table

| Feature | CSS Property | font-feature-settings | Use case |
|---|---|---|---|
| Tabular figures | `font-variant-numeric: tabular-nums` | `'tnum' 1` | Tables, dashboards, pricing |
| Proportional figures | `font-variant-numeric: proportional-nums` | `'pnum' 1` | Running text (default) |
| Oldstyle figures | `font-variant-numeric: oldstyle-nums` | `'onum' 1` | Editorial, long-form content |
| Lining figures | `font-variant-numeric: lining-nums` | `'lnum' 1` | Headlines, UI (default) |
| Slashed zero | — | `'zero' 1` | Code, serials, technical docs |
| Fractions | — | `'frac' 1` | Recipes, measurements |

## Browser Support

- **font-variant-numeric:** Modern browsers (Chrome 52+, Firefox 34+, Safari 9.1+)
- **font-feature-settings:** Wider support, use as fallback

**Recommended approach:**

```css
.feature-class {
  font-variant-numeric: tabular-nums; /* Modern, semantic */
  font-feature-settings: 'tnum' 1;    /* Fallback for older browsers */
}
```

## Google Fonts Limitation

When using Google Fonts, OpenType feature control is **limited**. You can still use `font-variant-numeric` and `font-feature-settings`, but:

1. Not all features may render correctly
2. Some advanced features (like stylistic sets) are unavailable
3. Performance may be slightly worse

**Recommendation:** Self-host Fira Sans OTF/WOFF2 files for full OpenType control.

## Fira Sans vs. Fira Mono

For code blocks and technical content, consider using **Fira Mono** (Fira's monospaced companion):

```css
code, pre {
  font-family: 'Fira Mono', ui-monospace, 'Courier New', monospace;
  font-feature-settings: 'zero' 1; /* Slashed zero always */
}
```

Fira Mono pairs perfectly with Fira Sans and shares the same x-height for visual harmony.

## Testing Features

To verify OpenType features are working:

```html
<!-- Test slashed zero -->
<p style="font-feature-settings: 'zero' 0;">Without: 0O0O0O</p>
<p style="font-feature-settings: 'zero' 1;">With: 0O0O0O</p>

<!-- Test tabular vs proportional -->
<div style="font-variant-numeric: proportional-nums;">
  <div>11111</div>
  <div>88888</div>
</div>
<div style="font-variant-numeric: tabular-nums;">
  <div>11111</div>
  <div>88888</div>
</div>

<!-- Test oldstyle vs lining -->
<p style="font-variant-numeric: lining-nums;">Lining: 1234567890</p>
<p style="font-variant-numeric: oldstyle-nums;">Oldstyle: 1234567890</p>
```

## Advanced: Stylistic Sets (if available)

Fira Sans may include stylistic sets for alternative glyphs. Check the font file's metadata:

```css
/* Example — verify your font version supports these */
.stylistic-set-1 {
  font-feature-settings: 'ss01' 1; /* Alternative 'a' or other glyphs */
}
```

**Note:** Stylistic set availability varies by Fira Sans version. Self-hosted OTF files give you the most control. Inspect font files with tools like FontForge or use browser DevTools to test.

## Summary

For most projects:

1. **Load self-hosted OTF/WOFF2** for full feature access
2. **Use `tabular-nums` on all data tables and dashboards**
3. **Use `zero` on code and technical content**
4. **Consider `oldstyle-nums` for editorial/long-form content**
5. **Test features in your target browsers**

Fira Sans's OpenType capabilities make it excellent for both UI and content-heavy applications when properly configured.
