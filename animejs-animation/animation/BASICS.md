# Animation Basics

`animate(targets, parameters)` → `JSAnimation`

## Targets

Prefer CSS class selectors — `createScope({ root })` automatically scopes them to the component subtree.

```tsx
animate('.card', { x: 100 });                         // CSS selector (preferred)
animate(ref.current, { opacity: 0 });                 // DOM ref
animate([ref1.current, ref2.current], { scale: 0 });  // array of elements
animate({ value: 0 }, { value: 100, onUpdate: ... }); // plain JS object
```

## CSS properties

Any camelCase CSS property:

```tsx
animate('.el', {
  opacity: 0.5,
  backgroundColor: '#ff6b6b',
  width: '200px',
  borderRadius: '50%',
  fontSize: '2rem',
});
```

## CSS transforms

Use shorthand names — no `transform:` wrapper needed:

```tsx
animate('.el', {
  x: 100,           // translateX — px by default
  y: '-50%',        // translateY with explicit unit
  rotate: '1turn',  // full rotation
  scale: 1.5,
  scaleX: 2,
  skewY: '15deg',
  translateZ: 50,
});
```

## CSS variables

```tsx
animate('.el', {
  '--my-color': '#ff6b6b',
  '--progress': 1,
  '--angle': '360deg',
});
```

## SVG attributes

```tsx
animate('circle', { r: [0, 50], cx: 100, fill: '#ff0000' });
animate('rect',   { width: 200, rx: 8 });
animate('line',   { x2: 300, strokeDashoffset: 0 });
```

For morphing, draw-on, and motion paths → [svg/BASICS.md](../svg/BASICS.md)
