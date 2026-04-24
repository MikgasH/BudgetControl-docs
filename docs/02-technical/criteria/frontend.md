# Criterion: Front-end

## Architecture Decision Record

### Status

Accepted — 2026-04-24

### Context

The rubric frames the front-end around a web SPA on Angular, React or Vue. BudgetControl is an Android application: target users (students abroad, expats, frequent travellers, and budget-conscious individuals) record expenses on the go, often offline, which rules out a browser-based SPA. Jetpack Compose is Google's modern declarative toolkit and is directly analogous to React — component-based, unidirectional data flow, state-driven recomposition — so the decision is framed as "Compose as the equivalent component-based declarative framework for the mobile target."

### Decision

Jetpack Compose + Material 3 with Clean Architecture (MVVM) and a feature-based module layout. Navigation Compose exposes 15+ routes grouped into five graphs (Main, Transaction, Analytics, Settings, Onboarding). Hilt wires ViewModels, repositories and Retrofit services; state flows from the domain layer to composables through StateFlow.

### Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|---|---|---|---|
| Android XML Views | Mature | Imperative; heavy boilerplate; no state-driven UI | Does not meet the declarative expectation |
| React Native | Cross-platform | Weaker Android integration; extra runtime | No iOS target; costs without benefits |
| Web SPA (React) | Matches rubric literally | Unsuited to on-the-go entry; poor offline | Wrong platform for mobile-first offline use case |

### Consequences

Positive: recomposition removes manual view invalidation; feature modules are self-contained; sealed classes give type-safe navigation. Negative: Compose tooling less mature than React; kapt slows builds. Neutral: rubric is satisfied by an equivalent rather than literal SPA.

### Implementation Details

```
app/src/main/java/com/budgetcontrol/
├── core/{domain, data, di, navigation, util}
├── feature/{main, transaction, analytics, settings, accounts, onboarding}
└── ui/components/        # shared Compose components
```

| Area | Decision |
|---|---|
| Reusable components | `PieChart`, `TransactionItem`, `AmountInputCard`, `CategorySelector`, `CurrencySelector`, `BankSelector`, `CustomColorPicker`, `DatePickerDialog`, `PeriodRangePicker` |
| State management | `StateFlow` + `collectAsStateWithLifecycle`; `SharingStarted.WhileSubscribed(5_000)` |
| API interaction | Centralised Retrofit in `NetworkModule`; `CerpsRepository` is the single data source |
| Error / loading | Offline banner from `NetworkStatusRepository`; 8-h stale-rate warning; per-screen loading states |
| Routing | 5 Nav-Compose graphs with type-safe `Screen` sealed routes |
| Type safety | Kotlin; `@Immutable` UI models; compile-time Room and Retrofit DSLs |

### Requirements Checklist

| # | Requirement | Status | Evidence |
|---|---|---|---|
| 1 | Component-based UI | Done | `ui/components/` + feature composables |
| 2 | Routing ≥ 3–4 | Done | 15+ routes across 5 graphs |
| 3 | Local + global state | Done | `remember` locally; `StateFlow` in ViewModels |
| 4 | Centralised API client | Done | `NetworkModule` + `CerpsRepository` |
| 5 | Error handling | Done | Offline/stale banners; loading states |
| 6 | Responsive layout | Done | Material 3 adaptive components |
| 7 | Forms + validation | Done | `TransactionFormScreen` + `ValidationHelper` |
| 8 | Filter / sort | Done | `UnifiedTransactionListScreen` (type/account/category/period) |
| 9 | Clean structure | Done | Feature modules + `core` / `ui` layers |
| 10 | UI library | Done | Material 3 + custom design system |
| 11 | Theme support | Done | Light / dark with first-launch auto-detection |
| 12 | HTTP error handling (401/403/404/500) | Done | CERPS `GlobalExceptionHandler` returns correct HTTP codes; Android Retrofit layer maps non-2xx responses to sealed `CerpsResult.Error`; offline state handled via `NetworkStatusRepository` |
| 13 | Code style enforcement | Done | Kotlin built-in formatter via Android Studio; OkHttp logging disabled in release builds via `BuildConfig.DEBUG` gate |

### Known Limitations

| Limitation | Impact | Potential Solution |
|---|---|---|
| No web or desktop client — the product is intentionally mobile-first; a web client is not planned for the current release | No browser access | — |
| No Compose UI instrumentation tests | Unit tests cover domain use cases and ViewModels; Compose layer tested manually | Add Espresso or Compose UI test runner before production |
| kapt instead of KSP | Slower builds; ~138 warnings | Migrate when Hilt's KSP processor stabilises |

### References

- Android repo — https://github.com/MikgasH/BudgetControl
- Jetpack Compose — https://developer.android.com/jetpack/compose
- Material 3 — https://m3.material.io/
