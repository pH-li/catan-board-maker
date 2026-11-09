# Architecture

> **Status:** this describes the target architecture for v1. It's being built in phases (see [the roadmap](#build-roadmap)). The reasoning behind each choice lives in the [ADRs](adr/README.md).

## 1. What the system does

Hex Set Studio turns **design parameters** into **3D-printable STL files** for a Catan-compatible board set.

```
design parameters (JSON)  ──►  geometry builders  ──►  triangle mesh  ──►  3D preview  /  STL file
{ piece: "settlement",          buildSettlement()       vertices +          (browser)     (export)
  roof: "gabled", ... }                                  triangles
```

The same geometry code produces both the live preview and the exported file, so **the preview always matches the print** ([ADR 0001](adr/0001-typescript-everywhere.md)).

## 2. Runtime view

What `docker compose up` starts on a user's machine:

```mermaid
flowchart LR
  subgraph Host["User's computer"]
    B["Browser<br/>React + R3F<br/>geometry: preview detail"]
    subgraph Docker["Docker network (internal only)"]
      WEB["web<br/>nginx (unprivileged)<br/>static files + security headers<br/>proxies /api"]
      API["api<br/>Express 5 + Zod"]
      WK["worker<br/>BullMQ worker<br/>geometry: full detail"]
      R[("redis<br/>queue + job state")]
    end
    D[("./data<br/>uploads/ · exports/")]
  end
  B -->|"127.0.0.1:8080"| WEB
  WEB -->|"/api/*"| API
  API -->|"enqueue · status"| R
  WK -->|"take job · progress"| R
  API <-->|"read · write"| D
  WK -->|"write STL / zip"| D
```

| Service | Responsibility | Published? |
|---|---|---|
| `web` | Serves the built React app; reverse-proxies `/api`; sets CSP and other headers | **Yes, `127.0.0.1:8080` only** |
| `api` | Validates requests, stores uploads, enqueues jobs, reports status, streams downloads | No |
| `worker` | Builds meshes at full detail, validates them, writes STL/zip, reports progress | No |
| `redis` | BullMQ queue and job state (`noeviction`, AOF persistence) | No |

`api` and `worker` share one image and one codebase (`apps/api`) with two entry points.

## 3. Export flow

```mermaid
sequenceDiagram
  participant B as Browser
  participant A as API
  participant F as FileStore (data/)
  participant Q as JobQueue (BullMQ/Redis)
  participant W as Worker
  B->>A: POST /api/uploads (multipart, optional, one per image)
  A->>A: size limit · magic bytes · pixel dimensions
  A->>F: save uploads/{imageId}
  A-->>B: 201 {imageId}
  B->>A: POST /api/exports {pieces, board, imageIds}
  A->>A: Zod validation + complexity budget
  A->>Q: enqueue(jobId = UUID, attempts 3, backoff)
  A-->>B: 202 {jobId}
  Q->>W: deliver job
  W->>F: read images
  W->>W: build meshes → manifold + wall checks → STL / zip
  W->>Q: progress %
  W->>F: write exports/{jobId}.tmp → atomic rename
  W->>Q: completed
  loop every 1–2 s until completed or failed
    B->>A: GET /api/exports/{jobId}
    A-->>B: {status, progress}
  end
  B->>A: GET /api/exports/{jobId}/download
  A-->>B: file stream (Content-Disposition: attachment)
```

**Reliability properties:**
- **Idempotent enqueue:** the `jobId` is the BullMQ job ID, so a duplicate submit doesn't create duplicate work.
- **At-least-once processing:** a crashed worker's job is retried (3 attempts, exponential backoff). Exhausted jobs stay in the failed set, which serves as the dead-letter queue.
- **Atomic output:** write to `.tmp`, then rename, so a partially written file is never served.
- **Retention:** jobs and files are removed after 24 h by BullMQ options and an hourly cleanup scheduler.

## 4. Code structure

### Packages
```mermaid
flowchart BT
  specs["packages/specs<br/>dimensions + piece counts"] --> schemas["packages/schemas<br/>Zod schemas + types"]
  specs --> geometry["packages/geometry<br/>pure TS + manifold-3d + STL writer"]
  schemas --> geometry
  geometry --> web["apps/web"]
  schemas --> web
  geometry --> api["apps/api"]
  schemas --> api
```

| Package | Contains | Must not contain |
|---|---|---|
| `packages/specs` | Component dimensions (mm), piece counts, expansion flags | Logic beyond constants and simple helpers |
| `packages/schemas` | Zod schemas, inferred types, OpenAPI registry, limits | I/O |
| `packages/geometry` | Pure builders `build<Piece>(params, detail)`, CSG features, validation, STL writer | DOM, React, Node I/O, Redis |
| `apps/web` | UI, viewer, stores, API client, Web Worker host | Duplicated geometry or schemas |
| `apps/api` | HTTP API, worker, ports and adapters | Geometry logic (it calls `packages/geometry`) |

### Backend layers (ports and adapters)
```mermaid
flowchart LR
  R[routes] --> C[controllers] --> S[services]
  S --> P{{ports<br/>JobQueue · FileStore}}
  BA[adapters/bullmq] -. implements .-> P
  FA[adapters/fs] -. implements .-> P
  MA[adapters/memory<br/>test fakes] -. implements .-> P
  CR["server.ts / worker.ts<br/>(composition root)"] --> BA & FA
```
- Controllers handle HTTP only. Services hold the business logic and depend on **ports**, never on adapters ([ADR 0014](adr/0014-ports-and-adapters.md)).
- `LocalFileStore` is the **only** code that touches the filesystem, and it confines every path to `DATA_DIR`.
- `config.ts` parses environment variables with Zod at startup, so misconfiguration fails fast.

### API surface (v1)

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/health` | API and Redis health |
| `POST` | `/api/uploads` | Upload one image (multipart, ≤5 MB, PNG/JPEG/SVG) → `{ imageId }` |
| `POST` | `/api/exports` | Create an export job → `202 { jobId }` |
| `GET` | `/api/exports/:jobId` | `{ status, progress, error? }` |
| `GET` | `/api/exports/:jobId/download` | Stream the finished file |
| `GET` | `/api/docs` | Generated OpenAPI documentation |

## 5. Frontend

```mermaid
flowchart LR
  UI["React components<br/>(designer · board · export)"] -->|selectors| ZS[("Zustand project store<br/>Immer · zundo · persist→IndexedDB")]
  UI -->|actions| ZS
  ZS -->|params| GW["geometry.worker.ts<br/>(Web Worker, preview detail)"]
  GW -->|mesh buffers| V["R3F viewer<br/>Canvas · Instances · OrbitControls"]
  UI --> TQ["TanStack Query<br/>uploads · exports · polling"]
  TQ -->|"/api"| API[(API)]
```

- **Project state** (Zustand) holds design parameters and the board layout. Components subscribe through selectors, so a slider drag re-renders only what depends on it ([ADR 0011](adr/0011-state-management.md)).
- **Geometry runs off the main thread** in a Web Worker. The UI stays responsive, and finished buffers are handed to three.js.
- **Server state** (TanStack Query) handles uploads, export creation and status polling.
- **Persistence:** auto-save to IndexedDB, plus manual project zip export and import. Imports are validated ([threat model](THREAT_MODEL.md)).

## 6. Environments

| | Users | Contributors |
|---|---|---|
| Start | `docker compose up` | `docker compose -f docker-compose.dev.yml up -d` + `pnpm dev` |
| Frontend | nginx on `127.0.0.1:8080` | Vite on `localhost:5173` (proxies `/api` → `:3000`) |
| API / worker | containers | `tsx watch`, reloads on save |
| Redis | internal container | container on `127.0.0.1:6379` |
| Data | `./data` volume | `./data` (configurable with `DATA_DIR`) |

## 7. Key invariants

1. **Millimeters only.** Every physical dimension comes from `packages/specs`.
2. **Geometry is pure and deterministic:** the same parameters and detail level always produce the same mesh.
3. **Validate at every trust boundary** with the shared Zod schemas: API requests, job payloads, imported projects, environment config.
4. **Only server-generated identifiers touch the filesystem.**
5. **Nothing listens beyond `127.0.0.1`.**
6. **Preview and export use the same builders**, differing only in `detail`.

## Build roadmap

| Phase | Scope |
|---|---|
| 0 | Docs, decisions, tooling, CI |
| 1 | Specs, schemas, base geometry for all v1 pieces, designer viewer |
| 2 | API, uploads, BullMQ export pipeline, security middleware, full Docker stack |
| 3 | Silhouette presets, text, icons, emboss and engrave |
| 4 | 3D board builder, randomizer, Export All + manifest |
| 5 | Heightmaps, multi-color split, print plates, image tracing, project zip import/export |
| 6 | E2E tests, polish, v1.0.0, optional GHCR images |
