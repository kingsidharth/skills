# Layout Controls

## layout-id — track elements across parents

By default, layout tracks elements by DOM position. Add `data-layout-id` to track by identity when elements move between containers:

```tsx
<div className="col-a">
  <div data-layout-id="card-42" className="card">Card 42</div>
</div>
<div className="col-b">
  {/* Moving card-42 here animates it from col-a to col-b */}
</div>
```

## Enter / exit states

Animate new elements entering and removed elements leaving:

```tsx
createLayout(root, {
  duration: 400,
  ease: 'outExpo',

  enterFrom: {
    opacity: 0,
    y: 20,
  },

  leaveTo: {
    opacity: 0,
    scale: 0.8,
    y: -10,
  },

  swapAt: 0.5, // opacity crossfade midpoint when element swaps content (0–1)
});
```

## record() — animate outside of update()

Use `record()` when the DOM change happens outside your control (e.g. browser-driven layout, scroll, resize):

```tsx
layout.current?.record(() => {
  // DOM changes here are captured before/after for animation
  element.classList.toggle('expanded');
});
```

## children selector

Scope which elements are tracked (default: direct children):

```tsx
createLayout(root, {
  children: '.card',  // only animate .card elements
  duration: 300,
});
```

## Gotchas

| Issue | Fix |
|---|---|
| Root element position animates unexpectedly | Wrap root in a parent, animate the parent |
| Transform shorthands (`x`, `y`) don't work in `enterFrom`/`leaveTo` | Use full `transform: 'translateY(20px)'` string |
| Unexpected fade on swap | Adjust `swapAt` value or restrict `children` selector |
| Text reflows during animation | Add `white-space: nowrap` to text elements |
| Inline elements animate incorrectly | Wrap text in a `<span>` or `<div>` |
| SVG elements | Not supported — use `animate()` directly |
