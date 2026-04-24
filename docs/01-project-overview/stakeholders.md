# Stakeholders and Personas

## Stakeholder Analysis

| Stakeholder | Role | Influence on the Product | Level |
|---|---|---|---|
| End Users | Anyone who wants to understand and control their personal finances — from multi-currency travellers to families tracking household budgets in a single currency | Drive multi-currency support, offline mode, meaningful categories, and deliberate manual entry | High |
| Academic Committee | Evaluates the diploma project | Require architectural justification, documentation completeness, working functionality — motivates Clean Architecture, ADRs, Swagger on CERPS | High |
| Project Supervisor (Jan Bartnitsky) | Technical advisor and reviewer | Guides technical decisions and scope alignment with the diploma topic | High |
| Rate Providers (Fixer.io, ExchangeRatesAPI, CurrencyAPI) | Supply live rates via external APIs | Free-tier limits dictate the 8-hour refresh cadence, median-of-providers aggregation, and offline-first cached fallback | Medium |
| Google (Gemini AI) | LLM behind server-side bank commission lookup | Shapes the narrow endpoint, `{commission, found}` contract, and 10 req/min/IP throttle on `/api/v1/ai/bank-commission` | Low |

## User Personas

BudgetControl is designed for anyone who wants visibility over their spending. The primary target is people managing finances across multiple currencies and banks, but the application is equally useful for single-currency users who want category-level insight without automatic bank integration.

### Persona 1 — Student Abroad

**Demographics.** Age 18–26, living temporarily abroad (Poland, Czech Republic, Germany). Fixed monthly budget of 500–1000 EUR from stipend or parents. Mid-range Android device on API 26+.

**Pain points.** Cannot explain where the budget went by mid-month; bank apps show merchants, not categories; two or three cards with no consolidated view; foreign currency feels less real, driving unconscious overspending.

**Usage pattern.** Enters expenses manually after each purchase; reviews the pie chart weekly; uses offline mode frequently without roaming data.

### Persona 2 — Expat or Digital Nomad

**Demographics.** Age 25–40, living abroad with income in one currency and expenses in several. Income 1500–4000 EUR. Two or three cards with distinct commissions — a zero-fee fintech card (Revolut, Wise, N26) and a local-bank card at 2–3%.

**Pain points.** Needs the true post-commission cost of each foreign purchase; wants a consolidated balance across cards in one base currency; no existing tool allows per-transaction bank selection with real commission applied to the interbank rate.

**Usage pattern.** Picks a specific bank per foreign transaction; uses `USER_CORRECTED` to pin the real debit from the bank app; reviews rate history before large transfers; reconciles per-card statistics monthly.

### Persona 3 — Frequent Traveler

**Demographics.** Age 28–50, based at home with three to twelve international trips per year. Payment mix of card and cash from exchange offices.

**Pain points.** Bureau cash does not appear in any bank app; cannot compare a card-funded trip against a cash-funded one; foreign amounts feel less real, so overspending goes unnoticed until the trip is over.

**Usage pattern.** Uses Cash mode to enter the bureau rate manually (saved to `currency_exchanges`, reused as `CASH_EXCHANGE`); checks rate history before exchanging; records expenses daily during a trip.

### Persona 4 — Budget-Conscious Individual or Family

**Demographics.** Age 25–55, living in one country, primarily spending in one currency. May have occasional foreign purchases (online shopping, streaming subscriptions billed in USD/EUR, or rare travel). Single person or family managing a shared budget.

**Pain points.** No clear picture of where money goes each month; bank statements show merchant names not spending categories; subscriptions and small purchases add up unnoticed; wants a simple tool that does not require bank integration or cloud accounts.

**Usage pattern.** Records expenses daily or weekly; uses category breakdown to identify overspending; reviews monthly statistics to compare periods; appreciates offline mode and no subscription fee.
