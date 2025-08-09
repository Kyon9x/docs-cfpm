## PRD — Net Worth Overview (Desktop-first)

### 1) Overview
A new top-level Net Worth page giving a broad personal finance overview: total net worth, trends (daily/weekly/monthly/yearly), asset allocation, liabilities (debts), and cashflow (income vs expenses). Desktop-first; 1-week delivery target; good performance.

### 2) Goals and Success Metrics
- **Goals**
  - Provide a truthful, up-to-date big picture of a user’s finances.
  - Help users track growth trends over selectable periods.
- **Success Metrics**
  - Net worth summary loads within 1s on cached data.
  - Trend chart renders within 1.5s on first view for a 1-year range.
  - ≥20% of users create at least one debt record within first week of feature adoption.

### 3) Users and Primary Use Cases
- View current net worth and change vs previous period.
- Track net worth growth using a trend chart across day/week/month/year intervals.
- See composition: assets vs liabilities and asset class breakdown.
- Manage liabilities manually (create/update/delete).
- Review cashflow (income vs expense) over the selected period.

### 4) Scope (MVP)
- New screen: Net Worth page (top-level navigation).
- Trend chart with intervals: day, week, month, year.
- Net worth summary: total, delta vs previous comparable period.
- Asset allocation: by asset class using existing assets/prices.
- Liabilities: manual “debts” CRUD with balances and basic terms.
- Cashflow: income vs expense, using existing transaction types.
- Onboarding guide to prefill initial net worth (manual inputs), then auto-update from transactions.

Out of scope (MVP): bank integrations, forecasts/projections, alerts/reminders.

### 5) Information Architecture and Navigation
- New nav item: Net Worth (desktop-first). The existing `Assets` screen remains for asset-centric CRUD.
- Deep links from Net Worth cards to detail pages: `Assets`, `Wallets`, `Transactions`, `Debts`.

### 6) UX and Components (MVP)
- **Header**
  - Total Net Worth (primary), delta vs previous period (secondary), currency display.
  - Range selector: Last 7D, 30D, 90D, 1Y, YTD, Custom.
- **Trend Chart**
  - Line/area chart for total net worth.
  - Interval aggregation toggle: day/week/month/year.
  - Hover tooltip: date, total, delta.
- **Cards**
  - Assets: total and breakdown by class, top holdings (name, weight).
  - Liabilities: total debt, next due (if provided), list by type with balances.
  - Cashflow: stacked bars or dual lines: income vs expense; same range selector.
- **Onboarding Guide**
  - Prompt user to input initial net worth: assets (aggregated or by class), debts with balances, cash balances.
  - Explain that subsequent transactions auto-adjust balances.

### 7) Data and Computation
- **Existing models to leverage**
  - Transactions (for cashflow and holdings derivation):
    ```ts
    export enum TransactionType {
      DEPOSIT, WITHDRAWAL, REBALANCE, TRANSFER,
      BUY, SELL, DIVIDEND, CONVERSION, FEE, INTEREST,
    }
    ```
  - Prices (for valuation and historical charting): `assetId`, `price`, `timestamp`.
  - Wallets (cash and accounts): `currencyCode`, `balance`.
  - Assets (allocation): `type`, `name`.
- **Functional computation**
  - Net worth at time t = sum(valued assets at t) + sum(wallet balances at t) − sum(debt balances at t).
  - Asset valuation needs holdings quantities and prices at t.
  - Cashflow mapping finalized after decision (see Decisions Pending).

### 8) Data Model Additions
- **Debts module (required for manual liabilities)**
  - Debt
    - id (uuid), userId (uuid)
    - name (string), type (enum: loan, credit_card, mortgage, other)
    - balance (decimal), currency (char[3])
    - apr? (decimal), minPayment? (decimal), dueDay? (int 1–31)
    - notes? (text)
    - createdAt, updatedAt (timestamps)
- **Net Worth Snapshots (for trends performance)**
  - net_worth_snapshots
    - id (uuid), userId (uuid)
    - date (date, UTC day bucket)
    - total (decimal), assetsTotal (decimal), debtsTotal (decimal), currency (char[3])
    - createdAt (timestamp)

### 9) API Endpoints (MVP)
- Net Worth summary
  - `GET /networth/summary?from&to&interval=day|week|month|year`
  - Returns: total, delta, assetsTotal, debtsTotal, allocation by asset class, debts by type.
- Trend data
  - `GET /networth/trend?from&to&interval=day|week|month|year`
  - Returns: series of `{ date, total, assetsTotal, debtsTotal }`.
- Cashflow summary
  - `GET /cashflow/summary?from&to&interval=day|week|month`
  - Returns: series for income and expense.
- Debts
  - `GET /debts`
  - `POST /debts`
  - `PATCH /debts/:id`
  - `DELETE /debts/:id`

All endpoints must be JWT-protected, user-scoped, and documented with Swagger including example payloads.

### 10) Performance
- Desktop-first; target sub-1s for cached summary, sub-1.5s for 1Y trend initial render.
- Server-side caching per user+range (short TTL, e.g., 60s).
- Use persisted daily snapshots for fast charting and deltas.

### 11) Security/Privacy
- Reuse existing JWT guard and rate limiting.
- Avoid logging sensitive payload values; redact balances in debug logs.
- Strict authorization: only access user’s own data.

### 12) Analytics
- Events: `networth_viewed`, `networth_range_changed`, `debt_created`, `debt_updated`, `cashflow_range_changed`.
- Funnels to monitor onboarding completion and early usage of debts.

### 13) Acceptance Criteria
- Net Worth page accessible from nav; loads summary and trend within performance targets.
- Trend chart supports day/week/month/year intervals and the provided range presets.
- Asset breakdown shows totals by class; values match underlying assets/prices (within rounding).
- Debts CRUD works; totals reflect in summary and trend promptly.
- Cashflow summary matches agreed mapping rules for the selected range.
- Swagger available for all new endpoints with realistic example payloads.

### 14) Decisions
- **Finalized**
  1. Screen placement: **A) New top-level Net Worth screen; keep Assets**.
  2. Trend data strategy: **B) Persist daily net worth snapshots** for performance.
  3. Holdings derivation (MVP): **B) Use current wallet balances + manually entered asset values; add holdings later**.
- **Pending (no assumptions)**
  4. Cashflow mapping rules: exact treatment for DEPOSIT/WITHDRAWAL vs TRANSFER to avoid double counting; whether BUY/SELL should be excluded from income/expense.
  5. Onboarding inputs granularity: aggregated per class vs per-asset entry for initial net worth.

### 15) Timeline Plan (1 week)
- Day 1: Finalize pending decisions; confirm API contracts; finalize page layout.
- Day 2–3: Define data contracts and examples for Swagger; specify `debts` and snapshot schemas (no implementation in this PRD).
- Day 3–4: Prepare UI spec and component structure for Net Worth page; states for loading/empty/onboarded.
- Day 5: Non-functional notes (caching, rate limiting), analytics events list, and QA checklist.

### 16) References to current code
- Transaction fields and types: see `be/src/transactions/entities/transaction.entity.ts`.
- Amount derivations used today (consistency target): see `be/src/transactions/transactions.service.ts`.
- Latest price helper: `pricesService.getLatestPriceForAsset(assetId)` in `be/src/prices/prices.service.ts`.
- Wallet balances and currencies: `be/src/wallets/entities/wallet.entity.ts`.
- Asset types for allocation: `be/src/assets/entities/asset.entity.ts`.


