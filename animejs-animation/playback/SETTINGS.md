# Playback Settings

Set on `animate()` or `createTimeline()`:

```tsx
animate('.el', {
  x: 100,
  duration: 600,       // ms (default: 1000)
  delay: 200,          // ms before start
  loop: true,          // infinite loop
  loop: 3,             // loop 3 times
  loopDelay: 500,      // ms between loops
  alternate: true,     // reverse direction each loop
  reversed: true,      // play in reverse from start
  autoplay: false,     // don't start automatically
  playbackRate: 2,     // 2× speed
  playbackEase: 'inOutSine', // ease across entire animation
});
```

## Staggered delay

```tsx
import { stagger } from 'animejs';

animate('.item', {
  y: [20, 0],
  delay: stagger(80),  // 80ms between each element
  duration: 500,
});
```

## Function-based duration / delay

```tsx
animate('.item', {
  x: 100,
  duration: (el, i) => 400 + i * 100,
  delay:    (el, i) => i * 60,
});
```

For callbacks → [CALLBACKS.md](CALLBACKS.md)
For control methods → [METHODS.md](METHODS.md)
