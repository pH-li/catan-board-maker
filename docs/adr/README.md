# Architecture Decision Records

An **ADR** is a short document that records one significant technical decision: the context, the choice, the alternatives we rejected, and the consequences. ADRs explain *why* the code looks the way it does, so future contributors (including future us) don't have to guess or re-argue settled questions.

## Index

| # | Decision | Status |
|---|---|---|
| [0001](0001-typescript-everywhere.md) | TypeScript for frontend, backend and shared geometry | Accepted |
| [0002](0002-pnpm-monorepo.md) | pnpm workspaces monorepo | Accepted |
| [0003](0003-react-vite.md) | React + Vite frontend | Accepted |
| [0004](0004-react-three-fiber.md) | react-three-fiber + drei for 3D (including a 3D board) | Accepted |
| [0005](0005-parametric-geometry.md) | Parametric models in mm with manifold-3d | Accepted |
| [0006](0006-express-api.md) | Express 5 API with layered structure | Accepted |
| [0007](0007-zod-shared-schemas.md) | Shared Zod schemas + generated OpenAPI | Accepted |
| [0008](0008-bullmq-job-queue.md) | Async exports with BullMQ + Redis | Accepted |
| [0009](0009-local-filesystem-storage.md) | Local filesystem storage, no database | Accepted |
| [0010](0010-no-accounts.md) | No accounts or authentication | Accepted |
| [0011](0011-state-management.md) | Zustand + TanStack Query + useState | Accepted |
| [0012](0012-tailwind-shadcn.md) | Tailwind CSS + shadcn/ui | Accepted |
| [0013](0013-local-docker-distribution.md) | Local Docker Compose distribution; no cloud deploy | Accepted |
| [0014](0014-ports-and-adapters.md) | Ports and adapters for infrastructure | Accepted |
| [0015](0015-testing-strategy.md) | Test pyramid; geometry tested by properties | Accepted |
| [0016](0016-git-workflow-and-ci.md) | GitHub Flow, Conventional Commits, required CI | Accepted |
| [0017](0017-security-baseline.md) | Security baseline for a local app | Accepted |

## Writing a new ADR

1. Copy [0000-template.md](0000-template.md) to `NNNN-short-kebab-title.md` using the next free number.
2. Set the status to **Proposed** and open a PR (branch `docs/adr-<topic>`). Discussion happens in the PR.
3. On merge, set the status to **Accepted** and add a row to the index above.
4. **Never rewrite an accepted decision.** Write a new ADR that supersedes it, and change the old one's status to `Superseded by NNNN`.

Write an ADR when a change:
- adds a dependency that shapes the architecture, or a new service
- changes a boundary between packages, or a public API
- changes the security posture
- would make a reviewer ask "why did we do it this way?"
