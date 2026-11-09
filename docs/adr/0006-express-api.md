# 0006. Use Express 5 for the API

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

The backend exposes a small HTTP API: health, uploads, creating export jobs, job status and downloads. We want a framework that handles routing, request parsing, middleware and errors, has strong TypeScript support, and builds skills that transfer to other jobs and codebases.

The API originally targeted AWS Lambda. After [0013](0013-local-docker-distribution.md) it runs as a normal Node server inside Docker.

## Decision

Use **Express 5**, structured in layers:

```
routes/ → controllers/ → services/ → ports/ (interfaces) ← adapters/ (BullMQ, filesystem)
```

- **Routes:** URL plus method, mapped to a controller, plus route-level middleware such as validation.
- **Controllers:** HTTP only. They read the validated input, call a service, and shape the response.
- **Services:** business logic. They never touch `req`/`res`, Redis or `fs` directly (see [0014](0014-ports-and-adapters.md)).
- `app.ts` **builds** the app; `server.ts` **starts** it; `worker.ts` starts the job worker. Tests import `app.ts` directly.
- Express 5 forwards rejected promises from async handlers to the error middleware. A **central error handler** maps typed errors to HTTP responses.
- Hardening: `helmet`, `x-powered-by` disabled, `express.json({ limit })`.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Hono | Tiny, modern, first-class TypeScript, serverless-friendly | Fewer tutorials and job listings | Lily prioritized Express's ubiquity |
| Fastify | Fast, schema-first | Plugin model to learn | Express is more widely known |
| NestJS | Enterprise structure, dependency injection | Heavy, boilerplate, steep learning curve | Overkill for five endpoints |
| tRPC | End-to-end types with no API layer | Not a REST API; less API-design practice | Conflicts with the learning goal |

## Consequences

**Positive**
- The most widely recognized Node framework, with plenty of examples, middleware and answers online.
- Layering keeps the business logic testable without HTTP.

**Negative / trade-offs**
- Express's own types are looser than Hono's. We compensate with Zod-validated input ([0007](0007-zod-shared-schemas.md)).

## Related

- [0007 Zod schemas](0007-zod-shared-schemas.md), [0014 Ports and adapters](0014-ports-and-adapters.md), [0017 Security](0017-security-baseline.md)
