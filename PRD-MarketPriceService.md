## Market Price Service — Brownfield Enhancement PRD

### Intro Project Analysis and Context

#### Existing Project Overview
- Analysis Source: IDE-based fresh analysis of current repository.

- Current Project State
  - Backend (NestJS + TypeORM): Has `prices` module with storage and basic CRUD + latest lookup. No external market ingestion yet.
    ```1:22:be/src/prices/entities/price.entity.ts
    import { Entity, PrimaryGeneratedColumn, Column, CreateDateColumn } from 'typeorm';

    @Entity('prices')
    export class Price {
      @PrimaryGeneratedColumn('uuid')
      id: string;

      @Column('uuid')
      assetId: string;

      @Column({ type: 'decimal', precision: 19, scale: 4 })
      price: string;

      @Column({ type: 'timestamptz' })
      timestamp: Date;

      @Column({ type: 'varchar', length: 100 })
      source: string;

      @Column({ type: 'varchar', length: 50 })
      market: string;
      // ...
    }
    ```
    ```1:60:be/src/assets/entities/asset.entity.ts
    export enum AssetType {
      SAVINGS = 'SAVINGS',
      STOCK = 'STOCK',
      BOND = 'BOND',
      CRYPTO = 'CRYPTO',
      ETF = 'ETF',
      MUTUAL_FUND = 'MUTUAL_FUND',
      REAL_ESTATE = 'REAL_ESTATE',
      COMMODITY = 'COMMODITY',
      CASH = 'CASH',
      COLLECTIONS = 'COLLECTIONS',
      OTHERS = 'OTHERS',
    }
    ```
    ```1:64:be/src/prices/prices.controller.ts
    @Get('latest')
    @ApiOperation({ summary: 'Get the latest price for an asset' })
    async getLatestPriceForAsset(@Query('assetId') assetId: string) {
      return this.pricesService.getLatestPriceForAsset(assetId);
    }
    ```
  - Frontend: Currently uses mock portfolio data; no clear live price consumption path yet.
  - Python `price_service`: Scaffolded with `vnstock==3.2.3` in `requirements.txt`, but no business logic implemented.

#### Available Documentation Analysis
- Available:
  - Tech stack rules and project structure docs
  - Existing PRD: `docs/PRD-NetWorth.md`
  - Portfolio page feed doc: `docs/PortfolioPageFeed.md`
- Missing:
  - No current “price ingestion” or “market data integration” docs
  - No symbol registry documentation

#### Enhancement Scope Definition
- Enhancement Type
  - New Feature Addition
  - Integration with New Systems (data providers)
  - Performance/Scalability Improvements

- Enhancement Description
  - Build a provider-agnostic price ingestion service. Initial provider: VN market via `vnstock` for STOCK, BOND, MUTUAL_FUND. Operate via scheduled jobs (hourly/daily) to ingest, deduplicate, and persist prices. Expose BE endpoints that serve cached prices without per-request third-party calls. Design for future multi-market expansion.

- Impact Assessment
  - Moderate to Significant: Adds a new Python microservice + BE APIs + DB additions (symbol registry, ingestion logs), with operational components (scheduling, rate limiting, monitoring).

#### Goals and Background Context
- Goals
  - Reliable price coverage for supported asset types with scheduled ingestion
  - No direct per-request dependency on external providers
  - Provider-agnostic design for future markets/providers
  - Efficient batch updates with rate limiting and idempotency
  - Backfill capability and admin-triggered re-ingest
  - Basic observability (health, metrics, logs)

- Background
  - Today, the app lacks an external market price pipeline; scaling would be constrained by provider rate limits and latency. This enhancement establishes a durable ingestion platform to support growth and future providers.
  - Target provider for VN market: `vnstock` Python library. References: [`vnstock` GitHub repo](https://github.com/thinh-vu/vnstock).

#### Change Log
- [Init] 2025-08-09 — v0.1 — Draft PRD for price ingestion platform (VN market first) — Author: CFPM

### Requirements

These requirements are based on the current repository’s price storage, assets model, and the plan to add a provider-agnostic ingestion microservice. Please review for alignment.

#### Functional (FR)
1. FR1: Implement a Python `price_service` that ingests VN market prices for `STOCK`, `BOND`, and `MUTUAL_FUND` using `vnstock==3.2.3` on a schedule (hourly/daily).
2. FR2: Introduce a `symbol_registry` mapping (`assetId` → { market, provider, providerSymbol, type }) to resolve provider symbols for ingestion.
3. FR3: Support batched retrieval per provider when possible to reduce external calls (e.g., board-level snapshots).
4. FR4: Persist ingested quotes to `prices` with idempotency (unique key: `assetId + timestamp + source`).
5. FR5: Extend BE to expose `GET /prices/series?assetId&from&to&interval` and keep `GET /prices/latest?assetId` compatible.
6. FR6: Provide a backfill CLI in `price_service` to ingest historical daily data by date range and symbol list.
7. FR7: Add an internal BE endpoint `POST /internal/prices/ingest` (authenticated by shared secret) to accept bulk ingest payloads from `price_service`.
8. FR8: Implement retry with jitter, circuit breaker, and exponential backoff at provider-call boundaries.
9. FR9: Add metrics endpoints/logging for ingestion runs (counts, errors, durations) and basic health checks.
10. FR10: Provide an admin-triggered reingest for a symbol/date range (via BE protected endpoint).
11. FR11: Normalize timestamps to UTC; store market session date; handle exchange holidays and partial days.
12. FR12: Design provider interface in `price_service` to add future providers without breaking existing flows.

#### Non-Functional (NFR)
1. NFR1: Throughput: sustain ingestion for 5k symbols/hour with < 5% provider error rate using retries.
2. NFR2: Latency: latest price endpoints must serve from DB within 100ms p95 (excluding network).
3. NFR3: Reliability: recover from transient provider failures without data loss; reruns idempotent.
4. NFR4: Rate limiting: adhere to provider limits via token buckets and batch scheduling.
5. NFR5: Observability: logs with correlation IDs per run; basic metrics exported (prometheus-ready).
6. NFR6: Security: internal ingest endpoints protected via IP allowlist and shared secret, rotateable via env.
7. NFR7: Data quality: validate numeric ranges; reject obviously bad prices; support alerting on outliers.
8. NFR8: Storage: retain tick-level or daily OHLCV as configured; default: daily close and intraday latest where available.
9. NFR9: Backward compatibility: no breaking changes to existing `GET /prices/latest`.

#### Compatibility Requirements (CR)
- CR1: Keep `GET /prices/latest?assetId` behavior intact; only improve data freshness and reliability.
- CR2: DB schema changes additive; no destructive modifications to existing tables.
- CR3: UI/UX: frontend should continue to work without changes; new endpoints optional.
- CR4: Integration: `assets` module remains the source of truth for asset definitions; registry augments with market mapping.

### Technical Constraints and Integration Requirements

#### Existing Technology Stack
- Languages: TypeScript (NestJS), Python (ingestion)
- Frameworks: NestJS 11, TypeORM 0.3.x; Python with `vnstock` 3.2.3
- Database: PostgreSQL (prices, assets, future registry/log tables)
- Infrastructure: Containerized services; cron or scheduler (cron/K8s CronJob/GitHub Actions as bootstrap)
- External Dependencies: `vnstock` provider initially; future providers to be added

#### Integration Approach
- Database Integration Strategy
  - Add tables:
    - `market_symbol_registry(id, assetId, market, provider, providerSymbol, type, metadata, createdAt, updatedAt)`
    - `ingestion_runs(id, provider, startedAt, finishedAt, status, counts, errorSummary, parameters)`
    - Optional: `daily_prices` (optimized daily OHLCV) for faster trend queries
  - Persist via BE ingest endpoint or direct DB access. Start with BE ingest endpoint for validation/auditing.

- API Integration Strategy
  - Existing:
    - `GET /prices/latest?assetId` (unchanged)
  - New:
    - `GET /prices/series?assetId&from&to&interval=1D|1H|1M` → time series from stored data
    - `POST /internal/prices/ingest` (bulk payload with secret header)
    - `POST /admin/prices/reingest` (symbol + date range) — admin guard
  - Swagger: add examples and schemas for new endpoints.

- Frontend Integration Strategy
  - Continue consuming `GET /prices/latest` for MVP. Optionally add `series` later for charts.

- Testing Integration Strategy
  - Unit: Providers mocked in `price_service`; Nest service/controller tests for new endpoints.
  - E2E: Ingestion run against a small symbol set; verify DB writes and endpoint reads.

#### Code Organization and Standards
- Backend (`be/src/prices`): add controllers/dto for series, internal ingest, and admin reingest; follow Swagger rules.
- Python `price_service`:
  - `/providers/vnstock_provider.py`
  - `/registry/loader.py` to load mappings from BE endpoint or DB
  - `/ingest/run_ingest.py` (scheduler entry) and `/ingest/backfill.py`
  - `/models/ingest_payload.py` (pydantic) aligned with BE DTOs
  - `.env` for provider config, secrets, and schedules

#### Deployment and Operations
- Build Process Integration: CI builds both BE and `price_service` images.
- Deployment Strategy: Deploy `price_service` as a separate container; schedule via cron or K8s CronJob. BE updated in tandem.
- Monitoring and Logging: Structured logs (JSON), metrics exporter (optional), health endpoints for both services.
- Configuration Management: Environment variables for provider keys (if any), shared secret for internal ingest, rate limits.

#### Risk Assessment and Mitigation
- Technical Risks: Provider breaking changes; data shape differences; timezones/holidays; large backfills.
- Integration Risks: Duplicate symbols; missing mappings; schema drift.
- Deployment Risks: Scheduler overlap; long-running backfills impacting DB.
- Mitigation Strategies: Idempotent upserts, feature flags for new providers, staged rollouts, max batch sizes, backpressure.

### Epic and Story Structure

#### Epic Approach
- Epic Structure Decision: Single epic for “Provider-Agnostic Price Ingestion Platform” to minimize risk and coordinate DB + BE + Python service changes coherently.

### Epic 1: Provider-Agnostic Price Ingestion Platform

**Epic Goal**: Deliver a scalable scheduled ingestion platform that persists VN market prices (via `vnstock`) and serves them through stable BE endpoints without per-request provider calls.

**Integration Requirements**: Maintain backward-compatible BE API; additive DB schema; Python service deployable independently; secure internal ingest.

- Story 1.1 Symbol Registry and Mapping
  - As an admin, I want a `market_symbol_registry` to map our assets to provider symbols so the ingestion knows what to fetch.
  - Acceptance Criteria
    1: Table created; CRUD endpoints or seed loader available
    2: Entries include `assetId, market, provider, providerSymbol, type`
    3: Registry export API for `price_service` to pull
  - Integration Verification
    - IV1: Existing assets unaffected
    - IV2: Prices service reads registry and logs count
    - IV3: No performance regression on asset operations

- Story 1.2 BE Endpoints for Series and Internal Ingest
  - As a developer, I want `GET /prices/series` and `POST /internal/prices/ingest` so FE can query time series and `price_service` can push data.
  - Acceptance Criteria
    1: Swagger-documented endpoints with examples
    2: Secret-authenticated internal ingest; validates payload; upserts idempotently
    3: Series endpoint returns aggregated data efficiently
  - Integration Verification
    - IV1: `GET /prices/latest` behavior unchanged
    - IV2: Ingest writes visible via series and latest
    - IV3: p95 latency for reads within target

- Story 1.3 Python Price Service Skeleton + vnstock Provider
  - As an engineer, I want a Python service with a provider interface and vnstock implementation.
  - Acceptance Criteria
    1: Provider abstraction defined
    2: vnstock provider fetches batch quotes for symbols
    3: Unit tests with provider mocks
  - Integration Verification
    - IV1: Adheres to rate limits config
    - IV2: Produces ingest payload matching BE DTO
    - IV3: Handles VN holidays gracefully

- Story 1.4 Ingestion Pipeline and Idempotency
  - As an operator, I want reliable, deduplicated ingestion with retries and circuit breaking.
  - Acceptance Criteria
    1: Unique constraint or upsert strategy on (`assetId`,`timestamp`,`source`)
    2: Retries with backoff for transient failures
    3: Clear run logs with counts/errors
  - Integration Verification
    - IV1: No duplicate rows after multiple runs
    - IV2: Error rate observable in metrics
    - IV3: DB load within acceptable bounds

- Story 1.5 Scheduler and Batching
  - As an operator, I want scheduled hourly/daily jobs with batched symbol fetching.
  - Acceptance Criteria
    1: Cron schedule configurable
    2: Batches respect provider limits
    3: Partial failures retried next run
  - Integration Verification
    - IV1: No per-request provider usage from FE
    - IV2: Latest prices updated as per schedule
    - IV3: Rate limit incidents drop below target

- Story 1.6 Backfill CLI and Admin Reingest
  - As an admin, I want to backfill and reingest specific ranges.
  - Acceptance Criteria
    1: CLI supports date ranges and symbol filters
    2: `POST /admin/prices/reingest` triggers job safely
    3: Backfill throttles to protect DB and provider
  - Integration Verification
    - IV1: Idempotent reruns
    - IV2: No data corruption on overlaps
    - IV3: Monitoring shows backfill progress

- Story 1.7 Observability and Alerts
  - As an operator, I want metrics, logs, and basic alerts.
  - Acceptance Criteria
    1: Health endpoint for `price_service`
    2: Metrics for runs, errors, durations
    3: Alerting hooks (threshold-based)
  - Integration Verification
    - IV1: Dashboards display live metrics
    - IV2: Log fields trace a run end-to-end
    - IV3: Alert fires on repeated failures

- Story 1.8 Multi-Provider Abstraction (Forward Compatibility)
  - As a team, we want an interface to add more markets later without breaking existing flows.
  - Acceptance Criteria
    1: Provider interface stable and documented
    2: Example “stub” provider added and tested
    3: Registry supports multiple providers per asset
  - Integration Verification
    - IV1: vnstock path unchanged
    - IV2: Switching providers by config works
    - IV3: Backward compatibility maintained

### References
- Current BE price entity and endpoints:
  - `be/src/prices/entities/price.entity.ts`
  - `be/src/prices/prices.controller.ts`
  - `be/src/prices/prices.service.ts`
- Asset types: `be/src/assets/entities/asset.entity.ts`
- Python dependency: `price_service/requirements.txt` (uses `vnstock==3.2.3`)
- Provider documentation: `vnstock` GitHub repository: [thinh-vu/vnstock](https://github.com/thinh-vu/vnstock)
