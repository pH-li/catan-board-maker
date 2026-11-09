# Security Policy

Hex Set Studio is a self-hosted app that runs on your own computer through Docker. Even so, we take security seriously. A bug that lets a malicious website, a shared project file or an uploaded image affect your machine is a bug we want to hear about.

## Supported versions

| Version | Supported |
|---|---|
| `main` (latest) | ✅ |
| Latest tagged release | ✅ |
| Older releases | ❌ Please upgrade (`git pull && docker compose up --build`) |

## Reporting a vulnerability

**Please do not open a public issue for security problems.**

Report privately through GitHub's private vulnerability reporting:

1. Go to the repository's **Security** tab.
2. Click **Report a vulnerability**.
3. Include:
   - what the issue is and where it is (file, endpoint, or feature)
   - steps to reproduce it, or a proof of concept
   - the impact you think it has
   - your environment (OS, Docker version, commit or release)

This is a volunteer-maintained project, so responses are best-effort:

- **Acknowledgement** within 7 days
- **Initial assessment** within 14 days
- **Fix or mitigation plan** as soon as practical, depending on severity

We'll credit you in the release notes unless you'd rather stay anonymous.

## Scope

**In scope:**
- The web app, API and worker in this repository
- The Docker images and Compose files we ship
- Handling of uploads, exports and imported project files
- Attacks from other websites against the local app (CSRF, DNS rebinding)
- Supply-chain issues in our dependencies or CI configuration

**Out of scope:**
- Deployments that ignore the documented setup, for example publishing the app to the internet or binding ports to `0.0.0.0`. Hex Set Studio is designed to be reachable from `localhost` only.
- Vulnerabilities in Docker, your OS or your browser themselves
- Denial of service that requires local access to your own machine

## How the app is secured

See [docs/THREAT_MODEL.md](docs/THREAT_MODEL.md) for the threat model and mitigations, and [ADR 0017](docs/adr/0017-security-baseline.md) for the security baseline.
