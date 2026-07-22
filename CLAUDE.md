# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Income/expense tracker for Chateau Leonor Resort (a family resort in the
Philippines). Replaces hand-kept books. **`DESIGN.md` is the source of truth** —
read it before doing architecture or data-model work; this file only captures
what's easy to get wrong or can't be inferred from the (currently minimal) code.

## Current state

Design phase. The repo holds `DESIGN.md`, this file, and a real sample POS
report under `docs/samples/`. **No `backend/` or `frontend/` code exists yet** —
they arrive per the phase plan in `DESIGN.md`. Do not assume build tooling is
present; check for `backend/pom.xml` / `frontend/package.json` first.

## Working agreement (important — not a normal repo)

- **Jude writes the Spring Boot backend himself, guided commit by commit.** He
  is learning. Do **not** silently generate backend code for him — explain,
  suggest the next slice, and review. Building the frontend and reviewing
  backend work is Claude's part.
- Build one **vertical slice** at a time (entity → repository → service →
  controller), starting with `category`, not layer-by-layer across features.

## Architecture (the big picture)

Everything converges on **one table, `transactions`** — one row per peso
movement. Three ingestion paths feed it, and all reports are `GROUP BY` queries
over it:

1. **Manual entry** — row written directly.
2. **Receipt photo scan** (expenses) — Claude vision API returns itemized JSON;
   becomes one transaction per line item *after human review*.
3. **POS daily-report CSV import** (income) — parses the resort's end-of-day
   export; one transaction per item line *after human review*.

Backend is layered Spring (controller → service → JPA repository → entity),
organized **by feature package** (`category`, `transaction`, `receipt`,
`salesreport`, `report`), not by layer. Almost no inheritance — composition via
constructor injection. The only deliberate polymorphism is
`ReceiptExtractionService` (interface) with a Claude implementation and a fake
for tests.

## Invariants that are easy to violate

- **Money is integer centavos** — `long` in Java, `BIGINT` in Postgres,
  formatted to `₱x.xx` only in the frontend. Never `double`/`float`, never store
  a formatted string.
- **Amounts are always positive; `type` (`INCOME`/`EXPENSE`) carries direction.**
  Enforced by a DB CHECK. Don't introduce negative amounts.
- **Nothing reaches `transactions` without human confirmation.** Scans and
  imports land as `PENDING_REVIEW` drafts; a person confirms before rows are
  written. This review gate is a hard requirement, not a nicety.
- **Categories are archived, never deleted** — old transactions must keep
  pointing at the category they were booked under. Category is a self-referencing
  tree; report totals roll *up* through it (computed in the service layer from a
  flat `GROUP BY`, not recursive SQL).
- **`business_date` is a user-chosen `LocalDate` in `Asia/Manila`**, distinct
  from the `created_at` audit timestamp. The app pins the timezone explicitly —
  never derive the business date from the server's local clock.
- **Extraction/parsing stays behind its interface**, with a fake used in tests.
  Swapping providers should touch one class.

### POS CSV parsing gotchas (see `docs/samples/Report_2026_06_19.csv`)

- The file starts with a **UTF-8 BOM** — strip it or the first field won't match.
- **Trust the report's item amounts; do not recompute `count × unit price`** —
  amounts are net of per-item (Senior/PWD) discounts and won't divide evenly.
- Store the payment split (cash/GCash) and discount totals on the report record
  for drawer reconciliation, not as transactions.

## Commands (apply once `backend/` is generated — Maven, per DESIGN.md)

```bash
cd backend
./mvnw spring-boot:run          # runs the API; Docker Compose Support starts Postgres
./mvnw test                     # all tests
./mvnw test -Dtest=ReportServiceTest              # one test class
./mvnw test -Dtest=ReportServiceTest#rollsUpTree  # one test method
./mvnw clean package            # build the jar
```

Requires Docker running (Postgres container) and Java 21. Secrets
(`ANTHROPIC_API_KEY`, DB password) come from environment variables via
`${...}` placeholders in `application.yml` — never hardcoded.

## Repo / workflow

- **Public repo.** Code and docs are world-readable; real financial data lives
  only in the database, never in the repo.
- Active branch: `claude/chateau-leonor-accounting-1k1iht`, tracked by **PR #1**.
  Push to that branch to update the PR; do not open new PRs.
