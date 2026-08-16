# Health API

[🌍 English](README.md) | [🇮🇷 فارسی](README-FA.md)

A lightweight, production-hardened HTTP/HTTPS service monitoring application built with Node.js, Express, and PostgreSQL.

Health API continuously checks registered services, persists health history and latency, tracks incidents, exposes a REST API, and includes a web dashboard.

## Production features

- Continuous background monitoring with bounded concurrency
- PostgreSQL-backed multi-instance work claiming with expiring per-service leases
- Persistent health-check history and latency
- Incident open/resolve lifecycle with configurable failure threshold
- Optional incident webhook notifications
- Manual checks without dashboard N+1 request storms
- HTTP/HTTPS-only target policy
- SSRF protection: private/special IP blocking, redirect re-validation, and DNS-pinned connections
- Explicit allowlist for intentionally monitored private hostnames
- HTTP Basic authentication for dashboard and management API
- Global/manual-check rate limiting and browser origin protection
- Strict security headers and CSP
- Liveness and readiness endpoints
- Structured JSON request/application logs
- Graceful shutdown
- Non-root, read-only hardened API container
- PostgreSQL network isolation in Docker Compose
- CI lint/test/audit and Docker-build gates, CodeQL, and Dependabot

## Tech stack

- Node.js 24
- Express 5
- PostgreSQL 17
- `pg`
- Jest + Supertest
- Docker / Docker Compose
- GitHub Actions / GHCR

## Quick start with Docker Compose

Copy the environment template and set strong secrets:

```bash
cp .env.example .env
```

At minimum set:

```text
DB_PASSWORD=<long-random-password>
ADMIN_USERNAME=admin
ADMIN_PASSWORD=<at-least-16-characters>
```

Then run:

```bash
docker compose up -d --build
```

Dashboard:

```text
http://localhost:3000/dashboard
```

The browser will request the configured Basic Auth credentials.

For real production deployment, terminate TLS in a trusted reverse proxy/load balancer and do not expose plaintext Basic Auth to untrusted networks. See `docs/PRODUCTION.md`.

## Local development

Install dependencies:

```bash
npm ci
```

For intentionally unauthenticated local-only development:

```bash
AUTH_DISABLED=true MONITOR_ENABLED=false npm start
```

Never use `AUTH_DISABLED=true` with `NODE_ENV=production`; startup rejects it.

## Health probes

Liveness checks only the Node process:

```http
GET /live
```

Readiness verifies PostgreSQL connectivity:

```http
GET /ready
```

`GET /health` is a compatibility alias for readiness.

These probe endpoints do not require application authentication.

## Metrics

Authenticated Prometheus-style metrics are available at:

```http
GET /metrics
```

They include service-state gauges, open incidents, process uptime, and resident memory.

## REST API

All management endpoints require Basic Auth unless authentication is explicitly disabled outside production.

### List services

```http
GET /services?limit=100&offset=0&search=payments
```

### Create service

```http
POST /services
Content-Type: application/json

{
  "name": "Payments API",
  "url": "https://payments.example.com/health",
  "intervalSeconds": 60,
  "timeoutMs": 5000,
  "expectedStatus": 200,
  "enabled": true
}
```

`intervalSeconds`, `timeoutMs`, `expectedStatus`, and `enabled` are optional. When `expectedStatus` is `null`, any 2xx response is healthy.

### Read/update/delete

```http
GET /services/:id
PATCH /services/:id
DELETE /services/:id
```

### Cached health status

```http
GET /services/:id/health
```

This returns the last persisted monitoring result and does **not** generate outbound traffic.

### Immediate manual check

```http
POST /services/:id/check
```

Manual checks share the same SSRF protections, persistence, incident logic, lease protection, and rate limit as background monitoring.

### History and incidents

```http
GET /services/:id/history?limit=100
GET /services/:id/incidents?limit=50
```

## Monitoring model

Workers claim due services atomically with PostgreSQL `FOR UPDATE SKIP LOCKED`. A time-bounded lease remains on each claimed service while the network check is in progress, preventing overlap after the SQL transaction ends. If a worker crashes, the lease expires automatically.

The monitor stores:

- status (`healthy`, `unhealthy`, `unreachable`, `blocked`)
- HTTP status code
- latency to response headers
- safe error code/message
- check timestamp
- consecutive failures
- incident lifecycle

Old check results are removed according to `RESULT_RETENTION_DAYS`.

## SSRF policy

Public internet targets are allowed by default. Loopback, private, link-local, metadata-style, multicast, documentation, and other special-use addresses are blocked for IPv4 and IPv6.

The outbound TCP connection is pinned to the IP address that passed policy validation, reducing DNS-rebinding risk. Every redirect is independently revalidated.

To monitor selected internal services, prefer an explicit hostname allowlist:

```text
TARGET_ALLOWLIST=api.internal.example,*.svc.example
```

Avoid broad `ALLOW_PRIVATE_TARGETS=true` unless the whole deployment is isolated and trusted.

## Incident webhooks

Optional environment variables:

```text
ALERT_WEBHOOK_URL=https://automation.example.com/health-events
ALERT_WEBHOOK_TOKEN=<optional-bearer-token>
ALERT_WEBHOOK_TIMEOUT_MS=5000
```

Events are emitted for `incident.opened` and `incident.resolved` after monitoring state has been persisted.

## Architecture and operations documentation

- [Architecture overview and Mermaid diagrams](docs/ARCHITECTURE.md)
- [Design document](docs/DESIGN.md)
- [Threat model](docs/THREAT_MODEL.md)
- [Operations runbook](docs/OPERATIONS.md)
- [Architecture Decision Records](docs/adr/README.md)
- [Production deployment guide](docs/PRODUCTION.md)
- [Production readiness assessment](docs/PRODUCTION_READINESS.md)

## Tests and quality

```bash
npm run lint
npm run format:check
npm test
```

Pull-request CI runs linting, database-backed tests, a production dependency audit, and a Docker image build. CodeQL runs in parallel. After merge to `main`, the validated image is tagged with both the commit SHA and `latest` and published to GHCR. Dependabot keeps npm, Actions, and Docker dependencies current.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Security-sensitive changes should also follow [SECURITY.md](SECURITY.md) and the architecture decision process under `docs/adr/`.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for notable changes.

## License

The project declares the ISC license in `package.json`; the matching license text is available in [LICENSE](LICENSE).
