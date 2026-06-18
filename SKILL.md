---
name: next-app-router-rsc-page-composition
description: Design, review, and refactor React Server Component architecture in Next.js App Router apps. Use when a task mentions RSCs, Server Components, Client Components, Suspense loading boundaries, skeletons, async page components, route/page composition, params/searchParams promises, cacheComponents, Server Functions/actions, useOptimistic, or feature-folder organization. Also use when the user wants a route split into a synchronous shell plus async regions, or wants existing client data tooling preserved instead of replaced.
metadata:
  author: Tim Seriakov
  source: Aurora Scharff, Component Architecture for React Server Components
  source_url: https://aurorascharff.no/posts/component-architecture-for-react-server-components/
---

# Next.js App Router RSC Page Composition

Design the route as a composition shell, push reads into the components that own them, and make the loading experience explicit.

Source basis: this skill operationalizes Aurora Scharff’s experience and blog post “Component Architecture for React Server Components” into a review and refactoring workflow for Next.js App Router applications. Keep the article attribution visible because the architectural stance comes from that work, even though this reusable skill is authored separately.

## When to Apply

Use this skill for Next.js App Router work involving:

- React Server Components, Server Components, or Client Components
- route/page/layout architecture, especially async page components or loader-style pages
- splitting a route into a synchronous shell plus async Server Component regions
- Suspense boundaries, skeleton loading UI, streaming, or `loading.tsx`
- server/client boundary decisions and serializable props
- preserving SWR, TanStack Query, Apollo, urql, Relay, or custom client data hooks when intentional
- Server Functions/actions, optimistic UI, or mutation/cache invalidation design
- Next.js 15 `params` / `searchParams` promises or Next.js 16 `cacheComponents`
- feature-folder organization for route regions, skeletons, errors, data helpers, client leaves, and actions

Do not use this skill for styling-only work, generic React cleanup, or advice that would collapse into “just use Server Components” without component ownership and loading design.

## Core Model

The route file should usually describe the loading experience, not gather every data dependency.

| If you see... | Prefer... |
| --- | --- |
| Page/layout mostly arranges regions, fallbacks, and errors | Keep it synchronous |
| Region owns a server-renderable read | Make that region an async Server Component |
| UI needs browser state, effects, events, or browser APIs | Use a Client Component leaf or wrapper |
| Data is live, user-driven, subscription-based, browser-dependent, or intentionally cache-managed client-side | Preserve the client data/cache tool |
| Route must block for auth, redirect, notFound, metadata/compliance, or a truly shared route-wide read | Route-level async is justified |

Read `references/architecture-rules.md` for the full decision rules, loading policy, component roles, feature-folder guidance, and anti-patterns.

## Quick Reference by Priority

1. Classify the route: static shell, dynamic server regions, client leaves, or intentional client cache islands.
2. Keep `page.tsx`/layouts as synchronous composition shells unless route identity, redirect, auth, metadata, or `notFound()` truly requires blocking.
3. Move each server read to the smallest owning async Server Component under a local `<Suspense>` boundary.
4. Do not centralize independent reads in `page.tsx`; preserve region ownership and independent failure/loading behavior.
5. Preserve browser-owned cache/data tools (SWR, TanStack Query, Apollo, Relay, custom hooks) when they power polling, focus/refetch, filters, optimistic updates, subscriptions, or shared client cache.
6. Push `use client` down to tiny leaves and pass serializable props only.
7. Treat `cache()` as request-scope dedupe for identical calls, not batching for many distinct IDs.
8. For Next 15+, treat `params`/`searchParams` as promises; pass/await them deliberately instead of assuming synchronous props.
9. For Next 16 `cacheComponents`, check `next.config.*` before recommending `'use cache'`, `cacheLife`, or `cacheTag`, and use those directives only inside cacheable scopes.
10. Require evidence: changed files, boundary locations, validation commands, route behavior, preserved client cache tooling, and version assumptions.

Read `references/boundaries-and-data.md` for server/client boundary rules, official Next/Vercel-style data pattern guidance, serializable props, repeated reads, mutations, and client cache preservation.

Read `references/next-version-notes.md` for Next.js 15 thenable route props, `params.then(...)`, Next.js 16 `cacheComponents`, and unknown-version fallback behavior.

## Workflow

Follow this sequence when reviewing or refactoring a route:

1. **Inspect the project**
   - `package.json` for React and Next.js versions
   - `app/` structure, nested layouts, and route segment files
   - `loading.tsx`, `error.tsx`, `not-found.tsx`
   - Server and Client Components
   - data and mutation layers
   - existing folder conventions, skeletons, and fallbacks

2. **Classify visible regions**
   - primary, secondary, optional, and interactive regions
   - route-blocking versus independently streamable work
   - existing Suspense/error topology
   - whether the page is a compositor or a loader page

3. **Map ownership and split route shape**
   - map every read/mutation/cache path: source, frequency, owner, consumers, invalidation, and failure mode
   - detect browser-owned cache signals explicitly: `useSWR`, `SWRConfig`, `fallbackData`, `mutate`, `refreshInterval`, focus/reconnect revalidation, filter-driven keys, TanStack/Apollo/Relay cache APIs, or custom subscription hooks
   - split route shape into synchronous shell first, async Server Component regions second, and tiny Client leaves only where interaction/cache requires them
   - apply version rules while splitting: Next 15 promise route props, Next 16 `cacheComponents`/cache directives, and older-version fallbacks

4. **Design loading and failure together**
   - local Suspense versus route-segment `loading.tsx`
   - shape-matched skeletons
   - region or segment error boundaries
   - recovery behavior and graceful degradation

5. **Recommend changes**
   - preserve local conventions and project-standard tooling
   - order changes by impact and safety
   - include version caveats and validation steps

Read `references/transformations.md` for concrete examples: loader page split, `params.then(...)`, client leaf extraction, intentional client data tooling, `cache()` dedupe not batching, and `cacheComponents` static-shell/dynamic-region design.

## Output Contract

Return these sections in this exact order whenever this skill is used. If a section has nothing to say, write `Not applicable` rather than dropping it.

1. **Architecture Classification**
2. **Boundary Assessment**
3. **Data Ownership Map**
4. **Loading and Error Topology**
5. **Client Data/Cache Note**
6. **Next.js Version Notes**
7. **Recommended Changes**
8. **Validation Plan**

Read `references/output-contract.md` for the full template, validation checklist, and grading cues.

## Resources

- Aurora Scharff, “Component Architecture for React Server Components”: https://aurorascharff.no/posts/component-architecture-for-react-server-components/
- Aurora Scharff, “Server and Client Component Composition in Practice”: https://aurorascharff.no/posts/server-client-component-composition-in-practice/
- React docs: Server Components, `Suspense`, `cache()`, `useOptimistic`, Server Functions/actions
- Next.js docs: App Router, Server and Client Components, `loading.tsx`, `error.tsx`, `not-found.tsx`, `params`, `searchParams`, caching, `cacheComponents`
- Reference files in this skill:
  - `references/architecture-rules.md`
  - `references/boundaries-and-data.md`
  - `references/next-version-notes.md`
  - `references/transformations.md`
  - `references/output-contract.md`
