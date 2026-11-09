# 0015. Follow the test pyramid, with geometry tested by properties

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

The things most likely to go wrong, and how much each costs:

- **Wrong dimensions:** wasted filament and pieces that don't fit. These are the most expensive bugs.
- **Non-watertight meshes:** slicer errors.
- **Broken API or queue behaviour:** exports that never finish.
- **Security regressions:** unvalidated input, path traversal, unsafe imports.
- **UI regressions.**

We want fast feedback on every change and confidence that the user-facing flows work, without a slow, flaky test suite.

## Decision

Follow the **test pyramid**: many fast unit tests, fewer integration tests, and a handful of end-to-end tests.

| Layer | Tool | What |
|---|---|---|
| Unit | Vitest | Geometry builders, schemas, services (with in-memory fakes), board logic, security middleware |
| Property-based | fast-check | Randomizer invariants; schema limits |
| Component | Vitest + React Testing Library | UI behaviour from the user's perspective (roles, labels), not internals |
| Integration | Vitest + Supertest + real Redis | HTTP → Express → BullMQ → worker → `data/` (temp dir) |
| End-to-end | Playwright against `docker compose up` | Design → export → download; board → Export All → zip |
| Containers | Trivy + compose smoke test | Images scan clean; `/api/health` returns 200 |

**Geometry is tested by properties, not snapshots.** Comparing exact triangle lists breaks on every harmless change. We assert what matters to a print instead:
- bounding box within **±0.05 mm** of the spec
- manifold (watertight)
- minimum wall thickness ≥ 0.8 mm
- volume within a tolerance
- determinism (same parameters give the same output)

**Coverage gates enforced in CI:**
- **80%** for `packages/*` and API services
- **70%** for `apps/web`

**Deliberately skipped:** pixel screenshot tests of the WebGL viewer. GPU and driver differences make them flaky. We test the meshes, not the pixels.

**Conventions:**
- tests sit next to the code as `*.test.ts(x)`
- integration tests are named `*.int.test.ts`
- E2E tests live in `apps/web/e2e/`

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Mostly E2E tests | Realistic | Slow, flaky, hard to diagnose | Inverted pyramid |
| Mesh snapshot tests | Easy to write | Brittle; failures don't mean anything is wrong | Properties are what matter |
| Jest | Popular | Slower; awkward with ESM and Vite | Vitest shares Vite's config |

## Consequences

**Positive**
- Most tests run in seconds.
- Geometry tests fail only when a print would actually be wrong.
- Real-Redis integration tests prove the adapters behave like the fakes.

**Negative / trade-offs**
- Coverage gates can encourage shallow tests. Reviewers check the quality of tests, not just the percentage.

## Related

- [0005 Parametric geometry](0005-parametric-geometry.md), [0014 Ports and adapters](0014-ports-and-adapters.md), [0016 Git workflow and CI](0016-git-workflow-and-ci.md)
