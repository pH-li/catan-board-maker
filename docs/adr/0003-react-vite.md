# 0003. Use React with Vite for the frontend

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

The frontend is an interactive design tool: a piece list, settings panels with many controls, a 3D piece viewer, a 3D board builder, an export dialog with progress, and project import/export. We need:

- a component model that matches this UI structure
- a mature 3D integration ([0004](0004-react-three-fiber.md))
- strong TypeScript support
- a fast development loop

## Decision

Use **React** with **TypeScript**, built and served in development by **Vite**.

- Function components and hooks only.
- Components focus on display. Logic lives in hooks, stores, or framework-free modules.
- Vite's dev server proxies `/api` to the local API, matching the nginx proxy used in the Docker setup.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Vue | Approachable, good tooling | Smaller 3D ecosystem (TresJS is younger than react-three-fiber) | 3D integration is central for us |
| Svelte | Very little boilerplate, fast | Smaller ecosystem; Threlte is less mature | Same reason; fewer component libraries |
| Next.js (React framework) | Routing, server rendering | We have no SEO or server-rendering needs; adds server concepts | Unnecessary complexity for a local single-page app |

## Consequences

**Positive**
- The largest ecosystem: react-three-fiber, shadcn/ui, TanStack Query, Zustand and testing tools.
- Highly transferable skills.
- Vite gives near-instant hot module reloading.

**Negative / trade-offs**
- React re-render behaviour has to be managed deliberately, especially near the 3D canvas (see [0011](0011-state-management.md)).

## Related

- [0004 3D rendering](0004-react-three-fiber.md), [0011 State management](0011-state-management.md), [0012 Styling](0012-tailwind-shadcn.md)
