# Output Contract

When this skill is used, return the eight sections below in this exact order. If a section has nothing to say, write `Not applicable` instead of omitting it.

## Template

```markdown
## 1. Architecture Classification
- Route/page/layout role:
- Primary visible regions:
- Secondary/optional regions:
- Interactive regions:
- Current architecture smell, if any:

## 2. Boundary Assessment
- Server Components:
- Async Server Component regions:
- Client Component leaves/wrappers:
- Boundary risks:
- Props crossing Server → Client:

## 3. Data Ownership Map
| Read/mutation | Current owner | Recommended owner | Reason |
| --- | --- | --- | --- |

## 4. Loading and Error Topology
- Route-level `loading.tsx` role:
- Local Suspense boundaries:
- Skeleton placement:
- Error boundaries/recovery:
- Static shell preservation:

## 5. Client Data/Cache Note
- Existing client data tools:
- Browser-owned cache signals found (`useSWR`, `SWRConfig`, `fallbackData`, `mutate`, `refreshInterval`, focus/reconnect revalidation, filter-driven keys, TanStack/Apollo/Relay/custom cache APIs):
- What to preserve as a Client leaf/wrapper:
- What can move server-side or seed initial client cache:
- Invalidation/mutation convention:

## 6. Next.js Version Notes
- Detected Next.js version:
- `params` / `searchParams` behavior:
- `notFound`, redirect, auth, or metadata gates that justify route-level blocking:
- `cacheComponents` / `'use cache'` / `cacheLife` / `cacheTag` implications:
- Unknowns or caveats:

## 7. Recommended Changes
1. ...
2. ...
3. ...

## 8. Validation Plan
- Type/lint/build checks:
- Runtime loading/streaming checks:
- Error/recovery/not-found checks:
- Data/query/cache checks, including client-cache preservation:
- Version-specific checks:
- Evidence to capture:
```

## Validation Checklist

Use this checklist before finalizing recommendations.

### Architecture

- Page/layout primarily composes regions, Suspense boundaries, and error topology.
- Async work is owned by async Server Components where region-specific.
- Route-level async is justified by gating, redirects, auth, metadata/compliance, or truly shared reads.
- Components are classified as route compositor, Server Component region, Client leaf/wrapper, skeleton, error boundary, data helper, or action.
- The route avoids centralizing all data in a loader-like page unless there is an explicit reason.

### Loading and Error UX

- Suspense boundaries correspond to meaningful user-visible regions.
- Slow/dynamic regions have fallbacks.
- Skeletons are colocated and layout-matched.
- One generic all-page spinner is used only if the whole route truly blocks.
- Static shell content is not blocked by unrelated secondary dynamic data.
- Failure scope is explicit, with region or segment recovery behavior.

### Data Ownership and Performance

- Server-renderable reads happen in owning async Server Components.
- React `cache()` is described only as identical-call dedupe.
- N+1/multi-ID work is handled by batching, bulk APIs, dataloaders, joins, or query optimization.
- Independent reads are parallelized when safe.
- Intentional client-side data fetching is preserved.
- Existing client data/cache tools are preserved unless the user asks to replace them.

### Client Boundaries

- `'use client'` appears only where browser APIs, hooks, events, or client-only state are needed.
- Interactive widgets are leaves or wrappers rather than promoted page regions.
- Server Components may be passed as children/JSX props to Client wrappers.
- Props into Client Components are serializable and minimal.
- Server-only imports do not cross into Client Components.

### Mutations

- Server Functions/actions are used where project-standard and supported.
- `useOptimistic` or an equivalent immediate-feedback pattern is considered.
- Client cache invalidation/mutation follows the chosen project tool.

### Next.js Versioning

- Project Next.js version is checked before version-specific advice.
- Next.js 15 `params` / `searchParams` promise-like behavior is handled correctly when present.
- `params.then(...)` is considered when it keeps route compositors synchronous without hurting clarity.
- Async page components are justified when used.
- Next.js 16 `cacheComponents`, dynamic data, Suspense, and static shell implications are considered.
- Static shell prerendering is not accidentally defeated by unscoped dynamic reads.

### Evidence Requirements

- Cite files or components that prove the route shell stayed server-rendered and synchronous where intended.
- Cite where each server read moved or remained, including owner region and Suspense boundary.
- Cite where SWR/TanStack/Apollo/Relay/custom client cache tooling is preserved, including key/fetcher/invalidation behavior when relevant.
- Cite route prop types or generated type evidence when `params` / `searchParams` are promise-shaped.
- Cite `next.config.*` and cached helper/component code when recommending `cacheComponents`, `'use cache'`, `cacheLife`, or `cacheTag`.
- Cite `not-found.tsx`, `error.tsx`, local error boundaries, failure knobs, traces, or screenshots when discussing recovery topology.
- Include exact validation commands or explain why they cannot be run.
## Grading Cues for Evals

A strong answer should:

- use the strict eight-section contract
- avoid generic “use RSC” advice
- preserve intentional client cache/data tooling
- explain loading topology as UX, not just performance
- mention version caveats when `params`, `searchParams`, or `cacheComponents` appear
- distinguish `cache()` dedupe from batching
- preserve SWR or equivalent browser-owned cache islands when they encode polling, focus/refetch, filters, optimistic state, or shared client cache
- check `cacheComponents: true` before recommending Next 16 cache directives and use `cacheLife`/`cacheTag` only inside cacheable scopes
- recommend validation steps specific to architecture, loading, errors, data, and version behavior

A weak answer often:

- deletes SWR/TanStack Query/Apollo/urql/Relay by default
- makes the whole route a Client Component
- centralizes every read in `page.tsx`
- adds Suspense without meaningful fallbacks
- treats skeletons as generic spinners
- assumes sync route props without checking Next.js version
- treats Server Actions as the default read mechanism
- claims `cache()` batches different IDs
- removes an SWR island or rewrites it as a Server Component while the UX still depends on browser refresh, focus refetch, filters, or shared client cache
- recommends `'use cache'`, `cacheLife`, or `cacheTag` without checking `cacheComponents` or whether the scope is actually cacheable
- treats `notFound`/redirect/auth gates as the same thing as region error recovery
- gives no concrete evidence for boundaries, route prop versions, client cache preservation, or validation
