# CLFinancialTracker
Expenses and income tracker custom-designed for Chateau Leonor Resort.

Tracks every peso in and out per day and per month. Expenses come in by
phone-camera receipt scanning (Claude vision API); income comes in by
importing the POS daily sales report — both land as categorized entries
after human review.

- **[DESIGN.md](DESIGN.md)** — architecture, data model, ingestion pipelines,
  API contract, and the build plan.
- `docs/samples/` — real example inputs (POS daily report CSV).
- `backend/` — Spring Boot API (Java 21). *(coming in phase 1)*
- `frontend/` — React mobile-first web app. *(coming in phase 2)*
