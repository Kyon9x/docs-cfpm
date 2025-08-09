### PRD: Portfolios Dashboard (FE PortfoliosPage.tsx Integration)

- **Feature**: Replace mock data in `fe/src/pages/PortfoliosPage.tsx` with live data from BE
- **Owner**: BE Team (NestJS)
- **Consumers**: FE `PortfoliosPage.tsx`, `PortfolioCard`, `PortfolioSummaryStats`, future performance chart
- **Status**: Draft for implementation

## 1) Background & Goals
- FE currently uses mocked portfolios and summary; needs a single BE endpoint to feed:
  - Page-level summary stats
  - Portfolio card list with value, performance, asset allocation
  - performance can be change by time (MTD, YTD, totalTime(define by first transaction), QTD, YOY,MOM,QOQ )
- Minimize FE changes; match existing structures in `PortfolioCard` and `PortfolioSummaryStats`.

## 2) In Scope
- Compute portfolios summary and card metrics for the authenticated user.
- Provide asset allocation breakdown by `AssetType`.
- Month-to-date performance per portfolio (absolute and percent).
- Single endpoint to serve the page in one round trip.

## 3) Out of Scope
- Historical timeseries for performance charts. (Future: `/portfolios/:id/performance`)
- CRUD for portfolios (already exists).
- Advanced analytics, tax lots, or rebalance logic.

## 4) Users & Security
- Auth required: JWT (`JwtAuthGuard`).
- Data must be scoped to `user.id`.
- Rate limits apply via middleware.

## 5) Data Sources
- `Portfolio` entity: ownership, name, description, optional `risk`
- `Transaction` entity: BUY/SELL quantities and prices via `details.assetId`, `details.quantity`, `details.price`
- `Asset` entity: `type` for allocation mapping
- `Price` entity: latest price per asset, and price on/before month start
- `CurrencyConversionService`: normalize values to user currency

## 6) Functional Requirements

- Summary block
  - totalValue: sum of current portfolio values
  - portfolioCount: number of active portfolios
  - totalAssets: sum of distinct active positions across portfolios
  - bestPerformer: portfolio with highest MTD % change { name, change }

- Portfolio cards (array)
  - id, name, description
  - value: current total value
  - change: MTD percent
  - changeValue: MTD absolute
  - riskLevel: map from `portfolio.risk` if present; else “Medium” default
  - assets: distinct active positions count
  - lastUpdated: latest transaction timestamp for that portfolio; fallback `portfolio.updatedAt`
  - allocation: [{ name, percentage, color }] aggregated by `AssetType`

- Allocation labels and colors (FE expects color classes)
  - STOCK → “Stocks” → bg-blue-500
  - ETF → “ETFs” → bg-green-500
  - CRYPTO → “Crypto” → bg-purple-500
  - BOND → “Bonds” → bg-gray-500
  - CASH/SAVINGS → “Cash” → bg-yellow-500
  - COMMODITY → “Commodities” → bg-orange-500
  - REAL_ESTATE → “Real Estate” → bg-teal-500
  - MUTUAL_FUND → “Mutual Funds” → bg-pink-500
  - OTHERS/COLLECTIONS → “Others” → bg-slate-500

## 7) Computation Details

- Positions (per portfolio)
  - Consider transactions with `type in [BUY, SELL]` and `details.assetId`.
  - netQty(asset) = SUM(BUY.quantity) − SUM(SELL.quantity)
  - Active position if netQty > 0.

- Current value
  - For each active asset: latest price × netQty
  - If price missing → treat value as 0 for that asset (log internally)
  - Convert to user currency at aggregation time

- Month-to-date (MTD)
  - monthStart = first day of current month at 00:00:00
  - monthStartValue(asset) = (price on/before monthStart) × netQty at monthStart
  - changeValue = currentValue − monthStartValue
  - changePercent = monthStartValue > 0 ? (changeValue / monthStartValue) × 100 : 0

- Assets count
  - Count of distinct active assetIds per portfolio

- Allocation
  - Group currentValue by `AssetType`, map to display label and color
  - percentage = (typeValue / totalCurrentValue) × 100, rounded to 1 decimal
  - If totalCurrentValue = 0, return empty allocation array

## 8) API Design

- GET ` /portfolios/summary`
  - Auth: Bearer token
  - Query: none (first version)
  - Response (200):
    ```
    {
      "summary": {
        "totalValue": number,
        "portfolioCount": number,
        "totalAssets": number,
        "bestPerformer": { "name": string, "change": number } | null
      },
      "portfolios": [
        {
          "id": string,
          "name": string,
          "description": string | null,
          "value": number,
          "change": number,
          "changeValue": number,
          "riskLevel": string,
          "assets": number,
          "lastUpdated": string, // ISO
          "allocation": [{ "name": string, "percentage": number, "color": string }]
        }
      ]
    }
    ```
  - Errors:
    - 401 Unauthorized
    - 500 on computation failures (logged)

- Swagger
  - `@ApiBearerAuth()`, `@ApiOperation`, `@ApiResponse` with example schemas
  - DTOs: `PortfoliosDashboardDto`, `PortfoliosSummaryDto`, `PortfolioCardDto`, `PortfolioAllocationItemDto`

## 9) Non-Functional Requirements
- Performance: one endpoint call; batch load transactions/assets/prices; avoid N+1.
- Caching: optional per-user short TTL (30–60s) for dashboard.
- Scalability: indexes on `transactions.userId`, `transactions.portfolioId`, `prices.assetId,timestamp`.
- Observability: log missing price cases; measure endpoint latency.

## 10) Acceptance Criteria
- FE page renders using BE response with no mock data.
- Summary totals and counts reflect user data.
- Portfolio cards show:
  - Correct current value and MTD metrics
  - Allocation percentages sum to ~100% (±0.5 due to rounding)
  - Asset counts match active positions
  - Reasonable lastUpdated
- Swagger docs published for endpoint with examples.
- Unit tests cover:
  - Portfolios with/without prices
  - Multi-asset portfolios across types
  - MTD zero-baseline (no prior price) handling
  - Multi-currency conversions

## 11) Dependencies & Risks
- Requires reliable `Price` coverage; gaps lower accuracy.
- Transactions without `portfolioId` excluded from portfolio metrics.
- Currency conversion must be deterministic and performant.

## 12) Future Enhancements (Not in this PRD)
- GET `/portfolios/:id/performance?range=1m|3m|1y|ytd` returning timeseries
- Filters/sorting for portfolios list
- Persisted risk scoring and strategy metadata

- Implement a single dashboard endpoint to feed `PortfoliosPage.tsx`, delivering summary and card arrays.
- Defined precise computation for holdings, current value, MTD performance, and allocation using existing `Portfolio`, `Transaction`, `Asset`, `Price`.
- Provided exact response schema aligned to `PortfolioCard` and `PortfolioSummaryStats`, with color mapping and Swagger expectations.