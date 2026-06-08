# Boundaries and Data

Use this reference for server/client boundary design, read ownership, serializable props, mutations, client cache preservation, repeated reads, and official Next/Vercel-style data-pattern choices.

## Read Ownership

Map every read to the smallest component or region that owns the rendered result.

Ask:

- Which region actually needs this data?
- Does the parent need the data to decide routing, auth, metadata, or layout?
- Can the child read it server-side instead of receiving a large payload?
- Is the data intentionally client-managed because it is live, user-driven, subscription-based, browser-dependent, or cache-coordinated?
- Are repeated reads identical, or are many distinct IDs creating N+1 work?

Prefer:

- Server Component reads for server-renderable UI reads.
- Server Functions/actions for mutations and server-side operations triggered from UI.
- Route Handlers for external clients, webhooks, public GET endpoints, mobile clients, or explicit HTTP APIs.
- Client data hooks when the UX needs browser-driven live state, polling, subscriptions, optimistic updates, or project-standard client cache coordination.

Avoid using Server Actions for ordinary reads by default. They are internal POST operations and usually do not provide the HTTP caching semantics expected for read endpoints.

## Server Component Reads

Server Components can read directly from databases, server SDKs, internal services, or cached helpers because they run on the server. Benefits:

- no client API roundtrip for initial render
- secrets stay server-side
- ownership stays close to rendering
- independent regions can stream behind Suspense

Do not move a read server-side if that destroys intentional live/browser-driven UX.

## Client Component Boundary Rules

Push `'use client'` down to the smallest component that needs browser-only behavior.

Use Client Components for:

- event handlers
- local state and effects
- browser APIs
- animations or DOM measurement
- optimistic UI hooks
- client cache hooks
- wrappers that need client context while accepting server-rendered children

Avoid:

- whole-route `'use client'` because one nested button needs interactivity
- importing server-only helpers into Client Components
- making async Client Components; fetch in a Server Component parent or a client hook instead
- passing non-serializable objects from Server to Client Components

Server Components may be passed as `children` or JSX props to Client Component wrappers. This is useful for interactive shells around server-rendered content.

## Serializable Props Across Boundaries

Props from Server Components to Client Components must be minimal and serializable.

Prefer passing:

- IDs, slugs, filters, cursors, handles
- small strings/numbers/booleans
- plain objects and arrays
- typed route prop thenables where valid
- initial values for client leaves

Avoid passing:

- functions, except Server Functions/actions explicitly marked for server use
- `Date` objects without serialization
- `Map`, `Set`, `WeakMap`, `WeakSet`
- class instances
- `Symbol`
- circular references
- database clients, request objects, response objects, headers/cookies objects, or other server-only resources
- large loader payloads through intermediate components

Convert values before crossing the boundary: `Date` to ISO string, `Map`/`Set` to arrays or plain objects, rich domain objects to plain view models.

## Server Functions/actions and Mutations

Use Server Functions/actions where the project and Next.js version support them, especially for form submissions and mutations that should run server-side.

Preserve immediate feedback with `useOptimistic` or the project's equivalent optimistic pattern when the UX depends on it.

For mutation aftermath, follow the project's chosen invalidation conventions:

- `revalidatePath` / `revalidateTag`
- router refresh
- TanStack Query invalidation
- SWR mutate
- Apollo/urql/Relay cache updates
- custom event or store invalidation

Do not replace a mature mutation/cache strategy just because a Server Action is possible.

## Preserve Intentional Client Data Tools

RSC does not make client data tools obsolete. Preserve SWR, TanStack Query, Apollo, urql, Relay, or custom hooks when they are serving an intentional UX or integration role:

- live filters
- polling
- subscriptions
- user-triggered refresh
- browser-only data
- optimistic local edits
- cache sharing across client interactions
- offline/persistent client state
- mature project-standard GraphQL/cache tooling

You may still move static initial reads server-side and seed or coordinate the client cache when useful. Separate initial render ownership from ongoing interaction ownership.

## Repeated Reads and Many IDs

React `cache()` dedupes identical calls during server rendering. It does not batch many different calls.

Use `cache()` for:

- identical server reads called from multiple components
- deduping stable helper calls with the same arguments
- preload patterns where the same promise is later consumed

Do not expect this to fix:

- 80 different `getProduct(id)` calls
- N+1 query patterns
- many unrelated API calls
- sequential waterfalls caused by component nesting

For many distinct IDs, prefer:

- bulk reads
- SQL joins or optimized queries
- dataloaders
- API batching
- precomputed read models
- region-level list reads that pass small props to cards

Validate by checking query counts, network calls, logs, traces, or timings, not by assuming server rendering fixed the issue.

## Waterfalls and Parallelism

Server rendering can still waterfall if reads are nested or sequential without a real dependency.

- Parallelize independent reads with `Promise.all` or sibling async regions.
- Sequence dependent reads only when the second read truly needs the first result.
- Use Suspense to stream independent slow regions.
- Avoid micro-Suspense around every card or primitive unless each item is truly independently meaningful.
- Consider preload helpers with `cache()` for shared identical reads.
