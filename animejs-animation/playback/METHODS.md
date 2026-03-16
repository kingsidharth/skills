# Animation Methods & Triggers

## Control methods

```tsx
const anim = animate('.el', { x: 100, autoplay: false });

anim.play();          // start / resume
anim.pause();         // pause
anim.restart();       // reset and play from start
anim.reverse();       // flip direction
anim.seek(300);       // jump to 300ms
anim.cancel();        // stop and remove (no revert)
anim.revert();        // stop and restore original styles
anim.complete();      // jump to end, fire onComplete
anim.reset();         // jump to start, don't play
```

## React: triggering animations

### On mount

```tsx
useEffect(() => {
  scope.current = createScope({ root }).add(() => {
    animate('.el', { y: [20, 0], opacity: [0, 1], duration: 500 });
    // autoplay: true by default — runs immediately on mount
  });
  return () => scope.current?.revert();
}, []); // empty deps = runs once on mount
```

### On state change

```tsx
const [isOpen, setIsOpen] = useState(false);

useEffect(() => {
  if (!scope.current) return;
  scope.current.methods.toggle(isOpen);
}, [isOpen]);

// register in setup effect:
self.add('toggle', (open: boolean) =>
  animate('.drawer', { x: open ? 0 : '-100%', duration: 300 })
);
```

### On user interaction (event handler)

```tsx
// Register method in scope setup:
self.add('pop', () => animate('.btn', { scale: [1, 1.2, 1], duration: 300 }));

// Use in JSX:
<button onClick={() => scope.current?.methods.pop()}>Click</button>
```

### Sequencing with async/await

```tsx
const runSequence = async () => {
  await animate('.step1', { x: 100 }).then();
  await animate('.step2', { opacity: 1 }).then();
  animate('.step3', { scale: 1.2 });
};
```
