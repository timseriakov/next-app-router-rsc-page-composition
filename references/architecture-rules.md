# Architecture Rules

Use these rules when deciding whether a Next.js App Router route should stay a synchronous compositor, split into async Server Component regions, introduce local Suspense, or keep route-level blocking.

## Core Decision Rule

The route file should usually describe the loading experience, not gather every data dependency.

| If you see... | Prefer... |
| --- | --- |
| Page/layout mostly arranges regions, fallbacks, and errors | Keep it synchronous |
| Region owns a server-renderable read | Make that region an async Server Component |
| UI needs browser state, effects, events, or browser APIs | Use a Client Component leaf or wrapper |
| Data is live, user-driven, subscription-based, browser-dependent, or intentionally cache-managed client-side | Preserve the client data/cache tool |
| Route must block for auth, redirect, notFound, metadata/compliance, or a truly shared route-wide read | Route-level async is justified |

Do not turn a route into an async loader page just because several children need data. Extract the children into async regions and let the route compose those regions with loading and error boundaries.

## Route-Level Async Is Exceptional

An async `page.tsx` or `layout.tsx` is justified when the route itself cannot render safely before the result is known:

- authentication or authorization gating
- redirect or `notFound()` decisions before rendering
- metadata, compliance, locale, tenant, or policy checks that govern the whole route
- a truly shared route-wide read that every visible region depends on
- framework-required async route props that cannot be handled as typed thenables without harming clarity

If only one visible region needs the data, move the read into that region instead.

## Loading Policy: `loading.tsx` vs Local `Suspense`

Prefer **route-segment `loading.tsx`** when the whole route segment or navigation state should show one fallback while the segment loads. This is useful for coarse navigation-level loading.

Prefer **local `<Suspense>` boundaries** when independent user-visible regions can stream separately. This is the default for the Aurora Scharff-style component architecture: the page describes the loading experience by placing meaningful fallbacks around async Server Component regions.

Rules:

- Put Suspense boundaries around meaningful visual regions, not every tiny component.
- Do not use one generic all-page spinner unless the whole route truly blocks.
- Keep static shell content visible when unrelated secondary dynamic data is pending.
- Colocate each skeleton with the component it represents.
- Make skeletons mirror the fulfilled component's layout to avoid drift and layout shift.
- Pair loading and error decisions: if a region can load independently, decide whether it should also fail independently.

## Component Roles

Classify components before recommending changes. A component may have secondary roles, but name a primary role first.

- **Route compositor**: `page.tsx` that arranges regions, Suspense, and error topology.
- **Layout compositor**: `layout.tsx` that arranges shared shell and nested segments.
- **Async Server Component region**: server component that owns a region-local read.
- **Pure Server Component view**: server-rendered view with no own async read.
- **Client Component leaf**: small interactive component for browser-only behavior.
- **Client Component wrapper**: client boundary that accepts server-rendered children or JSX props.
- **Skeleton/fallback**: colocated loading UI with matching geometry.
- **Error fallback/boundary**: segment or region failure UI and recovery behavior.
- **Server data helper**: server-only read helper, query, loader, or cached function.
- **Client data hook**: intentional client cache/live data hook.
- **Server Function/action**: mutation or server-side operation callable from UI.
- **Shared UI primitive**: reusable component with no route ownership decision by itself.

When one component appears to be both a region and a client boundary, ask why it has `'use client'`. Usually extract a server region plus a client leaf.

## Feature Folder Guidance

Adapt to the repository's existing conventions, but prefer keeping a route region's pieces close together:

```text
app/dashboard/
  page.tsx
  _components/
    metrics/
      MetricsRegion.tsx
      MetricsSkeleton.tsx
      MetricsError.tsx
      data.ts
    notifications/
      NotificationsRegion.tsx
      NotificationsSkeleton.tsx
      NotificationsClientControls.tsx
      actions.ts
```

The useful grouping is conceptual, not mandatory naming. Keep the component, skeleton, error UI, data helper, client leaf, and actions discoverable together when they evolve together.

## Anti-Patterns to Call Out

- **Async Everything Page**: every read is awaited in `page.tsx` before any shell can render.
- **Route Loader Redux**: the route becomes a central loader that passes large payloads down.
- **Suspense Afterthought**: boundaries are added after fetching decisions instead of shaping the UX.
- **Skeleton Drift**: fallback layout no longer matches the fulfilled component.
- **Client Boundary Creep**: `'use client'` spreads upward until whole regions become client-only.
- **Client Data Tool Dogma**: deleting SWR/TanStack Query/Apollo/urql/Relay only because RSC exists.
- **RSC Absolutism**: assuming all data must move server-side.
- **Cache Means Batch Fallacy**: treating React `cache()` as a batching mechanism.
- **Boundary Prop Flood**: passing large loader payloads or server-only objects through client seams.
- **Unscoped Failure**: one secondary region failure breaks the whole route unnecessarily.
- **Next Version Blindness**: assuming sync `params` or ignoring `cacheComponents` behavior.
- **Feature Scatter**: component, skeleton, data helper, and action live far apart without project reason.
