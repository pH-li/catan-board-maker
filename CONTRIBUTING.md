# Contributing to Hex Set Studio

Thanks for helping! This guide covers the development setup, how work flows from idea to merged code, and the standards every change follows.

> **Just want to use the app?** You don't need any of this. See the [Quick start in the README](README.md#quick-start).

**Contents**
1. [Before you start](#1-before-you-start)
2. [Running locally for development](#2-running-locally-for-development)
3. [How work flows](#3-how-work-flows)
4. [Branches and commits](#4-branches-and-commits)
5. [Pull requests](#5-pull-requests)
6. [Coding standards](#6-coding-standards)
7. [Testing requirements](#7-testing-requirements)
8. [Common tasks](#8-common-tasks)
9. [Troubleshooting](#9-troubleshooting)
10. [Maintainer setup](#10-maintainer-setup)

## 1. Before you start

- Read [ARCHITECTURE.md](ARCHITECTURE.md). It covers how the system fits together, the **design decisions** and why we made them, and the **security** model.
- **Be respectful.** Assume good intent, keep feedback about the code rather than the person, and help newcomers.
- **Found a security issue?** Don't open a public issue. Report it privately (see [ARCHITECTURE.md → Reporting a vulnerability](ARCHITECTURE.md#reporting-a-vulnerability)).

## 2. Running locally for development

> 🚧 The project is in Phase 0 (foundations). Some commands below arrive with the tooling and backend phases. This guide describes the intended setup.

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Git | any recent | https://git-scm.com |
| Node.js | the LTS version in `.nvmrc` | https://nodejs.org, or `nvm use` / `fnm use` |
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
issue  →  branch  →  commits  →  pull request  →  CI green + review  →  squash merge  →  branch auto-deleted
```

1. **Start from an issue.** Open one or comment on an existing one to say you're working on it. For anything big, discuss the approach in the issue first.
2. **Changing a design decision?** Update the [Design decisions](ARCHITECTURE.md#design-decisions) section of ARCHITECTURE.md in the same PR, and explain the reasoning in the PR description.
3. **Keep changes small.** One feature or fix per branch and PR. Large features are split into several PRs that each leave `main` working.

## 4. Branches and commits

We use **GitHub Flow**. `main` is protected and always working.

### Branch names

`<type>/<short-kebab-description>`, cut from the latest `main`:

| Prefix | Use for | Example |
|---|---|---|
| `feat/` | New feature | `feat/hex-geometry` |
| `fix/` | Bug fix | `fix/chit-pip-spacing` |
| `docs/` | Documentation only | `docs/export-flow-diagram` |
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
- **Scopes** (optional): `web`, `api`, `worker`, `geometry`, `schemas`, `specs`, `docker`, `deps`

```
feat(geometry): add gabled roof preset for settlements
fix(api): reject jobIds that are not UUIDs
docs: add export flow diagram
```

`commitlint` checks your messages in a Git hook.

## 5. Pull requests

- **Title:** a Conventional Commit. It becomes the squash commit on `main`, and the description becomes its body.
- **Description:** what and why, how you tested it, screenshots for UI changes, and the security questions below.
- **Link the issue** (`Closes #12`).
- **Size:** aim for under about 400 changed lines, excluding lockfiles and generated files.
- **Merging:** squash merge is the only option. Branches are deleted automatically.

### Definition of done

- [ ] CI is green: lint, typecheck, unit, integration, coverage, build, Docker, security scans
- [ ] Tests cover the change, and every bug fix has a test that fails without the fix
- [ ] README / ARCHITECTURE are updated where behaviour or decisions changed
- [ ] No new warnings, `any`, or unexplained lint disables

### Security questions (answer in the PR)

- Does this add or change an input? Is it validated with a Zod schema **with limits**?
- Does any user-influenced value reach a file path, shell, query or HTML? (It shouldn't.)
- Does it add a port or service? Is it bound to `127.0.0.1` or internal only?
- Does it add a dependency? Is it maintained and needed, and does it run install scripts?
- Does [ARCHITECTURE.md → Security](ARCHITECTURE.md#security) need updating?

### Reviews

- Reviewers check **correctness, clarity, tests and security** first, style last (Prettier handles style).
- Use "nit:" for optional suggestions. Authors respond to every comment and resolve threads once they're addressed.

## 6. Coding standards

Most of these are enforced by TypeScript, ESLint, Prettier and CI. The rest are checked in review. If you need an exception, explain it in a comment and in the PR.

### TypeScript
- `strict` plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` and `noImplicitOverride`.
- **No `any`.** Use `unknown` and narrow it, usually with a Zod schema. **No `!`** without a comment explaining why.
- **Derive types from schemas** (`z.infer<typeof X>`). Never hand-write a duplicate type.
- Discriminated unions for states (`{ status: 'completed', … } | { status: 'failed', error }`). `as const` objects instead of `enum`.
- ES modules, with the `node:` prefix for built-ins.

### Naming and files

| Thing | Convention | Example |
|---|---|---|
| Variables, functions | `camelCase` | `buildSettlement`, `jobId` |
| Types, components, classes | `PascalCase` | `ExportRequest`, `PieceViewer` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_UPLOAD_BYTES` |
| Files | `kebab-case.ts`; components `PascalCase.tsx` | `file-store.ts`, `PieceViewer.tsx` |
| Booleans | `is`/`has`/`can` prefix | `isManifold` |

Name things after the domain (`hex`, `chit`, `port`, `settlement`). Avoid abbreviations except `id`, `url` and `mm`.

### Units and dimensions
- **Millimeters only.** Convert only at the edges (for example, a UI that displays inches).
- **Every physical dimension comes from `packages/specs`.** No size literals in geometry code (`no-magic-numbers` is on for `packages/geometry`).
- Names carry units when unclear: `wallThicknessMm`, `angleRad`.

### Package boundaries
- `packages/geometry` is **pure**: no DOM, React, `node:` modules, Redis or network.
- Apps import packages. Packages never import apps.
- In `apps/api`, **services depend on ports, never adapters.** Only `LocalFileStore` touches the filesystem.

### React
- Function components and hooks only. Components handle display; logic lives in hooks, stores or plain modules.
- Read Zustand state through **narrow selectors**, never the whole store.
- Keep per-frame 3D work out of React state (use refs and `useFrame`).
- Accessibility: use shadcn/Radix components, label every input, support the keyboard.
- `dangerouslySetInnerHTML` is **banned**.

### API
- routes → controllers → services → ports. Controllers have no business logic; services never touch `req`/`res`.
- Every body, parameter and query goes through `validate(schema)`.
- Throw typed errors (`NotFoundError`, `LimitExceededError`). The central handler maps them to status codes. **Never send stack traces** to clients.
- Configuration comes only from `config.ts` (validated with Zod). Log structured JSON with `requestId`/`jobId`, and **never log user content**.

### Security
- Every new input gets a Zod schema **with limits**.
- User-influenced values never become file paths, shell arguments or HTML.
- New ports bind to `127.0.0.1` or stay internal. No secrets in code, tests or logs (`.env` is gitignored).

### Comments
- Explain **why**, not what. Public exports from `packages/*` get TSDoc, including units.
- Point to the reasoning when it isn't obvious: `// See ARCHITECTURE.md → Design decisions (ports and adapters)`.

### Formatting, linting, Docker
- Prettier owns formatting (`pnpm format`). ESLint must pass with **zero warnings**.
- Disable a rule only with a reason: `// eslint-disable-next-line rule -- reason`.
- Docker: base images pinned by digest, multi-stage builds, **non-root**, never mount the Docker socket, never publish beyond `127.0.0.1`.

## 7. Testing requirements

See [ARCHITECTURE.md → Testing strategy](ARCHITECTURE.md#testing-strategy) for the full picture.

- **Unit tests** sit next to the code: `thing.ts` → `thing.test.ts`. **Integration tests** are named `*.int.test.ts`.
- **Geometry:** assert properties (bounding box ±0.05 mm of spec, manifold, wall ≥0.8 mm, volume), never exact triangle lists.
- **Components:** React Testing Library, querying by role and label.
- **Coverage gates:** 80% for `packages/*` and API services, 70% for `apps/web`.
- Tests must be **deterministic**: seed randomness, fake timers, no network.

## 8. Common tasks

### Adding a new component (piece type)
1. **Spec:** add its dimensions (mm) and counts to `packages/specs`, citing the source.
2. **Schema:** add its design parameters, with limits, to `packages/schemas`.
3. **Geometry:** add `build<Piece>(params, detail)` using only spec values.
4. **Tests:** bounding box vs spec, manifold, wall thickness, determinism, preview vs full.
5. **UI:** register it in the designer's piece list and settings panel.
6. **Export:** include it in the manifest counts.
7. **Docs:** update the component table in ARCHITECTURE.md.

### Adding an API endpoint
1. Request and response schemas, with limits, in `packages/schemas`.
2. Route + `validate(schema)` → controller → service, using ports for I/O.
3. Register it in the OpenAPI registry.
4. Tests: service unit tests, plus a route integration test covering invalid input, a foreign `Origin` and the error shape.
5. Update the API table and Security section in ARCHITECTURE.md.

### Adding a dependency
Explain in the PR why it's needed, check that it's maintained, and check whether it runs install scripts (pnpm blocks them unless allow-listed). If it shapes the architecture, add it to [Design decisions](ARCHITECTURE.md#design-decisions).

### Adding images to the docs
Put them in `docs/images/` with descriptive kebab-case names (`export-dialog.png`) and reference them from ARCHITECTURE.md or the README. Prefer Mermaid for diagrams so they stay editable.

## 9. Troubleshooting

| Problem | Fix |
|---|---|
| `pnpm: command not found` | `corepack enable`, then open a new terminal (or `npm i -g pnpm`) |
| A newly installed tool "is not recognized" | Restart the terminal or VS Code so it reloads `PATH` |
| `Cannot connect to the Docker daemon` | Start Docker Desktop and wait until it says "running" |
| Port 5173, 3000, 6379 or 8080 already in use | Stop the other program, or change the port in `.env` |
| API can't reach Redis | `docker compose -f docker-compose.dev.yml ps` shows whether it's running; check `REDIS_URL` in `.env` |
| Every line shows as changed on Windows | `.gitattributes` forces LF. Run `git add --renormalize .` once, and set your editor to LF |
| Hooks didn't run on commit | Re-run `pnpm install` (it installs Husky hooks) |
| Slow file watching on Windows + WSL | Keep the repo inside the WSL filesystem (`~/…`), not `/mnt/c/…` |
| Stale build or type errors after switching branches | `pnpm install && pnpm -r build`, then restart `pnpm dev` |

Still stuck? Open an issue with your OS, Node, pnpm and Docker versions and the full error.

## 10. Maintainer setup

One-time GitHub repository settings:

- **Merging:** squash merge only (PR title as the commit title, description as the body); delete head branches automatically. ✅ Already configured.
- **Branch protection on `main`:** require a PR, require the CI status checks to pass with the branch up to date, block force pushes and deletion. Set this up once the CI workflow exists.
- **Security:** Dependabot alerts and security updates, secret scanning with **push protection**, and **private vulnerability reporting**.
- **Actions:** default workflow permissions set to **read-only**.
