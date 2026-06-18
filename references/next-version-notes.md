# Next.js Version Notes

Always inspect `package.json`, framework typings, and existing route signatures before giving version-specific advice. Do not assume all App Router projects have the same `params`, `searchParams`, caching, or dynamic rendering behavior.

## Next.js 15: `params` and `searchParams` Thenables

In Next.js 15-era App Router code, `params` and `searchParams` can be promise-like route props. Treat the actual project typings as authoritative.

When a route is only async because it awaits `params`, consider deriving minimal typed promises and keeping the page as a synchronous compositor.

Example pattern:

```tsx
type PageProps = {
  params: Promise<{ slug: string }>
}

export default function Page({ params }: PageProps) {
  const slug = params.then((p) => p.slug)

  return (
    <>
      <ProductHeader slug={slug} />
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews slug={slug} />
      </Suspense>
    </>
  )
}

async function ProductHeader({ slug }: { slug: Promise<string> }) {
  const product = await getProductBySlug(await slug)
  return <h1>{product.name}</h1>
}
```

Use this only when it improves architecture and type clarity. If route-level gating or redirect decisions require awaiting the value in the page, async page can be justified.

## `searchParams`

Apply the same caution to `searchParams`:

- inspect whether it is sync or promise-like in the project
- derive only the minimal filter/sort/page values needed
- pass small typed values or promises to owning regions
- avoid turning the page into a centralized query parser plus loader unless the route truly owns those decisions

## Next.js 16: `cacheComponents`, `'use cache'`, `cacheLife`, `cacheTag`

With `cacheComponents`, static shell versus dynamic region placement becomes more important, and cache directives become explicit opt-ins rather than generic performance decoration.

First check `next.config.*`:

```ts
const nextConfig = {
  cacheComponents: true,
}

export default nextConfig
```

Evaluate:

- Is `cacheComponents: true` enabled before recommending cache directives?
- Which parts of the route are static/cacheable copy or chrome?
- Which reads use cookies, headers, request-specific data, uncached fetches, browser-owned filters, or other dynamic sources?
- Can dynamic reads move behind meaningful Suspense boundaries?
- Does an unscoped dynamic read in `page.tsx` accidentally make the whole route dynamic?
- Do fallbacks preserve layout while dynamic regions stream?

Use `'use cache'`, `cacheLife`, and `cacheTag` only inside genuinely cacheable Server Component or server-helper scopes:

```tsx
import { cacheLife, cacheTag } from 'next/cache'

export async function getMarketingHeroCopy() {
  'use cache'

  cacheTag('marketing-hero-copy')
  cacheLife('hours')

  return getStaticMarketingCopy()
}
```

Rules:

- import `cacheLife` and `cacheTag` from `next/cache`
- call `cacheLife(...)` and `cacheTag(...)` inside the cached scope
- do not pass runtime promises, request-specific values, SWR state, cookies, or browser-owned filters into a shared `'use cache'` scope
- keep uncached personalized/request-time regions outside the cached helper and behind their own Suspense boundary
- use `'use cache: private'` or `'use cache: remote'` only when the project version and route context support them and the cache ownership is clear
- do not use cache directives decoratively just to make a route look “Next 16 aware”

Prefer preserving static shell content while wrapping personalized or dynamic regions locally:

```tsx
export default function Page() {
  return (
    <main>
      <Hero />
      <Pricing />
      <Suspense fallback={<RecommendationsSkeleton />}>
        <PersonalizedRecommendations />
      </Suspense>
    </main>
  )
}
```
## Unknown or Mixed Versions

If the Next.js version cannot be inspected:

- say that version-specific behavior is unknown
- ask for `package.json`, route prop types, or generated type errors when needed
- give conditional guidance instead of asserting sync or async route props
- avoid recommending Next 15/16-specific APIs as mandatory
- still apply the stable architecture principles: synchronous compositor, async server regions, meaningful Suspense, colocated skeletons, leaf Client Components, and intentional client cache preservation

## Validation for Version-Specific Advice

Include validation steps such as:

- run TypeScript or the project's typecheck command
- inspect generated route prop types
- verify streaming/fallback behavior in development
- verify static/dynamic behavior after `cacheComponents` changes
- confirm redirects/notFound/auth behavior still happens before rendering when required
