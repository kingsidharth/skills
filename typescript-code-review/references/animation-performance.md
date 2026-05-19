# Animation performance

**When this applies:** any time something on screen moves, fades, scales, slides, or transitions. Whether it's a hover effect or a route transition.

The browser's compositor can move pixels at 60–120 fps for free, *if* you only animate the right properties. The moment an animation triggers layout or paint, you've left the fast path.

---

## Rule 1: Only animate compositor-only properties

| Cheap (compositor) | Expensive (paint / layout) |
|---|---|
| `transform` (translate, scale, rotate, skew) | `top`, `left`, `right`, `bottom` |
| `opacity` | `width`, `height` |
| `filter` | `margin`, `padding` |
| `clip-path` (with caveats) | `border-width`, `font-size` |
| | `box-shadow` (paint-heavy) |

If you're using anything in the right column inside a `transition` or `@keyframes`, you're animating on the main thread. On a busy page or a low-end device, that's where jank lives.

```css
/* 🚨 layout on every frame */
.menu {
  transition: left 200ms;
}
.menu.open { left: 0; }

/* ✅ compositor */
.menu {
  transform: translateX(-100%);
  transition: transform 200ms;
}
.menu.open { transform: translateX(0); }
```

---

## Rule 2: Promote heavy animations to their own layer

Browsers can lift specific elements onto the GPU when you signal it. Use sparingly — every promoted layer costs memory.

```css
.card {
  will-change: transform, opacity;
}
```

`will-change` should be set *before* animation starts and removed when done — otherwise every promoted element costs memory permanently.

For a hover effect:

```css
.card { transition: transform 150ms; }
.card:hover {
  will-change: transform;
  transform: translateY(-2px);
}
```

For longer animations: toggle `will-change` via JS at start/end. For most cases, just don't set `will-change` and trust the browser — it's getting smarter.

---

## Rule 3: Avoid layout thrashing

A render loop that *reads* a DOM measurement (`getBoundingClientRect`, `offsetWidth`) and then *writes* (`style`, `class`) forces layout recalculation between every read/write pair.

```ts
// 🚨 measure-write-measure-write — layout thrashes
items.forEach((el, i) => {
  const h = el.offsetHeight        // read
  el.style.top = `${i * h}px`     // write — invalidates layout
  // next iteration's read has to recompute
})

// ✅ batch reads, then batch writes
const heights = items.map(el => el.offsetHeight)  // all reads
items.forEach((el, i) => {
  el.style.top = `${i * heights[i]}px`            // all writes
})
```

The `requestAnimationFrame` callback is the right place for batched DOM reads + writes if you're doing imperative animation.

---

## Rule 4: `requestAnimationFrame`, not `setInterval`

For any animation driven from JS, use `requestAnimationFrame`. It syncs with the display refresh rate and pauses when the tab is hidden.

```ts
function animate(start: number) {
  const tick = (now: number) => {
    const elapsed = now - start
    el.style.transform = `translateX(${elapsed / 10}px)`
    if (elapsed < 1000) requestAnimationFrame(tick)
  }
  requestAnimationFrame(tick)
}
```

Don't drive animation with `setInterval(..., 16)` — it's not synced to display, runs in background tabs, and drifts.

---

## Rule 5: `prefers-reduced-motion`

Always respect the user's preference. People with vestibular disorders, ADHD, or migraine may have this on.

```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

For decorative animation (page transitions, cosmetic motion), this is enough — disabling them does no harm.

For *meaningful* animation (loading indicators, focus indicators, drag/drop feedback), don't fully disable — replace with a non-motion equivalent (cross-fade, color change, instant snap):

```ts
const reduceMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches

const config = reduceMotion
  ? { duration: 0 }
  : { duration: 200, easing: 'ease-out' }
```

Animation libraries (Framer Motion / Motion, React Spring) have built-in reduced-motion support — turn it on.

---

## Rule 6: View Transitions for cross-route / cross-state changes

Browser-native View Transitions API (Chromium, Safari 18) lets the browser snapshot the current state, snapshot the next state, and animate between them — no manual orchestration.

For React Router 7+ / Next.js 14+:

```tsx
// React Router
<Link to="/posts/123" viewTransition>...</Link>

// Next.js
<Link href="/posts/123" prefetch>...</Link>
// + opt in to ViewTransitions in app code
```

```css
/* shared element — same name on source and destination */
.post-card { view-transition-name: post-image; }

::view-transition-old(post-image),
::view-transition-new(post-image) {
  animation-duration: 200ms;
}
```

This is the right tool for "nice transitions between routes" without a JS animation library.

---

## Rule 7: Animation library choice

| Library | When to use |
|---|---|
| **CSS transitions / animations** | Default. Hover states, simple enter/exit, anything keyframable in CSS. |
| **View Transitions API** | Route transitions, layout transitions, shared elements across navigations. |
| **Motion / Framer Motion** | Layout animations (`layout` prop), gestures, complex orchestration, exit animations. |
| **React Spring** | Physics-based motion (drag, fling), interruptible animations. |
| **anime.js / GSAP** | SVG-heavy work, timelines, fine-grained sequencing. |

Don't reach for Framer Motion to fade an opacity. Use CSS. Don't reach for GSAP to animate a height. Use a different layout (transform with anchored origin).

---

## Rule 8: Avoid animating expensive shadows and filters

```css
/* 🚨 box-shadow animation triggers paint on every frame, expensive */
.card {
  box-shadow: 0 2px 4px rgba(0,0,0,.1);
  transition: box-shadow 200ms;
}
.card:hover {
  box-shadow: 0 8px 24px rgba(0,0,0,.2);
}

/* ✅ animate opacity of a pseudo-element with the bigger shadow */
.card { position: relative; box-shadow: 0 2px 4px rgba(0,0,0,.1); }
.card::after {
  content: '';
  position: absolute; inset: 0;
  box-shadow: 0 8px 24px rgba(0,0,0,.2);
  opacity: 0;
  transition: opacity 200ms;
  pointer-events: none;
  border-radius: inherit;
}
.card:hover::after { opacity: 1; }
```

Same visual, compositor-only animation, no paint.

---

## Rule 9: Don't animate during scroll without coordination

Scrolling is already on the compositor. JS-driven scroll-linked animation can stall it.

- For parallax / scroll-tied effects: CSS `scroll-timeline` (browser-native, off-main-thread).
- Or `IntersectionObserver` to *trigger* CSS classes — not to drive a value continuously.
- If you must use scroll listeners, mark the listener `passive: true`.

```ts
window.addEventListener('scroll', onScroll, { passive: true })
```

Without `passive`, the browser must wait for your handler before scrolling, which can feel like a frozen page.

---

## Rule 10: Animation budget per frame

At 60 fps, you have **16.6 ms** per frame. Browser layout/paint/composite eats some of that. Your JS animation work should fit in **~5 ms** to leave headroom.

- A keyframe-driven CSS animation is ~free.
- A React state-driven animation costs render + reconciliation per frame; risk of dropping frames if the tree is large.
- A 60 fps animation on a 1000-row table is going to drop frames if every row re-renders. Animate the *container*, not the rows.

---

## Quick checklist

```
- [ ] Animation properties are only transform / opacity / filter
- [ ] No top/left/width/height in transitions
- [ ] prefers-reduced-motion respected
- [ ] Long animations don't pin will-change permanently
- [ ] Scroll listeners are passive
- [ ] Per-frame work is in requestAnimationFrame, not setInterval
- [ ] View Transitions used for route/layout changes where supported
```

---

## References

- MDN — *CSS containment*, *will-change*, *requestAnimationFrame*
- Web.dev — *Animations Guide*, *INP optimization*
- Rauno Freiberg — *Web Interface Guidelines* (motion section)
- Vercel `agent-skills/react-best-practices` — `client-passive-event-listeners`
