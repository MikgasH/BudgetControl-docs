# Problem and Goals

## Problem Statement

People holding cards from several banks across different countries — students abroad, expats, freelancers, frequent travellers — have no single tool that consolidates spending into one base currency while honestly accounting for bank commissions. Standard apps show a single bank only, statements list merchant names that do not map to meaningful categories, and spreadsheets require constant rate lookups without a clear summary. Foreign currency feels abstract at purchase, driving unconscious overspending: by mid-month the balance is materially lower than expected with no breakdown of where the money went.

## User Pain Points

| # | Pain point | Consequence |
|---|---|---|
| 1 | Cards from several banks, no consolidated view | Total spending is invisible |
| 2 | Merchant-based bank statements | One store visit covering groceries, medicine and clothing appears as a single entry |
| 3 | Foreign currency feels less real than home currency | Spending abroad exceeds the intended budget unconsciously |
| 4 | Manual rate lookups and commission math | Users give up on tracking |
| 5 | No historical context for past rates | Retrospective trip cost cannot be understood |
| 6 | Impulsive purchases go unrecorded | No moment of reflection between decision and expense |

## Business Goals and Objectives

| Goal | Measurable Objective |
|---|---|
| Accurate multi-currency tracking | Under 2% deviation from actual bank debit with correct bank selected |
| Offline-first operation | All core flows function without network connectivity |
| Meaningful spending visibility | Every transaction categorised by user-controlled categories, not merchants |
| Reduced friction during bank setup | AI-assisted commission lookup removes manual research |
| Conscious spending by design | Manual entry preserved instead of automatic import |

## Success Criteria

The project is successful when:

1. A user records an expense in any of the eighteen supported currencies, selects a bank, and sees the converted amount in the base currency with the bank's commission applied to the CERPS interbank rate.
2. The application works fully offline on cached data, with a visible warning when rates exceed eight hours of age.
3. Exchange-rate history renders as an interactive chart for any supported pair over 1D, 7D, 30D, 90D, 180D and 1Y, with scrubbing that reveals rate and percentage change.
4. Onboarding completes first-time configuration — language, theme, base currency, banks, favourite currencies, initial accounts — without external documentation.

## Key Performance Indicators

| KPI | Target |
|---|---|
| Conversion accuracy vs. bank debit | ≤ 2% for `BANK_AUTO`; 0% for `USER_CORRECTED` |
| Offline availability | 100% of core flows usable without network |
| Stale-rate warning | Triggered when cached rate age > 8 hours |
| Cold-start time | ≤ 3 s on a mid-range device (API 26+) |
| Rate-request latency | ≤ 500 ms under normal load |
| Rate refresh cadence | Every 8 hours via Currency Service scheduler |
| Localisation coverage | 100% of strings in English and Russian |
