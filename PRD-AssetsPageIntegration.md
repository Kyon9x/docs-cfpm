## Assets Page Integration — Brownfield Enhancement PRD

### 1) Intro Project Analysis and Context

#### Existing Project Overview
- Analysis Source: IDE-based fresh analysis of current repository.

- Current Project State
  - Frontend Assets page renders from mock data and expects:
    - Per-asset fields: `id`, `name`, `type`, `isActive`, `description`, `risk`, `details.currentValue`, `details.shares`, `details.purchasePrice`, `details.purchaseDate`, `details.currency`, timestamps.
    - Summary widgets: `totalValue` (sum of currentValue), count of asset types present, best performer (by performance%), active assets count, category totals.
  - Backend:
    - Assets CRUD with JSONB `details` and enums for `AssetType` and `Risk`.
    - Prices storage (`prices` table) with CRUD and `GET /prices/latest?assetId`.
    - No consolidated summary endpoint yet combining Assets + latest Prices.

- Notable data model gaps for FE expectations:
  - FE `Risk` includes `VERY_HIGH`; BE `Risk` enum does not (has `MEDIUM_HIGH`, `MEDIUM_LOW`). Map on FE or treat as `HIGH` for now.
  - FE `AssetType` includes `GOLD`; BE does not. Map `GOLD → COMMODITY` for now.
  - Performance% needs `shares` and `purchasePrice` from `assets.details` plus latest price.

#### Available Documentation Analysis
- Available: `docs/PRD-NetWorth.md`, `docs/PortfolioPageFeed.md`, BE/FE project structure and standards.
- Missing: No “assets summary” API spec; no explicit holdings schema beyond `Asset.details`.

#### Enhancement Scope Definition
- Enhancement Type
  - New Feature Addition
  - Integration with New Systems (consumption of BE prices within assets summary)
  - Performance/Scalability Improvements (aggregated summary and caching)

- Enhancement Description
  - Add a new summary API (`GET /assets/summary`) that returns all data needed by the FE Assets page in a single request, computed from Assets + latest Prices, with optional filters. FE replaces mock data with this API.

- Impact Assessment
  - Moderate: New controller/service logic in `assets` module; reuse existing `prices` data; FE integration change; Swagger documentation; optional short-ttl cache.

### 2) Goals and Background Context
- Goals
  - Provide all Assets page data via a single backend call.
  - Avoid per-row provider calls; rely on stored Prices only.
  - Maintain backward compatibility with existing Assets endpoints.
  - Keep computation fast and cacheable.

- Background
  - Assets page is currently mock-driven. We have stored asset definitions and price history but lack a consolidated summary endpoint.

### 3) Change Log
- [Init] 2025-08-09 — v0.1 — Draft PRD for `assets/summary` integration — Author: CFPM

### 4) Requirements

These requirements are based on the current FE expectations and BE modules. Please review alignment.

#### Functional Requirements (FR)
1. FR1: Implement `GET /assets/summary` that returns:
   - `totals`: `{ totalValue, totalAssets, activeAssetsCount, assetTypesCount }`
   - `bestPerformer`: `{ assetId, name, performancePct }`
   - `groups`: `[{ type, count, totalValue }]`
   - `assets`: `[{ id, name, type, isActive, description, risk, details: { currentValue, shares?, purchasePrice?, purchaseDate?, currency? }, updatedAt }]`
2. FR2: Compute per-asset `currentValue` as:
   - If `details.shares` and latest price available: `currentPrice = latestPrice`; `currentValue = shares * currentPrice`.
   - Else fallback to `details.currentValue` when present.
3. FR3: Compute `performancePct` when both `shares` and `purchasePrice` are available:
   - `performance = ((currentPrice - purchasePrice) / purchasePrice) * 100`; otherwise `null`.
4. FR4: Group totals per `AssetType` with counts; include only types with at least one asset.
5. FR5: Support optional filters: `?includeInactive=false|true`, `?types=STOCK,BOND,...`.
6. FR6: Return timestamps in ISO 8601 UTC; return currency when available.
7. FR7: Swagger docs with request/response examples and schemas.
8. FR8: FE integrates to replace mocks; display remains consistent.
9. FR9: No provider calls during request handling; summary uses DB-only data.
10. FR10: Add simple server-side cache (in-memory) per filter combination with short TTL (30–60s).

#### Non-Functional Requirements (NFR)
1. NFR1: p95 latency ≤ 150ms for `GET /assets/summary` on ~1k assets (warm cache).
2. NFR2: CPU/memory stable under typical load; no excessive allocations.
3. NFR3: Cache staleness ≤ 60s; invalidation on write not required for MVP.
4. NFR4: Endpoint documented and covered by unit tests; at least one integration test.
5. NFR5: Secure by existing auth guard if applicable; otherwise, keep parity with current assets endpoints.

#### Compatibility Requirements (CR)
- CR1: Do not change existing `GET /assets`, `GET /assets/:id` behaviors.
- CR2: Keep `prices` module behavior intact; only read latest prices.
- CR3: FE `Risk` and `AssetType` mapping handled on FE without changing BE enums (`VERY_HIGH`/`GOLD` mapped accordingly).
- CR4: Response is additive and does not require UI changes beyond wiring.

### 5) Technical Constraints and Integration Requirements

#### Existing Technology Stack
- BE: NestJS 11, TypeORM 0.3, PostgreSQL; existing `assets` and `prices` modules.
- FE: React + TypeScript; Assets page expects typed structure for summary and rows.

#### Integration Approach

- Database Integration Strategy
  - No schema changes required for MVP.
  - Use `prices` table for latest price by asset via `ORDER BY timestamp DESC LIMIT 1`.
  - Read `assets.details` JSONB for `shares`, `purchasePrice`, `currency`, etc.

- API Integration Strategy
  - New Controller method in `be/src/assets/`:
    - `GET /assets/summary`
      - Query params: `includeInactive?=false`, `types?=CSV_ENUM`
      - Response example:
        ```json
        {
          "totals": {
            "totalValue": 123456.78,
            "totalAssets": 6,
            "activeAssetsCount": 5,
            "assetTypesCount": 4
          },
          "bestPerformer": {
            "assetId": "uuid",
            "name": "Tesla Inc.",
            "performancePct": 27.3
          },
          "groups": [
            { "type": "STOCK", "count": 3, "totalValue": 44123.45 }
          ],
          "assets": [
            {
              "id": "uuid",
              "name": "Apple Inc.",
              "type": "STOCK",
              "isActive": true,
              "description": "Technology company stock",
              "risk": "MEDIUM",
              "details": {
                "currentValue": 18525.5,
                "shares": 100,
                "purchasePrice": 150.0,
                "purchaseDate": "2024-01-15",
                "currency": "USD"
              },
              "updatedAt": "2024-01-15T10:00:00Z"
            }
          ]
        }
        ```
  - Swagger: Provide examples per above and error cases (no assets). Follow `be/swagger-docs` rule.

- Frontend Integration Strategy
  - Replace mock assets with data from `GET /assets/summary`.
  - Map BE `AssetType.COMMODITY → GOLD` for display when needed; map BE risks to FE labels (optionally treat `VERY_HIGH` as `HIGH` for now).
  - Keep existing UI structure; plug values from response.

- Testing Integration Strategy
  - Unit: Summary service computations (with repo mocks).
  - Controller test: filters and Swagger.
  - FE smoke test to render with API payload and empty state.

#### Code Organization and Standards
- Backend
  - Add `assets-summary.dto.ts` for response typing.
  - Add `@Get('summary')` in `AssetsController` or a dedicated controller.
  - Implement `AssetsService.getSummary(params)` which:
    - Fetches assets by filters
    - For each asset, fetch latest price via repository query (MVP: per-asset; follow-up: optimize with subquery or DISTINCT ON)
    - Computes `currentValue`, `performancePct`
    - Aggregates totals and groups
  - Add in-memory cache with TTL from env (e.g., `ASSETS_SUMMARY_TTL=60`).
  - Swagger annotations and examples.

- Frontend
  - Create `src/services/assetService.ts` with `getAssetsSummary()`.
  - Wire into `AssetsPage.tsx`, replacing mocks and handling loading/empty states.

#### Deployment and Operations
- No new service. Standard BE deploy.
- Cache TTL configurable via env.

#### Risk Assessment and Mitigation
- Technical Risks: N+1 queries for price lookups. Mitigation: future optimization via subqueries or materialized views.
- Data Risks: Missing `shares/purchasePrice` yields null performance; FE handles gracefully.
- Integration Risks: Enum mismatches. Mitigation: FE mapping layer for GOLD/VERY_HIGH.

### 6) Epic and Story Structure

#### Epic Approach
- Single epic, as this is a coherent enhancement touching one BE endpoint and one FE page integration.

#### Epic 1: Assets Summary API and FE Integration

- Story 1.1 BE: Add `GET /assets/summary`
  - As an API consumer, I want a single endpoint to retrieve all summary data for the Assets page.
  - Acceptance Criteria
    1. Endpoint returns `totals`, `bestPerformer`, `groups`, `assets[]`.
    2. Filters `includeInactive`, `types` supported.
    3. Swagger docs with examples; 200/400/500 responses.
  - Integration Verification
    - Existing assets endpoints unchanged; latency within target; response shape matches FE needs.

- Story 1.2 BE: Compute values from Assets + Prices
  - As a developer, I want per-asset `currentValue` and `performancePct` computed correctly.
  - Acceptance Criteria
    1. Uses latest price by `timestamp` where available.
    2. Fallback to `details.currentValue` when price/shares missing.
    3. Unit tests cover edge cases (no prices, zero shares, missing purchasePrice).

- Story 1.3 BE: Add short-ttl cache
  - As an operator, I want response caching to reduce DB load.
  - Acceptance Criteria
    1. Configurable TTL; cache keyed by filters.
    2. Optional bypass via header for admins.

- Story 1.4 FE: Integrate Assets page
  - As a user, I want to see real data instead of mocks.
  - Acceptance Criteria
    1. Replace mock array with API data.
    2. Correct totals, best performer, groups, and table.
    3. Risk/type label mapping rendered as before.

- Story 1.5 E2E: Contract test
  - Acceptance Criteria
    1. JSON contract validated.
    2. Swagger examples align with live response.
    3. At least one integration test path.

### 7) Acceptance Criteria (end-to-end)
- `GET /assets/summary` available with Swagger docs and examples.
- FE `AssetsPage` loads from API, not mocks, and renders all widgets and table with correct values.
- Computation uses stored latest prices; no external provider calls during request handling.
- p95 latency ≤ 150ms on ~1k assets with cache warm.

### 8) Performance
- Short-ttl server-side cache (30–60s).
- Optimize N+1 in a follow-up if needed.

### 9) Security/Privacy
- Same auth scope as other assets endpoints.
- No sensitive data returned.

### 10) Timeline (target 2–3 days)
- Day 1: BE endpoint + computation + Swagger + tests.
- Day 2: FE integration + mapping + states.
- Day 3: Cache, polish, and contract tests.

### 11) References
- FE page: `fe/src/pages/AssetsPage.tsx`
- BE storage and endpoints:
  - `be/src/assets/entities/asset.entity.ts`
  - `be/src/prices/entities/price.entity.ts`
  - `be/src/prices/prices.service.ts`, `be/src/prices/prices.controller.ts`


