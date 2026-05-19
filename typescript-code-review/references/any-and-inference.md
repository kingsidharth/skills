# `any` and inference

**When this applies:** every TypeScript file. The presence of `any` in committed code is a defect; the absence of inference is wasted DX.

---

## Rule 1: `any` is banned

`any` disables type checking for the value it's attached to *and propagates through every operation involving it*. A single `any` in a hot data path defeats type safety for thousands of lines downstream.

Configuration:

```jsonc
{
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-unsafe-argument": "error",
    "@typescript-eslint/no-unsafe-assignment": "error",
    "@typescript-eslint/no-unsafe-call": "error",
    "@typescript-eslint/no-unsafe-member-access": "error",
    "@typescript-eslint/no-unsafe-return": "error"
  }
}
```

The five `no-unsafe-*` rules catch implicit `any` propagation that the explicit rule misses.

---

## Replacements for `any`

### For "I don't know the shape" → `unknown`

```ts
// 🚨 disables type checking
function parse(json: string): any { return JSON.parse(json) }
const result = parse('...')
result.anything.you.want()  // compiles, blows up at runtime

// ✅ forces narrowing
function parse(json: string): unknown { return JSON.parse(json) }
const result = parse('...')
result.foo  // ❌ compile error — must narrow first

if (typeof result === 'object' && result !== null && 'foo' in result) {
  result.foo  // narrowed to unknown still — go further
}
```

For real boundaries (HTTP, localStorage, postMessage, JSON), parse with **Zod** (or Valibot/ArkType):

```ts
const UserSchema = z.object({ id: z.string(), email: z.string().email() })
type User = z.infer<typeof UserSchema>

const user: User = UserSchema.parse(await response.json())
```

You get a typed value *and* runtime validation. One source of truth.

### For "this function works on many types" → generics

```ts
// 🚨 lazy any
function first(arr: any[]): any { return arr[0] }
const x = first([1, 2, 3])  // x: any 😞

// ✅ generic
function first<T>(arr: readonly T[]): T | undefined { return arr[0] }
const x = first([1, 2, 3])  // x: number | undefined ✅
```

### For "third-party untyped" → `unknown` + a typed adapter

If a library has no types, write a thin typed wrapper at the boundary:

```ts
declare const untypedThing: unknown

function getThingTitle(thing: unknown): string {
  if (typeof thing === 'object' && thing !== null && 'title' in thing && typeof thing.title === 'string') {
    return thing.title
  }
  return ''
}
```

Or write a `.d.ts` for the library. Or — preferably — use a typed alternative.

### For React event handlers — never `any`

```ts
// 🚨
const onChange = (e: any) => setValue(e.target.value)

// ✅ infer from element
const onChange = (e: React.ChangeEvent<HTMLInputElement>) => setValue(e.target.value)

// ✅ even better: rely on inference at the JSX site
<input onChange={(e) => setValue(e.target.value)} />  // e is fully typed
```

### For "I'm casting to make TS shut up" — that's the bug

```ts
// 🚨 lying with `as`
const user = data as User

// ✅ validate
const user = UserSchema.parse(data)

// ✅ or narrow
if (isUser(data)) { /* data is User */ }

function isUser(x: unknown): x is User {
  return typeof x === 'object' && x !== null && 'id' in x && /* ... */
}
```

`as` is a one-way write that bypasses the type checker. Treat every `as` (other than `as const`) as a code smell.

---

## Rule 2: Don't fight inference

TypeScript infers many types accurately. Don't paste annotations everywhere — annotate at boundaries (function parameters, exported values, complex constants), and let inference handle the rest.

```ts
// 🚨 redundant annotation
const items: string[] = ['a', 'b', 'c']
const count: number = items.length

// ✅ let inference work
const items = ['a', 'b', 'c']  // string[]
const count = items.length     // number
```

For function return types: it's a judgment call.

- **Library exports / public API** — annotate the return type explicitly. Catches accidental shape changes.
- **Internal helpers** — let it infer. Less noise, refactors are easier.

---

## Rule 3: `satisfies` over annotation when you want both narrow inference and shape checking

```ts
// 🚨 widens the value's type
const config: Record<string, string | number> = {
  apiUrl: 'https://...',
  timeout: 30,
}
config.apiUrl.toUpperCase()  // works
config.timeout.toUpperCase() // 💥 'string | number' — annotation widened apiUrl to string|number

// ✅ shape-check without widening
const config = {
  apiUrl: 'https://...',
  timeout: 30,
} satisfies Record<string, string | number>
config.apiUrl.toUpperCase()  // works — apiUrl is still narrowed to string
config.timeout.toFixed(2)    // works — timeout is still number
```

`satisfies` validates "this matches the constraint" without losing the specific type.

---

## Rule 4: Specific React inference patterns

### Component props

```tsx
// 🚨 redeclaring native attributes
type ButtonProps = {
  onClick?: () => void
  className?: string
  disabled?: boolean
  // ...
}

// ✅ inherit from the underlying element
type ButtonProps = React.ComponentProps<'button'> & {
  variant?: 'primary' | 'secondary'
}
```

When wrapping a component:

```tsx
type CardProps = React.ComponentProps<typeof BaseCard> & { highlighted?: boolean }
```

### useState with non-trivial defaults

```ts
// 🚨 widening to ANY
const [user, setUser] = useState(null)  // user: null

// ✅ explicit type parameter
const [user, setUser] = useState<User | null>(null)
```

### `useReducer` with discriminated union

```ts
type State =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: Error }

type Action =
  | { type: 'fetch' }
  | { type: 'success'; data: User }
  | { type: 'error'; error: Error }

function reducer(state: State, action: Action): State { /* ... */ }
```

TypeScript narrows `action.type` and `state.status` automatically. You write less, get more safety.

### Element refs

```tsx
// 🚨
const inputRef = useRef<any>(null)

// ✅
const inputRef = useRef<HTMLInputElement>(null)

// ✅ for refs to a specific component
const dialogRef = useRef<React.ElementRef<typeof Dialog>>(null)
```

---

## Rule 5: Avoid type assertions when narrowing works

```ts
// 🚨 cast
const value = formData.get('email') as string

// ✅ narrow
const raw = formData.get('email')
if (typeof raw !== 'string') throw new Error('email missing')
const value = raw  // typed string from here
```

For form input specifically, validate with Zod up front:

```ts
const FormSchema = z.object({ email: z.string().email() })
const data = FormSchema.parse(Object.fromEntries(formData))
data.email  // typed and validated
```

---

## Rule 6: `as const` is allowed and useful

`as const` widens to literal types and freezes:

```ts
const ROLES = ['admin', 'editor', 'viewer'] as const
type Role = typeof ROLES[number]  // 'admin' | 'editor' | 'viewer'
```

This is good. Don't ban this when banning `as`.

---

## When `any` (or its near-cousin) might actually be unavoidable

Vanishingly rare cases. When it happens:

- Localize it to a single line.
- Add an inline justification comment.
- Use `unknown` if at all possible, falling back to `any` only after exhausting alternatives.

```ts
// eslint-disable-next-line @typescript-eslint/no-explicit-any -- third-party callback signature is genuinely (...args: any[]) => any
const callback: (...args: any[]) => any = lib.subscribe(...)
```

If you find yourself writing more than two of these comments in a feature, the design has a problem the comments are papering over.

---

## References

- Matt Pocock — *No, any is not okay* and Total TypeScript articles
- TypeScript handbook — *Narrowing*, *Generics*, *Type Assertions*
- Zod docs — Standard Schema integration
