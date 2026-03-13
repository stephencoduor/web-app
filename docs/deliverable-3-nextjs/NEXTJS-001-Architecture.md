# NEXTJS-001: Architecture Overview

> Migration guide for the Mifos X Web App from Angular 20 to Next.js 15+ with App Router.

---

## Table of Contents

1. [Why Next.js](#1-why-nextjs)
2. [App Router vs Pages Router](#2-app-router-vs-pages-router)
3. [Tech Stack Decision Matrix](#3-tech-stack-decision-matrix)
4. [Project Structure](#4-project-structure)
5. [Rendering Strategy](#5-rendering-strategy)
6. [Performance Goals](#6-performance-goals)
7. [Monorepo vs Single Package](#7-monorepo-vs-single-package)
8. [package.json](#8-packagejson)

---

## 1. Why Next.js

The current Angular 20 application is a client-side single-page app (SPA). Every route transition, every data fetch, and every render happens entirely in the browser. This creates several pain points that Next.js addresses directly:

### Server-Side Rendering (SSR) for Initial Load Performance

The current Angular app ships a ~2 MB JavaScript bundle before the user sees any content. With Next.js Server Components, list pages (clients, loans, savings) can render on the server and stream HTML to the browser immediately. The user sees a populated table within the first paint instead of a blank screen with a spinner.

### File-System Routing Simplicity

The current app has a complex `app-routing.module.ts` with 60+ lazy-loaded feature modules, each with their own routing module. Next.js file-system routing eliminates all routing boilerplate. A file at `app/clients/[id]/page.tsx` automatically creates a route for `/clients/:id` with no configuration needed.

### React Ecosystem

React has a significantly larger ecosystem of libraries, components, and developer tooling. Hiring React developers is easier than hiring Angular developers. The React community provides battle-tested solutions for virtually every use case the M-SACCO application needs.

### Server Components for Data Fetching

The current architecture requires every API call to originate from the browser, exposing the Fineract API URL and auth tokens to the client. With React Server Components (RSC), data fetching can happen on the server, keeping API credentials and backend URLs out of the browser entirely for read-only pages.

### Incremental Static Regeneration

Reference data (offices, staff, products, code values) rarely changes. Next.js can statically generate these pages at build time and revalidate them on a timer, eliminating redundant API calls entirely.

---

## 2. App Router vs Pages Router

**Decision: App Router (introduced in Next.js 13.4, stable in 15).**

| Feature              | Pages Router                            | App Router                          |
| -------------------- | --------------------------------------- | ----------------------------------- |
| Server Components    | No                                      | Yes (default)                       |
| Layouts              | Per-page `getLayout` hack               | Native nested layouts               |
| Streaming / Suspense | Limited                                 | Full support                        |
| Parallel Routes      | No                                      | Yes (`@modal`, `@sidebar`)          |
| Intercepting Routes  | No                                      | Yes (modal overlays)                |
| Loading UI           | Manual                                  | `loading.tsx` convention            |
| Error Boundaries     | Manual                                  | `error.tsx` convention              |
| Metadata API         | `Head` component                        | Built-in `generateMetadata`         |
| Data Fetching        | `getServerSideProps` / `getStaticProps` | `async` Server Components + `fetch` |

### Why App Router Wins for M-SACCO

1. **Layouts** -- The current Angular app has a shell layout (sidebar + toolbar + content area) implemented in `ShellComponent`. In App Router, this becomes `app/layout.tsx` and is automatically shared across all child routes without re-rendering on navigation.

2. **Server Components** -- List pages for clients, loans, savings, groups, and centers are read-heavy. Server Components fetch data on the server and send rendered HTML, reducing client-side JavaScript by 30-50% for these pages.

3. **Streaming** -- The Fineract API can be slow for large datasets. Streaming lets us show the page shell immediately and progressively render data tables as the API responds.

4. **Parallel Routes** -- The current app opens dialogs (approve loan, assign staff, create charge) as modal overlays. Parallel routes with `@modal` slots model this pattern natively, keeping the underlying page mounted.

5. **Intercepting Routes** -- Clicking a client row can open a quick-view modal (intercepted route) while a direct URL to the same client opens the full detail page. This matches the current UX pattern.

---

## 3. Tech Stack Decision Matrix

| Current (Angular)                 | Replacement (Next.js)                               | Rationale                                                                                                                                                                                                                                                  |
| --------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Angular Material** `^20.2.14`   | **shadcn/ui + Radix UI**                            | Headless primitives with full control over markup and styling. Tailwind-native, no CSS-in-JS runtime. Copy-paste model means components live in the project and can be customized freely. No version-lock to a UI framework.                               |
| **RxJS** `^7.8.2`                 | **TanStack Query v5**                               | Simpler mental model for server state management. No manual `subscribe()`/`unsubscribe()` lifecycle. Built-in caching, deduplication, background refetching, optimistic updates, and devtools.                                                             |
| **Angular Reactive Forms**        | **React Hook Form + Zod**                           | Smaller bundle (~8 KB vs ~45 KB for Angular Forms). Schema-first validation with Zod provides type-safe runtime validation with TypeScript inference. Uncontrolled inputs by default for better performance on large forms (loan creation has 30+ fields). |
| **ngx-translate** `^16.0.4`       | **next-intl**                                       | SSR-compatible (server components can render translations). Type-safe message keys. ICU message format support. Built-in number/date formatting per locale.                                                                                                |
| **angular-oauth2-oidc** `^20.0.0` | **NextAuth.js v5**                                  | Built specifically for Next.js. Supports Credentials provider (for Fineract Basic Auth) and generic OIDC provider (for Zitadel). JWT and database session strategies. Middleware-based route protection.                                                   |
| **Moment.js** `^2.29.4`           | **date-fns v3**                                     | Tree-shakeable (import only the functions used). ~7 KB for common operations vs ~72 KB for Moment. Immutable by default. Native `Date` objects, no wrapper. The current app uses Moment in `Dates` utility and custom date adapters.                       |
| **Angular CDK DragDrop**          | **@dnd-kit/core + @dnd-kit/sortable**               | React-native drag-and-drop with accessibility built in (keyboard, screen reader). Used for reordering survey questions, report columns, and workflow steps.                                                                                                |
| **Chart.js** `^4.5.1`             | **Chart.js + react-chartjs-2**                      | Same library, React wrapper. Minimal migration effort.                                                                                                                                                                                                     |
| **@swimlane/ngx-graph** `^11.0.0` | **reactflow**                                       | React-native graph/flow visualization. More actively maintained, better documentation.                                                                                                                                                                     |
| **ExcelJS** `^4.4.0`              | **ExcelJS** (keep)                                  | Framework-agnostic. No change needed.                                                                                                                                                                                                                      |
| **jsPDF + jspdf-autotable**       | **jsPDF + jspdf-autotable** (keep)                  | Framework-agnostic. No change needed.                                                                                                                                                                                                                      |
| **FontAwesome** `^6.7.2`          | **lucide-react** (primary) + FontAwesome (fallback) | lucide-react is the default for shadcn/ui. Tree-shakeable SVG icons. Map existing FontAwesome icons to lucide equivalents where possible, keep FontAwesome for any icons without a match.                                                                  |
| **TinyMCE** `^8.2.2`              | **@tinymce/tinymce-react**                          | Official React wrapper. Same TinyMCE engine.                                                                                                                                                                                                               |
| **lightgallery** `^2.9.0`         | **lightgallery** (keep) + `lightgallery/react`      | Framework-agnostic core with React plugin.                                                                                                                                                                                                                 |
| **signature_pad** `^5.1.3`        | **signature_pad** (keep)                            | Framework-agnostic canvas library. Wrap in a React component.                                                                                                                                                                                              |

---

## 4. Project Structure

```
msacco-nextjs/
├── app/                              # Next.js App Router
│   ├── (auth)/                       # Route group for unauthenticated pages
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── callback/
│   │   │   └── page.tsx
│   │   └── layout.tsx                # Minimal layout (no sidebar)
│   ├── (dashboard)/                  # Route group for authenticated pages
│   │   ├── layout.tsx                # Shell: sidebar + toolbar + content
│   │   ├── loading.tsx               # Global loading skeleton
│   │   ├── error.tsx                 # Global error boundary
│   │   ├── page.tsx                  # Home / Dashboard
│   │   ├── @modal/                   # Parallel route for modal dialogs
│   │   │   └── default.tsx
│   │   ├── clients/
│   │   │   ├── page.tsx              # Client list (Server Component)
│   │   │   ├── loading.tsx           # List skeleton
│   │   │   ├── create/
│   │   │   │   └── page.tsx          # Create client wizard (Client Component)
│   │   │   └── [id]/
│   │   │       ├── page.tsx          # Client detail (Server Component shell)
│   │   │       ├── layout.tsx        # Client detail tabs layout
│   │   │       ├── general/
│   │   │       │   └── page.tsx
│   │   │       ├── accounts/
│   │   │       │   └── page.tsx
│   │   │       ├── loans/
│   │   │       │   └── page.tsx
│   │   │       ├── charges/
│   │   │       │   └── page.tsx
│   │   │       ├── documents/
│   │   │       │   └── page.tsx
│   │   │       └── notes/
│   │   │           └── page.tsx
│   │   ├── groups/
│   │   │   ├── page.tsx
│   │   │   └── [id]/
│   │   │       └── ...
│   │   ├── centers/
│   │   │   ├── page.tsx
│   │   │   └── [id]/
│   │   │       └── ...
│   │   ├── loans/
│   │   │   ├── page.tsx
│   │   │   └── [id]/
│   │   │       └── ...
│   │   ├── savings/
│   │   │   ├── page.tsx
│   │   │   └── [id]/
│   │   │       └── ...
│   │   ├── deposits/
│   │   │   ├── fixed-deposits/
│   │   │   └── recurring-deposits/
│   │   ├── shares/
│   │   │   └── ...
│   │   ├── accounting/
│   │   │   ├── journal-entries/
│   │   │   ├── chart-of-accounts/
│   │   │   ├── closing-entries/
│   │   │   └── ...
│   │   ├── reports/
│   │   │   └── ...
│   │   ├── organization/
│   │   │   ├── offices/
│   │   │   ├── employees/
│   │   │   ├── manage-funds/
│   │   │   ├── bulk-loan-reassignment/
│   │   │   └── ...
│   │   ├── system/
│   │   │   ├── manage-codes/
│   │   │   ├── manage-roles/
│   │   │   ├── manage-hooks/
│   │   │   ├── configurations/
│   │   │   └── ...
│   │   ├── products/
│   │   │   ├── loan-products/
│   │   │   ├── savings-products/
│   │   │   ├── share-products/
│   │   │   ├── charges/
│   │   │   └── ...
│   │   ├── users/
│   │   │   └── ...
│   │   ├── tasks/
│   │   │   └── page.tsx              # Checker inbox / pending approvals
│   │   ├── notifications/
│   │   │   └── page.tsx
│   │   ├── collections/
│   │   │   └── page.tsx
│   │   ├── account-transfers/
│   │   │   └── ...
│   │   ├── templates/
│   │   │   └── ...
│   │   ├── search/
│   │   │   └── page.tsx
│   │   ├── profile/
│   │   │   └── page.tsx
│   │   └── settings/
│   │       └── page.tsx
│   ├── api/                          # API routes (Next.js Route Handlers)
│   │   └── auth/
│   │       └── [...nextauth]/
│   │           └── route.ts          # NextAuth.js handler
│   ├── layout.tsx                    # Root layout (html, body, providers)
│   ├── not-found.tsx                 # 404 page
│   └── globals.css                   # Global styles + Tailwind directives
│
├── components/                       # Shared React components
│   ├── ui/                           # shadcn/ui components (auto-generated)
│   │   ├── button.tsx
│   │   ├── input.tsx
│   │   ├── select.tsx
│   │   ├── dialog.tsx
│   │   ├── table.tsx
│   │   ├── tabs.tsx
│   │   ├── sheet.tsx
│   │   ├── toast.tsx
│   │   ├── badge.tsx
│   │   ├── command.tsx
│   │   ├── popover.tsx
│   │   ├── calendar.tsx
│   │   ├── checkbox.tsx
│   │   ├── radio-group.tsx
│   │   ├── switch.tsx
│   │   ├── textarea.tsx
│   │   ├── tooltip.tsx
│   │   ├── skeleton.tsx
│   │   ├── separator.tsx
│   │   ├── scroll-area.tsx
│   │   ├── dropdown-menu.tsx
│   │   ├── accordion.tsx
│   │   └── form.tsx                  # React Hook Form + shadcn integration
│   ├── layout/                       # Layout-level components
│   │   ├── sidebar.tsx               # Main sidebar navigation
│   │   ├── toolbar.tsx               # Top toolbar
│   │   ├── breadcrumb.tsx            # Breadcrumb navigation
│   │   ├── content-wrapper.tsx       # Content area with padding
│   │   └── theme-toggle.tsx          # Dark mode toggle
│   ├── shared/                       # Domain-agnostic reusable components
│   │   ├── data-table/
│   │   │   ├── data-table.tsx        # TanStack Table wrapper
│   │   │   ├── data-table-toolbar.tsx
│   │   │   ├── data-table-pagination.tsx
│   │   │   ├── data-table-column-header.tsx
│   │   │   ├── data-table-faceted-filter.tsx
│   │   │   └── data-table-export.tsx
│   │   ├── wizard/
│   │   │   ├── wizard.tsx            # Multi-step form container
│   │   │   ├── wizard-step.tsx
│   │   │   └── wizard-nav.tsx
│   │   ├── approval-action-bar.tsx   # Approve / Reject / Undo action bar
│   │   ├── status-badge.tsx          # Colored status badges
│   │   ├── file-upload.tsx           # Drag-and-drop file upload
│   │   ├── form-dialog.tsx           # Dynamic form in dialog
│   │   ├── confirm-dialog.tsx        # Confirmation dialog
│   │   ├── search-select.tsx         # Searchable select (combobox)
│   │   ├── date-picker.tsx           # Date picker with business date support
│   │   ├── currency-input.tsx        # Formatted currency input
│   │   └── empty-state.tsx           # Empty state placeholder
│   └── domain/                       # Domain-specific reusable components
│       ├── client-select.tsx
│       ├── office-select.tsx
│       ├── staff-select.tsx
│       ├── product-select.tsx
│       └── account-select.tsx
│
├── lib/                              # Shared utilities and services
│   ├── api/
│   │   ├── client.ts                 # Axios instance configuration
│   │   ├── server-client.ts          # Server-side fetch wrapper
│   │   ├── endpoints.ts              # All Fineract API endpoint constants
│   │   ├── types/                    # API response/request types
│   │   │   ├── client.ts
│   │   │   ├── loan.ts
│   │   │   ├── savings.ts
│   │   │   ├── group.ts
│   │   │   ├── center.ts
│   │   │   ├── accounting.ts
│   │   │   ├── organization.ts
│   │   │   ├── products.ts
│   │   │   ├── system.ts
│   │   │   ├── user.ts
│   │   │   └── common.ts             # Shared types (pagination, error, etc.)
│   │   └── errors.ts                 # Typed error handling
│   ├── auth/
│   │   ├── auth-options.ts           # NextAuth.js configuration
│   │   ├── auth-context.tsx          # React context for auth/permissions
│   │   └── middleware.ts             # Auth middleware logic
│   ├── services/                     # TanStack Query hooks per domain
│   │   ├── clients.ts
│   │   ├── loans.ts
│   │   ├── savings.ts
│   │   ├── groups.ts
│   │   ├── centers.ts
│   │   ├── accounting.ts
│   │   ├── organization.ts
│   │   ├── products.ts
│   │   ├── system.ts
│   │   ├── users.ts
│   │   ├── reports.ts
│   │   ├── search.ts
│   │   └── notifications.ts
│   ├── stores/                       # Zustand stores for UI state
│   │   ├── theme-store.ts
│   │   ├── settings-store.ts
│   │   ├── sidebar-store.ts
│   │   └── alert-store.ts
│   ├── hooks/                        # Custom React hooks
│   │   ├── use-permissions.ts
│   │   ├── use-date-format.ts
│   │   ├── use-debounce.ts
│   │   ├── use-business-date.ts
│   │   └── use-keyboard-shortcuts.ts
│   ├── i18n/
│   │   ├── config.ts                 # next-intl configuration
│   │   ├── request.ts                # Server-side locale detection
│   │   └── navigation.ts             # Localized navigation helpers
│   ├── utils/
│   │   ├── dates.ts                  # Date formatting (migrated from core/utils/dates.ts)
│   │   ├── accounting.ts             # Accounting helpers (migrated from core/utils/accounting.ts)
│   │   ├── charges.ts                # Charge calculation helpers
│   │   ├── datatables.ts             # Datatable field utilities
│   │   ├── password.ts               # Password validation
│   │   ├── cn.ts                     # Tailwind className merge utility
│   │   └── format.ts                 # Number/currency formatting
│   └── validators/                   # Zod schemas for form validation
│       ├── client.ts
│       ├── loan.ts
│       ├── savings.ts
│       ├── group.ts
│       ├── center.ts
│       └── user.ts
│
├── styles/
│   └── globals.css                   # Tailwind directives + CSS variables
│
├── types/
│   ├── next-auth.d.ts                # NextAuth type augmentation
│   └── global.d.ts                   # Global type declarations
│
├── public/
│   ├── images/                       # Static images (logos, icons)
│   └── locales/                      # Translation JSON files
│       ├── cs-CS.json
│       ├── de-DE.json
│       ├── en-US.json
│       ├── es-MX.json
│       ├── fr-FR.json
│       ├── it-IT.json
│       ├── ko-KO.json
│       ├── lt-LT.json
│       ├── lv-LV.json
│       ├── ne-NE.json
│       ├── pt-PT.json
│       └── sw-SW.json
│
├── middleware.ts                      # Next.js middleware (auth + i18n)
├── next.config.ts                    # Next.js configuration
├── tailwind.config.ts                # Tailwind CSS configuration
├── tsconfig.json                     # TypeScript configuration
├── .env.local                        # Local environment variables
├── .env.example                      # Environment variable template
└── package.json
```

### Key Structural Decisions

- **Route Groups** `(auth)` and `(dashboard)` -- Separate layouts for authenticated and unauthenticated pages without affecting the URL structure.
- **Parallel Route** `@modal` -- Modal dialogs (approve, reject, assign, unassign) render as intercepted routes in a parallel slot, keeping the background page mounted.
- **`lib/services/`** -- One file per Fineract domain, containing all TanStack Query hooks. This directly maps to the current Angular service files (e.g., `clients.service.ts` becomes `lib/services/clients.ts`).
- **`lib/api/types/`** -- TypeScript interfaces extracted from Fineract API responses. The current Angular codebase uses `any` extensively; the migration is an opportunity to add proper typing.
- **`components/ui/`** -- shadcn/ui components are copied into the project (not installed as a dependency). This gives full control over the source code.

---

## 5. Rendering Strategy

### Server Components (Default)

Use for pages that primarily display data fetched from the Fineract API. These components run only on the server, send zero JavaScript to the client, and can directly call the API without exposing credentials.

| Page Type            | Example Routes                                          | Why Server Component                                                                  |
| -------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| List pages           | `/clients`, `/loans`, `/savings`, `/groups`, `/centers` | Data fetching + table rendering. No client interactivity until the user clicks a row. |
| Detail page shells   | `/clients/[id]`, `/loans/[id]`                          | Header info, status badge, tab layout. Static once loaded.                            |
| Reference data pages | `/organization/offices`, `/system/manage-codes`         | Rarely changed data. Can use `revalidate` for ISR.                                    |
| Report pages         | `/reports/[id]`                                         | Read-only report display.                                                             |
| Static content       | `/not-found`, Settings display                          | No dynamic data or interactivity.                                                     |

```tsx
// app/(dashboard)/clients/page.tsx -- Server Component (default)
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth/auth-options';
import { fetchClients } from '@/lib/api/server-client';
import { ClientDataTable } from './client-data-table'; // Client Component

export default async function ClientsPage({ searchParams }: { searchParams: Promise<{ page?: string; limit?: string; search?: string }> }) {
  const params = await searchParams;
  const session = await getServerSession(authOptions);
  const clients = await fetchClients(session!, {
    offset: Number(params.page ?? 0) * Number(params.limit ?? 50),
    limit: Number(params.limit ?? 50),
    sqlSearch: params.search
  });

  return (
    <div>
      <h1>Clients</h1>
      {/* ClientDataTable is a Client Component for sorting, filtering, row clicks */}
      <ClientDataTable initialData={clients} />
    </div>
  );
}
```

### Client Components (`"use client"`)

Use for pages or components that require browser APIs, event handlers, state management, or third-party libraries that use React hooks.

| Page Type          | Example Routes/Components                                        | Why Client Component                                       |
| ------------------ | ---------------------------------------------------------------- | ---------------------------------------------------------- |
| Forms              | Create Client wizard, Loan application, Savings account creation | React Hook Form, controlled inputs, multi-step navigation  |
| Interactive tables | Sortable/filterable data tables with row selection               | TanStack Table with client-side sorting, column visibility |
| Dialogs            | Approve Loan, Assign Staff, Delete Charge                        | Dialog open/close state, form submission                   |
| Drag-and-drop      | Survey question ordering, report column arrangement              | @dnd-kit requires DOM access                               |
| Charts             | Dashboard charts, portfolio analytics                            | Chart.js renders to canvas                                 |
| Rich text editors  | Note creation with TinyMCE                                       | TinyMCE requires DOM                                       |
| Signature capture  | Client signature pad                                             | Canvas API                                                 |

```tsx
// app/(dashboard)/clients/create/page.tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { clientSchema } from '@/lib/validators/client';
import { useCreateClient } from '@/lib/services/clients';
import { Wizard, WizardStep } from '@/components/shared/wizard/wizard';

export default function CreateClientPage() {
  const form = useForm({ resolver: zodResolver(clientSchema) });
  const createClient = useCreateClient();

  return (
    <Wizard onComplete={(data) => createClient.mutate(data)}>
      <WizardStep title="Personal Information">{/* form fields */}</WizardStep>
      <WizardStep title="Address Details">{/* form fields */}</WizardStep>
      <WizardStep title="Family Members">{/* form fields */}</WizardStep>
    </Wizard>
  );
}
```

### Hybrid Pattern

Most detail pages use a hybrid approach: the page shell (header, tabs, metadata) is a Server Component, while interactive sections (action buttons, editable fields, dialogs) are Client Components embedded within.

```tsx
// app/(dashboard)/clients/[id]/page.tsx -- Server Component
import { ClientActions } from './client-actions'; // Client Component
import { StatusBadge } from '@/components/shared/status-badge';

export default async function ClientDetailPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const client = await fetchClient(session!, Number(id));

  return (
    <div>
      <div className="flex items-center justify-between">
        <h1>{client.displayName}</h1>
        <StatusBadge status={client.status} />
      </div>
      {/* Client Component for approve/reject/close/transfer actions */}
      <ClientActions clientId={client.id} status={client.status} />
    </div>
  );
}
```

---

## 6. Performance Goals

Target Core Web Vitals scores for the migrated application:

| Metric                             | Target   | Current Angular App (estimated) | Strategy                                                                                                         |
| ---------------------------------- | -------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **LCP** (Largest Contentful Paint) | < 2.0s   | ~4.5s                           | Server-side rendering of list pages; streaming for slow API calls; `<Image>` optimization for logos              |
| **FID** (First Input Delay)        | < 100ms  | ~250ms                          | Server Components reduce client JS by ~40%; code-split Client Components per route                               |
| **CLS** (Cumulative Layout Shift)  | < 0.1    | ~0.15                           | Skeleton loading states with fixed dimensions; `next/font` for font loading; explicit `width`/`height` on images |
| **TTFB** (Time to First Byte)      | < 500ms  | ~200ms (SPA, just HTML shell)   | Server-side data fetching adds time, mitigated by edge caching and streaming                                     |
| **Bundle Size** (initial JS)       | < 150 KB | ~580 KB (gzipped)               | Server Components send zero JS; route-level code splitting; tree-shakeable libraries                             |

### Measurement Plan

1. **Lighthouse CI** -- Run on every PR against key pages (home, client list, client detail, loan detail)
2. **Web Vitals Reporting** -- Integrate `next/web-vitals` to report real user metrics to analytics
3. **Bundle Analyzer** -- Run `@next/bundle-analyzer` monthly to identify bloat

---

## 7. Monorepo vs Single Package

**Decision: Single package (no monorepo) for the initial migration.**

### Reasoning

| Factor          | Monorepo                                             | Single Package                   |
| --------------- | ---------------------------------------------------- | -------------------------------- |
| Complexity      | Higher (Turborepo/Nx config, workspace dependencies) | Lower (standard Next.js project) |
| Build speed     | Cached parallel builds                               | Single build, simpler CI         |
| Team size       | Beneficial for 10+ developers                        | Current team is < 10             |
| Shared packages | Needed if multiple apps share code                   | Only one web application         |
| Migration speed | Slower setup                                         | Faster to start migrating        |

### When to Reconsider

- If a mobile app (React Native) is added that shares API client code and validators
- If a separate admin panel is built alongside the main app
- If the team grows beyond 10 developers working on the same codebase

At that point, extract `lib/api/`, `lib/validators/`, and `components/ui/` into shared packages within a Turborepo workspace.

---

## 8. package.json

```jsonc
{
  "name": "msacco-web-app",
  "version": "1.0.0",
  "private": true,
  "description": "M-SACCO Web App - Next.js migration of the Mifos X Web Application",
  "license": "MPL-2.0",
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "type-check": "tsc --noEmit",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "db:generate": "prisma generate",
    "analyze": "ANALYZE=true next build"
  },
  "dependencies": {
    // --- Framework ---
    "next": "^15.1.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",

    // --- Authentication ---
    "next-auth": "^5.0.0-beta.25",

    // --- Data Fetching & State ---
    "@tanstack/react-query": "^5.62.0",
    "@tanstack/react-query-devtools": "^5.62.0",
    "axios": "^1.7.9",
    "zustand": "^5.0.2",
    "nuqs": "^2.2.3",

    // --- UI Components ---
    "@radix-ui/react-accordion": "^1.2.2",
    "@radix-ui/react-alert-dialog": "^1.1.4",
    "@radix-ui/react-checkbox": "^1.1.3",
    "@radix-ui/react-dialog": "^1.1.4",
    "@radix-ui/react-dropdown-menu": "^2.1.4",
    "@radix-ui/react-label": "^2.1.1",
    "@radix-ui/react-popover": "^1.1.4",
    "@radix-ui/react-radio-group": "^1.2.2",
    "@radix-ui/react-scroll-area": "^1.2.2",
    "@radix-ui/react-select": "^2.1.4",
    "@radix-ui/react-separator": "^1.1.1",
    "@radix-ui/react-sheet": "^1.0.0",
    "@radix-ui/react-slot": "^1.1.1",
    "@radix-ui/react-switch": "^1.1.2",
    "@radix-ui/react-tabs": "^1.1.2",
    "@radix-ui/react-toast": "^1.2.4",
    "@radix-ui/react-tooltip": "^1.1.6",
    "class-variance-authority": "^0.7.1",
    "clsx": "^2.1.1",
    "tailwind-merge": "^2.6.0",
    "cmdk": "^1.0.4",
    "sonner": "^1.7.1",

    // --- Tables ---
    "@tanstack/react-table": "^8.20.6",

    // --- Forms ---
    "react-hook-form": "^7.54.1",
    "@hookform/resolvers": "^3.9.1",
    "zod": "^3.24.1",

    // --- Internationalization ---
    "next-intl": "^3.25.3",

    // --- Dates ---
    "date-fns": "^4.1.0",
    "react-day-picker": "^9.4.4",

    // --- Drag & Drop ---
    "@dnd-kit/core": "^6.3.1",
    "@dnd-kit/sortable": "^10.0.0",
    "@dnd-kit/utilities": "^3.2.2",

    // --- Charts ---
    "chart.js": "^4.5.1",
    "react-chartjs-2": "^5.2.0",

    // --- Icons ---
    "lucide-react": "^0.468.0",

    // --- Theme ---
    "next-themes": "^0.4.4",

    // --- Rich Text ---
    "@tinymce/tinymce-react": "^5.1.1",

    // --- Export ---
    "exceljs": "^4.4.0",
    "jspdf": "^4.2.0",
    "jspdf-autotable": "^5.0.2",

    // --- Signature ---
    "signature_pad": "^5.1.3",

    // --- Utilities ---
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    // --- TypeScript ---
    "typescript": "^5.7.2",
    "@types/node": "^22.10.2",
    "@types/react": "^19.0.2",
    "@types/react-dom": "^19.0.2",
    "@types/lodash": "^4.17.13",

    // --- Styling ---
    "tailwindcss": "^3.4.17",
    "postcss": "^8.4.49",
    "autoprefixer": "^10.4.20",
    "@tailwindcss/forms": "^0.5.9",
    "@tailwindcss/typography": "^0.5.15",

    // --- Linting ---
    "eslint": "^9.16.0",
    "eslint-config-next": "^15.1.0",
    "prettier": "^3.4.2",
    "prettier-plugin-tailwindcss": "^0.6.9",

    // --- Testing ---
    "vitest": "^2.1.8",
    "@testing-library/react": "^16.1.0",
    "@testing-library/jest-dom": "^6.6.3",
    "@testing-library/user-event": "^14.5.2",
    "@playwright/test": "^1.49.1",

    // --- Build Analysis ---
    "@next/bundle-analyzer": "^15.1.0"
  },
  "engines": {
    "node": ">=20.11.0",
    "npm": ">=10.0.0"
  }
}
```

### Version Pinning Notes

- `next-auth` uses a v5 beta because v5 is the only version that supports App Router natively. It has been stable since beta.20 and is widely deployed in production.
- `@tanstack/react-query` v5 is chosen over v4 for improved TypeScript support and the new `useSuspenseQuery` hook that integrates with React Suspense boundaries.
- `react` and `react-dom` v19 are required by Next.js 15 for Server Components and the new `use()` hook.
- `nuqs` is preferred over `next-usequerystate` (they merged) for type-safe URL search parameter state management.
