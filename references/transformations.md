# Transformations

Use these examples as patterns, not as recipes to apply blindly. Preserve local naming, folder conventions, data tools, and validation commands.

## 1. Loader Page to Synchronous Shell plus Async Regions

Problem:

```tsx
export default async function DashboardPage() {
  const user = await requireUser()
  const metrics = await getMetrics(user.id)
  const notifications = await getNotifications(user.id)
  const recommendations = await getRecommendations(user.id)

  return <Dashboard metrics={metrics} notifications={notifications} recommendations={recommendations} />
}
```

Better shape:

```tsx
export default async function DashboardPage() {
  const user = await requireUser() // route-level auth gate is justified

  return (
    <DashboardShell user={user}>
      <Suspense fallback={<MetricsSkeleton />}>
        <MetricsRegion userId={user.id} />
      </Suspense>
      <Suspense fallback={<NotificationsSkeleton />}>
        <NotificationsRegion userId={user.id} />
      </Suspense>
      <Suspense fallback={<RecommendationsSkeleton />}>
        <RecommendationsRegion userId={user.id} />
      </Suspense>
    </DashboardShell>
  )
}

async function MetricsRegion({ userId }: { userId: string }) {
  const metrics = await getMetrics(userId)
  return <MetricsView metrics={metrics} />
}
```

Notes:

- Keep auth/redirect gating at route level when it must block.
- Move region-specific reads into owning async Server Components.
- Add meaningful Suspense and colocated skeletons.
- Scope errors per region or segment where possible.
- Do not promote the route to `'use client'`.

## 2. Keep a Page Synchronous with `params.then(...)`

Problem:

```tsx
export default async function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  const product = await getProduct(slug)
  const reviews = await getReviews(slug)

  return <Product product={product} reviews={reviews} />
}
```

Better shape when route-level blocking is not needed:

```tsx
type PageProps = {
  params: Promise<{ slug: string }>
}

export default function ProductPage({ params }: PageProps) {
  const slug = params.then((p) => p.slug)

  return (
    <>
      <Suspense fallback={<ProductSummarySkeleton />}>
        <ProductSummary slug={slug} />
      </Suspense>
      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews slug={slug} />
      </Suspense>
    </>
  )
}

async function ProductSummary({ slug }: { slug: Promise<string> }) {
  const product = await getProduct(await slug)
  return <ProductSummaryView product={product} />
}
```

Notes:

- Inspect actual Next.js version and generated types first.
- Pass minimal typed thenables or values.
- Keep the page focused on composition and loading topology.
- If the page must redirect/notFound based on slug lookup, route-level async may still be justified.

## 3. Extract a Client Leaf from an Overgrown Client Region

Problem:

```tsx
'use client'

export default function SettingsPage() {
  const account = getAccountOnServerSomehow() // invalid direction
  return (
    <section>
      <h1>{account.name}</h1>
      <SaveButton />
    </section>
  )
}
```

Better shape:

```tsx
export default async function SettingsPage() {
  const account = await getAccount()

  return (
    <section>
      <h1>{account.name}</h1>
      <SaveSettingsForm accountId={account.id} initialName={account.name} />
    </section>
  )
}
```

```tsx
'use client'

export function SaveSettingsForm({ accountId, initialName }: { accountId: string; initialName: string }) {
  // browser state, events, optimistic UI, action invocation
}
```

Notes:

- Move server reads out of Client Components.
- Keep the browser-only part as a small leaf or wrapper.
- Pass minimal serializable props.
- Preserve `useOptimistic` or equivalent immediate feedback where the UX requires it.

## 4. Preserve Intentional Client Data Tooling

Problem:

> “Delete TanStack Query because RSC can fetch server-side.”

Better recommendation:

- Keep TanStack Query for live filters, polling, user-triggered refresh, optimistic edits, cache sharing, or browser-dependent data.
- Move only static initial reads server-side when that improves first render without harming UX.
- Optionally seed or hydrate the client cache if the project already does this.
- Follow existing mutation invalidation conventions.

Example shape:

```tsx
export default async function AnalyticsPage() {
  const initialFilters = await getDefaultFilters()

  return (
    <AnalyticsShell>
      <LiveAnalyticsPanel initialFilters={initialFilters} />
    </AnalyticsShell>
  )
}
```

```tsx
'use client'

function LiveAnalyticsPanel({ initialFilters }: { initialFilters: Filters }) {
  // TanStack Query remains because filters, polling, refresh, and optimistic UX are client-owned.
}
```

SWR-specific shape:

```tsx
export default function CatalogCategoryPage({ params }: { params: Promise<{ category: string }> }) {
  const categoryPromise = params.then(({ category }) => category)

  return (
    <main>
      <CatalogShell />
      <Suspense fallback={<InventorySkeleton />}>
        <InventoryCacheRegion category={categoryPromise} />
      </Suspense>
    </main>
  )
}
```

```tsx
async function InventoryCacheRegion({ category }: { category: Promise<string> }) {
  const resolvedCategory = await category
  const initialInventory = await getInitialInventory(resolvedCategory)

  return (
    <InventoryBadgeClient
      category={resolvedCategory}
      fallbackData={initialInventory}
    />
  )
}
```

```tsx
'use client'

import useSWR from 'swr'

function InventoryBadgeClient({ category, fallbackData }: Props) {
  const { data, isLoading, isValidating, mutate } = useSWR(
    `/api/inventory?category=${category}`,
    fetchInventory,
    { fallbackData, revalidateOnFocus: false, keepPreviousData: true },
  )

  // SWR remains because browser refresh, local filters, validation state, and cache updates are client-owned.
}
```

Do not replace this leaf with a Server Component read unless the browser-owned cache behavior is intentionally removed by the user.

## 5. `cache()` Dedupe Is Not Batching

Problem:

```tsx
const getProductCached = cache(getProduct)

async function ProductCard({ id }: { id: string }) {
  const product = await getProductCached(id)
  return <Card product={product} />
}

export function ProductGrid({ ids }: { ids: string[] }) {
  return ids.map((id) => <ProductCard key={id} id={id} />)
}
```

If there are 80 different IDs, `cache()` does not make that one batched read. It only dedupes identical calls.

Better shape:

```tsx
async function ProductGridRegion({ ids }: { ids: string[] }) {
  const products = await getProductsByIds(ids)

  return products.map((product) => <ProductCard key={product.id} product={product} />)
}
```

Use bulk reads, joins, dataloaders, API batching, or precomputed read models. Validate query count and waterfall behavior.

## 6. `cacheComponents`: Preserve Static Shell, Isolate Dynamic Region

Problem:

```tsx
export default async function MarketingPage() {
  const recommendations = await getPersonalizedRecommendations() // cookies/user-specific

  return (
    <main>
      <Hero />
      <Pricing />
      <Recommendations items={recommendations} />
    </main>
  )
}
```

Better shape:

```tsx
export default function MarketingPage() {
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

async function PersonalizedRecommendations() {
  const recommendations = await getPersonalizedRecommendations()
  return <Recommendations items={recommendations} />
}
```

If the project is on Next.js 16 and `cacheComponents: true` is enabled, cache truly static server helpers explicitly:

```tsx
import { cacheLife, cacheTag } from 'next/cache'

export async function getMarketingHeroCopy() {
  'use cache'

  cacheTag('marketing-hero-copy')
  cacheLife('hours')

  return getStaticMarketingCopy()
}
```

Notes:

- Keep static shell content outside unscoped dynamic reads.
- Put personalized/dynamic data behind meaningful Suspense.
- Add region-level skeleton and error recovery.
- Use `'use cache'`, `cacheLife`, and `cacheTag` only inside cacheable server scopes after confirming `cacheComponents: true`.
- Do not pass runtime promises, request-specific values, cookies, or browser-owned filter state into a shared cached scope.
- Verify static/dynamic behavior under the project's Next.js version and `cacheComponents` settings.

## 7. Nested Failure and Not Found Topology

Identity failures are not the same as region failures.

```tsx
export default function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  const slugPromise = params.then(({ slug }) => slug)

  return (
    <main>
      <Suspense fallback={<SummarySkeleton />}>
        <ProductSummaryRegion slug={slugPromise} />
      </Suspense>
      <Suspense fallback={<ReviewsSkeleton />}>
        <ReviewsRegion slug={slugPromise} />
      </Suspense>
    </main>
  )
}
```

Use `notFound()` in the smallest route/region gate that proves the route identity is invalid. Use scoped error boundaries or recoverable region UI for failures in optional or independent regions. A reviews failure should not blank a valid product summary unless the product identity itself is invalid.
