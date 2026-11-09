# 0013. Distribute as a local app run with Docker Compose; cloud deployment is out of scope

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

The original plan was a public website on the AWS free tier, using Lambda, SQS, DynamoDB, S3, CloudFront, Terraform and GitHub OIDC deploys. Working through it showed that cloud hosting adds significant scope (infrastructure-as-code, a cost-attack surface, cloud security) on top of an already ambitious app (3D geometry, a queue-backed API, a monorepo).

Lily decided to **learn one hard thing at a time**: this project runs locally, and AWS and Terraform get practiced on a separate, simpler project.

That leaves two audiences:
- **Users**, who want to design and print, ideally without installing a development toolchain.
- **Contributors**, who want fast reloads and debugging.

## Decision

- Hex Set Studio is a **clone-and-run local app**. Users need only **Docker and Git**:
  ```
  git clone https://github.com/pH-li/catan-board-maker.git
  cd catan-board-maker
  docker compose up
  # open http://localhost:8080
  ```
- `docker-compose.yml` runs the **full stack**:

  | Service | Role |
  |---|---|
  | `web` | unprivileged nginx serving the built frontend, proxying `/api`, sending security headers |
  | `api` | Express |
  | `worker` | BullMQ worker, same image as `api` with a different command |
  | `redis` | queue and job state |

  `./data` is mounted for files.
- **Only `web` is published, and only on `127.0.0.1:8080`.** Every other service is reachable only inside the Docker network.
- `docker-compose.dev.yml` runs **only Redis** (on `127.0.0.1:6379`) for contributors, who run `pnpm dev` for hot reloading.
- Images: multi-stage builds, production dependencies only, **non-root**, base images pinned by digest, read-only root filesystem where possible.
- Cloud deployment is **out of scope**. The code stays deployable through ports and adapters ([0014](0014-ports-and-adapters.md)).
- Optional later: publish images to GHCR from a release workflow, so users skip the build step.

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| AWS serverless + Terraform | Public website; cloud and IaC practice | Large scope, cost risk, cloud security | Moved to a separate practice project |
| Require Node + pnpm for users | No Docker | Users need a dev toolchain; "works on my machine" problems | Docker gives a one-command, reproducible setup |
| Desktop app (Electron/Tauri) | Native feel, no Docker | Packaging and signing per OS; a much bigger change | Out of scope for v1 |
| Browser-only static app (no backend) | Simplest | Loses the backend practice Lily wants | Backend is a learning goal |

## Consequences

**Positive**
- No hosting cost and no internet-facing attack surface.
- A reproducible environment for everyone.
- Development stays fast.

**Negative / trade-offs**
- Users must install Docker.
- No shareable URL. Designs are shared as project zip files.

**Follow-ups**
- If hosting is ever wanted, write a new ADR that supersedes this one, covering auth, cost controls and a cloud threat model.

## Related

- [0008 BullMQ](0008-bullmq-job-queue.md), [0014 Ports and adapters](0014-ports-and-adapters.md), [0017 Security](0017-security-baseline.md)
