# Next.js App Router RSC Page Composition

An OpenCode skill for reviewing and refactoring React Server Component architecture in Next.js App Router applications.

The skill helps an assistant turn loader-style routes into composition-oriented pages: keep the route shell focused on layout, loading, and error topology; move server reads into the regions that own them; push client boundaries down to leaves; and preserve intentional client data/cache tooling when it is the right tool for the UX.

## What it helps with

Use this skill when working on Next.js App Router code that involves:

- React Server Components, Server Components, and Client Components
- async `page.tsx` / `layout.tsx` files that behave like route-level loaders
- route shells that should stay synchronous while async regions stream independently
- Suspense boundaries, skeletons, `loading.tsx`, and scoped error handling
- server/client boundary decisions and serializable props
- preserving SWR, TanStack Query, Apollo, urql, Relay, or custom client data hooks
- Server Functions/actions, optimistic UI, mutations, and invalidation flows
- Next.js 15 `params` / `searchParams` promises
- Next.js 16 `cacheComponents` and static shell vs dynamic region decisions
- React `cache()` usage where dedupe is being confused with batching

## Core idea

> Design the route as a composition shell, push reads into the components that own them, and make the loading experience explicit.

In practice, that usually means:

1. Classify whether a route truly needs to block at the route level.
2. Keep `page.tsx` / `layout.tsx` synchronous when it mostly arranges regions, fallbacks, and errors.
3. Move server-renderable reads into async Server Component regions.
4. Use Suspense around meaningful independently-loading UI regions, not around every tiny component.
5. Push `'use client'` down to small leaves or wrappers.
6. Preserve intentional client-side cache/data tools for live, polling, optimistic, subscription, or browser-owned flows.
7. Treat React `cache()` as request-scope dedupe for identical calls, not batching for many different IDs.
8. Check the Next.js version before giving version-specific advice.

## Skill contents

```text
next-app-router-rsc-page-composition/
├── SKILL.md
└── references/
    ├── architecture-rules.md
    ├── boundaries-and-data.md
    ├── next-version-notes.md
    ├── output-contract.md
    └── transformations.md
```

`SKILL.md` is intentionally compact. The detailed guidance is split into focused reference files so the assistant can load only the material it needs.

## Output contract

When the skill is used for architecture review, it asks the assistant to answer with these sections:

1. Architecture Classification
2. Boundary Assessment
3. Data Ownership Map
4. Loading and Error Topology
5. Client Data/Cache Note
6. Next.js Version Notes
7. Recommended Changes
8. Validation Plan

This keeps reviews specific, auditable, and less likely to collapse into generic “just use Server Components” advice.

## Evaluation status

The first evaluation pass covered six scenarios:

- loader-style dashboard page to sync shell plus async regions
- Next.js 15 promise-like `params` / `searchParams`
- preserving intentional TanStack Query usage
- extracting a client leaf from an overgrown `'use client'` page
- React `cache()` dedupe vs batching for product cards
- Next.js 16 `cacheComponents` static shell with a dynamic personalized region

Iteration 1 benchmark summary:

- With skill: 100% pass rate
- Baseline: 93.3% pass rate
- Main gains: strict architecture output contract and explicit Next.js 15 `params.then(...)` guidance

Trigger optimization is intentionally deferred until testing against a real example project.

## Attribution

This skill operationalizes ideas from Aurora Scharff’s React Server Component architecture writing, especially:

- [Component Architecture for React Server Components](https://aurorascharff.no/posts/component-architecture-for-react-server-components/)
- [Server and Client Component Composition in Practice](https://aurorascharff.no/posts/server-and-client-component-composition-in-practice/)

The OpenCode skill package is authored separately by Tim Seriakov.

## Development notes

Generated eval workspaces, local package artifacts, and assistant runtime files are intentionally ignored by git. Package from a clean staging directory containing only `SKILL.md` and `references/` so the distributed `.skill` file does not include local tooling or evaluation artifacts.
