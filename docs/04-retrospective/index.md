# Retrospective

An honest look at what worked, what did not, what remains, and what was learned while building BudgetControl.

## What Went Well

- **CERPS was more reliable than any single provider.** Median aggregation across Fixer.io, ExchangeRatesAPI and CurrencyAPI hid outages that individually would have empty-stated the app.
- **Clean Architecture made features independently testable.** `ConvertCurrencyUseCase` depends on a `CurrencyRateProvider` interface, not Retrofit, so conversion and commission logic run without a network, Android or Room.
- **The Canvas rate-history rewrite fixed two bugs at once.** Canvas + PCHIP + LTTB replaced YCharts, removing the clipped last point and the ghost dips from Catmull-Rom, and added scrubbing as a side effect.
- **Offline-first with the three-level cache (memory → DataStore → CERPS) works seamlessly.** Recording expenses in airplane mode and still getting correct conversions is the promise of the app, and turned out simpler than alternatives.
- **Moving the Gemini key server-side materially improved security.** The AI endpoint sits behind input validation, rate limiting and a narrow system prompt — none of which were possible with the earlier direct-from-Android call.
- **Database hardening — unique constraint on `exchange_rates` (v1.9) and role separation (v1.8) applied before defence.**

## What Did Not Work as Planned

- **No CI/CD pipeline.** Railway deploys are manual; no automated gate catches a broken commit.
- **No Figma prototype.** The UX evolved in code; reviewers see the running app instead of a clickable prototype.
- **Persistence not unified.** `ExpenseEntity` and `IncomeEntity` remain parallel; the Stage C merge was deferred.
- **No formal usability testing.** Friction was surfaced through personal use, weaker than a moderated session with the target personas.

## Technical Debt

- kapt still in use for Hilt and Room — migration to KSP pending; contributes to ~138 Kotlin warnings.
- Split `ExpenseEntity` / `IncomeEntity` — parallel DAOs and mappers; unified `TransactionEntity` with `kind` discriminator deferred until inter-account transfers enter scope.

## Lessons Learned

- **Audit before refactor.** The v8/v9 audits catalogued concrete issues (cache-eviction race, SpEL in `@Cacheable`, clipped chart, recomposition costs) that a broad rewrite would have missed.
- **Secrets off the device from day one.** Shipping Gemini calls from Android was a shortcut with a large security cost; server-side should have been the first design.
- **Caching is cheap; invalidation is the work.** Value came from treating "stale" as a first-class UI state, not hiding it.
- **Write ADRs during development.** Decisions captured while fresh read cleanly; reconstructed ones need archaeology.

## Future Work

- Spending limits per category with live remaining-budget previews.
- Recurring transactions through WorkManager.
- GitHub Actions: build, JaCoCo, SonarCloud gate, Docker push, Railway deploy.
- Unified `TransactionEntity` once transfers between accounts enter scope.
- JSON export and import via `ActivityResultContracts`.
