# Database Schema

## Overview

BudgetControl operates two independent data stores. The Android client owns a Room SQLite database for fully offline transaction recording; the CERPS Currency Service owns a PostgreSQL 16 database for rate history and encrypted provider keys. The Analytics Service has no database of its own and reads through the Currency Service over REST.

| Attribute | Mobile (Android) | Backend (CERPS) |
|---|---|---|
| Database | SQLite (Room v17) | PostgreSQL 16 |
| ORM | Room 2.6.1 | Spring Data JPA |
| Migration tool | Room `Migration` objects (3 → 17) | Liquibase XML changelogs (v1.0 → v1.9) |
| Normalization | Denormalized for offline read paths | 3NF |

## Android Room (SQLite v17)

The Room database lives in `core/data/local/` and exposes nine entities. Foreign keys cascade on delete; every monetary column is stored in **base currency** (default EUR) so that aggregations work without per-row conversion.

### `expenses`

Records money-out transactions.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | UUID generated client-side |
| amount | REAL | NOT NULL | Amount in base currency after conversion |
| categoryId | TEXT | NOT NULL, FK → categories.id ON DELETE CASCADE | Owning category |
| description | TEXT | NULLABLE | Free-form note |
| date | INTEGER | NOT NULL | Unix epoch milliseconds; indexed |
| createdAt | INTEGER | NOT NULL | Insertion timestamp |
| originalAmount | REAL | NOT NULL | Amount as entered by the user |
| originalCurrency | TEXT | NOT NULL | Currency code as entered |
| exchangeRate | REAL | NULLABLE | Rate used at conversion time |
| bankName | TEXT | NULLABLE | Issuing bank name when paid by card |
| bankCommission | REAL | NULLABLE | Applied commission percentage |
| rateSource | TEXT | NULLABLE | `BANK_AUTO` / `USER_CORRECTED` / `CASH_EXCHANGE` / `CACHED_RATE` / `HOME_CURRENCY` |
| accountId | TEXT | NULLABLE, FK → accounts.id | Source account |

Indexes:
- `idx_expenses_categoryId` on `categoryId`
- `idx_expenses_date` on `date`
- `idx_expenses_accountId` on `accountId` (added v15 → v16)

### `incomes`

Records money-in transactions; structurally parallel to `expenses` since migration 8 → 9.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | UUID |
| amount | REAL | NOT NULL | Amount in base currency |
| categoryId | TEXT | NOT NULL, FK → categories.id ON DELETE CASCADE | Owning category |
| description | TEXT | NULLABLE | Free-form note |
| date | INTEGER | NOT NULL | Unix epoch ms; indexed |
| createdAt | INTEGER | NOT NULL | Insertion timestamp |
| originalAmount | REAL | NOT NULL | Amount as entered |
| originalCurrency | TEXT | NOT NULL | Currency code as entered |
| exchangeRate | REAL | NULLABLE | Rate used at conversion time |
| bankName | TEXT | NULLABLE | Issuing bank when applicable |
| bankCommission | REAL | NULLABLE | Applied commission percentage |
| rateSource | TEXT | NULLABLE | Same enumeration as `expenses.rateSource` |
| accountId | TEXT | NULLABLE, FK → accounts.id | Destination account |

Indexes:
- `idx_incomes_categoryId` on `categoryId`
- `idx_incomes_date` on `date`
- `idx_incomes_accountId` on `accountId` (added v15 → v16)

### `categories`

Holds the 24 default categories (16 expense + 8 income) plus user-created ones.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | Stable id (`cat_groceries`, UUID for custom) |
| name | TEXT | NOT NULL | Display name |
| iconName | TEXT | NOT NULL | Material icon key (50+ supported) |
| color | TEXT | NOT NULL | Hex string |
| isDefault | INTEGER | NOT NULL DEFAULT 0 | Boolean flag for prepopulated rows |
| type | TEXT | NOT NULL DEFAULT `'EXPENSE'` | `EXPENSE` or `INCOME` |
| nameKey | TEXT | NULLABLE | Localization key in `strings.xml` |
| isSystem | INTEGER | NOT NULL DEFAULT 0 | System categories cannot be deleted |
| usageCount | INTEGER | NOT NULL DEFAULT 0 | Drives sort by frequency |

Indexes:
- `idx_categories_type` on `type` (added v11 → v12)

### `banks`

Holds the 23 prepopulated banks plus user-created ones.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | INTEGER | PK AUTOINCREMENT | Surrogate key |
| name | TEXT | NOT NULL UNIQUE | Bank display name |
| commissionPercent | REAL | NOT NULL | Commission applied to foreign-currency card transactions |
| isDefault | INTEGER | NOT NULL DEFAULT 0 | Default bank for new transactions |
| isFavorite | INTEGER | NOT NULL DEFAULT 0 | Pinned to the top of the picker |

Indexes:
- `idx_banks_isFavorite` on `isFavorite` (added v10 → v11)

### `currency_exchanges`

Records cash exchange-office transactions; the most recent row per pair becomes the rate for `CASH_EXCHANGE` transactions.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | UUID |
| fromAmount | REAL | NOT NULL | Amount sold |
| fromCurrency | TEXT | NOT NULL | Sold currency |
| toAmount | REAL | NOT NULL | Amount received |
| toCurrency | TEXT | NOT NULL | Received currency |
| exchangeRate | REAL | NOT NULL | `toAmount / fromAmount` |
| location | TEXT | NULLABLE | Free-form description (for example, "exchange office at the station") |
| date | INTEGER | NOT NULL | Unix epoch ms; indexed |
| createdAt | INTEGER | NOT NULL | Insertion timestamp |

Indexes:
- `idx_currency_exchanges_date` on `date`
- `idx_currency_exchanges_pair` on `(fromCurrency, toCurrency)`

### `accounts`

Per-account balances supporting multi-currency tracking.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | UUID |
| name | TEXT | NOT NULL | Display name |
| currency | TEXT | NOT NULL | Native currency of the account (for example, `PLN`) |
| color | TEXT | NOT NULL | Hex color for UI |
| iconName | TEXT | NOT NULL | Material icon key |
| createdAt | INTEGER | NOT NULL | Insertion timestamp |

### `account_groups`

Groups accounts together for combined views.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | TEXT | PK | UUID |
| name | TEXT | NOT NULL | Display name |
| color | TEXT | NOT NULL | Hex color |
| createdAt | INTEGER | NOT NULL | Insertion timestamp |

### `account_group_members`

Junction table linking accounts to groups.

| Column | Type | Constraints | Description |
|---|---|---|---|
| groupId | TEXT | PK (composite), FK → account_groups.id ON DELETE CASCADE | Group reference |
| accountId | TEXT | PK (composite), FK → accounts.id ON DELETE CASCADE | Account reference |

### `category_limits`

Per-category monthly spending ceiling, added in v17.

| Column | Type | Constraints | Description |
|---|---|---|---|
| categoryId | TEXT | PK, FK → categories.id ON DELETE CASCADE | One limit per category |
| amount | REAL | NOT NULL | Limit in base currency |
| periodType | TEXT | NOT NULL DEFAULT `'MONTH'` | Reset cadence |
| createdAt | INTEGER | NOT NULL | Insertion timestamp |
| updatedAt | INTEGER | NOT NULL | Last edit timestamp |

## CERPS PostgreSQL 16

Owned by the Currency Service; normalized to 3NF.

### `supported_currencies`

The 18 currencies CERPS aggregates rates for.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PK | Surrogate key |
| code | VARCHAR(3) | NOT NULL UNIQUE | ISO 4217 code (for example, `EUR`) |
| name | VARCHAR(64) | NOT NULL | Display name |
| active | BOOLEAN | NOT NULL DEFAULT TRUE | Soft-disable flag |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL DEFAULT NOW() | Insertion timestamp |

### `exchange_rates`

Median-aggregated rate history. The 8-hour scheduler writes one row per pair per tick. Rows older than 395 days are pruned.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PK | Surrogate key |
| base_currency | VARCHAR(3) | NOT NULL | Always `EUR` (cross-rates derived) |
| target_currency | VARCHAR(3) | NOT NULL | ISO 4217 |
| rate | NUMERIC(18, 8) | NOT NULL | Median across providers |
| timestamp | TIMESTAMP WITH TIME ZONE | NOT NULL | Tick timestamp |
| provider_count | SMALLINT | NOT NULL | Number of providers that contributed |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL DEFAULT NOW() | Insertion timestamp |

Constraints:
- `UNIQUE (base_currency, target_currency, timestamp)` — prevents duplicates from overlapping scheduler ticks (Liquibase v1.9)

Indexes:
- `idx_latest_rate (base_currency, target_currency, timestamp DESC)` — backs latest-rate lookups
- `idx_timestamp (timestamp)` — backs retention pruning
- `idx_exchange_rates_hour` — functional index on `date_trunc('hour', timestamp)` for same-hour bucket joins used by cross-rates (Liquibase v1.7)

### `api_provider_keys`

AES-256-GCM-encrypted provider API tokens.

| Column | Type | Constraints | Description |
|---|---|---|---|
| id | BIGSERIAL | PK | Surrogate key |
| provider_name | VARCHAR(64) | NOT NULL | `Fixer.io`, `ExchangeRatesAPI`, `CurrencyAPI` |
| encrypted_key | BYTEA | NOT NULL | AES-256-GCM ciphertext |
| iv | BYTEA | NOT NULL | 12-byte initialization vector |
| auth_tag | BYTEA | NOT NULL | GCM authentication tag |
| active | BOOLEAN | NOT NULL DEFAULT TRUE | Deactivation flag (no row deletion) |
| created_at | TIMESTAMP WITH TIME ZONE | NOT NULL DEFAULT NOW() | Insertion timestamp |
| rotated_at | TIMESTAMP WITH TIME ZONE | NULLABLE | Set when the key is rotated |

Without the Railway-held master key, every encrypted row is unusable.

## Relationships

| Relationship | Type | Description |
|---|---|---|
| categories → expenses | One-to-Many | A category owns many expenses; `ON DELETE CASCADE` removes the expense row |
| categories → incomes | One-to-Many | Same cascade rule for incomes |
| categories → category_limits | One-to-One | At most one monthly limit per category, cascading delete |
| accounts → expenses | One-to-Many | An account owns its outgoing transactions |
| accounts → incomes | One-to-Many | An account owns its incoming transactions |
| account_groups → accounts | Many-to-Many | Through `account_group_members` (composite PK, double cascade) |
| supported_currencies → exchange_rates | One-to-Many (logical) | A currency code drives many rate rows; enforced at application level, not by FK |

## Migrations

### Room (Android, 3 → 17)

| Version | Description |
|---|---|
| 3 → 4 | Add `originalAmount`, `originalCurrency`, `exchangeRate` to `expenses` |
| 4 → 5 | Add `bankName`, `bankCommission` to `expenses`; create `banks` |
| 5 → 6 | Add `rateSource` to `expenses` |
| 6 → 7 | Add `isFavorite` to `banks` |
| 7 → 8 | Add `nameKey`, `isSystem`, `usageCount` to `categories` |
| 8 → 9 | Mirror expense fields onto `incomes` (`originalAmount`, `originalCurrency`, `exchangeRate`, `bankName`, `bankCommission`, `rateSource`) |
| 9 → 10 | Create `currency_exchanges` |
| 10 → 11 | Add indexes on `expenses.date`, `incomes.date`, `currency_exchanges.date`, `currency_exchanges(fromCurrency, toCurrency)`, `banks.isFavorite` |
| 11 → 12 | Add index on `categories.type` |
| 12 → 13 | Add `accountId` columns to `expenses` and `incomes` |
| 13 → 14 | Create `accounts`, `account_groups`, `account_group_members` |
| 14 → 15 | Schema refinements on account-related tables |
| 15 → 16 | Add `Index("accountId")` on `expenses` and `incomes` for per-account filter pushdown |
| 16 → 17 | Create `category_limits` (FK → categories CASCADE) |

### Liquibase (PostgreSQL, v1.0 → v1.9)

| Version | Description |
|---|---|
| v1.0 | Initial schema: `supported_currencies`, `exchange_rates`, `api_provider_keys` |
| v1.1 | Seed `supported_currencies` with the 18 default codes |
| v1.2 | Add `idx_latest_rate` and `idx_timestamp` on `exchange_rates` |
| v1.3 | Add `provider_count` to `exchange_rates` |
| v1.4 | Add `rotated_at` and rotation metadata on `api_provider_keys` |
| v1.5 | Tighten `NOT NULL` and length constraints; widen `rate` to `NUMERIC(18,8)` |
| v1.6 | Drop redundant `idx_trends` (covered by `idx_latest_rate`) |
| v1.7 | Add functional index `idx_exchange_rates_hour` on `date_trunc('hour', timestamp)` |
| v1.8 | Define database roles `app_read`, `app_write`, `app_admin` |
| v1.9 | Add `UNIQUE (base_currency, target_currency, timestamp)` on `exchange_rates` |

## Seeding

### Room (Android)

`AppDatabase` registers a `RoomDatabase.Callback` that runs on `onCreate` and `onOpen` (when tables are empty):

- **23 banks** inserted from `DEFAULT_BANKS` (Revolut, Wise, SEB, N26, Monzo, Starling, bunq, DKB, ING, Deutsche Bank, BBVA, Handelsbanken, LHV, Nordea, Luminor, Citadele, PKO, mBank, Česká spořitelna, ČSOB, Santander ES, Swedbank, Santander PL).
- **24 default categories** (16 expense + 8 income) initialized in `BudgetControlApplication.onCreate()` via `CategoryRepository.initializeDefaultCategories()`.

### Liquibase (PostgreSQL)

The v1.1 changeset seeds `supported_currencies` with the 18 default codes:

```
EUR, USD, GBP, CHF, JPY, PLN, CZK, HUF, BYN, UAH,
GEL, SEK, NOK, DKK, TRY, CAD, AUD, NZD
```

Provider API keys are not seeded — they are inserted at runtime through `POST /api/v1/admin/provider-keys` so that plaintext tokens never enter the changelog.
