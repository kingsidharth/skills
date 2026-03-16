# Timeline

`createTimeline(params?)` sequences animations with precise timing control.

## Basic usage

```tsx
import { createTimeline, stagger } from 'animejs';

useEffect(() => {
  scope.current = createScope({ root }).add(self => {
    const tl = createTimeline({ loop: true, duration: 2000 });

    tl.add('.box1', { x: 100, duration: 500 })
      .add('.box2', { y: 50,  duration: 500 })   // starts after box1
      .add('.box3', { scale: 2, duration: 400 }); // starts after box2
  });

  return () => scope.current?.revert();
}, []);
```

## Time position (3rd argument to `.add()`)

Controls when the animation starts relative to the timeline:

```tsx
tl.add('.a', { x: 100 }, 0)         // absolute: start at 0ms
  .add('.b', { y: 50  }, 200)        // absolute: start at 200ms
  .add('.c', { scale: 2 }, '+=100')  // 100ms after previous ends
  .add('.d', { opacity: 0 }, '-=50') // 50ms before previous ends (overlap)
  .add('.e', { rotate: 90 }, '<')    // same start as previous
  .add('.f', { x: -50 }, '<+=200'); // 200ms after previous started
```

## Stagger in timeline

```tsx
tl.add('.card', {
  y: [20, 0],
  opacity: [0, 1],
  delay: stagger(80),
  duration: 400,
});
```

## Defaults

Set shared parameters for all timeline children:

```tsx
const tl = createTimeline({
  defaults: {
    duration: 400,
    ease: 'outExpo',
  },
  loop: true,
  alternate: true,
});

tl.add('.a', { x: 100 })     // inherits duration: 400, ease: 'outExpo'
  .add('.b', { y: 50, duration: 600 }); // overrides duration only
```

## sync() — add existing animations

```tsx
const fadeIn  = animate('.label', { opacity: [0, 1], autoplay: false });
const slideUp = animate('.card',  { y: [30, 0],      autoplay: false });

const tl = createTimeline();
tl.sync(fadeIn)
  .sync(slideUp, '+=100');
```

## Timeline playback

Timeline has the same methods as `animate()`:

```tsx
tl.play() / tl.pause() / tl.restart() / tl.seek(ms) / tl.revert()
await tl.then()
```

Register as scope methods for React:

```tsx
self.add('playIntro', () => tl.restart());
// → scope.current?.methods.playIntro()
```
