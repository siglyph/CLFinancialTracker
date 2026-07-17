# Chateau Leonor Financial Tracker — Design

Income and expense tracking for Chateau Leonor Resort. Replaces hand-kept books:
every peso in or out is recorded, receipts are scanned from a phone camera into
itemized entries, the POS daily sales report is imported instead of retyped,
and daily/monthly reports show exactly where money went.

**Division of labor:** Jude writes the Spring Boot backend (guided, commit by
commit). Claude builds the frontend and reviews backend work.

## Settled decisions

| Decision | Choice | Notes |
|---|---|---|
| Backend | Java 21 + Spring Boot 3.x (Maven) | Written by Jude |
| Frontend | React + Vite + TypeScript, mobile-first web app | Written by Claude; phone camera via `<input type="file" accept="image/*" capture="environment">` |
| Database | PostgreSQL | Docker locally; managed or containerized in prod |
| Hosting | Cloud (any small VPS — Hetzner, DigitalOcean, Lightsail) | Staff phones reach it over WiFi/data |
| Receipt extraction | **Claude vision API** (Anthropic Messages API, structured outputs) | Model `claude-opus-4-8` by default (config property); ~₱0.30–₱1.50 per receipt depending on tier. Returns itemized JSON with suggested categories in one call |
| Income ingestion | **POS daily-report CSV import** + manual entry | The resort's POS exports an end-of-day report (see `docs/samples/`); importing it beats retyping |
| Users | Family only (2–3 people) | Single ADMIN role, form login + session; added at deploy time |
| Currency | PHP only, stored as **integer centavos** (`BIGINT`) | Never float/double |
| Timezone | `Asia/Manila` everywhere | Business date is a `LocalDate`, not a timestamp |

## Architecture

```
 phone / laptop browser
        │  HTTPS
        ▼
 ┌─────────────────────────── VPS ───────────────────────────┐
 │  Caddy (TLS, serves frontend static files, proxies /api)  │
 │        │                                                   │
 │        ▼                                                   │
 │  Spring Boot API ──── PostgreSQL                           │
 │        │        └──── local disk: receipt images,          │
 │        │                          imported POS reports     │
 └────────┼───────────────────────────────────────────────────┘
          ▼
   Claude API  (vision + structured outputs)
```

The backend is the only thing that talks to the Claude API; the
`ANTHROPIC_API_KEY` lives in an environment variable on the server, never in
the repo or the frontend.

## Data model

`transactions` is the single source of truth: one row per peso movement.
Manual entries create a row directly; a confirmed receipt creates one row per
line item; a confirmed POS report creates one row per item line. Reports are
`GROUP BY` queries over this one table.

```sql
CREATE TABLE categories (
  id        BIGSERIAL PRIMARY KEY,
  name      VARCHAR(100) NOT NULL,
  type      VARCHAR(10)  NOT NULL CHECK (type IN ('INCOME', 'EXPENSE')),
  parent_id BIGINT REFERENCES categories(id),   -- tree: "Food > Employees"
  archived  BOOLEAN NOT NULL DEFAULT FALSE
);

CREATE TABLE receipts (
  id                    BIGSERIAL PRIMARY KEY,
  image_path            VARCHAR(500) NOT NULL,
  status                VARCHAR(20) NOT NULL DEFAULT 'PENDING_REVIEW',
                        -- PENDING_REVIEW | CONFIRMED | REJECTED
  merchant_name         VARCHAR(200),
  receipt_date          DATE,
  parsed_total_centavos BIGINT,
  raw_extraction        JSONB,                  -- full extractor response, for audit/debug
  created_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE receipt_line_items (
  id                    BIGSERIAL PRIMARY KEY,
  receipt_id            BIGINT NOT NULL REFERENCES receipts(id),
  description           VARCHAR(300) NOT NULL,
  quantity              NUMERIC(10,3) NOT NULL DEFAULT 1,
  unit_price_centavos   BIGINT,
  line_total_centavos   BIGINT NOT NULL,
  suggested_category_id BIGINT REFERENCES categories(id),
  uncertain             BOOLEAN NOT NULL DEFAULT FALSE  -- extractor flagged as hard to read
);

CREATE TABLE sales_reports (
  id                BIGSERIAL PRIMARY KEY,
  file_path         VARCHAR(500) NOT NULL,      -- original CSV, kept for audit
  business_date     DATE NOT NULL,              -- from the report's From/To lines
  gross_centavos    BIGINT,
  discount_centavos BIGINT,                     -- e.g. Senior/PWD discounts
  net_centavos      BIGINT,
  cash_centavos     BIGINT,                     -- payment split, for drawer
  gcash_centavos    BIGINT,                     --   reconciliation
  status            VARCHAR(20) NOT NULL DEFAULT 'PENDING_REVIEW',
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pos_item_mappings (
  item_name   VARCHAR(200) PRIMARY KEY,         -- POS item, e.g. "Evening: Adult 180"
  category_id BIGINT NOT NULL REFERENCES categories(id)
);

CREATE TABLE transactions (
  id                   BIGSERIAL PRIMARY KEY,
  type                 VARCHAR(10) NOT NULL CHECK (type IN ('INCOME', 'EXPENSE')),
  business_date        DATE NOT NULL,
  amount_centavos      BIGINT NOT NULL CHECK (amount_centavos > 0),
  category_id          BIGINT NOT NULL REFERENCES categories(id),
  description          VARCHAR(300),
  receipt_line_item_id BIGINT REFERENCES receipt_line_items(id),
  sales_report_id      BIGINT REFERENCES sales_reports(id),
  -- both source refs null = manual entry; never both set
  created_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tx_business_date ON transactions (business_date);
CREATE INDEX idx_tx_category      ON transactions (category_id);
```

Design rules encoded here:

- **Amounts are always positive**; `type` carries direction. Simpler to reason
  about, and the CHECK constraint makes corrupt data impossible.
- **Category on the line item, not the receipt** — one store run can contain
  pool chlorine and employee food, and itemized scanning lets us split it.
- **Categories are archived, never deleted** — old transactions must keep
  pointing at the category they were booked under.
- **`raw_extraction` keeps the full extractor response** so mistakes in our
  mapping code can be diagnosed (and re-parsed) without rescanning.
- **Original files (receipt photos, POS CSVs) are always kept** — audit trail.

### Starter category tree (seeded via Flyway, editable in the app)

Derived from Jude's list plus the actual POS report items
(`docs/samples/Report_2026_06_19.csv`):

```
INCOME                          EXPENSE
├── Entrance fees               ├── Food
│   ├── Morning                 │   ├── Employees
│   ├── Evening                 │   └── Personal
│   └── Kids                    ├── Resort Expenses
├── Rentals                     │   ├── Chlorine for Pool
│   ├── Kubo                    │   └── Appliances
│   ├── Tent                    ├── Utilities
│   ├── Umbrella / table        ├── Gasoline / Car expenses
│   └── Videoke                 └── Store stock (resort store purchases)
├── Kiosk (Restaurant)
│   ├── Fish and chips
│   ├── Burger fries
│   └── Halo-Halo
└── Store sales
```

## Receipt pipeline (expenses)

```
photo → POST /api/receipts (multipart)
      → save image to disk
      → Claude API: image + active category list + JSON schema
      → validated JSON: merchant, date, total, line items — each with a
        suggested category and an `uncertain` flag
      → status = PENDING_REVIEW
      → human reviews on phone: fix text/amounts, confirm categories
      → POST /api/receipts/{id}/confirm → one transaction per line item
```

**The review step is non-negotiable.** Extraction can misread a digit; a wrong
amount silently written into the books is worse than manual entry. Nothing
reaches `transactions` without a human confirming it. The review UI highlights
lines the extractor marked `uncertain` and flags when the line items don't sum
to the parsed total.

Extraction specifics:

- Official Java SDK (`com.anthropic:anthropic-java`). Model is a config
  property, default `claude-opus-4-8` (most accurate; `claude-haiku-4-5` is
  the cheaper tier if cost ever matters). The photo goes in as a base64 image
  content block; **structured outputs** constrain the response to our schema,
  so the SDK returns a typed `ExtractedReceipt` record — no JSON wrangling.
- The schema requests amounts **as integer centavos**, and the prompt includes
  the active category tree so each line item arrives with a suggested
  category. The server still validates everything: suggested category IDs
  must exist and be `EXPENSE`-typed, line totals should sum to the receipt
  total (mismatch ⇒ flagged in review), dates must be plausible.
- Behind an interface:

```java
public interface ReceiptExtractionService {
    ExtractedReceipt extract(byte[] imageBytes, List<CategoryRef> categories);
}
// ClaudeReceiptExtractionService implements it.
// Tests use a fake. If we ever want a different provider (Textract,
// Document AI, ...), the replacement slots in without touching anything else.
```

## POS daily-report import (income)

The resort's POS already exports an end-of-day CSV (sample:
`docs/samples/Report_2026_06_19.csv`) with sections for Sales, Payment
(Cash / GCash), Discount (Senior or PWD), Expense, and an itemized breakdown
(item name, count, amount). Importing it replaces manual re-entry of a whole
day's income:

```
CSV upload → POST /api/sales-reports (multipart)
           → parse sections; business_date from the "From:" line
           → map item lines via pos_item_mappings (unmapped ⇒ user assigns)
           → preview: PENDING_REVIEW
           → human confirms on the review screen
           → POST /api/sales-reports/{id}/confirm
           → one INCOME transaction per item line, dated business_date
```

Parsing rules learned from the sample:

- The file starts with a **UTF-8 BOM** — strip it before parsing, or the
  first field silently fails to match.
- **Trust the report's amounts, don't recompute** `count × list price` —
  item amounts are net of per-item discounts and won't always divide evenly.
- One transaction per item line, description like `"Evening: Adult 180 ×141"`;
  the payment split and discount totals are stored on `sales_reports` for
  drawer reconciliation, not as transactions.
- **The app remembers mappings**: the first import asks the user to assign
  each POS item name to a category; `pos_item_mappings` pre-fills every
  subsequent import, so day 2 onward is upload → glance → confirm.
- Importing a second report for the same `business_date` warns (but doesn't
  block — a resort can have more than one register).

## API contract (v1)

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/categories` | Full tree (both types) |
| POST | `/api/categories` | Create (name, type, parentId) |
| PATCH | `/api/categories/{id}` | Rename / archive |
| GET | `/api/transactions?from=&to=&categoryId=&type=` | List entries |
| POST | `/api/transactions` | Manual entry |
| DELETE | `/api/transactions/{id}` | Remove a mistaken entry |
| GET | `/api/reports/daily?date=` | Totals in/out for one day + per-category |
| GET | `/api/reports/monthly?month=YYYY-MM` | Per-category totals (rolled up through the tree), income vs expense, net |
| POST | `/api/receipts` | Multipart image upload → extraction → PENDING_REVIEW draft |
| GET | `/api/receipts?status=` | Review queue |
| GET | `/api/receipts/{id}` | Draft with line items |
| PUT | `/api/receipts/{id}/lines` | Save edits to draft lines |
| POST | `/api/receipts/{id}/confirm` | Write transactions, status → CONFIRMED |
| GET | `/api/receipts/{id}/image` | Original photo (shown beside the review form) |
| POST | `/api/sales-reports` | Multipart CSV upload → parsed preview, PENDING_REVIEW |
| GET | `/api/sales-reports/{id}` | Parsed lines + mapped categories |
| POST | `/api/sales-reports/{id}/confirm` | Write income transactions for the business date |

Monthly report response sketch:

```json
{
  "month": "2026-07",
  "incomeCentavos": 18550000,
  "expenseCentavos": 12030050,
  "netCentavos": 6519950,
  "byCategory": [
    { "categoryId": 4, "name": "Kiosk (Restaurant)", "type": "INCOME",
      "totalCentavos": 5210000, "children": [ ... ] }
  ]
}
```

Rollup rule: a parent's total includes all descendants. Compute it in the
service layer from one flat `GROUP BY category_id` query — no recursive SQL
needed at this scale.

## Correctness rules (apply everywhere)

1. **Money**: `long` centavos in Java, `BIGINT` in Postgres, formatted to
   `₱1,234.50` only in the frontend. No `double`, no `float`, ever.
2. **Dates**: `business_date` is a `LocalDate` chosen by the user (defaulting
   to today in `Asia/Manila`). `created_at` is a separate audit timestamp.
3. **Validation at the boundary**: controllers validate DTOs
   (`@Valid`, positive amounts, category must exist and match the
   transaction's type); entities stay clean.
4. **No secrets in the repo**: the Anthropic API key and DB passwords come
   from environment variables (`application.yml` uses `${...}` placeholders).

## Build phases

| Phase | What | Who |
|---|---|---|
| 1 | Ledger API: skeleton, Flyway schema, categories, manual transactions, daily/monthly reports | Jude (guided) |
| 2 | Web app v1: manual entry + reports UI against the phase-1 API | Claude |
| 3 | Ingestion — a: POS daily-report CSV import (no external services, good warmup); b: receipt photo scanning via the Claude API | Jude (API) + Claude (UI) |
| 4 | Ship it: VPS deploy (Caddy + docker-compose), Spring Security form login, nightly `pg_dump` backup off-box | together |

## Phase 1 — getting started (Jude)

1. Generate the project at [start.spring.io](https://start.spring.io):
   - Project **Maven**, Language **Java 21**, Spring Boot **3.5.x**
   - Group `com.chateauleonor`, Artifact `tracker`, Packaging **Jar**
   - Dependencies: **Spring Web**, **Spring Data JPA**, **Validation**,
     **PostgreSQL Driver**, **Flyway Migration**, **Docker Compose Support**,
     **Spring Boot DevTools**
2. Unzip into `backend/` in this repo, commit, push. (Docker Compose Support
   generates a `compose.yaml` with Postgres — `./mvnw spring-boot:run` starts
   the database for you if Docker is installed.)
3. Write `backend/src/main/resources/db/migration/V1__init.sql` from the
   schema above, and `V2__seed_categories.sql` from the starter tree.
4. First vertical slice: `Category` entity → repository → service →
   controller, until `GET /api/categories` returns the seeded tree.

Suggested package layout (by feature, not by layer):

```
com.chateauleonor.tracker
├── TrackerApplication.java
├── category/      Category, CategoryRepository, CategoryService,
│                  CategoryController, dto/
├── transaction/   ...same shape
├── receipt/       Receipt, ReceiptLineItem, ReceiptExtractionService,
│                  ClaudeReceiptExtractionService, ...
├── salesreport/   SalesReport, PosReportParser, ...
└── report/        ReportService, ReportController
```

Definition of done for phase 1: categories CRUD, manual transaction entry,
and both report endpoints working against local Postgres, with service-layer
unit tests for the report rollup and money parsing.
