# 0008. Run exports asynchronously with BullMQ and Redis

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Building a full set (19 detailed hexes, chits, ports and every player piece) and zipping it can take far longer than a comfortable HTTP request. A **synchronous** export (the request waits for the file) risks timeouts, loses work if the connection drops, and ties up the API.

So we use the **asynchronous job** pattern:

```
POST /exports → 202 {jobId}        (fast)
worker builds the file in the background
GET /exports/{jobId} → status       (poll)
GET /exports/{jobId}/download       (when completed)
```

### How this decision evolved

This is a good example of **constraints driving decisions**:

1. Lily first asked about **Celery** (Python). It was set aside once the backend became TypeScript ([0001](0001-typescript-everywhere.md)).
2. While the plan targeted AWS, **SQS + Lambda** was chosen, and **BullMQ was rejected** because it needs an always-on Redis server and worker, which costs money in the cloud.
3. When the project became a **local, clone-and-run app** ([0013](0013-local-docker-distribution.md)), that cost constraint disappeared. Redis runs for free in Docker. Emulating SQS locally would add complexity for no benefit, so BullMQ became the simplest correct choice.

## Decision

Use **BullMQ** on **Redis** (running in Docker).

- **jobId:** a server-generated UUID, also passed to BullMQ as the job ID, which makes enqueuing **idempotent**.
- **Retries:** 3 attempts with exponential backoff. Jobs that exhaust them stay in BullMQ's **failed set**, which acts as our dead-letter queue.
- **Worker:** a separate process (`apps/api/src/worker.ts`), with configurable concurrency (default 2) and a per-job timeout.
- **Progress:** `job.updateProgress()` drives the UI's progress bar.
- **Atomic output:** write to `*.tmp`, then rename, so a crash never leaves a half-written file that looks complete.
- **Cleanup:** `removeOnComplete`/`removeOnFail` after 24 h, plus a BullMQ **job scheduler** (repeatable job, like Celery Beat) that runs hourly and deletes old exports and orphaned uploads.
- **Redis config:** `maxmemory-policy noeviction` (required by BullMQ, so job data is never silently evicted), with AOF persistence on a Docker volume.
- Services use BullMQ only through the `JobQueue` port ([0014](0014-ports-and-adapters.md)).

### Celery-to-BullMQ mapping

| Celery | BullMQ |
|---|---|
| Broker (Redis/RabbitMQ) | Redis |
| Worker process | `Worker` in `worker.ts` |
| Result backend | Redis (job state, return value, progress) |
| Retries | `attempts` + `backoff` |
| Beat (scheduled tasks) | Job schedulers / repeatable jobs |

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Synchronous export | Simplest | Timeouts, lost work, blocks the API | Large exports are slow |
| In-process queue (inside the API) | No Redis | Jobs lost on restart; heavy CPU work blocks the API | Not robust |
| SQS + DynamoDB via LocalStack | Matches AWS | Emulating a cloud we don't deploy to | Unnecessary complexity after [0013](0013-local-docker-distribution.md) |
| pg-boss / Graphile Worker | Postgres-backed | Requires Postgres, which we otherwise don't need | Redis is lighter for this |
| Celery | Mature | Python; doesn't fit [0001](0001-typescript-everywhere.md) | Wrong language |

## Consequences

**Positive**
- Robust: retries, crash recovery, progress and scheduled cleanup are built in.
- The API stays responsive because CPU-heavy work runs in a separate process.
- Teaches the queue patterns used everywhere: idempotency, dead letters, at-least-once delivery.

**Negative / trade-offs**
- Redis is one more service to run (handled by Docker Compose).
- Workers must be idempotent, because a job may run more than once.

## Related

- [0009 Storage](0009-local-filesystem-storage.md), [0013 Local distribution](0013-local-docker-distribution.md), [0014 Ports and adapters](0014-ports-and-adapters.md)
