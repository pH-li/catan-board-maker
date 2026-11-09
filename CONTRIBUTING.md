# Contributing to Hex Set Studio

Thanks for helping! This guide covers how to set up a development environment, how work flows from idea to merged code, and the standards every change follows.

> **Just want to use the app?** You don't need any of this. See the [Quick start in the README](README.md#quick-start).

**Contents**
1. [Before you start](#1-before-you-start)
2. [Running locally for development](#2-running-locally-for-development)
3. [How work flows](#3-how-work-flows)
4. [Branches and commits](#4-branches-and-commits)
5. [Pull requests](#5-pull-requests)
6. [Testing requirements](#6-testing-requirements)
7. [Common tasks](#7-common-tasks)
8. [Troubleshooting](#8-troubleshooting)
9. [Maintainer setup](#9-maintainer-setup)

## 1. Before you start

Read these first. They're short:
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): how the system fits together
- [docs/CODING_STANDARDS.md](docs/CODING_STANDARDS.md): the rules for code
- [docs/adr/](docs/adr/README.md): why things are the way they are
- [docs/THREAT_MODEL.md](docs/THREAT_MODEL.md): what we defend against

Be kind. This project follows the [Code of Conduct](CODE_OF_CONDUCT.md). Found a security issue? **Don't open a public issue.** Follow [SECURITY.md](SECURITY.md).

## 2. Running locally for development

> 🚧 The project is in Phase 0 (foundations). Some commands below arrive with the tooling and backend phases. This guide describes the intended setup.

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Git | any recent | https://git-scm.com |
| Node.js | the LTS version in [`.nvmrc`](.nvmrc) | https://nodejs.org, or `nvm use` / `fnm use` |
| pnpm | pinned in `package.json` (`packageManager`) | `corepack enable` (bundled with Node), or `npm i -g pnpm` |
| Docker Desktop (or Docker Engine + Compose v2) | recent | https://docs.docker.com/get-docker/ |

On **Windows**, use Git Bash or WSL2 for the commands below, and make sure Docker Desktop is running.

### First-time setup

```bash
# 1. Clone and enter the repo
git clone https://github.com/pH-li/catan-board-maker.git
cd catan-board-maker

# 2. Install dependencies (also installs Git hooks)
pnpm install

# 3. Create your local environment file (the defaults work as-is)
cp .env.example .env

# 4. Start Redis for development (bound to 127.0.0.1 only)
docker compose -f docker-compose.dev.yml up -d
```

### Day-to-day

```bash
pnpm dev
```

This starts everything with hot reload:

| What | URL |
|---|---|
| Web app (Vite) | http://localhost:5173 |
| API | http://localhost:3000 |
| API docs (OpenAPI) | http://localhost:3000/api/docs |
| Worker | runs in the same terminal, reloads on save |

Vite proxies `/api` to the API, just like nginx does in the full Docker setup.

### Checks (run these before pushing)

| Command | What it does | Needs Docker? |
|---|---|---|
| `pnpm lint` | ESLint + Prettier check | No |
| `pnpm format` | Auto-format with Prettier | No |
| `pnpm typecheck` | TypeScript across all packages | No |
| `pnpm test` | Unit and component tests | No |
| `pnpm test:coverage` | Tests + coverage gates | No |
| `pnpm test:integration` | API + BullMQ + real Redis | **Yes** (dev Redis running) |
| `pnpm test:e2e` | Playwright against the full Docker stack | **Yes** |
| `pnpm build` | Build every package | No |
| `pnpm ci:local` | Everything CI runs, in order | **Yes** |

### Testing the user setup

Before changing anything Docker-related, check the full stack exactly as users run it:

```bash
docker compose up --build
# open http://localhost:8080
```

### Resetting

```bash
docker compose down -v                              # stop the full stack and delete its volumes
docker compose -f docker-compose.dev.yml down -v    # stop dev Redis and delete its data
rm -rf data/                                        # delete local uploads and exports
```

To reset the browser's saved project, clear site data for `localhost` in your browser's dev tools.

## 3. How work flows

```
issue  →  branch  →  commits  →  pull request  →  CI green + review  →  squash merge  →  delete branch
```

1. **Start from an issue.** Open one (bug or feature template) or comment on an existing one to say you're working on it. For anything big, discuss the approach in the issue first.
2. **Significant technical decisions need an ADR.** See [Making an architecture decision](#making-an-architecture-decision).
3. **Keep changes small.** One feature or fix per branch and PR. Large features are split into several PRs that each leave `main` working.

## 4. Branches and commits

We use **GitHub Flow** ([ADR 0016](docs/adr/0016-git-workflow-and-ci.md)). `main` is protected and always working.

### Branch names

`<type>/<short-kebab-description>`, cut from the latest `main`:

| Prefix | Use for | Example |
|---|---|---|
| `feat/` | New feature | `feat/hex-geometry` |
| `fix/` | Bug fix | `fix/chit-pip-spacing` |
| `docs/` | Documentation only | `docs/adr-queue-retention` |
| `refactor/` | Code change with no behaviour change | `refactor/file-store-paths` |
| `test/` | Tests only | `test/randomizer-properties` |
| `chore/` | Tooling, dependencies, config | `chore/bump-vitest` |
| `ci/` | CI pipeline | `ci/cache-docker-layers` |

```bash
git switch main && git pull
git switch -c feat/hex-geometry
```

### Commit messages: Conventional Commits

```
<type>(<optional scope>): <imperative summary, ≤72 chars>

<optional body: what and why, wrapped at 100>

<optional footer: Closes #12, BREAKING CHANGE: ...>
```

- **Types:** `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `ci`, `perf`, `build`, `style`
- **Scopes** (optional): `web`, `api`, `worker`, `geometry`, `schemas`, `specs`, `docker`, `adr`, `deps`

```
feat(geometry): add gabled roof preset for settlements
fix(api): reject jobIds that are not UUIDs
docs(adr): record export retention decision
```

`commitlint` checks your messages in a Git hook. Small, focused commits make review easier; the PR is squashed on merge anyway.

## 5. Pull requests

- **Title:** a Conventional Commit, because it becomes the squash commit on `main`.
- **Description:** fill in the PR template: what and why, how you tested it, screenshots or recordings for UI changes, and the security checklist.
- **Link the issue** (`Closes #12`).
- **Size:** aim for under about 400 changed lines, excluding generated files and lockfiles. Split it up if it's bigger.

### Definition of done

- [ ] CI is green: lint, typecheck, unit, integration, coverage, build, Docker, security scans
- [ ] Tests cover the change, and every bug fix has a test that fails without the fix
- [ ] Docs are updated (README, ARCHITECTURE, ADRs, threat model) where behaviour or decisions changed
- [ ] No new warnings, `any`, or unexplained lint disables
- [ ] The security checklist in the PR template is answered

### Reviews

- Reviewers check **correctness, clarity, tests and security** first, style last (Prettier handles style).
- Comment on the code, not the person. Use "nit:" for optional suggestions.
- Authors respond to every comment, by changing the code or explaining. The author resolves threads once they're addressed.
- Merge with **Squash and merge**, then delete the branch.

## 6. Testing requirements

Full strategy in [ADR 0015](docs/adr/0015-testing-strategy.md).

- **Unit tests** for all logic, next to the code: `thing.ts` → `thing.test.ts`.
- **Integration tests** (`*.int.test.ts`) for API routes and adapters, against real Redis.
- **Geometry:** assert properties (bounding box within ±0.05 mm of spec, manifold, wall ≥0.8 mm, volume), never exact triangle lists.
- **Components:** React Testing Library, querying by role and label as users would.
- **Coverage gates:** 80% for `packages/*` and API services, 70% for `apps/web`. CI fails below these.
- Tests must be **deterministic**: seed randomness, fake timers, no network.

## 7. Common tasks

### Adding a new component (piece type)

1. **Spec:** add its dimensions (mm) and counts to `packages/specs`, citing the source of the measurements.
2. **Schema:** add its design parameters, with limits, to `packages/schemas`.
3. **Geometry:** add `build<Piece>(params, detail)` to `packages/geometry`, using only spec values.
4. **Tests:** bounding box vs spec, manifold, wall thickness, determinism, preview vs full detail.
5. **UI:** register it in the designer's piece list and settings panel.
6. **Export:** include it in the manifest counts.
7. **Docs:** update the component table in the README if it's user-visible.

### Adding an API endpoint

1. Request and response schemas in `packages/schemas`, with limits.
2. Route + `validate(schema)` → controller → service. Use ports for any I/O.
3. Register it in the OpenAPI registry.
4. Unit tests for the service, plus an integration test for the route, covering invalid input, a foreign `Origin` and the error shape.
5. Update the API table in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and review the [threat model](docs/THREAT_MODEL.md).

### Adding a dependency

Explain in the PR why it's needed, check that it's maintained, and check whether it runs install scripts (pnpm blocks them unless allow-listed). Architecture-shaping dependencies need an ADR.

### Making an architecture decision

1. Copy [docs/adr/0000-template.md](docs/adr/0000-template.md) to the next number.
2. Open a `docs/adr-<topic>` PR with status **Proposed**.
3. Discuss in the PR. On merge, set it to **Accepted** and add it to the [ADR index](docs/adr/README.md).

Never rewrite an accepted ADR. Supersede it with a new one.

## 8. Troubleshooting

| Problem | Fix |
|---|---|
| `pnpm: command not found` | `corepack enable`, then open a new terminal (or `npm i -g pnpm`) |
| `Cannot connect to the Docker daemon` | Start Docker Desktop and wait until it says "running" |
| Port 5173, 3000, 6379 or 8080 already in use | Stop the other program, or change the port in `.env` |
| API can't reach Redis | `docker compose -f docker-compose.dev.yml ps` shows whether it's running; check `REDIS_URL` in `.env` |
| Every line shows as changed on Windows | `.gitattributes` forces LF. Run `git add --renormalize .` once, and set your editor to LF |
| Hooks didn't run on commit | Re-run `pnpm install` (it installs Husky hooks); check that `git config core.hooksPath` is set |
| Slow file watching on Windows + WSL | Keep the repo inside the WSL filesystem (`~/…`), not `/mnt/c/…` |
| Stale build or type errors after switching branches | `pnpm install && pnpm -r build`, then restart `pnpm dev` |

Still stuck? Open an issue with your OS, Node, pnpm and Docker versions and the full error.

## 9. Maintainer setup

One-time GitHub repository settings that enforce the rules above:

- **Branch protection on `main`:**
  - require a pull request before merging
  - require the CI status checks (all jobs plus CodeQL) to pass and branches to be up to date
  - block force pushes and deletion
- **Merge options:** allow squash merging only; automatically delete head branches.
- **Security:** enable Dependabot alerts and security updates, secret scanning with **push protection**, and **private vulnerability reporting**.
- **Actions:** default workflow permissions set to **read-only**.
