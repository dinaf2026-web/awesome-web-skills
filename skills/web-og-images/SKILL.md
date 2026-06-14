---
name: web-og-images
description: Dynamic Open Graph image generation for Next.js — @vercel/og patterns, per-page OG images, template system, and social sharing optimization for Twitter/X, LinkedIn, and Facebook.
origin: community
tags: [og-images, open-graph, social-sharing, vercel-og, metadata, seo]
---

# web-og-images

Dynamic Open Graph image generation for Next.js using `@vercel/og`. Covers per-page OG images, reusable templates, Twitter/X and LinkedIn optimization, font loading at the Edge, and caching strategy.

---

## 1. When to Use Dynamic vs Static OG Images

### Use a static image when:
- The page content never changes (homepage, about, pricing)
- You want zero cold-start latency
- You have a single brand image that works everywhere

Place static images in `public/` and reference them directly in metadata:

```tsx
// app/layout.tsx
export const metadata = {
  openGraph: {
    images: [{ url: '/og-default.png', width: 1200, height: 630 }],
  },
};
```

### Use a dynamic image when:
- Each page has unique content (blog posts, product pages, user profiles)
- You want the title/description/author rendered into the image
- You need per-page social previews that drive click-through

Dynamic images are generated on demand via an Edge Function route and cached by the CDN after the first request.

**Decision matrix:**

| Signal | Static | Dynamic |
|---|---|---|
| Same image across all pages | Yes | No |
| Content changes per URL | No | Yes |
| Build-time known count < 100 | Maybe static gen | Either |
| Personalized or user-specific | No | Yes |

---

## 2. Setup

Install the package:

```bash
npm install @vercel/og
# or
pnpm add @vercel/og
```

`@vercel/og` exports `ImageResponse`, which accepts JSX and renders it to a PNG using [Satori](https://github.com/vercel/satori) + [resvg](https://github.com/RazrFalcon/resvg) at the Edge. No Node.js APIs — CSS-subset only (flexbox, no grid).

**Edge Runtime requirement:** OG routes must declare `export const runtime = 'edge'`. The default Node.js runtime is not supported by `ImageResponse`.

**Supported CSS:**
- Flexbox layout (not Grid)
- `position: absolute`
- `border-radius`, `box-shadow`
- `font-family`, `font-size`, `font-weight`, `color`
- `background-color`, `background-image` (linear/radial gradient)
- `object-fit`, `object-position`
- No `overflow: hidden` on non-flex containers

---

## 3. Basic OG Image Route

```
app/
  og/
    route.tsx       ← handles GET /og
```

```tsx
// app/og/route.tsx
import { ImageResponse } from '@vercel/og';
import type { NextRequest } from 'next/server';

export const runtime = 'edge';

export async function GET(request: NextRequest) {
  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          width: '100%',
          height: '100%',
          alignItems: 'center',
          justifyContent: 'center',
          background: 'linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%)',
        }}
      >
        <h1 style={{ color: '#e94560', fontSize: 72, fontWeight: 700 }}>
          My Site
        </h1>
      </div>
    ),
    {
      width: 1200,
      height: 630,
    }
  );
}
```

Visit `/og` in your browser during development to preview the output immediately.

---

## 4. Dynamic OG from URL Params

Pass page-specific data via search params so a single route serves all pages.

```tsx
// app/og/route.tsx
import { ImageResponse } from '@vercel/og';
import type { NextRequest } from 'next/server';

export const runtime = 'edge';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);

  const title = searchParams.get('title') ?? 'Untitled';
  const description = searchParams.get('description') ?? '';
  const author = searchParams.get('author') ?? '';

  // Clamp lengths to prevent overflow
  const safeTitle = title.slice(0, 70);
  const safeDescription = description.slice(0, 120);

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          flexDirection: 'column',
          width: '100%',
          height: '100%',
          padding: '60px 80px',
          background: '#ffffff',
          justifyContent: 'space-between',
        }}
      >
        {/* Top bar */}
        <div style={{ display: 'flex', alignItems: 'center' }}>
          <span style={{ fontSize: 24, color: '#6366f1', fontWeight: 700 }}>
            mysite.com
          </span>
        </div>

        {/* Main content */}
        <div style={{ display: 'flex', flexDirection: 'column', gap: 16 }}>
          <h1
            style={{
              fontSize: safeTitle.length > 40 ? 52 : 68,
              fontWeight: 800,
              color: '#111827',
              lineHeight: 1.1,
              margin: 0,
            }}
          >
            {safeTitle}
          </h1>
          {safeDescription && (
            <p style={{ fontSize: 28, color: '#6b7280', margin: 0, lineHeight: 1.4 }}>
              {safeDescription}
            </p>
          )}
        </div>

        {/* Bottom bar */}
        {author && (
          <div style={{ display: 'flex', alignItems: 'center', gap: 12 }}>
            <span style={{ fontSize: 22, color: '#374151' }}>by {author}</span>
          </div>
        )}
      </div>
    ),
    { width: 1200, height: 630 }
  );
}
```

Reference from metadata:

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({ params }: { params: { slug: string } }): Promise<Metadata> {
  const post = await getPost(params.slug);
  const ogUrl = new URL('/og', process.env.NEXT_PUBLIC_SITE_URL);
  ogUrl.searchParams.set('title', post.title);
  ogUrl.searchParams.set('description', post.excerpt);
  ogUrl.searchParams.set('author', post.author.name);

  return {
    title: post.title,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [{ url: ogUrl.toString(), width: 1200, height: 630 }],
    },
    twitter: {
      card: 'summary_large_image',
      images: [ogUrl.toString()],
    },
  };
}
```

---

## 5. Blog Post OG Template

A richer template with author avatar, date, category badge, and site logo — all composed with flexbox.

```tsx
// app/og/blog/route.tsx
import { ImageResponse } from '@vercel/og';
import type { NextRequest } from 'next/server';

export const runtime = 'edge';

const SITE_URL = process.env.NEXT_PUBLIC_SITE_URL ?? 'http://localhost:3000';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);

  const title = (searchParams.get('title') ?? 'Untitled').slice(0, 80);
  const author = searchParams.get('author') ?? 'Anonymous';
  const date = searchParams.get('date') ?? '';
  const category = searchParams.get('category') ?? '';
  const avatarUrl = searchParams.get('avatar') ?? '';

  // Fetch logo as ArrayBuffer so it works at the Edge
  const logoData = await fetch(new URL('/logo.png', SITE_URL)).then((r) =>
    r.arrayBuffer()
  );

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          flexDirection: 'column',
          width: '100%',
          height: '100%',
          background: '#0f172a',
          padding: '64px 72px',
          justifyContent: 'space-between',
        }}
      >
        {/* Header: logo + category */}
        <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
          {/* eslint-disable-next-line @next/next/no-img-element */}
          <img src={logoData as unknown as string} width={140} height={40} alt="logo" />
          {category && (
            <div
              style={{
                display: 'flex',
                background: '#6366f1',
                borderRadius: 999,
                padding: '6px 20px',
                fontSize: 18,
                color: '#fff',
                fontWeight: 600,
                textTransform: 'uppercase',
                letterSpacing: 1,
              }}
            >
              {category}
            </div>
          )}
        </div>

        {/* Title */}
        <h1
          style={{
            fontSize: title.length > 50 ? 52 : 68,
            fontWeight: 800,
            color: '#f8fafc',
            lineHeight: 1.15,
            margin: '0',
            maxWidth: '90%',
          }}
        >
          {title}
        </h1>

        {/* Author row */}
        <div style={{ display: 'flex', alignItems: 'center', gap: 20 }}>
          {avatarUrl && (
            <img
              src={avatarUrl}
              width={56}
              height={56}
              style={{ borderRadius: '50%', objectFit: 'cover' }}
              alt={author}
            />
          )}
          <div style={{ display: 'flex', flexDirection: 'column', gap: 4 }}>
            <span style={{ fontSize: 22, fontWeight: 700, color: '#f1f5f9' }}>
              {author}
            </span>
            {date && (
              <span style={{ fontSize: 18, color: '#94a3b8' }}>
                {new Date(date).toLocaleDateString('en-US', {
                  month: 'long',
                  day: 'numeric',
                  year: 'numeric',
                })}
              </span>
            )}
          </div>
        </div>
      </div>
    ),
    { width: 1200, height: 630 }
  );
}
```

Usage URL: `/og/blog?title=My+Post&author=Jane&date=2024-01-15&category=Engineering&avatar=https://...`

---

## 6. Product OG Template

Combines a product photo on the right with metadata on the left — a split layout.

```tsx
// app/og/product/route.tsx
import { ImageResponse } from '@vercel/og';
import type { NextRequest } from 'next/server';

export const runtime = 'edge';

function StarRating({ rating }: { rating: number }) {
  return (
    <div style={{ display: 'flex', gap: 4 }}>
      {Array.from({ length: 5 }).map((_, i) => (
        <span
          key={i}
          style={{ fontSize: 28, color: i < Math.round(rating) ? '#f59e0b' : '#d1d5db' }}
        >
          ★
        </span>
      ))}
    </div>
  );
}

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);

  const name = (searchParams.get('name') ?? 'Product').slice(0, 60);
  const price = searchParams.get('price') ?? '';
  const rating = parseFloat(searchParams.get('rating') ?? '0');
  const reviewCount = searchParams.get('reviews') ?? '';
  const imageUrl = searchParams.get('image') ?? '';
  const badge = searchParams.get('badge') ?? ''; // e.g. "New" or "Sale"

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          width: '100%',
          height: '100%',
          background: '#ffffff',
        }}
      >
        {/* Left: product info */}
        <div
          style={{
            display: 'flex',
            flexDirection: 'column',
            justifyContent: 'center',
            padding: '60px 64px',
            flex: 1,
            gap: 24,
          }}
        >
          {badge && (
            <div
              style={{
                display: 'flex',
                alignSelf: 'flex-start',
                background: '#ef4444',
                color: '#fff',
                borderRadius: 8,
                padding: '4px 16px',
                fontSize: 18,
                fontWeight: 700,
                textTransform: 'uppercase',
              }}
            >
              {badge}
            </div>
          )}

          <h1 style={{ fontSize: 52, fontWeight: 800, color: '#111827', margin: 0, lineHeight: 1.2 }}>
            {name}
          </h1>

          {rating > 0 && (
            <div style={{ display: 'flex', alignItems: 'center', gap: 12 }}>
              <StarRating rating={rating} />
              {reviewCount && (
                <span style={{ fontSize: 20, color: '#6b7280' }}>
                  ({reviewCount} reviews)
                </span>
              )}
            </div>
          )}

          {price && (
            <div style={{ display: 'flex', alignItems: 'baseline', gap: 8 }}>
              <span style={{ fontSize: 56, fontWeight: 800, color: '#111827' }}>
                {price}
              </span>
            </div>
          )}

          <div
            style={{
              display: 'flex',
              marginTop: 8,
              fontSize: 20,
              color: '#6366f1',
              fontWeight: 600,
            }}
          >
            mystore.com
          </div>
        </div>

        {/* Right: product image */}
        {imageUrl && (
          <div
            style={{
              display: 'flex',
              width: 480,
              background: '#f3f4f6',
              alignItems: 'center',
              justifyContent: 'center',
            }}
          >
            <img
              src={imageUrl}
              style={{ width: '100%', height: '100%', objectFit: 'cover' }}
              alt={name}
            />
          </div>
        )}
      </div>
    ),
    { width: 1200, height: 630 }
  );
}
```

---

## 7. Twitter Card vs OG (Different Sizes)

Twitter/X `summary_large_image` uses a 1200×628 crop (effectively identical to OG). Twitter `summary` uses a 1:1 card at 800×800. LinkedIn and Facebook both read `og:image` at 1200×630.

**Dedicated Twitter square image route:**

```tsx
// app/og/twitter/route.tsx — 800x800 for summary card
import { ImageResponse } from '@vercel/og';
import type { NextRequest } from 'next/server';

export const runtime = 'edge';

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const title = (searchParams.get('title') ?? '').slice(0, 50);

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          flexDirection: 'column',
          width: '100%',
          height: '100%',
          alignItems: 'center',
          justifyContent: 'center',
          background: '#1d1f27',
          padding: 60,
          gap: 24,
        }}
      >
        <div style={{ fontSize: 28, color: '#6366f1', fontWeight: 700 }}>
          mysite.com
        </div>
        <h1 style={{ fontSize: 52, fontWeight: 800, color: '#fff', textAlign: 'center', margin: 0 }}>
          {title}
        </h1>
      </div>
    ),
    { width: 800, height: 800 }
  );
}
```

**Metadata wiring for both cards:**

```tsx
// In generateMetadata
const ogImageUrl = `${siteUrl}/og?title=${encodeURIComponent(title)}`;
const twitterSquareUrl = `${siteUrl}/og/twitter?title=${encodeURIComponent(title)}`;

return {
  openGraph: {
    images: [{ url: ogImageUrl, width: 1200, height: 630, alt: title }],
  },
  twitter: {
    card: 'summary_large_image',      // use 'summary' to trigger square crop
    images: [ogImageUrl],             // swap to twitterSquareUrl for summary
  },
};
```

**Platform requirements:**

| Platform | Property | Minimum size | Recommended |
|---|---|---|---|
| Facebook | `og:image` | 200×200 | 1200×630 |
| Twitter/X | `twitter:image` | 144×144 | 1200×628 |
| LinkedIn | `og:image` | 1200×627 | 1200×627 |
| Slack | `og:image` | any | 1200×630 |

Always provide `og:image:width` and `og:image:height` — scrapers use these to avoid layout shift while fetching.

---

## 8. Next.js Metadata API

### `generateMetadata` (dynamic metadata function)

```tsx
// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

const SITE_URL = process.env.NEXT_PUBLIC_SITE_URL!;

export async function generateMetadata({
  params,
}: {
  params: { slug: string };
}): Promise<Metadata> {
  const post = await getPostBySlug(params.slug);

  if (!post) {
    return { title: 'Not Found' };
  }

  const ogUrl = new URL('/og/blog', SITE_URL);
  ogUrl.searchParams.set('title', post.title);
  ogUrl.searchParams.set('author', post.author.name);
  ogUrl.searchParams.set('date', post.publishedAt);
  ogUrl.searchParams.set('category', post.category);
  if (post.author.avatarUrl) {
    ogUrl.searchParams.set('avatar', post.author.avatarUrl);
  }

  return {
    title: post.title,
    description: post.excerpt,
    authors: [{ name: post.author.name }],
    openGraph: {
      type: 'article',
      title: post.title,
      description: post.excerpt,
      publishedTime: post.publishedAt,
      authors: [post.author.name],
      images: [
        {
          url: ogUrl.toString(),
          width: 1200,
          height: 630,
          alt: post.title,
        },
      ],
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.excerpt,
      images: [ogUrl.toString()],
    },
  };
}
```

### `opengraph-image.tsx` file convention

Next.js supports a co-located file convention that auto-wires the `og:image` meta tag — no manual `generateMetadata` needed for the image.

```
app/
  blog/
    [slug]/
      page.tsx
      opengraph-image.tsx    ← auto-generates og:image for this route
      twitter-image.tsx      ← auto-generates twitter:image
```

```tsx
// app/blog/[slug]/opengraph-image.tsx
import { ImageResponse } from 'next/og';

export const runtime = 'edge';
export const alt = 'Blog post image';
export const size = { width: 1200, height: 630 };
export const contentType = 'image/png';

export default async function Image({
  params,
}: {
  params: { slug: string };
}) {
  const post = await getPostBySlug(params.slug);

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          flexDirection: 'column',
          width: '100%',
          height: '100%',
          background: '#0f172a',
          padding: 64,
          justifyContent: 'center',
          gap: 24,
        }}
      >
        <h1 style={{ fontSize: 64, color: '#f8fafc', fontWeight: 800, margin: 0 }}>
          {post.title}
        </h1>
        <p style={{ fontSize: 28, color: '#94a3b8', margin: 0 }}>
          {post.excerpt}
        </p>
      </div>
    ),
    { ...size }
  );
}
```

The file convention is simpler but less flexible than the manual route approach. Use the manual `/og/route.tsx` when you need shared templates across many route segments.

---

## 9. Font Loading in OG

Satori (the renderer behind `@vercel/og`) requires fonts as `ArrayBuffer`. System fonts are not available at the Edge.

### Google Fonts (Edge-compatible fetch)

```tsx
// app/og/route.tsx
import { ImageResponse } from '@vercel/og';

export const runtime = 'edge';

// Fetch once per cold start — cached by the runtime between requests
async function loadGoogleFont(family: string, weight: number): Promise<ArrayBuffer> {
  const url = `https://fonts.googleapis.com/css2?family=${family}:wght@${weight}&display=swap`;
  const css = await fetch(url, {
    headers: { 'User-Agent': 'Mozilla/5.0' }, // required — Google blocks bots otherwise
  }).then((r) => r.text());

  // Extract the actual font file URL from the CSS
  const match = css.match(/src: url\((.+?)\) format\('(opentype|truetype|woff2)'\)/);
  if (!match) throw new Error('Could not parse Google Fonts CSS');

  return fetch(match[1]).then((r) => r.arrayBuffer());
}

export async function GET() {
  const interBold = await loadGoogleFont('Inter', 700);

  return new ImageResponse(
    (
      <div
        style={{
          display: 'flex',
          fontFamily: 'Inter',
          fontSize: 64,
          fontWeight: 700,
          color: '#111',
          width: '100%',
          height: '100%',
          alignItems: 'center',
          justifyContent: 'center',
          background: '#fff',
        }}
      >
        Hello World
      </div>
    ),
    {
      width: 1200,
      height: 630,
      fonts: [
        {
          name: 'Inter',
          data: interBold,
          weight: 700,
          style: 'normal',
        },
      ],
    }
  );
}
```

### Self-hosted font (fastest, no external fetch)

```tsx
// Place font file at: public/fonts/Inter-Bold.woff
// In your OG route:
const fontData = await fetch(
  new URL('/fonts/Inter-Bold.woff', process.env.NEXT_PUBLIC_SITE_URL)
).then((r) => r.arrayBuffer());
```

### Font caching pattern (module-level)

Fetching fonts on every request adds ~200ms latency. Cache at module scope — the Edge runtime reuses the module between warm invocations:

```tsx
// Module-level cache — persists across warm requests on the same Edge instance
let cachedFont: ArrayBuffer | null = null;

async function getFont(): Promise<ArrayBuffer> {
  if (cachedFont) return cachedFont;
  cachedFont = await fetch('https://...').then((r) => r.arrayBuffer());
  return cachedFont;
}
```

---

## 10. Testing OG Images

### Local browser preview

Navigate directly to your OG route during `next dev`:

```
http://localhost:3000/og?title=Hello+World&author=Jane
http://localhost:3000/og/blog?title=My+Post&date=2024-01-15
```

The browser renders the PNG inline. Resize the browser to check text truncation at different widths.

### Automated snapshot test with Playwright

```ts
// tests/og-images.spec.ts
import { test, expect } from '@playwright/test';

const cases = [
  { name: 'short-title', params: 'title=Hi' },
  { name: 'long-title', params: `title=${'A'.repeat(70)}` },
  { name: 'with-author', params: 'title=Post&author=Jane+Doe&date=2024-01-01' },
];

for (const { name, params } of cases) {
  test(`OG image: ${name}`, async ({ page }) => {
    const response = await page.goto(`http://localhost:3000/og?${params}`);
    expect(response?.status()).toBe(200);
    expect(response?.headers()['content-type']).toContain('image/png');
    await expect(page).toHaveScreenshot(`og-${name}.png`);
  });
}
```

### Online debuggers

| Tool | URL | Notes |
|---|---|---|
| OpenGraph.xyz | https://www.opengraph.xyz | Paste any URL, see real preview |
| Twitter Card Validator | https://cards-dev.twitter.com/validator | Official Twitter preview tool |
| LinkedIn Post Inspector | https://www.linkedin.com/post-inspector | Clears LinkedIn cache too |
| Facebook Sharing Debugger | https://developers.facebook.com/tools/debug | Refreshes Facebook scrape cache |
| og:image.app | https://ogimage.app | Visualizes all OG tags + preview |

After deploying changes, use the LinkedIn and Facebook tools to bust their scrape cache — they cache aggressively and won't show the new image otherwise.

---

## 11. Performance

### Cache headers

By default, Vercel caches `ImageResponse` for 0 seconds. Set explicit `Cache-Control` to avoid regenerating on every social media bot request:

```tsx
export async function GET(request: NextRequest) {
  const imageResponse = new ImageResponse(/* ... */, { width: 1200, height: 630 });

  // Return a new Response with cache headers
  return new Response(imageResponse.body, {
    headers: {
      'Content-Type': 'image/png',
      // Cache for 1 hour at the edge, serve stale for up to 1 day while revalidating
      'Cache-Control': 'public, max-age=3600, s-maxage=3600, stale-while-revalidate=86400',
    },
  });
}
```

For content that changes on deploy only (blog post OG images whose design may update):

```
Cache-Control: public, max-age=31536000, immutable
```

Include a version or content hash in the URL to bust the cache when design changes:

```ts
ogUrl.searchParams.set('v', process.env.NEXT_PUBLIC_BUILD_ID ?? '1');
```

### ISR-style revalidation

If the OG image depends on data that changes (e.g. a product's price), set a shorter `s-maxage` with `stale-while-revalidate`:

```
Cache-Control: public, s-maxage=300, stale-while-revalidate=600
```

This serves a cached image for up to 5 minutes, then revalidates in the background — bots always get a fast response.

### Avoid per-request font fetching

Move font fetches to module scope (see Section 9). Cold starts pay the fetch cost once; subsequent warm requests reuse the cached buffer.

### Measure real latency

```bash
# Time the OG route, bypassing browser cache
curl -o /dev/null -s -w "Total: %{time_total}s\n" \
  "https://mysite.com/og?title=Test"
```

Target: < 300ms on warm Edge invocation, < 800ms on cold start.

---

## 12. Anti-Patterns

### Same OG image for every page

```tsx
// BAD — every page shares one static image
export const metadata = {
  openGraph: { images: ['/og-default.png'] },
};
```

Social platforms show the same generic preview regardless of page content. Click-through rate drops significantly. Fix: use `generateMetadata` with per-page `og:image` URLs.

### Text overflow without length clamping

```tsx
// BAD — title could be 300 characters, satori clips it invisibly
<h1 style={{ fontSize: 72 }}>{post.title}</h1>

// GOOD — clamp and scale font size based on length
const safeTitle = post.title.slice(0, 80);
const fontSize = safeTitle.length > 50 ? 52 : 72;
<h1 style={{ fontSize }}>{safeTitle}</h1>
```

Satori does not wrap text the same way browsers do. Always test with your longest realistic title.

### Wrong dimensions

```tsx
// BAD — too small, platforms upscale and it looks blurry
new ImageResponse(<div />, { width: 600, height: 315 });

// BAD — non-standard ratio, gets letterboxed
new ImageResponse(<div />, { width: 1200, height: 400 });

// GOOD
new ImageResponse(<div />, { width: 1200, height: 630 }); // 1.91:1 ratio
```

### Using Grid layout

```tsx
// BAD — satori does not support CSS Grid
<div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr' }}>

// GOOD — use flexbox exclusively
<div style={{ display: 'flex', flexDirection: 'row' }}>
  <div style={{ flex: 1 }}>Left</div>
  <div style={{ flex: 1 }}>Right</div>
</div>
```

### Fetching images without error handling

```tsx
// BAD — if avatar fetch fails, the whole OG route returns 500
const avatar = await fetch(avatarUrl).then((r) => r.arrayBuffer());

// GOOD — fall back gracefully
let avatarData: ArrayBuffer | null = null;
try {
  const res = await fetch(avatarUrl);
  if (res.ok) avatarData = await res.arrayBuffer();
} catch {
  // Render without avatar
}
```

### Hardcoded localhost URL in production

```tsx
// BAD
const logoUrl = 'http://localhost:3000/logo.png';

// GOOD
const logoUrl = new URL('/logo.png', process.env.NEXT_PUBLIC_SITE_URL);
```

Always set `NEXT_PUBLIC_SITE_URL` in your deployment environment variables.

### Skipping `og:image:width` and `og:image:height`

```tsx
// BAD — scrapers fetch blindly, causing layout shifts in link previews
openGraph: { images: [{ url: ogUrl }] }

// GOOD — always declare dimensions
openGraph: {
  images: [{ url: ogUrl, width: 1200, height: 630, alt: post.title }]
}
```
