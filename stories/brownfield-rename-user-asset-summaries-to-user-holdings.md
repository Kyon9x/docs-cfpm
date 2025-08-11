# Story: Rename user-asset-summaries module to user-holdings

## Status: Draft

## Story

As a developer and product owner,
I want to rename the `user-asset-summaries` module to `user-holdings`,
so that the domain language matches actual functionality and future extensions (per-asset holdings, performance metrics) are clearer.

## Context Source

- Source Document: User request in chat; existing codebase
- Enhancement Type: Refactor/Rename (module, entity, service, cron, controller & routes); no DB migration required (fresh DB allowed)
- Existing System Impact: Affects imports, providers, module references, entity names, schedule cron, service call sites (assets, transactions), and HTTP routes

## Acceptance Criteria

1. Codebase compiles and starts successfully after the rename.
2. All references to `UserAssetSummary` and `UserAssetSummaries*` are replaced by `UserHolding` and `UserHoldings*` respectively.
3. Entity table renamed to `user_holdings` with identical columns and indexes; no migration file required (fresh DB is acceptable).
4. HTTP routes are updated:
   - `POST /assets/user-update` is removed.
   - New `GET /user-holdings` returns holdings for the authenticated user.
   - New `POST /user-holdings` triggers recompute (all or single asset) for the authenticated user.
5. Existing features using the service still work via the new routes:
   - Transactions that affect holdings automatically trigger recomputation via the renamed service.
   - Cron continues to run on the same schedule and recomputes stale holdings.
6. Swagger documentation fully updated for the new `user-holdings` routes (operation, params/body, examples, responses) per project rule.
7. All imports and providers updated (`AppModule`, `AssetsModule`, `TransactionsService`, cron, etc.).
8. Unit tests (if present) compile and run; test names and route expectations updated to reflect Holdings.

## Dev Technical Guidance

### Existing System Context

- Current module: `src/user-asset-summaries/`
  - Entity: `entities/user-asset-summary.entity.ts` with table `user_asset_summaries`
  - Service: `user-asset-summaries.service.ts`
  - Cron: `user-asset-summaries.cron.ts`
  - Module: `user-asset-summaries.module.ts`
- References:
  - `src/app.module.ts` imports `UserAssetSummariesModule`
  - `src/assets/assets.module.ts` imports `UserAssetSummariesModule`
  - `src/assets/assets.controller.ts` injects `UserAssetSummariesService` and exposes `POST /assets/user-update`
  - `src/transactions/transactions.service.ts` injects `UserAssetSummariesService`

### Integration Approach

- Pure rename plus route relocation.
- Replace naming across files, directories, imports, class names, token names, table name, and Swagger descriptions.
- Remove the legacy route `POST /assets/user-update` and introduce a dedicated controller for `user-holdings` with `GET` and `POST`.
- No database migration file needed; set new entity to use table name `user_holdings` and rebuild DB as per user note.

### Technical Constraints

- NestJS/TypeORM conventions must be followed; keep `autoLoadEntities: true` behavior intact.
- Ensure circular dependency handling remains the same (e.g., forwardRef patterns are preserved if present).
- Maintain service method names and return types to minimize impact; adapt controller layer to new routes.
- Apply `JwtAuthGuard` and `@CurrentUser()` to the new routes just like the previous endpoint.

### Missing Information

- None required; all changes are internal refactors and route relocation.

## Tasks / Subtasks

- [ ] Create new module directory `src/user-holdings/` and port files with renamed classes and filenames:
  - [ ] `entities/user-holding.entity.ts` (from `user-asset-summary.entity.ts`)
    - Rename class `UserAssetSummary` → `UserHolding`
    - Change `@Entity('user_asset_summaries')` → `@Entity('user_holdings')`
    - Keep columns and indexes identical
  - [ ] `user-holdings.service.ts` (from `user-asset-summaries.service.ts`)
    - Rename class to `UserHoldingsService`
    - Update repository injection to `UserHolding`
    - Keep API: `recomputeForAsset`, `recomputeForUserAsset`, `recomputeAllStale`
  - [ ] `user-holdings.cron.ts` (from `user-asset-summaries.cron.ts`)
    - Rename class to `UserHoldingsCron`
    - Inject `UserHoldingsService`
  - [ ] `user-holdings.module.ts` (from `user-asset-summaries.module.ts`)
    - Rename class to `UserHoldingsModule`
    - Update `TypeOrmModule.forFeature([UserHolding, Price])`
    - Export `UserHoldingsService`
  - [ ] `user-holdings.controller.ts` (new)
    - Controller base path: `user-holdings`
    - Apply `@UseGuards(JwtAuthGuard)` where needed
    - `GET /user-holdings`
      - Returns list of holdings for current user (previously the rows of `user_asset_summaries`)
      - Optional query params (future-friendly): `assetId?: uuid`
      - Swagger: `@ApiOperation`, `@ApiResponse` with example list
    - `POST /user-holdings`
      - Body oneOf: `{ all: true }` or `{ assetId: 'uuid' }`
      - Calls `UserHoldingsService.recomputeAllStale()` or `recomputeForUserAsset(userId, assetId)`
      - Returns: `{ status: 'completed', all: true, assetsCount, holdingsCount }` OR `{ status: 'completed', assetId, recomputed }`
      - Swagger: `@ApiBody` with examples; `@ApiResponse` examples

- [ ] Update all imports/references:
  - [ ] `src/app.module.ts`: import and register `UserHoldingsModule` (remove old)
  - [ ] `src/assets/assets.module.ts`: import `UserHoldingsModule` (remove old)
  - [ ] `src/assets/assets.controller.ts`: remove the method and Swagger docs for `POST /assets/user-update` and the service injection if no longer needed
  - [ ] `src/transactions/transactions.service.ts`: inject `UserHoldingsService` in constructor and update usage
  - [ ] Any other references via grep for `UserAssetSumm` and `user-asset-summaries`

- [ ] Update Swagger descriptions where applicable:
  - [ ] New `user-holdings` controller: document `GET /user-holdings` and `POST /user-holdings` fully (operation summary, params/body, examples, responses)
  - [ ] Remove legacy `POST /assets/user-update` docs

- [ ] Remove old directory `src/user-asset-summaries/` after successful rename and build

- [ ] Run lint and build:
  - [ ] `npm run lint` (fix errors)
  - [ ] `npm run build`

- [ ] Verify runtime behavior manually (optional):
  - [ ] `POST /user-holdings` with `{ all: true }` returns `{ status: 'completed', all: true, assetsCount, holdingsCount }`
  - [ ] `POST /user-holdings` with `{ assetId: 'uuid' }` returns `{ status: 'completed', assetId, recomputed }`
  - [ ] `GET /user-holdings` returns holdings for current user
  - [ ] Create a BUY transaction to confirm recompute is still triggered automatically
  - [ ] Confirm cron triggers without errors

## Risk Assessment

### Implementation Risks

- Import path or provider token mismatches causing runtime injection errors.
- Overlooked references in services or modules leading to compilation failures.
- Swagger drift if wording is not updated consistently.
- Client code still calling removed route `POST /assets/user-update`.

### Mitigation

- Use grep-based sweeping replacement for `UserAssetSumm` and directory name.
- Build and lint after each logical step to catch errors early.
- Keep method signatures stable to minimize blast radius.
- Communicate route change in changelog and update any clients.

### Rollback Plan

- Revert to a branch/commit with `user-asset-summaries` intact and the legacy route in `assets.controller.ts`.
- If needed, restore the original directory and imports and re-run build.

## Notes

- DB: No migration needed; fresh DB will be used. Ensure entity `@Entity('user_holdings')` is configured.
- Follow NestJS module structure and keep services exported for cross-module usage.

## Definition of Done

- `src/user-holdings/` fully replaces `src/user-asset-summaries/`.
- Legacy `POST /assets/user-update` is removed and replaced by `GET /user-holdings` and `POST /user-holdings`.
- App compiles and starts with no runtime errors.
- Recompute endpoint and transaction-triggered recomputation continue to function.
- Cron continues to execute without errors.
- Swagger text references "Holdings" consistently and documents the new routes.
