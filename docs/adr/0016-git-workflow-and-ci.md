# 0016. Use GitHub Flow, Conventional Commits and required CI checks

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Code quality is a primary goal. Standards only matter if they're **enforced automatically**, and history is only useful if it's **readable**. We need a branching model, a commit convention and a CI pipeline that blocks bad changes before they reach `main`.

## Decision

### Branching: GitHub Flow
- `main` is always working and **protected**: no direct pushes, and a PR with all required checks green is needed to merge.
- **One short-lived branch per feature or fix**, cut from the latest `main`:
  - `feat/`: new feature
  - `fix/`: bug fix
  - `docs/`: documentation
  - `refactor/`: code change with no behaviour change
  - `test/`: tests only
  - `chore/`: tooling, dependencies
  - `ci/`: pipeline changes
  - then a kebab-case slug, for example `feat/hex-geometry`
- Flow: small commits → PR → CI green → review → **squash merge** → delete the branch.
- **Releases** are SemVer tags on `main` (`v1.0.0`).

### Commits: Conventional Commits
```
<type>(<optional scope>): <imperative summary>

feat(geometry): add gabled roof preset for settlements
fix(api): reject jobIds that are not UUIDs
docs(adr): record BullMQ decision
```
Types match the branch prefixes (plus `perf`, `build`, `style`). `commitlint` enforces this in a Git hook. Squash-merge titles must follow it too, because they become the `main` history and the source for the changelog.

### CI: GitHub Actions
Runs on every PR and every push to `main`. Outdated runs are cancelled.

| Job | Purpose |
|---|---|
| install | `pnpm install --frozen-lockfile`, with a cached store |
| lint | ESLint + Prettier check |
| typecheck | `tsc --noEmit` across workspaces |
| unit | Vitest + coverage gates |
| integration | Vitest + Supertest with a Redis service container |
| security | `pnpm audit`, gitleaks |
| build | build every package |
| docker | build the images, Trivy scan, compose smoke test (`/api/health`) |
| CodeQL | separate workflow, static security analysis |

- The jobs after install run **in parallel** and report as **separate checks**, so failures are obvious.
- Every job is a **required status check** for merging.
- Workflows use `permissions: contents: read`. Actions are pinned to commit SHAs, and Dependabot updates them. There are no secrets in CI.
- `pnpm ci:local` mirrors the pipeline locally. The pre-commit hook runs only fast checks on staged files (lint-staged, gitleaks).

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Git Flow (`develop`, `release/*`) | Suits scheduled, versioned releases | Long-lived branches drift; overhead | Too heavy for this project |
| Trunk-based, direct commits to `main` | Fastest | No review gate | We want PR review and CI gates |
| Merge commits / rebase merges | Keeps branch history | Noisy `main` history | Squash gives one meaningful commit per change |
| Free-form commit messages | No friction | Unreadable history; no automated changelog | Conventional Commits pay off quickly |

## Consequences

**Positive**
- `main` is always releasable, and history reads like a changelog.
- Quality rules are enforced by machines, not memory.

**Negative / trade-offs**
- Some friction for tiny changes, which still need a branch and a PR.
- CI time grows as tests grow. Revisit caching and Turborepo if it gets slow ([0002](0002-pnpm-monorepo.md)).

## Related

- [0002 Monorepo](0002-pnpm-monorepo.md), [0015 Testing](0015-testing-strategy.md), [0017 Security](0017-security-baseline.md), [CONTRIBUTING.md](../../CONTRIBUTING.md)
