# State Management Patterns — Zustand Feature Stores & TanStack Query

This reference describes when and how to use feature-level Zustand stores alongside
TanStack Query. Include this content in Section 9 of the generated specification and
in any module SPEC.md that requires client-side state beyond server cache.

The state model is identical to the MUI variant — TanStack Query owns server state, Zustand
owns UI state, React Hook Form owns form state. Only the rendered UI in the examples is
Tailwind/Headless UI instead of MUI.

---

## State Separation Principle

The golden rule for React SPA state management:

| State Type | Tool | Examples |
|---|---|---|
| **Server state** (data from API) | TanStack Query | Hero section list, user profile, blog posts |
| **UI state** (ephemeral, per-session) | Zustand | Selected rows, filter panel open/closed, active tab |
| **Global UI state** | Zustand (persisted) | Theme mode, sidebar open, user preferences |
| **Auth state** | Zustand (memory-only, no persist) | JWT token, logged-in user info |
| **Form state** | React Hook Form | All form field values and validation errors |

**Do NOT use Zustand to cache API responses.** If data comes from the server, TanStack
Query owns it. Zustand holds only the filters/pagination params that TanStack Query uses
as its query key.

---

## Feature Store Pattern

Create a feature store only when the module has complex UI state that needs to be shared
between sibling components (e.g., a list page and a detail panel rendered side-by-side).

### When to create a feature store

| Scenario | Use feature store? |
|---|---|
| List page with filter bar → filter values | **Yes** — filter state drives TanStack Query params |
| Multi-step form with shared wizard state | **Yes** — wizard step and accumulated values |
| List page + slide-over detail panel (side-by-side) | **Yes** — selected item ID |
| Simple list page with local filter `useState` | **No** — keep state local |
| Simple create/edit form | **No** — React Hook Form handles form state |

### Feature Store Template

```typescript
// Example: src/features/heroSection/store/heroSection.store.ts
import { create } from 'zustand';
import type { HeroSectionListParams } from '../heroSection.types';

interface HeroSectionState {
  // Filter and pagination params — these drive the TanStack Query key
  filters: HeroSectionListParams;

  // UI state
  selectedIds: Set<string>;
  isDetailPanelOpen: boolean;
  selectedId: string | null;

  // Actions
  setFilter: (key: keyof HeroSectionListParams, value: unknown) => void;
  resetFilters: () => void;
  setPage: (page: number) => void;
  toggleSelectId: (id: string) => void;
  clearSelection: () => void;
  openDetail: (id: string) => void;
  closeDetail: () => void;
}

const DEFAULT_FILTERS: HeroSectionListParams = { page: 0, size: 10, status: '' };

export const useHeroSectionStore = create<HeroSectionState>((set) => ({
  filters: { ...DEFAULT_FILTERS },
  selectedIds: new Set(),
  isDetailPanelOpen: false,
  selectedId: null,

  setFilter: (key, value) =>
    set((state) => ({ filters: { ...state.filters, [key]: value, page: 0 } })),

  resetFilters: () => set({ filters: { ...DEFAULT_FILTERS } }),

  setPage: (page) => set((state) => ({ filters: { ...state.filters, page } })),

  toggleSelectId: (id) =>
    set((state) => {
      const next = new Set(state.selectedIds);
      next.has(id) ? next.delete(id) : next.add(id);
      return { selectedIds: next };
    }),

  clearSelection: () => set({ selectedIds: new Set() }),

  openDetail: (id) => set({ selectedId: id, isDetailPanelOpen: true }),

  closeDetail: () => set({ isDetailPanelOpen: false, selectedId: null }),
}));
```

### Integration with TanStack Query

```tsx
// src/features/heroSection/pages/HeroSectionListPage.tsx
import { useHeroSectionStore } from '../store/heroSection.store';
import { useHeroSections } from '../hooks/useHeroSections';
import { SearchInput } from '../../../shared/components/SearchInput';
import { DataTable } from '../../../shared/components/DataTable';
import { Pagination } from '../../../shared/components/Pagination';

export function HeroSectionListPage() {
  // Filter state lives in Zustand — changes trigger React re-render
  const { filters, setFilter, setPage } = useHeroSectionStore();

  // TanStack Query uses filters as the query key — auto-refetches when filters change
  const { data, isLoading } = useHeroSections(filters);

  return (
    <div className="space-y-4">
      <div className="flex flex-wrap items-center gap-3">
        <SearchInput
          placeholder="Search…"
          onChange={(value) => setFilter('search', value)}
        />
        <select
          value={filters.status ?? ''}
          onChange={(e) => setFilter('status', e.target.value)}
          className="rounded border border-slate-300 bg-surface px-3 py-2 text-sm focus:border-primary focus:ring-1 focus:ring-primary dark:border-slate-600"
        >
          <option value="">All Statuses</option>
          <option value="ACTIVE">Active</option>
          <option value="DRAFT">Draft</option>
        </select>
      </div>

      {/* DataTable + Pagination — see routing-patterns.md for implementations */}
      <DataTable columns={columns} data={data?.content ?? []} emptyMessage={isLoading ? 'Loading…' : undefined} />
      <Pagination
        page={filters.page ?? 0}
        size={filters.size ?? 10}
        totalElements={data?.totalElements ?? 0}
        onPageChange={(page) => setPage(page)}
        onSizeChange={(size) => setFilter('size', size)}
      />
    </div>
  );
}
```

---

## Optimistic Updates Pattern

For fast UI feedback on mutations (useful for toggle operations like status changes):

```typescript
// src/features/heroSection/hooks/useHeroSectionMutations.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { heroSectionApi } from '../api/heroSectionApi';
import { heroSectionKeys } from './useHeroSections';
import type { HeroSection } from '../heroSection.types';

export function useToggleHeroSectionStatus() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, status }: { id: string; status: HeroSection['status'] }) =>
      heroSectionApi.update(id, { status }),

    // Optimistic update — immediately update the cache before the API responds
    onMutate: async ({ id, status }) => {
      await queryClient.cancelQueries({ queryKey: heroSectionKeys.all });
      const previousData = queryClient.getQueryData<HeroSection>(heroSectionKeys.detail(id));
      queryClient.setQueryData<HeroSection>(heroSectionKeys.detail(id), (old) =>
        old ? { ...old, status } : old,
      );
      return { previousData };
    },

    // On error, roll back to the previous value
    onError: (_error, { id }, context) => {
      if (context?.previousData) {
        queryClient.setQueryData(heroSectionKeys.detail(id), context.previousData);
      }
    },

    // Always refetch after error or success to sync with server
    onSettled: (_, __, { id }) => {
      queryClient.invalidateQueries({ queryKey: heroSectionKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: heroSectionKeys.lists() });
    },
  });
}
```

---

## Multi-Step Form Pattern (Wizard)

For complex forms spanning multiple steps (e.g., blog post creation with metadata, content, and preview steps):

```typescript
// src/features/blog/store/blogCreate.store.ts
import { create } from 'zustand';
import type { BlogCreateStep1Values, BlogCreateStep2Values } from '../blog.schema';

type WizardStep = 'metadata' | 'content' | 'preview';

interface BlogCreateState {
  currentStep: WizardStep;
  step1Data: BlogCreateStep1Values | null;
  step2Data: BlogCreateStep2Values | null;

  goToStep: (step: WizardStep) => void;
  saveStep1: (data: BlogCreateStep1Values) => void;
  saveStep2: (data: BlogCreateStep2Values) => void;
  reset: () => void;
}

export const useBlogCreateStore = create<BlogCreateState>((set) => ({
  currentStep: 'metadata',
  step1Data: null,
  step2Data: null,

  goToStep: (step) => set({ currentStep: step }),
  saveStep1: (data) => set({ step1Data: data, currentStep: 'content' }),
  saveStep2: (data) => set({ step2Data: data, currentStep: 'preview' }),
  reset: () => set({ currentStep: 'metadata', step1Data: null, step2Data: null }),
}));
```

> Render wizard steps with the Headless UI `Tab` group, or with conditional rendering keyed
> on `currentStep`. A step indicator can be a simple Tailwind flex row of numbered badges.

---

## Cache Invalidation Strategy

Follow this decision tree when invalidating TanStack Query cache after mutations:

| Mutation | Invalidation Target |
|---|---|
| Create item | `queryKey: entityKeys.lists()` — all list queries |
| Update item | `queryKey: entityKeys.detail(id)` + `entityKeys.lists()` |
| Delete item | `queryKey: entityKeys.lists()` |
| Bulk action (delete multiple) | `queryKey: entityKeys.all` |
| Status toggle | `queryKey: entityKeys.detail(id)` + `entityKeys.lists()` |

Always use `invalidateQueries` (not `removeQueries`) so the UI shows stale data with
a loading indicator rather than an empty flash.

---

## Prefetching for Navigation

Prefetch list data when hovering a navigation link to make page transitions feel instant:

```typescript
// In AppSidebar or a NavLink wrapper component
import { useQueryClient } from '@tanstack/react-query';
import { heroSectionKeys } from '../../features/heroSection/hooks/useHeroSections';
import { heroSectionApi } from '../../features/heroSection/api/heroSectionApi';

function useHeroSectionPrefetch() {
  const queryClient = useQueryClient();

  return () =>
    queryClient.prefetchQuery({
      queryKey: heroSectionKeys.list({ page: 0, size: 10 }),
      queryFn: () => heroSectionApi.list({ page: 0, size: 10 }),
      staleTime: 30_000,
    });
}

// Usage: <NavLink to="/hero-section" onMouseEnter={prefetch}>…</NavLink>
```

---

## Error State Handling in Pages

```tsx
// Pattern for handling query errors in page components (Tailwind alert)
import { ExclamationTriangleIcon } from '@heroicons/react/24/outline';
import { Button } from '../../../shared/components/Button';

export function HeroSectionListPage() {
  const { data, isLoading, isError, error, refetch } = useHeroSections(filters);

  if (isError) {
    return (
      <div className="mt-6 flex items-start gap-3 rounded border border-danger/40 bg-danger/5 p-4">
        <ExclamationTriangleIcon className="mt-0.5 h-5 w-5 shrink-0 text-danger" />
        <div className="flex-1">
          <p className="text-sm text-danger">
            {error instanceof Error ? error.message : 'Failed to load data. Please try again.'}
          </p>
        </div>
        <Button variant="outline" size="sm" onClick={() => refetch()}>
          Retry
        </Button>
      </div>
    );
  }

  // Normal render...
}
```

---

## Scroll Position Restoration

React Router v7 handles scroll restoration automatically when navigating between routes.
For paginated lists, use TanStack Query's `placeholderData` to keep showing previous data
while new data loads (avoids an empty flash on page change):

```typescript
export function useHeroSections(params: HeroSectionListParams) {
  return useQuery({
    queryKey: heroSectionKeys.list(params),
    queryFn: () => heroSectionApi.list(params),
    placeholderData: (previousData) => previousData,
  });
}
```
