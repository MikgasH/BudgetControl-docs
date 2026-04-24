# Glossary

## General Terms

| Term | Definition |
|---|---|
| Expense | A money-out transaction recorded by the user, stored in `expenses` with `originalAmount` (as entered) and `amount` (in base currency after conversion). |
| Income | A money-in transaction recorded by the user, stored in `incomes` and structurally parallel to an expense (parity since Room migration 8 → 9). |
| Base currency | The single currency in which every Room `amount` field is stored and in which all aggregate balances are reported. Default is EUR; configurable in Settings. |
| Interbank rate | The wholesale market exchange rate between two currencies, returned by CERPS as the median of three providers. It contains no retail markup. |
| Bank commission | The percentage markup a specific issuing bank applies to foreign-currency card transactions on top of the interbank rate (for example, Revolut 0.5%, SEB 3.0%). |
| Stale rate | A cached exchange rate older than `STALE_THRESHOLD_MS` (8 hours). The app remains usable; transactions recorded against a stale rate are marked `rateSource = CACHED_RATE` and surface a warning. |
| rateSource | A discriminator stored on every multi-currency transaction that records how the rate was obtained: `BANK_AUTO`, `USER_CORRECTED`, `CASH_EXCHANGE`, `CACHED_RATE`, or `HOME_CURRENCY`. |
| Offline-first | A design principle requiring that recording, viewing, editing, and deleting transactions all work without network connectivity, falling back to the most recent cached rates. |
| Median aggregation | The CERPS scheduler queries three independent rate providers every 8 hours and persists the median value per currency pair, hiding individual provider outages and outliers. |

## Acronyms

| Acronym | Full Form | Description |
|---|---|---|
| API | Application Programming Interface | Contract that lets the Android client talk to CERPS over HTTPS. |
| CERPS | Currency Exchange Rate Processing Service | The Java 21 / Spring Boot 3 backend that aggregates rates and proxies the Gemini commission lookup. |
| UI | User Interface | The visual surface rendered by Jetpack Compose screens. |
| UX | User Experience | The overall flow and feel of the application, captured in the Refined UX criterion. |
| DI | Dependency Injection | Hilt-managed wiring of repositories, use cases, and ViewModels at compile time. |
| DAO | Data Access Object | Room-annotated interface defining typed SQL queries against an entity table. |
| DTO | Data Transfer Object | Plain object exchanged across the network or domain boundary, separate from Room entities and domain models. |
| MVVM | Model-View-ViewModel | The presentation pattern used by every Compose screen via Hilt-injected `ViewModel`s exposing `StateFlow`. |
| PCHIP | Piecewise Cubic Hermite Interpolating Polynomial | The Fritsch-Carlson monotone cubic interpolation used by `RateHistoryScreen` to avoid spurious dips. |
| LTTB | Largest-Triangle-Three-Buckets | The downsampling algorithm that reduces a rate series to a maximum of 100 points before drawing. |
| TTL | Time To Live | Lifetime of a cached value; for example, 8 hours for the in-memory rate cache and 15 minutes for the supported-currencies Caffeine cache. |
| JWT | JSON Web Token | Standard token format for authenticated APIs; not currently used by CERPS, which authenticates admin calls via `X-API-Key`. |
| FK | Foreign Key | Relational reference with `ON DELETE CASCADE` on every Room cross-table link. |
| PK | Primary Key | Unique identifier of a row; UUID strings in Room, BIGSERIAL in PostgreSQL. |
| ORM | Object-Relational Mapping | Translation between database rows and code objects: Room on Android, Spring Data JPA on CERPS. |
| SDK | Software Development Kit | The Android SDK target (35) and minimum (26) levels that bound Compose API availability. |
| JSON | JavaScript Object Notation | The wire format used by every CERPS endpoint and by DataStore-cached rate snapshots. |
| REST | Representational State Transfer | The architectural style of the CERPS HTTP API and the Analytics → Currency hop. |
| HTTP | Hypertext Transfer Protocol | The transport protocol; CERPS is exposed exclusively over HTTPS in production. |
| AES | Advanced Encryption Standard | Symmetric cipher used for at-rest encryption of provider API keys. |
| GCM | Galois/Counter Mode | Authenticated encryption mode used together with AES-256 for `api_provider_keys`. |

## Domain-Specific Terms

### Mobile (Android)

| Term | Definition |
|---|---|
| Room | Google's typed SQLite library; provides compile-time query checking, `Flow` queries, and the migration DSL used for the 14 migrations 3 → 17. |
| Hilt | Google's compile-time DI framework built on Dagger. Wires `DatabaseModule`, `RepositoryModule`, `NetworkModule`, and `UseCaseModule` at singleton scope. |
| Jetpack Compose | Declarative UI toolkit used exclusively for every screen and component; combined with Material 3 from BOM 2024.09. |
| DataStore | Jetpack key-value store used for preferences and the persistent EUR-based rate cache (Gson-serialized rate map plus timestamp). |
| StateFlow | Kotlin coroutine cold-to-hot stream produced by ViewModels; collected by Compose screens via `collectAsState()`. Most flows use `SharingStarted.WhileSubscribed(5_000)`. |
| Retrofit | Type-safe HTTP client backed by OkHttp; declares `CerpsApiService` and `CerpsAnalyticsApiService` with a 3-second timeout and logging disabled in release. |
| ViewModel | Hilt-injected `androidx.lifecycle.ViewModel` that owns UI state and survives configuration changes; one per screen feature. |
| Use Case | Single-responsibility class in `core/domain/usecase/` (for example, `AddTransactionUseCase`, `ConvertCurrencyUseCase`) that depends only on repository interfaces. |
| Entity | Room-annotated `@Entity` data class mapped to one SQLite table; nine entities total at v17. |
| Migration | A `Migration(from, to)` object registered in `DatabaseModule` that mutates SQLite schema between versions; `fallbackToDestructiveMigration()` is the safety net. |
| WindowSizeClass | Material 3 classification of the available width into `Compact` / `Medium` / `Expanded`. BudgetControl resolves it in `MainActivity` and exposes it through `LocalWindowWidthSizeClass`; the `Expanded` branch drives the tablet / foldable layouts in `MainScreen`, `SettingsScreen`, and `UnifiedTransactionListScreen`. |
| HiltTestRunner | Custom `AndroidJUnitRunner` that installs `HiltTestApplication` for instrumentation tests, paired with a `HiltTestActivity` (`@AndroidEntryPoint` `ComponentActivity` in `src/debug/`) so Compose UI tests can use `createAndroidComposeRule`. |
| `BANK_AUTO` | `rateSource` value used when the converted amount is computed as `interbankRate × (1 − commission/100)` for a card transaction. |
| `USER_CORRECTED` | `rateSource` value when the user overrides the computed amount with the exact figure from the bank app, yielding 100% accuracy. |
| `CASH_EXCHANGE` | `rateSource` value for transactions paid in cash; the rate comes from the most recent `CurrencyExchangeEntity` recorded by the user. |
| `CACHED_RATE` | `rateSource` value when conversion uses an in-memory or DataStore rate older than 8 hours. |
| `HOME_CURRENCY` | `rateSource` value when the transaction currency equals the account's native currency, so no conversion or commission applies. |

### Backend (CERPS)

| Term | Definition |
|---|---|
| Currency Service | The 8080 microservice that aggregates provider rates, persists history, manages encrypted provider keys, and proxies the Gemini commission lookup. |
| Analytics Service | The 8082 microservice that exposes `GET /api/v1/analytics/trends`; fetches data from Currency over REST and has no direct database access. |
| `cerps-common` | The shared Maven module that holds DTOs, the error model, and constants used by both services to keep request and response shapes consistent. |
| Liquibase | XML-based schema migration tool driving PostgreSQL evolution from changelogs v1.0 through v1.9 under `currency-service/resources/db/changelog/`. |
| Caffeine cache | High-performance in-process Java cache used by Analytics for trend results and by Currency for the supported-currencies list (TTL 15 minutes). |
| Resilience4j | Resilience library configured on `CurrencyServiceClient` for retry with exponential backoff (3 attempts) plus 5 s connect / 10 s read timeouts. |
| Scheduler | The Spring `@Scheduled` job inside Currency Service that fires every `28800000` ms (8 hours) to refresh `exchange_rates` from the three providers. |
| Provider key | An AES-256-GCM-encrypted row in `api_provider_keys` storing one provider's API token plus rotation metadata; managed via the admin endpoints. |
| Exchange rate bucket | A row in `exchange_rates` keyed by `(base_currency, target_currency, timestamp)`; same-hour buckets are joined via the functional index `idx_exchange_rates_hour`. |
| Cross-rate | A non-EUR pair (for example, USD → PLN) computed by joining two same-hour EUR-based buckets, since CERPS stores every rate against EUR. |
| AbstractRateLimitFilter | Shared `OncePerRequestFilter` in `cerps-common/filter/` that owns IP extraction (X-Forwarded-For with trusted-proxy normalisation), bucket lookup, TTL eviction, and the `429` response. `PublicEndpointRateLimitFilter` (Currency) and `TrendsEndpointRateLimitFilter` (Analytics) extend it and only declare path matchers and `RateLimitRule` sets. |
| LogstashEncoder | JSON encoder from `logstash-logback-encoder` 8.0 used in the `prod` Spring profile. The `dev` profile keeps a plain-text encoder with `correlationId`. Encoder selection lives in each service's `logback-spring.xml`, which includes the shared `cerps-common/logback-common.xml`. |
| JaCoCo | Java code coverage tool. The CERPS parent `pom.xml` registers a `jacoco-check` execution that fails the build when `LINE` coverage falls below 70%; the same gate runs locally via `./mvnw verify` and on every PR through `.github/workflows/ci.yml`. |
