# 0010. No user accounts or authentication

- **Status:** Accepted
- **Date:** 2026-11-08
- **Deciders:** Lily Phan

## Context

Accounts enable cloud saves, galleries and per-user permissions, but they bring password handling, sessions, account recovery, personal data and a much larger attack surface. Hex Set Studio is a self-hosted app that each person runs on their own computer ([0013](0013-local-docker-distribution.md)).

The guiding principle: **add authentication when there is something to protect or someone to identify.** A single-user local app has neither.

## Decision

- No accounts, login, sessions or per-user data.
- Projects are stored in the user's browser and in zip files they export themselves.
- Protection of the local API comes from **network and origin controls** instead: localhost-only binding, plus Origin and Host checks against malicious websites and DNS rebinding ([0017](0017-security-baseline.md)).

## Alternatives considered

| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Amazon Cognito / third-party auth | Cloud saves, gallery | Only meaningful for a hosted service | Out of scope |
| Local password on the app | Protects against other local users | Friction for little benefit on a personal machine | Not worth it |

## Consequences

**Positive**
- No personal data and no secrets.
- A smaller attack surface and simpler code.

**Negative / trade-offs**
- No cross-device sync. Users move projects by exporting and importing zips.
- If the app is ever hosted, auth must be designed then, in a new ADR.

## Related

- [0013 Local distribution](0013-local-docker-distribution.md), [0017 Security](0017-security-baseline.md)
