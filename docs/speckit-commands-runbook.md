# Spec-Kit Commands Runbook

Copy-paste commands for each app. Run them **in order** within a Claude Code session inside each repo.

---

## Prerequisites

```bash
pip install uv
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

---

---

# APP 1: msacco-angular

```bash
cd D:/msacco-angular
specify init . --here --ai claude --script ps
```

## 1. Constitution

```
/speckit.constitution Create principles for an M-SACCO microfinance platform: Angular 20 with spartan/ui (shadcn for Angular) replacing Angular Material. Fineract REST API as sole backend — no custom backend code. Tailwind CSS utility-first styling with M-SACCO design tokens (primary #1074b9, accent #b4d575). Standalone components with signals (modern Angular patterns). Accessibility WCAG 2.1 AA via spartan/ui brain layer. i18n for 12 languages via ngx-translate. Mobile-first responsive design. Jest unit tests for all new components. Playwright E2E for critical user flows. ESLint + Stylelint + Prettier compliance. No direct DOM manipulation — use Angular APIs. Reuse existing services and patterns from the codebase.
```

---

## 2. Features (specify → plan → tasks → implement)

Run each block sequentially. After each `/speckit.implement`, run `npm run lint && npm run test && npm run build`.

---

### Feature 1: Spartan/UI Migration (foundation — do first)

```
/speckit.specify Migrate the entire UI from Angular Material to spartan/ui (brain+helm architecture). Replace all 82 mat-* component tags across 200+ template files. 5 phases: foundation (install spartan, Tailwind, CDK), forms (inputs, selects, checkboxes, datepicker), tables/nav (data tables, tabs, toolbar, menus), dialogs (all dialog components), layout/cleanup (sidenav, cards, remove @angular/material). See docs/deliverable-1-angular/TECH-012-Spartan-UI-Migration.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 2: Navigation Restructure

```
/speckit.specify Redesign the shell navigation from Angular Material mega-menu dropdowns to a flat horizontal top nav bar matching M-SACCO wireframes. Nav items: Clients, Groups, Products, Reports, Accounting, Configuration, Search. Make sidenav mobile-only (hidden on desktop, hamburger toggle on <768px). Rebrand footer to "Help • Support • Logout • ©M-Sacco". Use spartan/ui components (brn-menu, hlm-menu). See docs/deliverable-1-angular/TECH-001-Navigation-Restructure.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 3: Shared Components

```
/speckit.specify Create 5 new shared components with spartan/ui: (1) MsaccoDataTable — wraps spartan table with status tabs, Copy/Excel/PDF export, "Showing X to Y of Z" pagination text. (2) StatusTabs — reusable tab bar with count badges (e.g. "Pending Approval (12)"). (3) MsaccoWizard — multi-step stepper with numbered circles and connecting lines. (4) ApprovalActionBar — sticky bottom bar with Approve/Reject/More Actions buttons. (5) StatusBadge — colored chip for entity statuses (Active=green, Pending=yellow, Closed=red). See docs/deliverable-1-angular/TECH-002-Shared-Components.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 4: Client Detail Tabs

```
/speckit.specify Add 7 new tabs to the client detail view: (1) CRB Account — credit score card, check history, status indicators. (2) Business Details — business name, type, start date, address, postal code, county. (3) PPI — Progress out of Poverty Index surveys with scoring. (4) Financial Statements — income/expense statements with totals. (5) Multi-funding — multiple funding sources per client. (6) Client Tasks — task assignments with status tracking. (7) Guarantor For — loans where this client is a guarantor. Each tab has its own component, service, and route. See docs/deliverable-1-angular/TECH-003-Client-Tabs.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 5: Group Wizard

```
/speckit.specify Convert the group creation from a single-page form to a 3-step wizard using the MsaccoWizard shared component. Step 1: Group Info (name, branch, loan officer, registration number, submission date, meeting location/days/frequency). Step 2: Select Clients (search existing clients, add/remove members from a table). Step 3: Overview (review all details before submission). See docs/deliverable-1-angular/TECH-004-Group-Wizard.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 6: Loan Guarantor

```
/speckit.specify Add enhanced guarantor management step to the loan creation wizard. Support 3 selection modes: (1) Existing Client — search by name/account, auto-fill details. (2) New Guarantor — manual entry form for external guarantors. (3) Group Members — select from group member list (if group loan). Fields: relationship, guarantee amount, savings on hold. Fineract API: POST /loans/{loanId}/guarantors. See docs/deliverable-1-angular/TECH-005-Loan-Guarantor.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 7: Client Transfers

```
/speckit.specify Create a new Client Transfers module with 4 pages: (1) Transfer Between Branches — select client, destination branch, reason. (2) Transfer Between Groups — select client, destination group. (3) Pending Approvals — list of transfers awaiting approval with approve/reject actions. (4) Transfer History — audit trail with filters by date, branch, status. Centralized ClientTransfersService using Fineract API: POST /clients/{clientId}/transfer. See docs/deliverable-1-angular/TECH-006-Client-Transfers.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 8: Data Exports

```
/speckit.specify Create a Data Exports module with wizard-based export creation: Step 1 — select entity type (clients, loans, groups, savings). Step 2 — drag-and-drop field selection using Angular CDK DragDropModule for column ordering. Step 3 — filter builder (date range, status, branch, custom filters). Step 4 — preview first 10 rows. Step 5 — export to CSV/Excel/PDF (using existing ExcelJS + jsPDF deps). Include saved export templates. See docs/deliverable-1-angular/TECH-007-Data-Exports.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 9: Short Codes

```
/speckit.specify Create a Short Codes CRUD module for SMS/USSD short code management. List page with MsaccoDataTable showing all short codes. Create/Edit form with fields: code (numeric, validated for uniqueness), name, description, type (SMS/USSD), provider, status (active/inactive). Delete with confirmation dialog. Custom validators for code format. See docs/deliverable-1-angular/TECH-008-Short-Codes.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 10: SMS Enhancement

```
/speckit.specify Enhance the SMS campaigns module with: (1) Template variable insertion toolbar — clickable buttons for {{firstname}}, {{lastname}}, {{LoanAmount}}, {{DueDate}}, {{OutstandingBalance}}, etc. that insert into message textarea. (2) Message preview panel — shows rendered message with sample client data substitution. (3) Business rule configuration — send conditions (e.g., overdue > 7 days), frequency limits, time-of-day restrictions. See docs/deliverable-1-angular/TECH-009-SMS-Enhancement.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 11: MPESA Integration

```
/speckit.specify Integrate MPESA mobile money as a payment channel: (1) Loan product config — add MPESA tab to loan product wizard with paybill number, account reference format, till number. (2) Disbursement — MPESA as disbursement channel option, B2C API integration. (3) Repayment — MPESA as repayment method, C2B confirmation handling. (4) Accounting — fund source mapping for MPESA payment channels in GL account configuration. See docs/deliverable-1-angular/TECH-010-MPESA-Integration.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 12: CRB Integration

```
/speckit.specify Integrate Credit Reference Bureau (CRB) checks: (1) Client CRB tab — credit score card with color-coded rating (green/yellow/red), check history table, last checked date. (2) Loan application credit gate — auto-trigger CRB check during loan approval, block approval if score below threshold, manual override with reason. (3) Loan product CRB config — enable/disable CRB check per product, minimum score threshold, CRB provider selection. See docs/deliverable-1-angular/TECH-011-CRB-Integration.md for full specification.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

---

# APP 2: msacco-nextjs

```bash
cd D:/msacco-nextjs
specify init . --here --ai claude --script ps
```

## 1. Constitution

```
/speckit.constitution Create principles for an M-SACCO Next.js microfinance platform: Next.js 15+ with App Router (server components by default, client components only for interactivity). shadcn/ui + Radix UI + Tailwind CSS for UI with M-SACCO design tokens (primary #1074b9, accent #b4d575). TanStack Query for server state, Zustand for UI state. React Hook Form + Zod for forms and validation. NextAuth.js for authentication (Basic Auth + OIDC dual-mode). next-intl for i18n (12 languages). Fineract REST API as sole backend — no custom backend. TypeScript strict mode. Vitest + React Testing Library for unit tests. Playwright for E2E. Mobile-first responsive, WCAG 2.1 AA accessible.
```

---

## 2. Features (specify → plan → tasks → implement)

---

### Feature 1: Project Foundation

```
/speckit.specify Set up the Next.js 15+ project foundation: (1) App Router with (auth) and (dashboard) route groups. (2) Tailwind CSS with M-SACCO theme tokens from docs/deliverable-2-design-system/DESIGN-001-Foundations.md. (3) shadcn/ui component installation (button, input, select, table, dialog, tabs, badge, card, sheet, dropdown-menu, command, toast). (4) Fineract API client — Axios instance with request/response interceptors for auth headers, tenant header, error handling, base URL config. (5) NextAuth.js dual-mode auth — Basic Auth credentials provider + OIDC provider for Zitadel. (6) Shell layout — toolbar with flat horizontal nav (Clients, Groups, Products, Reports, Accounting, Configuration, Search), breadcrumbs, footer. (7) React Query provider + devtools. (8) next-intl setup with 12 language JSON files. (9) Dark mode with next-themes. See docs/deliverable-3-nextjs/NEXTJS-001-Architecture.md, docs/deliverable-3-nextjs/NEXTJS-002-API-Client.md, docs/deliverable-3-nextjs/NEXTJS-003-Authentication.md, docs/deliverable-3-nextjs/NEXTJS-005-Component-Library.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 2: State Management & Forms

```
/speckit.specify Implement the state management and form layer: (1) TanStack Query hooks for all Fineract API endpoints — useClients, useClient, useCreateClient, useLoans, useLoan, useGroups, useAccounting, useProducts, etc. with proper cache keys, stale times, and optimistic updates. (2) Zustand stores — useThemeStore, useSettingsStore, useSidebarStore, useAlertStore. (3) React Hook Form + Zod validation schemas for all entity forms (client, loan, group, product). (4) URL state management with nuqs for table filters, pagination, and sort. See docs/deliverable-3-nextjs/NEXTJS-004-State-Management.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 3: Shared Components

```
/speckit.specify Build shared components using shadcn/ui: (1) DataTable — TanStack Table with column sorting, pagination, status tabs, row selection, Copy/Excel/PDF export. (2) Wizard — multi-step form with step indicator, validation per step, previous/next navigation. (3) ApprovalActionBar — sticky bar with approve/reject/more actions. (4) StatusBadge — colored badge variants for entity statuses. (5) FileUpload — drag-and-drop with preview. (6) FormDialog, DeleteDialog, ConfirmationDialog — standard dialog patterns. See docs/deliverable-3-nextjs/NEXTJS-005-Component-Library.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 4: Clients Module

```
/speckit.specify Implement the Clients module: (1) Client list page — server component with DataTable, status tabs (All, Pending, Active, Closed), search, "Create Client" button. (2) Create client wizard — 5 steps: Personal Info, Family Members, Identification, Address, Additional Info. (3) Client detail layout with 14 tabs: General, CRB Account, Identification, Business Details, Documents, PPI, Family, Financial Statements, Multi-funding, Tasks, Guarantor For, Audit, Notes, Datatable. (4) Client actions — activate, close, transfer, assign staff. (5) Edit client form. Migrate from Angular services to React Query hooks. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 5: Groups Module

```
/speckit.specify Implement the Groups module: (1) Group list page with status tabs and DataTable. (2) Create group 3-step wizard: Group Info, Select Clients, Overview. (3) Group detail layout with tabs: General, Clients, Documents, Notes, Calendar. (4) Group actions — activate, close, approve/reject. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 6: Loans Module

```
/speckit.specify Implement the Loans module: (1) Loan list page with status tabs (Pending, Active, Overpaid, Closed). (2) Create loan 6-step wizard: Terms, Settings, Collateral, Guarantor (3 modes: existing client, new, group members), Repayment Schedule preview, Overview. (3) Loan detail layout with tabs: Summary, Repayment Schedule, Transactions, Charges, Collateral, Guarantors, Documents. (4) Loan actions — approve, reject, disburse, make repayment, waive interest, write off. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 7: Accounting Module

```
/speckit.specify Implement the Accounting module: (1) Chart of Accounts — tree view with create/edit GL accounts. (2) Journal Entries — list, create manual entry with debit/credit rows. (3) Closing Entries — period close with date selection. (4) Financial Reports — trial balance, income statement, balance sheet with date range filters. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 8: Products Module

```
/speckit.specify Implement the Products module: (1) Loan Product list with DataTable. (2) Create Loan Product 10-step wizard: Details, Currency, Terms, Settings, Charges, Accounting, MPESA Config, CRB Config, Fund Sources, Summary. (3) Loan Product detail view. (4) Saving Products and Charges CRUD pages. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 9: Organization Module

```
/speckit.specify Implement the Organization module: (1) Offices — tree view CRUD. (2) Employees — list, create, detail. (3) SMS Campaigns — list, create with template variable insertion ({{firstname}}, {{LoanAmount}}, etc.), message preview, campaign scheduling. (4) Holidays — calendar view with CRUD. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 10: System Module

```
/speckit.specify Implement the System module: (1) Manage Codes — code/code value CRUD. (2) Data Tables — manage custom data tables. (3) Scheduler Jobs — list, run, configure schedules. (4) Audit Trails — searchable log with filters. (5) Reports — report list, run report with parameters. (6) Users — user list, create/edit with role assignment. (7) Roles & Permissions — role CRUD with permission checkboxes. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 11: New M-SACCO Features

```
/speckit.specify Implement 3 new M-SACCO features as Next.js modules: (1) Data Exports — wizard with entity type selection, drag-and-drop field ordering (@dnd-kit/core), filter builder, preview, export to CSV/Excel/PDF. (2) Short Codes — CRUD for SMS/USSD short codes with code validation. (3) Client Transfers — transfer between branches/groups, pending approvals list, transfer history with audit trail. These are greenfield features with no Angular equivalent to migrate.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 12: Routing & i18n

```
/speckit.specify Set up all remaining routes and i18n: (1) Complete App Router file-system routing for all 358 routes per docs/deliverable-3-nextjs/NEXTJS-006-Routing.md — including parallel routes for modals, loading.tsx and error.tsx for each route segment, not-found.tsx pages. (2) Migrate all 12 language JSON translation files from Angular's src/translations/ to next-intl messages/ directory. (3) Language switcher component in toolbar. (4) RTL support detection. See docs/deliverable-3-nextjs/NEXTJS-006-Routing.md and docs/deliverable-3-nextjs/NEXTJS-007-i18n.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

### Feature 13: Testing & Deployment

```
/speckit.specify Set up testing infrastructure and deployment: (1) Vitest config with React Testing Library, MSW for API mocking, coverage thresholds (80% for new code). (2) Unit tests for all shared components and React Query hooks. (3) Playwright E2E tests for critical flows — login, create client, create loan, group creation, client transfer. (4) Docker multi-stage build with standalone Next.js output. (5) CI/CD pipeline — lint, test, build, deploy stages. (6) Environment variable configuration for dev/staging/prod. See docs/deliverable-3-nextjs/NEXTJS-009-Testing.md and docs/deliverable-3-nextjs/NEXTJS-010-Deployment.md.
```

```
/speckit.plan
```

```
/speckit.tasks
```

```
/speckit.implement
```

---

---

# Quick Reference

## After each feature implementation:

```bash
# Angular
npm run lint && npm run test && npm run build
git add -A && git commit -m "feat: <feature-name>"
git push origin main

# Next.js
npm run lint && npm run test && npm run build
git add -A && git commit -m "feat: <feature-name>"
git push origin main
```

## Optional quality commands (run between plan and tasks):

```
/speckit.clarify     # Ask up to 5 clarification questions about the spec
/speckit.analyze     # Cross-artifact consistency check across spec/plan/tasks
/speckit.checklist   # Generate a quality checklist for the feature
```
