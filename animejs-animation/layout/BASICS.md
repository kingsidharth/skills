# Layout Animations

`createLayout(root, params?)` auto-animates DOM layout changes — reordering, flex/grid shifts, and element enter/exit.

## Setup

```tsx
import { createLayout } from 'animejs';
import { useEffect, useRef } from 'react';
import { flushSync } from 'react-dom';

function AnimatedList({ items }: { items: string[] }) {
  const root    = useRef<HTMLUListElement>(null);
  const scope   = useRef<ReturnType<typeof createScope> | null>(null);
  const layout  = useRef<ReturnType<typeof createLayout> | null>(null);

  useEffect(() => {
    scope.current = createScope({ root }).add(() => {
      layout.current = createLayout(root, {
        duration: 400,
        ease: 'outExpo',
      });
    });
    return () => scope.current?.revert();
  }, []);

  const reorder = (newItems: string[]) => {
    layout.current?.update(() => {
      flushSync(() => setItems(newItems)); // synchronous state update required
    });
  };

  return <ul ref={root}>{items.map(i => <li key={i}>{i}</li>)}</ul>;
}
```

> **Always use `flushSync`** inside `layout.update()` when mutating React state — anime.js needs the DOM to update synchronously to capture before/after positions.

## Controls

See [CONTROLS.md](CONTROLS.md) for `layout-id`, enter/exit states, and `record()`.
