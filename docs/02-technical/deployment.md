# Deployment

## Production on Railway

Both CERPS microservices run as independent Docker containers on Railway, each with its own subdomain.

| Service | Production URL | Port |
|---|---|---|
| Currency | `https://supportive-vision-production.up.railway.app` | 8080 |
| Analytics | `https://sparkling-curiosity-production-9ffd.up.railway.app` | 8082 |

OpenAPI at `/swagger-ui.html`. Analytics has no direct DB access; it calls Currency via REST through a Spring `RestClient` with Caffeine, Resilience4j retry (3×, exp. backoff) and 5 s / 10 s timeouts.

### Environment Variables

| Variable | Scope | Description |
|---|---|---|
| `SPRING_PROFILES_ACTIVE` | Both | Activates `prod` profile |
| `SPRING_DATASOURCE_URL` | Currency | JDBC URL to Railway PostgreSQL |
| `SPRING_DATASOURCE_USERNAME` | Currency | DB user |
| `SPRING_DATASOURCE_PASSWORD` | Currency | DB password |
| `POSTGRES_CURRENCY_DB` | Currency | Database name |
| `LIQUIBASE_ENABLED` | Currency | Runs changelog on startup |
| `ENCRYPTION_MASTER_KEY` | Currency | 32-byte Base64; AES-256-GCM master key |
| `ADMIN_API_KEY` | Both | `X-API-Key` for admin endpoints |
| `FIXER_API_KEY` | Currency | Bootstrap Fixer.io key |
| `EXCHANGERATES_API_KEY` | Currency | Bootstrap ExchangeRatesAPI |
| `CURRENCYAPI_KEY` | Currency | Bootstrap CurrencyAPI |
| `GEMINI_API_KEY` | Currency | Backs `/ai/bank-commission` |

Secrets live only in Railway's store.

## Local Development

```bash
cp .env.example .env
# ENCRYPTION_MASTER_KEY=$(openssl rand -base64 32); ADMIN_API_KEY=any
docker-compose up --build
curl http://localhost:8080/actuator/health
```

Providers are registered via Admin API (`X-API-Key`):

```bash
curl -X POST http://localhost:8080/api/v1/admin/provider-keys \
  -H "X-API-Key: $ADMIN_API_KEY" -H "Content-Type: application/json" \
  -d '{"providerName": "Fixer.io", "apiKey": "..."}'

curl -X POST http://localhost:8080/api/v1/admin/refresh -H "X-API-Key: $ADMIN_API_KEY"
```

The same call registers `ExchangeRatesAPI` and `CurrencyAPI`.

## Android Release Configuration

CERPS URLs live in `local.properties`, surfaced via `BuildConfig`:

```properties
cerps.base.url=https://supportive-vision-production.up.railway.app/
cerps.analytics.url=https://sparkling-curiosity-production-9ffd.up.railway.app/
```

`app/build.gradle.kts` emits `BuildConfig.CERPS_BASE_URL`, `CERPS_ANALYTICS_URL`, `GEMINI_BASE_URL`, consumed by Retrofit via `NetworkModule`; switching to emulator loopback `http://10.0.2.2:8080/` is a property change. `network_security_config.xml` permits cleartext on `10.0.2.2`, `localhost` and dev LAN; Railway uses HTTPS.

## Health Checks and Monitoring

Actuator exposes `/actuator/health` on both services. Railway polls it; Android hits it via `NetworkStatusRepository`, caches for 30 s, and combines it with `STALE_THRESHOLD_MS` (8 h) to drive the stale-rate banner. JSON logs stream to Railway; `currency.rates.refresh.failures` and `currency.provider.failures` surface provider degradation.
