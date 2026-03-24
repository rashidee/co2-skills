# State Management Patterns — Zustand Feature Stores & TanStack Query

This reference describes when and how to use feature-level Zustand stores alongside
TanStack Query. Include this content in Section 9 of the generated specification and
in any module SPEC.md that requires client-side state beyond server cache.

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
import type { HeroSectionListParams, HeroSectionStatus } from '../heroSection.types';

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

const DEFAULT_FILTERS: HeroSectionListParams = {
  page: 0,
  size: 10,
  status: '',
};

export const useHeroSectionStore = create<HeroSectionState>((set) => ({
  filters: { ...DEFAULT_FILTERS },
  selectedIds: new Set(),
  isDetailPanelOpen: false,
  selectedId: null,

  setFilter: (key, value) =>
    set((state) => ({
      filters: { ...state.filters, [key]: value, page: 0 },
    })),

  resetFilters: () => set({ filters: { ...DEFAULT_FILTERS } }),

  setPage: (page) =>
    set((state) => ({ filters: { ...state.filters, page } })),

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

export function HeroSectionListPage() {
  // Filter state lives in Zustand — changes trigger React re-render
  const { filters, setFilter, setPage } = useHeroSectionStore();

  // TanStack Query uses filters as the query key — auto-refetches when filters change
  const { data, isLoading } = useHeroSections(filters);

  return (
    <>
      <FilterBar
        status={filters.status}
        onStatusChange={(value) => setFilter('status', value)}
      />
      <DataTable
        rows={data?.content ?? []}
        loading={isLoading}
      />
      <TablePagination
        count={data?.totalElements ?? 0}
        page={filters.page ?? 0}
        onPageChange={(_, page) => setPage(page)}
      />
    </>
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
      // Cancel any in-flight refetches to avoid overwriting optimistic update
      await queryClient.cancelQueries({ queryKey: heroSectionKeys.all });

      // Snapshot previous data for rollback
      const previousData = queryClient.getQueryData<HeroSection>(
        heroSectionKeys.detail(id),
      );

      // Optimistically update the cache
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
// In AppSidebar or NavigationLink component
import { useQueryClient } from '@tanstack/react-query';
import { heroSectionKeys } from '../../features/heroSection/hooks/useHeroSections';
import { heroSectionApi } from '../../features/heroSection/api/heroSectionApi';

function NavItemWithPrefetch({ item }: { item: NavItem }) {
  const queryClient = useQueryClient();

  const handleMouseEnter = () => {
    // Prefetch the default page when the user hovers the nav item
    queryClient.prefetchQuery({
      queryKey: heroSectionKeys.list({ page: 0, size: 10 }),
      queryFn: () => heroSectionApi.list({ page: 0, size: 10 }),
      staleTime: 30_000,
    });
  };

  return (
    <ListItemButton onMouseEnter={handleMouseEnter} onClick={() => navigate(item.path)}>
      ...
    </ListItemButton>
  );
}
```

---

## Error State Handling in Pages

```tsx
// Pattern for handling query errors in page components
import { Alert, Button, Box } from '@mui/material';

export function HeroSectionListPage() {
  const { data, isLoading, isError, error, refetch } = useHeroSections(filters);

  if (isError) {
    return (
      <Box sx={{ mt: 4 }}>
        <Alert
          severity="error"
          action={
            <Button color="inherit" size="small" onClick={() => refetch()}>
              Retry
            </Button>
          }
        >
          {error instanceof Error ? error.message : 'Failed to load data. Please try again.'}
        </Alert>
      </Box>
    );
  }

  // Normal render...
}
```

---

## Scroll Position Restoration

React Router v7 handles scroll restoration automatically when navigating between routes.
For pages with virtual scroll or infinite pagination, use TanStack Query's `keepPreviousData`
equivalent:

```typescript
export function useHeroSections(params: HeroSectionListParams) {
  return useQuery({
    queryKey: heroSectionKeys.list(params),
    queryFn: () => heroSectionApi.list(params),
    // Keep showing previous data while new data loads (avoids empty flash on page change)
    placeholderData: (previousData) => previousData,
  });
}
```
