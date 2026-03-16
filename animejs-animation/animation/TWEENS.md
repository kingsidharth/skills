# Tween Value Types

## Simple to-value

```tsx
animate('.el', { x: 100 });
```

## From → to

```tsx
animate('.el', { x: { from: 0, to: 200 } });
animate('.el', { opacity: { from: 0, to: 1 } });
```

## Keyframes array (per-property)

```tsx
animate('.el', {
  y: [
    { to: -30, ease: 'outExpo', duration: 400 },
    { to: 0,   ease: 'outBounce', duration: 600, delay: 50 },
  ],
  rotate: [
    { to: '-1turn', delay: 0 },
  ],
});
```

## Relative values

```tsx
animate('.el', { x: '+=100' });    // add 100 to current value
animate('.el', { x: '-=50' });     // subtract
animate('.el', { scale: '*=2' });  // multiply
```

## Function-based (different value per element)

```tsx
animate('.item', {
  x: (el, index) => index * 60,
  opacity: (el, index, total) => 1 - index / total,
  delay: (el, index) => index * 80,
});
```

Receives `(element, index, total)`.

## Unit conversion

Anime.js converts units automatically:

```tsx
animate('.el', { width: '50%' });     // from px → %
animate('.el', { x: '10rem' });       // from px → rem
```

## Color values

```tsx
animate('.el', { backgroundColor: '#ff6b6b' });
animate('.el', { color: 'rgb(255, 107, 107)' });
animate('.el', { borderColor: 'hsl(0, 100%, 71%)' });
```

## Stagger (shorthand for function-based delay)

```tsx
import { stagger } from 'animejs';

animate('.item', {
  y: [20, 0],
  opacity: [0, 1],
  delay: stagger(80),                      // 80ms between each
  duration: 500,
});

animate('.dot', {
  scale: [0, 1],
  delay: stagger(50, { from: 'center' }), // outward from center
});

animate('.cell', {
  opacity: [0, 1],
  delay: stagger(30, {
    grid: [5, 4],   // 5 cols × 4 rows
    from: 'center',
  }),
});
```
