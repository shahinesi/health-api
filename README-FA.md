<div dir="rtl" align="right">

# Health API — راهنمای فارسی

[🌍 English](README.md) | **🇮🇷 فارسی**

Health API یک سرویس سبک برای **مانیتورینگ مداوم endpointهای HTTP/HTTPS** است که با Node.js، Express و PostgreSQL ساخته شده است.

این پروژه سرویس‌ها را در بازه‌های زمانی مشخص بررسی می‌کند، نتیجه و latency را در PostgreSQL ذخیره می‌کند، incident باز/بسته می‌کند و یک Dashboard و REST API در اختیار می‌گذارد.

## قابلیت‌های اصلی

- مانیتورینگ پس‌زمینه با concurrency محدود
- هماهنگی چند instance با PostgreSQL، `FOR UPDATE SKIP LOCKED` و lease زمان‌دار
- ذخیره تاریخچه health check و latency
- Incident lifecycle با threshold خطا
- Webhook اختیاری برای باز/بسته‌شدن incident
- Health check دستی بدون ایجاد request storm در Dashboard
- محافظت SSRF برای IPv4/IPv6، redirect و DNS rebinding
- Allowlist صریح برای سرویس‌های داخلی
- Basic Auth با رفتار fail-closed در Production
- Rate limit، Origin Guard و Security Headerهای سخت‌گیرانه
- `/live`، `/ready` و metrics سازگار با Prometheus
- Structured JSON logs و Request ID
- Graceful shutdown
- Docker non-root و read-only
- Migration نسخه‌دار PostgreSQL
- CI شامل lint، test، audit، Docker build و CodeQL

## شروع سریع با Docker Compose

ابتدا فایل env نمونه را کپی کن:

```bash
cp .env.example .env
```

حداقل این مقادیر را با secret واقعی جایگزین کن:

```text
DB_PASSWORD=<long-random-password>
ADMIN_USERNAME=admin
ADMIN_PASSWORD=<at-least-16-characters>
```

سپس:

```bash
docker compose up -d --build
```

Dashboard:

```text
http://localhost:3000/dashboard
```

> در Production، TLS باید روی reverse proxy یا load balancer معتبر terminate شود. Basic Auth را روی شبکه عمومی بدون HTTPS استفاده نکن.

## Endpointهای عملیاتی

```http
GET /live
GET /ready
GET /health
GET /metrics
```

`/live` فقط زنده‌بودن process را بررسی می‌کند. `/ready` اتصال PostgreSQL را هم اعتبارسنجی می‌کند. `/metrics` نیازمند authentication است.

## API سرویس‌ها

```http
GET    /services?limit=100&offset=0&search=payments
POST   /services
GET    /services/:id
PATCH  /services/:id
DELETE /services/:id
```

نمونه ایجاد سرویس:

```json
{
  "name": "Payments API",
  "url": "https://payments.example.com/health",
  "intervalSeconds": 60,
  "timeoutMs": 5000,
  "expectedStatus": 200,
  "enabled": true
}
```

## وضعیت، Check دستی، History و Incident

```http
GET  /services/:id/health
POST /services/:id/check
GET  /services/:id/history?limit=100
GET  /services/:id/incidents?limit=50
```

`GET /services/:id/health` فقط آخرین نتیجه ذخیره‌شده را برمی‌گرداند و outbound request ایجاد نمی‌کند. برای check فوری از `POST /services/:id/check` استفاده می‌شود.

## مدل مانیتورینگ

Worker سرویس‌های due را به‌صورت atomic claim می‌کند. بعد از claim، یک lease زمان‌دار روی سرویس باقی می‌ماند تا دو instance هم‌زمان یک endpoint را check نکنند. اگر worker crash کند، lease منقضی می‌شود و سرویس دوباره قابل claim است.

هر check این اطلاعات را ذخیره می‌کند:

- وضعیت: `healthy`، `unhealthy`، `unreachable` یا `blocked`
- HTTP status code
- latency
- خطای امن‌شده
- زمان check
- تعداد خطاهای متوالی
- وضعیت incident

## سیاست SSRF

به‌صورت پیش‌فرض فقط مقصدهای عمومی HTTP/HTTPS مجاز هستند. loopback، private، link-local، metadata، multicast و سایر محدوده‌های special-use برای IPv4 و IPv6 بلاک می‌شوند.

اتصال TCP به IPای pin می‌شود که policy را پاس کرده است و هر redirect دوباره از ابتدا validate می‌شود.

برای سرویس‌های داخلی بهتر است hostname مشخص را allowlist کنی:

```text
TARGET_ALLOWLIST=api.internal.example,*.svc.example
```

## مستندات معماری و عملیات

- [Architecture](docs/ARCHITECTURE.md)
- [Design Document](docs/DESIGN.md)
- [Threat Model](docs/THREAT_MODEL.md)
- [Operations / Runbook](docs/OPERATIONS.md)
- [ADR Index](docs/adr/README.md)
- [Production Guide](docs/PRODUCTION.md)
- [Production Readiness](docs/PRODUCTION_READINESS.md)
- [Security Policy](SECURITY.md)
- [Contributing](CONTRIBUTING.md)
- [Changelog](CHANGELOG.md)

## توسعه محلی

```bash
npm ci
AUTH_DISABLED=true MONITOR_ENABLED=false npm start
```

Production اجازه `AUTH_DISABLED=true` نمی‌دهد و در این حالت startup fail می‌شود.

## تست و کیفیت

```bash
npm run lint
npm run format:check
npm test
```

CI علاوه بر lint و test، dependency audit، Docker image build و CodeQL را هم اجرا می‌کند.

## مجوز

پروژه در `package.json` مجوز ISC را اعلام کرده و متن مجوز در [LICENSE](LICENSE) قرار دارد.

</div>
