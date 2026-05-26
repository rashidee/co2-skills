# Specification Template — React SPA with Tailwind CSS

This is the authoritative template for the generated specification. The specification is
split into **two types of files**:

1. **`SPECIFICATION.md`** (root) — Table of Contents, shared infrastructure, and all
   application-level sections. Generated once per application.
2. **`<module>/SPEC.md`** (per-module) — Self-contained module blueprint.
   Generated once per module from PRD.md.

```
spec/
├── SPECIFICATION.md
├── hero-section/
│   └── SPEC.md
├── product-service/
│   └── SPEC.md
└── ...
```

Placeholders use `{{VARIABLE}}` syntax and must be replaced with actual values gathered
from context files.

Sections marked **[CONDITIONAL]** should only be included when the corresponding
integration is selected.

**CRITICAL: Sample code requirement.** Every component in the spec MUST include a
complete, continuous code sample in a fenced code block (TypeScript, TSX, or JSON as
appropriate). The code must be self-explanatory and directly usable as a reference for
a coding agent. Do not describe components with bullet points alone — always accompany
descriptions with full code.

---

# Part A: Root SPECIFICATION.md

---

## Table of Contents

Generate a TOC with clickable Markdown anchor links to every H2 and H3 section.
Additionally, include a **Modules** section in the TOC that links to each module's
`SPEC.md` file:

```markdown
## Table of Contents

### Shared Infrastructure
- [1. Project Overview](#1-project-overview)
- [2. Package Configuration](#2-package-configuration)
- ...all shared sections...

### Modules
- [Hero Section](hero-section/SPEC.md)
- [Product and Service](product-service/SPEC.md)
- [Blog](blog/SPEC.md)
- ...(one link per module)...
```

Only include conditional sections that apply based on determined selections.

---

## Section 1: Project Overview

```
# {{APPLICATION_NAME}} — Technical Specification

## 1. Project Overview

**Application Name**: {{APPLICATION_NAME}}
**Project Slug**: {{PROJECT_SLUG}}
**App Title**: {{APP_TITLE}}
**Framework**: React SPA (Tailwind CSS)
**Description**: {{APP_DESCRIPTION}}
**Versions Covered**: v1.0.0 — v{{LATEST_VERSION}}

### Optional Components (Auto-Determined)
**Backend API Base URL**: {{API_BASE_URL}}
**Authentication**: {{AUTH_TYPE}}
**DataGrid**: {{DATAGRID}}      (TanStack Table)
**Charts**: {{CHARTS}}          (Recharts)
**DatePickers**: {{DATE_PICKERS}} (react-day-picker)
**WebSocket**: {{WEBSOCKET}}
**i18n**: {{I18N}}
**RichText**: {{RICH_TEXT}}     (Tiptap)

### Technology Stack
(Render the core stack version table from SKILL.md, plus selected optional integration versions)

### User Roles
(List each user role extracted from mockup role folders, with role constant and access scope.)

| Role | Description | Constant | Accessible Modules |
|------|-------------|----------|-------------------|
| Administrator | Full system access | `ADMIN` | All modules |
| Editor | Content management | `EDITOR` | Hero Section, Blog, ... |

### Modules
(List each module from PRD.md, with type and link to its SPEC.md.)

| Module | Type | Stories | Versions | Spec |
|--------|------|---------|----------|------|
| Hero Section | Business | 3 | 1.0.0, 1.0.4 | [SPEC](hero-section/SPEC.md) |
| Blog | Business | 5 | 1.0.0 | [SPEC](blog/SPEC.md) |

### Input Sources
- CLAUDE.md: {{path}}
- PRD.md: {{path}}
- Module Model: {{path to MODEL.md}}
- Mockup Screens: {{path to MOCKUP.html}}
```

---

## Section 2: Package Configuration

### 2.1 package.json

Generate a complete `package.json` inside a code block. Include all core and
conditional dependencies based on the determined selections.

```json
{
  "name": "{{PROJECT_SLUG}}",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router-dom": "^7.0.0",
    "@tanstack/react-query": "^5.0.0",
    "zustand": "^5.0.0",
    "react-hook-form": "^7.0.0",
    "@hookform/resolvers": "^3.0.0",
    "zod": "^3.0.0",
    "axios": "^1.0.0",
    "@headlessui/react": "^2.0.0",
    "@heroicons/react": "^2.0.0",
    "clsx": "^2.0.0",
    "tailwind-merge": "^2.0.0",
    "class-variance-authority": "^0.7.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0",
    "@tailwindcss/forms": "^0.5.0",
    "@tanstack/react-query-devtools": "^5.0.0",
    "eslint": "^9.0.0",
    "@typescript-eslint/eslint-plugin": "^8.0.0",
    "@typescript-eslint/parser": "^8.0.0"
  }
}
```

Include conditional dependencies as annotated comments in the final output:

```jsonc
// [If Auth = Keycloak or OIDC]
//   "oidc-client-ts": "^3.1.0",
//   "react-oidc-context": "^3.2.0"
// [If DataGrid = yes]
//   "@tanstack/react-table": "^8.0.0"
// [If Charts = yes]
//   "recharts": "^2.0.0"
// [If DatePickers = yes]
//   "react-day-picker": "^9.0.0",
//   "date-fns": "^4.0.0"
// [If WebSocket = yes]
//   "socket.io-client": "^4.0.0"
// [If i18n = yes]
//   "react-i18next": "^15.0.0", "i18next": "^24.0.0",
//   "i18next-browser-languagedetector": "^8.0.0", "i18next-http-backend": "^3.0.0"
// [If RichText = yes]
//   "@tiptap/react": "^2.0.0", "@tiptap/starter-kit": "^2.0.0",
//   "@tiptap/extension-link": "^2.0.0", "@tiptap/extension-placeholder": "^2.0.0",
//   "dompurify": "^3.0.0"  (+ devDep "@types/dompurify", "@tailwindcss/typography")
// [If Reporting = yes]
//   "xlsx": "^0.18.5", "papaparse": "^5.4.1"  (+ devDep "@types/papaparse")
```

### 2.2 vite.config.ts

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: {{DEV_PORT}},
    proxy: {
      '/api': {
        target: '{{API_BASE_URL_DEV}}',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, ''),
      },
    },
  },
});
```

> Note: Tailwind v3 is wired through PostCSS (see `postcss.config.js` in Section 3), so no
> Vite plugin is required for Tailwind itself.

### 2.3 tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

### 2.4 Environment Variables

```env
# .env.development
VITE_APP_TITLE={{APP_TITLE}}
VITE_APP_VERSION={{LATEST_VERSION}}
VITE_API_BASE_URL={{API_BASE_URL_DEV}}

# [If Auth = Keycloak]
VITE_KEYCLOAK_AUTHORITY=http://localhost:8180/realms/{{KEYCLOAK_REALM}}
VITE_KEYCLOAK_CLIENT_ID={{KEYCLOAK_CLIENT_ID}}
VITE_KEYCLOAK_REDIRECT_URI=http://localhost:{{DEV_PORT}}/auth/callback
VITE_KEYCLOAK_POST_LOGOUT_REDIRECT_URI=http://localhost:{{DEV_PORT}}
VITE_KEYCLOAK_SCOPE=openid profile email
```

```env
# .env.production
VITE_APP_TITLE={{APP_TITLE}}
VITE_APP_VERSION={{LATEST_VERSION}}
VITE_API_BASE_URL=${API_BASE_URL}

# [If Auth = Keycloak]
VITE_KEYCLOAK_AUTHORITY=${KEYCLOAK_AUTHORITY}
VITE_KEYCLOAK_CLIENT_ID=${KEYCLOAK_CLIENT_ID}
VITE_KEYCLOAK_REDIRECT_URI=${APP_URL}/auth/callback
VITE_KEYCLOAK_POST_LOGOUT_REDIRECT_URI=${APP_URL}
VITE_KEYCLOAK_SCOPE=openid profile email
```

---

## Section 3: Application Configuration

### 3.1 .gitignore

```gitignore
# Build output
dist/
dist-ssr/

# Dependencies
node_modules/

# IDE
.idea/
*.iml
.vscode/
*.swp
*~

# OS
.DS_Store
Thumbs.db

# Environment (never commit secrets)
.env
.env.local
.env.production
.env.*.local

# TypeScript
*.tsbuildinfo

# Logs
*.log
npm-debug.log*
yarn-debug.log*
pnpm-debug.log*

# Testing
coverage/
playwright-report/
test-results/
e2e/node_modules/
```

### 3.2 Environment Type Declarations

```typescript
// src/types/global.d.ts

/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_APP_TITLE: string;
  readonly VITE_APP_VERSION: string;
  readonly VITE_API_BASE_URL: string;
  // [If Auth = Keycloak]
  readonly VITE_KEYCLOAK_AUTHORITY: string;
  readonly VITE_KEYCLOAK_CLIENT_ID: string;
  readonly VITE_KEYCLOAK_REDIRECT_URI: string;
  readonly VITE_KEYCLOAK_POST_LOGOUT_REDIRECT_URI: string;
  readonly VITE_KEYCLOAK_SCOPE: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

### 3.3 postcss.config.js

```javascript
export default {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

---

## Section 4: Directory Structure

Generate the full `src/` directory tree. Use actual module names from PRD.md and MODEL.md.
Follow the feature-based structure from `references/routing-patterns.md`.

```
src/
├── main.tsx
├── App.tsx
├── index.css                            ← Tailwind directives + design-token CSS variables
├── router/
│   └── index.tsx
├── lib/
│   ├── api/
│   │   ├── axiosInstance.ts
│   │   └── [If Auth = Keycloak] authBridge.ts
│   ├── auth/
│   │   ├── [If Auth = Keycloak] oidcConfig.ts
│   │   ├── [If Auth = Keycloak] keycloakRoles.ts
│   │   ├── [If Auth = Keycloak] useAuthUser.ts
│   │   └── roles.ts
│   ├── utils/
│   │   ├── cn.ts                         ← twMerge(clsx(...)) helper
│   │   └── zodErrors.ts
│   └── queryClient.ts
├── store/
│   ├── [If Auth = Local] auth.store.ts
│   ├── notification.store.ts
│   └── ui.store.ts
├── shared/
│   ├── layouts/
│   │   ├── DashboardLayout.tsx
│   │   ├── PublicLayout.tsx
│   │   ├── AuthLayout.tsx
│   │   └── Footer.tsx
│   ├── components/
│   │   ├── ProtectedRoute.tsx
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   ├── AppTopBar.tsx
│   │   ├── AppSidebar.tsx
│   │   ├── navigationConfig.ts
│   │   ├── PageHeader.tsx
│   │   ├── DataTable.tsx
│   │   ├── Pagination.tsx
│   │   ├── ConfirmDialog.tsx
│   │   ├── StatusBadge.tsx
│   │   ├── Modal.tsx
│   │   ├── ImageUpload.tsx
│   │   ├── LoadingOverlay.tsx
│   │   ├── Spinner.tsx
│   │   ├── EmptyState.tsx
│   │   ├── ErrorBoundary.tsx
│   │   ├── NotificationProvider.tsx
│   │   └── SearchInput.tsx
│   ├── hooks/
│   │   ├── useDebounce.ts
│   │   ├── useNotification.ts
│   │   ├── useColorMode.ts
│   │   └── useConfirm.ts
│   └── form/
│       ├── TextFieldController.tsx
│       ├── TextAreaController.tsx
│       ├── SelectController.tsx          ← Headless UI Listbox
│       ├── DatePickerController.tsx      ← react-day-picker in a Popover
│       ├── SwitchController.tsx          ← Headless UI Switch
│       ├── ComboboxController.tsx        ← Headless UI Combobox
│       └── [If RichText = yes] RichTextController.tsx  ← Tiptap
├── features/
│   ├── auth/
│   │   ├── pages/
│   │   │   ├── [If Auth = Keycloak] AuthCallbackPage.tsx
│   │   │   └── [If Auth = Local] LoginPage.tsx
│   │   └── index.ts
│   ├── dashboard/
│   │   ├── pages/
│   │   │   └── DashboardPage.tsx
│   │   └── index.ts
│   ├── {{module1}}/               ← Replace with actual module names
│   │   ├── api/
│   │   │   └── {{module1}}Api.ts
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── pages/
│   │   ├── {{module1}}.types.ts
│   │   ├── {{module1}}.schema.ts
│   │   └── index.ts
│   └── {{module2}}/
│       └── (same structure)
└── types/
    └── global.d.ts
```

> **Note:** `tailwind.config.js` and `postcss.config.js` live at the project root (not under
> `src/`), alongside `package.json` and `vite.config.ts`.

---

## Section 5: Tailwind Configuration & Design Tokens

Generate the complete Tailwind configuration using design tokens extracted from MOCKUP.html.
Reference `references/component-patterns.md` for the token setup pattern.

Include:
- `tailwind.config.js` — `darkMode: 'class'`, content globs, `theme.extend` with actual
  color/font/radius/shadow tokens from the mockup, and `plugins`
- `src/index.css` — `@tailwind` directives + `:root` / `.dark` CSS custom properties
- `src/lib/utils/cn.ts` — the `cn()` helper
- Design token mapping table showing mockup CSS property → Tailwind token

### 5.1 tailwind.config.js

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  darkMode: 'class',
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        // Map to CSS variables so light/dark switch automatically (see index.css)
        primary: {
          DEFAULT: 'rgb(var(--color-primary) / <alpha-value>)',
          dark: 'rgb(var(--color-primary-dark) / <alpha-value>)',
        },
        secondary: 'rgb(var(--color-secondary) / <alpha-value>)',
        danger: 'rgb(var(--color-danger) / <alpha-value>)',
        warning: 'rgb(var(--color-warning) / <alpha-value>)',
        success: 'rgb(var(--color-success) / <alpha-value>)',
        surface: 'rgb(var(--color-surface) / <alpha-value>)',
        canvas: 'rgb(var(--color-canvas) / <alpha-value>)',
      },
      fontFamily: {
        // Extract from mockup CSS font-family declaration
        sans: ['{{FONT_FAMILY}}', 'system-ui', 'sans-serif'],
      },
      borderRadius: {
        // Extract base radius from mockup (e.g., '0.5rem')
        DEFAULT: '{{BORDER_RADIUS_BASE}}',
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    // [If RichText = yes] require('@tailwindcss/typography'),
  ],
};
```

### 5.2 src/index.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* Semantic design tokens — values extracted from MOCKUP.html.
   Stored as space-separated RGB channels so Tailwind's <alpha-value> works. */
:root {
  --color-primary: 34 113 177;        /* e.g., #2271b1 */
  --color-primary-dark: 19 94 150;    /* e.g., #135e96 */
  --color-secondary: 100 105 112;     /* e.g., #646970 */
  --color-danger: 214 54 56;          /* e.g., #d63638 */
  --color-warning: 219 166 23;        /* e.g., #dba617 */
  --color-success: 0 163 42;          /* e.g., #00a32a */
  --color-canvas: 240 240 241;        /* page background — e.g., #f0f0f1 */
  --color-surface: 255 255 255;       /* card/surface background — #ffffff */
}

.dark {
  --color-primary: 96 165 250;
  --color-primary-dark: 59 130 246;
  --color-secondary: 148 163 184;
  --color-danger: 248 113 113;
  --color-warning: 250 204 21;
  --color-success: 74 222 128;
  --color-canvas: 15 23 42;           /* slate-900 */
  --color-surface: 30 41 59;          /* slate-800 */
}

body {
  @apply bg-canvas text-slate-900 antialiased dark:text-slate-100;
}
```

### 5.3 cn() helper

```typescript
// src/lib/utils/cn.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

/**
 * Merge Tailwind class names, resolving conflicts (last-wins) via tailwind-merge.
 * Use this for every conditional/composed className in the application.
 */
export function cn(...inputs: ClassValue[]): string {
  return twMerge(clsx(inputs));
}
```

### 5.4 main.tsx integration

```tsx
// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from './lib/queryClient';
import { App } from './App';
import './index.css';
// [If Auth = Keycloak]
// import { AuthProvider } from 'react-oidc-context';
// import { oidcConfig } from './lib/auth/oidcConfig';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    {/* [If Auth = Keycloak] <AuthProvider {...oidcConfig}> */}
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
    {/* [If Auth = Keycloak] </AuthProvider> */}
  </React.StrictMode>,
);
```

### 5.5 App.tsx integration

```tsx
// src/App.tsx
import { useEffect } from 'react';
import { RouterProvider } from 'react-router-dom';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useUiStore } from './store/ui.store';
import { router } from './router';
import { NotificationProvider } from './shared/components/NotificationProvider';
import { ConfirmDialog } from './shared/components/ConfirmDialog';
// [If Auth = Keycloak]
// import { useAuth } from 'react-oidc-context';
// import { setAuthUser } from './lib/api/authBridge';

export function App() {
  const themeMode = useUiStore((s) => s.themeMode);

  // Sync the Tailwind `dark` class on the document root
  useEffect(() => {
    document.documentElement.classList.toggle('dark', themeMode === 'dark');
  }, [themeMode]);

  // [If Auth = Keycloak] Keep the auth bridge in sync
  // const auth = useAuth();
  // useEffect(() => { setAuthUser(auth.user ?? null); }, [auth.user]);

  return (
    <>
      <RouterProvider router={router} />
      <NotificationProvider />
      <ConfirmDialog />
      <ReactQueryDevtools initialIsOpen={false} />
    </>
  );
}
```

---

## Section 6: Authentication Configuration [CONDITIONAL — Include only if Auth != none]

**[If Auth = Keycloak]:**
Refer to `references/security-patterns.md` for the complete content.
Must cover:
- `src/lib/auth/oidcConfig.ts` — OIDC configuration object
- `src/lib/auth/keycloakRoles.ts` — role extraction utilities
- `src/lib/auth/useAuthUser.ts` — typed auth hook
- `src/lib/auth/roles.ts` — role constants
- `src/lib/api/authBridge.ts` — bridge for Axios interceptor
- `src/shared/components/ProtectedRoute.tsx` — route guard component
- `src/features/auth/pages/AuthCallbackPage.tsx` — OIDC callback handler
- main.tsx integration with `AuthProvider`
- Axios interceptor that injects Bearer token

**[If Auth = Local]:**
Refer to `references/security-patterns.md` (Local JWT section) for the complete content.
Must cover:
- `src/store/auth.store.ts` — Zustand auth store with login/logout/token management
- `src/lib/api/axiosInstance.ts` — interceptor reads token from Zustand store
- Token refresh interceptor for 401 handling
- `src/shared/components/ProtectedRoute.tsx` — redirects to /login
- `src/features/auth/pages/LoginPage.tsx` — email/password form with RHF + Zod + Tailwind
- `src/lib/auth/roles.ts` — role constants

---

## Section 7: Router Configuration

Refer to `references/routing-patterns.md` for the complete route tree.
Must cover:
- `src/router/index.tsx` — complete `createBrowserRouter()` with ALL routes from PRD.md
- Lazy loading via `React.lazy()` for every page component
- `<Suspense>` with `<LoadingOverlay />` fallback
- `<ProtectedRoute>` wrappers with `requiredRoles` matching the mockup role folders
- Public routes (login, callback, 404)
- The exact module-based URL paths (e.g., `/hero-section`, NOT `/admin/hero-section`)

---

## Section 8: API Client Configuration

```typescript
// src/lib/api/axiosInstance.ts
import axios from 'axios';

export const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 30_000,
  headers: { 'Content-Type': 'application/json' },
  // [If Auth = Local] withCredentials: true, // for httpOnly refresh token cookie
});

// Auth interceptor — see Section 6 for implementation
// Error interceptor — see below
axiosInstance.interceptors.response.use(
  (response) => response,
  (error) => {
    if (axios.isAxiosError(error)) {
      const status = error.response?.status;
      const message =
        (error.response?.data as { message?: string })?.message ??
        error.message ??
        'An unexpected error occurred';

      // Global error notifications for non-401 errors
      // (401 handled by the auth interceptor)
      if (status && status !== 401) {
        // Imported lazily to avoid circular dependency with Zustand store
        import('../../store/notification.store').then(({ useNotificationStore }) => {
          useNotificationStore.getState().push('error', message);
        });
      }
    }
    return Promise.reject(error);
  },
);
```

---

## Section 9: Global State (Zustand)

Describe and provide code for:
- `src/store/notification.store.ts` — toast notification queue
- `src/store/ui.store.ts` — sidebar open/closed, theme mode (persisted)
- [If Auth = Local] `src/store/auth.store.ts` — auth state and actions

Reference `references/component-patterns.md` for the full store implementations.

---

## Section 10: TanStack Query Setup

```typescript
// src/lib/queryClient.ts
// Full implementation from references/component-patterns.md
```

Describe:
- `QueryClient` configuration (staleTime, gcTime, retry policy)
- `QueryClientProvider` in `main.tsx`
- `ReactQueryDevtools` in `App.tsx` (dev mode only)
- Query key factory pattern (object with `all`, `lists`, `list(params)`, `detail(id)`)
- Naming convention: `use{Entity}s` for list queries, `use{Entity}` for single queries,
  `use{Entity}Mutations` for all CUD mutations of a feature

---

## Section 11: Shared Layouts

For each layout, provide:
1. The TypeScript props interface
2. The complete TSX component

- **DashboardLayout**: Tailwind flex layout — fixed sidebar (`AppSidebar`) + topbar
  (`AppTopBar`) + scrollable `<main>` with `<Outlet>` and `<Footer>`
  from `references/routing-patterns.md`
- **PublicLayout**: Minimal layout for landing/public pages
- **AuthLayout**: Centered card (`max-w-md` panel) for login/callback pages

---

## Section 12: Shared Components

For each shared component, provide the TypeScript props interface and complete TSX.
Reference `references/component-patterns.md` for implementations.

| Component | Purpose |
|---|---|
| `Button` | cva-based button with `variant`/`size` props (primary, outline, ghost, danger) |
| `Card` | Surface container (`bg-surface rounded shadow-sm border`) |
| `ProtectedRoute` | Auth + role guard |
| `AppTopBar` | Sticky topbar with sidebar toggle, user menu (Headless UI `Menu`), theme toggle |
| `AppSidebar` | Off-canvas/persistent sidebar with role-filtered navigation items |
| `navigationConfig.ts` | Navigation item definitions with role requirements |
| `PageHeader` | Page title + breadcrumbs + action buttons area |
| `StatusBadge` | cva-based colored badge for enum status values |
| `ConfirmDialog` + `useConfirm` | Imperative confirmation dialog (Headless UI `Dialog`) |
| `Modal` / `FormDialog` | Headless UI `Dialog` wrapper for forms |
| `LoadingOverlay` / `Spinner` | Full-page spinner overlay |
| `EmptyState` | Empty list placeholder with icon and message |
| `ErrorBoundary` | React ErrorBoundary with Tailwind fallback UI |
| `NotificationProvider` | Headless UI `Transition` toast stack renderer |
| `DataTable` | TanStack Table wrapper (or simple Tailwind table) |
| `Pagination` | Page navigation control |
| `ImageUpload` | Drag-and-drop file input with image preview |
| `SearchInput` | Debounced search text field |

---

## Section 13: Navigation Configuration

Generate `src/shared/components/navigationConfig.ts` with the actual navigation items
from the mockup sidebar files. For each role:
- List all nav items that role can see
- Include Heroicons imports (use appropriate icons from `@heroicons/react/24/outline`)
- Sidebar hrefs use module-based paths only (e.g., `/hero-section`, `/users`)

---

## Section 14: Form Infrastructure

Reference `references/component-patterns.md` for the complete RHF controller components.
Describe:
- `useForm()` + `zodResolver()` usage pattern
- All `*Controller` components with full TypeScript generics
- Form reset pattern for edit modes (`reset(data)` in `useEffect`)
- Loading state on submit button (`isSubmitting` → disabled + spinner inside `<Button>`)

---

## Section 15: Error Handling Strategy

Describe:
- `ErrorBoundary` component wrapping the router outlet
- Axios response interceptor for API errors (already in Section 8)
- Zod error message extraction utility
- TanStack Query `mutation.error` → displayed in form via `useEffect`

```typescript
// src/lib/utils/zodErrors.ts

import type { ZodError } from 'zod';

/**
 * Extract a flat error message from a Zod error or generic Error.
 */
export function extractErrorMessage(error: unknown): string {
  if (error instanceof Error) return error.message;
  if (typeof error === 'string') return error;
  return 'An unexpected error occurred';
}

/**
 * Map Zod field errors to React Hook Form setError calls.
 */
export function mapZodErrorsToForm<T extends Record<string, unknown>>(
  error: ZodError,
  setError: (field: keyof T, error: { message: string }) => void,
): void {
  for (const issue of error.issues) {
    const field = issue.path[0] as keyof T;
    if (field) {
      setError(field, { message: issue.message });
    }
  }
}
```

---

## Section 16: Notification System

Reference `references/component-patterns.md` for:
- `src/store/notification.store.ts` — full Zustand store
- `src/shared/components/NotificationProvider.tsx` — Headless UI `Transition` toast stack
- Usage: `useNotificationStore().push('success', 'Item saved.')` from any component or
  mutation `onSuccess`/`onError` callback

---

## Section 17: Theming & Dark Mode

```tsx
// src/shared/hooks/useColorMode.ts
import { useUiStore } from '../../store/ui.store';

/** Returns the current theme mode and a toggle action. */
export function useColorMode() {
  const themeMode = useUiStore((s) => s.themeMode);
  const toggleThemeMode = useUiStore((s) => s.toggleThemeMode);
  return { themeMode, toggleThemeMode };
}
```

```tsx
// src/shared/components/ThemeToggle.tsx (theme toggle example)
import { MoonIcon, SunIcon } from '@heroicons/react/24/outline';
import { useColorMode } from '../hooks/useColorMode';

export function ThemeToggle() {
  const { themeMode, toggleThemeMode } = useColorMode();

  return (
    <button
      type="button"
      onClick={toggleThemeMode}
      aria-label="Toggle theme"
      className="rounded p-2 text-slate-600 hover:bg-slate-100 dark:text-slate-300 dark:hover:bg-slate-700"
    >
      {themeMode === 'dark' ? (
        <SunIcon className="h-5 w-5" />
      ) : (
        <MoonIcon className="h-5 w-5" />
      )}
    </button>
  );
}
```

The `dark` class is toggled on `document.documentElement` in `App.tsx` (see Section 5.5),
driven by the persisted `themeMode` value in the Zustand UI store.

---

## Section 18: Testing Strategy

Overview only — detailed per-feature test patterns in each module's SPEC.md.

- **Vitest + React Testing Library**: Unit tests for hooks and components
- **MSW (Mock Service Worker)**: API mocking for integration tests
- **Playwright**: E2E test scripts per CO2 workflow

```bash
# Add to package.json devDependencies:
# "vitest": "^2.0.0"
# "@testing-library/react": "^16.0.0"
# "@testing-library/user-event": "^14.0.0"
# "msw": "^2.0.0"
# "jsdom": "^25.0.0"
```

---

## Section 19: Build & Deployment

### Vite Build

```bash
npm run build
# Output: dist/ — static files ready for deployment
```

### nginx Configuration (SPA Routing)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # All routes fall back to index.html for client-side routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Static assets — cache aggressively (Vite hashes filenames)
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Chunk Splitting (vite.config.ts)

```typescript
// Optimised chunk splitting — add to build.rollupOptions
rollupOptions: {
  output: {
    manualChunks: {
      vendor: ['react', 'react-dom', 'react-router-dom'],
      ui: ['@headlessui/react', '@heroicons/react'],
      query: ['@tanstack/react-query', 'axios'],
      forms: ['react-hook-form', '@hookform/resolvers', 'zod'],
      zustand: ['zustand'],
      // [If DataGrid = yes] table: ['@tanstack/react-table'],
      // [If Charts = yes] charts: ['recharts'],
    },
  },
},
```

> Tailwind ships no runtime JS — utility classes are compiled to a single CSS file at build
> time and tree-shaken against the `content` globs, so there is no "ui CSS" vendor chunk to split.

---

## Section 20: Internationalisation [CONDITIONAL — Include only if i18n = yes]

```typescript
// src/lib/i18n.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';
import HttpBackend from 'i18next-http-backend';

i18n
  .use(HttpBackend)
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    ns: ['common'],
    defaultNS: 'common',
    backend: {
      loadPath: '/locales/{{lng}}/{{ns}}.json',
    },
    interpolation: { escapeValue: false },
  });

export default i18n;
```

---

# Part B: Per-Module SPEC.md Template

Each module from PRD.md and MODEL.md gets its own `<module-name>/SPEC.md` file.

---

## Module SPEC.md Structure

```markdown
# {{ModuleName}} — Module Specification

> Part of [{{APPLICATION_NAME}} Technical Specification](../SPECIFICATION.md)

## Overview

**Module:** {{ModuleName}}
**Feature Path:** `src/features/{{moduleName}}/`
**Type:** {{System Module | Business Module}}
**Backend Endpoint Base:** `/{{api-path}}` (inferred from model/API context)
```

---

### Traceability

```markdown
## Traceability

### User Stories
| ID | Version | Description |
|---|---|---|
| USA000030 | v1.0.0 | Create/update hero section content |
| USA000033 | v1.0.0 | View list of hero section content |

### Non-Functional Requirements
| ID | Version | Description |
|---|---|---|
| NFRA00021 | v1.0.0 | Image must be exactly 1600x500 pixels |
| NFRA00024 | v1.0.0 | Headline max 100 characters |

### Constraints
| ID | Version | Description |
|---|---|---|
| CONSA0012 | v1.0.0 | Status values: DRAFT, ACTIVE, EXPIRED |

### Removed / Replaced
| ID | Removed In | Replaced By | Reason |
|---|---|---|---|
| CONSA0012 (READY status) | v1.0.4 | — | Auto-computed status; READY removed |
_or "None." if no removals_

### Data Sources
| Artifact | Reference |
|---|---|
| DB Table | `hero_section` |
| Mockup Screen | `admin/content/hero_section.html` |
```

---

### TypeScript Types

```markdown
## TypeScript Types

File: `src/features/heroSection/heroSection.types.ts`

(Complete TypeScript interfaces and enums — field-for-field from MODEL.md)
```

All enum values must match PRD.md constraints exactly (e.g., status DRAFT/ACTIVE/EXPIRED).
Include the generic `PaginatedResponse<T>` type in the shared types file, not per-module.

---

### Zod Schema

```markdown
## Zod Schema

File: `src/features/heroSection/heroSection.schema.ts`

(Complete Zod schema with all validation rules from PRD.md NFRs and Constraints)
```

Rules must be explicitly derived from tagged NFR IDs:
- Character limits → `.max(N, '...')`
- URL format → `.url('...')`
- Required fields → `.min(1, '...')`
- Cross-field rules → `.refine()`

---

### API Functions

```markdown
## API Functions

File: `src/features/heroSection/api/heroSectionApi.ts`

(Complete API client object with all methods matching user stories)
```

- `list(params)` — for list/search user stories
- `getById(id)` — for view detail user stories
- `create(payload, imageFile?)` — for create user stories
- `update(id, payload, imageFile?)` — for update user stories
- `delete(id)` — for delete user stories
- `imageUrl(id)` — if the module has image fields stored as blobs/binary

---

### TanStack Query Hooks

```markdown
## TanStack Query Hooks

Files:
- `src/features/heroSection/hooks/useHeroSections.ts`
- `src/features/heroSection/hooks/useHeroSectionMutations.ts`

(Complete query key factory and hook implementations)
```

- Query key factory object with `all`, `lists()`, `list(params)`, `detail(id)` entries
- `use{Entity}s(params)` — paginated list query
- `use{Entity}(id)` — single item query (disabled when id is empty)
- `useCreate{Entity}()` — create mutation with cache invalidation
- `useUpdate{Entity}()` — update mutation with cache invalidation
- `useDelete{Entity}()` — delete mutation with cache invalidation

---

### Page Components

```markdown
## Page Components

Files:
- `src/features/heroSection/pages/HeroSectionListPage.tsx`
- `src/features/heroSection/pages/HeroSectionFormPage.tsx`

(Complete page component implementations matching the mockup screens)
```

**CRITICAL — page component requirements:**
1. Use `PageHeader` with breadcrumbs matching `Dashboard → Module Name`
2. Use `useParams()` in form page to distinguish create vs edit mode
3. Reset form when edit data loads: `useEffect(() => { if (data) reset(mapToFormValues(data)); }, [data, reset])`
4. Navigate back after successful submit: `navigate('/hero-section')`
5. Use `LoadingOverlay` while edit data is loading
6. Use `ConfirmDialog` for delete actions via `useConfirm()`
7. Style all layout with Tailwind utilities; use Headless UI for any modal/menu/dropdown

Form page pattern for create/edit:
```tsx
// The same page handles both create and edit based on the presence of :id param
export function HeroSectionFormPage() {
  const { id } = useParams<{ id: string }>();
  const isEditMode = !!id;
  // ...fetch data if isEditMode, reset form when data loads
}
```

---

### Form Component

```markdown
## Form Component

File: `src/features/heroSection/components/HeroSectionForm.tsx`

(Complete form component using React Hook Form + Zod + Tailwind/Headless UI controllers)
```

---

### Route Definitions

```markdown
## Route Definitions

Add to `src/router/index.tsx`:

(Complete route definitions for this module — lazy-loaded, with ProtectedRoute wrapping
and requiredRoles matching the mockup role folder access control)
```

---

### Navigation Items

```markdown
## Navigation Items

Add to `src/shared/components/navigationConfig.ts`:

(NavItem objects for this module, per role)
```

---

### Status Badge Map (if module has status enum)

```markdown
## Status Configuration

```typescript
// Status display map for use with <StatusBadge />
const HERO_SECTION_STATUS_MAP = {
  DRAFT:   { label: 'Draft',   variant: 'neutral' as const },
  ACTIVE:  { label: 'Active',  variant: 'success' as const },
  EXPIRED: { label: 'Expired', variant: 'danger'  as const },
};
```

---

### CRITICAL Rules for Module SPEC.md

1. **Use real field names** from MODEL.md — not `fieldOne`/`fieldTwo`
2. **Use real table/collection names** from MODEL.md
3. **Map every user story** to a specific API function or hook
4. **Map every NFR constraint** to a Zod rule or component behavior
5. **Use module-based URLs** — `/hero-section`, NOT `/admin/hero-section`
6. **Role guards** come from mockup role folder structure → `requiredRoles` on routes
7. **Version-tag every ID** in traceability tables
8. **Never use `any` type** — use proper TypeScript types or `unknown`
9. **All code samples must be complete** — no `// ...` gaps
10. **Form page handles both create and edit** via `useParams()` unless the mockup
    has clearly separate screens
11. **Tailwind + Headless UI only** — no MUI/Chakra/Ant; compose classes with `cn()`,
    variants with `cva`, dark mode via `dark:` variants
```
