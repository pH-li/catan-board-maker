# 0002. Use a pnpm workspaces monorepo

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Following [0001](0001-typescript-everywhere.md), several packages depend on each other:

```
apps/web  ──► packages/geometry ──► packages/schemas ──► packages/specs
apps/api  ──► packages/geometry, packages/schemas
```

We need to:
1. **Link** local packages without publishing them to npm.
2. **Run tasks** (build, test, lint) across every package.
3. Keep dependencies **honest**: every package declares exactly what it uses.

## Decision

Use **pnpm workspaces** in a single repository:

```
apps/       web, api
packages/   specs, schemas, geometry
```

- Internal packages are referenced with the `workspace:*` protocol and scoped names (for example `@hex/geometry`).
- The root `package.json` holds the scripts that fan out across workspaces (`pnpm -r build`, `pnpm -r test`).
- The lockfile is committed, and CI installs with `--frozen-lockfile`.
- **No Turborepo or Nx for now.** With two apps and three packages, builds take seconds.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| npm workspaces | Built in, no extra tool | Slower installs; allows **phantom dependencies** (using a package you never declared because a sibling hoisted it) | pnpm's strictness catches real bugs |
| pnpm + Turborepo | Task caching and ordered pipelines | Another config and concept to learn | YAGNI: no build-speed problem to solve yet |
| Nx | Very capable: generators, dependency graph, caching | Heavy; steep learning curve | Overkill at this size |
| Separate repositories | Independent versioning | Cross-repo changes become slow and painful; shared code needs publishing | We change geometry, schemas and apps together constantly |

## Consequences

**Positive**
- One PR can change the geometry, the API and the UI together, and CI checks them all at once.
- The strict `node_modules` layout means a package can only import what it declares.
- pnpm blocks dependency install scripts by default (we allow-list the ones we need), which reduces supply-chain risk.

**Negative / trade-offs**
- Contributors need pnpm installed (`corepack enable` or `npm i -g pnpm`).
- Some tools need pnpm-aware configuration.

**Follow-ups**
- Revisit Turborepo if CI regularly takes more than about 10 minutes, or local builds get noticeably slow.

## Related

- [0016 Git workflow and CI](0016-git-workflow-and-ci.md)
