# Routing Patterns — React Router v7 + Feature-Based Architecture

This reference describes the React Router v7 conventions and feature-based directory
structure used in the spec. Include this content in Sections 4 and 7 of the generated
specification.

---

## Feature-Based Directory Structure

Each PRD module maps to a `src/features/<module>/` folder. This co-locates everything
related to a feature (types, API, hooks, components, pages) so the coding agent can
implement one feature folder at a time.

```
src/
├── main.tsx                          # App entry — providers, ReactDOM.render
├── App.tsx                           # Root component — router, theme, notifications
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
│   └── queryClient.ts                # TanStack Query QueryClient instance
├── store/
│   ├── auth.store.ts                 # [If Auth = Local] Zustand auth store
│   └── ui.store.ts                   # Zustand UI store (sidebar, theme mode)
├── shared/
│   ├── layouts/
│   │   ├── DashboardLayout.tsx       # Authenticated layout (sidebar + topbar + outlet)
│   │   ├── PublicLayout.tsx          # Public layout (header/footer + outlet)
│   │   └── AuthLayout.tsx            # Minimal centered layout for login/callback
│   ├── components/
│   │   ├── ProtectedRoute.tsx        # Auth + role guard wrapper
│   │   ├── PageHeader.tsx            # Page title + breadcrumbs + action buttons
│   │   ├── DataTable.tsx             # Generic MUI Table with sorting/pagination
│   │   ├── ConfirmDialog.tsx         # Generic confirmation Modal
│   │   ├── StatusChip.tsx            # Colored MUI Chip for status badges
│   │   ├── FormDialog.tsx            # Full-screen or modal form dialog wrapper
│   │   ├── ImageUpload.tsx           # Drag-and-drop image upload with preview
│   │   ├── LoadingOverlay.tsx        # Full-page CircularProgress overlay
│   │   ├── EmptyState.tsx            # Empty list placeholder with icon + message
│   │   ├── ErrorBoundary.tsx         # React ErrorBoundary with fallback UI
│   │   └── SearchInput.tsx           # Debounced search text field
│   ├── hooks/
│   │   ├── useDebounce.ts            # Generic debounce hook
│   │   ├── useNotification.ts        # Toast notification Zustand hook
│   │   └── useConfirm.ts             # Confirm dialog imperative hook
│   └── form/
│       ├── TextFieldController.tsx   # RHF Controller wrapping MUI TextField
│       ├── SelectController.tsx      # RHF Controller wrapping MUI Select
│       ├── DatePickerController.tsx  # RHF Controller wrapping MUI X DatePicker
│       ├── SwitchController.tsx      # RHF Controller wrapping MUI Switch
│       ├── AutocompleteController.tsx# RHF Controller wrapping MUI Autocomplete
│       └── RichTextController.tsx    # [If RichText = yes] RHF Controller for Quill
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
│   │   │   ├── {{Module1}}List.tsx   # List table/grid component
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
├── theme/
│   ├── index.ts                      # MUI createTheme() with design tokens
│   ├── palette.ts                    # Color palette extracted from mockup
│   └── typography.ts                 # Font config extracted from mockup
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
  import('../features/auth/pages/AuthCallbackPage').then((m) => ({
    default: m.AuthCallbackPage,
  })),
);

// [If Auth = Local]
const LoginPage = lazy(() =>
  import('../features/auth/pages/LoginPage').then((m) => ({ default: m.LoginPage })),
);

// Dashboard / home page
const DashboardPage = lazy(() =>
  import('../features/dashboard/pages/DashboardPage').then((m) => ({
    default: m.DashboardPage,
  })),
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

// Fallback wrapper for lazy-loaded components
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
      {
        index: true,
        element: <Navigate to="/dashboard" replace />,
      },
      {
        path: '/dashboard',
        element: (
          <SuspenseWrapper>
            <DashboardPage />
          </SuspenseWrapper>
        ),
      },

      // Hero Section — accessible by ADMIN and EDITOR
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
    element: <div>You do not have permission to access this page.</div>,
  },
  {
    path: '*',
    element: <div>404 — Page not found.</div>,
  },
]);
```

---

## DashboardLayout

```tsx
// src/shared/layouts/DashboardLayout.tsx
import { useState } from 'react';
import { Outlet } from 'react-router-dom';
import { Box, Toolbar } from '@mui/material';
import { AppTopBar } from '../components/AppTopBar';
import { AppSidebar } from '../components/AppSidebar';

const DRAWER_WIDTH = 240;

export function DashboardLayout() {
  const [sidebarOpen, setSidebarOpen] = useState(true);

  return (
    <Box sx={{ display: 'flex', minHeight: '100vh' }}>
      <AppTopBar
        drawerWidth={DRAWER_WIDTH}
        sidebarOpen={sidebarOpen}
        onToggleSidebar={() => setSidebarOpen((prev) => !prev)}
      />
      <AppSidebar
        drawerWidth={DRAWER_WIDTH}
        open={sidebarOpen}
      />
      <Box
        component="main"
        sx={{
          flexGrow: 1,
          p: 3,
          ml: sidebarOpen ? `${DRAWER_WIDTH}px` : 0,
          transition: (theme) =>
            theme.transitions.create('margin', {
              easing: theme.transitions.easing.sharp,
              duration: theme.transitions.duration.leavingScreen,
            }),
        }}
      >
        <Toolbar /> {/* Spacer for fixed AppBar */}
        <Outlet />
      </Box>
    </Box>
  );
}
```

---

## Sidebar Navigation

```typescript
// src/shared/components/navigationConfig.ts
import DashboardIcon from '@mui/icons-material/Dashboard';
import ImageIcon from '@mui/icons-material/Image';
import PeopleIcon from '@mui/icons-material/People';
import type { SvgIconComponent } from '@mui/icons-material';
import { Roles } from '../../lib/auth/roles';

export interface NavItem {
  label: string;
  path: string;
  icon: SvgIconComponent;
  /** If provided, user must have at least one of these roles to see this item */
  requiredRoles?: string[];
  children?: NavItem[];
}

/**
 * All sidebar navigation items.
 * Items with requiredRoles are hidden for users who lack those roles.
 * Replace with actual modules from the PRD.md.
 */
export const navigationItems: NavItem[] = [
  {
    label: 'Dashboard',
    path: '/dashboard',
    icon: DashboardIcon,
    // No requiredRoles = accessible to all authenticated users
  },
  {
    label: 'Hero Section',
    path: '/hero-section',
    icon: ImageIcon,
    requiredRoles: [Roles.ADMIN, Roles.EDITOR],
  },
  {
    label: 'Users',
    path: '/users',
    icon: PeopleIcon,
    requiredRoles: [Roles.ADMIN],
  },
  // Add additional nav items matching the mockup sidebar files...
];
```

```tsx
// src/shared/components/AppSidebar.tsx
import {
  Drawer,
  List,
  ListItemButton,
  ListItemIcon,
  ListItemText,
  Toolbar,
} from '@mui/material';
import { useNavigate, useLocation } from 'react-router-dom';
import { useAuthUser } from '../../lib/auth/useAuthUser';
import { navigationItems } from './navigationConfig';

interface AppSidebarProps {
  drawerWidth: number;
  open: boolean;
}

export function AppSidebar({ drawerWidth, open }: AppSidebarProps) {
  const { hasAnyRole } = useAuthUser();
  const navigate = useNavigate();
  const location = useLocation();

  const visibleItems = navigationItems.filter(
    (item) => !item.requiredRoles || hasAnyRole(item.requiredRoles),
  );

  return (
    <Drawer
      variant="persistent"
      anchor="left"
      open={open}
      sx={{
        width: drawerWidth,
        flexShrink: 0,
        '& .MuiDrawer-paper': { width: drawerWidth, boxSizing: 'border-box' },
      }}
    >
      <Toolbar />
      <List>
        {visibleItems.map((item) => {
          const Icon = item.icon;
          const isActive = location.pathname.startsWith(item.path);
          return (
            <ListItemButton
              key={item.path}
              selected={isActive}
              onClick={() => navigate(item.path)}
            >
              <ListItemIcon>
                <Icon color={isActive ? 'primary' : 'inherit'} />
              </ListItemIcon>
              <ListItemText primary={item.label} />
            </ListItemButton>
          );
        })}
      </List>
    </Drawer>
  );
}
```

---

## Breadcrumb Pattern

```tsx
// src/shared/components/PageHeader.tsx
import { Box, Breadcrumbs, Link, Typography } from '@mui/material';
import { useNavigate } from 'react-router-dom';

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
  const navigate = useNavigate();

  return (
    <Box sx={{ mb: 3 }}>
      {breadcrumbs && (
        <Breadcrumbs sx={{ mb: 1 }}>
          {breadcrumbs.map((crumb, index) =>
            crumb.path ? (
              <Link
                key={index}
                component="button"
                variant="body2"
                onClick={() => navigate(crumb.path!)}
                underline="hover"
                color="inherit"
              >
                {crumb.label}
              </Link>
            ) : (
              <Typography key={index} variant="body2" color="text.primary">
                {crumb.label}
              </Typography>
            ),
          )}
        </Breadcrumbs>
      )}
      <Box sx={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
        <Typography variant="h5" fontWeight={600}>
          {title}
        </Typography>
        {actions && <Box sx={{ display: 'flex', gap: 1 }}>{actions}</Box>}
      </Box>
    </Box>
  );
}
```
