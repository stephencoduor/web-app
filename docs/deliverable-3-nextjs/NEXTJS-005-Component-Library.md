# NEXTJS-005: Component Library Design

> Migration guide for the UI component layer from Angular Material to shadcn/ui + Radix UI + Tailwind CSS.

---

## Table of Contents

1. [shadcn/ui Setup](#1-shadcnui-setup)
2. [Tailwind Theme Tokens](#2-tailwind-theme-tokens)
3. [Component Mapping](#3-component-mapping)
4. [Custom Components](#4-custom-components)
5. [Icon Strategy](#5-icon-strategy)
6. [Dark Mode](#6-dark-mode)

---

## 1. shadcn/ui Setup

shadcn/ui is not a component library in the traditional sense. It is a collection of reusable components that are copied into the project source code. This gives full ownership of the component code -- no version upgrades break the UI, and every component can be customized freely.

### Installation

```bash
# Initialize shadcn/ui in the project
npx shadcn@latest init

# Configuration choices:
# Style: Default
# Base color: Slate
# CSS variables: Yes
# Tailwind CSS config: tailwind.config.ts
# Components alias: @/components
# Utils alias: @/lib/utils
# React Server Components: Yes
```

### Installing Individual Components

```bash
# Install the components needed for the M-SACCO app
npx shadcn@latest add button input label select textarea checkbox radio-group switch \
  dialog sheet tabs table badge tooltip popover dropdown-menu command scroll-area \
  separator accordion skeleton form calendar toast sonner card alert-dialog \
  avatar breadcrumb collapsible navigation-menu progress sidebar
```

Each command copies the component source into `components/ui/`. The files are now owned by the project and can be modified directly.

### Tailwind Configuration

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss';
import { fontFamily } from 'tailwindcss/defaultTheme';

const config: Config = {
  darkMode: ['class'], // next-themes uses class-based dark mode
  content: [
    './app/**/*.{ts,tsx}',
    './components/**/*.{ts,tsx}',
    './lib/**/*.{ts,tsx}'
  ],
  theme: {
    container: {
      center: true,
      padding: '2rem',
      screens: {
        '2xl': '1400px'
      }
    },
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))'
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))'
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))'
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))'
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))'
        },
        popover: {
          DEFAULT: 'hsl(var(--popover))',
          foreground: 'hsl(var(--popover-foreground))'
        },
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))'
        },
        // M-SACCO custom semantic colors
        success: {
          DEFAULT: 'hsl(var(--success))',
          foreground: 'hsl(var(--success-foreground))'
        },
        warning: {
          DEFAULT: 'hsl(var(--warning))',
          foreground: 'hsl(var(--warning-foreground))'
        },
        info: {
          DEFAULT: 'hsl(var(--info))',
          foreground: 'hsl(var(--info-foreground))'
        },
        sidebar: {
          DEFAULT: 'hsl(var(--sidebar-background))',
          foreground: 'hsl(var(--sidebar-foreground))',
          primary: 'hsl(var(--sidebar-primary))',
          'primary-foreground': 'hsl(var(--sidebar-primary-foreground))',
          accent: 'hsl(var(--sidebar-accent))',
          'accent-foreground': 'hsl(var(--sidebar-accent-foreground))',
          border: 'hsl(var(--sidebar-border))',
          ring: 'hsl(var(--sidebar-ring))'
        }
      },
      fontFamily: {
        sans: [
          'Inter',
          ...fontFamily.sans
        ]
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)'
      },
      keyframes: {
        'accordion-down': {
          from: { height: '0' },
          to: { height: 'var(--radix-accordion-content-height)' }
        },
        'accordion-up': {
          from: { height: 'var(--radix-accordion-content-height)' },
          to: { height: '0' }
        }
      },
      animation: {
        'accordion-down': 'accordion-down 0.2s ease-out',
        'accordion-up': 'accordion-up 0.2s ease-out'
      }
    }
  },
  plugins: [
    require('tailwindcss-animate'),
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography')
  ]
};

export default config;
```

---

## 2. Tailwind Theme Tokens

The Angular Material theme uses a custom palette defined in SCSS. The Next.js application maps these to CSS custom properties consumed by Tailwind and shadcn/ui.

### CSS Variables

```css
/* styles/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    /* === M-SACCO Brand Colors (mapped from Angular Material palette) === */

    /* Primary: Blue (#1074b9 from the Angular app toolbar/sidebar) */
    --primary: 205 83% 39%;
    --primary-foreground: 0 0% 100%;

    /* Secondary */
    --secondary: 210 17% 95%;
    --secondary-foreground: 210 11% 15%;

    /* Accent: Green (#b4d575 used for success indicators) */
    --accent: 85 50% 65%;
    --accent-foreground: 210 11% 15%;

    /* === UI Semantic Colors === */

    --background: 0 0% 100%;
    --foreground: 210 11% 15%;

    --card: 0 0% 100%;
    --card-foreground: 210 11% 15%;

    --popover: 0 0% 100%;
    --popover-foreground: 210 11% 15%;

    --muted: 210 17% 95%;
    --muted-foreground: 215 16% 47%;

    --destructive: 0 84% 60%;
    --destructive-foreground: 0 0% 100%;

    --border: 214 32% 91%;
    --input: 214 32% 91%;
    --ring: 205 83% 39%;

    /* === Status Colors (used extensively in M-SACCO for account/loan status) === */
    --success: 142 71% 45%;
    --success-foreground: 0 0% 100%;

    --warning: 38 92% 50%;
    --warning-foreground: 0 0% 100%;

    --info: 205 83% 39%;
    --info-foreground: 0 0% 100%;

    /* === Layout === */
    --radius: 0.5rem;

    /* === Sidebar (matches Angular app sidebar dark blue) === */
    --sidebar-background: 210 29% 24%;
    --sidebar-foreground: 210 17% 95%;
    --sidebar-primary: 205 83% 55%;
    --sidebar-primary-foreground: 0 0% 100%;
    --sidebar-accent: 210 29% 30%;
    --sidebar-accent-foreground: 210 17% 95%;
    --sidebar-border: 210 29% 30%;
    --sidebar-ring: 205 83% 55%;
  }

  .dark {
    /* === Dark Mode (matches Angular app's dark theme) === */

    --primary: 205 83% 55%;
    --primary-foreground: 0 0% 100%;

    --secondary: 215 25% 17%;
    --secondary-foreground: 210 17% 90%;

    --accent: 85 50% 55%;
    --accent-foreground: 0 0% 100%;

    --background: 222 47% 11%;
    --foreground: 210 17% 90%;

    --card: 215 25% 15%;
    --card-foreground: 210 17% 90%;

    --popover: 215 25% 15%;
    --popover-foreground: 210 17% 90%;

    --muted: 215 25% 20%;
    --muted-foreground: 215 16% 60%;

    --destructive: 0 63% 50%;
    --destructive-foreground: 0 0% 100%;

    --border: 215 25% 22%;
    --input: 215 25% 22%;
    --ring: 205 83% 55%;

    --success: 142 71% 45%;
    --success-foreground: 0 0% 100%;

    --warning: 38 92% 50%;
    --warning-foreground: 0 0% 100%;

    --info: 205 83% 55%;
    --info-foreground: 0 0% 100%;

    --sidebar-background: 222 47% 8%;
    --sidebar-foreground: 210 17% 85%;
    --sidebar-primary: 205 83% 55%;
    --sidebar-primary-foreground: 0 0% 100%;
    --sidebar-accent: 222 47% 14%;
    --sidebar-accent-foreground: 210 17% 90%;
    --sidebar-border: 222 47% 14%;
    --sidebar-ring: 205 83% 55%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

### Color Mapping Reference

| Angular Material Token    | CSS Variable           | HSL Value (Light) | Hex Approx |
| ------------------------- | ---------------------- | ----------------- | ---------- |
| Primary (toolbar/buttons) | `--primary`            | `205 83% 39%`     | `#1074b9`  |
| Accent (success actions)  | `--accent`             | `85 50% 65%`      | `#b4d575`  |
| Warn (destructive)        | `--destructive`        | `0 84% 60%`       | `#ef4444`  |
| Background                | `--background`         | `0 0% 100%`       | `#ffffff`  |
| Foreground (text)         | `--foreground`         | `210 11% 15%`     | `#212529`  |
| Card background           | `--card`               | `0 0% 100%`       | `#ffffff`  |
| Sidebar background        | `--sidebar-background` | `210 29% 24%`     | `#2b3e50`  |
| Muted (disabled, borders) | `--muted`              | `210 17% 95%`     | `#f0f2f5`  |

---

## 3. Component Mapping

Detailed mapping from Angular Material components to shadcn/ui equivalents. Each entry shows the Angular component, the shadcn replacement, and migration notes.

### Buttons

| Angular Material              | shadcn/ui                                       | Notes                               |
| ----------------------------- | ----------------------------------------------- | ----------------------------------- |
| `<button mat-button>`         | `<Button variant="ghost">`                      | Text button without background      |
| `<button mat-raised-button>`  | `<Button>`                                      | Default shadcn button (filled)      |
| `<button mat-stroked-button>` | `<Button variant="outline">`                    | Outlined button                     |
| `<button mat-flat-button>`    | `<Button>`                                      | Same as raised in shadcn            |
| `<button mat-icon-button>`    | `<Button variant="ghost" size="icon">`          | Icon-only button                    |
| `<button mat-fab>`            | `<Button size="lg" className="rounded-full">`   | Floating action button              |
| `<button mat-mini-fab>`       | `<Button size="icon" className="rounded-full">` | Small FAB                           |
| `color="primary"`             | Default (no extra class needed)                 | Primary is the default button color |
| `color="accent"`              | `className="bg-accent text-accent-foreground"`  | Or create a custom variant          |
| `color="warn"`                | `variant="destructive"`                         | Red/warning button                  |

### Form Controls

| Angular Material                                        | shadcn/ui                                | Notes                                                       |
| ------------------------------------------------------- | ---------------------------------------- | ----------------------------------------------------------- |
| `<mat-form-field>` + `<mat-label>` + `<input matInput>` | `<FormItem>` + `<FormLabel>` + `<Input>` | shadcn Form components integrate with React Hook Form       |
| `<mat-select>`                                          | `<Select>`                               | Radix UI select with search via `<Command>`                 |
| `<mat-checkbox>`                                        | `<Checkbox>`                             | Radix UI checkbox                                           |
| `<mat-radio-group>`                                     | `<RadioGroup>`                           | Radix UI radio group                                        |
| `<mat-slide-toggle>`                                    | `<Switch>`                               | Radix UI switch                                             |
| `<textarea matInput>`                                   | `<Textarea>`                             | Standard textarea with styling                              |
| `<mat-datepicker>`                                      | `<DatePicker>` (custom)                  | Composed from `<Popover>` + `<Calendar>` (react-day-picker) |
| `<mat-autocomplete>`                                    | `<Command>` (cmdk)                       | Command palette component used as autocomplete              |
| `<mat-chip-list>`                                       | Custom chip component                    | Build from `<Badge>` with close button                      |
| `<mat-error>`                                           | `<FormMessage>`                          | Error message below form field                              |
| `<mat-hint>`                                            | `<FormDescription>`                      | Helper text below form field                                |

### Layout and Navigation

| Angular Material                            | shadcn/ui                                                   | Notes                                                         |
| ------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------- |
| `<mat-sidenav-container>` + `<mat-sidenav>` | `<Sidebar>` (shadcn sidebar)                                | shadcn has a full sidebar component with collapsible sections |
| `<mat-toolbar>`                             | Custom `<Toolbar>`                                          | Build from `<div>` with flex layout, no shadcn equivalent     |
| `<mat-tab-group>` + `<mat-tab>`             | `<Tabs>` + `<TabsList>` + `<TabsTrigger>` + `<TabsContent>` | Radix UI tabs                                                 |
| `<mat-stepper>` + `<mat-step>`              | Custom `<Wizard>`                                           | See Section 4 for full implementation                         |
| `<mat-expansion-panel>`                     | `<Accordion>` + `<AccordionItem>`                           | Radix UI accordion                                            |
| `<mat-menu>`                                | `<DropdownMenu>`                                            | Radix UI dropdown menu                                        |
| `<mat-card>`                                | `<Card>` + `<CardHeader>` + `<CardContent>`                 | shadcn card components                                        |
| `<mat-divider>`                             | `<Separator>`                                               | Radix UI separator                                            |

### Data Display

| Angular Material         | shadcn/ui                     | Notes                                         |
| ------------------------ | ----------------------------- | --------------------------------------------- |
| `<mat-table>`            | `<Table>` + TanStack Table    | See DataTable custom component in Section 4   |
| `<mat-paginator>`        | Custom pagination             | Part of the DataTable component               |
| `<mat-sort-header>`      | TanStack Table column sorting | Built into DataTable                          |
| `<mat-list>`             | `<div>` with Tailwind         | Simple list with `divide-y` class             |
| `<mat-badge>`            | `<Badge>`                     | shadcn badge                                  |
| `<mat-progress-bar>`     | `<Progress>`                  | shadcn progress bar                           |
| `<mat-progress-spinner>` | Spinner icon or `<Skeleton>`  | Use lucide `Loader2` icon with `animate-spin` |

### Feedback

| Angular Material     | shadcn/ui          | Notes                                         |
| -------------------- | ------------------ | --------------------------------------------- |
| `<mat-dialog>`       | `<Dialog>`         | Radix UI dialog                               |
| `<mat-snack-bar>`    | `<Sonner>` (toast) | Sonner is the recommended toast for shadcn    |
| `<mat-tooltip>`      | `<Tooltip>`        | Radix UI tooltip                              |
| `<mat-bottom-sheet>` | `<Sheet>`          | Radix UI sheet (can open from bottom)         |
| Confirmation dialogs | `<AlertDialog>`    | Radix UI alert dialog for destructive actions |

---

## 4. Custom Components

Components that don't have a direct shadcn equivalent and need to be built for the M-SACCO application.

### DataTable

The most critical custom component. Wraps TanStack Table with shadcn Table for a fully-featured data table with sorting, filtering, pagination, row selection, and export.

```tsx
// components/shared/data-table/data-table.tsx
'use client';

import { useState } from 'react';
import { type ColumnDef, type ColumnFiltersState, type SortingState, type VisibilityState, flexRender, getCoreRowModel, getFilteredRowModel, getSortedRowModel, useReactTable } from '@tanstack/react-table';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { DataTableToolbar } from './data-table-toolbar';
import { DataTablePagination } from './data-table-pagination';
import { Skeleton } from '@/components/ui/skeleton';

interface DataTableProps<TData, TValue> {
  /** Column definitions. */
  columns: ColumnDef<TData, TValue>[];
  /** Table data (current page). */
  data: TData[];
  /** Total record count for server-side pagination. */
  totalCount?: number;
  /** Current page index (0-based). */
  page?: number;
  /** Page size. */
  pageSize?: number;
  /** Whether data is currently loading. */
  isLoading?: boolean;
  /** Callback when the page changes. */
  onPageChange?: (page: number) => void;
  /** Callback when the page size changes. */
  onPageSizeChange?: (pageSize: number) => void;
  /** Callback when search input changes. */
  onSearchChange?: (search: string) => void;
  /** Search placeholder text. */
  searchPlaceholder?: string;
  /** Callback when a row is clicked. */
  onRowClick?: (row: TData) => void;
  /** Whether to show the export button. */
  showExport?: boolean;
  /** Status tabs above the table (e.g., All | Active | Closed). */
  statusTabs?: { label: string; value: string; count?: number }[];
  /** Currently selected status tab. */
  activeStatus?: string;
  /** Callback when a status tab is clicked. */
  onStatusChange?: (status: string) => void;
  /** Additional toolbar actions (e.g., "Create New" button). */
  toolbarActions?: React.ReactNode;
}

export function DataTable<TData, TValue>({ columns, data, totalCount, page = 0, pageSize = 50, isLoading, onPageChange, onPageSizeChange, onSearchChange, searchPlaceholder = 'Search...', onRowClick, showExport = false, statusTabs, activeStatus, onStatusChange, toolbarActions }: DataTableProps<TData, TValue>) {
  const [
    sorting,
    setSorting
  ] = useState<SortingState>([]);
  const [
    columnFilters,
    setColumnFilters
  ] = useState<ColumnFiltersState>([]);
  const [
    columnVisibility,
    setColumnVisibility
  ] = useState<VisibilityState>({});
  const [
    rowSelection,
    setRowSelection
  ] = useState({});

  const table = useReactTable({
    data,
    columns,
    state: {
      sorting,
      columnFilters,
      columnVisibility,
      rowSelection
    },
    onSortingChange: setSorting,
    onColumnFiltersChange: setColumnFilters,
    onColumnVisibilityChange: setColumnVisibility,
    onRowSelectionChange: setRowSelection,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
    getFilteredRowModel: getFilteredRowModel(),
    // Server-side pagination: don't use getPaginationRowModel
    manualPagination: true,
    pageCount: totalCount ? Math.ceil(totalCount / pageSize) : -1
  });

  return (
    <div className="space-y-4">
      {/* Status tabs */}
      {statusTabs && (
        <div className="flex gap-1 border-b">
          {statusTabs.map((tab) => (
            <button key={tab.value} onClick={() => onStatusChange?.(tab.value)} className={`px-4 py-2 text-sm font-medium border-b-2 transition-colors ${activeStatus === tab.value ? 'border-primary text-primary' : 'border-transparent text-muted-foreground hover:text-foreground'}`}>
              {tab.label}
              {tab.count !== undefined && <span className="ml-1.5 rounded-full bg-muted px-2 py-0.5 text-xs">{tab.count}</span>}
            </button>
          ))}
        </div>
      )}

      {/* Toolbar: search, filters, column visibility, export, custom actions */}
      <DataTableToolbar table={table} onSearchChange={onSearchChange} searchPlaceholder={searchPlaceholder} showExport={showExport} toolbarActions={toolbarActions} />

      {/* Table */}
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            {table.getHeaderGroups().map((headerGroup) => (
              <TableRow key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <TableHead key={header.id}>{header.isPlaceholder ? null : flexRender(header.column.columnDef.header, header.getContext())}</TableHead>
                ))}
              </TableRow>
            ))}
          </TableHeader>
          <TableBody>
            {isLoading ? (
              // Loading skeleton rows
              Array.from({ length: 5 }).map((_, i) => (
                <TableRow key={`skeleton-${i}`}>
                  {columns.map((_, j) => (
                    <TableCell key={`skeleton-${i}-${j}`}>
                      <Skeleton className="h-4 w-full" />
                    </TableCell>
                  ))}
                </TableRow>
              ))
            ) : table.getRowModel().rows.length ? (
              table.getRowModel().rows.map((row) => (
                <TableRow key={row.id} data-state={row.getIsSelected() && 'selected'} onClick={() => onRowClick?.(row.original)} className={onRowClick ? 'cursor-pointer hover:bg-muted/50' : ''}>
                  {row.getVisibleCells().map((cell) => (
                    <TableCell key={cell.id}>{flexRender(cell.column.columnDef.cell, cell.getContext())}</TableCell>
                  ))}
                </TableRow>
              ))
            ) : (
              <TableRow>
                <TableCell colSpan={columns.length} className="h-24 text-center">
                  No results found.
                </TableCell>
              </TableRow>
            )}
          </TableBody>
        </Table>
      </div>

      {/* Pagination */}
      <DataTablePagination table={table} totalCount={totalCount ?? data.length} page={page} pageSize={pageSize} onPageChange={onPageChange} onPageSizeChange={onPageSizeChange} />
    </div>
  );
}
```

### ApprovalActionBar

Action bar for entity approval workflows (loan approval, savings approval, client activation).

```tsx
// components/shared/approval-action-bar.tsx
'use client';

import { Button } from '@/components/ui/button';
import { useAuth } from '@/lib/auth/auth-context';
import { CheckCircle, XCircle, RotateCcw, Ban } from 'lucide-react';

interface Action {
  label: string;
  command: string;
  icon: React.ReactNode;
  variant?: 'default' | 'destructive' | 'outline' | 'ghost';
  permission: string;
}

interface ApprovalActionBarProps {
  /** Entity status code (e.g., 'loanStatusType.submitted.and.pending.approval') */
  statusCode: string;
  /** Available actions for the current status. */
  actions: Action[];
  /** Whether a mutation is in progress. */
  isLoading?: boolean;
  /** Callback when an action is clicked. */
  onAction: (command: string) => void;
}

/**
 * Approval action bar displayed below entity headers.
 *
 * Replaces the Angular pattern of conditionally showing mat-raised-buttons
 * based on entity status and user permissions.
 *
 * Usage:
 *   <ApprovalActionBar
 *     statusCode={loan.status.code}
 *     actions={[
 *       { label: 'Approve', command: 'approve', icon: <CheckCircle />, permission: 'APPROVE_LOAN' },
 *       { label: 'Reject', command: 'reject', icon: <XCircle />, variant: 'destructive', permission: 'REJECT_LOAN' },
 *     ]}
 *     onAction={(cmd) => handleAction(cmd)}
 *   />
 */
export function ApprovalActionBar({ statusCode, actions, isLoading, onAction }: ApprovalActionBarProps) {
  const { hasPermission } = useAuth();

  const visibleActions = actions.filter((action) => hasPermission(action.permission));

  if (visibleActions.length === 0) return null;

  return (
    <div className="flex flex-wrap gap-2 rounded-lg border bg-muted/50 p-3">
      {visibleActions.map((action) => (
        <Button key={action.command} variant={action.variant ?? 'default'} size="sm" disabled={isLoading} onClick={() => onAction(action.command)}>
          {action.icon}
          <span className="ml-2">{action.label}</span>
        </Button>
      ))}
    </div>
  );
}

/**
 * Predefined action sets for common entity types.
 */
export const LOAN_ACTIONS: Record<string, Action[]> = {
  'loanStatusType.submitted.and.pending.approval': [
    { label: 'Approve', command: 'approve', icon: <CheckCircle className="h-4 w-4" />, permission: 'APPROVE_LOAN' },
    { label: 'Reject', command: 'reject', icon: <XCircle className="h-4 w-4" />, variant: 'destructive', permission: 'REJECT_LOAN' },
    { label: 'Withdraw', command: 'withdrawnByApplicant', icon: <Ban className="h-4 w-4" />, variant: 'outline', permission: 'WITHDRAW_LOAN' }
  ],
  'loanStatusType.approved': [
    { label: 'Disburse', command: 'disburse', icon: <CheckCircle className="h-4 w-4" />, permission: 'DISBURSE_LOAN' },
    { label: 'Undo Approval', command: 'undoapproval', icon: <RotateCcw className="h-4 w-4" />, variant: 'outline', permission: 'APPROVE_LOAN' }
  ]
};
```

### StatusBadge

Colored badge for entity statuses used throughout the application.

```tsx
// components/shared/status-badge.tsx
import { Badge } from '@/components/ui/badge';
import { cn } from '@/lib/utils/cn';

type StatusType = 'active' | 'pending' | 'approved' | 'closed' | 'rejected' | 'withdrawn' | 'overpaid' | 'disbursed' | 'written-off' | 'rescheduled' | 'submitted' | 'invalid' | 'transfer';

const STATUS_STYLES: Record<StatusType, string> = {
  active: 'bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200',
  pending: 'bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200',
  approved: 'bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200',
  submitted: 'bg-blue-100 text-blue-800 dark:bg-blue-900 dark:text-blue-200',
  closed: 'bg-gray-100 text-gray-800 dark:bg-gray-900 dark:text-gray-200',
  rejected: 'bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200',
  withdrawn: 'bg-orange-100 text-orange-800 dark:bg-orange-900 dark:text-orange-200',
  overpaid: 'bg-purple-100 text-purple-800 dark:bg-purple-900 dark:text-purple-200',
  disbursed: 'bg-teal-100 text-teal-800 dark:bg-teal-900 dark:text-teal-200',
  'written-off': 'bg-red-100 text-red-800 dark:bg-red-900 dark:text-red-200',
  rescheduled: 'bg-indigo-100 text-indigo-800 dark:bg-indigo-900 dark:text-indigo-200',
  invalid: 'bg-gray-100 text-gray-600 dark:bg-gray-900 dark:text-gray-400',
  transfer: 'bg-cyan-100 text-cyan-800 dark:bg-cyan-900 dark:text-cyan-200'
};

/**
 * Maps Fineract status objects to a normalized status type.
 */
function normalizeStatus(status: { value?: string; code?: string }): StatusType {
  const code = (status.code ?? status.value ?? '').toLowerCase();

  if (code.includes('active') || code.includes('open')) return 'active';
  if (code.includes('pending') || code.includes('submitted')) return 'pending';
  if (code.includes('approved')) return 'approved';
  if (code.includes('closed') || code.includes('mature')) return 'closed';
  if (code.includes('rejected')) return 'rejected';
  if (code.includes('withdrawn')) return 'withdrawn';
  if (code.includes('overpaid')) return 'overpaid';
  if (code.includes('disburse')) return 'disbursed';
  if (code.includes('written')) return 'written-off';
  if (code.includes('reschedule')) return 'rescheduled';
  if (code.includes('transfer')) return 'transfer';
  return 'invalid';
}

interface StatusBadgeProps {
  /** Fineract status object with value and/or code properties. */
  status: { value?: string; code?: string };
  className?: string;
}

export function StatusBadge({ status, className }: StatusBadgeProps) {
  const type = normalizeStatus(status);
  const label = status.value ?? status.code ?? 'Unknown';

  return (
    <Badge variant="outline" className={cn('border-0 font-medium', STATUS_STYLES[type], className)}>
      {label}
    </Badge>
  );
}
```

### FileUpload

Drag-and-drop file upload component replacing Angular CDK drag-and-drop file upload patterns.

```tsx
// components/shared/file-upload.tsx
'use client';

import { useCallback, useState, type ChangeEvent, type DragEvent } from 'react';
import { Upload, X, FileText } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils/cn';

interface FileUploadProps {
  /** Accepted file types (e.g., '.pdf,.jpg,.png' or 'image/*'). */
  accept?: string;
  /** Whether multiple files can be uploaded. */
  multiple?: boolean;
  /** Maximum file size in bytes. */
  maxSize?: number;
  /** Callback when files are selected. */
  onFilesSelected: (files: File[]) => void;
  /** Currently uploaded files (controlled). */
  files?: File[];
  /** Callback to remove a file. */
  onRemove?: (index: number) => void;
  /** Custom label text. */
  label?: string;
  className?: string;
}

export function FileUpload({
  accept,
  multiple = false,
  maxSize = 5 * 1024 * 1024, // 5 MB default
  onFilesSelected,
  files = [],
  onRemove,
  label = 'Drag and drop files here, or click to browse',
  className
}: FileUploadProps) {
  const [
    isDragging,
    setIsDragging
  ] = useState(false);

  const handleDragOver = useCallback((e: DragEvent) => {
    e.preventDefault();
    setIsDragging(true);
  }, []);

  const handleDragLeave = useCallback((e: DragEvent) => {
    e.preventDefault();
    setIsDragging(false);
  }, []);

  const handleDrop = useCallback(
    (e: DragEvent) => {
      e.preventDefault();
      setIsDragging(false);
      const droppedFiles = Array.from(e.dataTransfer.files);
      const validFiles = droppedFiles.filter((f) => f.size <= maxSize);
      if (validFiles.length > 0) {
        onFilesSelected(multiple ? validFiles : [validFiles[0]]);
      }
    },
    [
      maxSize,
      multiple,
      onFilesSelected
    ]
  );

  const handleChange = useCallback(
    (e: ChangeEvent<HTMLInputElement>) => {
      const selectedFiles = Array.from(e.target.files ?? []);
      const validFiles = selectedFiles.filter((f) => f.size <= maxSize);
      if (validFiles.length > 0) {
        onFilesSelected(validFiles);
      }
      e.target.value = ''; // Reset input
    },
    [
      maxSize,
      onFilesSelected
    ]
  );

  return (
    <div className={cn('space-y-3', className)}>
      <label onDragOver={handleDragOver} onDragLeave={handleDragLeave} onDrop={handleDrop} className={cn('flex cursor-pointer flex-col items-center justify-center rounded-lg border-2 border-dashed p-8 transition-colors', isDragging ? 'border-primary bg-primary/5' : 'border-muted-foreground/25 hover:border-primary/50')}>
        <Upload className="mb-2 h-8 w-8 text-muted-foreground" />
        <p className="text-sm text-muted-foreground">{label}</p>
        <p className="mt-1 text-xs text-muted-foreground">Max file size: {(maxSize / (1024 * 1024)).toFixed(0)} MB</p>
        <input type="file" accept={accept} multiple={multiple} onChange={handleChange} className="hidden" />
      </label>

      {/* File list */}
      {files.length > 0 && (
        <ul className="space-y-2">
          {files.map((file, index) => (
            <li key={`${file.name}-${index}`} className="flex items-center justify-between rounded-md border p-2">
              <div className="flex items-center gap-2">
                <FileText className="h-4 w-4 text-muted-foreground" />
                <span className="text-sm">{file.name}</span>
                <span className="text-xs text-muted-foreground">({(file.size / 1024).toFixed(1)} KB)</span>
              </div>
              {onRemove && (
                <Button variant="ghost" size="icon" onClick={() => onRemove(index)}>
                  <X className="h-4 w-4" />
                </Button>
              )}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### FormDialog

Renders a form inside a dialog. Used for quick actions like adding a charge, assigning staff, or adding a note.

```tsx
// components/shared/form-dialog.tsx
'use client';

import { type ReactNode } from 'react';
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from '@/components/ui/dialog';
import { Button } from '@/components/ui/button';

interface FormDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title: string;
  description?: string;
  children: ReactNode;
  onSubmit: () => void;
  submitLabel?: string;
  cancelLabel?: string;
  isSubmitting?: boolean;
  /** Dialog width variant. */
  size?: 'sm' | 'md' | 'lg' | 'xl';
}

const SIZE_MAP = {
  sm: 'max-w-sm',
  md: 'max-w-md',
  lg: 'max-w-lg',
  xl: 'max-w-xl'
};

export function FormDialog({ open, onOpenChange, title, description, children, onSubmit, submitLabel = 'Submit', cancelLabel = 'Cancel', isSubmitting = false, size = 'md' }: FormDialogProps) {
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className={SIZE_MAP[size]}>
        <DialogHeader>
          <DialogTitle>{title}</DialogTitle>
          {description && <DialogDescription>{description}</DialogDescription>}
        </DialogHeader>

        <form
          onSubmit={(e) => {
            e.preventDefault();
            onSubmit();
          }}
          className="space-y-4"
        >
          {children}

          <DialogFooter>
            <Button type="button" variant="outline" onClick={() => onOpenChange(false)} disabled={isSubmitting}>
              {cancelLabel}
            </Button>
            <Button type="submit" disabled={isSubmitting}>
              {isSubmitting ? 'Submitting...' : submitLabel}
            </Button>
          </DialogFooter>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

---

## 5. Icon Strategy

The Angular app uses FontAwesome 6 via `@fortawesome/angular-fontawesome`. The Next.js app uses **lucide-react** as the primary icon library (default for shadcn/ui) with a mapping table for migration.

### Why lucide-react

- Default for shadcn/ui (consistent with component library)
- Tree-shakeable (import only used icons)
- Consistent 24x24 grid, 2px stroke
- 1500+ icons covering most use cases
- TypeScript types included

### Icon Mapping Table

The Angular app uses FontAwesome icons extensively in the toolbar, sidebar, and throughout feature modules. Below is the mapping for the most commonly used icons.

| FontAwesome Icon     | Usage in Angular App | lucide-react Equivalent | Import                                                             |
| -------------------- | -------------------- | ----------------------- | ------------------------------------------------------------------ |
| `fa-bars`            | Sidebar toggle       | `Menu`                  | `import { Menu } from 'lucide-react'`                              |
| `fa-chevron-left`    | Navigation back      | `ChevronLeft`           | `import { ChevronLeft } from 'lucide-react'`                       |
| `fa-chevron-right`   | Navigation forward   | `ChevronRight`          | `import { ChevronRight } from 'lucide-react'`                      |
| `fa-university`      | Office/institution   | `Building2`             | `import { Building2 } from 'lucide-react'`                         |
| `fa-money-bill-alt`  | Accounting/finance   | `Banknote`              | `import { Banknote } from 'lucide-react'`                          |
| `fa-chart-bar`       | Reports              | `BarChart3`             | `import { BarChart3 } from 'lucide-react'`                         |
| `fa-shield-alt`      | System admin         | `ShieldCheck`           | `import { ShieldCheck } from 'lucide-react'`                       |
| `fa-info`            | Information          | `Info`                  | `import { Info } from 'lucide-react'`                              |
| `fa-question-circle` | Help                 | `HelpCircle`            | `import { HelpCircle } from 'lucide-react'`                        |
| `fa-user`            | User profile         | `User`                  | `import { User } from 'lucide-react'`                              |
| `fa-cog`             | Settings             | `Settings`              | `import { Settings } from 'lucide-react'`                          |
| `fa-sign-out-alt`    | Logout               | `LogOut`                | `import { LogOut } from 'lucide-react'`                            |
| `fa-search`          | Search               | `Search`                | `import { Search } from 'lucide-react'`                            |
| `fa-bell`            | Notifications        | `Bell`                  | `import { Bell } from 'lucide-react'`                              |
| `fa-home`            | Home/dashboard       | `Home`                  | `import { Home } from 'lucide-react'`                              |
| `fa-users`           | Clients/groups       | `Users`                 | `import { Users } from 'lucide-react'`                             |
| `fa-user-plus`       | Create client        | `UserPlus`              | `import { UserPlus } from 'lucide-react'`                          |
| `fa-briefcase`       | Loans                | `Briefcase`             | `import { Briefcase } from 'lucide-react'`                         |
| `fa-piggy-bank`      | Savings              | `PiggyBank`             | `import { PiggyBank } from 'lucide-react'`                         |
| `fa-building`        | Centers              | `Building`              | `import { Building } from 'lucide-react'`                          |
| `fa-exchange-alt`    | Transfers            | `ArrowLeftRight`        | `import { ArrowLeftRight } from 'lucide-react'`                    |
| `fa-tasks`           | Tasks/checker inbox  | `ClipboardCheck`        | `import { ClipboardCheck } from 'lucide-react'`                    |
| `fa-file-alt`        | Documents/reports    | `FileText`              | `import { FileText } from 'lucide-react'`                          |
| `fa-edit`            | Edit                 | `Pencil`                | `import { Pencil } from 'lucide-react'`                            |
| `fa-trash`           | Delete               | `Trash2`                | `import { Trash2 } from 'lucide-react'`                            |
| `fa-plus`            | Add/create           | `Plus`                  | `import { Plus } from 'lucide-react'`                              |
| `fa-check`           | Approve/confirm      | `Check`                 | `import { Check } from 'lucide-react'`                             |
| `fa-times`           | Close/cancel/reject  | `X`                     | `import { X } from 'lucide-react'`                                 |
| `fa-download`        | Download/export      | `Download`              | `import { Download } from 'lucide-react'`                          |
| `fa-upload`          | Upload               | `Upload`                | `import { Upload } from 'lucide-react'`                            |
| `fa-eye`             | View                 | `Eye`                   | `import { Eye } from 'lucide-react'`                               |
| `fa-eye-slash`       | Hide                 | `EyeOff`                | `import { EyeOff } from 'lucide-react'`                            |
| `fa-calendar`        | Date picker          | `Calendar`              | `import { Calendar } from 'lucide-react'`                          |
| `fa-globe`           | Language             | `Globe`                 | `import { Globe } from 'lucide-react'`                             |
| `fa-moon`            | Dark mode            | `Moon`                  | `import { Moon } from 'lucide-react'`                              |
| `fa-sun`             | Light mode           | `Sun`                   | `import { Sun } from 'lucide-react'`                               |
| `fa-spinner`         | Loading              | `Loader2`               | `import { Loader2 } from 'lucide-react'` (use with `animate-spin`) |
| `fa-arrow-left`      | Back                 | `ArrowLeft`             | `import { ArrowLeft } from 'lucide-react'`                         |
| `fa-arrow-right`     | Forward              | `ArrowRight`            | `import { ArrowRight } from 'lucide-react'`                        |
| `fa-key`             | Permissions/roles    | `Key`                   | `import { Key } from 'lucide-react'`                               |
| `fa-lock`            | Lock/secure          | `Lock`                  | `import { Lock } from 'lucide-react'`                              |
| `fa-unlock`          | Unlock               | `Unlock`                | `import { Unlock } from 'lucide-react'`                            |

### FontAwesome Fallback

For any FontAwesome icons without a lucide equivalent (unlikely, but possible for brand icons or niche symbols), keep `@fortawesome/react-fontawesome` as a fallback dependency:

```typescript
// Only install if needed for specific icons
// npm install @fortawesome/react-fontawesome @fortawesome/fontawesome-svg-core @fortawesome/free-solid-svg-icons

import { FontAwesomeIcon } from '@fortawesome/react-fontawesome';
import { faSomeNicheIcon } from '@fortawesome/free-solid-svg-icons';

// Usage (avoid this pattern; prefer lucide)
<FontAwesomeIcon icon={faSomeNicheIcon} />
```

---

## 6. Dark Mode

The Angular app supports dark mode via `SettingsService.themeDarkEnabled` and Angular Material's dark theme. The Next.js implementation uses `next-themes` for SSR-safe theme switching.

### Setup

```tsx
// app/layout.tsx
import { ThemeProvider } from 'next-themes';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
          {/* Other providers */}
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}
```

### Theme Toggle Component

```tsx
// components/layout/theme-toggle.tsx
'use client';

import { useTheme } from 'next-themes';
import { Button } from '@/components/ui/button';
import { Moon, Sun } from 'lucide-react';
import { DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger } from '@/components/ui/dropdown-menu';

/**
 * Theme toggle button.
 *
 * Replaces the Angular ThemingService toggle behavior.
 *
 * Options:
 * - Light: Forces light mode
 * - Dark: Forces dark mode
 * - System: Follows OS preference (not available in Angular app -- new feature)
 */
export function ThemeToggle() {
  const { setTheme, theme } = useTheme();

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="icon">
          <Sun className="h-5 w-5 rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
          <Moon className="absolute h-5 w-5 rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
          <span className="sr-only">Toggle theme</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => setTheme('light')}>Light</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme('dark')}>Dark</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme('system')}>System</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### How It Works

1. `next-themes` adds a `class="dark"` to the `<html>` element when dark mode is active.
2. The CSS variables in `globals.css` (Section 2) define different values under `.dark`.
3. All shadcn/ui components reference these CSS variables, so they automatically switch colors.
4. Tailwind's `dark:` variant prefix works because `darkMode: ['class']` is configured.
5. The theme preference is persisted in `localStorage` by `next-themes` (key: `theme`).

### Migration from Angular ThemingService

| Angular Pattern                                  | Next.js Pattern                                  |
| ------------------------------------------------ | ------------------------------------------------ |
| `settingsService.themeDarkEnabled`               | `useTheme().theme === 'dark'`                    |
| `settingsService.setThemeDarkEnabled(true)`      | `useTheme().setTheme('dark')`                    |
| `localStorage.getItem('mifosXThemeDarkEnabled')` | Handled by `next-themes` internally              |
| Angular Material theme overlay mixin             | `.dark` CSS class + CSS variables                |
| `@include mat.core-theme($dark-theme)`           | CSS variables swap values under `.dark` selector |

### Respecting User's OS Preference

The Angular app does not support system-level dark mode preference. The Next.js app adds this as a new feature via `enableSystem` in `ThemeProvider`. When `theme` is set to `system`, `next-themes` watches the `prefers-color-scheme` media query and automatically switches.
