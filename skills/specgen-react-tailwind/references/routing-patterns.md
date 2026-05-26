# Routing Patterns — React Router v7 + Feature-Based Architecture (Tailwind)

This reference describes the React Router v7 conventions, the feature-based directory
structure, and the Tailwind/Headless UI layout shell used in the spec. Include this content
in Sections 4, 7, 11, 12, and 13 of the generated specification.

---

## Feature-Based Directory Structure

Each PRD module maps to a `src/features/<module>/` folder. This co-locates everything
related to a feature (types, API, hooks, components, pages) so the coding agent can
implement one feature folder at a time.

```
src/
├── main.tsx                          # App entry — providers, ReactDOM.render
├── App.tsx                           # Root component — router, dark-mode sync, notifications
├── index.css                         # Tailwind directives + design-token CSS variables
├── router/
│   └── index.tsx                     # Route tree definition
├── lib/
│   ├── api/
│   │   ├── axiosInstance.ts          # Axios instance with interceptors
│   │   └── authBridge.ts             # [If Auth = Keycloak] Auth bridge for interceptors
│   ├── auth/
│   │   ├── oidcConfig.ts             # [If Auth = Keycloak] oidc-client-ts config
│   │   ├── keycloakRoles.ts          # [If Auth = Keycloak] Role extraction utilities
│   │   ├── useAuthUser.ts            # [If Auth = Keycloak] Auth hook wrapper
│   │   └── roles.ts                  # Role constants
│   ├── utils/
│   │   ├── cn.ts                     # twMerge(clsx(...)) helper
│   │   └── zodErrors.ts              # Zod error extraction utilities
│   └── queryClient.ts                # TanStack Query QueryClient instance
├── store/
│   ├── auth.store.ts                 # [If Auth = Local] Zustand auth store
│   ├── notification.store.ts         # Zustand toast queue
│   └── ui.store.ts                   # Zustand UI store (sidebar, theme mode)
├── shared/
│   ├── layouts/
│   │   ├── DashboardLayout.tsx       # Authenticated layout (sidebar + topbar + outlet)
│   │   ├── PublicLayout.tsx          # Public layout (header/footer + outlet)
│   │   ├── AuthLayout.tsx            # Minimal centered layout for login/callback
│   │   └── Footer.tsx                # Footer with version string
│   ├── components/
│   │   ├── ProtectedRoute.tsx        # Auth + role guard wrapper
│   │   ├── Button.tsx                # cva button
│   │   ├── Card.tsx                  # Surface container
│   │   ├── Spinner.tsx               # Inline spinner
│   │   ├── AppTopBar.tsx             # Sticky topbar (toggle, user menu, theme toggle)
│   │   ├── AppSidebar.tsx            # Role-filtered navigation sidebar
│   │   ├── navigationConfig.ts       # Navigation item definitions
│   │   ├── PageHeader.tsx            # Page title + breadcrumbs + action buttons
│   │   ├── DataTable.tsx             # Generic TanStack Table wrapper (sorting/selection)
│   │   ├── Pagination.tsx            # Page navigation control
│   │   ├── ConfirmDialog.tsx         # Headless UI Dialog confirm
│   │   ├── StatusBadge.tsx           # cva status badge
│   │   ├── Modal.tsx                 # Headless UI Dialog wrapper for forms
│   │   ├── ImageUpload.tsx           # Drag-and-drop image upload with preview
│   │   ├── LoadingOverlay.tsx        # Full-page spinner overlay
│   │   ├── EmptyState.tsx            # Empty list placeholder with icon + message
│   │   ├── ErrorBoundary.tsx         # React ErrorBoundary with Tailwind fallback UI
│   │   ├── NotificationProvider.tsx  # Headless UI Transition toast stack
│   │   └── SearchInput.tsx           # Debounced search text field
│   ├── hooks/
│   │   ├── useDebounce.ts            # Generic debounce hook
│   │   ├── useNotification.ts        # Toast notification Zustand hook
│   │   ├── useColorMode.ts           # Dark-mode toggle hook
│   │   └── useConfirm.ts             # Confirm dialog imperative hook
│   └── form/
│       ├── TextFieldController.tsx   # RHF Controller wrapping a Tailwind <input>
│       ├── TextAreaController.tsx    # RHF Controller wrapping a Tailwind <textarea>
│       ├── SelectController.tsx      # RHF Controller wrapping Headless UI Listbox
│       ├── DatePickerController.tsx  # RHF Controller wrapping react-day-picker
│       ├── SwitchController.tsx      # RHF Controller wrapping Headless UI Switch
│       ├── ComboboxController.tsx    # RHF Controller wrapping Headless UI Combobox
│       └── RichTextController.tsx    # [If RichText = yes] RHF Controller for Tiptap
├── features/
│   ├── auth/                         # [If Auth != none] Auth feature
│   │   ├── pages/
│   │   │   ├── LoginPage.tsx         # [If Auth = Local] Login form page
│   │   │   └── AuthCallbackPage.tsx  # [If Auth = Keycloak] OIDC callback handler
│   │   └── index.ts
│   ├── {{module1}}/                  # PRD module 1 (e.g., heroSection)
│   │   ├── api/
│   │   │   └── {{module1}}Api.ts     # Axios API functions
│   │   ├── components/
│   │   │   ├── {{Module1}}Form.tsx   # Create/edit form
│   │   │   └── {{Module1}}Card.tsx   # Card component (if applicable)
│   │   ├── hooks/
│   │   │   ├── use{{Module1}}s.ts    # useQuery for list
│   │   │   ├── use{{Module1}}.ts     # useQuery for single item
│   │   │   └── use{{Module1}}Mutations.ts # useMutation for CUD operations
│   │   ├── pages/
│   │   │   ├── {{Module1}}ListPage.tsx   # List page component (lazy-loaded)
│   │   │   └── {{Module1}}FormPage.tsx   # Create/edit page (lazy-loaded)
│   │   ├── store/
│   │   │   └── {{module1}}.store.ts  # [If needed] Feature-level Zustand store
│   │   ├── {{module1}}.types.ts      # TypeScript interfaces
│   │   ├── {{module1}}.schema.ts     # Zod validation schemas
│   │   └── index.ts                  # Public API re-exports
│   └── {{module2}}/
│       └── (same structure)
└── types/
    └── global.d.ts                   # Global type augmentations (e.g., import.meta.env)
```

---

## Route Tree Definition

```tsx
// src/router/index.tsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter, Navigate } from 'react-router-dom';
import { DashboardLayout } from '../shared/layouts/DashboardLayout';
import { PublicLayout } from '../shared/layouts/PublicLayout';
import { AuthLayout } from '../shared/layouts/AuthLayout';
import { ProtectedRoute } from '../shared/components/ProtectedRoute';
import { LoadingOverlay } from '../shared/components/LoadingOverlay';
import { Roles } from '../lib/auth/roles';

// Lazy-load all page components for code splitting
// [If Auth = Keycloak]
const AuthCallbackPage = lazy(() =>
  import('../features/auth/pages/AuthCallbackPage').then((m) => ({ default: m.AuthCallbackPage })),
);

// [If Auth = Local]
const LoginPage = lazy(() =>
  import('../features/auth/pages/LoginPage').then((m) => ({ default: m.LoginPage })),
);

const DashboardPage = lazy(() =>
  import('../features/dashboard/pages/DashboardPage').then((m) => ({ default: m.DashboardPage })),
);

// Hero Section (example — replace with actual modules from PRD)
const HeroSectionListPage = lazy(() =>
  import('../features/heroSection/pages/HeroSectionListPage').then((m) => ({
    default: m.HeroSectionListPage,
  })),
);
const HeroSectionFormPage = lazy(() =>
  import('../features/heroSection/pages/HeroSectionFormPage').then((m) => ({
    default: m.HeroSectionFormPage,
  })),
);

function SuspenseWrapper({ children }: { children: React.ReactNode }) {
  return <Suspense fallback={<LoadingOverlay />}>{children}</Suspense>;
}

export const router = createBrowserRouter([
  // ── Auth routes (public) ────────────────────────────────────────────
  {
    element: <AuthLayout />,
    children: [
      // [If Auth = Keycloak]
      {
        path: '/auth/callback',
        element: (
          <SuspenseWrapper>
            <AuthCallbackPage />
          </SuspenseWrapper>
        ),
      },
      // [If Auth = Local]
      {
        path: '/login',
        element: (
          <SuspenseWrapper>
            <LoginPage />
          </SuspenseWrapper>
        ),
      },
    ],
  },

  // ── Protected / dashboard routes ────────────────────────────────────
  {
    element: (
      <ProtectedRoute>
        <DashboardLayout />
      </ProtectedRoute>
    ),
    children: [
      { index: true, element: <Navigate to="/dashboard" replace /> },
      {
        path: '/dashboard',
        element: (
          <SuspenseWrapper>
            <DashboardPage />
          </SuspenseWrapper>
        ),
      },

      // Hero Section — accessible by ADMIN and EDITOR (module-based path, NOT role-prefixed)
      {
        path: '/hero-section',
        element: (
          <ProtectedRoute requiredRoles={[Roles.ADMIN, Roles.EDITOR]}>
            <SuspenseWrapper>
              <HeroSectionListPage />
            </SuspenseWrapper>
          </ProtectedRoute>
        ),
      },
      {
        path: '/hero-section/create',
        element: (
          <ProtectedRoute requiredRoles={[Roles.ADMIN, Roles.EDITOR]}>
            <SuspenseWrapper>
              <HeroSectionFormPage />
            </SuspenseWrapper>
          </ProtectedRoute>
        ),
      },
      {
        path: '/hero-section/:id/edit',
        element: (
          <ProtectedRoute requiredRoles={[Roles.ADMIN, Roles.EDITOR]}>
            <SuspenseWrapper>
              <HeroSectionFormPage />
            </SuspenseWrapper>
          </ProtectedRoute>
        ),
      },

      // Add additional module routes following the same pattern...
    ],
  },

  // ── Error routes ────────────────────────────────────────────────────
  {
    path: '/unauthorized',
    element: <div className="p-8 text-slate-700 dark:text-slate-200">You do not have permission to access this page.</div>,
  },
  {
    path: '*',
    element: <div className="p-8 text-slate-700 dark:text-slate-200">404 — Page not found.</div>,
  },
]);
```

---

## DashboardLayout

```tsx
// src/shared/layouts/DashboardLayout.tsx
import { Outlet } from 'react-router-dom';
import { AppTopBar } from '../components/AppTopBar';
import { AppSidebar } from '../components/AppSidebar';
import { Footer } from './Footer';
import { useUiStore } from '../../store/ui.store';
import { cn } from '../../lib/utils/cn';

export function DashboardLayout() {
  const sidebarOpen = useUiStore((s) => s.sidebarOpen);

  return (
    <div className="min-h-screen bg-canvas">
      <AppTopBar />
      <AppSidebar />
      <div
        className={cn(
          'flex min-h-screen flex-col pt-16 transition-[padding] duration-200',
          sidebarOpen ? 'lg:pl-60' : 'lg:pl-0',
        )}
      >
        <main className="flex-1 p-4 sm:p-6">
          <Outlet />
        </main>
        <Footer />
      </div>
    </div>
  );
}
```

```tsx
// src/shared/layouts/Footer.tsx
export function Footer() {
  const version = import.meta.env.VITE_APP_VERSION;
  return (
    <footer className="border-t border-slate-200 px-6 py-3 text-center text-xs text-slate-500 dark:border-slate-700">
      © {new Date().getFullYear()} {import.meta.env.VITE_APP_TITLE} — v{version}
    </footer>
  );
}
```

---

## AppTopBar (Headless UI Menu user dropdown + theme toggle)

```tsx
// src/shared/components/AppTopBar.tsx
import { Menu, MenuButton, MenuItem, MenuItems } from '@headlessui/react';
import { Bars3Icon, UserCircleIcon, ArrowRightStartOnRectangleIcon } from '@heroicons/react/24/outline';
import { useUiStore } from '../../store/ui.store';
import { useAuthUser } from '../../lib/auth/useAuthUser';
import { ThemeToggle } from './ThemeToggle';
import { cn } from '../../lib/utils/cn';

export function AppTopBar() {
  const toggleSidebar = useUiStore((s) => s.toggleSidebar);
  const { name, logout } = useAuthUser();

  return (
    <header className="fixed inset-x-0 top-0 z-30 flex h-16 items-center justify-between border-b border-slate-200 bg-surface px-4 dark:border-slate-700">
      <div className="flex items-center gap-3">
        <button
          type="button"
          onClick={toggleSidebar}
          aria-label="Toggle navigation"
          className="rounded p-2 text-slate-600 hover:bg-slate-100 dark:text-slate-300 dark:hover:bg-slate-700"
        >
          <Bars3Icon className="h-5 w-5" />
        </button>
        <span className="text-lg font-semibold text-slate-900 dark:text-slate-100">
          {import.meta.env.VITE_APP_TITLE}
        </span>
      </div>

      <div className="flex items-center gap-2">
        <ThemeToggle />
        <Menu>
          <MenuButton className="flex items-center gap-2 rounded p-1.5 text-slate-700 hover:bg-slate-100 dark:text-slate-200 dark:hover:bg-slate-700">
            <UserCircleIcon className="h-6 w-6" />
            <span className="hidden text-sm sm:inline">{name}</span>
          </MenuButton>
          <MenuItems
            anchor="bottom end"
            className="z-50 mt-1 w-48 rounded border border-slate-200 bg-surface py-1 text-sm shadow-lg focus:outline-none dark:border-slate-700"
          >
            <MenuItem>
              <button
                onClick={logout}
                className={cn(
                  'flex w-full items-center gap-2 px-3 py-2 text-left text-slate-700 dark:text-slate-200',
                  'data-[focus]:bg-slate-100 dark:data-[focus]:bg-slate-700',
                )}
              >
                <ArrowRightStartOnRectangleIcon className="h-4 w-4" />
                Sign out
              </button>
            </MenuItem>
          </MenuItems>
        </Menu>
      </div>
    </header>
  );
}
```

---

## Sidebar Navigation

```typescript
// src/shared/components/navigationConfig.ts
import { HomeIcon, PhotoIcon, UsersIcon } from '@heroicons/react/24/outline';
import type { ComponentType, SVGProps } from 'react';
import { Roles } from '../../lib/auth/roles';

export interface NavItem {
  label: string;
  path: string;
  icon: ComponentType<SVGProps<SVGSVGElement>>;
  /** If provided, user must have at least one of these roles to see this item */
  requiredRoles?: string[];
}

/**
 * All sidebar navigation items.
 * Items with requiredRoles are hidden for users who lack those roles.
 * Replace with actual modules from the PRD.md (module-based paths only).
 */
export const navigationItems: NavItem[] = [
  { label: 'Dashboard', path: '/dashboard', icon: HomeIcon },
  { label: 'Hero Section', path: '/hero-section', icon: PhotoIcon, requiredRoles: [Roles.ADMIN, Roles.EDITOR] },
  { label: 'Users', path: '/users', icon: UsersIcon, requiredRoles: [Roles.ADMIN] },
  // Add additional nav items matching the mockup sidebar files...
];
```

```tsx
// src/shared/components/AppSidebar.tsx
import { NavLink } from 'react-router-dom';
import { useAuthUser } from '../../lib/auth/useAuthUser';
import { useUiStore } from '../../store/ui.store';
import { navigationItems } from './navigationConfig';
import { cn } from '../../lib/utils/cn';

export function AppSidebar() {
  const { hasAnyRole } = useAuthUser();
  const sidebarOpen = useUiStore((s) => s.sidebarOpen);

  const visibleItems = navigationItems.filter(
    (item) => !item.requiredRoles || hasAnyRole(item.requiredRoles),
  );

  return (
    <aside
      className={cn(
        'fixed inset-y-0 left-0 z-20 w-60 border-r border-slate-200 bg-surface pt-16 transition-transform duration-200 dark:border-slate-700',
        sidebarOpen ? 'translate-x-0' : '-translate-x-full',
      )}
    >
      <nav className="flex flex-col gap-1 p-3">
        {visibleItems.map((item) => {
          const Icon = item.icon;
          return (
            <NavLink
              key={item.path}
              to={item.path}
              className={({ isActive }) =>
                cn(
                  'flex items-center gap-3 rounded px-3 py-2 text-sm font-medium transition-colors',
                  isActive
                    ? 'bg-primary/10 text-primary'
                    : 'text-slate-700 hover:bg-slate-100 dark:text-slate-200 dark:hover:bg-slate-700',
                )
              }
            >
              <Icon className="h-5 w-5 shrink-0" />
              {item.label}
            </NavLink>
          );
        })}
      </nav>
    </aside>
  );
}
```

---

## PageHeader (breadcrumbs)

```tsx
// src/shared/components/PageHeader.tsx
import { Link } from 'react-router-dom';
import { ChevronRightIcon } from '@heroicons/react/20/solid';

export interface BreadcrumbItem {
  label: string;
  path?: string;
}

interface PageHeaderProps {
  title: string;
  breadcrumbs?: BreadcrumbItem[];
  actions?: React.ReactNode;
}

export function PageHeader({ title, breadcrumbs, actions }: PageHeaderProps) {
  return (
    <div className="mb-6">
      {breadcrumbs && (
        <nav className="mb-1 flex items-center gap-1 text-xs text-slate-500" aria-label="Breadcrumb">
          {breadcrumbs.map((crumb, index) => (
            <span key={index} className="flex items-center gap-1">
              {index > 0 && <ChevronRightIcon className="h-3.5 w-3.5 text-slate-400" />}
              {crumb.path ? (
                <Link to={crumb.path} className="hover:text-primary hover:underline">
                  {crumb.label}
                </Link>
              ) : (
                <span className="text-slate-700 dark:text-slate-300">{crumb.label}</span>
              )}
            </span>
          ))}
        </nav>
      )}
      <div className="flex items-center justify-between gap-3">
        <h1 className="text-xl font-semibold text-slate-900 dark:text-slate-100">{title}</h1>
        {actions && <div className="flex items-center gap-2">{actions}</div>}
      </div>
    </div>
  );
}
```

---

## Pagination

```tsx
// src/shared/components/Pagination.tsx
import { ChevronLeftIcon, ChevronRightIcon } from '@heroicons/react/20/solid';
import { Button } from './Button';

interface PaginationProps {
  /** Zero-based current page index */
  page: number;
  size: number;
  totalElements: number;
  onPageChange: (page: number) => void;
  onSizeChange?: (size: number) => void;
}

export function Pagination({ page, size, totalElements, onPageChange, onSizeChange }: PaginationProps) {
  const totalPages = Math.max(1, Math.ceil(totalElements / size));
  const from = totalElements === 0 ? 0 : page * size + 1;
  const to = Math.min((page + 1) * size, totalElements);

  return (
    <div className="flex flex-wrap items-center justify-between gap-3 border-t border-slate-200 px-4 py-3 text-sm dark:border-slate-700">
      <p className="text-slate-600 dark:text-slate-300">
        Showing <span className="font-medium">{from}</span>–<span className="font-medium">{to}</span> of{' '}
        <span className="font-medium">{totalElements}</span>
      </p>
      <div className="flex items-center gap-3">
        {onSizeChange && (
          <select
            value={size}
            onChange={(e) => onSizeChange(Number(e.target.value))}
            className="rounded border border-slate-300 bg-surface px-2 py-1 text-sm focus:border-primary focus:ring-1 focus:ring-primary dark:border-slate-600"
            aria-label="Rows per page"
          >
            {[10, 25, 50].map((n) => (
              <option key={n} value={n}>
                {n} / page
              </option>
            ))}
          </select>
        )}
        <div className="flex items-center gap-1">
          <Button variant="outline" size="icon" disabled={page <= 0} onClick={() => onPageChange(page - 1)} aria-label="Previous page">
            <ChevronLeftIcon className="h-4 w-4" />
          </Button>
          <span className="px-2 text-slate-600 dark:text-slate-300">
            {page + 1} / {totalPages}
          </span>
          <Button variant="outline" size="icon" disabled={page + 1 >= totalPages} onClick={() => onPageChange(page + 1)} aria-label="Next page">
            <ChevronRightIcon className="h-4 w-4" />
          </Button>
        </div>
      </div>
    </div>
  );
}
```

---

## DataTable (TanStack Table wrapper) [Include when DataGrid = yes]

```tsx
// src/shared/components/DataTable.tsx
import {
  flexRender,
  getCoreRowModel,
  getSortedRowModel,
  useReactTable,
  type ColumnDef,
  type SortingState,
} from '@tanstack/react-table';
import { useState } from 'react';
import { ChevronUpDownIcon } from '@heroicons/react/20/solid';
import { cn } from '../../lib/utils/cn';

interface DataTableProps<TData> {
  columns: ColumnDef<TData, unknown>[];
  data: TData[];
  /** Optional empty-state message */
  emptyMessage?: string;
}

export function DataTable<TData>({ columns, data, emptyMessage = 'No records found.' }: DataTableProps<TData>) {
  const [sorting, setSorting] = useState<SortingState>([]);

  const table = useReactTable({
    data,
    columns,
    state: { sorting },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  return (
    <div className="overflow-x-auto">
      <table className="min-w-full divide-y divide-slate-200 dark:divide-slate-700">
        <thead className="bg-canvas">
          {table.getHeaderGroups().map((headerGroup) => (
            <tr key={headerGroup.id}>
              {headerGroup.headers.map((header) => {
                const canSort = header.column.getCanSort();
                return (
                  <th
                    key={header.id}
                    className="px-4 py-3 text-left text-xs font-semibold uppercase tracking-wide text-slate-600 dark:text-slate-300"
                  >
                    <button
                      type="button"
                      disabled={!canSort}
                      onClick={header.column.getToggleSortingHandler()}
                      className={cn('flex items-center gap-1', canSort && 'cursor-pointer select-none')}
                    >
                      {flexRender(header.column.columnDef.header, header.getContext())}
                      {canSort && <ChevronUpDownIcon className="h-3.5 w-3.5 text-slate-400" />}
                    </button>
                  </th>
                );
              })}
            </tr>
          ))}
        </thead>
        <tbody className="divide-y divide-slate-100 dark:divide-slate-800">
          {table.getRowModel().rows.length === 0 ? (
            <tr>
              <td colSpan={columns.length} className="px-4 py-8 text-center text-sm text-slate-500">
                {emptyMessage}
              </td>
            </tr>
          ) : (
            table.getRowModel().rows.map((row) => (
              <tr key={row.id} className="hover:bg-canvas/60">
                {row.getVisibleCells().map((cell) => (
                  <td key={cell.id} className="px-4 py-3 text-sm text-slate-700 dark:text-slate-200">
                    {flexRender(cell.column.columnDef.cell, cell.getContext())}
                  </td>
                ))}
              </tr>
            ))
          )}
        </tbody>
      </table>
    </div>
  );
}
```

---

## LoadingOverlay & EmptyState

```tsx
// src/shared/components/LoadingOverlay.tsx
import { Spinner } from './Spinner';

export function LoadingOverlay({ message }: { message?: string }) {
  return (
    <div className="flex min-h-[40vh] flex-col items-center justify-center gap-3 text-slate-500">
      <Spinner className="h-8 w-8 text-primary" />
      {message && <p className="text-sm">{message}</p>}
    </div>
  );
}
```

```tsx
// src/shared/components/EmptyState.tsx
import { InboxIcon } from '@heroicons/react/24/outline';

export function EmptyState({ message, action }: { message: string; action?: React.ReactNode }) {
  return (
    <div className="flex flex-col items-center justify-center gap-3 py-12 text-center text-slate-500">
      <InboxIcon className="h-10 w-10 text-slate-300" />
      <p className="text-sm">{message}</p>
      {action}
    </div>
  );
}
```
