# 0007. Share Zod schemas between frontend and backend, and generate OpenAPI from them

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Data crosses several boundaries: user input → frontend → API → job queue → worker, plus project files imported from disk. Each boundary needs both:

- **runtime validation**, because outside data is untrusted, whatever the frontend checked
- **compile-time types**, so code knows the data's shape

Writing types and validators separately makes them drift apart. We also want accurate API documentation.

## Decision

- **Zod** schemas live in `packages/schemas` and are the single definition of every request, response, job payload and project-file shape.
- TypeScript types are derived from them with `z.infer`, never written by hand alongside.
- The **frontend** validates forms with them, for user experience. The **API** validates every request with them, for security and correctness. The **worker** re-validates job data. **Project import** validates `project.json`.
- Schemas include **limits**: string lengths, numeric ranges, array sizes and a total complexity budget ([0017](0017-security-baseline.md)).
- An OpenAPI document is generated with `@asteasolutions/zod-to-openapi` and served at `/api/docs`.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Hand-written validation + separate TS types | No dependency | Drift between types and checks; verbose | Error-prone |
| JSON Schema + Ajv | Standard format | Separate type generation step; less ergonomic | Zod integrates directly with TS |
| Valibot / ArkType | Smaller or faster | Less ecosystem support for OpenAPI and forms | Zod is the most widely adopted |

## Consequences

**Positive**
- Frontend and backend can't disagree about data shapes: changing a schema breaks the compile on both sides.
- Bad input is rejected with a 400 and a clear message before any business logic runs.
- The API docs stay accurate because they're generated from the same source.

**Negative / trade-offs**
- `packages/schemas` becomes a central dependency, so changes there ripple outward (intentionally).

## Related

- [0006 Express](0006-express-api.md), [0017 Security baseline](0017-security-baseline.md)
