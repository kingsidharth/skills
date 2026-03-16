# Easing & Modifier

## Built-in named eases

```tsx
ease: 'linear'
ease: 'outExpo'
ease: 'inOutCirc'
ease: 'outBounce'
ease: 'inElastic'

// Parametric — pass exponent or strength
ease: 'out(3)'        // cubic out
ease: 'inOut(4)'      // quartic in-out
ease: 'outElastic(1, 0.5)'
```

Full list: `linear` · `in` · `out` · `inOut` · `inBack` · `outBack` · `inOutBack` · `inElastic` · `outElastic` · `inOutElastic` · `inBounce` · `outBounce` · `inOutBounce` · `inCirc` · `outCirc` · `inOutCirc` · `inExpo` · `outExpo` · `inOutExpo` · `inQuad`…`inOutQuint` · `inSine` · `outSine` · `inOutSine`

## Spring (physics — ignores `duration`)

```tsx
import { spring } from 'animejs';

ease: spring()
ease: spring({ mass: 1, stiffness: 100, damping: 10, velocity: 0 })
ease: spring({ bounce: 0.4 })  // shorthand; bounce 0–1
```

Use spring for interactive, gesture-driven, or natural-feeling motion.

## Cubic Bézier

```tsx
ease: 'cubicBezier(.17, .67, .83, .67)'
```

## Steps

```tsx
ease: 'steps(5)'       // 5 discrete steps
ease: 'steps(10, end)'
```

## Per-property easing

Each tween can have its own ease:

```tsx
animate('.el', {
  x: { to: 200, ease: 'outExpo' },
  opacity: { to: 0, ease: 'linear' },
  ease: 'outQuad', // fallback for properties without their own
});
```

## playbackEase

Applies an ease across the entire animation's timeline (not per tween):

```tsx
animate('.el', {
  y: [0, -100, 0],
  duration: 2000,
  playbackEase: 'inOutSine', // the overall progress curve
});
```

## Modifier

Post-processes the animated value on every frame. Apply per-property.

```tsx
import { utils } from 'animejs';

// Snap to integers
animate('.el', {
  x: 100,
  modifier: utils.round(0),
});

// Clamp range
animate('.el', {
  opacity: { to: 2, modifier: utils.clamp(0, 1) },
});

// Wrap (loop in range)
animate('.el', {
  rotate: 720,
  modifier: utils.wrap(0, 360),
});

// Custom function
animate('.el', {
  x: 100,
  modifier: v => Math.sin(v) * 50,
});
```
