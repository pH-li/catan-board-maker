# 0014. Isolate infrastructure behind ports and adapters

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Business logic (create an export, validate an upload, clean up old files) shouldn't care whether jobs go to BullMQ or SQS, or whether files live on disk or in S3. Coupling services directly to Redis and `fs`:

- makes unit tests slow and fragile, because they need real infrastructure
- makes a future move (for example, deploying to the cloud) a rewrite instead of a swap
- scatters security-critical logic (path confinement) across many call sites

## Decision

Use the **ports and adapters** (hexagonal) pattern in `apps/api`:

```
services/  ──depend on──►  ports/  (TypeScript interfaces)
                              ▲
                  implemented │ by
                              │
adapters/bullmq/   adapters/fs/   adapters/memory/ (test fakes)
```

- **Ports:**
  - `JobQueue`: `enqueue`, `getStatus`, `onProgress`
  - `FileStore`: `saveUpload`, `readUpload`, `writeExport` (atomic), `openExport`, `deleteOlderThan`
- **Adapters:**
  - `BullMqJobQueue` and `LocalFileStore` in production
  - in-memory fakes for unit tests
- **Composition root:** `server.ts` and `worker.ts` build the adapters from validated config and pass them to the services, using plain constructor or factory injection with no DI framework.
- **Rule:** services and controllers never import from `adapters/`. A lint rule (import restrictions) enforces this.

The same idea applies at the app edge: `app.ts` builds the Express app and doesn't know how it's started.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Call Redis/fs directly in services | Less code | Hard to test, hard to change, scattered security logic | Fails our quality goals |
| DI framework (InversifyJS, NestJS) | Automatic wiring | Decorators, magic, learning curve | Plain injection is enough |

## Consequences

**Positive**
- Unit tests run in milliseconds against in-memory fakes.
- Path-traversal protection lives in one place (`LocalFileStore`).
- A future cloud move means writing `SqsJobQueue` and `S3FileStore`, with no change to the services.

**Negative / trade-offs**
- Some extra interfaces and wiring code.
- Integration tests are still needed to prove the real adapters behave like the fakes ([0015](0015-testing-strategy.md)).

## Related

- [0006 Express](0006-express-api.md), [0008 BullMQ](0008-bullmq-job-queue.md), [0009 Storage](0009-local-filesystem-storage.md)
