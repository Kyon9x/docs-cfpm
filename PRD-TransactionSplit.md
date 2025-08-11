# PRD: Transaction Type – SPLIT (Stock Split / Reverse Split)

## Overview

Introduce a first-class `SPLIT` transaction type to model stock splits and reverse splits. Today, splits can be represented implicitly under `REBALANCE` with a `splitRatio`. This PRD formalizes `SPLIT` as its own type for clarity, validation, and consistent holdings aggregation.

Scope: Backend + Frontend.

## Goals

- Provide a dedicated transaction type `SPLIT` with explicit details schema.
- Ensure holdings aggregation adjusts `shares` by the split ratio while keeping `purchaseValue` unchanged.
- Support both stock splits (ratio > 1, e.g., 2.0 for 2-for-1) and reverse splits (ratio < 1, e.g., 0.5 for 1-for-2).
- Maintain idempotency and accurate audit fields.

## Non-Goals

- Corporate action cash distributions (handled elsewhere).
- Portfolio rebalancing of multiple assets (still via `REBALANCE`).

## Data Model

- Reuse existing `transactions` table; add enum value `SPLIT` to backend `TransactionType`.
- No new tables. Continue to store details in `transactions.details` (JSONB).

Details schema (`SplitTransactionDetails`):
- `assetId: uuid` (required) — target asset
- `splitRatio: number` (required, > 0) — multiplier applied to shares
- `effectiveDate: date` (optional) — date corporate action takes effect; defaults to transaction timestamp
- `notes: string` (optional)

Validation rules:
- `splitRatio > 0`; 2.0 means 2-for-1 split; 0.5 means 1-for-2 reverse split
- `assetId` must reference existing asset

## Holdings Aggregation Rules

- On `SPLIT`:
  - `shares = shares_before * splitRatio`
  - `purchaseValue = purchaseValue_before` (unchanged)
  - Implied average cost per share adjusts as `avgCostPerShare_after = purchaseValue / shares_after`

Edge cases:
- If `shares_before = 0`, a split still yields `shares_after = 0`; do nothing to purchaseValue
- Rounding is not persisted at this stage; use high precision decimal columns for aggregation

## Backend Changes

1. Update enum `TransactionType` to include `SPLIT`.
2. Add `split-transaction-details.dto.ts` with class-validator rules for `assetId`, `splitRatio`.
3. Update `create-transaction.dto.ts` and validators to accept `SPLIT` with the new details DTO.
4. Update holdings aggregation logic used by `user_asset_summaries` to apply SPLIT rule.
5. Swagger: Document `SPLIT` examples.

## Frontend Changes

1. Update FE transaction type unions to include `split` where applicable.
2. Allow `createInvestmentTransaction` to submit `type: 'SPLIT'` with details `{ assetId, splitRatio, ... }`.
3. Optional: UI affordance for entering split ratio and previewing post-split shares.

## API

Create Transaction (existing):
- POST `/transactions`
- Body for SPLIT:
```json
{
  "walletId": "uuid",
  "currency": "USD",
  "timestamp": "2025-01-01T00:00:00Z",
  "status": "COMPLETED",
  "referenceId": "ref-123",
  "type": "SPLIT",
  "details": {
    "assetId": "uuid",
    "splitRatio": 2.0,
    "effectiveDate": "2025-01-01"
  }
}
```

Responses:
- 201 Created with transaction payload
- 400 on validation error

## Interactions with User Asset Summaries

- `SPLIT` must trigger recompute for the affected `(userId, assetId)` via event-driven path.
- Aggregation applies the rules above; ratios and current values recompute accordingly.

## Testing

- Unit: DTO validation for `splitRatio` bounds.
- Integration: Holdings aggregation before/after SPLIT; purchaseValue unchanged; shares scaled.
- E2E: Create `SPLIT` transaction and verify user asset summary reflects adjusted shares.

## Rollout

- Backend: merge enum + DTO + logic; add Swagger docs.
- Frontend: ship type union and service allowance; UI enhancements can follow.


