# NEXTJS-004: State Management Design

> Migration guide for state management from Angular's RxJS-based services and Reactive Forms to React state patterns using TanStack Query, Zustand, React Hook Form, and URL state.

---

## Table of Contents

1. [State Categories](#1-state-categories)
2. [TanStack Query Patterns](#2-tanstack-query-patterns)
3. [Zustand Stores](#3-zustand-stores)
4. [Form State with React Hook Form + Zod](#4-form-state-with-react-hook-form--zod)
5. [Migration Mapping](#5-migration-mapping)

---

## 1. State Categories

The Angular app conflates all state into RxJS Observables and BehaviorSubjects. The Next.js architecture separates state into four distinct categories, each managed by the most appropriate tool.

| State Category   | Description                              | Tool                       | Angular Equivalent                                                                 |
| ---------------- | ---------------------------------------- | -------------------------- | ---------------------------------------------------------------------------------- |
| **Server State** | Data from the Fineract API               | TanStack Query             | RxJS Observables from `HttpClient.get()`, `BehaviorSubject` in services            |
| **UI State**     | Theme, sidebar, modals, alerts           | Zustand                    | BehaviorSubjects in services (`AlertService`, `SettingsService`, `ThemingService`) |
| **Form State**   | Input values, validation, dirty/pristine | React Hook Form + Zod      | Angular Reactive Forms (`FormGroup`, `FormControl`, `Validators`)                  |
| **URL State**    | Filters, pagination, search, sort        | `nuqs` / `useSearchParams` | Angular `ActivatedRoute.queryParams`                                               |

### Why This Separation Matters

In the Angular app, a single service might use a `BehaviorSubject` to hold both server data and UI state, subscribe to route params for URL state, and inject `FormBuilder` for form state. This creates implicit coupling. The Next.js architecture makes each state category explicit and independently testable.

```
Angular:                              Next.js:
┌─────────────────────┐              ┌─────────────────────┐
│  SomeComponent.ts   │              │  page.tsx            │
│  ┌───────────────┐  │              │  ┌───────────────┐  │
│  │ this.route    │──┼─ URL state   │  │ useQueryStates │──┼─ URL state (nuqs)
│  │ .queryParams  │  │              │  └───────────────┘  │
│  └───────────────┘  │              │  ┌───────────────┐  │
│  ┌───────────────┐  │              │  │ useClients()  │──┼─ Server state (TanStack Query)
│  │ this.service  │──┼─ All state   │  └───────────────┘  │
│  │ .getData()    │  │  mixed       │  ┌───────────────┐  │
│  └───────────────┘  │              │  │ useSidebar()  │──┼─ UI state (Zustand)
│  ┌───────────────┐  │              │  └───────────────┘  │
│  │ this.form     │──┼─ Form state  │  ┌───────────────┐  │
│  │ .get('name')  │  │              │  │ useForm()     │──┼─ Form state (RHF + Zod)
│  └───────────────┘  │              │  └───────────────┘  │
└─────────────────────┘              └─────────────────────┘
```

---

## 2. TanStack Query Patterns

TanStack Query replaces the Angular pattern of subscribing to `HttpClient` observables in `ngOnInit` and unsubscribing in `ngOnDestroy`. It handles caching, deduplication, background refetching, and garbage collection automatically.

### Query Key Conventions

Query keys are structured arrays that enable precise cache invalidation. The convention follows a hierarchical pattern:

```typescript
// Pattern: [entity, ...qualifiers]

// Entity lists
['clients'][('clients', 'list')][('clients', 'list', { page: 0, limit: 50, search: '' })][ // All client queries // Client list queries // Specific list query
  // Single entities
  ('clients', 'detail')
][('clients', 'detail', 42)][ // All client detail queries // Specific client (id=42)
  // Nested resources
  ('clients', 'detail', 42, 'accounts')
][('clients', 'detail', 42, 'loans')][('clients', 'detail', 42, 'documents')][ // Client 42's accounts // Client 42's loans // Client 42's documents
  // Reference data
  'offices'
]['staff'][('products', 'loan')][('products', 'savings')][('codeValues', 'Gender')]; // All offices // All staff // Loan products // Savings products // Code values for 'Gender' code
```

### Query Key Factory Pattern

Each service file exports a key factory for type-safe, consistent key generation:

```typescript
// lib/services/loans.ts

export const loanKeys = {
  all: ['loans'] as const,
  lists: () => [
      ...loanKeys.all,
      'list'
    ] as const,
  list: (params: Record<string, unknown>) => [
      ...loanKeys.lists(),
      params
    ] as const,
  details: () => [
      ...loanKeys.all,
      'detail'
    ] as const,
  detail: (id: number) => [
      ...loanKeys.details(),
      id
    ] as const,
  transactions: (id: number) => [
      ...loanKeys.detail(id),
      'transactions'
    ] as const,
  charges: (id: number) => [
      ...loanKeys.detail(id),
      'charges'
    ] as const,
  documents: (id: number) => [
      ...loanKeys.detail(id),
      'documents'
    ] as const,
  notes: (id: number) => [
      ...loanKeys.detail(id),
      'notes'
    ] as const,
  guarantors: (id: number) => [
      ...loanKeys.detail(id),
      'guarantors'
    ] as const,
  collaterals: (id: number) => [
      ...loanKeys.detail(id),
      'collaterals'
    ] as const,
  schedule: (id: number) => [
      ...loanKeys.detail(id),
      'schedule'
    ] as const,
  template: (params?: Record<string, unknown>) => [
      ...loanKeys.all,
      'template',
      params
    ] as const
};
```

### Custom Hook Pattern per Service

Each Angular service file maps to a single file in `lib/services/` containing all TanStack Query hooks for that domain.

```typescript
// lib/services/loans.ts
'use client';

import { useQuery, useMutation, useQueryClient, useSuspenseQuery } from '@tanstack/react-query';
import fineractApi from '@/lib/api/client';
import { API } from '@/lib/api/endpoints';
import type { LoanListResponse, LoanResponse, LoanTemplate, CreateLoanPayload, LoanTransaction, LoanScheduleResponse } from '@/lib/api/types/loan';

// --- Key Factory (see above) ---
export const loanKeys = {
  /* ... */
};

// --- List Queries ---

/**
 * Fetch paginated loan list.
 *
 * Angular equivalent:
 *   this.loansService.getLoans(offset, limit, orderBy, sortOrder)
 *     .subscribe(loans => this.dataSource.data = loans);
 */
export function useLoans(params: { page: number; limit: number; orderBy?: string; sortOrder?: 'ASC' | 'DESC'; accountNo?: string; externalId?: string; sqlSearch?: string }) {
  return useQuery({
    queryKey: loanKeys.list(params),
    queryFn: async () => {
      const { data } = await fineractApi.get<LoanListResponse>(API.LOANS.BASE, {
        params: {
          offset: params.page * params.limit,
          limit: params.limit,
          orderBy: params.orderBy,
          sortOrder: params.sortOrder,
          sqlSearch: params.sqlSearch
        }
      });
      return data;
    },
    staleTime: 30_000,
    placeholderData: (previousData) => previousData // Keep previous data while fetching next page
  });
}

// --- Detail Queries ---

/**
 * Fetch single loan by ID.
 *
 * Angular equivalent:
 *   this.route.data.subscribe(({ loanDetailsData }) => { ... });
 *   (data was resolved by a route resolver calling loansService.getLoan(id))
 */
export function useLoan(id: number) {
  return useQuery({
    queryKey: loanKeys.detail(id),
    queryFn: async () => {
      const { data } = await fineractApi.get<LoanResponse>(`${API.LOANS.BASE}/${id}`, {
        params: {
          associations: 'all',
          exclude: 'guarantors,futureSchedule'
        }
      });
      return data;
    },
    staleTime: 0, // Always refetch on focus
    enabled: id > 0
  });
}

/**
 * Suspense-compatible loan query for use in Server Component-like patterns.
 * Throws a promise while loading (caught by nearest <Suspense> boundary).
 */
export function useLoanSuspense(id: number) {
  return useSuspenseQuery({
    queryKey: loanKeys.detail(id),
    queryFn: async () => {
      const { data } = await fineractApi.get<LoanResponse>(`${API.LOANS.BASE}/${id}`, {
        params: { associations: 'all' }
      });
      return data;
    }
  });
}

// --- Template Queries ---

/**
 * Fetch loan creation template (dropdown options).
 *
 * Angular equivalent:
 *   this.loansService.getLoansAccountTemplateResource(clientId)
 *     .subscribe(template => this.loanProducts = template.productOptions);
 */
export function useLoanTemplate(params?: { clientId?: number; productId?: number }) {
  return useQuery({
    queryKey: loanKeys.template(params),
    queryFn: async () => {
      const { data } = await fineractApi.get<LoanTemplate>(API.LOANS.TEMPLATE, {
        params: { clientId: params?.clientId, productId: params?.productId, templateType: 'individual' }
      });
      return data;
    },
    staleTime: 5 * 60 * 1000, // 5 minutes (template data is stable)
    enabled: !!params?.clientId
  });
}

// --- Nested Resource Queries ---

export function useLoanTransactions(loanId: number) {
  return useQuery({
    queryKey: loanKeys.transactions(loanId),
    queryFn: async () => {
      const { data } = await fineractApi.get<LoanTransaction[]>(API.LOANS.TRANSACTIONS(loanId));
      return data;
    },
    enabled: loanId > 0
  });
}

export function useLoanSchedule(loanId: number) {
  return useQuery({
    queryKey: loanKeys.schedule(loanId),
    queryFn: async () => {
      const { data } = await fineractApi.get<LoanScheduleResponse>(`${API.LOANS.BASE}/${loanId}`, { params: { associations: 'repaymentSchedule' } });
      return data;
    },
    enabled: loanId > 0
  });
}

// --- Mutations ---

/**
 * Create a new loan application.
 *
 * Angular equivalent:
 *   this.loansService.createLoan(loanData).subscribe(() => this.router.navigate(['/loans', id]));
 */
export function useCreateLoan() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (payload: CreateLoanPayload) => {
      const { data } = await fineractApi.post(API.LOANS.BASE, payload);
      return data;
    },
    onSuccess: (_data, _variables) => {
      // Invalidate all loan lists (new loan appears in list)
      queryClient.invalidateQueries({ queryKey: loanKeys.lists() });
    }
  });
}

/**
 * Approve a loan.
 *
 * Angular equivalent:
 *   this.loansService.loanActionButtons(loanId, { ... }, 'approve')
 *     .subscribe(() => { ... });
 */
export function useApproveLoan() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, ...payload }: { id: number; approvedOnDate: string; locale: string; dateFormat: string; note?: string }) => {
      const { data } = await fineractApi.post(`${API.LOANS.BASE}/${id}?command=approve`, payload);
      return data;
    },
    // Optimistic update: immediately show loan as approved
    onMutate: async ({ id }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: loanKeys.detail(id) });

      // Snapshot the current loan data
      const previousLoan = queryClient.getQueryData<LoanResponse>(loanKeys.detail(id));

      // Optimistically update the status
      if (previousLoan) {
        queryClient.setQueryData<LoanResponse>(loanKeys.detail(id), {
          ...previousLoan,
          status: { ...previousLoan.status, value: 'Approved', code: 'loanStatusType.approved' }
        });
      }

      return { previousLoan };
    },
    onError: (_error, { id }, context) => {
      // Roll back to previous data on error
      if (context?.previousLoan) {
        queryClient.setQueryData(loanKeys.detail(id), context.previousLoan);
      }
    },
    onSettled: (_data, _error, { id }) => {
      // Always refetch after mutation to ensure consistency
      queryClient.invalidateQueries({ queryKey: loanKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: loanKeys.lists() });
    }
  });
}

/**
 * Disburse a loan.
 */
export function useDisburseLoan() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, ...payload }: { id: number; actualDisbursementDate: string; locale: string; dateFormat: string; transactionAmount?: number; note?: string }) => {
      const { data } = await fineractApi.post(`${API.LOANS.BASE}/${id}?command=disburse`, payload);
      return data;
    },
    onSuccess: (_data, { id }) => {
      queryClient.invalidateQueries({ queryKey: loanKeys.detail(id) });
      queryClient.invalidateQueries({ queryKey: loanKeys.lists() });
      queryClient.invalidateQueries({ queryKey: loanKeys.transactions(id) });
    }
  });
}

/**
 * Make a loan repayment.
 */
export function useLoanRepayment() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ loanId, ...payload }: { loanId: number; transactionDate: string; transactionAmount: number; locale: string; dateFormat: string; paymentTypeId?: number; note?: string }) => {
      const { data } = await fineractApi.post(`${API.LOANS.TRANSACTIONS(loanId)}?command=repayment`, payload);
      return data;
    },
    onSuccess: (_data, { loanId }) => {
      queryClient.invalidateQueries({ queryKey: loanKeys.detail(loanId) });
      queryClient.invalidateQueries({ queryKey: loanKeys.transactions(loanId) });
      queryClient.invalidateQueries({ queryKey: loanKeys.schedule(loanId) });
    }
  });
}
```

### Prefetching in Layouts

Layouts can prefetch data for child routes, so the data is already cached when the user navigates:

```typescript
// app/(dashboard)/clients/[id]/layout.tsx
'use client';

import { useQueryClient } from '@tanstack/react-query';
import { useEffect } from 'react';
import { clientKeys } from '@/lib/services/clients';
import fineractApi from '@/lib/api/client';

export default function ClientDetailLayout({
  children,
  params,
}: {
  children: React.ReactNode;
  params: { id: string };
}) {
  const queryClient = useQueryClient();
  const clientId = Number(params.id);

  // Prefetch related data when the client detail layout mounts
  useEffect(() => {
    queryClient.prefetchQuery({
      queryKey: clientKeys.accounts(clientId),
      queryFn: () => fineractApi.get(`/clients/${clientId}/accounts`).then(r => r.data),
    });
    queryClient.prefetchQuery({
      queryKey: clientKeys.documents(clientId),
      queryFn: () => fineractApi.get(`/clients/${clientId}/documents`).then(r => r.data),
    });
  }, [clientId, queryClient]);

  return <>{children}</>;
}
```

### Reference Data Hooks

Reference data (offices, staff, products, code values) is fetched with long stale times since it rarely changes:

```typescript
// lib/services/reference-data.ts
'use client';

import { useQuery } from '@tanstack/react-query';
import fineractApi from '@/lib/api/client';

const REFERENCE_STALE_TIME = 5 * 60 * 1000; // 5 minutes
const REFERENCE_GC_TIME = 30 * 60 * 1000; // 30 minutes

export function useOffices() {
  return useQuery({
    queryKey: ['offices'],
    queryFn: async () => {
      const { data } = await fineractApi.get('/offices');
      return data;
    },
    staleTime: REFERENCE_STALE_TIME,
    gcTime: REFERENCE_GC_TIME
  });
}

export function useStaff(params?: { officeId?: number; status?: string }) {
  return useQuery({
    queryKey: [
      'staff',
      params
    ],
    queryFn: async () => {
      const { data } = await fineractApi.get('/staff', { params });
      return data;
    },
    staleTime: REFERENCE_STALE_TIME,
    gcTime: REFERENCE_GC_TIME,
    enabled: params?.officeId != null || params === undefined
  });
}

export function useCodeValues(codeName: string) {
  return useQuery({
    queryKey: [
      'codeValues',
      codeName
    ],
    queryFn: async () => {
      // First get the code to find its ID, then get its values
      const { data: codes } = await fineractApi.get('/codes');
      const code = codes.find((c: any) => c.name === codeName);
      if (!code) return [];
      const { data: values } = await fineractApi.get(`/codes/${code.id}/codevalues`);
      return values;
    },
    staleTime: 10 * 60 * 1000, // 10 minutes -- code values almost never change
    gcTime: REFERENCE_GC_TIME
  });
}

export function useLoanProducts() {
  return useQuery({
    queryKey: [
      'products',
      'loan'
    ],
    queryFn: async () => {
      const { data } = await fineractApi.get('/loanproducts');
      return data;
    },
    staleTime: REFERENCE_STALE_TIME,
    gcTime: REFERENCE_GC_TIME
  });
}

export function useSavingsProducts() {
  return useQuery({
    queryKey: [
      'products',
      'savings'
    ],
    queryFn: async () => {
      const { data } = await fineractApi.get('/savingsproducts');
      return data;
    },
    staleTime: REFERENCE_STALE_TIME,
    gcTime: REFERENCE_GC_TIME
  });
}

export function usePaymentTypes() {
  return useQuery({
    queryKey: ['paymentTypes'],
    queryFn: async () => {
      const { data } = await fineractApi.get('/paymenttypes');
      return data;
    },
    staleTime: REFERENCE_STALE_TIME,
    gcTime: REFERENCE_GC_TIME
  });
}
```

---

## 3. Zustand Stores

Zustand replaces Angular services that hold UI state in `BehaviorSubject`s. Each store is a single function that returns the state shape and actions.

### Theme Store

Replaces `SettingsService.themeDarkEnabled` and the `ThemingService` pattern.

```typescript
// lib/stores/theme-store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface ThemeState {
  isDark: boolean;
  toggleTheme: () => void;
  setDark: (isDark: boolean) => void;
}

/**
 * Theme store.
 *
 * Persisted to localStorage (replaces Angular's mifosXThemeDarkEnabled localStorage key).
 * Works in conjunction with next-themes for SSR-safe theme switching.
 */
export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      isDark: false,
      toggleTheme: () => set((state) => ({ isDark: !state.isDark })),
      setDark: (isDark: boolean) => set({ isDark })
    }),
    {
      name: 'msacco-theme' // localStorage key
    }
  )
);
```

### Settings Store

Replaces `SettingsService` for user preferences (language, date format, server URL, tenant).

```typescript
// lib/stores/settings-store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface Language {
  name: string;
  code: string;
}

interface SettingsState {
  // Language
  language: Language;
  setLanguage: (language: Language) => void;

  // Date format
  dateFormat: string;
  setDateFormat: (format: string) => void;
  datetimeFormat: string;
  setDatetimeFormat: (format: string) => void;

  // Server
  serverUrl: string;
  setServerUrl: (url: string) => void;
  servers: string[];
  setServers: (servers: string[]) => void;

  // Tenant
  tenantId: string;
  setTenantId: (id: string) => void;
  tenantIds: string[];
  setTenantIds: (ids: string[]) => void;

  // Decimals
  decimalsToDisplay: string;
  setDecimalsToDisplay: (decimals: string) => void;

  // Business date
  businessDate: string | null;
  setBusinessDate: (date: string | null) => void;
  businessDateEnabled: boolean;
  setBusinessDateEnabled: (enabled: boolean) => void;
}

/**
 * Settings store.
 *
 * Replaces Angular SettingsService (src/app/settings/settings.service.ts).
 * Persisted to localStorage with the same keys the Angular app uses,
 * enabling a smooth transition where user settings are preserved.
 */
export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      language: { name: process.env.NEXT_PUBLIC_DEFAULT_LANGUAGE ?? 'en-US', code: 'en' },
      setLanguage: (language) => set({ language }),

      dateFormat: process.env.NEXT_PUBLIC_DEFAULT_DATE_FORMAT ?? 'dd MMMM yyyy',
      setDateFormat: (dateFormat) => set({ dateFormat }),
      datetimeFormat: process.env.NEXT_PUBLIC_DEFAULT_DATETIME_FORMAT ?? 'dd MMMM yyyy HH:mm:ss',
      setDatetimeFormat: (datetimeFormat) => set({ datetimeFormat }),

      serverUrl: process.env.NEXT_PUBLIC_FINERACT_API_URL ?? '',
      setServerUrl: (serverUrl) => set({ serverUrl }),
      servers: (process.env.NEXT_PUBLIC_FINERACT_API_URLS ?? '').split(',').filter(Boolean),
      setServers: (servers) => set({ servers }),

      tenantId: process.env.NEXT_PUBLIC_TENANT_ID ?? 'default',
      setTenantId: (tenantId) => set({ tenantId }),
      tenantIds: (process.env.NEXT_PUBLIC_TENANT_IDS ?? 'default').split(',').filter(Boolean),
      setTenantIds: (tenantIds) => set({ tenantIds }),

      decimalsToDisplay: '2',
      setDecimalsToDisplay: (decimalsToDisplay) => set({ decimalsToDisplay }),

      businessDate: null,
      setBusinessDate: (businessDate) => set({ businessDate }),
      businessDateEnabled: false,
      setBusinessDateEnabled: (businessDateEnabled) => set({ businessDateEnabled })
    }),
    {
      name: 'msacco-settings'
    }
  )
);
```

### Sidebar Store

Replaces the sidebar open/collapsed state managed in `SidenavComponent`.

```typescript
// lib/stores/sidebar-store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface SidebarState {
  isCollapsed: boolean;
  isOpen: boolean; // For mobile: whether the sidebar drawer is open
  toggle: () => void;
  setCollapsed: (collapsed: boolean) => void;
  setOpen: (open: boolean) => void;
}

export const useSidebarStore = create<SidebarState>()(
  persist(
    (set) => ({
      isCollapsed: false,
      isOpen: false,
      toggle: () => set((state) => ({ isCollapsed: !state.isCollapsed })),
      setCollapsed: (isCollapsed) => set({ isCollapsed }),
      setOpen: (isOpen) => set({ isOpen })
    }),
    {
      name: 'msacco-sidebar'
    }
  )
);
```

### Alert Store

Replaces `AlertService` and `Alert` model. Used by the response interceptor to show error/success notifications.

```typescript
// lib/stores/alert-store.ts
import { create } from 'zustand';

interface Alert {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  title: string;
  message: string;
  timestamp: number;
}

interface AlertState {
  alerts: Alert[];
  addAlert: (alert: Omit<Alert, 'id' | 'timestamp'>) => void;
  removeAlert: (id: string) => void;
  clearAlerts: () => void;
}

/**
 * Alert store.
 *
 * Replaces Angular AlertService (src/app/core/alert/alert.service.ts).
 *
 * Alerts are stored in an array and rendered by a global <Toaster> component.
 * The response interceptor (lib/api/client.ts) calls addAlert() on API errors.
 *
 * Note: This store is intentionally NOT persisted. Alerts are ephemeral.
 */
export const useAlertStore = create<AlertState>((set) => ({
  alerts: [],

  addAlert: (alert) => {
    const newAlert: Alert = {
      ...alert,
      id: `${Date.now()}-${Math.random().toString(36).slice(2, 9)}`,
      timestamp: Date.now()
    };
    set((state) => ({
      alerts: [
        ...state.alerts,
        newAlert
      ]
    }));

    // Auto-remove after 5 seconds
    setTimeout(() => {
      set((state) => ({
        alerts: state.alerts.filter((a) => a.id !== newAlert.id)
      }));
    }, 5000);
  },

  removeAlert: (id) =>
    set((state) => ({
      alerts: state.alerts.filter((a) => a.id !== id)
    })),

  clearAlerts: () => set({ alerts: [] })
}));
```

### URL State with nuqs

URL state replaces Angular's `ActivatedRoute.queryParams` for filters, pagination, and search. The `nuqs` library provides type-safe URL search parameter state management.

```typescript
// app/(dashboard)/clients/page.tsx
'use client';

import { useQueryStates, parseAsInteger, parseAsString } from 'nuqs';
import { useClients } from '@/lib/services/clients';
import { DataTable } from '@/components/shared/data-table/data-table';

export default function ClientsPage() {
  // URL state: /clients?page=0&limit=50&search=john&status=active
  const [params, setParams] = useQueryStates({
    page: parseAsInteger.withDefault(0),
    limit: parseAsInteger.withDefault(50),
    search: parseAsString.withDefault(''),
    status: parseAsString.withDefault(''),
    orderBy: parseAsString.withDefault(''),
    sortOrder: parseAsString.withDefault(''),
  });

  // Server state: TanStack Query fetches data based on URL params
  const { data, isLoading, isError } = useClients({
    page: params.page,
    limit: params.limit,
    search: params.search || undefined,
  });

  return (
    <div>
      <h1>Clients</h1>
      <DataTable
        data={data?.pageItems ?? []}
        totalCount={data?.totalFilteredRecords ?? 0}
        page={params.page}
        pageSize={params.limit}
        isLoading={isLoading}
        onPageChange={(page) => setParams({ page })}
        onPageSizeChange={(limit) => setParams({ limit, page: 0 })}
        onSearchChange={(search) => setParams({ search, page: 0 })}
      />
    </div>
  );
}
```

---

## 4. Form State with React Hook Form + Zod

React Hook Form (RHF) + Zod replaces Angular Reactive Forms (`FormGroup`, `FormControl`, `FormArray`, `Validators`). The key advantage is schema-first validation: the Zod schema defines both the TypeScript type and the runtime validation rules.

### Schema-First Validation

```typescript
// lib/validators/client.ts
import { z } from 'zod';

/**
 * Client creation schema.
 *
 * Replaces the Angular FormGroup from clients/clients-view/create-client/create-client.component.ts
 * which used Validators.required, Validators.pattern, etc.
 *
 * Zod provides:
 * - Runtime validation (replaces Validators)
 * - TypeScript type inference (replaces manual interface definition)
 * - Composable schemas (replaces FormGroup nesting)
 */

// Sub-schemas for reuse
const nameSchema = z.string().min(1, 'This field is required').max(100, 'Maximum 100 characters');

const phoneSchema = z
  .string()
  .regex(/^\+?[0-9]{10,15}$/, 'Invalid phone number format')
  .optional()
  .or(z.literal(''));

const dateSchema = z
  .string()
  .regex(/^\d{4}-\d{2}-\d{2}$/, 'Invalid date format (YYYY-MM-DD)')
  .optional()
  .or(z.literal(''));

// Main client schema
export const clientSchema = z.object({
  // Personal information
  officeId: z.number({ required_error: 'Office is required' }).positive(),
  firstname: nameSchema,
  middlename: z.string().max(100).optional(),
  lastname: nameSchema,
  fullname: z.string().optional(), // For entity clients
  mobileNo: phoneSchema,
  emailAddress: z.string().email('Invalid email address').optional().or(z.literal('')),
  dateOfBirth: dateSchema,
  genderId: z.number().optional(),
  clientTypeId: z.number().optional(),
  clientClassificationId: z.number().optional(),
  legalFormId: z.number().optional(),

  // Status
  active: z.boolean().default(false),
  activationDate: dateSchema,
  submittedOnDate: z.string().optional(),

  // Staff assignment
  staffId: z.number().optional(),

  // External ID
  externalId: z.string().max(100).optional(),

  // Address
  address: z
    .array(
      z.object({
        addressTypeId: z.number().optional(),
        street: z.string().optional(),
        addressLine1: z.string().optional(),
        addressLine2: z.string().optional(),
        addressLine3: z.string().optional(),
        city: z.string().optional(),
        stateProvinceId: z.number().optional(),
        countryId: z.number().optional(),
        postalCode: z.string().optional()
      })
    )
    .optional(),

  // Family members
  familyMembers: z
    .array(
      z.object({
        firstName: z.string().min(1),
        middleName: z.string().optional(),
        lastName: z.string().min(1),
        qualification: z.string().optional(),
        relationshipId: z.number(),
        genderId: z.number(),
        dateOfBirth: dateSchema,
        profession: z.string().optional(),
        mobileNumber: phoneSchema
      })
    )
    .optional(),

  // Format params (required by Fineract)
  locale: z.string().default('en'),
  dateFormat: z.string().default('dd MMMM yyyy')
});

// Inferred TypeScript type (replaces manual interface)
export type CreateClientPayload = z.infer<typeof clientSchema>;

// Partial schema for editing (all fields optional except ID)
export const updateClientSchema = clientSchema.partial().extend({
  id: z.number()
});

export type UpdateClientPayload = z.infer<typeof updateClientSchema>;
```

### Multi-Step Form Pattern (Wizards)

The Angular app uses `MatStepper` for multi-step forms (client creation, loan application). The Next.js replacement persists form state across steps using RHF's context.

```typescript
// components/shared/wizard/wizard.tsx
'use client';

import { useState, type ReactNode, createContext, useContext } from 'react';
import { useFormContext, FormProvider, type UseFormReturn, type FieldValues } from 'react-hook-form';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils/cn';

interface WizardContextValue {
  currentStep: number;
  totalSteps: number;
  nextStep: () => void;
  prevStep: () => void;
  goToStep: (step: number) => void;
  isFirstStep: boolean;
  isLastStep: boolean;
}

const WizardContext = createContext<WizardContextValue | null>(null);

export function useWizard() {
  const context = useContext(WizardContext);
  if (!context) throw new Error('useWizard must be used within a <Wizard>');
  return context;
}

interface WizardProps<T extends FieldValues> {
  form: UseFormReturn<T>;
  onComplete: (data: T) => void;
  children: ReactNode[];
  /** Fields to validate at each step. Index matches step index. */
  stepFields?: (keyof T)[][];
}

/**
 * Multi-step form wizard.
 *
 * Replaces Angular Material MatStepper.
 *
 * Usage:
 *   <Wizard form={form} onComplete={handleSubmit} stepFields={[['firstname', 'lastname'], ['officeId']]}>
 *     <WizardStep title="Personal Info">
 *       <Input {...form.register('firstname')} />
 *     </WizardStep>
 *     <WizardStep title="Office">
 *       <Select {...form.register('officeId')} />
 *     </WizardStep>
 *   </Wizard>
 */
export function Wizard<T extends FieldValues>({
  form,
  onComplete,
  children,
  stepFields,
}: WizardProps<T>) {
  const [currentStep, setCurrentStep] = useState(0);
  const steps = Array.isArray(children) ? children : [children];
  const totalSteps = steps.length;

  async function nextStep() {
    // Validate only the fields for the current step
    if (stepFields?.[currentStep]) {
      const isValid = await form.trigger(stepFields[currentStep] as any);
      if (!isValid) return;
    }
    if (currentStep < totalSteps - 1) {
      setCurrentStep((s) => s + 1);
    }
  }

  function prevStep() {
    if (currentStep > 0) {
      setCurrentStep((s) => s - 1);
    }
  }

  function goToStep(step: number) {
    if (step >= 0 && step < totalSteps) {
      setCurrentStep(step);
    }
  }

  async function handleComplete() {
    const isValid = await form.trigger();
    if (isValid) {
      onComplete(form.getValues());
    }
  }

  const wizardValue: WizardContextValue = {
    currentStep,
    totalSteps,
    nextStep,
    prevStep,
    goToStep,
    isFirstStep: currentStep === 0,
    isLastStep: currentStep === totalSteps - 1,
  };

  return (
    <WizardContext.Provider value={wizardValue}>
      <FormProvider {...form}>
        {/* Step indicator */}
        <div className="mb-8 flex items-center justify-center gap-2">
          {steps.map((step, index) => (
            <div key={index} className="flex items-center">
              <button
                type="button"
                onClick={() => goToStep(index)}
                className={cn(
                  'flex h-8 w-8 items-center justify-center rounded-full text-sm font-medium',
                  index === currentStep
                    ? 'bg-primary text-primary-foreground'
                    : index < currentStep
                      ? 'bg-primary/20 text-primary'
                      : 'bg-muted text-muted-foreground',
                )}
              >
                {index + 1}
              </button>
              {index < totalSteps - 1 && (
                <div className={cn('mx-2 h-0.5 w-12', index < currentStep ? 'bg-primary' : 'bg-muted')} />
              )}
            </div>
          ))}
        </div>

        {/* Current step content */}
        <div className="min-h-[400px]">{steps[currentStep]}</div>

        {/* Navigation buttons */}
        <div className="mt-6 flex justify-between">
          <Button type="button" variant="outline" onClick={prevStep} disabled={currentStep === 0}>
            Previous
          </Button>
          {currentStep < totalSteps - 1 ? (
            <Button type="button" onClick={nextStep}>
              Next
            </Button>
          ) : (
            <Button type="button" onClick={handleComplete}>
              Submit
            </Button>
          )}
        </div>
      </FormProvider>
    </WizardContext.Provider>
  );
}

interface WizardStepProps {
  title: string;
  children: ReactNode;
}

export function WizardStep({ title, children }: WizardStepProps) {
  return (
    <div>
      <h2 className="mb-4 text-lg font-semibold">{title}</h2>
      {children}
    </div>
  );
}
```

### Loan Application Form Example

```typescript
// lib/validators/loan.ts
import { z } from 'zod';

export const loanApplicationSchema = z.object({
  // Step 1: Product selection
  clientId: z.number({ required_error: 'Client is required' }),
  productId: z.number({ required_error: 'Loan product is required' }),
  loanOfficerId: z.number().optional(),

  // Step 2: Terms
  principal: z.number({ required_error: 'Principal amount is required' }).positive('Amount must be positive'),
  loanTermFrequency: z.number().positive(),
  loanTermFrequencyType: z.number(),
  numberOfRepayments: z.number().positive(),
  repaymentEvery: z.number().positive(),
  repaymentFrequencyType: z.number(),
  interestRatePerPeriod: z.number().min(0),
  amortizationType: z.number(),
  interestType: z.number(),
  interestCalculationPeriodType: z.number(),
  transactionProcessingStrategyCode: z.string(),

  // Step 3: Dates
  expectedDisbursementDate: z.string().min(1, 'Disbursement date is required'),
  submittedOnDate: z.string().min(1, 'Submitted date is required'),
  repaymentsStartingFromDate: z.string().optional(),

  // Step 4: Charges
  charges: z
    .array(
      z.object({
        chargeId: z.number(),
        amount: z.number(),
        dueDate: z.string().optional()
      })
    )
    .optional(),

  // Fineract format params
  locale: z.string().default('en'),
  dateFormat: z.string().default('dd MMMM yyyy')
});

export type LoanApplicationPayload = z.infer<typeof loanApplicationSchema>;
```

```tsx
// app/(dashboard)/loans/create/page.tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loanApplicationSchema, type LoanApplicationPayload } from '@/lib/validators/loan';
import { useCreateLoan } from '@/lib/services/loans';
import { useLoanTemplate } from '@/lib/services/loans';
import { useRouter } from 'next/navigation';
import { Wizard, WizardStep } from '@/components/shared/wizard/wizard';
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

export default function CreateLoanPage() {
  const router = useRouter();
  const createLoan = useCreateLoan();

  const form = useForm<LoanApplicationPayload>({
    resolver: zodResolver(loanApplicationSchema),
    defaultValues: {
      locale: 'en',
      dateFormat: 'dd MMMM yyyy'
    }
  });

  const clientId = form.watch('clientId');
  const productId = form.watch('productId');

  // Fetch template data (dropdown options) based on selections
  const { data: template } = useLoanTemplate({
    clientId: clientId,
    productId: productId
  });

  async function onComplete(data: LoanApplicationPayload) {
    const result = await createLoan.mutateAsync(data);
    router.push(`/loans/${result.loanId}`);
  }

  return (
    <div className="mx-auto max-w-3xl">
      <h1 className="mb-6 text-2xl font-bold">Create Loan Application</h1>

      <Wizard
        form={form}
        onComplete={onComplete}
        stepFields={[
          [
            'clientId',
            'productId'
          ],
          [
            'principal',
            'numberOfRepayments',
            'loanTermFrequency'
          ],
          [
            'expectedDisbursementDate',
            'submittedOnDate'
          ],
          [] // Charges step has no required fields
        ]}
      >
        <WizardStep title="Product Selection">
          <FormField
            control={form.control}
            name="productId"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Loan Product</FormLabel>
                <FormControl>
                  <Select onValueChange={(v) => field.onChange(Number(v))} value={String(field.value ?? '')}>
                    <SelectTrigger>
                      <SelectValue placeholder="Select a product" />
                    </SelectTrigger>
                    <SelectContent>
                      {template?.productOptions?.map((product: any) => (
                        <SelectItem key={product.id} value={String(product.id)}>
                          {product.name}
                        </SelectItem>
                      ))}
                    </SelectContent>
                  </Select>
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </WizardStep>

        <WizardStep title="Loan Terms">
          <div className="grid grid-cols-2 gap-4">
            <FormField
              control={form.control}
              name="principal"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Principal Amount</FormLabel>
                  <FormControl>
                    <Input type="number" {...field} onChange={(e) => field.onChange(Number(e.target.value))} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
            {/* Additional term fields... */}
          </div>
        </WizardStep>

        <WizardStep title="Dates">{/* Date fields */}</WizardStep>

        <WizardStep title="Charges">{/* Charge selection and configuration */}</WizardStep>
      </Wizard>
    </div>
  );
}
```

---

## 5. Migration Mapping

A comprehensive mapping from Angular Observable patterns to React Query equivalents for each major service in the codebase.

### Pattern: Simple Data Fetch

```typescript
// ANGULAR (in component):
ngOnInit() {
  this.clientsService.getClients(this.offset, this.limit)
    .subscribe((clients) => {
      this.dataSource.data = clients.pageItems;
      this.totalResults = clients.totalFilteredRecords;
    });
}
ngOnDestroy() {
  // Unsubscribe logic (often missing, causing memory leaks)
}

// NEXT.JS:
function ClientsPage() {
  const { data, isLoading } = useClients({ page: 0, limit: 50 });
  // No cleanup needed. TanStack Query handles subscription lifecycle.
  return <DataTable data={data?.pageItems ?? []} />;
}
```

### Pattern: Route Resolver Data

```typescript
// ANGULAR (route resolver):
@Injectable()
export class ClientResolver implements Resolve<any> {
  resolve(route: ActivatedRouteSnapshot) {
    return this.clientsService.getClient(route.paramMap.get('id'));
  }
}
// In component:
ngOnInit() {
  this.route.data.subscribe(({ clientData }) => { this.client = clientData; });
}

// NEXT.JS (Server Component):
export default async function ClientPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const client = await fetchClient(Number(id)); // serverFetch()
  return <ClientDetail client={client} />;
}

// NEXT.JS (Client Component alternative):
function ClientDetail({ id }: { id: number }) {
  const { data: client, isLoading } = useClient(id);
  if (isLoading) return <Skeleton />;
  return <div>{client.displayName}</div>;
}
```

### Pattern: Dependent Queries

```typescript
// ANGULAR:
ngOnInit() {
  this.clientsService.getClient(this.clientId).pipe(
    switchMap(client => this.loansService.getLoans(client.id))
  ).subscribe(loans => { this.loans = loans; });
}

// NEXT.JS:
function ClientLoans({ clientId }: { clientId: number }) {
  const { data: client } = useClient(clientId);
  const { data: loans } = useLoans(
    { clientId: client?.id },
    { enabled: !!client?.id } // Only fetch loans after client is loaded
  );
  return <LoanList loans={loans} />;
}
```

### Pattern: Form Submission with Reload

```typescript
// ANGULAR:
onSubmit() {
  this.loansService.approveLoan(this.loanId, this.approveForm.value)
    .subscribe(() => {
      this.alertService.alert({ type: 'Success', message: 'Loan approved!' });
      this.router.navigate(['/loans', this.loanId]); // Reload by navigating
    });
}

// NEXT.JS:
function ApproveLoanDialog({ loanId }: { loanId: number }) {
  const approveLoan = useApproveLoan();
  const router = useRouter();

  async function onSubmit(data: ApprovePayload) {
    await approveLoan.mutateAsync({ id: loanId, ...data });
    // Cache is automatically invalidated by the mutation's onSuccess callback.
    // No manual navigation needed -- the data table and detail view will refetch.
    router.refresh(); // Optional: force server component re-render
  }
}
```

### Pattern: Polling (Notifications)

```typescript
// ANGULAR:
ngOnInit() {
  this.notificationInterval = setInterval(() => {
    this.notificationService.getNotifications().subscribe(data => {
      this.notifications = data;
    });
  }, environment.waitTimeForNotifications * 1000);
}
ngOnDestroy() {
  clearInterval(this.notificationInterval);
}

// NEXT.JS:
function useNotifications() {
  return useQuery({
    queryKey: ['notifications'],
    queryFn: () => fineractApi.get('/notifications').then(r => r.data),
    refetchInterval: Number(process.env.NEXT_PUBLIC_NOTIFICATION_POLL_INTERVAL ?? 60) * 1000,
    refetchIntervalInBackground: false, // Don't poll when tab is hidden
  });
}
```

### Pattern: BehaviorSubject for Shared State

```typescript
// ANGULAR:
@Injectable({ providedIn: 'root' })
export class SomeSharedService {
  private data$ = new BehaviorSubject<Data | null>(null);
  getData() { return this.data$.asObservable(); }
  setData(data: Data) { this.data$.next(data); }
}
// In component:
this.sharedService.getData().subscribe(data => { ... });

// NEXT.JS (if the data is from the API, use TanStack Query):
// Both components call useClient(42) -- TanStack Query deduplicates the request.

// NEXT.JS (if the data is UI state, use Zustand):
const useSharedStore = create<{ data: Data | null; setData: (d: Data) => void }>((set) => ({
  data: null,
  setData: (data) => set({ data }),
}));
// In component:
const data = useSharedStore((s) => s.data);
```

### Service-by-Service Migration Reference

| Angular Service         | Location                                                | Next.js Replacement                         | File                            |
| ----------------------- | ------------------------------------------------------- | ------------------------------------------- | ------------------------------- |
| `ClientsService`        | `src/app/clients/clients.service.ts`                    | TanStack Query hooks                        | `lib/services/clients.ts`       |
| `LoansService`          | `src/app/loans/loans.service.ts`                        | TanStack Query hooks                        | `lib/services/loans.ts`         |
| `SavingsService`        | `src/app/savings/savings.service.ts`                    | TanStack Query hooks                        | `lib/services/savings.ts`       |
| `GroupsService`         | `src/app/groups/groups.service.ts`                      | TanStack Query hooks                        | `lib/services/groups.ts`        |
| `CentersService`        | `src/app/centers/centers.service.ts`                    | TanStack Query hooks                        | `lib/services/centers.ts`       |
| `AccountingService`     | `src/app/accounting/accounting.service.ts`              | TanStack Query hooks                        | `lib/services/accounting.ts`    |
| `OrganizationService`   | `src/app/organization/organization.service.ts`          | TanStack Query hooks                        | `lib/services/organization.ts`  |
| `ProductsService`       | `src/app/products/products.service.ts`                  | TanStack Query hooks                        | `lib/services/products.ts`      |
| `SystemService`         | `src/app/system/system.service.ts`                      | TanStack Query hooks                        | `lib/services/system.ts`        |
| `UsersService`          | `src/app/users/users.service.ts`                        | TanStack Query hooks                        | `lib/services/users.ts`         |
| `SearchService`         | `src/app/search/search.service.ts`                      | TanStack Query hooks                        | `lib/services/search.ts`        |
| `NotificationsService`  | `src/app/notifications/notifications.service.ts`        | TanStack Query hooks with `refetchInterval` | `lib/services/notifications.ts` |
| `AlertService`          | `src/app/core/alert/alert.service.ts`                   | Zustand store                               | `lib/stores/alert-store.ts`     |
| `SettingsService`       | `src/app/settings/settings.service.ts`                  | Zustand store                               | `lib/stores/settings-store.ts`  |
| `I18nService`           | `src/app/core/i18n/i18n.service.ts`                     | `next-intl` hooks (`useTranslations`)       | `lib/i18n/config.ts`            |
| `AuthenticationService` | `src/app/core/authentication/authentication.service.ts` | NextAuth.js + auth context                  | `lib/auth/`                     |
| `ProgressBarService`    | `src/app/core/progress-bar/progress-bar.service.ts`     | TanStack Query `isFetching` global state    | Built-in                        |
| `RouteService`          | `src/app/core/route/route.service.ts`                   | Next.js `useRouter`, `usePathname`          | Built-in                        |
