# NEXTJS-008: Module-by-Module Migration Guide

## Status: Draft

## Last Updated: 2026-03-13

---

## Overview

This document provides a detailed migration guide for converting each Angular module to Next.js. The Clients module is covered in full depth as the reference implementation. Subsequent modules follow the same patterns with module-specific notes.

### Universal Conversion Patterns

Before diving into individual modules, these patterns apply across all modules:

| Angular Pattern                                     | Next.js Equivalent                                  |
| --------------------------------------------------- | --------------------------------------------------- | ------------------------------------- |
| `@Component` with template                          | React functional component (`.tsx`)                 |
| `@Injectable` service with `HttpClient`             | React Query hook calling Fineract API client        |
| `Resolve<T>` route resolver                         | `async` Server Component with direct data fetching  |
| Angular Reactive Forms (`FormGroup`, `FormControl`) | React Hook Form (`useForm`, `Controller`)           |
| Angular Material (`mat-*`)                          | shadcn/ui components                                |
| `                                                   | translate` pipe                                     | `t('key')` from next-intl             |
| `                                                   | date`/`DateFormatPipe`                              | `format.dateTime()` from next-intl    |
| `                                                   | currency`/`FormatNumberPipe`                        | `format.number()` with currency style |
| `StatusLookupPipe`                                  | Utility function `getStatusLabel()`                 |
| `AccountsFilterPipe`                                | Array `.filter()` in component                      |
| `ActivatedRoute.data` / `resolve`                   | `params` prop + async fetch in Server Component     |
| `Router.navigate()`                                 | `useRouter().push()`                                |
| `MatStepper` (wizard)                               | Custom stepper component or `@shadcn/ui` tabs       |
| `MatTable`                                          | `@tanstack/react-table` with shadcn DataTable       |
| `MatDialog`                                         | shadcn `Dialog` component                           |
| RxJS `Observable`                                   | React Query `useQuery` / `useMutation`              |
| `Subject` / `takeUntil` pattern                     | React Query automatic cleanup / `useEffect` cleanup |

---

## 1. Clients Module (Reference Implementation)

The Clients module is the most complex module with 20+ components, 15+ resolvers, and 30+ routes. It serves as the reference for all other migrations.

### 1.1 Angular Source Files

**Components (in `src/app/clients/`):**

- `clients.component.ts` -- Client list
- `create-client/create-client.component.ts` -- 5-step wizard (general, family, address, datatables, preview)
- `edit-client/edit-client.component.ts` -- Edit form
- `clients-view/clients-view.component.ts` -- Detail view wrapper with tabs
- `clients-view/general-tab/general-tab.component.ts` -- General info, accounts, charges
- `clients-view/family-members-tab/` -- List, add, edit family members
- `clients-view/identities-tab/` -- Identity documents
- `clients-view/notes-tab/` -- Notes
- `clients-view/documents-tab/` -- Uploaded documents
- `clients-view/address-tab/` -- Address management
- `clients-view/personal-data-tab/` -- Personal data
- `clients-view/datatable-tab/` -- Dynamic datatables
- `clients-view/charges/` -- Charge overview, view, pay
- `clients-view/client-actions/` -- Activate, close, transfer, etc.

**Services:**

- `clients.service.ts` -- All HTTP calls to `/fineract-provider/api/v1/clients`

**Resolvers (in `common-resolvers/`):**

- `ClientViewResolver`, `ClientAccountsResolver`, `ClientAddressResolver`, `ClientChargesResolver`, `ClientSummaryResolver`, `ClientFamilyMembersResolver`, `ClientFamilyMemberResolver`, `ClientTemplateResolver`, `ClientIdentitiesResolver`, `ClientNotesResolver`, `ClientDocumentsResolver`, `ClientDatatablesResolver`, `ClientDatatableResolver`, `ClientIdentifierTemplateResolver`, `ClientAddressFieldConfigurationResolver`, `ClientAddressTemplateResolver`, `ClientChargeOverviewResolver`, `ClientActionsResolver`, `ClientChargeViewResolver`, `ClientTransactionPayResolver`, `ClientDataAndTemplateResolver`, `ClientCollateralResolver`

### 1.2 Next.js Target Files

```
src/
├── app/(dashboard)/clients/
│   ├── page.tsx                                    # Client list (Server Component)
│   ├── loading.tsx                                 # List skeleton
│   ├── create/
│   │   └── page.tsx                                # Create wizard (Client Component)
│   ├── [clientId]/
│   │   ├── layout.tsx                              # Client header + tab nav
│   │   ├── page.tsx                                # Redirect to general
│   │   ├── loading.tsx                             # Detail skeleton
│   │   ├── general/page.tsx                        # General tab
│   │   ├── personal-data/page.tsx
│   │   ├── address/page.tsx
│   │   ├── family-members/
│   │   │   ├── page.tsx                            # List
│   │   │   ├── add/page.tsx                        # Add form
│   │   │   └── [familyMemberId]/edit/page.tsx      # Edit form
│   │   ├── identities/page.tsx
│   │   ├── documents/page.tsx
│   │   ├── notes/page.tsx
│   │   ├── datatables/[datatableName]/page.tsx
│   │   ├── edit/page.tsx                           # Edit client
│   │   ├── actions/[name]/page.tsx                 # Client actions
│   │   ├── charges/
│   │   │   ├── overview/page.tsx
│   │   │   └── [chargeId]/
│   │   │       ├── page.tsx
│   │   │       └── pay/page.tsx
│   │   ├── loans-accounts/...                      # Loans sub-module
│   │   ├── savings-accounts/...                    # Savings sub-module
│   │   └── ...other sub-modules
├── lib/
│   ├── api/
│   │   └── fineract/
│   │       └── clients.ts                          # Fineract API functions
│   ├── hooks/
│   │   └── clients.ts                              # React Query hooks
│   └── types/
│       └── clients.ts                              # TypeScript interfaces
└── components/
    └── clients/
        ├── client-header.tsx                        # Entity header
        ├── client-tabs.tsx                          # Tab navigation
        ├── client-list-table.tsx                    # DataTable for list
        ├── client-general-step.tsx                  # Wizard step 1
        ├── client-family-step.tsx                   # Wizard step 2
        ├── client-address-step.tsx                  # Wizard step 3
        ├── client-datatable-step.tsx                # Wizard step 4
        └── client-preview-step.tsx                  # Wizard step 5
```

### 1.3 Step-by-Step Migration

#### Step A: Create TypeScript Types

Extract interfaces from Angular component classes and service method signatures:

```typescript
// src/lib/types/clients.ts
export interface Client {
  id: number;
  accountNo: string;
  externalId?: string;
  status: ClientStatus;
  active: boolean;
  activationDate: number[];
  firstname: string;
  middlename?: string;
  lastname: string;
  displayName: string;
  mobileNo?: string;
  emailAddress?: string;
  dateOfBirth?: number[];
  gender: CodeValue;
  clientType?: CodeValue;
  clientClassification?: CodeValue;
  legalForm?: CodeValue;
  officeId: number;
  officeName: string;
  staffId?: number;
  staffName?: string;
  timeline: ClientTimeline;
  imagePresent: boolean;
  groups?: GroupSummary[];
}

export interface ClientStatus {
  id: number;
  code: string;
  value: string;
}

export interface CodeValue {
  id: number;
  name: string;
  isActive: boolean;
}

export interface ClientTemplate {
  officeOptions: Office[];
  staffOptions: Staff[];
  genderOptions: CodeValue[];
  clientTypeOptions: CodeValue[];
  clientClassificationOptions: CodeValue[];
  clientLegalFormOptions: CodeValue[];
  // ... more template fields
}

export interface ClientCreatePayload {
  officeId: number;
  firstname: string;
  lastname: string;
  middlename?: string;
  externalId?: string;
  dateOfBirth?: string;
  genderId?: number;
  mobileNo?: string;
  emailAddress?: string;
  active: boolean;
  activationDate?: string;
  submittedOnDate: string;
  locale: string;
  dateFormat: string;
  // ... more fields
}
```

#### Step B: Create Fineract API Client Functions

```typescript
// src/lib/api/fineract/clients.ts
import { fineractFetch } from '../fineract-fetch';
import type { Client, ClientTemplate, ClientCreatePayload } from '@/lib/types/clients';

const BASE = '/fineract-provider/api/v1';

export const clientsApi = {
  list: (params?: { offset?: number; limit?: number; sqlSearch?: string }) => fineractFetch<{ totalFilteredRecords: number; pageItems: Client[] }>(`${BASE}/clients?${new URLSearchParams(params as Record<string, string>)}`),

  get: (clientId: string | number) => fineractFetch<Client>(`${BASE}/clients/${clientId}`),

  getTemplate: () => fineractFetch<ClientTemplate>(`${BASE}/clients/template`),

  create: (data: ClientCreatePayload) => fineractFetch<{ clientId: number; resourceId: number }>(`${BASE}/clients`, { method: 'POST', body: JSON.stringify(data) }),

  update: (clientId: number, data: Partial<ClientCreatePayload>) => fineractFetch<{ clientId: number }>(`${BASE}/clients/${clientId}`, { method: 'PUT', body: JSON.stringify(data) }),

  getAccounts: (clientId: string | number) => fineractFetch(`${BASE}/clients/${clientId}/accounts`),

  getCharges: (clientId: string | number) => fineractFetch(`${BASE}/clients/${clientId}/charges`),

  getNotes: (clientId: string | number) => fineractFetch(`${BASE}/clients/${clientId}/notes`),

  getDocuments: (clientId: string | number) => fineractFetch(`${BASE}/clients/${clientId}/documents`),

  getFamilyMembers: (clientId: string | number) => fineractFetch(`${BASE}/clients/${clientId}/familymembers`),

  getIdentities: (clientId: string | number) => fineractFetch(`${BASE}/clients/${clientId}/identifiers`),

  getAddresses: (clientId: string | number) => fineractFetch(`${BASE}/clients/${clientId}/addresses`),

  getDatatables: (clientId: string | number) => fineractFetch(`${BASE}/datatables?apptable=m_client`),

  getDatatable: (datatableName: string, clientId: string | number) => fineractFetch(`${BASE}/datatables/${datatableName}/${clientId}`),

  executeAction: (clientId: string | number, action: string, data?: unknown) => fineractFetch(`${BASE}/clients/${clientId}?command=${action}`, { method: 'POST', body: data ? JSON.stringify(data) : undefined })
};
```

#### Step C: Create React Query Hooks

```typescript
// src/lib/hooks/clients.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { clientsApi } from '@/lib/api/fineract/clients';
import type { ClientCreatePayload } from '@/lib/types/clients';

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
  detail: (id: string | number) => [
      ...clientKeys.details(),
      id
    ] as const,
  accounts: (id: string | number) => [
      ...clientKeys.detail(id),
      'accounts'
    ] as const,
  charges: (id: string | number) => [
      ...clientKeys.detail(id),
      'charges'
    ] as const,
  notes: (id: string | number) => [
      ...clientKeys.detail(id),
      'notes'
    ] as const,
  documents: (id: string | number) => [
      ...clientKeys.detail(id),
      'documents'
    ] as const,
  familyMembers: (id: string | number) => [
      ...clientKeys.detail(id),
      'familyMembers'
    ] as const,
  identities: (id: string | number) => [
      ...clientKeys.detail(id),
      'identities'
    ] as const,
  template: () => [
      ...clientKeys.all,
      'template'
    ] as const
};

export function useClients(params?: { offset?: number; limit?: number }) {
  return useQuery({
    queryKey: clientKeys.list(params ?? {}),
    queryFn: () => clientsApi.list(params)
  });
}

export function useClient(clientId: string | number) {
  return useQuery({
    queryKey: clientKeys.detail(clientId),
    queryFn: () => clientsApi.get(clientId),
    enabled: !!clientId
  });
}

export function useClientAccounts(clientId: string | number) {
  return useQuery({
    queryKey: clientKeys.accounts(clientId),
    queryFn: () => clientsApi.getAccounts(clientId),
    enabled: !!clientId
  });
}

export function useClientTemplate() {
  return useQuery({
    queryKey: clientKeys.template(),
    queryFn: () => clientsApi.getTemplate()
  });
}

export function useCreateClient() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: ClientCreatePayload) => clientsApi.create(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: clientKeys.lists() });
    }
  });
}

export function useUpdateClient(clientId: number) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: Partial<ClientCreatePayload>) => clientsApi.update(clientId, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: clientKeys.detail(clientId) });
    }
  });
}

export function useClientAction(clientId: string | number) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ action, data }: { action: string; data?: unknown }) => clientsApi.executeAction(clientId, action, data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: clientKeys.detail(clientId) });
    }
  });
}
```

#### Step D: Build List Page (Server Component)

**Angular (ClientsComponent):**

- Fetches client list via resolver or on-demand search
- Renders `MatTable` with pagination
- Has search functionality

**Next.js:**

```typescript
// app/(dashboard)/clients/page.tsx
import { Suspense } from 'react';
import { getTranslations } from 'next-intl/server';
import { ClientListTable } from '@/components/clients/client-list-table';

export async function generateMetadata() {
  const t = await getTranslations();
  return { title: t('labels.clients') };
}

interface Props {
  searchParams: Promise<{ page?: string; search?: string }>;
}

export default async function ClientsPage({ searchParams }: Props) {
  const { page, search } = await searchParams;
  const t = await getTranslations();

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">{t('labels.clients')}</h1>
      </div>
      <Suspense fallback={<div>Loading...</div>}>
        <ClientListTable
          initialPage={page ? parseInt(page) : 1}
          initialSearch={search}
        />
      </Suspense>
    </div>
  );
}
```

```typescript
// components/clients/client-list-table.tsx
'use client';

import { useClients } from '@/lib/hooks/clients';
import { DataTable } from '@/components/ui/data-table';
import { useTranslations } from 'next-intl';
import Link from 'next/link';
import { useRouter, useSearchParams } from 'next/navigation';
import { useState } from 'react';

export function ClientListTable({ initialPage, initialSearch }: {
  initialPage: number;
  initialSearch?: string;
}) {
  const t = useTranslations();
  const router = useRouter();
  const [page, setPage] = useState(initialPage);
  const [search, setSearch] = useState(initialSearch ?? '');

  const { data, isLoading } = useClients({
    offset: (page - 1) * 20,
    limit: 20,
    ...(search ? { sqlSearch: search } : {}),
  });

  const columns = [
    {
      accessorKey: 'displayName',
      header: t('labels.name'),
      cell: ({ row }: any) => (
        <Link href={`/clients/${row.original.id}`} className="text-primary hover:underline">
          {row.original.displayName}
        </Link>
      ),
    },
    { accessorKey: 'accountNo', header: t('labels.accountNumber') },
    { accessorKey: 'officeName', header: t('labels.office') },
    {
      accessorKey: 'status',
      header: t('labels.status'),
      cell: ({ row }: any) => (
        <span className={`badge badge-${row.original.status.code}`}>
          {row.original.status.value}
        </span>
      ),
    },
  ];

  return (
    <DataTable
      columns={columns}
      data={data?.pageItems ?? []}
      isLoading={isLoading}
      totalCount={data?.totalFilteredRecords ?? 0}
      page={page}
      onPageChange={setPage}
      searchValue={search}
      onSearchChange={setSearch}
    />
  );
}
```

#### Step E: Build Create Wizard (Client Component with React Hook Form)

**Angular pattern:** `MatStepper` with `FormGroup` per step, `@ViewChild` to access step data.

**Next.js pattern:** Custom stepper component with React Hook Form, each step is a sub-component.

```typescript
// app/(dashboard)/clients/create/page.tsx
import { getTranslations } from 'next-intl/server';
import { CreateClientWizard } from '@/components/clients/create-client-wizard';

export async function generateMetadata() {
  const t = await getTranslations();
  return { title: t('labels.createClient') };
}

export default function CreateClientPage() {
  return <CreateClientWizard />;
}
```

```typescript
// components/clients/create-client-wizard.tsx
'use client';

import { useState } from 'react';
import { useForm, FormProvider } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useRouter } from 'next/navigation';
import { useTranslations } from 'next-intl';
import { useCreateClient, useClientTemplate } from '@/lib/hooks/clients';
import { ClientGeneralStep } from './client-general-step';
import { ClientFamilyStep } from './client-family-step';
import { ClientAddressStep } from './client-address-step';
import { ClientDatatableStep } from './client-datatable-step';
import { ClientPreviewStep } from './client-preview-step';
import { Button } from '@/components/ui/button';
import { toast } from 'sonner';

const clientSchema = z.object({
  officeId: z.number().min(1),
  firstname: z.string().min(1),
  lastname: z.string().min(1),
  middlename: z.string().optional(),
  externalId: z.string().optional(),
  dateOfBirth: z.string().optional(),
  genderId: z.number().optional(),
  mobileNo: z.string().optional(),
  emailAddress: z.string().email().optional().or(z.literal('')),
  active: z.boolean(),
  activationDate: z.string().optional(),
  submittedOnDate: z.string(),
  // ... additional fields
});

type ClientFormData = z.infer<typeof clientSchema>;

const STEPS = ['General', 'Family Members', 'Address', 'Data Tables', 'Preview'];

export function CreateClientWizard() {
  const t = useTranslations();
  const router = useRouter();
  const [currentStep, setCurrentStep] = useState(0);
  const { data: template, isLoading: templateLoading } = useClientTemplate();
  const createClient = useCreateClient();

  const methods = useForm<ClientFormData>({
    resolver: zodResolver(clientSchema),
    defaultValues: {
      active: false,
      submittedOnDate: new Date().toISOString().split('T')[0],
    },
  });

  async function onSubmit(data: ClientFormData) {
    try {
      const result = await createClient.mutateAsync({
        ...data,
        locale: 'en',
        dateFormat: 'dd MMMM yyyy',
      });
      toast.success(t('labels.clientCreatedSuccessfully'));
      router.push(`/clients/${result.clientId}`);
    } catch (error) {
      toast.error(t('errors.clientCreationFailed'));
    }
  }

  if (templateLoading || !template) return <div>Loading template...</div>;

  return (
    <FormProvider {...methods}>
      <form onSubmit={methods.handleSubmit(onSubmit)}>
        {/* Step indicator */}
        <div className="mb-8 flex items-center gap-2">
          {STEPS.map((step, index) => (
            <div key={step} className="flex items-center gap-2">
              <div className={`flex h-8 w-8 items-center justify-center rounded-full
                ${index <= currentStep ? 'bg-primary text-white' : 'bg-gray-200'}`}>
                {index + 1}
              </div>
              <span className="text-sm">{step}</span>
              {index < STEPS.length - 1 && <div className="h-px w-8 bg-gray-300" />}
            </div>
          ))}
        </div>

        {/* Step content */}
        {currentStep === 0 && <ClientGeneralStep template={template} />}
        {currentStep === 1 && <ClientFamilyStep template={template} />}
        {currentStep === 2 && <ClientAddressStep />}
        {currentStep === 3 && <ClientDatatableStep />}
        {currentStep === 4 && <ClientPreviewStep />}

        {/* Navigation buttons */}
        <div className="mt-6 flex justify-between">
          <Button
            type="button"
            variant="outline"
            onClick={() => setCurrentStep(Math.max(0, currentStep - 1))}
            disabled={currentStep === 0}
          >
            {t('labels.buttons.previous')}
          </Button>
          {currentStep < STEPS.length - 1 ? (
            <Button
              type="button"
              onClick={() => setCurrentStep(currentStep + 1)}
            >
              {t('labels.buttons.next')}
            </Button>
          ) : (
            <Button type="submit" disabled={createClient.isPending}>
              {createClient.isPending ? t('labels.buttons.creating') : t('labels.buttons.submit')}
            </Button>
          )}
        </div>
      </form>
    </FormProvider>
  );
}
```

#### Step F: Build Detail Layout with Tab Navigation

```typescript
// app/(dashboard)/clients/[clientId]/layout.tsx
import { clientsApi } from '@/lib/api/fineract/clients';
import { ClientHeader } from '@/components/clients/client-header';
import { ClientTabs } from '@/components/clients/client-tabs';
import { notFound } from 'next/navigation';

interface Props {
  children: React.ReactNode;
  params: Promise<{ clientId: string }>;
}

export default async function ClientDetailLayout({ children, params }: Props) {
  const { clientId } = await params;

  let client;
  try {
    client = await clientsApi.get(clientId);
  } catch {
    notFound();
  }

  const datatables = await clientsApi.getDatatables(clientId);

  return (
    <div className="space-y-4">
      <ClientHeader client={client} />
      <ClientTabs clientId={clientId} datatables={datatables} />
      <div>{children}</div>
    </div>
  );
}
```

#### Step G: Build Each Tab Page

```typescript
// app/(dashboard)/clients/[clientId]/general/page.tsx
import { clientsApi } from '@/lib/api/fineract/clients';
import { ClientGeneralTab } from '@/components/clients/client-general-tab';

interface Props {
  params: Promise<{ clientId: string }>;
}

export default async function ClientGeneralPage({ params }: Props) {
  const { clientId } = await params;

  // Replaces ClientAccountsResolver, ClientChargesResolver, ClientCollateralResolver
  const [accounts, charges, collateral] = await Promise.all([
    clientsApi.getAccounts(clientId),
    clientsApi.getCharges(clientId),
    clientsApi.getCollateral(clientId),
  ]);

  return (
    <ClientGeneralTab
      clientId={clientId}
      accounts={accounts}
      charges={charges}
      collateral={collateral}
    />
  );
}
```

---

## 2. Groups Module

### 2.1 Angular Structure

- **Components:** `GroupsComponent` (list), `CreateGroupComponent` (3-step), `GroupsViewComponent` (detail with tabs: general, notes, committee, datatables), `EditGroupComponent`, `GroupActionsComponent`, `AddRoleComponent`
- **Services:** Methods for group CRUD, group accounts, group summary, GSIM/GLIM
- **Resolvers:** `GroupViewResolver`, `GroupAccountsResolver`, `GroupSummaryResolver`, `GroupNotesResolver`, `GroupDatatablesResolver`, `GroupDatatableResolver`, `GroupDataAndTemplateResolver`, `GroupActionsResolver`, `GLIMAccountsResolver`, `GSIMAccountsResolver`

### 2.2 Next.js Target

```
app/(dashboard)/groups/
├── page.tsx                                # Group list
├── create/page.tsx                         # 3-step wizard
└── [groupId]/
    ├── layout.tsx                          # Group header + tabs
    ├── page.tsx                            # Redirect to general
    ├── general/page.tsx                    # Accounts, summary, GSIM/GLIM
    ├── notes/page.tsx
    ├── committee/page.tsx
    ├── committee/add-role/page.tsx
    ├── datatables/[datatableName]/page.tsx
    ├── edit/page.tsx
    ├── actions/[action]/page.tsx
    ├── loans-accounts/...                  # Shared loans sub-module
    └── savings-accounts/...                # Shared savings sub-module
```

### 2.3 Key Conversion: React Query Hooks

```typescript
// src/lib/hooks/groups.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { groupsApi } from '@/lib/api/fineract/groups';

export const groupKeys = {
  all: ['groups'] as const,
  detail: (id: string | number) => [
      ...groupKeys.all,
      'detail',
      id
    ] as const,
  accounts: (id: string | number) => [
      ...groupKeys.detail(id),
      'accounts'
    ] as const,
  summary: (id: string | number) => [
      ...groupKeys.detail(id),
      'summary'
    ] as const,
  notes: (id: string | number) => [
      ...groupKeys.detail(id),
      'notes'
    ] as const,
  gsim: (id: string | number) => [
      ...groupKeys.detail(id),
      'gsim'
    ] as const,
  glim: (id: string | number) => [
      ...groupKeys.detail(id),
      'glim'
    ] as const
};

export function useGroup(groupId: string | number) {
  return useQuery({
    queryKey: groupKeys.detail(groupId),
    queryFn: () => groupsApi.get(groupId)
  });
}

export function useGroupAccounts(groupId: string | number) {
  return useQuery({
    queryKey: groupKeys.accounts(groupId),
    queryFn: () => groupsApi.getAccounts(groupId)
  });
}

export function useGroupSummary(groupId: string | number) {
  return useQuery({
    queryKey: groupKeys.summary(groupId),
    queryFn: () => groupsApi.getSummary(groupId)
  });
}
```

### 2.4 Create Group Wizard

The Angular wizard uses `OfficesResolver` to prefetch office data. In Next.js:

```typescript
// app/(dashboard)/groups/create/page.tsx
'use client'; // Wizard requires interactivity

import { useForm } from 'react-hook-form';
import { useQuery } from '@tanstack/react-query';
import { officesApi } from '@/lib/api/fineract/offices';

export default function CreateGroupPage() {
  const { data: offices } = useQuery({
    queryKey: ['offices'],
    queryFn: () => officesApi.list()
  });

  // 3-step wizard: details, members, preview
  // ... similar pattern to Clients wizard
}
```

---

## 3. Loans Module

### 3.1 Angular Structure

The Loans module is the second most complex, featuring:

- **CreateLoansAccountComponent:** 7-step wizard (details, terms, charges, overdue charges, tranche details, multi-disburse, review)
- **LoansViewComponent:** Detail view with 18+ tabs
- **LoanAccountActionsComponent:** Dynamic action forms (approve, disburse, repay, write-off, reschedule, etc.)
- **GLIM support:** Group Loan Individual Monitoring
- **30+ resolvers** for various data dependencies

### 3.2 Next.js Target

```
app/(dashboard)/clients/[clientId]/loans-accounts/
├── create/page.tsx                         # 7-step wizard
└── [loanId]/
    ├── layout.tsx                          # Loan header + tabs
    ├── page.tsx                            # Redirect to general
    ├── general/page.tsx
    ├── dashboard/page.tsx
    ├── accountdetail/page.tsx
    ├── repayment-schedule/page.tsx
    ├── original-schedule/page.tsx
    ├── transactions/
    │   ├── page.tsx
    │   ├── export/page.tsx
    │   └── [id]/
    │       ├── page.tsx                    # View transaction
    │       ├── edit/page.tsx
    │       └── receipt/page.tsx
    ├── charges/
    │   ├── page.tsx
    │   └── [id]/
    │       ├── page.tsx
    │       └── adjustment/page.tsx
    ├── notes/page.tsx
    ├── loan-documents/page.tsx
    ├── loan-collateral/page.tsx
    ├── delinquencytags/page.tsx
    ├── loan-reschedules/page.tsx
    ├── term-variations/page.tsx
    ├── deferred-income/page.tsx
    ├── buy-down-fees/page.tsx
    ├── originators/page.tsx
    ├── standing-instruction/page.tsx
    ├── external-asset-owner/page.tsx
    ├── floating-interest-rates/page.tsx
    ├── loan-tranche-details/page.tsx
    ├── overdue-charges/page.tsx
    ├── datatables/[datatableName]/page.tsx
    ├── edit/page.tsx
    └── actions/[action]/page.tsx
```

### 3.3 Key Pattern: Loan Action Forms

The Angular `LoanAccountActionsComponent` dynamically renders different forms based on the `action` route parameter. In Next.js:

```typescript
// app/(dashboard)/clients/[clientId]/loans-accounts/[loanId]/actions/[action]/page.tsx
import { notFound } from 'next/navigation';
import { LoanActionForm } from '@/components/loans/loan-action-form';

const VALID_ACTIONS = [
  'approve', 'reject', 'withdraw', 'disburse', 'disbursement-undo',
  'repayment', 'prepay-loan', 'write-off', 'close', 'close-as-rescheduled',
  'reschedule', 'waive-interest', 'add-charge', 'foreclosure',
  'make-refund', 'undo-approval', 'undo-disbursal',
  'credit-balance-refund', 'charge-off',
] as const;

interface Props {
  params: Promise<{ clientId: string; loanId: string; action: string }>;
}

export default async function LoanActionPage({ params }: Props) {
  const { clientId, loanId, action } = await params;

  if (!VALID_ACTIONS.includes(action as any)) {
    notFound();
  }

  return (
    <LoanActionForm
      clientId={clientId}
      loanId={loanId}
      action={action}
    />
  );
}
```

### 3.4 Loan Create Wizard (7 Steps)

```typescript
// components/loans/create-loan-wizard.tsx
'use client';

const LOAN_STEPS = [
  'Details',
  'Terms',
  'Charges',
  'Overdue Charges',
  'Tranche Details',
  'Multi Disburse',
  'Review'
];

// Same pattern as Client wizard but with 7 steps
// Uses useForm with loan-specific schema
// Fetches template via useQuery({ queryKey: ['loanTemplate'], queryFn: ... })
```

---

## 4. Accounting Module

### 4.1 Angular Structure

- **AccountingComponent:** Landing page with navigation cards
- **Journal Entries:** Search, create, frequent postings, view transactions
- **Chart of Accounts:** Tree view of GL accounts with CRUD
- **Closing Entries:** List, create, view, edit closures
- **Accounting Rules:** CRUD with template-based forms
- **Financial Activity Mappings:** CRUD
- **Provisioning Entries:** CRUD with journal entry views
- **Periodic Accruals:** Single form page

### 4.2 Key Pattern: Chart of Accounts Tree

The chart of accounts uses a tree structure. In Next.js, render it as a server-fetched tree with client-side expand/collapse:

```typescript
// app/(dashboard)/accounting/chart-of-accounts/page.tsx
import { accountingApi } from '@/lib/api/fineract/accounting';
import { ChartOfAccountsTree } from '@/components/accounting/chart-of-accounts-tree';

export default async function ChartOfAccountsPage() {
  const chartOfAccounts = await accountingApi.getChartOfAccounts();

  return <ChartOfAccountsTree data={chartOfAccounts} />;
}
```

```typescript
// components/accounting/chart-of-accounts-tree.tsx
'use client';

import { useState } from 'react';
import Link from 'next/link';

interface GlAccount {
  id: number;
  name: string;
  glCode: string;
  type: { value: string };
  children?: GlAccount[];
}

export function ChartOfAccountsTree({ data }: { data: GlAccount[] }) {
  return (
    <div className="space-y-1">
      {data.map(account => (
        <TreeNode key={account.id} account={account} depth={0} />
      ))}
    </div>
  );
}

function TreeNode({ account, depth }: { account: GlAccount; depth: number }) {
  const [isExpanded, setIsExpanded] = useState(false);
  const hasChildren = account.children && account.children.length > 0;

  return (
    <div>
      <div
        className="flex items-center gap-2 rounded px-2 py-1 hover:bg-muted"
        style={{ paddingLeft: `${depth * 24 + 8}px` }}
      >
        {hasChildren && (
          <button onClick={() => setIsExpanded(!isExpanded)} className="text-xs">
            {isExpanded ? '[-]' : '[+]'}
          </button>
        )}
        <Link href={`/accounting/chart-of-accounts/gl-accounts/view/${account.id}`}>
          {account.glCode} - {account.name}
        </Link>
        <span className="text-xs text-muted-foreground">{account.type.value}</span>
      </div>
      {isExpanded && account.children?.map(child => (
        <TreeNode key={child.id} account={child} depth={depth + 1} />
      ))}
    </div>
  );
}
```

### 4.3 Journal Entry Form

The journal entry form has debit/credit rows with GL account selection. This maps well to React Hook Form's `useFieldArray`:

```typescript
// components/accounting/journal-entry-form.tsx
'use client';

import { useForm, useFieldArray } from 'react-hook-form';

interface JournalEntryForm {
  officeId: number;
  currencyCode: string;
  transactionDate: string;
  referenceNumber?: string;
  debits: { glAccountId: number; amount: number }[];
  credits: { glAccountId: number; amount: number }[];
}

export function JournalEntryForm() {
  const { control, handleSubmit } = useForm<JournalEntryForm>({
    defaultValues: {
      debits: [{ glAccountId: 0, amount: 0 }],
      credits: [{ glAccountId: 0, amount: 0 }]
    }
  });

  const debits = useFieldArray({ control, name: 'debits' });
  const credits = useFieldArray({ control, name: 'credits' });

  // ... render form with dynamic debit/credit rows
}
```

---

## 5. Products Module

### 5.1 Angular Structure

The most form-heavy module with complex multi-step wizards:

- **Loan Product:** 10-step wizard (details, currency, terms, settings, charges, accounting, tranche, rates, overdue charges, collateral management)
- **Saving Product, Share Product, Fixed/Recurring Deposit Products:** Each with multi-step create/edit
- **Charges, Collaterals, Floating Rates, Products Mix:** Standard CRUD
- **Tax Configurations:** Tax components and groups
- **Delinquency Buckets:** Ranges and buckets configuration

### 5.2 Loan Product Wizard Migration

The 10-step loan product wizard is the most complex form in the application.

```typescript
// components/products/create-loan-product-wizard.tsx
'use client';

import { useForm, FormProvider } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

const LOAN_PRODUCT_STEPS = [
  'Details',
  'Currency',
  'Terms',
  'Settings',
  'Charges',
  'Accounting',
  'Tranche Details',
  'Interest Rates',
  'Overdue Charges',
  'Collateral Management'
];

// The form state is shared across all 10 steps via FormProvider
// Each step component receives the template data and accesses
// form state via useFormContext()

export function CreateLoanProductWizard({ template, configurations }: { template: LoanProductTemplate; configurations: GlobalConfigurations }) {
  const methods = useForm({
    resolver: zodResolver(loanProductSchema),
    defaultValues: buildDefaultValues(template, configurations)
  });

  // ... stepper UI with 10 steps
}
```

### 5.3 Service to Hook Mapping (Products)

| Angular Service Method                                               | React Query Hook                                            |
| -------------------------------------------------------------------- | ----------------------------------------------------------- |
| `ProductsService.getLoanProducts()`                                  | `useQuery({ queryKey: ['loanProducts'], ... })`             |
| `ProductsService.getLoanProduct(id)`                                 | `useQuery({ queryKey: ['loanProducts', id], ... })`         |
| `ProductsService.getLoanProductTemplate()`                           | `useQuery({ queryKey: ['loanProducts', 'template'], ... })` |
| `ProductsService.createLoanProduct(data)`                            | `useMutation({ mutationFn: ... })`                          |
| `ProductsService.updateLoanProduct(id, data)`                        | `useMutation({ mutationFn: ... })`                          |
| Same pattern for savings, shares, fixed deposits, recurring deposits | Same pattern                                                |

---

## 6. Organization Module

### 6.1 Sub-module Breakdown

The Organization module has 16 sub-features. Each follows the standard list/create/view/edit pattern:

| Sub-feature                   | Angular Components        | Next.js Pages |
| ----------------------------- | ------------------------- | ------------- |
| Offices                       | 5 components + datatables | 5 pages       |
| Employees                     | 4 components              | 4 pages       |
| Currencies                    | 2 components              | 2 pages       |
| SMS Campaigns                 | 4 components              | 4 pages       |
| Tellers/Cashiers              | 12 components (nested)    | 12 pages      |
| Payment Types                 | 3 components              | 3 pages       |
| Holidays                      | 4 components              | 4 pages       |
| Adhoc Queries                 | 4 components              | 4 pages       |
| Provisioning Criteria         | 4 components              | 4 pages       |
| Manage Funds                  | 4 components              | 4 pages       |
| Loan Originators              | 4 components              | 4 pages       |
| Bulk Import                   | 2 components              | 2 pages       |
| Bulk Loan Reassignment        | 1 component               | 1 page        |
| Entity Data Table Checks      | 2 components              | 2 pages       |
| Working Days                  | 1 component               | 1 page        |
| Password Preferences          | 1 component               | 1 page        |
| Standing Instructions History | 1 component               | 1 page        |
| Fund Mapping                  | 1 component               | 1 page        |
| Investors                     | 1 component               | 1 page        |

### 6.2 Tellers/Cashiers Nesting

The tellers module has deep nesting (teller > cashier > actions). This maps to:

```
organization/tellers/
├── page.tsx
├── create/page.tsx
└── [id]/
    ├── page.tsx
    ├── edit/page.tsx
    └── cashiers/
        ├── page.tsx
        ├── create/page.tsx
        └── [cid]/
            ├── page.tsx
            ├── edit/page.tsx
            ├── transactions/page.tsx
            ├── settle/page.tsx
            └── allocate/page.tsx
```

---

## 7. System Module

### 7.1 Sub-module Breakdown

| Sub-feature                 | Angular Components | Next.js Pages |
| --------------------------- | ------------------ | ------------- |
| Codes (+ Code Values)       | 4 components       | 4 pages       |
| Data Tables                 | 4 components       | 4 pages       |
| Hooks                       | 4 components       | 4 pages       |
| Roles and Permissions       | 4 components       | 4 pages       |
| Surveys                     | 4 components       | 4 pages       |
| Manage Jobs                 | 4 components       | 4 pages       |
| Configurations              | 2 components       | 2 pages       |
| Account Number Preferences  | 4 components       | 4 pages       |
| Reports                     | 4 components       | 4 pages       |
| External Services (4 types) | 9 components       | 9 pages       |
| External Events             | 1 component        | 1 page        |
| Entity Mapping              | 1 component        | 1 page        |
| Maker Checker               | 1 component        | 1 page        |
| Audit Trails                | 2 components       | 2 pages       |
| System Information          | 1 component        | 1 page        |
| About Us                    | 1 component        | 1 page        |

### 7.2 Configurations Page

The global configurations page uses `MatTable` with inline editing:

```typescript
// app/(dashboard)/system/configurations/page.tsx
import { systemApi } from '@/lib/api/fineract/system';
import { ConfigurationsTable } from '@/components/system/configurations-table';

export default async function ConfigurationsPage() {
  const configurations = await systemApi.getGlobalConfigurations();

  return <ConfigurationsTable data={configurations} />;
}
```

### 7.3 Manage Jobs with Real-time Status

The scheduler jobs page needs periodic refresh for job status:

```typescript
// components/system/scheduler-jobs-table.tsx
'use client';

import { useQuery } from '@tanstack/react-query';
import { systemApi } from '@/lib/api/fineract/system';

export function SchedulerJobsTable() {
  const { data: jobs } = useQuery({
    queryKey: ['schedulerJobs'],
    queryFn: () => systemApi.getSchedulerJobs(),
    refetchInterval: 5000 // Poll every 5 seconds for status updates
  });

  // ... render jobs table with run/pause controls
}
```

---

## 8. Migration Priority and Sequencing

### Recommended Migration Order

1. **Clients Module** (Week 1-2): Most complex, establishes all patterns
2. **Groups Module** (Week 2-3): Similar to clients, shares loans/savings sub-modules
3. **Centers Module** (Week 3): Simpler version of groups
4. **Loans Module** (Week 3-4): Complex but patterns established
5. **Savings Module** (Week 4-5): Similar to loans
6. **Deposits Modules** (Week 5): Fixed and recurring, similar to savings
7. **Shares Module** (Week 5-6): Similar pattern
8. **Products Module** (Week 6-8): Many forms, wizards
9. **Accounting Module** (Week 8-9): Independent, moderate complexity
10. **Organization Module** (Week 9-10): Many small CRUD sub-features
11. **System Module** (Week 10-11): Admin features, lower priority
12. **Remaining Modules** (Week 11-12): Search, reports, templates, users, tasks

### Per-Module Checklist

For each module migration:

- [ ] Define TypeScript types in `src/lib/types/{module}.ts`
- [ ] Create API client functions in `src/lib/api/fineract/{module}.ts`
- [ ] Create React Query hooks in `src/lib/hooks/{module}.ts`
- [ ] Build list page (Server Component with search/pagination)
- [ ] Build create form (Client Component with React Hook Form)
- [ ] Build detail layout with tab navigation
- [ ] Build each tab page
- [ ] Build edit form
- [ ] Build action/command pages
- [ ] Add `loading.tsx` for key route segments
- [ ] Add `error.tsx` for key route segments
- [ ] Write unit tests for hooks and utility functions
- [ ] Write component tests for forms and interactive elements
- [ ] Verify i18n for all user-facing strings
