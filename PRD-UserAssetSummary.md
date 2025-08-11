# PRD: User Asset Summary Table

## Overview
Introduce a user-specific asset summary capability to support accurate holdings valuation and performance reporting. Today, `assets` and `prices` are global. We will add a per-user summary table to persist user holdings inputs and store computed outputs using the latest price and optional currency conversion at read time.

- currentValue = shares/units * latestPrice
- performanceRatio = currentValue / purchaseValue
- Performance horizons: YTD, 1Y, 2Y, 3Y, 4Y, 5Y (rolling)

Scope: Backend only.

## Goals
- Persist user-specific holding metadata (shares, purchaseValue, purchaseDate) without storing currency (rely on asset base currency).
- Compute and persist current value and performance ratios (YTD and 1–5Y) via hybrid processing: event-driven recompute on user/price changes, plus periodic cron full sweep.
- Keep read paths simple: read precomputed values; apply optional currency conversion on response if needed.

## Non-Goals
- Building transaction ingestion or automatic cost-basis from transactions.
- Portfolio allocation logic beyond per-asset holdings.
- Time-weighted or money-weighted returns.

## Definitions
- purchaseValue: Total cost basis for the current holding, in the asset's base currency.
- performanceRatio: currentValue / purchaseValue (dimensionless). performancePct = (performanceRatio - 1) * 100.
- YTD performance: current year performance using a baseline price from the later of Jan 1 or purchaseDate.

## Assumptions
- Latest price is resolved by `prices.isLatest = true` in `prices` table.
- Shares and purchaseValue are calculated from the transactions table (no multi-currency buys).
- Cost-basis method is average and includes fees and taxes.
- Price currency must equal the asset base currency; reject or normalize inputs otherwise.
- If price data is missing for a baseline, the corresponding performance metric is null.

## Data Model

Table: `user_asset_summaries`
- id: uuid PK
- userId: uuid NOT NULL, FK -> `users.id`
- assetId: uuid NOT NULL, FK -> `assets.id`
- shares: numeric(28,8) NOT NULL, CHECK shares >= 0 (calculated from transactions table)
- purchaseValue: numeric(19,4) NOT NULL, CHECK purchaseValue >= 0  // in asset base currency calculate from transactions
- purchaseDate: date NULL  // inception date of the current position; see rules below
- currentValue: numeric(19,4) NULL
- perfAllRatio: numeric(19,12) NULL
- perfYTDRatio: numeric(19,12) NULL
- perf1yRatio: numeric(19,12) NULL
- perf2yRatio: numeric(19,12) NULL
- perf3yRatio: numeric(19,12) NULL
- perf4yRatio: numeric(19,12) NULL
- perf5yRatio: numeric(19,12) NULL
- latestPriceTimestamp: timestamptz NULL         // for idempotency/freshness checks
- lastHoldingsUpdatedAt: timestamptz NULL        // for idempotency/freshness checks
- lastCalculatedAt: timestamptz NULL
- createdAt: timestamptz DEFAULT now()
- updatedAt: timestamptz auto (updated on each write)

Constraints and Indexes:
- UNIQUE (userId, assetId)
- INDEX (userId)
- INDEX (assetId)
- INDEX (lastCalculatedAt)
- INDEX (latestPriceTimestamp)
- FK: userId ON DELETE CASCADE; assetId ON DELETE RESTRICT

### Holdings aggregation rules (from transactions)
- Shares: sum of quantities from BUY minus SELL; ignore currency conversions (no multi-currency buys). Rebalances/splits/dividends handled as below.
- Purchase value (cost basis): average cost method, including fees and taxes from transactions; computed in asset base currency.
- Transfers between wallets do not change shares or purchase value; only location.
- Purchase date: the inception date of the current holding. This is the earliest BUY date contributing to the remaining position after matching prior SELL quantities. If the position is fully closed and later re-opened, `purchaseDate` resets to the first BUY date of the new position. If derivation is ambiguous (e.g., missing historical data), leave `purchaseDate` null.

Detailed per-transaction handling
- BUY: increase `shares += quantity`; increase `purchaseValue += quantity * price + fee + tax`.
- SELL: decrease `shares -= quantity`; decrease `purchaseValue -= avgCostPerShare * quantity`, where `avgCostPerShare = purchaseValue_before / shares_before` (fees/taxes on sale do not change cost basis).
- DIVIDEND:
  - Cash dividend: no change to `shares` or `purchaseValue` (income tracked elsewhere).
  - Reinvested dividend (DRIP): if details include `quantity` (or reinvest flag), treat as BUY with amount equal to reinvested value (include fees/taxes if provided).
- REBALANCE: treat as composite SELL/BUY legs per details. For the affected asset, apply the same SELL/BUY rules above using provided quantities/amounts.
- SPLIT (encoded under REBALANCE details with `splitRatio` when needed): adjust `shares = shares * splitRatio`; keep total `purchaseValue` unchanged (per-share cost adjusts implicitly).

## Calculations

currentValue
- shares * latestPrice
- Value is stored in asset base currency
- If no latest price, currentValue = null.

perfAllRatio (vs purchase)
- If purchaseValue > 0 and currentValue present: currentValue / purchaseValue
- perfAllPct = (perfAllRatio - 1) * 100 (derived at read-time if needed)

YTD performance
- baselineDate = max(start-of-year, purchaseDate if provided)
- baselinePrice = first price on/after baselineDate; fallback to nearest after; if none in year -> null
- baselineValue = shares * baselinePrice
- perfYTDRatio = currentValue / baselineValue

Multi-horizon performance (1–5Y)
- windows = {1Y, 2Y, 3Y, 4Y, 5Y} (rolling lookback from now)
- For each window X in years:
  - baselineDateX = max(purchaseDate, now minus X calendar years) [UTC]
  - baselinePriceX = first price on/after baselineDateX; if none, nearest after; if still none -> null
  - baselineValueX = shares * baselinePriceX
  - perfXRatio = currentValue / baselineValueX (null if any component missing)
- Store ratios in corresponding columns (perf1yRatio..perf5yRatio). Percent values can be derived at read-time.

## Computation Strategy: Hybrid (official)
- Event-driven recompute (user-focused):
  - Trigger on price updates that set `isLatest = true` for an `assetId` (recompute all summaries for that asset).
  - Trigger on transactions that change holdings/cost basis for a `(userId, assetId)` (recompute that row only).
  - Debounce/coalesce per asset and per `(userId, assetId)`; default 120 seconds (config: `USER_ASSET_SUMMARY_DEBOUNCE_MS`).
- Cron full sweep (system-wide):
  - Default: hourly at minute 0 (`0 * * * *`), configurable via `USER_ASSET_SUMMARY_CRON`.
  - Recomputes all users to backfill missed events and refresh rows where `lastCalculatedAt` < max(`latestPriceTimestamp`, `lastHoldingsUpdatedAt`).

## Computation via Cron Job

Schedule
- Cadence: every 1 hour (configurable via env, e.g., `USER_ASSET_SUMMARY_CRON`), or nightly if sufficient.

Processing steps
- Fetch all active `user_asset_summaries` (or only changed since lastCalculatedAt if change tracking available).
- For all distinct `assetId`s, fetch latest price (`isLatest = true`).
- Compute `currentValue` for each summary with available price.
- For each summary, compute YTD ratio and 1–5Y ratios using price baselines per rules above.
- Upsert computed fields and set `lastCalculatedAt = now()`.

Performance
- Batch computations by asset to minimize price lookups.
- Preload necessary baseline prices with date-bounded queries per asset and window.
- Use transactions and chunking to avoid long locks.

Failure handling
- Partial failures logged with asset/user context; retries on next cron tick.
- Leave previous values intact if computation for a row fails.

## Alternative: Event-Driven Only Recompute

Triggers
- Price updates: when a new price becomes latest for an `assetId` (`isLatest = true`), enqueue recompute for all summaries of that asset.
- Holdings changes: when transactions alter effective `shares` or `purchaseValue` for a user/asset, enqueue recompute for that specific `(userId, assetId)`.
- Asset changes: if relevant fields (e.g., base attributes affecting valuation logic) change, enqueue affected rows.

Processing model
- Use a job queue (e.g., BullMQ) and coalesce by `assetId` and `(userId, assetId)` with a short debounce window (1–5 minutes) to avoid bursty recomputes during rapid updates.
- Make recompute idempotent: skip if `lastCalculatedAt` is newer than both the latest price timestamp and the latest holdings change version.
- Apply per-asset concurrency limits or locks to avoid contention.

Pros
- Fresher metrics with minimal lag after changes.
- Less wasted work compared to blind cron sweeps.
- Scales with change volume instead of table size.

Cons/Risks
- Higher complexity and infra needs (queue, event publication, idempotency).
- If events are dropped or mispublished, rows can become stale without a safety sweep.
- Bursty markets can create large spikes; requires proper backpressure and batching.

Mitigations
- Durable eventing (transactional outbox pattern) and at-least-once processing.
- Metrics/alerts on recompute lag and queue depth.
- Optional lightweight audit tooling (manual or admin-triggered) to re-enqueue suspected-stale rows if strictly avoiding scheduled cron.

## APIs (Hybrid support)

### POST /assets/user-update
- Summary: Proactively recompute a user’s asset summary (single asset or all assets) on demand.
- Auth: JWT required; only updates summaries for the authenticated user.
- Body (one of):
  - { assetId: string }  // recompute this asset only
  - { all: true }        // recompute all assets for the user
- Behavior:
  - Enqueue event-driven recompute jobs for the current user. Debounce/coalesce applies.
  - Returns 202 Accepted with a job reference and a brief status.
- Responses:
  - 202: { status: 'queued', jobId: 'uuid', assetId?: string, all?: boolean }
  - 400: invalid payload or assetId not owned by user
  - 401: unauthorized

## Reading and Response Behavior
- APIs return values in asset base currency;
- Returned fields:
  - `currentValue`, `perfAllRatio` and derived `perfAllPct`
  - `perfYTDRatio` and derived `perfYTDPct`
  - `perf{1..5}yRatio` and derived percents

Note: Optional currency conversion may be applied at read-time based on the client’s requested currency; ratios remain dimensionless.

## Performance & Caching
- Read path is O(1) against precomputed columns; minimal additional caching required.
- Optionally use in-memory TTL cache for hot requests.

## Error Handling
- 400: validation errors (on writes elsewhere managing this table)
- 404: not found (when reading specific rows)
- Cron errors: logged and observable; do not surface to clients directly.

## Environment Variables

- `USER_ASSET_SUMMARY_DEBOUNCE_MS` (default: 120000) — debounce/coalesce window for event-driven recomputations
- `USER_ASSET_SUMMARY_CRON` (default: `0 * * * *`) — cron schedule for periodic full sweep

## Open Questions
- Debounce window defaults and per-asset concurrency limits for event-driven recompute.
- Staleness threshold for rows to be picked up by cron full sweep.
- Any edge-case transactions that should adjust holdings (e.g., rebalances) in this phase, or defer.
- Admin tooling requirements to re-enqueue or backfill specific users/assets.

## Migration Plan
- Create TypeORM migration for `user_asset_summaries` with constraints and indexes.
- No backfill required initially.

## Testing
- Unit tests: calculation of currentValue, YTD, and 1–5Y ratios from given price series.
- Integration tests: cron processor batches and upserts correctly.
- E2E: seed prices and summaries; run cron; verify persisted outputs and read responses.

## Rollout
- Deploy migration.
- Deploy cron processor with safe cadence.
- Monitor logs for missing prices and long-running batches; tune cadence/chunk size.
