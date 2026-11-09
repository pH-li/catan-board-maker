# 0017. Adopt a security baseline for a local, self-hosted app

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Running locally ([0013](0013-local-docker-distribution.md)) with no accounts ([0010](0010-no-accounts.md)) removes many risks: there's no cloud bill to attack, no personal data and no secrets. It does **not** remove all of them. A local web server has its own threats:

- **Other websites** the user visits can send requests to `localhost` (CSRF), or use **DNS rebinding** to read responses.
- **Shared project zips** from other people can be zip bombs or contain path-traversal entries.
- **Uploaded images** can be malicious: SVGs with scripts, decompression bombs, fake file types.
- **Expensive requests** can exhaust the user's CPU and memory.
- **Supply-chain** compromise of dependencies, images or CI.
- **Network exposure** if services listen on all interfaces, making them reachable from other devices on the Wi-Fi.

The full analysis is in [docs/THREAT_MODEL.md](../THREAT_MODEL.md).

## Decision

Apply these principles everywhere:
- **defense in depth:** several layers of protection
- **least privilege:** each part gets only the access it needs
- **secure by default:** everything starts locked, and we open only what's needed
- **small attack surface:** fewer services, endpoints and dependencies

The baseline controls:

1. **Network:** only `web` is published, on `127.0.0.1`. Redis, the API and the worker stay on the internal Docker network.
2. **Origin and Host guard:**
   - state-changing API requests with a foreign `Origin` are rejected (403)
   - requests whose `Host` isn't the expected localhost host and port are rejected, which blocks DNS rebinding
   - JSON endpoints require `Content-Type: application/json`
   - no CORS headers are sent
3. **Validation at every trust boundary**, using Zod schemas with limits ([0007](0007-zod-shared-schemas.md)), plus a per-request **complexity budget**.
4. **Uploads:**
   - multer size and count limits
   - **magic-byte** type detection
   - pixel-dimension check before decoding
   - SVGs parsed only for path data and never rendered as HTML (DOMPurify wherever the client displays them)
5. **Filesystem:** only server-generated UUID filenames. `FileStore` confines every path to `DATA_DIR`. IDs are validated as UUIDs before use.
6. **Project import** (in the browser): before extracting, check the zip's central directory against caps on total size, entry count and compression ratio. Entry names are allow-listed. `project.json` is validated with Zod.
7. **Resource limits:** worker concurrency, a job timeout, a Node heap limit, and container memory and CPU limits.
8. **Headers** (nginx): a strict CSP, `X-Content-Type-Options`, `Referrer-Policy` and `frame-ancestors 'none'`. API responses also use `helmet`.
9. **Errors:** generic messages plus a `requestId`, never stack traces. Logs never contain user content.
10. **Containers:** non-root users, pinned base image digests, production dependencies only, read-only root filesystem where possible, and a Trivy scan.
11. **Supply chain and CI:**
    - frozen lockfile
    - pnpm build-script allow-list
    - Dependabot, `pnpm audit` and CodeQL
    - gitleaks (pre-commit and CI) plus GitHub push protection
    - SHA-pinned actions and read-only workflow permissions
    - branch protection
12. **Process:**
    - [SECURITY.md](../../SECURITY.md) for private vulnerability reports
    - a security checklist in the PR template
    - the threat model is reviewed whenever an input, endpoint, service or dependency category is added

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Rely on "it's only localhost" | No effort | Ignores CSRF, DNS rebinding and malicious files | A false sense of safety |
| A local password/token for the API | Strong CSRF defence | UX friction; a token to manage | Origin + Host checks are sufficient for a localhost app |

## Consequences

**Positive**
- Users can safely open project files from strangers and browse the web while the app runs.
- Security checks are automated in CI rather than left to memory.

**Negative / trade-offs**
- More middleware and tests to maintain.
- A strict CSP may need a carefully justified exception (for example `'wasm-unsafe-eval'` for manifold-3d).

## Related

- [docs/THREAT_MODEL.md](../THREAT_MODEL.md), [SECURITY.md](../../SECURITY.md), [0013 Local distribution](0013-local-docker-distribution.md)
