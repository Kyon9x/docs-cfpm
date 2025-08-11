## PRD — Manual Price Update from Assets List (POST /prices)

### 1) Overview & Context

- Purpose: Enable users to manually update the latest price of an asset directly from the Assets list via an action button. The action will call backend `POST /prices` and persist a new price row.
- Scope: Frontend action (modal + POST) and backend price creation semantics, with a new field `isManual` to distinguish user-entered prices from automated provider updates.
- References:
  - Backend service: `be/src/prices/prices.service.ts` (create flow, latest semantics)
  - Existing Architecture: `docs/assets/Assets-Summary-Architecture.md` (latest-price strategy, summary usage)
  - Future: `docs/PRD-MarketPriceService.md` (automated price ingestion)

### 2) Goals

- Allow user to add a price for an asset from the Assets page.
- Ensure new price participates in the latest-price logic used by `/assets/summary`.
- Mark user-entered prices with `isManual = true` for traceability.

### 3) Change Log

- v0.1 (2025-08-10): Initial PRD for manual price update.

### 4) Functional Requirements (FR)

1. FR1: FE provides an “Update Price” action on each asset row that opens a modal to input:
   - Required: price (number)
   - Optional: timestamp (defaults to now)
   - Implicit: assetId (from row)
2. FR2: FE calls `POST /prices` with payload:
   - `assetId: string`
   - `price: number`
   - `timestamp?: string (ISO)` defaults to server time when omitted
   - `source?: string` (optional, may set `MANUAL`)
   - `market?: string` (optional)
   - `isManual: true`
3. FR3: BE persists the new price and maintains latest semantics (flip previous latest if new is more recent or equal in time). Existing logic in `PricesService.create` applies.
4. FR4: `/assets/summary` reflects the manual price in computed values when it becomes latest (no per-asset calls, uses join to latest price).
5. FR5: Auditability: returned `Price` entity includes `isManual` in response for admin/analytics (optional FE use now; required for future reports).

### 5) Non-Functional Requirements (NFR)

1. NFR1: Price creation p95 ≤ 100ms typical.
2. NFR2: No additional load to summary beyond existing join/cache.
3. NFR3: Input validation (price > 0; valid UUID for assetId; timestamp parseable if provided).
4. NFR4: Auth required; reuse existing auth/guards.

### 6) Data Model & API

#### Backend Data Model

- Entity: `Price` (`be/src/prices/entities/price.entity.ts`)
  - New field: `isManual: boolean` (default false)
  - Existing fields: `id`, `assetId`, `price`, `timestamp`, `source`, `market`, etc.

Notes: With a fresh DB (synchronize enabled), adding the column in the entity will auto-create. For existing DBs, a migration would be needed (out of scope here if fresh DB is used).

#### API — Create Price

- Method/Route: `POST /prices`
- Request (JSON):
```
{
  "assetId": "uuid",
  "price": 123.45,
  "timestamp": "2025-08-10T10:15:00Z" (optional),
  "source": "MANUAL" (optional),
  "market": "NASDAQ" (optional),
  "isManual": true
}
```
- Responses:
  - 201: Created — returns created `Price` (including `isManual` and `isLatest`)
  - 400: Validation error (invalid price/assetId/timestamp)
  - 404: Asset not found (if validation checks are added)
  - 500: Server error

#### BE Logic (Create)

- On create (`PricesService.create`):
  - If there is an existing latest price for the `assetId` and the incoming `timestamp` is newer or equal, set previous latest to false and mark new price as latest.
  - If incoming `timestamp` is older, keep `isLatest=false` on the new row.
  - Always persist `isManual` (true for this flow). Future automated updates will set `isManual=false`.

### 7) Frontend UX

- Assets table row action: “Update Price”.
- Modal fields: Price (required, numeric); Date/Time (optional; default now). Display currency context if available.
- On submit: call service to `POST /prices` with `isManual=true`.
- On success: show toast; optionally refresh summary or rely on short cache TTL for `/assets/summary`.

### 8) Security & Permissions

- Reuse existing auth guard.
- Optionally restrict to roles allowed to update prices (future enhancement).

### 9) Performance & Caching

- `/assets/summary` cache TTL (30–60s) remains as-is. Manual updates may be visible after TTL expiry; optional button to force refresh on FE.

### 10) Future — Market Price Service

- A backend market price service will ingest provider data and create prices with `isManual=false` and `source='MARKET_SERVICE'`.
- Manual vs. automated updates can coexist; latest determination remains timestamp-based.

### 11) Acceptance Criteria

1. FE action posts a new price with `isManual=true` and required fields validated.
2. BE returns created price and maintains latest semantics.
3. `/assets/summary` reflects the manual latest price after cache expiry or refresh.
4. Price responses include `isManual` for traceability.

### 12) Out of Scope

- Role-based restrictions for price updates (may be added later).
- Real-time cache invalidation and push updates.


