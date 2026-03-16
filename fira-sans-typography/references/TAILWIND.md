# Fira Sans Tailwind Configuration

Complete Tailwind setup for the Fira Sans typography system with custom font sizes, weights, and utility classes.

## Tailwind Config

Add this to your `tailwind.config.js`:

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  theme: {
    extend: {
      fontFamily: {
        sans: ['Fira Sans', 'ui-sans-serif', 'system-ui', '-apple-system', 'sans-serif'],
        mono: ['Fira Mono', 'ui-monospace', 'Courier New', 'monospace'],
      },
      fontSize: {
        // Base scale
        'xs': ['0.75rem', { lineHeight: '1.4em' }],      // 12px
        'sm': ['0.875rem', { lineHeight: '1.4em' }],     // 14px
        'base': ['1rem', { lineHeight: '1.5em' }],       // 16px
        'md': ['1.125rem', { lineHeight: '1.35em' }],    // 18px
        'lg': ['1.25rem', { lineHeight: '1.35em' }],     // 20px
        'xl': ['1.5rem', { lineHeight: '1.3em' }],       // 24px
        '2xl': ['2rem', { lineHeight: '1.2em' }],        // 32px
        '3xl': ['2.5rem', { lineHeight: '1.2em' }],      // 40px
        '4xl': ['2.75rem', { lineHeight: '1.2em' }],     // 44px
        
        // Semantic tokens
        'display': ['2.75rem', { lineHeight: '1.2em', letterSpacing: '-0.0175em', fontWeight: '500' }],
        'h1': ['2rem', { lineHeight: '1.2em', letterSpacing: '-0.0175em', fontWeight: '500' }],
        'h2': ['1.75rem', { lineHeight: '1.2em', fontWeight: '500' }],
        'h3': ['1.25rem', { lineHeight: '1.3em', letterSpacing: '0.0035em', fontWeight: '500' }],
        'h4': ['1.125rem', { lineHeight: '1.35em', fontWeight: '500' }],
        'h5': ['1rem', { lineHeight: '1.4em', fontWeight: '500' }],
        'h6': ['0.875rem', { lineHeight: '1.4em', letterSpacing: '0.04em', fontWeight: '400' }],
        'lead': ['1.125rem', { lineHeight: '1.5em', fontWeight: '300' }],
        'body': ['1rem', { lineHeight: '1.5em' }],
        'body-sm': ['0.875rem', { lineHeight: '1.4em' }],
        'caption': ['0.75rem', { lineHeight: '1.4em' }],
        'button': ['1rem', { lineHeight: '1.6em', fontWeight: '500' }],
        'label': ['0.875rem', { lineHeight: '1.2em', letterSpacing: '0.042em', fontWeight: '400' }],
        'overline': ['1rem', { lineHeight: '1.6em', letterSpacing: '0.109em', fontWeight: '400' }],
      },
      fontWeight: {
        light: '300',
        normal: '400',
        medium: '500',
        semibold: '600',
      },
      letterSpacing: {
        'tighter': '-0.0175em',
        'tight': '-0.01em',
        'normal': '0',
        'wide': '0.04em',
        'wider': '0.109em',
        'widest': '0.125em',
      },
    },
  },
  plugins: [],
}
```

## CSS Setup

Add Fira Sans to your CSS (before Tailwind directives):

```css
/* Self-hosted (recommended) */
@font-face {
  font-family: 'Fira Sans';
  src: url('/fonts/FiraSans-Regular.woff2') format('woff2');
  font-weight: 400;
  font-display: swap;
}
@font-face {
  font-family: 'Fira Sans';
  src: url('/fonts/FiraSans-Medium.woff2') format('woff2');
  font-weight: 500;
  font-display: swap;
}
@font-face {
  font-family: 'Fira Sans';
  src: url('/fonts/FiraSans-Light.woff2') format('woff2');
  font-weight: 300;
  font-display: swap;
}

/* Fira Mono for code */
@font-face {
  font-family: 'Fira Mono';
  src: url('/fonts/FiraMono-Regular.woff2') format('woff2');
  font-weight: 400;
  font-display: swap;
}

@tailwind base;
@tailwind components;
@tailwind utilities;
```

Or use Google Fonts in your HTML:

```html
<link href="https://fonts.googleapis.com/css2?family=Fira+Sans:wght@300;400;500;600&family=Fira+Mono&display=swap" rel="stylesheet">
```

## Token-to-Class Mapping

| Design Token | Tailwind Classes | Desktop | Mobile | Usage |
|---|---|---|---|---|
| Display / H1 | `text-4xl font-medium tracking-tighter md:text-display` | 44px | 32px | Hero headings |
| H2 | `text-2xl font-medium md:text-3xl` | 40px | 28px | Section headings |
| H3 | `text-lg font-medium tracking-[0.0035em] md:text-xl` | 24px | 20px | Card titles |
| H4 | `text-md font-medium` | 18px | 18px | Subsections |
| H5 | `text-base font-medium` | 16px | 16px | Minor headings |
| H6 / Overline | `text-sm font-normal tracking-wide uppercase md:text-base` | 16px | 14px | Preheaders |
| Lead | `text-md font-light md:text-lg` | 20px | 18px | Intro text |
| Body Large | `text-md` | 18px | 16px | Sub-headings |
| Body | `text-base` | 16px | 16px | Paragraphs |
| Body Small | `text-sm` | 14px | 14px | Secondary text |
| Caption | `text-xs` | 12px | 12px | Footnotes |
| Button / CTA | `text-base font-medium` | 16px | 16px | Buttons |
| Label | `text-xs font-normal tracking-wide uppercase md:text-sm` | 14px | 12px | Form labels |

## Example Markup

### Hero Section

```html
<section class="bg-white py-16 px-6">
  <div class="max-w-4xl mx-auto text-center">
    <!-- Overline -->
    <p class="text-sm font-normal tracking-wider uppercase text-gray-500 mb-4 md:text-base">
      Introducing Our Platform
    </p>
    
    <!-- H1 -->
    <h1 class="text-2xl font-medium tracking-tighter text-gray-900 mb-6 md:text-4xl md:tracking-tighter">
      Build Better Products with Fira Sans Typography
    </h1>
    
    <!-- Lead paragraph -->
    <p class="text-md font-light text-gray-600 mb-8 md:text-lg">
      A comprehensive type system designed for modern web applications, 
      combining readability with professional aesthetics.
    </p>
    
    <!-- CTA Button -->
    <button class="px-6 py-3 text-base font-medium bg-blue-600 text-white rounded-lg hover:bg-blue-700">
      Get Started
    </button>
  </div>
</section>
```

### Content Section

```html
<article class="max-w-3xl mx-auto px-6 py-12">
  <!-- H2 -->
  <h2 class="text-2xl font-medium text-gray-900 mb-6 md:text-3xl">
    Design System Principles
  </h2>
  
  <!-- Body text -->
  <p class="text-base text-gray-700 mb-4">
    Our typography system is built on three core principles: readability, 
    consistency, and scalability. Every decision in this system prioritizes 
    the reading experience across all devices.
  </p>
  
  <p class="text-base text-gray-700 mb-8">
    Fira Sans, originally designed by Erik Spiekermann for Mozilla, brings 
    a humanist quality that works exceptionally well for both UI and content.
  </p>
  
  <!-- H3 -->
  <h3 class="text-lg font-medium tracking-[0.0035em] text-gray-900 mb-4 md:text-xl">
    Mobile-First Approach
  </h3>
  
  <!-- Body text -->
  <p class="text-base text-gray-700 mb-4">
    All sizes start from mobile and scale up. We never compromise readability 
    on small screens—16px is our minimum for body text.
  </p>
  
  <!-- Caption -->
  <p class="text-xs text-gray-500 italic">
    Figure 1: Type scale comparison across breakpoints
  </p>
</article>
```

### Card Component

```html
<div class="bg-white rounded-lg shadow-md p-6">
  <!-- Label -->
  <span class="text-xs font-normal tracking-wide uppercase text-blue-600 mb-2 inline-block md:text-sm">
    Feature
  </span>
  
  <!-- H3 -->
  <h3 class="text-lg font-medium tracking-[0.0035em] text-gray-900 mb-3 md:text-xl">
    Responsive Typography
  </h3>
  
  <!-- Body Small -->
  <p class="text-sm text-gray-600 mb-4">
    Automatically adapts to different screen sizes while maintaining 
    perfect readability and visual hierarchy.
  </p>
  
  <!-- Button -->
  <button class="text-base font-medium text-blue-600 hover:text-blue-700">
    Learn more →
  </button>
</div>
```

### Data Dashboard

```html
<div class="bg-white rounded-lg p-6">
  <!-- Label -->
  <p class="text-xs font-normal tracking-wide uppercase text-gray-500 mb-4 md:text-sm">
    Monthly Revenue
  </p>
  
  <!-- Numeric (with tabular figures) -->
  <p class="text-xl font-medium [font-variant-numeric:tabular-nums] text-gray-900 mb-2 md:text-[24px]">
    $124,567.89
  </p>
  
  <!-- Caption -->
  <p class="text-xs text-green-600">
    ↑ 12.5% from last month
  </p>
</div>
```

### Form Example

```html
<form class="space-y-6">
  <div>
    <!-- Label (uppercase) -->
    <label class="block text-xs font-normal tracking-wide uppercase text-gray-700 mb-2 md:text-sm">
      Email Address
    </label>
    <!-- Input -->
    <input 
      type="email" 
      class="w-full px-4 py-2 text-base border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
      placeholder="you@example.com"
    />
  </div>
  
  <div>
    <label class="block text-xs font-normal tracking-wide uppercase text-gray-700 mb-2 md:text-sm">
      Message
    </label>
    <textarea 
      rows="4"
      class="w-full px-4 py-2 text-base border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500"
    ></textarea>
    <!-- Caption -->
    <p class="text-xs text-gray-500 mt-1">
      Maximum 500 characters
    </p>
  </div>
  
  <!-- Submit button -->
  <button class="w-full px-6 py-3 text-base font-medium bg-blue-600 text-white rounded-lg hover:bg-blue-700">
    Submit
  </button>
</form>
```

### Blog Post Layout

```html
<article class="max-w-2xl mx-auto px-6 py-12">
  <!-- Metadata (caption + label) -->
  <div class="flex items-center gap-3 mb-6">
    <span class="text-xs font-normal tracking-wide uppercase text-gray-500 md:text-sm">
      Design Systems
    </span>
    <span class="text-xs text-gray-400">•</span>
    <time class="text-xs text-gray-500">March 10, 2026</time>
  </div>
  
  <!-- H1 -->
  <h1 class="text-2xl font-medium tracking-tighter text-gray-900 mb-6 md:text-4xl">
    Creating a Scalable Typography System
  </h1>
  
  <!-- Lead -->
  <p class="text-md font-light text-gray-600 mb-8 md:text-lg">
    A comprehensive guide to building type scales that work across 
    devices, teams, and design tools.
  </p>
  
  <!-- Body content -->
  <div class="prose">
    <p class="text-base text-gray-700 mb-4">
      Typography is the foundation of any design system. Get it right, 
      and everything else falls into place. Get it wrong, and even the 
      best components will feel disjointed.
    </p>
    
    <h2 class="text-2xl font-medium text-gray-900 mb-4 mt-8 md:text-3xl">
      The Challenge
    </h2>
    
    <p class="text-base text-gray-700 mb-4">
      Most teams struggle with typography because they treat it as an 
      afterthought. Font sizes are chosen arbitrarily, line-heights 
      are eyeballed, and spacing rules are inconsistent.
    </p>
    
    <h3 class="text-lg font-medium tracking-[0.0035em] text-gray-900 mb-3 mt-6 md:text-xl">
      Start with Constraints
    </h3>
    
    <p class="text-base text-gray-700 mb-4">
      Begin by defining your minimum readable size (never below 16px for 
      body text on mobile) and your maximum display size. Everything 
      else should fit within this range.
    </p>
  </div>
</article>
```

## Custom Utilities

Add these to your CSS for common patterns:

```css
@layer components {
  /* Overline style */
  .overline {
    @apply text-sm font-normal tracking-wider uppercase md:text-base;
  }
  
  /* Lead paragraph */
  .lead {
    @apply text-md font-light md:text-lg;
  }
  
  /* Stat/metric display */
  .metric {
    @apply text-xl font-medium [font-variant-numeric:tabular-nums] md:text-[24px];
  }
  
  /* Caption style */
  .caption {
    @apply text-xs text-gray-500;
  }
  
  /* Button text (for non-button elements styled as buttons) */
  .btn-text {
    @apply text-base font-medium;
  }
}
```

Usage:

```html
<p class="overline">Featured Article</p>
<p class="lead">An introduction to our new design system...</p>
<div class="metric">$45,678.90</div>
<p class="caption">Data as of March 2026</p>
```

## Responsive Patterns

### Fluid Type (Advanced)

```html
<!-- Scales smoothly from mobile to desktop -->
<h1 class="text-[clamp(2rem,1.5rem+2vw,2.75rem)] font-medium tracking-tighter">
  Fluid Headline
</h1>

<!-- Body with fluid line-height -->
<p class="text-base [line-height:clamp(1.5em,1.4em+0.5vw,1.6em)]">
  Paragraph with adaptive leading
</p>
```

### Breakpoint-Specific Overrides

```html
<!-- Different sizes at each breakpoint -->
<h2 class="text-xl sm:text-2xl md:text-3xl lg:text-4xl font-medium">
  Multi-breakpoint Heading
</h2>

<!-- Adjust letter-spacing at larger sizes -->
<h1 class="text-2xl tracking-tight md:text-4xl md:tracking-tighter">
  Optical Adjustment
</h1>
```

## OpenType Features with Tailwind

```html
<!-- Tabular figures for tables -->
<td class="[font-variant-numeric:tabular-nums]">
  $1,234.56
</td>

<!-- Slashed zero for code -->
<code class="[font-feature-settings:'zero'_1]">
  Order #00123
</code>

<!-- Oldstyle figures for editorial -->
<article class="[font-variant-numeric:oldstyle-nums]">
  <p>Founded in 1984...</p>
</article>

<!-- Multiple features -->
<div class="[font-feature-settings:'tnum'_1,'zero'_1]">
  Data with tabular nums + slashed zero
</div>
```

## Complete Page Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Fira Sans Typography Demo</title>
  <link href="https://fonts.googleapis.com/css2?family=Fira+Sans:wght@300;400;500;600&display=swap" rel="stylesheet">
  <script src="https://cdn.tailwindcss.com"></script>
  <script>
    tailwind.config = {
      theme: {
        extend: {
          fontFamily: {
            sans: ['Fira Sans', 'sans-serif'],
          },
        },
      },
    }
  </script>
</head>
<body class="font-sans bg-gray-50">
  <!-- Header -->
  <header class="bg-white border-b border-gray-200 px-6 py-4">
    <div class="max-w-6xl mx-auto flex items-center justify-between">
      <h1 class="text-lg font-medium md:text-xl">Fira Sans Demo</h1>
      <nav class="flex gap-6">
        <a href="#" class="text-base font-medium text-gray-600 hover:text-gray-900">Features</a>
        <a href="#" class="text-base font-medium text-gray-600 hover:text-gray-900">Docs</a>
        <a href="#" class="text-base font-medium text-blue-600">Sign In</a>
      </nav>
    </div>
  </header>

  <!-- Hero -->
  <section class="bg-gradient-to-b from-white to-gray-50 px-6 py-20">
    <div class="max-w-4xl mx-auto text-center">
      <p class="text-sm font-normal tracking-wider uppercase text-gray-500 mb-4 md:text-base">
        Typography System
      </p>
      <h1 class="text-2xl font-medium tracking-tighter text-gray-900 mb-6 md:text-4xl">
        Beautiful, Consistent Type at Any Scale
      </h1>
      <p class="text-md font-light text-gray-600 mb-8 md:text-lg">
        A production-ready system built on Fira Sans, designed for 
        modern web applications and design teams.
      </p>
      <div class="flex gap-4 justify-center">
        <button class="px-6 py-3 text-base font-medium bg-blue-600 text-white rounded-lg hover:bg-blue-700">
          Get Started
        </button>
        <button class="px-6 py-3 text-base font-medium border border-gray-300 text-gray-700 rounded-lg hover:bg-gray-50">
          View Docs
        </button>
      </div>
    </div>
  </section>

  <!-- Content -->
  <main class="max-w-4xl mx-auto px-6 py-16">
    <article class="prose max-w-none">
      <h2 class="text-2xl font-medium text-gray-900 mb-6 md:text-3xl">
        Why Fira Sans?
      </h2>
      <p class="text-base text-gray-700 mb-4">
        Fira Sans combines the warmth of humanist letterforms with the 
        clarity needed for user interfaces. It's readable at small sizes 
        and striking at display sizes.
      </p>
      <p class="text-base text-gray-700 mb-8">
        Originally commissioned by Mozilla, it's now open source and 
        available for any project.
      </p>
      
      <h3 class="text-lg font-medium tracking-[0.0035em] text-gray-900 mb-4 md:text-xl">
        Key Features
      </h3>
      <ul class="space-y-2 mb-8">
        <li class="text-base text-gray-700">Excellent screen readability</li>
        <li class="text-base text-gray-700">Wide range of weights (300–600)</li>
        <li class="text-base text-gray-700">OpenType features for numerics</li>
        <li class="text-base text-gray-700">Pairs perfectly with Fira Mono</li>
      </ul>
    </article>
  </main>

  <!-- Footer -->
  <footer class="bg-white border-t border-gray-200 px-6 py-8 mt-16">
    <div class="max-w-6xl mx-auto text-center">
      <p class="text-sm text-gray-500">
        © 2026 Fira Sans Typography System. Open source and free to use.
      </p>
    </div>
  </footer>
</body>
</html>
```

## Summary

- Use semantic tokens (`text-h1`, `text-lead`, etc.) for consistency
- Always include responsive variants (`md:`, `lg:`) for headings
- Leverage arbitrary values `[...]` for precise letter-spacing
- Use OpenType features via arbitrary values for data displays
- Start mobile-first (16px body minimum) and scale up
- Combine weight + size carefully (don't over-bold)

This configuration gives you a complete, production-ready Fira Sans typography system in Tailwind.
