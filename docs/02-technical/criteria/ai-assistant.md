# Criterion: AI Assistant

## Architecture Decision Record

### Status

Accepted — 2026-04-24

### Context

When a user adds a bank that is not in the 23 presets, they must supply the foreign-currency commission percentage. Manually researching that number on the bank's fee page is friction that, in practice, leads users to skip the field or enter a guess; either outcome breaks the conversion accuracy target of ≤ 2% deviation. An LLM-backed lookup removes that friction with a single tap. The overriding constraint is that the LLM API key must not ship inside the APK, where reverse engineering can recover it and abuse the free-tier quota.

### Decision

A narrow, server-side Gemini proxy on the Currency Service: `GET /api/v1/ai/bank-commission?bankName=X`. Android calls CERPS; CERPS calls Gemini. The proxy pins the model to `gemini-3.1-flash-lite-preview`, externalises every prompt to `application.yml`, validates input, applies a 10 req/min/IP filter, and returns a structured `BankCommissionResponse { commission: Double?, found: Boolean }`. Invocation flows from `BanksSection` and the onboarding bank step through `GeminiRepository` → `CerpsRepository.getBankCommission(bankName)`.

### Alternatives Considered

| Alternative | Pros | Cons | Why Not Chosen |
|---|---|---|---|
| Direct Gemini call from Android | Simplest wiring | Key exposed in APK; no throttling; no input filter | Security and abuse risk |
| Hardcoded commission table | Offline; predictable | Requires manual upkeep; fails on any bank outside the preset list | Defeats the point of an AI feature |
| Web scraping bank fee pages | Authoritative source | Fragile HTML; legal/ToS risk; per-bank parser | Unmaintainable for 23+ banks across regions |

### Consequences

Positive: key never leaves Railway; prompts are editable without an APK rebuild; strict `{commission, found}` contract keeps UI logic simple. Negative: one more hop on the critical path; free-tier Gemini latency is occasionally > 3 s. Neutral: the lookup is a focused tool, not a general chatbot.

### Implementation Details

```
currency-service/
├── controller/AiController.java          # GET /api/v1/ai/bank-commission
├── service/AiService.java                # validation + parsing
├── service/GeminiClient.java             # Spring RestClient, 10 s timeout
├── config/GeminiProperties.java          # @ConfigurationProperties("gemini")
├── filter/PublicRateLimitFilter.java     # 10 req/min/IP
└── resources/application.yml             # gemini.prompts.{system,bank-commission}
```

| Decision | Rationale |
|---|---|
| System prompt in `application.yml` | Operators tune prompts without redeploy or rebuild |
| Narrow `{commission, found}` DTO | UI renders a sealed `LookupState`; no freeform parsing on the client |
| Input charset filter | Unicode letters/digits/spaces/hyphens/apostrophes, ≤ 100 chars — blocks injection payloads |
| Model pinned to `gemini-3.1-flash-lite-preview` | Deterministic cost and latency; no silent model upgrades |
| `GeminiApiException` → 503 | Mapped by `GlobalExceptionHandler`; Android shows a clear retry banner |

**Prompt design.** The system prompt restricts the assistant to "answer only with the foreign-currency transaction commission for the named bank, as a single decimal number, or `NOT_FOUND`." The bank-commission prompt wraps the user input in a templated request. The server applies `(\d+\.?\d*)` to the response, and maps `NOT_FOUND` to `found = false`.

**UI states.** Android exposes a `LookupState` sealed class (`Loading`, `Success(Double)`, `NotFound`, `Error`) consumed by `BanksSection` and the onboarding bank step.

### Prompt Templates

The system uses two prompt layers configured in `application.yml`:

**System prompt** (`gemini.prompts.system`):
Restricts the model to bank commission topics only. Refuses any question outside foreign currency transaction fees.

**User prompt template** (`gemini.prompts.bank-commission`):
"For the bank named '{bankName}', what is the typical foreign currency transaction commission percentage? Return ONLY a single decimal number (e.g. 0.5 or 3.0). If unknown, return NOT_FOUND"

**Edge case template** (implicit, handled by `AiService`):
When `bankName` contains special characters or exceeds 100 chars, the request is rejected before reaching Gemini with HTTP 400.

**Commission context template** (`gemini.prompts.commission-context`, planned):
"For {bankName} charging {commission}% on foreign currency transactions, in one sentence explain what type of fee this is (for example: network markup, conversion spread, or flat transaction fee). Be brief and factual. Answer in {language}."

Planned behaviour:

- If Gemini returns a meaningful explanation → show as small gray text below the commission percentage in `BanksSection`
- If Gemini returns nothing useful or `NOT_FOUND` → show nothing, no fallback text
- Language parameter matches the app's current locale (`en` or `ru`)

### Example Interactions

| # | Input (bankName) | Gemini Response | App Output |
|---|---|---|---|
| 1 | Revolut | 0.5 | commission: 0.5, found: true |
| 2 | SEB | 3.0 | commission: 3.0, found: true |
| 3 | Wise | 0.5 | commission: 0.5, found: true |
| 4 | N26 | 0.0 | commission: 0.0, found: true |
| 5 | Monzo | 0.0 | commission: 0.0, found: true |
| 6 | Deutsche Bank | 2.0 | commission: 2.0, found: true |
| 7 | UnknownBankXYZ | NOT_FOUND | commission: null, found: false |
| 8 | `<script>alert(1)</script>` | HTTP 400 (validation) | Error: invalid input |
| 9 | [empty string] | HTTP 400 (validation) | Error: blank name |
| 10 | [101 character string] | HTTP 400 (validation) | Error: name too long |
| 11 | Revolut / commission-context (en) | "Standard Mastercard network markup applied on all non-EUR transactions" | Shown as subtitle below commission |
| 12 | SEB / commission-context (ru) | "Стандартная комиссия за конвертацию валюты через сеть Visa" | Shown as subtitle below commission |
| 13 | UnknownBankXYZ / commission-context | NOT_FOUND | Nothing shown, no error |

### Scope and Limitations

This assistant is intentionally narrow in scope. It answers one question: what commission does this bank charge for foreign currency transactions? It does not:

- Answer general banking questions
- Provide financial advice
- Remember previous lookups
- Support conversation history

The narrow scope is a deliberate product decision: the feature is a friction-reducer during bank setup, not a general assistant.

### Requirements Checklist

| # | Requirement | Status | Evidence |
|---|---|---|---|
| 1 | Modern LLM via API | Done | Gemini `gemini-3.1-flash-lite-preview` |
| 2 | Multi-layer architecture | Done | Android → CERPS → Gemini |
| 3 | System prompt defines behaviour | Done | `gemini.prompts.system` in `application.yml` |
| 4 | Prompt templates | Done | `gemini.prompts.system`, `gemini.prompts.bank-commission`, `gemini.prompts.commission-context` (planned) — three templates covering behaviour restriction, commission lookup, and commission context explanation |
| 5 | Input validation | Done | ≤ 100 chars; Unicode-letter/digit/space/hyphen filter |
| 6 | API error handling | Done | Timeout → 503; 429 surfaced via retry banner |
| 7 | API key in env only | Done | `GEMINI_API_KEY` lives only in Railway env |
| 8 | Rate limiting | Done | 10 req/min/IP via `PublicRateLimitFilter` |
| 9 | Prompt-injection protection | Done | Charset filter + narrow system prompt scope |
| 10 | API documentation | Done | Swagger on `/api/v1/ai/bank-commission` |
| 11 | Structured response | Done | `BankCommissionResponse { commission, found }` |
| 12 | Third prompt template (commission-context) | Planned | Defined in `application.yml` with `{bankName}`, `{commission}`, `{language}` parameters; UI shows subtitle below commission when explanation available, nothing when not — planned for next release |

### Known Limitations

| Limitation | Impact | Potential Solution |
|---|---|---|
| Single use case | No general chat surface | Add a broader assistant when a clear user need emerges |
| No conversation history | Each call is stateless | Persist a short session history when a dialog is introduced |
| Gemini free-tier cold latency > 3 s | Occasional slow first lookup | Warm-up ping on service startup or move to a paid tier |
| No fallback model | Gemini outage returns 503 | Add a secondary provider behind a circuit breaker |

### References

- Gemini API — https://ai.google.dev/gemini-api/docs
- Currency Swagger — https://supportive-vision-production.up.railway.app/swagger-ui.html
- CERPS repo — https://github.com/MikgasH/CERPS
