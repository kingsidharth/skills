---
name: animejs-react
description: Animate React components with anime.js v4. Use when animating UI elements, creating motion/transitions, SVG animations, layout animations, or sequenced timelines in React. Covers the createScope pattern, animate(), timelines, SVG utilities, and the Layout API.
---

# anime.js + React

> Assumes React hooks, anime.js v4, TypeScript.

## Install

```bash
npm install animejs
```

## The React Pattern (always use this)

```tsx
import { animate, createScope } from 'animejs';
import { useEffect, useRef } from 'react';

function MyComponent() {
  const root  = useRef<HTMLDivElement>(null);
  const scope = useRef<ReturnType<typeof createScope> | null>(null);

  useEffect(() => {
    scope.current = createScope({ root }).add(self => {
      animate('.box', { x: 100, duration: 600 });

      // Register methods callable from event handlers
      self.add('slideIn', () => animate('.box', { x: 0, ease: 'outExpo' }));
    });

    return () => scope.current?.revert(); // cleanup on unmount
  }, []);

  return (
    <div ref={root}>
      <div className="box" />
      <button onClick={() => scope.current?.methods.slideIn()}>Slide</button>
    </div>
  );
}
```

- `createScope({ root })` scopes CSS selectors to the component subtree
- `scope.current.revert()` in cleanup stops animations and restores styles
- Methods registered via `self.add()` are available at `scope.current.methods.*`
- Never store animation instances in `useState`; use `useRef`

## Common imports

```tsx
import {
  animate,
  createScope,
  createTimeline,
  createLayout,
  stagger,
  spring,
  svg,
} from 'animejs';
```

## Routing

**Animation**
- Targets, CSS properties, transforms, SVG attributes → [animation/BASICS.md](animation/BASICS.md)
- Value types: from/to, keyframes, relative, function-based, stagger → [animation/TWEENS.md](animation/TWEENS.md)
- Easing, spring physics, modifier → [animation/EASING.md](animation/EASING.md)

**Playback**
- duration, delay, loop, alternate, playbackRate → [playback/SETTINGS.md](playback/SETTINGS.md)
- onBegin, onUpdate, onComplete, .then() → [playback/CALLBACKS.md](playback/CALLBACKS.md)
- play/pause/seek/revert, triggering from React state/events → [playback/METHODS.md](playback/METHODS.md)

**Timeline**
- Sequencing, time positions, defaults, sync() → [timeline/BASICS.md](timeline/BASICS.md)

**SVG**
- SVG attribute animation basics → [svg/BASICS.md](svg/BASICS.md)
- Shape morphing → [svg/MORPH.md](svg/MORPH.md)
- Draw-on / draw-off strokes → [svg/DRAW.md](svg/DRAW.md)
- Animate along a path → [svg/MOTIONPATH.md](svg/MOTIONPATH.md)

**Layout**
- Auto-animate DOM layout changes (flex/grid reorder, enter/exit) → [layout/BASICS.md](layout/BASICS.md)
- layout-id, enterFrom/leaveTo states, record() → [layout/CONTROLS.md](layout/CONTROLS.md)
