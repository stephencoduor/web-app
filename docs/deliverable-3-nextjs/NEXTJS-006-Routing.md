# NEXTJS-006: Routing Architecture

## Status: Draft

## Last Updated: 2026-03-13

---

## 1. Angular Router to Next.js App Router -- Conceptual Mapping

The M-SACCO Angular app uses a custom `Route.withShell()` helper that wraps all authenticated routes in a `ShellComponent` with an `AuthenticationGuard`. The Angular router relies on hash-based routing (`useHash: true`) and extensive use of route resolvers for prefetching data. Next.js App Router replaces all of these patterns with file-system-based conventions.

### 1.1 Concept Mapping Table

| Angular Concept                                 | Next.js App Router Equivalent                                                     |
| ----------------------------------------------- | --------------------------------------------------------------------------------- |
| `NgModule` with lazy `loadChildren`             | Automatic per-route-segment code splitting (no config needed)                     |
| Route resolvers (`resolve: { data: Resolver }`) | Server Components with `async` page/layout functions (data fetched in `page.tsx`) |
| `AuthenticationGuard` (canActivate)             | `middleware.ts` at project root + layout-level session checks                     |
| `Route.withShell([...])` wrapping               | `(dashboard)/layout.tsx` route group layout                                       |
| Hash routing (`useHash: true`, `/#/clients`)    | Standard URL path routing (`/clients`)                                            |
| `routeParamBreadcrumb` in route data            | Dynamic breadcrumb generation from URL segments + `generateMetadata()`            |
| `data: { title: '...' }`                        | `export const metadata` or `generateMetadata()` per page                          |
| `loadChildren: () => import(...)`               | Automatic -- each `page.tsx` is its own chunk                                     |
| `children: [...]` nested routes                 | Nested folders with `layout.tsx` for shared UI                                    |
| `redirectTo: 'general'`                         | `redirect()` in `page.tsx` or Next.js `redirects` in config                       |

### 1.2 Hash Routing Migration

The current Angular app uses `useHash: true`, meaning all routes are prefixed with `/#/`. For example: `https://app.example.com/#/clients/42/general`.

In Next.js, routes are standard URL paths: `https://app.example.com/clients/42`. This change requires:

- Updating all bookmarked URLs (provide a redirect middleware for `/#/` paths during transition)
- Updating any external systems that link to the app
- Adding a hash-to-path redirect in `middleware.ts`:

```typescript
// middleware.ts -- hash redirect support during migration
import { NextRequest, NextResponse } from 'next/server';

export function middleware(request: NextRequest) {
  // Handle legacy hash-based URLs
  const url = request.nextUrl;
  if (url.pathname === '/' && url.hash) {
    const newPath = url.hash.replace('#/', '/');
    return NextResponse.redirect(new URL(newPath, url.origin));
  }
  // ... auth checks below
}
```

### 1.3 Route Resolvers to Server Components

Angular resolvers pre-fetch data before the route activates. In Next.js, Server Components fetch data directly in the component tree. The resolved data becomes the return value of an async function call.

**Angular pattern (resolver):**

```typescript
// client-view.resolver.ts
@Injectable()
export class ClientViewResolver implements Resolve<any> {
  constructor(private clientsService: ClientsService) {}
  resolve(route: ActivatedRouteSnapshot) {
    return this.clientsService.getClientData(route.paramMap.get('clientId'));
  }
}

// In routing module:
{
  path: ':clientId',
  component: ClientsViewComponent,
  resolve: { clientViewData: ClientViewResolver }
}
```

**Next.js equivalent (Server Component):**

```typescript
// app/(dashboard)/clients/[clientId]/page.tsx
import { fineract } from '@/lib/api/fineract';

interface Props {
  params: Promise<{ clientId: string }>;
}

export default async function ClientPage({ params }: Props) {
  const { clientId } = await params;
  const clientData = await fineract.clients.get(clientId);

  return <ClientView data={clientData} />;
}
```

### 1.4 Route Guards to Middleware

The Angular `AuthenticationGuard` checks authentication on every shell-wrapped route. In Next.js, `middleware.ts` performs this check at the edge before any page renders.

```typescript
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import { getSession } from '@/lib/auth/session';

const PUBLIC_PATHS = [
  '/login',
  '/callback',
  '/api/auth'
];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Allow public paths
  if (PUBLIC_PATHS.some((p) => pathname.startsWith(p))) {
    return NextResponse.next();
  }

  // Check authentication
  const session = await getSession(request);
  if (!session) {
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('redirect', pathname);
    return NextResponse.redirect(loginUrl);
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|assets).*)']
};
```

---

## 2. Route Group Structure

Next.js route groups (parenthesized folders) organize routes without affecting the URL path. The M-SACCO app uses two primary groups.

```
src/app/
├── (auth)/                     # No shell, centered card layout
│   ├── layout.tsx              # Minimal layout: centered container
│   ├── login/
│   │   └── page.tsx
│   └── callback/
│       └── page.tsx
├── (dashboard)/                # Shell layout with nav, breadcrumbs, footer
│   ├── layout.tsx              # ShellComponent equivalent
│   ├── home/
│   │   └── page.tsx
│   ├── dashboard/
│   │   └── page.tsx
│   ├── clients/
│   │   └── ...
│   ├── groups/
│   │   └── ...
│   ├── centers/
│   │   └── ...
│   ├── loans/
│   │   └── ...                 # Nested under clients or standalone
│   ├── savings/
│   │   └── ...
│   ├── accounting/
│   │   └── ...
│   ├── products/
│   │   └── ...
│   ├── organization/
│   │   └── ...
│   ├── system/
│   │   └── ...
│   └── ...other modules
├── layout.tsx                  # Root layout: <html>, providers
├── not-found.tsx               # Global 404
└── page.tsx                    # Redirects to /home
```

### 2.1 (auth) Group Layout

```typescript
// app/(auth)/layout.tsx
export default function AuthLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-100">
      <div className="w-full max-w-md">{children}</div>
    </div>
  );
}
```

### 2.2 (dashboard) Group Layout

This replaces the Angular `ShellComponent` that `Route.withShell()` wraps around all authenticated routes.

```typescript
// app/(dashboard)/layout.tsx
import { TopNav } from '@/components/shell/top-nav';
import { Sidebar } from '@/components/shell/sidebar';
import { Breadcrumbs } from '@/components/shell/breadcrumbs';
import { Footer } from '@/components/shell/footer';

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar />
      <div className="flex flex-1 flex-col overflow-hidden">
        <TopNav />
        <div className="flex-1 overflow-auto p-6">
          <Breadcrumbs />
          <main>{children}</main>
        </div>
        <Footer />
      </div>
    </div>
  );
}
```

---

## 3. Full Route Map

The table below maps every Angular route from all 28 routing modules to the corresponding Next.js App Router file path. Routes are grouped by module.

### 3.1 Home and Navigation

| Angular Route             | Next.js File                          | Notes                                |
| ------------------------- | ------------------------------------- | ------------------------------------ |
| `/` (redirect to `/home`) | `app/page.tsx`                        | `redirect('/home')`                  |
| `/home`                   | `app/(dashboard)/home/page.tsx`       | HomeComponent                        |
| `/dashboard`              | `app/(dashboard)/dashboard/page.tsx`  | DashboardComponent, resolves offices |
| `/navigation`             | `app/(dashboard)/navigation/page.tsx` | Office-based navigation              |

### 3.2 Login and Authentication

| Angular Route | Next.js File                   | Notes                 |
| ------------- | ------------------------------ | --------------------- |
| `/login`      | `app/(auth)/login/page.tsx`    | LoginComponent        |
| `/callback`   | `app/(auth)/callback/page.tsx` | Zitadel OIDC callback |

### 3.3 Clients Module (30+ routes)

| Angular Route                                            | Next.js File                                                                       | Notes                                                     |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `/clients`                                               | `app/(dashboard)/clients/page.tsx`                                                 | Client list                                               |
| `/clients/create`                                        | `app/(dashboard)/clients/create/page.tsx`                                          | CreateClientComponent, resolves template + address config |
| `/clients/:clientId` (redirects to general)              | `app/(dashboard)/clients/[clientId]/page.tsx`                                      | Redirects to general or renders default                   |
| `/clients/:clientId/general`                             | `app/(dashboard)/clients/[clientId]/general/page.tsx`                              | GeneralTab: accounts, charges, collateral                 |
| `/clients/:clientId/personal-data`                       | `app/(dashboard)/clients/[clientId]/personal-data/page.tsx`                        | PersonalDataTab                                           |
| `/clients/:clientId/address`                             | `app/(dashboard)/clients/[clientId]/address/page.tsx`                              | AddressTab: field config, template, data                  |
| `/clients/:clientId/family-members`                      | `app/(dashboard)/clients/[clientId]/family-members/page.tsx`                       | FamilyMembersTab                                          |
| `/clients/:clientId/family-members/add`                  | `app/(dashboard)/clients/[clientId]/family-members/add/page.tsx`                   | AddFamilyMember                                           |
| `/clients/:clientId/family-members/:familyMemberId/edit` | `app/(dashboard)/clients/[clientId]/family-members/[familyMemberId]/edit/page.tsx` | EditFamilyMember                                          |
| `/clients/:clientId/identities`                          | `app/(dashboard)/clients/[clientId]/identities/page.tsx`                           | IdentitiesTab                                             |
| `/clients/:clientId/documents`                           | `app/(dashboard)/clients/[clientId]/documents/page.tsx`                            | DocumentsTab                                              |
| `/clients/:clientId/notes`                               | `app/(dashboard)/clients/[clientId]/notes/page.tsx`                                | NotesTab                                                  |
| `/clients/:clientId/datatables/:datatableName`           | `app/(dashboard)/clients/[clientId]/datatables/[datatableName]/page.tsx`           | DatatableTab                                              |
| `/clients/:clientId/edit`                                | `app/(dashboard)/clients/[clientId]/edit/page.tsx`                                 | EditClientComponent                                       |
| `/clients/:clientId/actions/:name`                       | `app/(dashboard)/clients/[clientId]/actions/[name]/page.tsx`                       | ClientActions (activate, close, etc.)                     |
| `/clients/:clientId/charges/overview`                    | `app/(dashboard)/clients/[clientId]/charges/overview/page.tsx`                     | ChargesOverview                                           |
| `/clients/:clientId/charges/:chargeId`                   | `app/(dashboard)/clients/[clientId]/charges/[chargeId]/page.tsx`                   | ViewCharge                                                |
| `/clients/:clientId/charges/:chargeId/pay`               | `app/(dashboard)/clients/[clientId]/charges/[chargeId]/pay/page.tsx`               | PayCharge                                                 |
| `/clients/:clientId/loans-accounts/...`                  | `app/(dashboard)/clients/[clientId]/loans-accounts/...`                            | Lazy-loaded loans (see Loans section)                     |
| `/clients/:clientId/savings-accounts/...`                | `app/(dashboard)/clients/[clientId]/savings-accounts/...`                          | Lazy-loaded savings                                       |
| `/clients/:clientId/fixed-deposits-accounts/...`         | `app/(dashboard)/clients/[clientId]/fixed-deposits-accounts/...`                   | Lazy-loaded fixed deposits                                |
| `/clients/:clientId/recurring-deposits-accounts/...`     | `app/(dashboard)/clients/[clientId]/recurring-deposits-accounts/...`               | Lazy-loaded recurring deposits                            |
| `/clients/:clientId/shares-accounts/...`                 | `app/(dashboard)/clients/[clientId]/shares-accounts/...`                           | Lazy-loaded shares                                        |
| `/clients/:clientId/standing-instructions/...`           | `app/(dashboard)/clients/[clientId]/standing-instructions/...`                     | Account transfers                                         |
| `/clients/:clientId/client-collateral/...`               | `app/(dashboard)/clients/[clientId]/client-collateral/...`                         | Collaterals module                                        |

### 3.4 Groups Module

| Angular Route                                | Next.js File                                                           | Notes                                    |
| -------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------- |
| `/groups`                                    | `app/(dashboard)/groups/page.tsx`                                      | Group list                               |
| `/groups/create`                             | `app/(dashboard)/groups/create/page.tsx`                               | CreateGroup, resolves offices            |
| `/groups/:groupId` (redirects to general)    | `app/(dashboard)/groups/[groupId]/page.tsx`                            | Redirects to general                     |
| `/groups/:groupId/general`                   | `app/(dashboard)/groups/[groupId]/general/page.tsx`                    | GeneralTab: accounts, summary, GSIM/GLIM |
| `/groups/:groupId/notes`                     | `app/(dashboard)/groups/[groupId]/notes/page.tsx`                      | NotesTab                                 |
| `/groups/:groupId/committee`                 | `app/(dashboard)/groups/[groupId]/committee/page.tsx`                  | CommitteeTab                             |
| `/groups/:groupId/datatables/:datatableName` | `app/(dashboard)/groups/[groupId]/datatables/[datatableName]/page.tsx` | DatatableTab                             |
| `/groups/:groupId/edit`                      | `app/(dashboard)/groups/[groupId]/edit/page.tsx`                       | EditGroup                                |
| `/groups/:groupId/committee/add-role`        | `app/(dashboard)/groups/[groupId]/committee/add-role/page.tsx`         | AddRole                                  |
| `/groups/:groupId/actions/:action`           | `app/(dashboard)/groups/[groupId]/actions/[action]/page.tsx`           | GroupActions                             |
| `/groups/:groupId/loans-accounts/...`        | `app/(dashboard)/groups/[groupId]/loans-accounts/...`                  | Loans sub-module                         |
| `/groups/:groupId/savings-accounts/...`      | `app/(dashboard)/groups/[groupId]/savings-accounts/...`                | Savings sub-module                       |

### 3.5 Centers Module

| Angular Route                                  | Next.js File                                                             | Notes                |
| ---------------------------------------------- | ------------------------------------------------------------------------ | -------------------- |
| `/centers`                                     | `app/(dashboard)/centers/page.tsx`                                       | Center list          |
| `/centers/create`                              | `app/(dashboard)/centers/create/page.tsx`                                | CreateCenter         |
| `/centers/:centerId` (redirects to general)    | `app/(dashboard)/centers/[centerId]/page.tsx`                            | Redirects to general |
| `/centers/:centerId/general`                   | `app/(dashboard)/centers/[centerId]/general/page.tsx`                    | GeneralTab           |
| `/centers/:centerId/notes`                     | `app/(dashboard)/centers/[centerId]/notes/page.tsx`                      | NotesTab             |
| `/centers/:centerId/datatables/:datatableName` | `app/(dashboard)/centers/[centerId]/datatables/[datatableName]/page.tsx` | DatatableTab         |
| `/centers/:centerId/edit`                      | `app/(dashboard)/centers/[centerId]/edit/page.tsx`                       | EditCenter           |
| `/centers/:centerId/actions/:action`           | `app/(dashboard)/centers/[centerId]/actions/[action]/page.tsx`           | CenterActions        |

### 3.6 Loans Module

Loans are loaded both standalone (via groups/clients sub-routes) and at top-level paths.

| Angular Route                                          | Next.js File                                                      | Notes                              |
| ------------------------------------------------------ | ----------------------------------------------------------------- | ---------------------------------- |
| `.../loans-accounts/create`                            | `.../loans-accounts/create/page.tsx`                              | CreateLoansAccount (7-step wizard) |
| `.../loans-accounts/:loanId` (redirect to general)     | `.../loans-accounts/[loanId]/page.tsx`                            | Redirects to general               |
| `.../loans-accounts/:loanId/general`                   | `.../loans-accounts/[loanId]/general/page.tsx`                    | GeneralTab                         |
| `.../loans-accounts/:loanId/dashboard`                 | `.../loans-accounts/[loanId]/dashboard/page.tsx`                  | LoanAccountDashboard               |
| `.../loans-accounts/:loanId/accountdetail`             | `.../loans-accounts/[loanId]/accountdetail/page.tsx`              | AccountDetails                     |
| `.../loans-accounts/:loanId/repayment-schedule`        | `.../loans-accounts/[loanId]/repayment-schedule/page.tsx`         | RepaymentScheduleTab               |
| `.../loans-accounts/:loanId/original-schedule`         | `.../loans-accounts/[loanId]/original-schedule/page.tsx`          | OriginalScheduleTab                |
| `.../loans-accounts/:loanId/transactions`              | `.../loans-accounts/[loanId]/transactions/page.tsx`               | TransactionsTab                    |
| `.../loans-accounts/:loanId/transactions/export`       | `.../loans-accounts/[loanId]/transactions/export/page.tsx`        | ExportTransactions                 |
| `.../loans-accounts/:loanId/charges`                   | `.../loans-accounts/[loanId]/charges/page.tsx`                    | ChargesTab                         |
| `.../loans-accounts/:loanId/notes`                     | `.../loans-accounts/[loanId]/notes/page.tsx`                      | NotesTab                           |
| `.../loans-accounts/:loanId/loan-documents`            | `.../loans-accounts/[loanId]/loan-documents/page.tsx`             | LoanDocumentsTab                   |
| `.../loans-accounts/:loanId/loan-collateral`           | `.../loans-accounts/[loanId]/loan-collateral/page.tsx`            | LoanCollateralTab                  |
| `.../loans-accounts/:loanId/delinquencytags`           | `.../loans-accounts/[loanId]/delinquencytags/page.tsx`            | DelinquencyTagsTab                 |
| `.../loans-accounts/:loanId/loan-reschedules`          | `.../loans-accounts/[loanId]/loan-reschedules/page.tsx`           | RescheduleLoanTab                  |
| `.../loans-accounts/:loanId/term-variations`           | `.../loans-accounts/[loanId]/term-variations/page.tsx`            | LoanTermVariationsTab              |
| `.../loans-accounts/:loanId/deferred-income`           | `.../loans-accounts/[loanId]/deferred-income/page.tsx`            | LoanDeferredIncomeTab              |
| `.../loans-accounts/:loanId/buy-down-fees`             | `.../loans-accounts/[loanId]/buy-down-fees/page.tsx`              | LoanBuyDownFeesTab                 |
| `.../loans-accounts/:loanId/originators`               | `.../loans-accounts/[loanId]/originators/page.tsx`                | LoanOriginatorsTab                 |
| `.../loans-accounts/:loanId/floating-interest-rates`   | `.../loans-accounts/[loanId]/floating-interest-rates/page.tsx`    | FloatingInterestRates              |
| `.../loans-accounts/:loanId/loan-tranche-details`      | `.../loans-accounts/[loanId]/loan-tranche-details/page.tsx`       | LoanTrancheDetails                 |
| `.../loans-accounts/:loanId/overdue-charges`           | `.../loans-accounts/[loanId]/overdue-charges/page.tsx`            | OverdueChargesTab                  |
| `.../loans-accounts/:loanId/standing-instruction`      | `.../loans-accounts/[loanId]/standing-instruction/page.tsx`       | StandingInstructionsTab            |
| `.../loans-accounts/:loanId/external-asset-owner`      | `.../loans-accounts/[loanId]/external-asset-owner/page.tsx`       | ExternalAssetOwnerTab              |
| `.../loans-accounts/:loanId/datatables/:datatableName` | `.../loans-accounts/[loanId]/datatables/[datatableName]/page.tsx` | DatatableTab                       |
| `.../loans-accounts/:loanId/edit-loans-account`        | `.../loans-accounts/[loanId]/edit/page.tsx`                       | EditLoansAccount                   |
| `.../loans-accounts/:loanId/actions/:action`           | `.../loans-accounts/[loanId]/actions/[action]/page.tsx`           | LoanAccountActions                 |
| `.../loans-accounts/:loanId/transactions/:id`          | `.../loans-accounts/[loanId]/transactions/[id]/page.tsx`          | ViewTransaction                    |
| `.../loans-accounts/:loanId/transactions/:id/edit`     | `.../loans-accounts/[loanId]/transactions/[id]/edit/page.tsx`     | EditTransaction                    |
| `.../loans-accounts/:loanId/transactions/:id/reciept`  | `.../loans-accounts/[loanId]/transactions/[id]/receipt/page.tsx`  | ViewReceipt (fix typo)             |
| `.../loans-accounts/:loanId/charges/:id`               | `.../loans-accounts/[loanId]/charges/[id]/page.tsx`               | ViewCharge                         |
| `.../loans-accounts/:loanId/charges/:id/adjustment`    | `.../loans-accounts/[loanId]/charges/[id]/adjustment/page.tsx`    | AdjustLoanCharge                   |
| `.../loans-accounts/:loanId/transfer-funds/...`        | `.../loans-accounts/[loanId]/transfer-funds/...`                  | AccountTransfers sub-module        |

**GLIM (Group Loan Individual Monitoring):**

| Angular Route              | Next.js File                                                      | Notes             |
| -------------------------- | ----------------------------------------------------------------- | ----------------- |
| `.../glim-account/create`  | `app/(dashboard)/groups/[groupId]/glim-account/create/page.tsx`   | CreateGlimAccount |
| `.../glim-account/:glimId` | `app/(dashboard)/groups/[groupId]/glim-account/[glimId]/page.tsx` | GlimAccountView   |

### 3.7 Savings Module

| Angular Route                                         | Next.js File                                                     | Notes                     |
| ----------------------------------------------------- | ---------------------------------------------------------------- | ------------------------- |
| `.../savings-accounts/create`                         | `.../savings-accounts/create/page.tsx`                           | CreateSavingsAccount      |
| `.../savings-accounts/:savingAccountId`               | `.../savings-accounts/[savingAccountId]/page.tsx`                | SavingsAccountView (tabs) |
| `.../savings-accounts/:savingAccountId/transactions`  | `.../savings-accounts/[savingAccountId]/transactions/page.tsx`   | TransactionsTab           |
| `.../savings-accounts/:savingAccountId/charges`       | `.../savings-accounts/[savingAccountId]/charges/page.tsx`        | ChargesTab                |
| `.../savings-accounts/:savingAccountId/edit`          | `.../savings-accounts/[savingAccountId]/edit/page.tsx`           | EditSavingsAccount        |
| `.../savings-accounts/:savingAccountId/actions/:name` | `.../savings-accounts/[savingAccountId]/actions/[name]/page.tsx` | SavingAccountActions      |

### 3.8 Fixed Deposit and Recurring Deposit Accounts

Follow the same pattern as Savings under `.../fixed-deposits-accounts/` and `.../recurring-deposits-accounts/`.

### 3.9 Shares Module

Follow the same pattern as Savings under `.../shares-accounts/`.

### 3.10 Accounting Module

| Angular Route                                             | Next.js File                                                                       | Notes                       |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------- | --------------------------- |
| `/accounting`                                             | `app/(dashboard)/accounting/page.tsx`                                              | Accounting landing          |
| `/accounting/journal-entries`                             | `app/(dashboard)/accounting/journal-entries/page.tsx`                              | SearchJournalEntry          |
| `/accounting/journal-entries/create`                      | `app/(dashboard)/accounting/journal-entries/create/page.tsx`                       | CreateJournalEntry          |
| `/accounting/journal-entries/frequent-postings`           | `app/(dashboard)/accounting/journal-entries/frequent-postings/page.tsx`            | FrequentPostings            |
| `/accounting/journal-entries/transactions/view/:id`       | `app/(dashboard)/accounting/journal-entries/transactions/view/[id]/page.tsx`       | ViewTransaction             |
| `/accounting/chart-of-accounts`                           | `app/(dashboard)/accounting/chart-of-accounts/page.tsx`                            | ChartOfAccounts tree        |
| `/accounting/chart-of-accounts/gl-accounts/create`        | `app/(dashboard)/accounting/chart-of-accounts/gl-accounts/create/page.tsx`         | CreateGlAccount             |
| `/accounting/chart-of-accounts/gl-accounts/view/:id`      | `app/(dashboard)/accounting/chart-of-accounts/gl-accounts/view/[id]/page.tsx`      | ViewGlAccount               |
| `/accounting/chart-of-accounts/gl-accounts/view/:id/edit` | `app/(dashboard)/accounting/chart-of-accounts/gl-accounts/view/[id]/edit/page.tsx` | EditGlAccount               |
| `/accounting/closing-entries`                             | `app/(dashboard)/accounting/closing-entries/page.tsx`                              | ClosingEntries list         |
| `/accounting/closing-entries/create`                      | `app/(dashboard)/accounting/closing-entries/create/page.tsx`                       | CreateClosure               |
| `/accounting/closing-entries/view/:id`                    | `app/(dashboard)/accounting/closing-entries/view/[id]/page.tsx`                    | ViewClosure                 |
| `/accounting/closing-entries/view/:id/edit`               | `app/(dashboard)/accounting/closing-entries/view/[id]/edit/page.tsx`               | EditClosure                 |
| `/accounting/accounting-rules`                            | `app/(dashboard)/accounting/accounting-rules/page.tsx`                             | AccountingRules list        |
| `/accounting/accounting-rules/create`                     | `app/(dashboard)/accounting/accounting-rules/create/page.tsx`                      | CreateRule                  |
| `/accounting/accounting-rules/view/:id`                   | `app/(dashboard)/accounting/accounting-rules/view/[id]/page.tsx`                   | ViewRule                    |
| `/accounting/accounting-rules/view/:id/edit`              | `app/(dashboard)/accounting/accounting-rules/view/[id]/edit/page.tsx`              | EditRule                    |
| `/accounting/financial-activity-mappings`                 | `app/(dashboard)/accounting/financial-activity-mappings/page.tsx`                  | List                        |
| `/accounting/financial-activity-mappings/create`          | `app/(dashboard)/accounting/financial-activity-mappings/create/page.tsx`           | Create                      |
| `/accounting/financial-activity-mappings/view/:id`        | `app/(dashboard)/accounting/financial-activity-mappings/view/[id]/page.tsx`        | View                        |
| `/accounting/financial-activity-mappings/view/:id/edit`   | `app/(dashboard)/accounting/financial-activity-mappings/view/[id]/edit/page.tsx`   | Edit                        |
| `/accounting/migrate-opening-balances`                    | `app/(dashboard)/accounting/migrate-opening-balances/page.tsx`                     | MigrateOpeningBalances      |
| `/accounting/periodic-accruals`                           | `app/(dashboard)/accounting/periodic-accruals/page.tsx`                            | PeriodicAccruals            |
| `/accounting/provisioning-entries`                        | `app/(dashboard)/accounting/provisioning-entries/page.tsx`                         | List                        |
| `/accounting/provisioning-entries/create`                 | `app/(dashboard)/accounting/provisioning-entries/create/page.tsx`                  | Create                      |
| `/accounting/provisioning-entries/view/:id`               | `app/(dashboard)/accounting/provisioning-entries/view/[id]/page.tsx`               | View                        |
| `/journal-entry/view/:id`                                 | `app/(dashboard)/journal-entry/view/[id]/page.tsx`                                 | ViewJournalEntryTransaction |
| `/journal-entry/view-transfer/:transferId`                | `app/(dashboard)/journal-entry/view-transfer/[transferId]/page.tsx`                | ViewTransferJournalEntry    |

### 3.11 Products Module

| Angular Route                                                        | Next.js File                                                                                  | Notes                   |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- | ----------------------- |
| `/products`                                                          | `app/(dashboard)/products/page.tsx`                                                           | Products landing        |
| `/products/loan-products`                                            | `app/(dashboard)/products/loan-products/page.tsx`                                             | LoanProducts list       |
| `/products/loan-products/create`                                     | `app/(dashboard)/products/loan-products/create/page.tsx`                                      | Create (10-step wizard) |
| `/products/loan-products/:productId` (redirect general)              | `app/(dashboard)/products/loan-products/[productId]/page.tsx`                                 | Redirect                |
| `/products/loan-products/:productId/general`                         | `app/(dashboard)/products/loan-products/[productId]/general/page.tsx`                         | GeneralTab              |
| `/products/loan-products/:productId/datatables/:name`                | `app/(dashboard)/products/loan-products/[productId]/datatables/[name]/page.tsx`               | DatatableTab            |
| `/products/loan-products/:productId/edit`                            | `app/(dashboard)/products/loan-products/[productId]/edit/page.tsx`                            | Edit (10-step wizard)   |
| `/products/saving-products`                                          | `app/(dashboard)/products/saving-products/page.tsx`                                           | List                    |
| `/products/saving-products/create`                                   | `app/(dashboard)/products/saving-products/create/page.tsx`                                    | Create                  |
| `/products/saving-products/:productId`                               | `app/(dashboard)/products/saving-products/[productId]/page.tsx`                               | View (with tabs)        |
| `/products/saving-products/:productId/edit`                          | `app/(dashboard)/products/saving-products/[productId]/edit/page.tsx`                          | Edit                    |
| `/products/share-products`                                           | `app/(dashboard)/products/share-products/page.tsx`                                            | List                    |
| `/products/share-products/create`                                    | `app/(dashboard)/products/share-products/create/page.tsx`                                     | Create                  |
| `/products/share-products/:productId`                                | `app/(dashboard)/products/share-products/[productId]/page.tsx`                                | View (with tabs)        |
| `/products/share-products/:productId/edit`                           | `app/(dashboard)/products/share-products/[productId]/edit/page.tsx`                           | Edit                    |
| `/products/share-products/:productId/dividends`                      | `app/(dashboard)/products/share-products/[productId]/dividends/page.tsx`                      | Dividends list          |
| `/products/share-products/:productId/dividends/create`               | `app/(dashboard)/products/share-products/[productId]/dividends/create/page.tsx`               | Create dividend         |
| `/products/share-products/:productId/dividends/:dividendId`          | `app/(dashboard)/products/share-products/[productId]/dividends/[dividendId]/page.tsx`         | View dividend           |
| `/products/recurring-deposit-products`                               | `app/(dashboard)/products/recurring-deposit-products/page.tsx`                                | List                    |
| `/products/recurring-deposit-products/create`                        | `app/(dashboard)/products/recurring-deposit-products/create/page.tsx`                         | Create                  |
| `/products/recurring-deposit-products/:productId`                    | `app/(dashboard)/products/recurring-deposit-products/[productId]/page.tsx`                    | View (tabs)             |
| `/products/recurring-deposit-products/:productId/edit`               | `app/(dashboard)/products/recurring-deposit-products/[productId]/edit/page.tsx`               | Edit                    |
| `/products/fixed-deposit-products`                                   | `app/(dashboard)/products/fixed-deposit-products/page.tsx`                                    | List                    |
| `/products/fixed-deposit-products/create`                            | `app/(dashboard)/products/fixed-deposit-products/create/page.tsx`                             | Create                  |
| `/products/fixed-deposit-products/:productId`                        | `app/(dashboard)/products/fixed-deposit-products/[productId]/page.tsx`                        | View (tabs)             |
| `/products/fixed-deposit-products/:productId/edit`                   | `app/(dashboard)/products/fixed-deposit-products/[productId]/edit/page.tsx`                   | Edit                    |
| `/products/charges`                                                  | `app/(dashboard)/products/charges/page.tsx`                                                   | List                    |
| `/products/charges/create`                                           | `app/(dashboard)/products/charges/create/page.tsx`                                            | Create                  |
| `/products/charges/:id`                                              | `app/(dashboard)/products/charges/[id]/page.tsx`                                              | View                    |
| `/products/charges/:id/edit`                                         | `app/(dashboard)/products/charges/[id]/edit/page.tsx`                                         | Edit                    |
| `/products/collaterals`                                              | `app/(dashboard)/products/collaterals/page.tsx`                                               | List                    |
| `/products/collaterals/create`                                       | `app/(dashboard)/products/collaterals/create/page.tsx`                                        | Create                  |
| `/products/collaterals/:id`                                          | `app/(dashboard)/products/collaterals/[id]/page.tsx`                                          | View                    |
| `/products/collaterals/:id/edit`                                     | `app/(dashboard)/products/collaterals/[id]/edit/page.tsx`                                     | Edit                    |
| `/products/floating-rates`                                           | `app/(dashboard)/products/floating-rates/page.tsx`                                            | List                    |
| `/products/floating-rates/create`                                    | `app/(dashboard)/products/floating-rates/create/page.tsx`                                     | Create                  |
| `/products/floating-rates/:id`                                       | `app/(dashboard)/products/floating-rates/[id]/page.tsx`                                       | View                    |
| `/products/floating-rates/:id/edit`                                  | `app/(dashboard)/products/floating-rates/[id]/edit/page.tsx`                                  | Edit                    |
| `/products/products-mix`                                             | `app/(dashboard)/products/products-mix/page.tsx`                                              | List                    |
| `/products/products-mix/create`                                      | `app/(dashboard)/products/products-mix/create/page.tsx`                                       | Create                  |
| `/products/products-mix/:id`                                         | `app/(dashboard)/products/products-mix/[id]/page.tsx`                                         | View                    |
| `/products/products-mix/:id/edit`                                    | `app/(dashboard)/products/products-mix/[id]/edit/page.tsx`                                    | Edit                    |
| `/products/tax-configurations`                                       | `app/(dashboard)/products/tax-configurations/page.tsx`                                        | Landing                 |
| `/products/tax-configurations/tax-components`                        | `app/(dashboard)/products/tax-configurations/tax-components/page.tsx`                         | List                    |
| `/products/tax-configurations/tax-components/create`                 | `app/(dashboard)/products/tax-configurations/tax-components/create/page.tsx`                  | Create                  |
| `/products/tax-configurations/tax-components/:id`                    | `app/(dashboard)/products/tax-configurations/tax-components/[id]/page.tsx`                    | View                    |
| `/products/tax-configurations/tax-components/:id/edit`               | `app/(dashboard)/products/tax-configurations/tax-components/[id]/edit/page.tsx`               | Edit                    |
| `/products/tax-configurations/tax-groups`                            | `app/(dashboard)/products/tax-configurations/tax-groups/page.tsx`                             | List                    |
| `/products/tax-configurations/tax-groups/create`                     | `app/(dashboard)/products/tax-configurations/tax-groups/create/page.tsx`                      | Create                  |
| `/products/tax-configurations/tax-groups/:id`                        | `app/(dashboard)/products/tax-configurations/tax-groups/[id]/page.tsx`                        | View                    |
| `/products/tax-configurations/tax-groups/:id/edit`                   | `app/(dashboard)/products/tax-configurations/tax-groups/[id]/edit/page.tsx`                   | Edit                    |
| `/products/delinquency-bucket-configurations`                        | `app/(dashboard)/products/delinquency-bucket-configurations/page.tsx`                         | Landing                 |
| `/products/delinquency-bucket-configurations/ranges`                 | `app/(dashboard)/products/delinquency-bucket-configurations/ranges/page.tsx`                  | List                    |
| `/products/delinquency-bucket-configurations/ranges/create`          | `app/(dashboard)/products/delinquency-bucket-configurations/ranges/create/page.tsx`           | Create                  |
| `/products/delinquency-bucket-configurations/ranges/:rangeId`        | `app/(dashboard)/products/delinquency-bucket-configurations/ranges/[rangeId]/page.tsx`        | View                    |
| `/products/delinquency-bucket-configurations/ranges/:rangeId/edit`   | `app/(dashboard)/products/delinquency-bucket-configurations/ranges/[rangeId]/edit/page.tsx`   | Edit                    |
| `/products/delinquency-bucket-configurations/buckets`                | `app/(dashboard)/products/delinquency-bucket-configurations/buckets/page.tsx`                 | List                    |
| `/products/delinquency-bucket-configurations/buckets/create`         | `app/(dashboard)/products/delinquency-bucket-configurations/buckets/create/page.tsx`          | Create                  |
| `/products/delinquency-bucket-configurations/buckets/:bucketId`      | `app/(dashboard)/products/delinquency-bucket-configurations/buckets/[bucketId]/page.tsx`      | View                    |
| `/products/delinquency-bucket-configurations/buckets/:bucketId/edit` | `app/(dashboard)/products/delinquency-bucket-configurations/buckets/[bucketId]/edit/page.tsx` | Edit                    |

### 3.12 Organization Module

| Angular Route                                          | Next.js File                                                                     | Notes                |
| ------------------------------------------------------ | -------------------------------------------------------------------------------- | -------------------- |
| `/organization`                                        | `app/(dashboard)/organization/page.tsx`                                          | Landing page         |
| `/organization/offices`                                | `app/(dashboard)/organization/offices/page.tsx`                                  | List                 |
| `/organization/offices/create`                         | `app/(dashboard)/organization/offices/create/page.tsx`                           | Create               |
| `/organization/offices/:officeId` (redirect general)   | `app/(dashboard)/organization/offices/[officeId]/page.tsx`                       | View with tabs       |
| `/organization/offices/:officeId/general`              | `app/(dashboard)/organization/offices/[officeId]/general/page.tsx`               | GeneralTab           |
| `/organization/offices/:officeId/datatables/:name`     | `app/(dashboard)/organization/offices/[officeId]/datatables/[name]/page.tsx`     | DatatableTab         |
| `/organization/offices/:officeId/edit`                 | `app/(dashboard)/organization/offices/[officeId]/edit/page.tsx`                  | Edit                 |
| `/organization/employees`                              | `app/(dashboard)/organization/employees/page.tsx`                                | List                 |
| `/organization/employees/create`                       | `app/(dashboard)/organization/employees/create/page.tsx`                         | Create               |
| `/organization/employees/:id`                          | `app/(dashboard)/organization/employees/[id]/page.tsx`                           | View                 |
| `/organization/employees/:id/edit`                     | `app/(dashboard)/organization/employees/[id]/edit/page.tsx`                      | Edit                 |
| `/organization/currencies`                             | `app/(dashboard)/organization/currencies/page.tsx`                               | View                 |
| `/organization/currencies/manage`                      | `app/(dashboard)/organization/currencies/manage/page.tsx`                        | ManageCurrencies     |
| `/organization/sms-campaigns`                          | `app/(dashboard)/organization/sms-campaigns/page.tsx`                            | List                 |
| `/organization/sms-campaigns/create`                   | `app/(dashboard)/organization/sms-campaigns/create/page.tsx`                     | Create               |
| `/organization/sms-campaigns/:id`                      | `app/(dashboard)/organization/sms-campaigns/[id]/page.tsx`                       | View                 |
| `/organization/sms-campaigns/:id/edit`                 | `app/(dashboard)/organization/sms-campaigns/[id]/edit/page.tsx`                  | Edit                 |
| `/organization/tellers`                                | `app/(dashboard)/organization/tellers/page.tsx`                                  | List                 |
| `/organization/tellers/create`                         | `app/(dashboard)/organization/tellers/create/page.tsx`                           | Create               |
| `/organization/tellers/:id`                            | `app/(dashboard)/organization/tellers/[id]/page.tsx`                             | View                 |
| `/organization/tellers/:id/edit`                       | `app/(dashboard)/organization/tellers/[id]/edit/page.tsx`                        | Edit                 |
| `/organization/tellers/:id/cashiers`                   | `app/(dashboard)/organization/tellers/[id]/cashiers/page.tsx`                    | Cashier list         |
| `/organization/tellers/:id/cashiers/create`            | `app/(dashboard)/organization/tellers/[id]/cashiers/create/page.tsx`             | Create cashier       |
| `/organization/tellers/:id/cashiers/:cid`              | `app/(dashboard)/organization/tellers/[id]/cashiers/[cid]/page.tsx`              | View cashier         |
| `/organization/tellers/:id/cashiers/:cid/edit`         | `app/(dashboard)/organization/tellers/[id]/cashiers/[cid]/edit/page.tsx`         | Edit cashier         |
| `/organization/tellers/:id/cashiers/:cid/transactions` | `app/(dashboard)/organization/tellers/[id]/cashiers/[cid]/transactions/page.tsx` | Transactions         |
| `/organization/tellers/:id/cashiers/:cid/settle`       | `app/(dashboard)/organization/tellers/[id]/cashiers/[cid]/settle/page.tsx`       | SettleCash           |
| `/organization/tellers/:id/cashiers/:cid/allocate`     | `app/(dashboard)/organization/tellers/[id]/cashiers/[cid]/allocate/page.tsx`     | AllocateCash         |
| `/organization/payment-types`                          | `app/(dashboard)/organization/payment-types/page.tsx`                            | List                 |
| `/organization/payment-types/create`                   | `app/(dashboard)/organization/payment-types/create/page.tsx`                     | Create               |
| `/organization/payment-types/:id/edit`                 | `app/(dashboard)/organization/payment-types/[id]/edit/page.tsx`                  | Edit                 |
| `/organization/password-preferences`                   | `app/(dashboard)/organization/password-preferences/page.tsx`                     | View/Edit            |
| `/organization/working-days`                           | `app/(dashboard)/organization/working-days/page.tsx`                             | View/Edit            |
| `/organization/holidays`                               | `app/(dashboard)/organization/holidays/page.tsx`                                 | List                 |
| `/organization/holidays/create`                        | `app/(dashboard)/organization/holidays/create/page.tsx`                          | Create               |
| `/organization/holidays/:id`                           | `app/(dashboard)/organization/holidays/[id]/page.tsx`                            | View                 |
| `/organization/holidays/:id/edit`                      | `app/(dashboard)/organization/holidays/[id]/edit/page.tsx`                       | Edit                 |
| `/organization/adhoc-query`                            | `app/(dashboard)/organization/adhoc-query/page.tsx`                              | List                 |
| `/organization/adhoc-query/create`                     | `app/(dashboard)/organization/adhoc-query/create/page.tsx`                       | Create               |
| `/organization/adhoc-query/:id`                        | `app/(dashboard)/organization/adhoc-query/[id]/page.tsx`                         | View                 |
| `/organization/adhoc-query/:id/edit`                   | `app/(dashboard)/organization/adhoc-query/[id]/edit/page.tsx`                    | Edit                 |
| `/organization/provisioning-criteria`                  | `app/(dashboard)/organization/provisioning-criteria/page.tsx`                    | List                 |
| `/organization/provisioning-criteria/create`           | `app/(dashboard)/organization/provisioning-criteria/create/page.tsx`             | Create               |
| `/organization/provisioning-criteria/:id`              | `app/(dashboard)/organization/provisioning-criteria/[id]/page.tsx`               | View                 |
| `/organization/provisioning-criteria/:id/edit`         | `app/(dashboard)/organization/provisioning-criteria/[id]/edit/page.tsx`          | Edit                 |
| `/organization/bulk-import`                            | `app/(dashboard)/organization/bulk-import/page.tsx`                              | List                 |
| `/organization/bulk-import/:import-name`               | `app/(dashboard)/organization/bulk-import/[importName]/page.tsx`                 | ViewBulkImport       |
| `/organization/bulkloan`                               | `app/(dashboard)/organization/bulkloan/page.tsx`                                 | BulkLoanReassignment |
| `/organization/entity-data-table-checks`               | `app/(dashboard)/organization/entity-data-table-checks/page.tsx`                 | List                 |
| `/organization/entity-data-table-checks/create`        | `app/(dashboard)/organization/entity-data-table-checks/create/page.tsx`          | Create               |
| `/organization/standing-instructions-history`          | `app/(dashboard)/organization/standing-instructions-history/page.tsx`            | View                 |
| `/organization/fund-mapping`                           | `app/(dashboard)/organization/fund-mapping/page.tsx`                             | FundMapping          |
| `/organization/investors`                              | `app/(dashboard)/organization/investors/page.tsx`                                | Investors            |
| `/organization/manage-funds`                           | `app/(dashboard)/organization/manage-funds/page.tsx`                             | List                 |
| `/organization/manage-funds/create`                    | `app/(dashboard)/organization/manage-funds/create/page.tsx`                      | Create               |
| `/organization/manage-funds/:id`                       | `app/(dashboard)/organization/manage-funds/[id]/page.tsx`                        | View                 |
| `/organization/manage-funds/:id/edit`                  | `app/(dashboard)/organization/manage-funds/[id]/edit/page.tsx`                   | Edit                 |
| `/organization/manage-loan-originators`                | `app/(dashboard)/organization/manage-loan-originators/page.tsx`                  | List                 |
| `/organization/manage-loan-originators/create`         | `app/(dashboard)/organization/manage-loan-originators/create/page.tsx`           | Create               |
| `/organization/manage-loan-originators/:id`            | `app/(dashboard)/organization/manage-loan-originators/[id]/page.tsx`             | View                 |
| `/organization/manage-loan-originators/:id/edit`       | `app/(dashboard)/organization/manage-loan-originators/[id]/edit/page.tsx`        | Edit                 |

### 3.13 System Module

| Angular Route                                 | Next.js File                                                           | Notes                        |
| --------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------- |
| `/system`                                     | `app/(dashboard)/system/page.tsx`                                      | Landing                      |
| `/system/codes`                               | `app/(dashboard)/system/codes/page.tsx`                                | List                         |
| `/system/codes/create`                        | `app/(dashboard)/system/codes/create/page.tsx`                         | Create                       |
| `/system/codes/:id`                           | `app/(dashboard)/system/codes/[id]/page.tsx`                           | View (with code values)      |
| `/system/codes/:id/edit`                      | `app/(dashboard)/system/codes/[id]/edit/page.tsx`                      | Edit                         |
| `/system/data-tables`                         | `app/(dashboard)/system/data-tables/page.tsx`                          | List                         |
| `/system/data-tables/create`                  | `app/(dashboard)/system/data-tables/create/page.tsx`                   | Create                       |
| `/system/data-tables/:datatableName`          | `app/(dashboard)/system/data-tables/[datatableName]/page.tsx`          | View                         |
| `/system/data-tables/:datatableName/edit`     | `app/(dashboard)/system/data-tables/[datatableName]/edit/page.tsx`     | Edit                         |
| `/system/hooks`                               | `app/(dashboard)/system/hooks/page.tsx`                                | List                         |
| `/system/hooks/create`                        | `app/(dashboard)/system/hooks/create/page.tsx`                         | Create                       |
| `/system/hooks/:id`                           | `app/(dashboard)/system/hooks/[id]/page.tsx`                           | View                         |
| `/system/hooks/:id/edit`                      | `app/(dashboard)/system/hooks/[id]/edit/page.tsx`                      | Edit                         |
| `/system/roles-and-permissions`               | `app/(dashboard)/system/roles-and-permissions/page.tsx`                | List                         |
| `/system/roles-and-permissions/add`           | `app/(dashboard)/system/roles-and-permissions/add/page.tsx`            | AddRole                      |
| `/system/roles-and-permissions/:id`           | `app/(dashboard)/system/roles-and-permissions/[id]/page.tsx`           | ViewRole                     |
| `/system/roles-and-permissions/:id/edit`      | `app/(dashboard)/system/roles-and-permissions/[id]/edit/page.tsx`      | EditRole                     |
| `/system/surveys`                             | `app/(dashboard)/system/surveys/page.tsx`                              | List                         |
| `/system/surveys/create`                      | `app/(dashboard)/system/surveys/create/page.tsx`                       | Create                       |
| `/system/surveys/:id`                         | `app/(dashboard)/system/surveys/[id]/page.tsx`                         | View                         |
| `/system/surveys/:id/edit`                    | `app/(dashboard)/system/surveys/[id]/edit/page.tsx`                    | Edit                         |
| `/system/manage-jobs`                         | `app/(dashboard)/system/manage-jobs/page.tsx`                          | ManageJobs (scheduler + COB) |
| `/system/manage-jobs/:id`                     | `app/(dashboard)/system/manage-jobs/[id]/page.tsx`                     | ViewSchedulerJob             |
| `/system/manage-jobs/:id/edit`                | `app/(dashboard)/system/manage-jobs/[id]/edit/page.tsx`                | EditSchedulerJob             |
| `/system/manage-jobs/:id/viewhistory`         | `app/(dashboard)/system/manage-jobs/[id]/viewhistory/page.tsx`         | ViewHistory                  |
| `/system/configurations`                      | `app/(dashboard)/system/configurations/page.tsx`                       | GlobalConfigurations         |
| `/system/configurations/:id/edit`             | `app/(dashboard)/system/configurations/[id]/edit/page.tsx`             | EditConfiguration            |
| `/system/account-number-preferences`          | `app/(dashboard)/system/account-number-preferences/page.tsx`           | List                         |
| `/system/account-number-preferences/create`   | `app/(dashboard)/system/account-number-preferences/create/page.tsx`    | Create                       |
| `/system/account-number-preferences/:id`      | `app/(dashboard)/system/account-number-preferences/[id]/page.tsx`      | View                         |
| `/system/account-number-preferences/:id/edit` | `app/(dashboard)/system/account-number-preferences/[id]/edit/page.tsx` | Edit                         |
| `/system/reports`                             | `app/(dashboard)/system/reports/page.tsx`                              | List                         |
| `/system/reports/create`                      | `app/(dashboard)/system/reports/create/page.tsx`                       | Create                       |
| `/system/reports/:id`                         | `app/(dashboard)/system/reports/[id]/page.tsx`                         | View                         |
| `/system/reports/:id/edit`                    | `app/(dashboard)/system/reports/[id]/edit/page.tsx`                    | Edit                         |
| `/system/external-services`                   | `app/(dashboard)/system/external-services/page.tsx`                    | Landing                      |
| `/system/external-services/amazon-s3`         | `app/(dashboard)/system/external-services/amazon-s3/page.tsx`          | View                         |
| `/system/external-services/amazon-s3/edit`    | `app/(dashboard)/system/external-services/amazon-s3/edit/page.tsx`     | Edit                         |
| `/system/external-services/email`             | `app/(dashboard)/system/external-services/email/page.tsx`              | View                         |
| `/system/external-services/email/edit`        | `app/(dashboard)/system/external-services/email/edit/page.tsx`         | Edit                         |
| `/system/external-services/sms`               | `app/(dashboard)/system/external-services/sms/page.tsx`                | View                         |
| `/system/external-services/sms/edit`          | `app/(dashboard)/system/external-services/sms/edit/page.tsx`           | Edit                         |
| `/system/external-services/notification`      | `app/(dashboard)/system/external-services/notification/page.tsx`       | View                         |
| `/system/external-services/notification/edit` | `app/(dashboard)/system/external-services/notification/edit/page.tsx`  | Edit                         |
| `/system/external-events`                     | `app/(dashboard)/system/external-events/page.tsx`                      | ManageExternalEvents         |
| `/system/entity-to-entity-mapping`            | `app/(dashboard)/system/entity-to-entity-mapping/page.tsx`             | EntityMapping                |
| `/system/configure-mc-tasks`                  | `app/(dashboard)/system/configure-mc-tasks/page.tsx`                   | MakerCheckerTasks            |
| `/system/audit-trails`                        | `app/(dashboard)/system/audit-trails/page.tsx`                         | AuditTrails search           |
| `/system/audit-trails/:id`                    | `app/(dashboard)/system/audit-trails/[id]/page.tsx`                    | ViewAudit                    |
| `/system/system-information`                  | `app/(dashboard)/system/system-information/page.tsx`                   | SystemInfo                   |
| `/system/about-us`                            | `app/(dashboard)/system/about-us/page.tsx`                             | AboutUs                      |

### 3.14 Other Modules

| Angular Route        | Next.js File                                 | Notes               |
| -------------------- | -------------------------------------------- | ------------------- |
| `/search`            | `app/(dashboard)/search/page.tsx`            | Global search       |
| `/notifications`     | `app/(dashboard)/notifications/page.tsx`     | Notifications       |
| `/profile`           | `app/(dashboard)/profile/page.tsx`           | User profile        |
| `/settings`          | `app/(dashboard)/settings/page.tsx`          | App settings        |
| `/tasks`             | `app/(dashboard)/tasks/page.tsx`             | Checker tasks       |
| `/templates`         | `app/(dashboard)/templates/page.tsx`         | Template management |
| `/users`             | `app/(dashboard)/users/page.tsx`             | User management     |
| `/collections`       | `app/(dashboard)/collections/page.tsx`       | Collections sheet   |
| `/reports/:id`       | `app/(dashboard)/reports/[id]/page.tsx`      | Run reports         |
| `/account-transfers` | `app/(dashboard)/account-transfers/page.tsx` | Account transfers   |
| `/remittances`       | `app/(dashboard)/remittances/page.tsx`       | Remittances         |

---

## 4. Layout Hierarchy

### 4.1 Root Layout

```typescript
// app/layout.tsx
import { Providers } from '@/components/providers';

export const metadata = { title: { template: '%s | M-SACCO', default: 'M-SACCO' } };

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

The `<Providers>` component wraps: `QueryClientProvider`, `ThemeProvider`, `AuthProvider`, `NextIntlClientProvider`.

### 4.2 Entity Detail Layouts

Entities with tabs (clients, groups, centers, loan accounts, saving accounts, loan products, etc.) get their own nested layout that displays entity-level information and tab navigation.

```typescript
// app/(dashboard)/clients/[clientId]/layout.tsx
import { fineract } from '@/lib/api/fineract';
import { ClientHeader } from '@/components/clients/client-header';
import { ClientTabs } from '@/components/clients/client-tabs';

interface Props {
  children: React.ReactNode;
  params: Promise<{ clientId: string }>;
}

export default async function ClientLayout({ children, params }: Props) {
  const { clientId } = await params;
  const client = await fineract.clients.get(clientId);
  const datatables = await fineract.clients.getDatatables(clientId);

  return (
    <div>
      <ClientHeader client={client} />
      <ClientTabs clientId={clientId} datatables={datatables} />
      <div className="mt-4">{children}</div>
    </div>
  );
}
```

Similar layouts exist for:

- `app/(dashboard)/groups/[groupId]/layout.tsx`
- `app/(dashboard)/centers/[centerId]/layout.tsx`
- `app/(dashboard)/clients/[clientId]/loans-accounts/[loanId]/layout.tsx`
- `app/(dashboard)/clients/[clientId]/savings-accounts/[savingAccountId]/layout.tsx`
- `app/(dashboard)/products/loan-products/[productId]/layout.tsx`
- `app/(dashboard)/organization/offices/[officeId]/layout.tsx`
- `app/(dashboard)/organization/tellers/[id]/layout.tsx`

---

## 5. Loading and Error States

### 5.1 Loading States

Each route segment can define a `loading.tsx` that renders as a Suspense fallback while the page's Server Component awaits data.

```typescript
// app/(dashboard)/clients/loading.tsx
import { Skeleton } from '@/components/ui/skeleton';

export default function ClientsLoading() {
  return (
    <div className="space-y-4">
      <Skeleton className="h-8 w-64" />
      <Skeleton className="h-[400px] w-full" />
    </div>
  );
}
```

Place `loading.tsx` at these key levels:

- `app/(dashboard)/loading.tsx` -- generic dashboard skeleton
- `app/(dashboard)/clients/loading.tsx` -- client list skeleton
- `app/(dashboard)/clients/[clientId]/loading.tsx` -- client detail skeleton
- Each module's list and detail pages

### 5.2 Error States

```typescript
// app/(dashboard)/clients/[clientId]/error.tsx
'use client';

export default function ClientError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="flex flex-col items-center gap-4 p-8">
      <h2 className="text-lg font-semibold">Failed to load client data</h2>
      <p className="text-muted-foreground">{error.message}</p>
      <button onClick={reset} className="btn btn-primary">
        Try again
      </button>
    </div>
  );
}
```

### 5.3 Not Found

```typescript
// app/not-found.tsx
export default function NotFound() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="text-center">
        <h1 className="text-4xl font-bold">404</h1>
        <p className="mt-2">Page not found</p>
      </div>
    </div>
  );
}
```

---

## 6. Parallel Routes

### 6.1 Client Detail Page

For the client detail view, parallel routes allow loading tab content independently.

```
app/(dashboard)/clients/[clientId]/
├── layout.tsx                    # Client header + tab navigation
├── page.tsx                      # redirect to /general
├── @tabs/
│   ├── default.tsx               # Fallback (general tab)
│   ├── general/page.tsx          # Accounts, charges, collateral
│   ├── personal-data/page.tsx
│   ├── address/page.tsx
│   ├── family-members/page.tsx
│   ├── identities/page.tsx
│   ├── documents/page.tsx
│   └── notes/page.tsx
```

**Note:** Parallel routes are optional and add complexity. For the initial migration, a simpler nested-page approach per tab is recommended. Parallel routes can be adopted later for tabs that benefit from independent loading.

### 6.2 Dashboard Page

```
app/(dashboard)/dashboard/
├── layout.tsx
├── page.tsx                      # Main dashboard content
├── @stats/
│   └── page.tsx                  # Stats widgets
└── @recentActivity/
    └── page.tsx                  # Recent transactions feed
```

---

## 7. Navigation

### 7.1 Link Component

```typescript
import Link from 'next/link';

// In a client list component
<Link href={`/clients/${client.id}`}>
  {client.displayName}
</Link>
```

### 7.2 Programmatic Navigation

```typescript
'use client';
import { useRouter } from 'next/navigation';

function CreateClientForm() {
  const router = useRouter();

  async function onSubmit(data: FormData) {
    const result = await createClient(data);
    router.push(`/clients/${result.clientId}`);
  }
}
```

### 7.3 Active Link Styling

```typescript
'use client';
import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { cn } from '@/lib/utils';

function NavLink({ href, children }: { href: string; children: React.ReactNode }) {
  const pathname = usePathname();
  const isActive = pathname === href || pathname.startsWith(href + '/');

  return (
    <Link href={href} className={cn('nav-link', isActive && 'nav-link-active')}>
      {children}
    </Link>
  );
}
```

---

## 8. Breadcrumbs

The Angular app uses `data: { breadcrumb: '...', routeParamBreadcrumb: '...' }` on route definitions. In Next.js, breadcrumbs are generated dynamically from the URL path segments.

```typescript
// components/shell/breadcrumbs.tsx
'use client';
import { usePathname } from 'next/navigation';
import Link from 'next/link';

const LABEL_MAP: Record<string, string> = {
  clients: 'Clients',
  groups: 'Groups',
  centers: 'Centers',
  accounting: 'Accounting',
  products: 'Products',
  organization: 'Organization',
  system: 'System',
  create: 'Create',
  edit: 'Edit',
  general: 'General',
  notes: 'Notes',
  // ... more mappings
};

export function Breadcrumbs() {
  const pathname = usePathname();
  const segments = pathname.split('/').filter(Boolean);

  const crumbs = segments.map((segment, index) => {
    const href = '/' + segments.slice(0, index + 1).join('/');
    const label = LABEL_MAP[segment] || decodeURIComponent(segment);
    const isLast = index === segments.length - 1;

    return { href, label, isLast };
  });

  return (
    <nav aria-label="Breadcrumb" className="mb-4">
      <ol className="flex items-center gap-2 text-sm">
        <li><Link href="/home">Home</Link></li>
        {crumbs.map((crumb) => (
          <li key={crumb.href} className="flex items-center gap-2">
            <span>/</span>
            {crumb.isLast ? (
              <span className="text-muted-foreground">{crumb.label}</span>
            ) : (
              <Link href={crumb.href}>{crumb.label}</Link>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}
```

For dynamic segments (entity IDs), the breadcrumb component can fetch entity names using React Query or accept them via a context provider populated by the entity layout.

---

## 9. Route Estimation Summary

| Module                             | Angular Routes (approx) | Next.js page.tsx Files (approx) |
| ---------------------------------- | ----------------------- | ------------------------------- |
| Home / Dashboard                   | 3                       | 3                               |
| Auth (login, callback)             | 2                       | 2                               |
| Clients                            | 30                      | 26                              |
| Groups                             | 14                      | 13                              |
| Centers                            | 10                      | 9                               |
| Loans                              | 35                      | 33                              |
| Savings                            | 15                      | 14                              |
| Fixed Deposits                     | 12                      | 11                              |
| Recurring Deposits                 | 12                      | 11                              |
| Shares                             | 12                      | 11                              |
| Account Transfers                  | 5                       | 5                               |
| Accounting                         | 28                      | 27                              |
| Products                           | 60                      | 58                              |
| Organization                       | 55                      | 52                              |
| System                             | 50                      | 48                              |
| Other (search, users, tasks, etc.) | 15                      | 14                              |
| **Total**                          | **~358**                | **~337**                        |

The reduction comes from Next.js eliminating the need for separate redirect routes and collapsing some view/tab patterns into single page files.
