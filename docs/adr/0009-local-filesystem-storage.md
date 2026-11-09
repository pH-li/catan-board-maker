# 0009. Store uploads and exports on the local filesystem; no database

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

The backend has three kinds of data to keep:

1. **Uploaded images** (for reliefs and traced outlines), until the export that uses them finishes.
2. **Export outputs** (STL/zip files), until the user downloads them.
3. **Job state** (status, progress, errors).

User **projects** (designs and board layouts) are saved in the browser (IndexedDB) and in user-exported zip files, not on the server ([0011](0011-state-management.md)). There are no accounts ([0010](0010-no-accounts.md)).

## Decision

- **Files** live in a local `data/` directory (mounted as a Docker volume, gitignored):
  ```
  data/uploads/{imageId}.{png|jpg|svg}
  data/exports/{jobId}.{stl|zip}
  ```
- **Job state** lives in BullMQ/Redis ([0008](0008-bullmq-job-queue.md)). No separate database.
- All file access goes through the `FileStore` port ([0014](0014-ports-and-adapters.md)). It accepts only server-generated IDs, resolves paths, and **asserts they stay inside `DATA_DIR`** (path traversal protection, [0017](0017-security-baseline.md)).
- Retention: 24 hours, enforced by the cleanup scheduler.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| DynamoDB Local + S3 via LocalStack | Cloud-ready | Emulators for a local app | See [0013](0013-local-docker-distribution.md) |
| SQLite | Real queries, single file | No relational data to query | YAGNI |
| Postgres | Powerful | Another service, migrations | YAGNI |
| Store files in Redis | One service | Memory-hungry for binary files | Wrong tool |

## Consequences

**Positive**
- The simplest possible storage. Users can see their exports in `data/` directly.
- Nothing to migrate or administer.

**Negative / trade-offs**
- Not suited to multiple servers. That's acceptable for a single-user local app; a future deploy would add an S3 `FileStore` adapter.

## Related

- [0008 BullMQ](0008-bullmq-job-queue.md), [0014 Ports and adapters](0014-ports-and-adapters.md), [0017 Security](0017-security-baseline.md)
