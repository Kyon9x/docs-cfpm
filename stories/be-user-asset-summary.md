# Story: User Asset Summary (Backend)

<!-- Source: PRD - docs/PRD-UserAssetSummary.md -->
<!-- Context: Brownfield enhancement to existing NestJS backend -->

## Status: Draft

## Story

As an authenticated user,
I want my asset summaries (holdings and performance) to be precomputed and up-to-date,
so that I can view accurate current values and performance without slow on-the-fly calculations.

## Context Source

- Source Document: docs/PRD-UserAssetSummary.md
- Enhancement Type: Feature (new module + data model + compute pipeline)
- Existing System Impact: Integrates with `assets`, `prices`, and `transactions` modules; adds JWT-protected recompute endpoint; background recompute via cron.

## Acceptance Criteria

1. Database table `user_asset_summaries` exists with columns and constraints per PRD:
   - Keys: `id (uuid PK)`, `userId (FK)`, `assetId (FK)`, `UNIQUE(userId, assetId)`
   - Numeric columns: `shares(28,8)`, `purchaseValue(19,4)`, `currentValue(19,4)`, ratios `perfAllRatio`, `perfYTDRatio`, `perf1yRatio..perf5yRatio` (19,12)
   - Timestamps: `latestPriceTimestamp`, `lastHoldingsUpdatedAt`, `lastCalculatedAt`, `createdAt`, `updatedAt`
   - Optional `purchaseDate: date`
   - Indexes on `(userId)`, `(assetId)`, `(lastCalculatedAt)`, `(latestPriceTimestamp)`
2. Recompute logic:
   - Uses `prices.isLatest = true` row per `assetId` to compute `currentValue = shares * latestPrice`
   - Computes `perfAllRatio` if `purchaseValue > 0` and `currentValue` present
   - Computes YTD and 1–5Y ratios using “first price on/after baselineDate” rules
   - Leaves ratio `NULL` when baseline price missing; no divisions by zero
   - Sets `lastCalculatedAt` and preserves previous values on failure
3. Event-driven triggers (initial, minimal):
   - On setting a new latest price for an `assetId`, affected summaries for that `assetId` are queued for recompute
   - On transactions that change effective holdings for `(userId, assetId)`, that row is queued for recompute
   - Debounce/coalesce via `USER_ASSET_SUMMARY_DEBOUNCE_MS` (default 120s)
4. Cron full sweep:
   - Scheduled via `USER_ASSET_SUMMARY_CRON` (default `0 * * * *`)
   - Recomputes rows where `lastCalculatedAt` < max(`latestPriceTimestamp`, `lastHoldingsUpdatedAt`)
5. API endpoint:
   - `POST /assets/user-update` (JWT required)
   - Body: `{ assetId: string }` or `{ all: true }`
   - Behavior: enqueues recompute for current user; returns `202 { status: 'queued', jobId, ... }`
   - Validates `assetId` belongs to an existing summary row for the authenticated user; returns 400 if no matching `(userId, assetId)` or invalid payload; 401 if unauthorized
   - Fully documented in Swagger with examples
6. Tests:
   - Unit tests for calculation logic (current value, YTD, 1–5Y)
   - Integration test for cron sweep batching/upserts
   - E2E test for `POST /assets/user-update` auth, validation, and 202 response
7. Config:
   - Supports env vars `USER_ASSET_SUMMARY_DEBOUNCE_MS`, `USER_ASSET_SUMMARY_CRON` with defaults
8. Performance:
   - Batch computations per asset to minimize price lookups; indexes utilized
9. Security:
   - Endpoint protected with JWT; only affects authenticated user’s rows

## Dev Technical Guidance

### Existing System Context

- NestJS v11, TypeORM v0.3, PostgreSQL; Swagger configured in `src/main.ts`
- `prices` table has `isLatest` flag (see `src/prices/entities/price.entity.ts`)
- Global validation pipe; security middleware; JWT auth strategy in place

### Integration Approach

1. New module: `src/user-asset-summaries/`
   - `entities/user-asset-summary.entity.ts`
   - `user-asset-summaries.service.ts`
   - `user-asset-summaries.controller.ts` (for admin/internal if needed; primary endpoint is in `assets.controller.ts` per PRD path)
   - `user-asset-summaries.module.ts`
   - DTOs as needed for internal usage
2. Data model:
   - Implement entity with precise TypeORM column definitions and indexes
   - Create TypeORM migration for table + constraints + indexes (even if `synchronize: true` exists; plan to turn off in prod)
3. Recompute service:
   - Methods: `recomputeForAsset(assetId)`, `recomputeForUserAsset(userId, assetId)`, `recomputeAllStale()`
   - Fetch baseline prices per asset and date window efficiently (bounded queries)
   - Idempotency via `lastCalculatedAt` vs change timestamps
4. Cron scheduling:
   - Add `@nestjs/schedule` and configure cron using env var
5. Event triggers:
   - Prices: hook into `src/prices/prices.service.ts` after a price is saved and marked `isLatest` to call `recomputeForAsset(assetId)`
   - Transactions: in `src/transactions/transactions.service.ts`, after create/update that affects holdings, call `recomputeForUserAsset(userId, assetId)`
   - Start with in-process debounce; job queue (BullMQ) can be added later if required
6. API endpoint:
   - Add to `assets.controller.ts` as `POST /assets/user-update`
   - DTO with union validation; JWT guard; Swagger docs with examples
7. Read behavior:
   - When exposing summaries via any read endpoint, apply optional currency conversion at response time; keep ratios dimensionless

### Holdings Aggregation Rules (from DTOs)

- BUY: `shares += quantity`; `purchaseValue += quantity * price + (fee ?? 0) + (tax ?? 0)`
- SELL: `shares -= quantity`; `purchaseValue -= avgCostPerShare_before * quantity`, where `avgCostPerShare_before = purchaseValue_before / shares_before` (ignore sale fees/taxes for cost basis)
- DIVIDEND (cash): no change; DIVIDEND (DRIP with reinvested quantity): treat as BUY
- REBALANCE: treat as component SELL/BUY legs using provided quantities/amounts
- SPLIT (if represented via rebalance `splitRatio`): adjust `shares = shares * splitRatio`, keep `purchaseValue` unchanged
- Purchase date: earliest BUY date contributing to the remaining position (resets after position closes and re-opens)

Field sources: see `buy-transaction-details.dto.ts` and `sell-transaction-details.dto.ts` (`quantity`, `price`, optional `fee`, `tax`).

### Technical Constraints

- Use decimal columns with specified precision/scale; map to string in TypeORM to avoid float issues
- Follow existing NestJS patterns for DI, guards, Swagger
- Keep performance O(1) on reads; avoid N+1 in computations by preloading prices per asset

### Baseline Price Lookup (per asset/window)

SQL pattern: first price on/after baseline date; fallback to nearest after if needed.

Example (TypeORM): order by `timestamp ASC`, `where: { assetId, timestamp: MoreThanOrEqual(baselineDate) }`, `take: 1`.
Fallback: if none, query `MoreThan(baselineDate)` with `take: 1`.

### Missing Information

- Exact transaction fields to include for fees/taxes in cost basis (confirm names/types in `transactions` DTOs)
- Ownership validation for `assetId` in `POST /assets/user-update` (confirm asset-user linkage path)
- Queueing mechanism choice if we outgrow in-process debounce (BullMQ vs simple cron)

### Security Notes

- Endpoint is JWT-protected; global rate limiting via `src/middleware/rate-limit.middleware.ts` applies
- Validate that recompute operations only touch rows for `request.user.id`

## Tasks / Subtasks

- [ ] Data model
  - [ ] Create `user-asset-summaries` entity with columns/indexes per PRD
  - [ ] Generate and run TypeORM migration
  - [ ] Wire entity into `app.module.ts` (auto-load entities)

- [ ] Module scaffolding
  - [ ] Create `user-asset-summaries.module.ts`, service, controller skeletons

- [ ] Recompute service
  - [ ] Implement holdings aggregation helpers (shares, purchaseValue, purchaseDate per rules)
  - [ ] Implement `currentValue` computation using latest price
  - [ ] Implement YTD and 1–5Y baseline price lookup and ratio computations
  - [ ] Implement idempotency checks and partial failure handling

- [ ] Cron job
  - [ ] Add `@nestjs/schedule` dependency and setup
  - [ ] Implement cron using `USER_ASSET_SUMMARY_CRON`

- [ ] Event triggers (minimal viable)
  - [ ] Hook into price update path that sets `isLatest = true` to call `recomputeForAsset`
  - [ ] Hook into transaction create/update that changes holdings to call `recomputeForUserAsset`
  - [ ] Implement debounce using `USER_ASSET_SUMMARY_DEBOUNCE_MS`

- [ ] API endpoint
  - [ ] Add `POST /assets/user-update` in `assets.controller.ts`
  - [ ] DTO validation for `{ assetId } | { all: true }`
  - [ ] JWT guard; verify asset ownership for `assetId`
  - [ ] Swagger decorators with request/response examples

- [ ] Tests
  - [ ] Unit tests for calculation functions (currentValue, YTD, 1–5Y)
  - [ ] Integration test for cron sweep and upserts
  - [ ] E2E test for recompute endpoint (auth, validation, 202). Include negative tests: non-owned `assetId`, invalid payload.

- [ ] Documentation & Config
  - [ ] Add env var docs to README and confirm defaults
  - [ ] Update Swagger tags/descriptions

## Risk Assessment

### Implementation Risks

- Performance degradation if baseline price lookups are naive
- Stale data if events are missed; cron mitigates but may increase lag
- Precision/rounding errors with decimals

### Mitigations

- Batch price queries by asset and window; use indexes
- Idempotency and periodic cron sweep
- Use TypeORM decimal-as-string and careful division checks

### Rollback Plan

- Revert module and code changes
- Run migration down to drop `user_asset_summaries` if necessary

### Safety Checks

- [ ] Guarded endpoint; changes limited to the authenticated user
- [ ] Idempotent recompute; preserves previous values on failure
- [ ] Tests cover success, null-baseline, and error scenarios


