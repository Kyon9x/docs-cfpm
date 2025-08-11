### Rename BE assets baseCurrency to currency - Brownfield Addition

#### User Story

As a backend developer,
I want the Asset top-level currency field to be consistently named `currency` (instead of `baseCurrency`),
So that the API and codebase use a single, clear term across modules and documentation.

#### Story Context

- Integrates with: `be/src/assets` module and dependent modules (`prices`, `transactions`)
- Technology: NestJS 11, TypeORM 0.3.x, Swagger, Jest
- Follows pattern: Additive DB changes, backward-compatible DTO handling, consistent Swagger docs
- Touch points:
  - `be/src/assets/entities/asset.entity.ts`
  - `be/src/assets/dto/create-asset.dto.ts`, `be/src/assets/dto/update-asset.dto.ts`
  - `be/src/assets/assets.controller.ts`, `be/src/assets/assets.service.ts`
  - References in `be/src/prices/prices.service.ts`, `be/src/transactions/transactions.service.ts`
  - Tests: `be/src/prices/prices.service.spec.ts`

#### Acceptance Criteria

- Functional Requirements:
  1. Rename top-level `Asset` field from `baseCurrency` to `currency` in entity and DTOs.
  2. API accepts both `currency` and legacy `baseCurrency` in request payloads; response payloads use `currency` only.
  3. Swagger docs, examples, and validation messages reference `currency` (not `baseCurrency`).

- Integration Requirements:
  4. Existing functionality remains unchanged for clients still sending `baseCurrency`.
  5. Follow existing pattern where other asset parts already use `currency` (e.g., `cash` details, `assets-summary.dto`).
  6. Dependent services (`prices`, `transactions`) compile and run without errors; logic updated to reference `currency`.

- Quality Requirements:
  7. Add a migration or an additive schema change: introduce `currency` column (nullable), backfill from `baseCurrency`, keep `baseCurrency` for now.
  8. Update and add unit tests to cover the aliasing behavior and ensure no regressions.
  9. Swagger is updated; lints/tests green.

#### Technical Notes

- Integration Approach:
  - Database:
    - Add `currency` column to `Asset` (varchar(10), nullable) and backfill from `baseCurrency`.
    - Keep `baseCurrency` temporarily for backward compatibility (future cleanup story to drop).
  - DTO/controller:
    - Update DTOs to primary `currency` field with validation; accept `baseCurrency` as deprecated alias by transforming input.
    - Update validation messages and examples in Swagger to `currency`.
  - Service layer:
    - Replace internal references to `baseCurrency` with `currency`.
    - For read operations, ensure responses expose `currency`. If `currency` is null but `baseCurrency` exists, return `baseCurrency` value in `currency` field as a temporary compatibility fallback.
  - Dependent modules:
    - `prices.service.ts` default quote currency should use `asset.currency` (fallback to deprecated if needed).
    - `transactions.service.ts` areas referencing `asset.baseCurrency` should read `asset.currency`. If some detail keys are named `assetBaseCurrency`, keep them as-is to avoid scope creep; only the Asset top-level field is renamed in this story.

- Existing Pattern Reference:
  - `be/src/assets/dto/assets-summary.dto.ts` uses `currency`.
  - `be/src/assets/dto/cash-asset-details.dto.ts` and interface use `currency`.

- Key Constraints:
  - Backward compatibility is mandatory in this story.
  - Keep DB change additive only; do not drop `baseCurrency` in this story.

#### Definition of Done

- [ ] `Asset` entity has `currency` column; `baseCurrency` remains for now.
- [ ] DTOs/controllers accept `currency` and deprecated `baseCurrency`, but respond with `currency` only.
- [ ] All references in `assets`, `prices`, `transactions` updated to use `currency` internally.
- [ ] Swagger docs updated to show `currency`.
- [ ] Tests updated and added; all pass.
- [ ] No regressions observed in related endpoints.

#### Minimal Risk Assessment

- Primary Risk: Breaking clients expecting `baseCurrency` responses or sending only `baseCurrency`.
- Mitigation: Accept both fields on input; for output, respond with `currency`, and for a temporary period map from `baseCurrency` if needed. Communicate deprecation of `baseCurrency`.
- Rollback: Revert code to previous state and keep `baseCurrency` as canonical; DB keeps additive `currency` column unused.

#### Compatibility Verification

- [ ] No breaking changes to existing APIs (input remains compatible; response field renamed but with fallback mapping).
- [ ] Database changes are additive only.
- [ ] UI/FE does not break if it relies on input with `baseCurrency`; FE should adapt to `currency` in a separate story if needed.
- [ ] Performance impact negligible.

#### Validation Checklist

- Scope Validation:
  - [ ] Story can be completed in one development session
  - [ ] Integration approach is straightforward
  - [ ] Follows existing patterns exactly
  - [ ] No design or architecture work required

- Clarity Check:
  - [ ] Story requirements are unambiguous
  - [ ] Integration points are clearly specified
  - [ ] Success criteria are testable
  - [ ] Rollback approach is simple

#### Follow-up

- Create a follow-up cleanup story to remove `baseCurrency` after client migration and DB data verification.

#### Files to touch (guidance)

- `be/src/assets/entities/asset.entity.ts`
- `be/src/assets/dto/create-asset.dto.ts`
- `be/src/assets/dto/update-asset.dto.ts`
- `be/src/assets/assets.controller.ts`
- `be/src/assets/assets.service.ts`
- `be/src/prices/prices.service.ts`
- `be/src/transactions/transactions.service.ts`
- Tests: `be/src/prices/prices.service.spec.ts` and any asset-related tests

#### Swagger updates

- Update schema names, examples, and error messages to reference `currency`.

#### Deprecation note (docs/CHANGELOG)

- Mark `baseCurrency` as deprecated (input accepted; not returned in responses).


