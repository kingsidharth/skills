# Component inventory control

**When this applies:** installing a UI / icon library, reviewing a `components/` folder, or auditing the file count of a frontend app.

The two failure modes:

1. A library silently adds thousands of components (icons, UI kits) into the module graph or component tree.
2. Wrapper components proliferate — `HomepageButtonCTAAnimated` wrapping `Button` for no reason — until the codebase has 800 components doing the work of 80.

Both make the app slower, harder to navigate, and harder for AI agents to reason about.

---

## Rule 1: Never barrel-import an icon or UI library

### Incorrect

```ts
// 🚨 pulls every icon symbol into module graph
import * as Icons from 'lucide-react'

<Icons.User />
<Icons.Settings />
```

```ts
// 🚨 same problem with phosphor
import { Icon, IconProps } from 'phosphor-react'
import * as PhosphorIcons from '@phosphor-icons/react'
```

```ts
// 🚨 dynamic icon name from a string
const Icon = Icons[iconName]
```

The third form is the worst because tree-shaking can't statically determine which icons are used; bundlers are forced to keep all of them.

### Correct

```ts
// ✅ named, direct import — only `User` and `Settings` ship
import { User, Settings } from 'lucide-react'
```

For dynamic icons, build a static, exhaustive map at module top-level:

```ts
import { User, Settings, Bell } from 'lucide-react'

const ICONS = { user: User, settings: Settings, bell: Bell } as const
type IconName = keyof typeof ICONS

export function Icon({ name, ...props }: { name: IconName } & ComponentProps<typeof User>) {
  const Component = ICONS[name]
  return <Component {...props} />
}
```

The map is the *allow-list*. Everything else stays out of the bundle.

### Configuration to enforce

Add to ESLint:

```jsonc
{
  "rules": {
    "no-restricted-imports": ["error", {
      "patterns": [
        { "group": ["lucide-react/*", "@phosphor-icons/react/*"], "message": "Import named icons from the package root, not subpaths." }
      ]
    }]
  }
}
```

For Next.js, configure `experimental.optimizePackageImports`:

```js
const nextConfig = {
  experimental: {
    optimizePackageImports: ['lucide-react', '@radix-ui/react-icons', 'date-fns', 'lodash-es'],
  },
}
```

This rewrites barrel imports to per-symbol imports at build time.

---

## Rule 2: Wrappers must justify their existence

A new wrapper component is only justified when it satisfies *at least one* of:

1. **Reuse** — used in ≥ 2 places with identical configuration.
2. **State / behaviour** — encapsulates state, refs, effects, or events that the caller would otherwise duplicate.
3. **Composition boundary** — defines a clear API for accepting children / slots that the primitive doesn't expose.
4. **Domain language** — names something the team genuinely talks about (`InvoiceStatusPill`, not `BluePillSmall`).

If none apply, **inline the primitive**.

### Anti-pattern: cosmetic wrappers

```tsx
// 🚨 HomepageButtonCTAAnimated — used once, sets two props
function HomepageButtonCTAAnimated({ children, onClick }: Props) {
  return (
    <Button
      variant="primary"
      size="lg"
      className="animate-pulse"
      onClick={onClick}
    >
      {children}
    </Button>
  )
}

// usage
<HomepageButtonCTAAnimated onClick={handleStart}>
  Get started
</HomepageButtonCTAAnimated>
```

This is three problems:

- **One-place reuse.** The wrapper has one caller. The "abstraction" obscures rather than reveals.
- **Combinatorial naming.** `HomepageButtonCTAAnimated` will spawn `LoginButtonCTAAnimated`, `PricingButtonCTASmall`, etc. The component count grows multiplicatively with feature × variant × placement.
- **Drift.** The next change to `Button` props won't propagate.

### Correct

Inline at the call site:

```tsx
<Button
  variant="primary"
  size="lg"
  className="animate-pulse"
  onClick={handleStart}
>
  Get started
</Button>
```

If the same three-prop combination appears across three or more files, *then* extract — but extract by **variant**, not by **placement**:

```tsx
// ✅ named for what it IS, not where it's used
function CtaButton(props: ComponentProps<typeof Button>) {
  return <Button variant="primary" size="lg" className="animate-pulse" {...props} />
}
```

Note the lift: `CtaButton` accepts all `Button` props (`ComponentProps<typeof Button>`) so it stays a thin override. The forbidden version was a closed wrapper with its own narrow prop type.

### When AI is generating code

AI assistants reflexively wrap. Push back. Before accepting a generated wrapper component, check:

- Does this wrap a primitive that already takes the props being set?
- Is the wrapper called in only one place?
- Is the name a placement description (`HomepageX`, `DashboardY`) rather than a thing-name?

Any "yes" → reject the wrapper, inline the primitive.

---

## Rule 3: Component file naming and location

| Symptom | Indicates |
|---|---|
| `components/HomepageHero.tsx`, `components/PricingHero.tsx`, `components/AboutHero.tsx` | Wrappers by placement. Move to feature folders or a single `<Hero>` primitive with variants. |
| `components/Button.tsx` and `components/buttons/PrimaryButton.tsx` | Two layers of indirection. Variants belong on the primitive. |
| 30+ files in flat `components/` | Folder needs splitting by domain or component is over-decomposed. |
| `useFooBar.ts` containing only `return useQuery(...)` | Hook adds no value. Inline. |

---

## Rule 4: Audit signals

When reviewing a codebase, run:

```sh
# Component count
find src -name '*.tsx' -not -path '*/node_modules/*' | wc -l

# Components named by placement (likely wrappers)
grep -rEn 'function (Homepage|Dashboard|Login|Signup|Pricing|About)[A-Z][a-zA-Z]+' src/

# Suspiciously specific names suggesting one-place wrappers
grep -rEn 'function [A-Z][a-zA-Z]+(Button|Card|Container|Wrapper|Section)Custom' src/

# Barrel icon imports
grep -rEn "import \* as .* from ['\"]@?phosphor|lucide|radix-ui" src/
```

Each match is a finding to investigate.

---

## References

- Vercel `agent-skills/react-best-practices` — `bundle-barrel-imports`, `bundle-analyzable-paths`
- TkDodo — *Component Composition is great btw*
- Next.js — `experimental.optimizePackageImports`
