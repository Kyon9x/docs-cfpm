## Frontend Specification — Assets Page Integration

Reference: `docs/PRD-AssetsPageIntegration.md`

### 1) Scope
- Replace mock data in `fe/src/pages/AssetsPage.tsx` with real data from `GET /assets/summary`.
- Maintain existing UI and translations; wire API and render dynamic values.
- Handle loading, error, and empty states.

### 2) API Contract
- Endpoint: `GET /assets/summary`
- Query params:
  - `includeInactive?: boolean` (default: false)
  - `types?: CSV` e.g., `STOCK,BOND`
- Response (per PRD):
```
{
  "totals": {
    "totalValue": number,
    "totalAssets": number,
    "activeAssetsCount": number,
    "assetTypesCount": number
  },
  "bestPerformer": {
    "assetId": string,
    "name": string,
    "performancePct": number | null
  } | null,
  "groups": Array<{ type: string; count: number; totalValue: number }>,
  "assets": Array<{
    id: string;
    name: string;
    type: string; // BE enum
    isActive: boolean;
    description?: string;
    risk: string; // BE enum
    details: {
      currentValue?: number;
      shares?: number;
      purchasePrice?: number;
      purchaseDate?: string; // ISO date
      currency?: string;
    };
    updatedAt: string; // ISO timestamp
  }>
}
```

Notes:
- FE uses BE enums directly for `AssetType` and `Risk`. No mapping/helpers.

### 3) TypeScript Models (FE)
Create local types aligned with BE enums:

```ts
export type AssetType =
  | 'SAVINGS' | 'STOCK' | 'BOND' | 'CRYPTO' | 'ETF' | 'MUTUAL_FUND'
  | 'REAL_ESTATE' | 'COMMODITY' | 'CASH' | 'COLLECTIONS' | 'OTHERS';

export type Risk = 'LOW' | 'MEDIUM_LOW' | 'MEDIUM' | 'MEDIUM_HIGH' | 'HIGH';

export interface AssetsSummaryTotals {
  totalValue: number;
  totalAssets: number;
  activeAssetsCount: number;
  assetTypesCount: number;
}

export interface AssetsSummaryGroup {
  type: AssetType; // BE AssetType
  count: number;
  totalValue: number;
}

export interface AssetsSummaryBestPerformer {
  assetId: string;
  name: string;
  performancePct: number | null;
}

export interface AssetsSummaryAsset {
  id: string;
  name: string;
  type: AssetType;
  isActive: boolean;
  description?: string;
  risk: Risk;
  details: {
    currentValue?: number;
    shares?: number;
    purchasePrice?: number;
    purchaseDate?: string;
    currency?: string;
  };
  updatedAt: string;
}

export interface AssetsSummaryResponse {
  totals: AssetsSummaryTotals;
  bestPerformer: AssetsSummaryBestPerformer | null;
  groups: AssetsSummaryGroup[];
  assets: AssetsSummaryAsset[];
}
```

### 4) Service Layer
Add to `fe/src/services/assetService.ts`:
```ts
export interface GetAssetsSummaryParams {
  includeInactive?: boolean;
  types?: AssetType[];
}

export async function getAssetsSummary(params: GetAssetsSummaryParams = {}): Promise<AssetsSummaryResponse> {
  const query: Record<string, string> = {};
  if (params.includeInactive) query.includeInactive = 'true';
  if (params.types?.length) query.types = params.types.join(',');
  const search = new URLSearchParams(query).toString();
  const url = `/assets/summary${search ? `?${search}` : ''}`;
  const res = await api.get(url);
  return res.data as AssetsSummaryResponse;
}
```

Optional: TanStack Query hook for caching/dedup.
```ts
export function useAssetsSummary(params: GetAssetsSummaryParams = {}) {
  return useQuery({
    queryKey: ['assets-summary', params],
    queryFn: () => getAssetsSummary(params),
    staleTime: 30_000,
  });
}
```

### 5) UI Integration Plan (`fe/src/pages/AssetsPage.tsx`)
Replace mock data with fetched summary:
1. Data fetch:
   - Use `useAssetsSummary()` or call `getAssetsSummary()` in `useEffect`.
2. Summary widgets:
   - Total value: `data.totals.totalValue`.
   - Asset types count: `data.totals.assetTypesCount`.
   - Best performer: `data.bestPerformer?.performancePct` and `name`.
   - Active assets: `data.totals.activeAssetsCount`.
3. Category cards:
   - Iterate `data.groups`.
   - For display label/icon use existing `assetTypeConfig` with BE `AssetType` values.
4. Table rows:
   - Source: `data.assets`.
   - `currentValue`: `asset.details.currentValue ?? 0`.
   - `shares`: `asset.details.shares ?? '—'`.
   - `risk`: use existing `riskConfig` directly.
   - `performance`: use `bestPerformer` for card; per-row performance compute locally only when `shares`, `purchasePrice`, and `currentValue` present.
5. Filters (phase 2):
   - UI toggles for `includeInactive` and multi-select `types` → pass to hook/service.

Loading & error states:
- While loading: skeletons/placeholders for summary cards and table.
- On error: show `ErrorBoundary` or inline error with retry.
- Empty: show existing empty copy when `assets.length === 0`.

### 6) i18n
- Reuse existing keys under `assets.*` in `fe/src/locales/en.json` and `vi.json`.
- Add keys if missing (examples):
```json
{
  "assets": {
    "summaryStats": {
      "bestPerformer": "Best performer",
      "activeAssets": "Active assets"
    },
    "table": {
      "totalValue": "Total value"
    }
  }
}
```

### 7) Validation & Edge Cases
- Missing prices: fall back to `details.currentValue`.
- Missing `shares`/`purchasePrice`: `performancePct` may be null; display `—`.
- Zero assets: render empty states for groups and table.

### 8) Accessibility
- Maintain semantic headings and button labels.
- Ensure icons have accessible labels via surrounding text.
- Preserve keyboard navigation in dropdown/menu.

### 9) Performance
- Prefer React Query for request dedup and caching.
- Memoize derived computations with `useMemo` if needed.
- Avoid per-row API calls.

### 10) Risks & Mitigations
- Enum mismatches: Not applicable; FE aligned to BE enums.
- Large asset lists: rely on single API call; consider pagination/virtualization later if needed.

### 11) Implementation Checklist
- [ ] Add service method `getAssetsSummary` and optional hook `useAssetsSummary`.
- [ ] Integrate into `AssetsPage.tsx` replacing mock data.
- [ ] Render summary cards and table from API data.
- [ ] Add loading, error, empty states.
- [ ] Update i18n keys if missing.
- [ ] Quick smoke test with a mocked API payload.

### 12) Test Plan
- Unit: type definitions, value formatting.
- Integration: render `AssetsPage` with mocked `AssetsSummaryResponse` → verify totals, best performer, groups, table rows.

### 13) Out of Scope (for now)
- Redis cache (BE) — may be considered later; in-memory cache only.
- Client-side filters for `types` and `includeInactive`.
- Per-row performance supplied by API.
- Column sorting and search.


