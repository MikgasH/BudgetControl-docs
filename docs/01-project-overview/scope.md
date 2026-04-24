# Scope

## In-Scope

### Android Application

- Manual entry, edit and deletion of expenses and incomes via a unified `TransactionFormScreen`.
- Multi-currency conversion with bank commission applied to the CERPS interbank rate; five rate-source modes (`BANK_AUTO`, `USER_CORRECTED`, `CASH_EXCHANGE`, `CACHED_RATE`, `HOME_CURRENCY`).
- Cash mode with manual bureau-rate entry persisted to `currency_exchanges`.
- Accounts with native currency, `AccountGroup` grouping, and `"~"` prefix on mixed totals.
- 16 expense + 8 income preset categories with localisation keys, plus user-defined categories.
- 23 preset banks with favourites, defaults and user entries.
- Dashboard, pie-chart statistics, and a unified transaction list with multi-select filters.
- Rate-history chart (Canvas, PCHIP, LTTB) over 1D–1Y.
- Seven-step onboarding and offline-first three-level rate cache with stale-rate warnings after 8 hours.
- English/Russian and light/dark, with automatic first-launch detection.

### CERPS Backend

- Currency Service: median aggregation from three providers, 8-hour scheduler, PostgreSQL, AES-256-GCM encrypted provider keys, admin endpoints.
- Analytics Service: trend endpoint (1D–1Y), Caffeine cache, Resilience4j retry.
- AI endpoint `GET /api/v1/ai/bank-commission` proxying Gemini with externalised prompts, input validation and 10 req/min/IP throttle.
- OpenAPI/Swagger on both services, global error handling, Railway.

## Out-of-Scope

- Automatic import from bank push notifications — planned future feature; manual entry preserved as the primary deliberate interaction.
- Real-time rate alerts and push notifications.
- Investment or portfolio tracking.
- Family or shared budgets.
- Google or third-party authentication and cloud backup.
- Transfers between accounts.
- iOS, web or desktop clients.
- 1Y period on rate history chart (maximum supported period is 180D in current release).

## Constraints

- Android only; no other client platforms.
- Rate provider free-tier quotas dictate the 8-hour refresh cadence.
- Gemini free-tier limits shape the UX of the commission lookup.
- Internet is required for the first rate load; subsequent use may be fully offline.
- Railway free-tier hosting sets resource ceilings for CERPS.

## Assumptions

- Users run Android 8.0 (API 26) or higher.
- The user picks a base currency during onboarding; all amounts are stored in it.
- Cross-currency rates are derived mathematically through EUR inside CERPS.
- CERPS is deployed to Railway before the defence, with URLs pinned in `local.properties`.
- Bank commission presets reflect typical retail rates and are user-editable.
