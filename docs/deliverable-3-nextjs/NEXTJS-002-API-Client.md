# NEXTJS-002: Fineract API Client Design

> Detailed design for the HTTP client layer that replaces Angular's `HttpService`, interceptors, and caching in the Next.js application.

---

## Table of Contents

1. [Current Angular Pattern](#1-current-angular-pattern)
2. [Next.js API Client](#2-nextjs-api-client)
3. [Request Interceptor](#3-request-interceptor)
4. [Response Interceptor](#4-response-interceptor)
5. [Server-Side vs Client-Side](#5-server-side-vs-client-side)
6. [Caching Strategy](#6-caching-strategy)
7. [Error Types](#7-error-types)
8. [API Endpoint Constants](#8-api-endpoint-constants)
9. [Environment Variables](#9-environment-variables)

---

## 1. Current Angular Pattern

The existing Angular application uses a custom `HttpService` that extends Angular's `HttpClient` with a dynamic interceptor chain. Understanding this architecture is critical to designing the replacement.

### HttpService (`src/app/core/http/http.service.ts`)

The `HttpService` class extends `HttpClient` and manages a configurable chain of interceptors. It exposes fluent methods to modify the chain per-request:

- `cache(forceUpdate?)` -- Adds a `CacheInterceptor` to the chain for GET request caching
- `skipErrorHandler()` -- Removes `ErrorHandlerInterceptor` from the chain for requests that handle errors manually
- `disableApiPrefix()` -- Removes `ApiPrefixInterceptor` for requests to external URLs

The default interceptor chain is: `ApiPrefixInterceptor` then `ErrorHandlerInterceptor`.

### ApiPrefixInterceptor (`src/app/core/http/api-prefix.interceptor.ts`)

Prepends the Fineract server URL to all relative request URLs. It reads the server URL from `SettingsService` and has three URL resolution modes:

1. **Standard requests** (e.g., `/clients`) -- Prefixed with `serverUrl` which equals `baseApiUrl + /fineract-provider/api + /v1`
2. **Versioned requests** (e.g., `/v2/loans`) -- Prefixed with `baseServerUrl` (without `/v1`) to avoid double-versioning
3. **Actuator requests** (e.g., `/actuator/health`) -- Prefixed with just the server host
4. **Absolute URLs** (starting with `http://` or `https://`) -- Passed through unchanged (used for i18n JSON files and external APIs)

### AuthenticationInterceptor (`src/app/core/authentication/authentication.interceptor.ts`)

Adds authentication headers to every request:

- `Authorization: Basic <base64key>` for Fineract Basic Auth
- `Authorization: Bearer <token>` for OAuth2/OIDC
- `Fineract-Platform-TenantId: <tenantId>` on every request
- `Fineract-Platform-TFA-Token: <token>` when two-factor authentication is active

Skips Fineract headers for external API calls (URLs starting with `http://` or `https://`).

### ErrorHandlerInterceptor (`src/app/core/http/error-handler.interceptor.ts`)

Catches HTTP errors and dispatches alerts through `AlertService`:

- **401** -- Authentication error alert, triggers re-login
- **400** (when OAuth enabled) -- Treated as authentication error
- **403** -- Unauthorized request alert (special case for invalid 2FA token)
- **404** -- Resource not found alert (silently ignores missing client images)
- **500** -- Internal server error alert
- **501** -- Not implemented alert (uses i18n for message)
- **Other** -- Generic unknown error alert

### CacheInterceptor (`src/app/core/http/cache.interceptor.ts`)

Caches GET responses in `HttpCacheService` (an in-memory Map). Only activated when explicitly called via `this.http.cache()`. Supports `forceUpdate` to bypass and refresh the cache entry. Gated behind `environment.httpCacheEnabled`.

---

## 2. Next.js API Client

The replacement uses Axios for client-side requests and native `fetch` for server-side requests. Both share the same base URL configuration and error handling patterns.

### Client-Side Axios Instance

```typescript
// lib/api/client.ts
import axios, { AxiosError, AxiosInstance, InternalAxiosRequestConfig } from 'axios';
import { getSession } from 'next-auth/react';
import { useAlertStore } from '@/lib/stores/alert-store';
import { FineractApiError, parseFineractError } from './errors';

/**
 * Creates and configures the Axios instance for client-side API calls.
 *
 * This replaces Angular's HttpService + ApiPrefixInterceptor +
 * AuthenticationInterceptor + ErrorHandlerInterceptor.
 */
function createFineractClient(): AxiosInstance {
  const client = axios.create({
    baseURL: process.env.NEXT_PUBLIC_FINERACT_API_URL,
    headers: {
      'Content-Type': 'application/json',
      'Fineract-Platform-TenantId': process.env.NEXT_PUBLIC_TENANT_ID ?? 'default'
    },
    timeout: 30_000 // 30 second timeout
  });

  // Request interceptor: add auth header
  client.interceptors.request.use(addAuthHeader, Promise.reject);

  // Response interceptor: handle errors
  client.interceptors.response.use(
    (response) => response,
    (error) => handleResponseError(error)
  );

  return client;
}

export const fineractApi = createFineractClient();

// Default export for convenience
export default fineractApi;
```

### URL Resolution

The Angular app's `ApiPrefixInterceptor` logic for versioned and actuator URLs is handled at the endpoint constant level (see Section 8) rather than in an interceptor:

```typescript
// lib/api/endpoints.ts (partial)
const BASE = ''; // baseURL already includes /fineract-provider/api/v1

// Standard v1 endpoints use empty prefix (baseURL handles it)
export const CLIENTS = `${BASE}/clients`;

// v2 endpoints use explicit version (baseURL is reconfigured or overridden)
export const LOANS_V2 = '/v2/loans'; // handled by a separate Axios instance or baseURL override
```

---

## 3. Request Interceptor

The request interceptor replaces both `AuthenticationInterceptor` and `ApiPrefixInterceptor`.

```typescript
// lib/api/client.ts (continued)

/**
 * Request interceptor that adds the Authorization header.
 *
 * Replaces Angular's AuthenticationInterceptor.
 *
 * For Basic Auth: Authorization: Basic <base64key>
 * For OAuth2/OIDC: Authorization: Bearer <token>
 *
 * Also dynamically updates the tenant ID if the user has switched tenants.
 */
async function addAuthHeader(config: InternalAxiosRequestConfig): Promise<InternalAxiosRequestConfig> {
  // Skip auth headers for external URLs (same behavior as Angular interceptor)
  if (config.url?.startsWith('http://') || config.url?.startsWith('https://')) {
    // Remove default Fineract headers for external calls
    delete config.headers['Fineract-Platform-TenantId'];
    return config;
  }

  const session = await getSession();

  if (session?.accessToken) {
    // Determine auth type from session
    if (session.authMode === 'basic') {
      config.headers.Authorization = `Basic ${session.accessToken}`;
    } else {
      config.headers.Authorization = `Bearer ${session.accessToken}`;
    }
  }

  // Two-factor authentication token
  if (session?.twoFactorToken) {
    config.headers['Fineract-Platform-TFA-Token'] = session.twoFactorToken;
  }

  // Dynamic tenant ID (user may have switched tenants in settings)
  if (session?.tenantId) {
    config.headers['Fineract-Platform-TenantId'] = session.tenantId;
  }

  return config;
}
```

### Skipping Auth for Specific Requests

In Angular, `this.http.disableApiPrefix()` was used for external calls. In the Next.js version, use absolute URLs to skip auth:

```typescript
// External API call -- auth headers are automatically skipped
const response = await axios.get('https://external-service.com/api/data');

// Fineract call -- auth headers are automatically added
const clients = await fineractApi.get('/clients');
```

---

## 4. Response Interceptor

The response interceptor replaces `ErrorHandlerInterceptor` and maps Fineract error responses to typed errors.

```typescript
// lib/api/client.ts (continued)

/**
 * Response error interceptor.
 *
 * Replaces Angular's ErrorHandlerInterceptor.
 *
 * Maps HTTP status codes to user-facing alerts and typed errors.
 * Mirrors the exact behavior of the Angular interceptor.
 */
function handleResponseError(error: AxiosError<FineractErrorResponse>): Promise<never> {
  if (!error.response) {
    // Network error or timeout
    useAlertStore.getState().addAlert({
      type: 'error',
      title: 'Network Error',
      message: 'Unable to connect to the server. Please check your connection.'
    });
    return Promise.reject(new FineractApiError('NETWORK_ERROR', 'Network error', 0));
  }

  const { status, data, config } = error.response;
  const errorMessage = extractErrorMessage(data);
  const isClientImage404 = status === 404 && config.url?.includes('/clients/') && config.url?.includes('/images');

  const alertStore = useAlertStore.getState();

  switch (status) {
    case 401:
      alertStore.addAlert({
        type: 'error',
        title: 'Authentication Error',
        message: 'Invalid credentials. Please log in again.'
      });
      // Redirect to login
      if (typeof window !== 'undefined') {
        window.location.href = '/login';
      }
      break;

    case 400:
      alertStore.addAlert({
        type: 'warning',
        title: 'Bad Request',
        message: errorMessage || 'Invalid parameters were passed in the request.'
      });
      break;

    case 403:
      if (errorMessage === 'The provided one time token is invalid') {
        alertStore.addAlert({
          type: 'error',
          title: 'Invalid Token',
          message: 'Invalid OTP token. Please try again.'
        });
      } else {
        alertStore.addAlert({
          type: 'warning',
          title: 'Unauthorized',
          message: errorMessage || 'You are not authorized for this request.'
        });
      }
      break;

    case 404:
      if (!isClientImage404) {
        alertStore.addAlert({
          type: 'warning',
          title: 'Not Found',
          message: errorMessage || 'Resource does not exist.'
        });
      }
      // Silently handle missing client images (same as Angular)
      break;

    case 500:
      alertStore.addAlert({
        type: 'error',
        title: 'Server Error',
        message: 'Internal server error. Please try again later.'
      });
      break;

    default:
      alertStore.addAlert({
        type: 'error',
        title: 'Error',
        message: errorMessage || 'An unexpected error occurred. Please try again.'
      });
  }

  return Promise.reject(new FineractApiError(`HTTP_${status}`, errorMessage, status, data?.errors));
}

/**
 * Extracts the most useful error message from a Fineract error response.
 * Fineract returns errors in multiple formats.
 */
function extractErrorMessage(data: FineractErrorResponse | undefined): string {
  if (!data) return '';

  // Fineract error format: { errors: [{ defaultUserMessage, developerMessage }] }
  if (data.errors?.length) {
    return data.errors[0].defaultUserMessage || data.errors[0].developerMessage || '';
  }

  // Simple format: { developerMessage }
  if (data.developerMessage) return data.developerMessage;

  // Fallback
  if (typeof data === 'string') return data;

  return '';
}
```

---

## 5. Server-Side vs Client-Side

The Next.js application has two execution contexts. Each requires a different approach to API calls.

### Server Components -- `fetch` with Next.js Caching

Server Components run on the server during request processing. They should use the native `fetch` API (extended by Next.js) to benefit from built-in request deduplication and caching.

```typescript
// lib/api/server-client.ts
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth/auth-options';
import { FineractApiError, type FineractErrorResponse } from './errors';

const FINERACT_API_URL = process.env.FINERACT_API_URL!; // Server-only env var (no NEXT_PUBLIC_ prefix)
const TENANT_ID = process.env.FINERACT_TENANT_ID ?? 'default';

interface ServerFetchOptions {
  /** Next.js revalidation interval in seconds. 0 = no cache. */
  revalidate?: number | false;
  /** Next.js cache tags for on-demand revalidation. */
  tags?: string[];
}

/**
 * Server-side API client using native fetch with Next.js caching.
 *
 * Key differences from client-side Axios:
 * - Uses server-only env vars (FINERACT_API_URL, not NEXT_PUBLIC_FINERACT_API_URL)
 * - Reads auth token from the server session (cookies)
 * - Supports Next.js fetch cache (revalidate, tags)
 * - No interceptors needed (auth and error handling are inline)
 */
export async function serverFetch<T>(path: string, options: ServerFetchOptions & RequestInit = {}): Promise<T> {
  const session = await getServerSession(authOptions);

  if (!session?.accessToken) {
    throw new FineractApiError('UNAUTHORIZED', 'No active session', 401);
  }

  const { revalidate, tags, ...fetchOptions } = options;

  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    'Fineract-Platform-TenantId': session.tenantId ?? TENANT_ID,
    ...(session.authMode === 'basic' ? { Authorization: `Basic ${session.accessToken}` } : { Authorization: `Bearer ${session.accessToken}` })
  };

  if (session.twoFactorToken) {
    headers['Fineract-Platform-TFA-Token'] = session.twoFactorToken;
  }

  const url = `${FINERACT_API_URL}${path}`;

  const response = await fetch(url, {
    ...fetchOptions,
    headers: { ...headers, ...fetchOptions.headers },
    next: {
      revalidate: revalidate ?? 0, // Default: no cache (always fresh)
      tags: tags ?? []
    }
  });

  if (!response.ok) {
    const errorBody = await response.json().catch(() => ({}));
    throw new FineractApiError(`HTTP_${response.status}`, extractServerErrorMessage(errorBody), response.status, errorBody.errors);
  }

  // Handle empty responses (204 No Content, or DELETE responses)
  if (response.status === 204 || response.headers.get('content-length') === '0') {
    return {} as T;
  }

  return response.json();
}

function extractServerErrorMessage(data: any): string {
  if (data?.errors?.length) {
    return data.errors[0].defaultUserMessage || data.errors[0].developerMessage || '';
  }
  return data?.developerMessage || '';
}

// --- Convenience wrappers ---

/**
 * Fetch clients list from server component.
 *
 * Usage in a Server Component:
 *   const clients = await fetchClients({ offset: 0, limit: 50 });
 */
export async function fetchClients(params?: { offset?: number; limit?: number; sqlSearch?: string }) {
  const searchParams = new URLSearchParams();
  if (params?.offset) searchParams.set('offset', String(params.offset));
  if (params?.limit) searchParams.set('limit', String(params.limit));
  if (params?.sqlSearch) searchParams.set('sqlSearch', params.sqlSearch);

  const query = searchParams.toString();
  return serverFetch<ClientListResponse>(`/clients${query ? `?${query}` : ''}`, { revalidate: 30, tags: ['clients'] });
}

/**
 * Fetch a single client by ID.
 */
export async function fetchClient(clientId: number) {
  return serverFetch<ClientResponse>(`/clients/${clientId}`, { revalidate: 0, tags: [`client-${clientId}`] });
}
```

### Client Components -- Axios + TanStack Query

Client Components use the Axios instance (from Section 2) wrapped in TanStack Query hooks for caching, deduplication, and background refetching.

```typescript
// lib/services/clients.ts
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import fineractApi from '@/lib/api/client';
import type { ClientListResponse, ClientResponse, CreateClientPayload } from '@/lib/api/types/client';

// --- Query Key Factory ---
export const clientKeys = {
  all: ['clients'] as const,
  lists: () => [
      ...clientKeys.all,
      'list'
    ] as const,
  list: (params: Record<string, unknown>) => [
      ...clientKeys.lists(),
      params
    ] as const,
  details: () => [
      ...clientKeys.all,
      'detail'
    ] as const,
  detail: (id: number) => [
      ...clientKeys.details(),
      id
    ] as const,
  accounts: (id: number) => [
      ...clientKeys.detail(id),
      'accounts'
    ] as const,
  loans: (id: number) => [
      ...clientKeys.detail(id),
      'loans'
    ] as const,
  documents: (id: number) => [
      ...clientKeys.detail(id),
      'documents'
    ] as const,
  notes: (id: number) => [
      ...clientKeys.detail(id),
      'notes'
    ] as const,
  images: (id: number) => [
      ...clientKeys.detail(id),
      'images'
    ] as const
};

// --- Queries ---
export function useClients(params: { page: number; limit: number; search?: string }) {
  return useQuery({
    queryKey: clientKeys.list(params),
    queryFn: async () => {
      const { data } = await fineractApi.get<ClientListResponse>('/clients', {
        params: {
          offset: params.page * params.limit,
          limit: params.limit,
          ...(params.search && { sqlSearch: params.search })
        }
      });
      return data;
    },
    staleTime: 30_000 // 30 seconds
  });
}

export function useClient(id: number) {
  return useQuery({
    queryKey: clientKeys.detail(id),
    queryFn: async () => {
      const { data } = await fineractApi.get<ClientResponse>(`/clients/${id}`);
      return data;
    },
    staleTime: 0 // Always refetch on window focus
  });
}

// --- Mutations ---
export function useCreateClient() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (payload: CreateClientPayload) => {
      const { data } = await fineractApi.post('/clients', payload);
      return data;
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: clientKeys.lists() });
    }
  });
}

export function useActivateClient() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, ...payload }: { id: number; activationDate: string; locale: string; dateFormat: string }) => {
      const { data } = await fineractApi.post(`/clients/${id}?command=activate`, payload);
      return data;
    },
    onSuccess: (_data, variables) => {
      queryClient.invalidateQueries({ queryKey: clientKeys.detail(variables.id) });
      queryClient.invalidateQueries({ queryKey: clientKeys.lists() });
    }
  });
}
```

### When to Use Which

| Context                   | Client                   | When                                                                         |
| ------------------------- | ------------------------ | ---------------------------------------------------------------------------- |
| Server Component          | `serverFetch()`          | Page-level data fetching in `page.tsx` files. Benefits from Next.js caching. |
| Client Component (reads)  | TanStack Query hooks     | Interactive tables, search-as-you-type, polling, background refetch.         |
| Client Component (writes) | TanStack Query mutations | Form submissions, approve/reject actions, any POST/PUT/DELETE.               |
| API Route Handlers        | `serverFetch()`          | Proxy endpoints in `app/api/` (if needed for CORS or credential hiding).     |

---

## 6. Caching Strategy

Caching is split between two layers: **Next.js fetch cache** (server-side) and **TanStack Query cache** (client-side).

### TanStack Query Cache Configuration

```typescript
// lib/providers/query-provider.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { useState } from 'react';

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            // Default: refetch on window focus, retry 1 time
            refetchOnWindowFocus: true,
            retry: 1,
            retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 10_000),

            // Default stale time: 0 (always considered stale)
            staleTime: 0,

            // Default garbage collection time: 5 minutes
            gcTime: 5 * 60 * 1000,
          },
          mutations: {
            retry: 0, // Don't retry mutations
          },
        },
      }),
  );

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === 'development' && (
        <ReactQueryDevtools initialIsOpen={false} />
      )}
    </QueryClientProvider>
  );
}
```

### Per-Entity Cache Tuning

| Entity Type                                                | `staleTime`      | `gcTime` | Rationale                                                                                              |
| ---------------------------------------------------------- | ---------------- | -------- | ------------------------------------------------------------------------------------------------------ |
| **Client list**                                            | 30s              | 5 min    | Changes when new clients are created. Short stale time ensures fresh data without constant refetching. |
| **Single client**                                          | 0 (always stale) | 5 min    | Always refetch on focus to show latest status (activate, close, transfer).                             |
| **Loan list**                                              | 30s              | 5 min    | Same pattern as clients.                                                                               |
| **Single loan**                                            | 0                | 5 min    | Loan state changes frequently (approve, disburse, repay).                                              |
| **Savings list**                                           | 30s              | 5 min    | Same pattern.                                                                                          |
| **Reference data** (offices, staff, products, code values) | 5 min            | 30 min   | Rarely changes. Long stale time reduces API calls.                                                     |
| **Templates** (dropdown options from API)                  | 5 min            | 30 min   | Template endpoints return option lists for forms. Stable data.                                         |
| **Reports**                                                | 0                | 2 min    | Always fresh. Short GC since reports can be large.                                                     |
| **Notifications**                                          | 10s              | 2 min    | Polled frequently.                                                                                     |
| **Search results**                                         | 0                | 1 min    | Always fresh, short-lived.                                                                             |

### Server-Side Cache (Next.js fetch)

```typescript
// Revalidation intervals for serverFetch()
const CACHE_TIMES = {
  CLIENTS_LIST: 30, // seconds -- client list refreshes every 30s
  CLIENT_DETAIL: 0, // no cache -- always fresh
  OFFICES: 300, // 5 minutes -- offices rarely change
  STAFF: 300, // 5 minutes
  PRODUCTS: 300, // 5 minutes
  CODE_VALUES: 600, // 10 minutes -- code values almost never change
  REPORT: false as const // no cache, no revalidation
} as const;
```

### Cache Invalidation

TanStack Query invalidation replaces the Angular `CacheInterceptor`'s `forceUpdate` mechanism:

```typescript
// After creating a client, invalidate the client list cache
const queryClient = useQueryClient();
queryClient.invalidateQueries({ queryKey: clientKeys.lists() });

// After approving a loan, invalidate both the loan detail and the client's loans
queryClient.invalidateQueries({ queryKey: loanKeys.detail(loanId) });
queryClient.invalidateQueries({ queryKey: clientKeys.loans(clientId) });

// For server-side cache, use Next.js revalidateTag
import { revalidateTag } from 'next/cache';
revalidateTag('clients'); // Invalidates all fetches tagged with 'clients'
```

---

## 7. Error Types

Typed error handling replaces the Angular `ErrorHandlerInterceptor`'s untyped `HttpErrorResponse`.

```typescript
// lib/api/errors.ts

/**
 * Fineract API error response shape.
 *
 * The Fineract backend returns errors in this format:
 * {
 *   "developerMessage": "...",
 *   "httpStatusCode": "...",
 *   "defaultUserMessage": "...",
 *   "userMessageGlobalisationCode": "...",
 *   "errors": [
 *     {
 *       "developerMessage": "...",
 *       "defaultUserMessage": "...",
 *       "userMessageGlobalisationCode": "...",
 *       "parameterName": "...",
 *       "value": ...
 *     }
 *   ]
 * }
 */
export interface FineractErrorResponse {
  developerMessage?: string;
  httpStatusCode?: string;
  defaultUserMessage?: string;
  userMessageGlobalisationCode?: string;
  errors?: FineractValidationError[];
}

export interface FineractValidationError {
  developerMessage?: string;
  defaultUserMessage?: string;
  userMessageGlobalisationCode?: string;
  parameterName?: string;
  value?: unknown;
}

/**
 * Custom error class for Fineract API errors.
 *
 * Provides structured error information that components can use
 * for form-level validation errors, field-level errors, and user-facing messages.
 */
export class FineractApiError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly status: number,
    public readonly validationErrors?: FineractValidationError[]
  ) {
    super(message);
    this.name = 'FineractApiError';
  }

  /**
   * Check if this is an authentication error (401).
   */
  get isAuthError(): boolean {
    return this.status === 401;
  }

  /**
   * Check if this is a validation error (400 with field-level errors).
   */
  get isValidationError(): boolean {
    return this.status === 400 && (this.validationErrors?.length ?? 0) > 0;
  }

  /**
   * Check if this is a permission error (403).
   */
  get isForbidden(): boolean {
    return this.status === 403;
  }

  /**
   * Check if this is a not-found error (404).
   */
  get isNotFound(): boolean {
    return this.status === 404;
  }

  /**
   * Returns field-level errors as a map of parameterName -> message.
   * Useful for mapping Fineract validation errors to React Hook Form field errors.
   *
   * Example usage with React Hook Form:
   *   catch (error) {
   *     if (error instanceof FineractApiError && error.isValidationError) {
   *       const fieldErrors = error.getFieldErrors();
   *       Object.entries(fieldErrors).forEach(([field, message]) => {
   *         form.setError(field, { message });
   *       });
   *     }
   *   }
   */
  getFieldErrors(): Record<string, string> {
    const fieldErrors: Record<string, string> = {};
    if (this.validationErrors) {
      for (const err of this.validationErrors) {
        if (err.parameterName) {
          fieldErrors[err.parameterName] = err.defaultUserMessage || err.developerMessage || 'Validation error';
        }
      }
    }
    return fieldErrors;
  }
}

/**
 * Type guard to check if an unknown error is a FineractApiError.
 */
export function isFineractApiError(error: unknown): error is FineractApiError {
  return error instanceof FineractApiError;
}
```

### Integration with React Hook Form

```typescript
// Example: Create client form with server-side validation error mapping
'use client';

import { useForm } from 'react-hook-form';
import { useCreateClient } from '@/lib/services/clients';
import { isFineractApiError } from '@/lib/api/errors';

export function CreateClientForm() {
  const form = useForm<CreateClientPayload>();
  const createClient = useCreateClient();

  const onSubmit = form.handleSubmit(async (data) => {
    try {
      await createClient.mutateAsync(data);
    } catch (error) {
      if (isFineractApiError(error) && error.isValidationError) {
        // Map Fineract field errors to React Hook Form field errors
        const fieldErrors = error.getFieldErrors();
        Object.entries(fieldErrors).forEach(([field, message]) => {
          form.setError(field as any, { type: 'server', message });
        });
      }
      // Non-field errors are already handled by the response interceptor (toast alerts)
    }
  });

  return <form onSubmit={onSubmit}>{/* ... fields ... */}</form>;
}
```

---

## 8. API Endpoint Constants

All Fineract API endpoints organized by module. This replaces the scattered endpoint strings found throughout Angular service files.

```typescript
// lib/api/endpoints.ts

/**
 * Fineract API v1 endpoint constants.
 *
 * Organized by module to mirror the Angular feature module structure.
 * All paths are relative to the base URL (e.g., https://demo.mifos.community/fineract-provider/api/v1).
 */
export const API = {
  // --- Authentication ---
  AUTH: {
    LOGIN: '/authentication',
    TWO_FACTOR: '/twofactor',
    TWO_FACTOR_VALIDATE: '/twofactor/validate',
    TWO_FACTOR_INVALIDATE: '/twofactor/invalidate'
  },

  // --- Clients ---
  CLIENTS: {
    BASE: '/clients',
    BY_ID: (id: number) => `/clients/${id}`,
    ACCOUNTS: (id: number) => `/clients/${id}/accounts`,
    LOANS: (id: number) => `/clients/${id}/loans`, // Not a real Fineract endpoint; use ACCOUNTS
    IMAGES: (id: number) => `/clients/${id}/images`,
    DOCUMENTS: (id: number) => `/clients/${id}/documents`,
    DOCUMENT: (clientId: number, docId: number) => `/clients/${clientId}/documents/${docId}`,
    NOTES: (id: number) => `/clients/${id}/notes`,
    NOTE: (clientId: number, noteId: number) => `/clients/${clientId}/notes/${noteId}`,
    IDENTIFIERS: (id: number) => `/clients/${id}/identifiers`,
    CHARGES: (id: number) => `/clients/${id}/charges`,
    ADDRESSES: (id: number) => `/clients/${id}/addresses`,
    FAMILY_MEMBERS: (id: number) => `/clients/${id}/familymembers`,
    TEMPLATE: '/clients/template',
    TRANSFER: (id: number) => `/clients/${id}?command=proposeTransfer`,
    SURVEY: (id: number) => `/clients/${id}/survey`
  },

  // --- Groups ---
  GROUPS: {
    BASE: '/groups',
    BY_ID: (id: number) => `/groups/${id}`,
    ACCOUNTS: (id: number) => `/groups/${id}/accounts`,
    TEMPLATE: '/groups/template',
    NOTES: (id: number) => `/groups/${id}/notes`
  },

  // --- Centers ---
  CENTERS: {
    BASE: '/centers',
    BY_ID: (id: number) => `/centers/${id}`,
    ACCOUNTS: (id: number) => `/centers/${id}/accounts`,
    TEMPLATE: '/centers/template'
  },

  // --- Loans ---
  LOANS: {
    BASE: '/loans',
    BY_ID: (id: number) => `/loans/${id}`,
    TEMPLATE: '/loans/template',
    TRANSACTIONS: (id: number) => `/loans/${id}/transactions`,
    TRANSACTION: (loanId: number, txId: number) => `/loans/${loanId}/transactions/${txId}`,
    CHARGES: (id: number) => `/loans/${id}/charges`,
    CHARGE: (loanId: number, chargeId: number) => `/loans/${loanId}/charges/${chargeId}`,
    DOCUMENTS: (id: number) => `/loans/${id}/documents`,
    NOTES: (id: number) => `/loans/${id}/notes`,
    GUARANTORS: (id: number) => `/loans/${id}/guarantors`,
    COLLATERALS: (id: number) => `/loans/${id}/collaterals`,
    SCHEDULE: (id: number) => `/loans/${id}/repaymentschedule`,
    RESCHEDULE: '/rescheduleloans',
    RESCHEDULE_BY_ID: (id: number) => `/rescheduleloans/${id}`,
    GLIM: '/batches/glimAccount'
  },

  // --- Savings ---
  SAVINGS: {
    BASE: '/savingsaccounts',
    BY_ID: (id: number) => `/savingsaccounts/${id}`,
    TEMPLATE: '/savingsaccounts/template',
    TRANSACTIONS: (id: number) => `/savingsaccounts/${id}/transactions`,
    CHARGES: (id: number) => `/savingsaccounts/${id}/charges`,
    DOCUMENTS: (id: number) => `/savingsaccounts/${id}/documents`,
    NOTES: (id: number) => `/savingsaccounts/${id}/notes`
  },

  // --- Fixed Deposits ---
  FIXED_DEPOSITS: {
    BASE: '/fixeddepositaccounts',
    BY_ID: (id: number) => `/fixeddepositaccounts/${id}`,
    TEMPLATE: '/fixeddepositaccounts/template',
    TRANSACTIONS: (id: number) => `/fixeddepositaccounts/${id}/transactions`
  },

  // --- Recurring Deposits ---
  RECURRING_DEPOSITS: {
    BASE: '/recurringdepositaccounts',
    BY_ID: (id: number) => `/recurringdepositaccounts/${id}`,
    TEMPLATE: '/recurringdepositaccounts/template',
    TRANSACTIONS: (id: number) => `/recurringdepositaccounts/${id}/transactions`
  },

  // --- Shares ---
  SHARES: {
    BASE: '/shareaccounts',
    BY_ID: (id: number) => `/shareaccounts/${id}`,
    TEMPLATE: '/shareaccounts/template'
  },

  // --- Accounting ---
  ACCOUNTING: {
    GL_ACCOUNTS: '/glaccounts',
    GL_ACCOUNT: (id: number) => `/glaccounts/${id}`,
    JOURNAL_ENTRIES: '/journalentries',
    JOURNAL_ENTRY: (id: number) => `/journalentries/${id}`,
    CLOSING_ENTRIES: '/glclosures',
    ACCOUNT_RULES: '/accountingrules',
    PROVISIONING: '/provisioningentries',
    FINANCIAL_ACTIVITY_MAPPINGS: '/financialactivityaccounts'
  },

  // --- Organization ---
  ORGANIZATION: {
    OFFICES: '/offices',
    OFFICE: (id: number) => `/offices/${id}`,
    EMPLOYEES: '/staff',
    EMPLOYEE: (id: number) => `/staff/${id}`,
    CURRENCIES: '/currencies',
    MANAGE_FUNDS: '/funds',
    PAYMENT_TYPES: '/paymenttypes',
    WORKING_DAYS: '/workingdays',
    HOLIDAYS: '/holidays',
    PASSWORD_PREFERENCES: '/passwordpreferences',
    ENTITY_DATA_TABLE_CHECKS: '/entityDatatableChecks',
    TELLERS: '/tellers',
    CASHIERS: (tellerId: number) => `/tellers/${tellerId}/cashiers`,
    BULK_LOAN_REASSIGNMENT: '/loans/loanreassignment',
    STANDING_INSTRUCTIONS: '/standinginstructions'
  },

  // --- Products ---
  PRODUCTS: {
    LOAN_PRODUCTS: '/loanproducts',
    LOAN_PRODUCT: (id: number) => `/loanproducts/${id}`,
    SAVINGS_PRODUCTS: '/savingsproducts',
    SAVINGS_PRODUCT: (id: number) => `/savingsproducts/${id}`,
    SHARE_PRODUCTS: '/products/share',
    SHARE_PRODUCT: (id: number) => `/products/share/${id}`,
    CHARGES: '/charges',
    CHARGE: (id: number) => `/charges/${id}`,
    FIXED_DEPOSIT_PRODUCTS: '/fixeddepositproducts',
    RECURRING_DEPOSIT_PRODUCTS: '/recurringdepositproducts',
    TAX_COMPONENTS: '/taxes/component',
    TAX_GROUPS: '/taxes/group',
    FLOATING_RATES: '/floatingrates',
    PRODUCT_MIX: '/loanproducts/{productId}/productmix'
  },

  // --- System ---
  SYSTEM: {
    CODES: '/codes',
    CODE: (id: number) => `/codes/${id}`,
    CODE_VALUES: (codeId: number) => `/codes/${codeId}/codevalues`,
    DATATABLES: '/datatables',
    DATATABLE: (name: string) => `/datatables/${name}`,
    DATATABLE_DATA: (table: string, appTableId: number) => `/datatables/${table}/${appTableId}`,
    HOOKS: '/hooks',
    HOOK: (id: number) => `/hooks/${id}`,
    ROLES: '/roles',
    ROLE: (id: number) => `/roles/${id}`,
    PERMISSIONS: '/permissions',
    CONFIGURATIONS: '/configurations',
    SURVEYS: '/surveys',
    AUDIT: '/audits',
    AUDIT_BY_ID: (id: number) => `/audits/${id}`,
    SCHEDULER_JOBS: '/jobs',
    SCHEDULER_JOB: (id: number) => `/jobs/${id}`,
    REPORTS: '/reports',
    REPORT: (id: number) => `/reports/${id}`,
    REPORT_RUN: (id: number) => `/runreports/${id}`,
    EXTERNAL_SERVICES: '/externalservice'
  },

  // --- Users ---
  USERS: {
    BASE: '/users',
    BY_ID: (id: number) => `/users/${id}`,
    TEMPLATE: '/users/template'
  },

  // --- Search ---
  SEARCH: '/search',
  SEARCH_ADVANCED: '/search/advance',

  // --- Notifications ---
  NOTIFICATIONS: '/notifications',

  // --- Account Transfers ---
  ACCOUNT_TRANSFERS: {
    BASE: '/accounttransfers',
    TEMPLATE: '/accounttransfers/template',
    STANDING_INSTRUCTIONS: '/standinginstructions'
  },

  // --- Batch API ---
  BATCH: '/batches',

  // --- Run Reports ---
  RUN_REPORT: (reportName: string) => `/runreports/${encodeURIComponent(reportName)}`,

  // --- Maker Checker ---
  MAKER_CHECKER: '/makercheckers',
  MAKER_CHECKER_BY_ID: (id: number) => `/makercheckers/${id}`
} as const;
```

---

## 9. Environment Variables

Complete list of environment variables with descriptions, mapping from the Angular `environment.ts` runtime config.

```bash
# .env.example

# =============================================================================
# Fineract API Connection
# =============================================================================

# Server-only: Used by Server Components and API routes.
# NOT exposed to the browser. Maps to Angular's environment.baseApiUrl + apiProvider + apiVersion.
FINERACT_API_URL=https://demo.mifos.community/fineract-provider/api/v1

# Client-only: Used by browser-side Axios. Exposed to the browser.
# Same value as FINERACT_API_URL but with NEXT_PUBLIC_ prefix.
NEXT_PUBLIC_FINERACT_API_URL=https://demo.mifos.community/fineract-provider/api/v1

# Tenant identifier. Maps to Angular's environment.fineractPlatformTenantId.
FINERACT_TENANT_ID=default
NEXT_PUBLIC_TENANT_ID=default

# Comma-separated list of available tenant identifiers for the tenant selector.
# Maps to Angular's environment.fineractPlatformTenantIds.
NEXT_PUBLIC_TENANT_IDS=default

# Comma-separated list of available Fineract server URLs for the server selector.
# Maps to Angular's environment.baseApiUrls.
NEXT_PUBLIC_FINERACT_API_URLS=https://demo.mifos.community,https://localhost:8443

# Whether to show the server switch UI in settings. Maps to environment.allowServerSwitch.
NEXT_PUBLIC_ALLOW_SERVER_SWITCH=true

# =============================================================================
# Authentication
# =============================================================================

# NextAuth.js secret for JWT encryption. Required in production.
NEXTAUTH_SECRET=your-secret-here

# NextAuth.js base URL. Set to the deployment URL.
NEXTAUTH_URL=http://localhost:3000

# Auth mode: "basic", "oauth2", or "oidc". Maps to Angular's getActiveAuthMode().
NEXT_PUBLIC_AUTH_MODE=basic

# --- OAuth2 (Fineract OAuth) ---
# Maps to Angular's environment.oauth.*
OAUTH_SERVER_URL=
OAUTH_LOGOUT_URL=
OAUTH_APP_ID=
OAUTH_AUTHORIZE_URL=
OAUTH_TOKEN_URL=
OAUTH_REDIRECT_URI=
OAUTH_SCOPE=

# --- OIDC (Zitadel) ---
# Maps to Angular's environment.OIDC.*
OIDC_BASE_URL=
OIDC_CLIENT_ID=
OIDC_API_URL=
OIDC_FRONTEND_URL=

# =============================================================================
# Internationalization
# =============================================================================

# Default language code. Maps to Angular's environment.defaultLanguage.
NEXT_PUBLIC_DEFAULT_LANGUAGE=en-US

# Comma-separated supported language codes.
# Maps to Angular's environment.supportedLanguages.
NEXT_PUBLIC_SUPPORTED_LANGUAGES=cs-CS,de-DE,en-US,es-MX,fr-FR,it-IT,ko-KO,lt-LT,lv-LV,ne-NE,pt-PT,sw-SW

# =============================================================================
# Date and Number Formatting
# =============================================================================

# Default date format (date-fns format string). Maps to environment.defaultFormatDate.
NEXT_PUBLIC_DEFAULT_DATE_FORMAT=dd MMMM yyyy

# Default datetime format. Maps to environment.defaultFormatDatetime.
NEXT_PUBLIC_DEFAULT_DATETIME_FORMAT=dd MMMM yyyy HH:mm:ss

# =============================================================================
# Feature Flags
# =============================================================================

# Show backend info in the UI. Maps to environment.displayBackEndInfo.
NEXT_PUBLIC_DISPLAY_BACKEND_INFO=true

# Show tenant selector in settings. Maps to environment.displayTenantSelector.
NEXT_PUBLIC_DISPLAY_TENANT_SELECTOR=true

# Production mode (minimal hero, only branding). Maps to environment.productionMode.
NEXT_PUBLIC_PRODUCTION_MODE=false

# Enable RBAC for menus/buttons. Maps to environment.productionModeEnableRBAC.
NEXT_PUBLIC_ENABLE_RBAC=false

# Remember Me functionality. Maps to environment.enableRememberMe.
NEXT_PUBLIC_ENABLE_REMEMBER_ME=false

# HTTP cache enabled. Maps to environment.httpCacheEnabled.
# Note: In the Next.js app, this is largely replaced by TanStack Query caching.
# This flag controls whether server-side fetch caching is enabled.
NEXT_PUBLIC_HTTP_CACHE_ENABLED=false

# Hide client data (compliance masking). Maps to environment.complianceHideClientData.
NEXT_PUBLIC_COMPLIANCE_HIDE_CLIENT_DATA=false

# =============================================================================
# Branding
# =============================================================================

# Tenant logo URL for light theme. Maps to environment.tenantLogoUrl.
NEXT_PUBLIC_TENANT_LOGO_URL=/images/default_home.png

# Tenant logo URL for dark theme. Maps to environment.tenantLogoUrlDark.
NEXT_PUBLIC_TENANT_LOGO_URL_DARK=/images/white-mifos.png

# Documentation base URL. Maps to environment.documentationBaseUrl.
NEXT_PUBLIC_DOCUMENTATION_BASE_URL=https://mifosforge.jira.com/wiki

# =============================================================================
# Session and Polling
# =============================================================================

# Session idle timeout in milliseconds. Maps to environment.session.timeout.idleTimeout.
NEXT_PUBLIC_SESSION_IDLE_TIMEOUT=300000

# Notification polling interval in seconds. Maps to environment.waitTimeForNotifications.
NEXT_PUBLIC_NOTIFICATION_POLL_INTERVAL=60

# COB catch-up polling interval in seconds. Maps to environment.waitTimeForCOBCatchUp.
NEXT_PUBLIC_COB_CATCHUP_POLL_INTERVAL=30

# =============================================================================
# Interbank Transfers
# =============================================================================

NEXT_PUBLIC_INTERBANK_TRANSFERS_ENABLED=true
NEXT_PUBLIC_INTERBANK_TRANSFERS_API_URL=https://apis.mifos.community
NEXT_PUBLIC_INTERBANK_TRANSFERS_API_PROVIDER=/vnext1
NEXT_PUBLIC_INTERBANK_TRANSFERS_API_VERSION=/v1.0

# =============================================================================
# Remittance Module
# =============================================================================

NEXT_PUBLIC_REMITTANCE_ENABLED=false
NEXT_PUBLIC_REMITTANCE_API_URL=
NEXT_PUBLIC_REMITTANCE_API_PROVIDER=
NEXT_PUBLIC_REMITTANCE_API_VERSION=
# Server-only (secrets):
REMITTANCE_API_HEADER=
REMITTANCE_API_KEY=

# =============================================================================
# External National ID System
# =============================================================================

NEXT_PUBLIC_EXTERNAL_NATIONAL_ID_ENABLED=false
NEXT_PUBLIC_EXTERNAL_NATIONAL_ID_URL=
NEXT_PUBLIC_EXTERNAL_NATIONAL_ID_REGEX=
# Server-only (secrets):
EXTERNAL_NATIONAL_ID_API_HEADER=
EXTERNAL_NATIONAL_ID_API_KEY=

# =============================================================================
# Password Policy
# =============================================================================

NEXT_PUBLIC_MIN_PASSWORD_LENGTH=8
NEXT_PUBLIC_PASSWORD_REGEX=^(?!.*(.)\\1)(?!.*\\s)(?=.*\\d)(?=.*[a-z])(?=.*[A-Z])(?=.*[^\\w\\s]).{8,50}$
```

### Server-Only vs Client-Exposed Variables

| Prefix                               | Accessible From                           | Use For                                       |
| ------------------------------------ | ----------------------------------------- | --------------------------------------------- |
| No prefix (e.g., `FINERACT_API_URL`) | Server Components, API routes, middleware | API secrets, internal URLs, API keys          |
| `NEXT_PUBLIC_` prefix                | Both server and client                    | Feature flags, branding, non-sensitive config |

**Security note**: API keys (`REMITTANCE_API_KEY`, `EXTERNAL_NATIONAL_ID_API_KEY`) are server-only variables without the `NEXT_PUBLIC_` prefix. They are never exposed to the browser. In the Angular app, these were exposed via `window.env` -- the migration is an opportunity to fix this security gap.
