# SVG Morphing

`svg.morphTo(target, precision?)` morphs a `<path>`, `<polygon>`, or `<polyline>` between shapes.

## Usage

```tsx
import { animate, createScope, svg } from 'animejs';

function MorphShape() {
  const root = useRef<SVGSVGElement>(null);
  const scope = useRef<ReturnType<typeof createScope> | null>(null);

  useEffect(() => {
    scope.current = createScope({ root }).add(self => {
      self.add('morph', () =>
        animate('#shape', {
          d: svg.morphTo('#target-shape'),
          duration: 800,
          ease: 'inOutExpo',
        })
      );
    });
    return () => scope.current?.revert();
  }, []);

  return (
    <svg ref={root}>
      <path id="shape" d="M10,10 L90,10 L90,90 L10,90 Z" fill="#ff6b6b" />
      {/* target shape can be hidden */}
      <path id="target-shape" d="M50,10 L90,90 L10,90 Z" style={{ display: 'none' }} />
      <button onClick={() => scope.current?.methods.morph()}>Morph</button>
    </svg>
  );
}
```

## Precision

```tsx
svg.morphTo('#target', 2)   // low precision (fewer points, faster)
svg.morphTo('#target', 10)  // high precision (more points, smoother)
// default: 4
```

## Rules

- Both shapes must be the same SVG element type (`<path>` → `<path>`)
- Target element can be hidden (`display: none` or outside viewport)
- Morphing complex paths with very different point counts: increase precision
