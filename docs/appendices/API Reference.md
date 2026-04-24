# API Reference

## Overview

CERPS is split into two independent Spring Boot 3 microservices, each deployed on Railway behind HTTPS. Public endpoints serve the Android client without authentication; admin endpoints require the `X-API-Key` header.

| Service | Base URL | Port (local) |
|---|---|---|
| Currency Service | `https://supportive-vision-production.up.railway.app` | `8080` |
| Analytics Service | `https://sparkling-curiosity-production-9ffd.up.railway.app` | `8082` |

## Authentication

| Endpoint Group | Mechanism | Header |
|---|---|---|
| Public endpoints (`/api/v1/currencies`, `/api/v1/rates/...`, `/api/v1/ai/...`, `/api/v1/analytics/...`, `/actuator/health`) | None | — |
| Admin endpoints (`/api/v1/admin/...` on both services) | Static API key | `X-API-Key: <ADMIN_API_KEY>` |

The admin key is provisioned in the Railway environment as `ADMIN_API_KEY` and never ships with the Android APK.

## Public Endpoints

### Currency Service

#### GET /api/v1/currencies

Returns the list of currencies supported by CERPS.

Request:

```
GET /api/v1/currencies HTTP/1.1
Host: supportive-vision-production.up.railway.app
```

Response:

```json
{
  "currencies": [
    "EUR", "USD", "GBP", "CHF", "JPY", "PLN", "CZK", "HUF", "BYN",
    "UAH", "GEL", "SEK", "NOK", "DKK", "TRY", "CAD", "AUD", "NZD"
  ]
}
```

| Status Code | Description |
|---|---|
| 200 | Success |
| 503 | All providers unavailable and no cached snapshot |

#### POST /api/v1/currencies/convert

Converts an amount between two currencies using the latest interbank median rate.

Request body:

```json
{ "amount": 100.00, "from": "PLN", "to": "EUR" }
```

Response:

```json
{
  "success": true,
  "originalAmount": 100.00,
  "fromCurrency": "PLN",
  "toCurrency": "EUR",
  "convertedAmount": 22.222222,
  "exchangeRate": 0.222222,
  "timestamp": "2026-03-20T12:00:00Z"
}
```

| Status Code | Description |
|---|---|
| 200 | Success |
| 400 | Unknown currency code or non-positive amount |
| 503 | Rate not available |

#### GET /api/v1/rates/current?base=EUR

Returns every supported target rate against the requested base.

Request:

```
GET /api/v1/rates/current?base=EUR
```

Response:

```json
{
  "base": "EUR",
  "rates": {
    "USD": 1.08,
    "PLN": 4.30,
    "GBP": 0.86,
    "CHF": 0.96
  },
  "timestamp": "2026-03-20T12:00:00Z"
}
```

| Status Code | Description |
|---|---|
| 200 | Success |
| 400 | Unsupported `base` |
| 503 | No rates persisted yet |

#### GET /api/v1/ai/bank-commission?bankName=X

Server-side Gemini proxy that returns the typical foreign-currency commission percentage for a named bank. Pinned to `gemini-3.1-flash-lite-preview`.

Request:

```
GET /api/v1/ai/bank-commission?bankName=Revolut
```

Response — found:

```json
{ "commission": 0.5, "found": true }
```

Response — not found:

```json
{ "commission": null, "found": false }
```

| Status Code | Description |
|---|---|
| 200 | Success (use `found` to discriminate) |
| 400 | `bankName` blank, longer than 100 chars, or contains characters outside Unicode letter/digit/space/hyphen/apostrophe |
| 429 | Rate limit exceeded (10 req/min/IP) |
| 503 | Gemini upstream timeout or error |

### Analytics Service

#### GET /api/v1/analytics/trends?from=USD&to=EUR&period=30D

Returns the rate trend for a currency pair over a fixed period.

Allowed `period` values: `1D`, `7D`, `30D`, `90D`, `180D`, `1Y`. Hourly variants are no longer accepted.

Request:

```
GET /api/v1/analytics/trends?from=USD&to=EUR&period=30D
```

Response:

```json
{
  "from": "USD",
  "to": "EUR",
  "period": "30D",
  "oldRate": 0.910000,
  "newRate": 0.930000,
  "changePercentage": 2.20,
  "startDate": "2026-02-18T00:00:00Z",
  "endDate": "2026-03-20T00:00:00Z",
  "dataPoints": 720,
  "points": [
    { "timestamp": "2026-02-18T00:00:00Z", "rate": 0.91 },
    { "timestamp": "2026-02-19T00:00:00Z", "rate": 0.92 }
  ]
}
```

| Status Code | Description |
|---|---|
| 200 | Success |
| 400 | Invalid `period`, unknown currency code, or `from == to` |
| 503 | Currency Service unreachable after Resilience4j retries |

### Health Checks

#### GET /actuator/health

Available on both services.

Response:

```json
{ "status": "UP" }
```

| Status Code | Description |
|---|---|
| 200 | Service is up |
| 503 | One or more health indicators are down |

## Admin Endpoints

All require `X-API-Key: <ADMIN_API_KEY>`. Each is rate-limited to 10 req/min/IP via `PublicEndpointRateLimitFilter`, which extends the shared `AbstractRateLimitFilter` in `cerps-common`. The base class extracts the client IP from `X-Forwarded-For` (with trusted-proxy normalisation) and falls back to `remoteAddr`, blocking the spoofing pattern that motivated the v8 fix.

### Currency Service (port 8080)

#### POST /api/v1/admin/refresh

Forces an immediate rate refresh outside the 8-hour scheduler.

Request:

```
POST /api/v1/admin/refresh
X-API-Key: <ADMIN_API_KEY>
```

Response:

```json
{ "status": "ok", "ratesUpdated": 17, "timestamp": "2026-03-20T12:00:00Z" }
```

#### POST /api/v1/admin/currencies?currency=PLN

Adds a currency to the supported list.

Request:

```
POST /api/v1/admin/currencies?currency=PLN
X-API-Key: <ADMIN_API_KEY>
```

Response:

```json
{ "currency": "PLN", "added": true }
```

#### Provider Key CRUD

| Method | URL | Purpose |
|---|---|---|
| `POST` | `/api/v1/admin/provider-keys` | Create a new encrypted provider key |
| `GET` | `/api/v1/admin/provider-keys` | List active keys (encrypted blobs are not returned) |
| `PUT` | `/api/v1/admin/provider-keys/{id}` | Rotate or update a key |
| `DELETE` | `/api/v1/admin/provider-keys/{id}` | Deactivate a key |

Create request:

```json
{ "providerName": "Fixer.io", "apiKey": "238b171c95ef2d4017af8c49f7b0fd19" }
```

Create response:

```json
{
  "id": 1,
  "providerName": "Fixer.io",
  "active": true,
  "createdAt": "2026-03-20T12:00:00Z"
}
```

### Analytics Service (port 8082)

#### POST /api/v1/admin/cache/refresh

Evicts the Caffeine trends cache and the supported-currencies cache.

Request:

```
POST /api/v1/admin/cache/refresh
X-API-Key: <ADMIN_API_KEY>
```

Response:

```json
{ "status": "ok", "evicted": ["trends", "supportedCurrencies"] }
```

## Error Responses

All errors flow through `GlobalExceptionHandler` and return a uniform JSON body.

```json
{
  "error": "BAD_REQUEST",
  "message": "Invalid currency code: XYZ",
  "timestamp": "2026-03-20T12:00:00Z"
}
```

| Error Code | Message | When This Occurs |
|---|---|---|
| 400 | Bad Request | Validation failure: unknown currency, blank `bankName`, name longer than 100 chars, illegal `period`, or non-positive amount. |
| 401 | Unauthorized | Admin endpoint called without `X-API-Key` header. |
| 403 | Forbidden | Admin endpoint called with an `X-API-Key` value that does not match `ADMIN_API_KEY`. |
| 404 | Not Found | Provider key id not present, or path does not exist. |
| 500 | Internal Server Error | Unexpected exception not mapped by `GlobalExceptionHandler`; logged with `currency.rates.refresh.failures` metric increment. |
| 503 | Service Unavailable | All rate providers failed and no cached snapshot is available; Gemini timeout; or Analytics → Currency hop exhausted Resilience4j retries. |

## Rate Limiting

Enforced via `AbstractRateLimitFilter` in `cerps-common`, with one concrete subclass per service: `PublicEndpointRateLimitFilter` in Currency Service and `TrendsEndpointRateLimitFilter` in Analytics Service. The client IP is read from `X-Forwarded-For` through a shared helper (trusted-proxy normalisation, fallback to `remoteAddr`).

| Endpoint Group | Limit |
|---|---|
| Public endpoints (`/api/v1/currencies`, `/api/v1/rates/current`, `/api/v1/analytics/trends`, `/actuator/health`) | 60 req/min/IP |
| AI endpoint (`/api/v1/ai/bank-commission`) | 10 req/min/IP |
| Admin endpoints (`/api/v1/admin/*` on both services) | 10 req/min/IP |

Exceeding a limit returns HTTP 429 with the standard error body.

## Swagger / OpenAPI

Live springdoc-openapi documentation is published per service:

| Service | Swagger UI |
|---|---|
| Currency Service | https://supportive-vision-production.up.railway.app/swagger-ui.html |
| Analytics Service | https://sparkling-curiosity-production-9ffd.up.railway.app/swagger-ui.html |
