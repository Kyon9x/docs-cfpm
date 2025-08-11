# Story: FE – User Asset Summary Recompute Trigger (Brownfield Addition)

## Status: Draft

## User Story

As an authenticated user,
I want app to automatically trigger recomputation of my asset summaries,
so that I can quickly refresh current values and performance when prices or holdings change.

## Story Context

**Existing System Integration:**
- Integrates with: Backend `POST /assets/user-update` endpoint (JWT protected)
- Technology: React 19, TypeScript, Axios, TanStack Query
- Follows pattern: Existing service wrappers in `fe/src/services/*` and toasts/i18n patterns
- Touch points: `fe/src/services/assetService.ts`, `fe/src/pages/AssetsPage.tsx` (or a shared action location), toasts, i18n

## Acceptance Criteria

1. No manual recompute control in UI; background update only.
2. After successful actions that affect holdings/prices:
   - Transaction create/update (e.g., `createInvestmentTransaction`)
   - Manual price create (e.g., `createManualPrice`)
   FE shows a non-blocking info toast (i18n) indicating summaries will refresh shortly.
3. FE triggers a lightweight refresh of relevant data after success:
   - Invalidate or refetch TanStack Query caches for assets summaries/overall lists used in UI.
   - Use a short debounce (e.g., 2–5s) if immediate refresh is noisy.
4. i18n
   - Add keys for info/success: `assets.autoRefresh.enqueued`, `assets.autoRefresh.done` (optional).
5. Error handling remains on the original actions (transaction/price APIs); no new API calls introduced.

## Technical Notes

- Use existing toast utility (`src/hooks/useToast.ts` or `components/ui/sonner.tsx`) and i18n.
- In calling components (e.g., where transactions/prices are created), after a successful response:
  - Show info toast: `t('assets.autoRefresh.enqueued')`.
  - Invalidate queries via TanStack Query for any of:
    - assets summary endpoint usage
    - overall assets lists
    - any dashboard widgets that surface current values/performance
- Prefer invalidation in the component layer (not inside services) to keep services pure.

## Definition of Done

- [ ] API method implemented and tested against backend
- [ ] Button added and wires to API; disabled while loading
- [ ] Success and error toasts implemented with i18n strings
- [ ] Lint passes; types are strict; no console errors in dev

## Minimal Risk Assessment

- Primary Risk: Button appears but backend rejects unauthorized calls
- Mitigation: Ensure Auth token is attached; guard UI for authenticated users (existing guards)
- Rollback: Remove button and service method; no schema changes on FE

## Compatibility Verification

- [ ] No breaking changes to existing FE APIs
- [ ] UI change is additive and follows design
- [ ] Performance impact negligible (single POST)

## Tasks

- [ ] UI integration
  - [ ] In transaction create/update flows, after success: show info toast and invalidate relevant queries
  - [ ] In manual price create flow, after success: show info toast and invalidate relevant queries
- [ ] Query invalidation
  - [ ] Identify and invalidate keys used by assets summaries/overall and dashboard views
  - [ ] Optionally add a small debounce before refetch
- [ ] i18n
  - [ ] Add keys: `assets.autoRefresh.enqueued`, `assets.autoRefresh.done`
  - [ ] Provide EN and VI translations
- [ ] QA
  - [ ] Verify UI auto-refreshes summaries without manual button
  - [ ] Verify error handling on original actions remains intact
