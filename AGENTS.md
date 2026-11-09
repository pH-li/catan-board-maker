# AGENTS.md

Instructions for AI coding agents working in this repository. Humans: see [CONTRIBUTING.md](CONTRIBUTING.md).

## Project

Hex Set Studio is a local, clone-and-run app (Docker Compose) for designing a Catan-compatible board set and exporting each part as an STL for 3D printing. TypeScript pnpm monorepo:

```
apps/web           React + Vite + react-three-fiber UI
apps/api           Express 5 API (server.ts) + BullMQ worker (worker.ts)
packages/specs     component dimensions (mm) + counts: the single source of truth
packages/schemas   Zod schemas + inferred types + limits
packages/geometry  pure mesh builders (manifold-3d) + STL writer
docs/              ARCHITECTURE, CODING_STANDARDS, THREAT_MODEL, adr/
```

Status: Phase 0 (foundations). Some commands below arrive with the tooling phase. If a script doesn't exist yet, say so; don't invent one.

## Commands

```bash
pnpm install                                     # install (frozen lockfile in CI)
docker compose -f docker-compose.dev.yml up -d   # dev Redis on 127.0.0.1:6379
pnpm dev                                         # web :5173, api :3000, worker
pnpm lint                                        # ESLint + Prettier check (zero warnings)
pnpm typecheck                                   # tsc across workspaces
pnpm test                                        # unit + component tests
pnpm test:integration                            # needs dev Redis
pnpm test:e2e                                    # needs the full Docker stack
pnpm build
pnpm ci:local                                    # everything CI runs
docker compose up --build                        # full user stack on 127.0.0.1:8080
```

Scope commands to one package with `pnpm --filter <name> <script>`, for example `pnpm --filter @hex/geometry test`.

## Rules

Follow [docs/CODING_STANDARDS.md](docs/CODING_STANDARDS.md). The ones most often missed:

- TypeScript strict: no `any`, no unexplained `!`. Derive types from Zod schemas with `z.infer`.
- **Units are millimeters.** Every dimension comes from `packages/specs`, with no magic numbers in geometry.
- `packages/geometry` is pure: no DOM, React, `node:` modules, Redis or network.
- In `apps/api`, services depend on **ports**, never adapters. Only `LocalFileStore` touches the filesystem.
- Validate every input at the boundary with a Zod schema **that has limits**.
- Comments explain *why*. Public package exports get TSDoc.
- Tests sit next to the code. Geometry tests assert properties (bounding box ±0.05 mm, manifold, wall ≥0.8 mm), not exact triangles.

## Definition of done

- `pnpm lint`, `pnpm typecheck` and `pnpm test` pass (plus `pnpm test:integration` when touching `apps/api`).
- New behaviour has tests, and bug fixes include a regression test.
- Docs are updated in the same change: README, ARCHITECTURE, ADRs (new or superseding), THREAT_MODEL for new inputs, endpoints or services.
- Report honestly what you ran and what failed. Never claim checks passed without running them.

## Workflow

- Never commit to `main`. Use one branch per change: `feat/ fix/ docs/ refactor/ test/ chore/ ci/` + a kebab-case slug.
- Conventional Commits: `type(scope): imperative summary`. Scopes: `web`, `api`, `worker`, `geometry`, `schemas`, `specs`, `docker`, `adr`, `deps`.
- Keep commits small and focused. Don't push or open PRs unless asked.

## Boundaries: ask first or don't do it

- **Don't** change dimensions in `packages/specs` without updating their spec tests and citing a source.
- **Don't** add dependencies, services, or architectural patterns without an ADR (or explicit approval).
- **Don't** publish ports beyond `127.0.0.1`, mount the Docker socket, or run containers as root.
- **Don't** turn user-influenced values into file paths, shell commands or HTML. **Don't** use `dangerouslySetInnerHTML`.
- **Don't** commit secrets, `.env` files, `data/`, or generated build output.
- **Don't** weaken lint rules, TypeScript settings, coverage gates or CI checks to make something pass.
- **Don't** edit an accepted ADR's decision. Write a superseding ADR.

## Where to look

- Architecture and data flow: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- Why a choice was made: [docs/adr/](docs/adr/README.md)
- Security expectations: [docs/THREAT_MODEL.md](docs/THREAT_MODEL.md), [SECURITY.md](SECURITY.md)
- Human contribution process: [CONTRIBUTING.md](CONTRIBUTING.md)
