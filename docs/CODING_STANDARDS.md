# Coding Standards

These rules keep the codebase consistent, safe and easy to change. Most are **enforced by tooling** (TypeScript, ESLint, Prettier, Vitest coverage, CI). The rest are checked in review. When a rule and a good reason conflict, explain the exception in a comment and in the PR.

## 1. TypeScript

- **`strict: true`** plus `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` and `noImplicitOverride`.
- **No `any`.** Use `unknown` and narrow it, usually by parsing with a Zod schema.
- **No non-null assertions (`!`)** without a comment explaining why the value can't be null.
- Prefer `type` aliases for data shapes and `interface` for ports (contracts that classes implement).
- **Derive types from schemas** (`z.infer<typeof X>`). Never hand-write a type that duplicates a schema.
- Use discriminated unions for state machines (`{ status: 'completed', downloadUrl } | { status: 'failed', error }`), not optional fields.
- Use `as const` objects or string-literal unions instead of `enum`.
- ES modules everywhere. Use the `node:` prefix for Node built-ins (`node:fs/promises`).

## 2. Naming and files

| Thing | Convention | Example |
|---|---|---|
| Variables, functions | `camelCase` | `buildSettlement`, `jobId` |
| Types, interfaces, React components, classes | `PascalCase` | `ExportRequest`, `PieceViewer` |
| True constants | `SCREAMING_SNAKE_CASE` | `MAX_UPLOAD_BYTES` |
| Files | `kebab-case.ts`; React components `PascalCase.tsx` | `file-store.ts`, `PieceViewer.tsx` |
| Tests | next to the code | `file-store.test.ts`, `exports.int.test.ts` |
| Booleans | `is`/`has`/`can` prefix | `isManifold`, `hasImage` |

- One React component per file. One port per file.
- Name things after the domain: `chit`, `hex`, `port` (harbor), `settlement`, not `item1` or `obj`.
- No abbreviations except widely known ones (`id`, `url`, `mm`).

## 3. Units and dimensions

- **Millimeters are the only unit** in specs, geometry, schemas and exports. Convert only at the edges (for example, a UI that displays inches).
- **Every physical dimension comes from `packages/specs`.** There are no numeric literals for sizes in geometry code. ESLint's `no-magic-numbers` is enabled for `packages/geometry`, with small exceptions (0, 1, 2, -1).
- Names carry units when it isn't obvious: `wallThicknessMm`, `angleRad`.

## 4. Package boundaries

- `packages/geometry` is **pure**: no DOM, React, `node:` modules, Redis or network. Same input, same output.
- `packages/schemas` is **declarative**: schemas, types and limits only.
- Apps import packages. Packages never import apps.
- In `apps/api`, **services depend on ports, never adapters.** Only `server.ts`/`worker.ts` (the composition roots) build adapters. Enforced with ESLint import restrictions.
- Only `LocalFileStore` touches the filesystem.

## 5. React

- Function components and hooks only.
- **Components handle display.** Move logic to hooks (`use-*.ts`), stores, or plain modules that can be tested without React.
- Read Zustand state through **narrow selectors** (`useProject(s => s.pieces.settlement.roof)`), never the whole store.
- Keep per-frame 3D work out of React state. Use refs and `useFrame`.
- Accessibility: use shadcn/Radix components, label every input, make everything reachable by keyboard, and write tests that query by role and label.
- `dangerouslySetInnerHTML` is **banned** (enforced by a lint rule).

## 6. API (Express)

- **Layering:** routes → controllers → services → ports. Controllers never contain business logic. Services never touch `req`/`res`.
- **Validate at the boundary:** every route body, parameter and query goes through `validate(schema)` using `packages/schemas`. Handlers receive typed, validated data only.
- **Errors:**
  - throw typed domain errors (`NotFoundError`, `ValidationError`, `LimitExceededError`)
  - the central error handler maps them to HTTP status codes
  - unknown errors become `500` with a generic message and a `requestId`
  - **never send stack traces or internal paths to the client**
- Use expected-failure results (`{ ok: false, reason }`) for validation outcomes inside geometry (for example a too-thin wall). Throw only for exceptional situations.
- Configuration comes only from `config.ts`, which parses environment variables with Zod at startup.
- Log structured JSON with `requestId`/`jobId`. **Never log user content** (design text, file contents).

## 7. Security rules

These come from the [threat model](THREAT_MODEL.md) and [ADR 0017](adr/0017-security-baseline.md):

- Every new input gets a Zod schema **with limits** (lengths, ranges, sizes).
- User-influenced values never become file paths, shell arguments or HTML.
- New ports bind to `127.0.0.1` or stay on the internal Docker network.
- No secrets in code, tests, fixtures or logs. `.env` is gitignored; `.env.example` holds safe defaults.
- New dependencies need a reason in the PR. Prefer well-maintained packages with no install scripts.

## 8. Comments and documentation

- **Comments explain *why*, not *what*.** The code says what.
  ```ts
  // Bad:  increment i
  // Good: BullMQ retries give at-least-once delivery, so this may run twice; skip if already completed.
  ```
- Public exports from `packages/*` get **TSDoc** (`/** ... */`), including units and invariants.
- Link to an ADR when code exists because of a decision: `// See ADR 0014`.
- Update docs in the same PR as the behaviour change.

## 9. Testing

- Follow [ADR 0015](adr/0015-testing-strategy.md): unit tests first, integration tests for adapters and HTTP, and E2E for key flows only.
- Test **behaviour, not implementation**. Assert outputs and effects, not private calls.
- Geometry tests assert **properties** (bounding box ±0.05 mm, manifold, wall thickness, volume), not exact triangles.
- Every bug fix comes with a test that fails without the fix.
- Coverage gates: **80%** for `packages/*` and API services, **70%** for `apps/web`.
- Tests must be deterministic: seed randomness, use fake timers, no network.

## 10. Formatting and linting

- **Prettier** owns formatting. Don't argue about style in reviews; run `pnpm format`.
- **ESLint** (typescript-eslint `strict-type-checked`, react-hooks, import order) must pass with **zero warnings** in CI.
- Don't disable a lint rule without an inline reason: `// eslint-disable-next-line rule-name -- reason`.

## 11. Docker

- Base images pinned by digest. Multi-stage builds. Runtime images contain production dependencies only.
- Run as a **non-root** user. Read-only root filesystem where possible; only `data/` writable.
- Never mount the Docker socket. Never publish ports beyond `127.0.0.1`.

## 12. Git

See [CONTRIBUTING.md](../CONTRIBUTING.md) and [ADR 0016](adr/0016-git-workflow-and-ci.md): one branch per change, Conventional Commits, small PRs, squash merge.
