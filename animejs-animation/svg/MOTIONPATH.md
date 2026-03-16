# Motion Path

`svg.createMotionPath(pathElement)` returns an object that spreads `x`, `y`, and `angle` — animates any element along an SVG path.

## Usage

```tsx
import { animate, createScope, svg } from 'animejs';

function FollowPath() {
  const root = useRef<HTMLDivElement>(null);
  const scope = useRef<ReturnType<typeof createScope> | null>(null);

  useEffect(() => {
    scope.current = createScope({ root }).add(() => {
      const motionPath = svg.createMotionPath('#track');

      animate('.follower', {
        ...motionPath,      // spreads x, y, angle
        duration: 2000,
        ease: 'linear',
        loop: true,
      });
    });
    return () => scope.current?.revert();
  }, []);

  return (
    <div ref={root} style={{ position: 'relative' }}>
      <svg width="300" height="200">
        <path id="track" d="M 50 150 C 100 50, 200 50, 250 150" fill="none" stroke="#ddd" />
      </svg>
      <div className="follower" style={{ position: 'absolute', width: 20, height: 20, background: '#ff6b6b', borderRadius: '50%' }} />
    </div>
  );
}
```

## Rotation along path

The `angle` property from `createMotionPath` auto-rotates the element to face the direction of travel. Remove it to keep the element upright:

```tsx
const { x, y } = svg.createMotionPath('#track'); // destructure to skip angle

animate('.follower', { x, y, duration: 2000 });
```

## Notes

- The path must be an SVG `<path>` element
- The animated element can be HTML or SVG
- Position the element with `position: absolute` relative to a common parent
- Use `ease: 'linear'` for smooth path following; other eases cause speed variation
