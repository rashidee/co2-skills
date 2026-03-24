# Security Patterns — OAuth2 PKCE (Keycloak) and Local JWT Auth

This reference describes the complete authentication architecture for the spec. Include
all of this content in Section 6 of the generated specification.

---

## Architecture Overview — Keycloak PKCE (Auth = Keycloak)

The application operates as an OAuth2 Public Client using Authorization Code flow with
PKCE (Proof Key for Code Exchange) and OIDC. Keycloak is the external identity provider
that handles login UI, user registration, and role management. The application's
responsibilities are:

1. Redirect unauthenticated users to Keycloak login page (PKCE challenge generated client-side)
2. Handle the OAuth2 callback and exchange the authorization code for tokens
3. Store tokens securely in memory (Zustand store) — NOT in localStorage
4. Extract roles from OIDC ID token claims
5. Attach Bearer token to all API requests via Axios interceptor
6. Silently refresh the access token before expiry

The flow:
```
User navigates to protected route
    → ProtectedRoute detects no auth
    → oidc-client-ts generates code_verifier + code_challenge (PKCE)
    → 302 Redirect to Keycloak /auth?response_type=code&code_challenge=...
    → User logs in at Keycloak
    → Keycloak redirects to /auth/callback?code=...
    → AuthCallback page exchanges code for tokens (ID token + access token)
    → react-oidc-context stores tokens in memory
    → AuthProvider exposes user + token via useAuth() hook
    → Axios interceptor reads token from auth context
    → User redirected to originally-requested page
```

---

## Package Dependencies

```json
{
  "dependencies": {
    "oidc-client-ts": "^3.1.0",
    "react-oidc-context": "^3.2.0"
  }
}
```

---

## Environment Variables

```env
# .env.development
VITE_KEYCLOAK_AUTHORITY=http://localhost:8180/realms/{{KEYCLOAK_REALM}}
VITE_KEYCLOAK_CLIENT_ID={{KEYCLOAK_CLIENT_ID}}
VITE_KEYCLOAK_REDIRECT_URI=http://localhost:3000/auth/callback
VITE_KEYCLOAK_POST_LOGOUT_REDIRECT_URI=http://localhost:3000
VITE_KEYCLOAK_SCOPE=openid profile email

# .env.production
VITE_KEYCLOAK_AUTHORITY=${KEYCLOAK_AUTHORITY}
VITE_KEYCLOAK_CLIENT_ID=${KEYCLOAK_CLIENT_ID}
VITE_KEYCLOAK_REDIRECT_URI=${APP_URL}/auth/callback
VITE_KEYCLOAK_POST_LOGOUT_REDIRECT_URI=${APP_URL}
VITE_KEYCLOAK_SCOPE=openid profile email
```

---

## OIDC Configuration

```typescript
// src/lib/auth/oidcConfig.ts
import type { AuthProviderProps } from 'react-oidc-context';

export const oidcConfig: AuthProviderProps = {
  authority: import.meta.env.VITE_KEYCLOAK_AUTHORITY,
  client_id: import.meta.env.VITE_KEYCLOAK_CLIENT_ID,
  redirect_uri: import.meta.env.VITE_KEYCLOAK_REDIRECT_URI,
  post_logout_redirect_uri: import.meta.env.VITE_KEYCLOAK_POST_LOGOUT_REDIRECT_URI,
  scope: import.meta.env.VITE_KEYCLOAK_SCOPE,
  response_type: 'code',
  // PKCE is enabled by default in oidc-client-ts
  // Silently renew the access token 60 seconds before expiry
  automaticSilentRenew: true,
  // Load user info from the OIDC userinfo endpoint
  loadUserInfo: true,
  onSigninCallback: (_user) => {
    // Remove the auth params from the URL after successful login
    window.history.replaceState({}, document.title, window.location.pathname);
  },
};
```

---

## AuthProvider Setup

```tsx
// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { AuthProvider } from 'react-oidc-context';
import { QueryClientProvider } from '@tanstack/react-query';
import { oidcConfig } from './lib/auth/oidcConfig';
import { queryClient } from './lib/queryClient';
import { App } from './App';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <AuthProvider {...oidcConfig}>
      <QueryClientProvider client={queryClient}>
        <App />
      </QueryClientProvider>
    </AuthProvider>
  </React.StrictMode>,
);
```

---

## Role Extraction from OIDC Token

Keycloak places roles in both `realm_access.roles` and `resource_access.<client_id>.roles`
within the ID token claims. Extract them using a utility:

```typescript
// src/lib/auth/keycloakRoles.ts
import type { User } from 'oidc-client-ts';

/**
 * Extract Keycloak roles from the OIDC user's ID token claims.
 * Combines realm roles and client-specific resource roles.
 */
export function extractKeycloakRoles(user: User | null | undefined): string[] {
  if (!user?.profile) return [];

  const profile = user.profile as Record<string, unknown>;
  const roles: string[] = [];

  // Realm-level roles
  const realmAccess = profile['realm_access'] as { roles?: string[] } | undefined;
  if (realmAccess?.roles) {
    roles.push(...realmAccess.roles);
  }

  // Client-specific roles
  const clientId = import.meta.env.VITE_KEYCLOAK_CLIENT_ID as string;
  const resourceAccess = profile['resource_access'] as
    Record<string, { roles?: string[] }> | undefined;
  if (resourceAccess?.[clientId]?.roles) {
    roles.push(...resourceAccess[clientId].roles);
  }

  return [...new Set(roles.map((r) => r.toUpperCase()))];
}

/**
 * Check if the user has a specific role.
 */
export function hasRole(user: User | null | undefined, role: string): boolean {
  return extractKeycloakRoles(user).includes(role.toUpperCase());
}

/**
 * Check if the user has any of the specified roles.
 */
export function hasAnyRole(user: User | null | undefined, roles: string[]): boolean {
  const userRoles = extractKeycloakRoles(user);
  return roles.some((r) => userRoles.includes(r.toUpperCase()));
}
```

---

## useAuthUser Hook

A thin wrapper over `react-oidc-context`'s `useAuth()` that adds role-based utilities:

```typescript
// src/lib/auth/useAuthUser.ts
import { useAuth } from 'react-oidc-context';
import { extractKeycloakRoles, hasRole, hasAnyRole } from './keycloakRoles';

export interface AuthUser {
  id: string;
  username: string;
  email: string;
  name: string;
  roles: string[];
  accessToken: string | undefined;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: () => Promise<void>;
  logout: () => Promise<void>;
  hasRole: (role: string) => boolean;
  hasAnyRole: (roles: string[]) => boolean;
}

export function useAuthUser(): AuthUser {
  const auth = useAuth();
  const roles = extractKeycloakRoles(auth.user);

  return {
    id: auth.user?.profile.sub ?? '',
    username: (auth.user?.profile.preferred_username as string) ?? '',
    email: auth.user?.profile.email ?? '',
    name: auth.user?.profile.name ?? '',
    roles,
    accessToken: auth.user?.access_token,
    isAuthenticated: auth.isAuthenticated,
    isLoading: auth.isLoading,
    login: () => auth.signinRedirect(),
    logout: () =>
      auth.signoutRedirect({
        post_logout_redirect_uri: import.meta.env.VITE_KEYCLOAK_POST_LOGOUT_REDIRECT_URI,
      }),
    hasRole: (role) => hasRole(auth.user, role),
    hasAnyRole: (r) => hasAnyRole(auth.user, r),
  };
}
```

---

## ProtectedRoute Component

```tsx
// src/shared/components/ProtectedRoute.tsx
import { useAuth } from 'react-oidc-context';
import { Navigate, useLocation } from 'react-router-dom';
import { hasAnyRole } from '../../lib/auth/keycloakRoles';
import { LoadingOverlay } from './LoadingOverlay';

interface ProtectedRouteProps {
  children: React.ReactNode;
  /** If provided, the user must have at least one of these roles */
  requiredRoles?: string[];
}

export function ProtectedRoute({ children, requiredRoles }: ProtectedRouteProps) {
  const auth = useAuth();
  const location = useLocation();

  if (auth.isLoading) {
    return <LoadingOverlay />;
  }

  if (!auth.isAuthenticated) {
    // Trigger Keycloak login redirect
    auth.signinRedirect({ state: { from: location.pathname } });
    return <LoadingOverlay />;
  }

  if (requiredRoles && !hasAnyRole(auth.user, requiredRoles)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <>{children}</>;
}
```

---

## AuthCallback Page

```tsx
// src/features/auth/pages/AuthCallbackPage.tsx
import { useEffect } from 'react';
import { useAuth } from 'react-oidc-context';
import { useNavigate } from 'react-router-dom';
import { LoadingOverlay } from '../../../shared/components/LoadingOverlay';

/**
 * Handles the OAuth2 callback redirect from Keycloak.
 * react-oidc-context processes the code automatically via onSigninCallback;
 * this page just shows a loading state during the exchange.
 */
export function AuthCallbackPage() {
  const auth = useAuth();
  const navigate = useNavigate();

  useEffect(() => {
    if (auth.isAuthenticated) {
      // Navigate to the page the user was originally trying to access, or home
      const from = (auth.user?.state as { from?: string })?.from ?? '/';
      navigate(from, { replace: true });
    }
  }, [auth.isAuthenticated, auth.user, navigate]);

  if (auth.error) {
    return (
      <div style={{ padding: 32 }}>
        <h2>Authentication Error</h2>
        <p>{auth.error.message}</p>
      </div>
    );
  }

  return <LoadingOverlay message="Completing sign-in..." />;
}
```

---

## Axios Interceptor (Keycloak)

```typescript
// src/lib/api/axiosInstance.ts
import axios from 'axios';
import { getAuthUser } from './authBridge';

export const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 30_000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Attach Bearer token to every request
axiosInstance.interceptors.request.use(
  (config) => {
    const token = getAuthUser()?.access_token;
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error),
);

// Global error handling
axiosInstance.interceptors.response.use(
  (response) => response,
  (error) => {
    if (axios.isAxiosError(error)) {
      const status = error.response?.status;
      if (status === 401) {
        // Token expired — oidc-client-ts silent renewal handles this automatically.
        // If silent renewal fails, the user is redirected to Keycloak login.
        console.warn('Unauthorized — triggering silent renew');
      } else if (status === 403) {
        console.warn('Forbidden — insufficient permissions');
      } else if (status && status >= 500) {
        console.error('Server error', error.response?.data);
      }
    }
    return Promise.reject(error);
  },
);
```

The `authBridge` module provides a way to access the current OIDC user from outside
React (needed for the Axios interceptor, which runs outside React's component tree):

```typescript
// src/lib/api/authBridge.ts
import type { User } from 'oidc-client-ts';

let _currentUser: User | null = null;

export function setAuthUser(user: User | null): void {
  _currentUser = user;
}

export function getAuthUser(): User | null {
  return _currentUser;
}
```

Register the bridge in the `App` component:

```tsx
// src/App.tsx (relevant excerpt)
import { useAuth } from 'react-oidc-context';
import { useEffect } from 'react';
import { setAuthUser } from './lib/api/authBridge';

export function App() {
  const auth = useAuth();

  // Keep the auth bridge in sync with the current OIDC user
  useEffect(() => {
    setAuthUser(auth.user ?? null);
  }, [auth.user]);

  return (/* router and providers */);
}
```

---

## OIDC ID Token Structure Reference

Keycloak places roles in the ID token as follows:

```json
{
  "sub": "f1234-abcd-5678",
  "preferred_username": "john.doe",
  "email": "john.doe@example.com",
  "name": "John Doe",
  "realm_access": {
    "roles": ["USER", "offline_access"]
  },
  "resource_access": {
    "{{KEYCLOAK_CLIENT_ID}}": {
      "roles": ["ADMIN"]
    }
  },
  "scope": "openid profile email"
}
```

This structure is what `extractKeycloakRoles()` parses.

---

## Role Constants

Define roles as a constants object to avoid string duplication:

```typescript
// src/lib/auth/roles.ts
export const Roles = {
  ADMIN: 'ADMIN',
  EDITOR: 'EDITOR',
  USER: 'USER',
} as const;

export type Role = (typeof Roles)[keyof typeof Roles];
```

---

## Architecture Overview — Local JWT Auth (Auth = Local)

When the backend manages authentication (no external IdP), the flow is:

```
User submits login form
    → POST /api/auth/login with { email, password }
    → Backend returns { accessToken, user }
    → Zustand auth store saves token in memory
    → Axios interceptor reads token from Zustand store
    → On page reload: token is lost, user must re-login (or use refresh token cookie)
```

### Zustand Auth Store (Local JWT)

```typescript
// src/store/auth.store.ts
import { create } from 'zustand';
import { axiosInstance } from '../lib/api/axiosInstance';

interface UserInfo {
  id: string;
  email: string;
  name: string;
  roles: string[];
}

interface AuthState {
  user: UserInfo | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
  setToken: (token: string, user: UserInfo) => void;
  clearAuth: () => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  user: null,
  accessToken: null,
  isAuthenticated: false,
  isLoading: false,

  login: async (email, password) => {
    set({ isLoading: true });
    try {
      const response = await axiosInstance.post<{ accessToken: string; user: UserInfo }>(
        '/auth/login',
        { email, password },
      );
      set({
        accessToken: response.data.accessToken,
        user: response.data.user,
        isAuthenticated: true,
        isLoading: false,
      });
    } catch (error) {
      set({ isLoading: false });
      throw error;
    }
  },

  logout: async () => {
    try {
      await axiosInstance.post('/auth/logout');
    } finally {
      set({ user: null, accessToken: null, isAuthenticated: false });
    }
  },

  setToken: (token, user) => set({ accessToken: token, user, isAuthenticated: true }),
  clearAuth: () => set({ user: null, accessToken: null, isAuthenticated: false }),
}));
```

### Axios Interceptor (Local JWT)

```typescript
// src/lib/api/axiosInstance.ts
import axios from 'axios';
import { useAuthStore } from '../../store/auth.store';

export const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 30_000,
  withCredentials: true, // Required for httpOnly refresh token cookie
  headers: { 'Content-Type': 'application/json' },
});

axiosInstance.interceptors.request.use((config) => {
  // Read token directly from Zustand store state (outside React)
  const token = useAuthStore.getState().accessToken;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

axiosInstance.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (!axios.isAxiosError(error)) return Promise.reject(error);
    const status = error.response?.status;
    const originalRequest = error.config as typeof error.config & { _retry?: boolean };

    if (status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      try {
        // Attempt token refresh using httpOnly cookie
        const res = await axiosInstance.post<{ accessToken: string; user: unknown }>(
          '/auth/refresh',
        );
        useAuthStore.getState().setToken(
          res.data.accessToken,
          res.data.user as Parameters<typeof useAuthStore.getState().setToken>[1],
        );
        // Retry the original request with the new token
        originalRequest.headers = originalRequest.headers ?? {};
        originalRequest.headers.Authorization = `Bearer ${res.data.accessToken}`;
        return axiosInstance(originalRequest);
      } catch {
        useAuthStore.getState().clearAuth();
        window.location.href = '/login';
      }
    }
    return Promise.reject(error);
  },
);
```

### ProtectedRoute Component (Local JWT)

```tsx
// src/shared/components/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuthStore } from '../../store/auth.store';
import { LoadingOverlay } from './LoadingOverlay';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRoles?: string[];
}

export function ProtectedRoute({ children, requiredRoles }: ProtectedRouteProps) {
  const { isAuthenticated, isLoading, user } = useAuthStore();
  const location = useLocation();

  if (isLoading) return <LoadingOverlay />;

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location.pathname }} replace />;
  }

  if (requiredRoles && !requiredRoles.some((r) => user?.roles.includes(r.toUpperCase()))) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <>{children}</>;
}
```
