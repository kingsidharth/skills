# SVG Basics

Standard SVG attributes animate the same as CSS properties — use `animate()` directly:

```tsx
animate('circle', { r: [0, 50], cx: 100, fill: '#ff6b6b', duration: 600 });
animate('rect',   { width: 200, height: 100, rx: 8 });
animate('line',   { x2: 300, strokeWidth: 2 });
```

## Stroke dash animation (manual)

```tsx
// set strokeDasharray = path length, then animate strokeDashoffset
animate('path', {
  strokeDashoffset: [anime.setDashoffset, 0],
  duration: 1200,
  ease: 'outExpo',
});
```

For draw-on with full control → [DRAW.md](DRAW.md)

## Utilities

| Utility | Use |
|---|---|
| `svg.morphTo()` | Morph between two shapes → [MORPH.md](MORPH.md) |
| `svg.createDrawable()` | Draw-on / draw-off stroke animation → [DRAW.md](DRAW.md) |
| `svg.createMotionPath()` | Move element along SVG path → [MOTIONPATH.md](MOTIONPATH.md) |
