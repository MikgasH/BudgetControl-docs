# Features and Requirements

## Epics

| ID | Epic | Scope |
|---|---|---|
| E1 | Transaction Management | Unified add/edit/delete, detail view, category-grouped and filterable lists |
| E2 | Multi-Currency and Bank Commission | Rate fetch, local conversion, five `RateSource` modes, cash mode |
| E3 | Accounts and Balances | Per-account currency, `AccountGroup` grouping, mixed-currency totals |
| E4 | Analytics and Insights | Dashboard, pie-chart statistics, rate-history chart with scrubbing |
| E5 | Configuration and Personalisation | Onboarding, bilingual UI, light/dark themes, category and bank management |
| E6 | CERPS Backend Services | Currency Service, Analytics Service, AI endpoint, Railway deployment |

## User Stories

| ID | As a… | I want to… | So that… | Pri. |
|---|---|---|---|---|
| US-01 | user with multiple cards | select my bank for a foreign expense | the correct commission is applied | Must |
| US-02 | user spending in several currencies | see all expenses in one base currency | I know my true total | Must |
| US-03 | traveller offline | add expenses without network | I do not miss transactions | Must |
| US-04 | user paying cash abroad | enter the bureau rate manually | I get an accurate conversion | Must |
| US-05 | user curious about rates | view an interactive rate-history chart | I see rate dynamics | Should |
| US-06 | new user | complete onboarding of banks, currencies, accounts | the app is configured immediately | Must |
| US-07 | user adding a bank | let AI look up the commission rate | I do not research it myself | Should |
| US-08 | user with multiple accounts | group accounts and see per-account plus total balance | wallets stay organised | Should |
| US-09 | user reviewing spending | filter a unified list by type, account, category, period | I answer ad-hoc questions | Should |
| US-10 | user with matching account and expense currency | skip the commission step | no fake conversion when none is needed | Must |

## Use Cases

**User (primary).** Records, edits and deletes expenses/incomes; selects bank for foreign entries; switches Card/Cash mode; views dashboard, statistics and rate-history charts; manages accounts, banks, categories, favourite currencies; completes onboarding; triggers AI commission lookup.

**CERPS Administrator (secondary).** Refreshes rates, manages provider keys, adds supported currencies, forces Analytics cache refresh. All actions require `X-API-Key` and per-IP rate limiting.

**External actors.** Fixer.io, ExchangeRatesAPI, CurrencyAPI feed quotes to the scheduler (median aggregation); Gemini answers commission lookups via `GET /api/v1/ai/bank-commission`.

## Non-Functional Requirements

| Category | Requirement | Target |
|---|---|---|
| Performance | Cold-start on a mid-range device (API 26+) | ≤ 3 s |
| Performance | CERPS rate-request latency | ≤ 500 ms |
| Availability | Core flows without network | 100% offline |
| Reliability | Rate-provider failure | Fallback; median of successful providers |
| Security | Provider keys at rest | AES-256-GCM, master key in env |
| Security | Admin endpoints | `X-API-Key` + per-IP rate limit |
| Security | AI endpoint | Input validation + 10 req/min/IP |
| Scalability | Rate lookups per transaction | Zero network calls (in-memory cache) |
| Usability | UI languages | EN + RU, auto-detected at first launch |
| Usability | Minimum Android | 8.0 (API 26) |
| Maintainability | Android architecture | Clean Architecture (MVVM) |
