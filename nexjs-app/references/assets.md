# Assets — Images, Fonts, `public/`

## `next/image`

Always use `<Image>` over raw `<img>`. Gives size optimization, WebP conversion, lazy-loading, layout-shift prevention.

### Local images — static import (preferred)

```tsx
import Image from 'next/image'
import profile from './profile.png'

export default function Page() {
  return <Image src={profile} alt="Author" />
  // width, height, blurDataURL auto-inferred
}
```

### Local images — from `public/`

Store in `public/`, reference from `/`:

```tsx
<Image src="/profile.png" alt="Author" width={500} height={500} />
```

### Remote images

Must be allowlisted in `next.config.ts`. Be as specific as possible:

```ts
const config: NextConfig = {
  images: {
    remotePatterns: [{
      protocol: 'https',
      hostname: 's3.amazonaws.com',
      pathname: '/my-bucket/**',
    }],
  },
}
```

Supply `width`/`height` (or `fill`) manually since Next can't inspect remote files at build time.

### Dynamic local imports (no static import possible)

Inside a Server Component:

```tsx
async function PostImage({ filename, alt }: { filename: string; alt: string }) {
  const { default: image } = await import(`../content/images/${filename}`)
  return <Image src={image} alt={alt} />
}
```

Path must have a static prefix (`../content/images/`). All files matching the prefix get bundled — only include files in a directory you control.

### `fill`

When you don't know dimensions — lets the image fill its parent:

```tsx
<div style={{ position: 'relative', width: 300, height: 200 }}>
  <Image src="/cover.jpg" alt="" fill style={{ objectFit: 'cover' }} />
</div>
```

### Static export / custom loader

When `output: 'export'` or when you want external optimization (Cloudinary, Imgix), set a custom loader in `next.config.ts`:

```ts
images: { loader: 'custom', loaderFile: './my-loader.ts' }
```

Loader returns a URL given `{ src, width, quality }`.

### Caching

Next sets `Cache-Control: public, max-age=31536000, immutable` on hashed assets (static imports). Configure TTL for remote via `minimumCacheTTL`.

## `next/font`

Self-hosts any font file. No external network requests, no layout shift.

### Google fonts

```tsx
// app/layout.tsx
import { Geist } from 'next/font/google'

const geist = Geist({ subsets: ['latin'] })

export default function RootLayout({ children }: LayoutProps<'/'>) {
  return (
    <html lang="en" className={geist.className}>
      <body>{children}</body>
    </html>
  )
}
```

Prefer variable fonts (no weight needed). Non-variable requires `weight`:

```ts
const roboto = Roboto({ weight: '400', subsets: ['latin'] })
```

### Local fonts

```tsx
import localFont from 'next/font/local'
const myFont = localFont({ src: './my-font.woff2' })
```

Multi-file family:

```ts
const roboto = localFont({
  src: [
    { path: './Roboto-Regular.woff2', weight: '400', style: 'normal' },
    { path: './Roboto-Italic.woff2', weight: '400', style: 'italic' },
    { path: './Roboto-Bold.woff2', weight: '700', style: 'normal' },
  ],
})
```

Fonts are scoped to the component where called. For app-wide fonts, put in root layout.

## `public/` folder

Static assets served from `/`. File `public/avatars/me.png` → URL `/avatars/me.png`.

- Files are not cached strongly by Next (may change). Default header: `Cache-Control: public, max-age=0`.
- Don't put `robots.txt`/`favicon.ico` here — use [metadata file conventions](https://nextjs.org/docs/app/api-reference/file-conventions/metadata) inside `app/` instead (`app/robots.ts`, `app/favicon.ico`, `app/icon.tsx`, `app/apple-icon.tsx`, `app/sitemap.ts`, `app/opengraph-image.tsx`).
- When using `src/`, keep `public/` at the project root.
