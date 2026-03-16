# Callbacks & Promise

## Lifecycle callbacks

```tsx
animate('.el', {
  x: 100,
  onBegin:    (anim) => console.log('started'),
  onUpdate:   (anim) => console.log(anim.progress), // 0→1
  onLoop:     (anim) => console.log('looped'),
  onPause:    (anim) => console.log('paused'),
  onComplete: (anim) => console.log('done'),
});
```

## Promise (.then)

```tsx
await animate('.el', { x: 100, duration: 600 }).then();

// chain animations
animate('.el', { x: 100 })
  .then(() => animate('.el', { opacity: 0 }))
  .then(() => console.log('all done'));
```

## React: callbacks in scope

Register side-effects as scope methods so they have stable references:

```tsx
useEffect(() => {
  scope.current = createScope({ root }).add(self => {
    const anim = animate('.el', {
      x: 100,
      autoplay: false,
      onComplete: () => self.methods.handleDone(),
    });

    self.add('handleDone', () => {
      // safe to update external state here via refs
      isDoneRef.current = true;
    });

    self.add('play', () => anim.play());
  });

  return () => scope.current?.revert();
}, []);
```

> Avoid closing over React state inside `useEffect` callbacks — use refs instead, or trigger state updates via `flushSync`.
