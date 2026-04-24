# Technology Stack

## Stack Overview

| Layer | Technology | Version | Justification |
|---|---|---|---|
| Mobile language | Kotlin | 1.9 | Null-safety and coroutines |
| UI toolkit | Jetpack Compose + Material 3 | BOM 2024.09 | Declarative state-driven UI |
| Dependency injection | Hilt | 2.48.1 | Compile-time DI, ViewModel integration |
| Local database | Room | 2.6.1 | Typed SQLite; compile-time queries; Flow |
| Network | Retrofit + OkHttp | 2.9.0 / 4.12.0 | Declarative HTTP; logging off in release |
| Async | Kotlin Coroutines + Flow | — | Structured concurrency |
| Preferences and cache | Jetpack DataStore | — | Async prefs; persists EUR-based rate cache |
| Charts | Canvas + YCharts | 2.1.0 | Canvas for rate history; YCharts for pie chart |
| AI client | Gemini via CERPS | gemini-3.1-flash-lite-preview | Key server-side |
| Backend language | Java | 21 | LTS with virtual threads |
| Backend framework | Spring Boot | 3 | Auto-config, validation, actuator, OpenAPI |
| Schema migrations | Liquibase | v1.9 | XML changelogs under Git |
| Backend database | PostgreSQL | 16 | Relational integrity; functional indexes |
| Backend cache | Caffeine | — | In-process cache on Analytics |
| Resilience | Resilience4j | — | Retry with exponential backoff |
| Deployment | Railway (Docker) | — | Currency + Analytics over HTTPS |
| API documentation | springdoc-openapi | — | `/swagger-ui.html` on both services |

## Key Technology Decisions

### Decision 1 — Jetpack Compose vs XML Views

Many stateful screens (transaction form, chart scrubbing, filter panels) with bilingual theming. Chose Compose + Material 3 exclusively. State-driven recomposition removes manual view invalidation; recomposition pitfalls are mitigated by `@Immutable` and `SharingStarted.WhileSubscribed(5_000)`.

### Decision 2 — Clean Architecture (MVVM) vs Standard Android MVC

Rate conversion, commission rules, and home-currency logic must not depend on Android or Retrofit. Chose a layered split into `core/domain`, `core/data`, `feature/*`. `ConvertCurrencyUseCase` depends on `CurrencyRateProvider` bound via Hilt `@Binds`, keeping domain unit-testable. Trade-off: mapper boilerplate between entity and domain.

### Decision 3 — Custom CERPS Backend vs Direct Third-Party Rate API

Free-tier providers have low quotas and occasional outages. Chose CERPS as the only contract the Android app depends on, aggregating three providers by median on an 8-hour scheduler. Shields the client from provider instability, centralises AES-256-GCM key management, hosts the AI commission lookup. Trade-off: an extra service to deploy and monitor.

### Decision 4 — Custom Canvas Chart vs YCharts for Rate History

YCharts produced artefacts on flat periods, clipped the last point, and used non-monotone interpolation. Chose to redraw `RateHistoryScreen` on raw `Canvas` with PCHIP interpolation and LTTB downsampling. Removes ghost dips, keeps the last point visible, supports scrubbing with exact rate and Δ%. YCharts kept for the pie chart.

## External Services

| Service | Purpose | Pricing |
|---|---|---|
| Fixer.io | Rate provider feeding the median aggregator | Free tier |
| ExchangeRatesAPI | Secondary rate provider in the aggregation | Free tier |
| CurrencyAPI | Third rate provider; improves resilience | Free tier |
| Google Gemini AI | Bank-commission lookup via `/api/v1/ai/bank-commission` | Free tier |
| Railway | Container hosting for both CERPS services | Free tier |
