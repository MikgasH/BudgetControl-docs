# Criterion: Back-end

## Architecture Decision Record

### Status

Accepted — 2026-04-24

### Context

BudgetControl needed reliable interbank rates for eighteen currencies. A direct Android-to-provider integration would be a single point of failure on free-tier exhaustion and would ship the provider key inside the APK. The project also required a historical trend endpoint and a server-mediated Gemini commission lookup that must not expose its key to the device.

### Decision

Two independent Spring Boot 3 / Java 21 microservices on Railway:

- **Currency Service (8080)** — rate aggregation on an 8-hour scheduler, AES-256-GCM provider-key management, Gemini proxy, admin endpoints behind `X-API-Key` with per-IP rate limiting.
- **Analytics Service (8082)** — trend endpoint for `1D/7D/30D/90D/180D/1Y`, Caffeine cache, Resilience4j retry (3×), 5 s / 10 s timeouts; reads Currency over REST, no shared database.

### Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|---|---|---|---|
| Fixer.io alone | No backend | SPOF; key in APK; no history | Unacceptable reliability/security |
| Monolithic Spring Boot | One container | Aggregation/analytics coupled | Contradicts microservice requirement |
| Firebase | Managed | Scheduler awkward; lock-in | Rules out history and admin tooling |

### Consequences

Positive: median hides provider outages; keys never leave the backend; services redeploy independently; Swagger describes both contracts. Negative: two containers on the free tier; REST hop adds latency. Neutral: 8-hour staleness is explicit to the client.

### Implementation Details

```
CERPS/
├── currency-service/     # controller / service / repository / entity / scheduler
├── analytics-service/    # controller / service / client (RestClient + Caffeine)
└── cerps-common/         # shared DTOs, error model, constants
```

Liquibase changelogs v1.0–v1.8 under `currency-service/resources/db/changelog/`.

| Decision | Rationale |
|---|---|
| Median of three providers | Hides outages; rejects outliers |
| AES-256-GCM provider keys | Rows useless without Railway master key |
| Analytics → Currency via REST | Schemas evolve independently |
| Gemini proxied via CERPS | Key server-side; enables validation + throttle |
| Liquibase over manual SQL | Versioned DDL in Git; reproducible schema |

| Method | URL | Service |
|---|---|---|
| `GET` | `/api/v1/currencies` | Currency |
| `POST` | `/api/v1/currencies/convert` | Currency |
| `GET` | `/api/v1/rates/current?base=EUR` | Currency |
| `GET` | `/api/v1/ai/bank-commission?bankName=X` | Currency |
| `GET` | `/api/v1/analytics/trends?from&to&period` | Analytics |
| `GET` | `/actuator/health` | Both |
| `POST` | `/api/v1/admin/refresh` | Currency (key) |
| `…` | `/api/v1/admin/provider-keys[/{id}]` | Currency (key) |
| `POST` | `/api/v1/admin/cache/refresh` | Analytics (key) |

Test coverage: **74%** Currency / **77%** Analytics (JaCoCo).

### Requirements Checklist

| # | Requirement | Status | Evidence |
|---|---|---|---|
| 1 | Modern framework | Done | Spring Boot 3 / Java 21 |
| 2 | Database + ORM | Done | PostgreSQL 16 + Spring Data JPA |
| 3 | Layered architecture | Done | Controller → service → repository |
| 4 | SOLID | Done | `RateProvider` interface (OCP/DIP) |
| 5 | API documentation | Done | Swagger on both services |
| 6 | Global error handling | Done | `GlobalExceptionHandler` → 503/400 |
| 7 | Structured logging | Done | JSON logs + failure metrics |
| 8 | Production deployment | Done | Railway over HTTPS |
| 9 | Coverage ≥ 70% | Done | 74% / 77% JaCoCo |
| 10 | Microservices | Done | Two independent services |
| 11 | Caching | Done | Caffeine + in-memory TTL 8 h |

### Known Limitations

| Limitation | Impact | Potential Solution |
|---|---|---|
| No CI/CD; manual Railway deploy | No automated quality gate | GitHub Actions + JaCoCo/SonarCloud |
| Duplicate `exchange_rates` on overlapping ticks | Minor Δ% skew | Unique constraint + idempotent upsert |
| Railway free-tier ceiling | Cold starts; limited headroom | Paid plan or self-hosted VPS |

### References

- https://github.com/MikgasH/CERPS
- https://supportive-vision-production.up.railway.app/swagger-ui.html
- https://sparkling-curiosity-production-9ffd.up.railway.app/swagger-ui.html
