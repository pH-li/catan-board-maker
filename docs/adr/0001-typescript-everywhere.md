# 0001. Use TypeScript for the frontend, backend and shared geometry

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

A design in Hex Set Studio is a set of parameters (for example "settlement, gabled roof, engraved star, 1 mm deep"). **Geometry code** turns those parameters into a triangle mesh, and two places need that code:

1. **The live 3D preview in the browser.** It must update instantly while the user drags a slider.
2. **The STL export on the backend.** It must produce exactly the triangles the user previewed.

Browsers run JavaScript/TypeScript (and WebAssembly), not Python. Lily initially preferred Python for the backend, to practice writing APIs.

## Decision

Use **TypeScript everywhere**: the React frontend, the Express backend and a shared `packages/geometry` package. Both the preview and the export import the same geometry functions, so what users see is exactly what prints.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Python backend owns all geometry; the browser requests preview meshes from the API | One copy of geometry; Python practice | Each slider change becomes a network round trip (hundreds of ms, more on a cold start); doesn't work offline | Preview latency hurts the core experience |
| Geometry written twice (TS for preview, Python for export) | Instant preview; Python backend | Every feature built twice; the two copies drift, so the preview stops matching the print | Highest long-term maintenance cost; correctness risk |
| Python API + TypeScript geometry service | Python APIs, single geometry copy | Two languages and toolchains | More complexity than the learning benefit justifies |

## Consequences

**Positive**
- A **single source of truth** for geometry, so the preview always matches the export.
- One language, one toolchain, one set of lint and test rules.
- Types can be shared end to end (see [0007](0007-zod-shared-schemas.md)).

**Negative / trade-offs**
- No Python practice in this project. API design skills (routing, validation, error handling, queues) still transfer directly.
- Heavy geometry runs in Node, which is fine for our piece sizes thanks to manifold-3d's WebAssembly build.

**Follow-ups**
- `packages/geometry` must stay **pure**: no DOM, React, Node I/O or Redis. Then it runs identically in a browser Web Worker and in the Node worker.

## Related

- [0002 Monorepo](0002-pnpm-monorepo.md), [0005 Parametric geometry](0005-parametric-geometry.md)
