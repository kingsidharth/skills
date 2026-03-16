# SVG Draw Animation

`svg.createDrawable(target)` creates a proxy with a `draw` property (`'start end'` as fractions 0–1).

## Draw on

```tsx
import { animate, createScope, svg } from 'animejs';

useEffect(() => {
  scope.current = createScope({ root }).add(() => {
    const drawable = svg.createDrawable('#my-path');

    animate(drawable, {
      draw: ['0 0', '0 1'],  // from nothing → fully drawn
      duration: 1200,
      ease: 'outExpo',
    });
  });
  return () => scope.current?.revert();
}, []);
```

```tsx
<path id="my-path" d="M10 80 C 40 10, 65 10, 95 80" stroke="#ff6b6b" strokeWidth="3" fill="none" />
```

## Draw on, then erase (wipe through)

```tsx
animate(drawable, {
  draw: ['0 0', '0 1', '1 1'],  // draw in, then wipe off from start
  duration: 2000,
  ease: 'linear',
});
```

## Partial draw (visible segment)

`draw: 'start end'` — both values are 0–1 fractions of total path length:

```tsx
animate(drawable, {
  draw: ['0.2 0.5', '0 1'],  // start from 20%-50% window → full path
  duration: 800,
});
```

## React: trigger on interaction

```tsx
self.add('drawLine', () => {
  const drawable = svg.createDrawable('#line');
  animate(drawable, { draw: ['0 0', '0 1'], duration: 600 });
});
```
