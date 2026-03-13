# Spec-Kit Integration Plan for M-SACCO Repos

## Context

Both GitHub repos for the M-SACCO project have 27 technical documents:

- **msacco-angular** (`stephencoduor/msacco-angular`) — TECH-001 to TECH-012, DESIGN-001 to DESIGN-005
- **msacco-nextjs** (`stephencoduor/msacco-nextjs`) — NEXTJS-001 to NEXTJS-010

We use **[spec-kit](https://github.com/github/spec-kit)** (Spec-Driven Development toolkit) with Claude CLI to convert these docs into executable specs via `/speckit.*` slash commands.

---

## Step 1: Install spec-kit CLI

```bash
pip install uv   # or: curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

---

## Step 2: Initialize spec-kit in msacco-angular

```bash
cd D:/web-app
specify init . --here --ai claude --script ps
```

Creates `.speckit/` directory with Claude CLI integration.

---

## Step 3: Constitution (msacco-angular)

```
/speckit.constitution
```

Principles:

- Angular 20 with spartan/ui replacing Angular Material
- Fineract REST API as sole backend
- Tailwind CSS utility-first with M-SACCO design tokens
- Standalone components with signals
- WCAG 2.1 AA via spartan/ui brain layer
- i18n: 12 languages via ngx-translate
- Mobile-first responsive design
- Jest unit tests for all new components
- Playwright E2E for critical flows
- ESLint + Stylelint + Prettier compliance
- No direct DOM manipulation
- Reuse existing services and patterns

---

## Step 4: Specify Features (msacco-angular)

Run `/speckit.specify` one at a time, feeding existing tech docs:

| #   | Feature                | Doc Reference | Command Summary                                                            |
| --- | ---------------------- | ------------- | -------------------------------------------------------------------------- |
| 1   | Navigation Restructure | TECH-001      | Flat horizontal nav, mobile sidenav, M-SACCO footer                        |
| 2   | Shared Components      | TECH-002      | DataTable, StatusTabs, Wizard, ApprovalBar, StatusBadge                    |
| 3   | Spartan/UI Migration   | TECH-012      | Replace 82 mat-\* tags across 200+ templates, 5 phases                     |
| 4   | Client Detail Tabs     | TECH-003      | 7 new tabs: CRB, Business, PPI, Financial, Multi-funding, Tasks, Guarantor |
| 5   | Group Wizard           | TECH-004      | 3-step wizard: Info, Select Clients, Overview                              |
| 6   | Loan Guarantor         | TECH-005      | 3 modes: existing client, new guarantor, group members                     |
| 7   | Client Transfers       | TECH-006      | Branch/group transfer, pending approvals, history                          |
| 8   | Data Exports           | TECH-007      | Drag-and-drop fields, filters, CSV/Excel/PDF export                        |
| 9   | Short Codes            | TECH-008      | SMS/USSD short code CRUD                                                   |
| 10  | SMS Enhancement        | TECH-009      | Template variables, message preview, business rules                        |
| 11  | MPESA Integration      | TECH-010      | Payment channel in loans, disbursements, repayments                        |
| 12  | CRB Integration        | TECH-011      | CRB tab, credit gate, loan product config                                  |

### Full `/speckit.specify` Commands

**Feature 1: Navigation Restructure**

```
/speckit.specify Redesign the shell navigation from Angular Material mega-menu dropdowns to a flat horizontal top nav bar matching M-SACCO wireframes. Items: Clients, Groups, Products, Reports, Accounting, Configuration, Search. Make sidenav mobile-only. Rebrand footer to "Help • Support • Logout • ©M-Sacco". Use spartan/ui components. See docs/deliverable-1-angular/TECH-001-Navigation-Restructure.md for full specification.
```

**Feature 2: Shared Components**

```
/speckit.specify Create 5 new shared components with spartan/ui: MsaccoDataTable (with status tabs, export, pagination), StatusTabs, MsaccoWizard (stepper), ApprovalActionBar, StatusBadge. See docs/deliverable-1-angular/TECH-002-Shared-Components.md for full specification.
```

**Feature 3: Spartan/UI Migration**

```
/speckit.specify Migrate the entire UI from Angular Material to spartan/ui (brain+helm architecture). Replace all 82 mat-* component tags across 200+ template files. 5 phases: foundation, forms, tables/nav, dialogs, layout/cleanup. See docs/deliverable-1-angular/TECH-012-Spartan-UI-Migration.md for full specification.
```

**Feature 4: Client Detail Tabs**

```
/speckit.specify Add 7 new tabs to the client detail view: CRB Account, Business Details, PPI, Financial Statements, Multi-funding, Client Tasks, Guarantor For. Each tab has its own component, service, and route. See docs/deliverable-1-angular/TECH-003-Client-Tabs.md for full specification.
```

**Feature 5: Group Wizard**

```
/speckit.specify Convert the group creation from a single-page form to a 3-step wizard: Group Info, Select Clients, Overview. See docs/deliverable-1-angular/TECH-004-Group-Wizard.md for full specification.
```

**Feature 6: Loan Guarantor**

```
/speckit.specify Add enhanced guarantor management step to the loan creation wizard with 3 modes: Existing Client search, New Guarantor form, Group Members selection. See docs/deliverable-1-angular/TECH-005-Loan-Guarantor.md for full specification.
```

**Feature 7: Client Transfers**

```
/speckit.specify Create a new Client Transfers module with pages for transfer between branches, transfer between groups, pending approvals, and transfer history. See docs/deliverable-1-angular/TECH-006-Client-Transfers.md for full specification.
```

**Feature 8: Data Exports**

```
/speckit.specify Create a Data Exports module with drag-and-drop field selection using Angular CDK, filter builder, preview, and export to CSV/Excel/PDF. See docs/deliverable-1-angular/TECH-007-Data-Exports.md for full specification.
```

**Feature 9: Short Codes**

```
/speckit.specify Create a Short Codes CRUD module for SMS/USSD short code management. See docs/deliverable-1-angular/TECH-008-Short-Codes.md for full specification.
```

**Feature 10: SMS Enhancement**

```
/speckit.specify Enhance the SMS campaigns module with template variable insertion, message preview with sample data, and business rule configuration. See docs/deliverable-1-angular/TECH-009-SMS-Enhancement.md for full specification.
```

**Feature 11: MPESA Integration**

```
/speckit.specify Integrate MPESA mobile money as a payment channel in loan products, disbursements, repayments, and accounting fund sources. See docs/deliverable-1-angular/TECH-010-MPESA-Integration.md for full specification.
```

**Feature 12: CRB Integration**

```
/speckit.specify Integrate Credit Reference Bureau checks with a client CRB tab (score card, history), loan application credit gate, and loan product CRB configuration. See docs/deliverable-1-angular/TECH-011-CRB-Integration.md for full specification.
```

---

## Step 5: Plan & Tasks (msacco-angular)

For each specified feature:

```
/speckit.plan      # Technical implementation plan
/speckit.clarify   # (optional) Identify ambiguities
/speckit.analyze   # (optional) Cross-artifact consistency
/speckit.tasks     # Ordered actionable tasks
```

---

## Step 6: Implement (msacco-angular)

```
/speckit.implement
```

**Recommended order** (dependencies first):

1. Spartan/UI Migration (TECH-012) — foundation
2. Navigation Restructure (TECH-001) — shell layout
3. Shared Components (TECH-002) — reusable blocks
4. Client Detail Tabs (TECH-003)
5. Group Wizard (TECH-004)
6. Loan Guarantor (TECH-005)
7. Client Transfers (TECH-006)
8. Data Exports (TECH-007)
9. Short Codes (TECH-008)
10. SMS Enhancement (TECH-009)
11. MPESA Integration (TECH-010)
12. CRB Integration (TECH-011)

---

## Step 7: Initialize spec-kit in msacco-nextjs

```bash
cd D:/msacco-nextjs
specify init . --here --ai claude --script ps
```

---

## Step 8: Constitution (msacco-nextjs)

```
/speckit.constitution
```

Principles:

- Next.js 15+ with App Router (server components default)
- shadcn/ui + Radix UI + Tailwind CSS
- TanStack Query for server state, Zustand for UI state
- React Hook Form + Zod for forms
- NextAuth.js (Basic Auth + OIDC dual-mode)
- next-intl for 12 languages
- Fineract REST API as sole backend
- TypeScript strict mode
- Vitest + React Testing Library + Playwright
- Mobile-first, WCAG 2.1 AA
- Server Components for data fetching, Client Components only for interactivity

---

## Step 9: Specify Features (msacco-nextjs)

| #     | Feature              | Doc References            | Summary                                                                         |
| ----- | -------------------- | ------------------------- | ------------------------------------------------------------------------------- |
| 1     | Foundation           | NEXTJS-001, 002, 003, 005 | App Router, Tailwind theme, shadcn/ui, API client, auth, shell, i18n, dark mode |
| 2     | State & Forms        | NEXTJS-004                | TanStack Query hooks, Zustand stores, RHF + Zod, URL state                      |
| 3     | Shared Components    | NEXTJS-005                | DataTable, Wizard, ApprovalBar, StatusBadge, dialogs                            |
| 4-10  | Module Migration     | NEXTJS-008                | Clients, Groups, Loans, Accounting, Products, Organization, System              |
| 11-13 | New M-SACCO Features | —                         | Data Exports, Short Codes, Client Transfers                                     |

### Full `/speckit.specify` Commands (Next.js)

**Feature 1: Foundation**

```
/speckit.specify Set up the Next.js project foundation: App Router, Tailwind with M-SACCO theme tokens, shadcn/ui components, Fineract API client (Axios + interceptors), NextAuth.js dual-mode auth, shell layout (toolbar, breadcrumbs, footer), React Query provider, next-intl with 12 languages, dark mode with next-themes. See docs/deliverable-3-nextjs/NEXTJS-001-Architecture.md, docs/deliverable-3-nextjs/NEXTJS-002-API-Client.md, docs/deliverable-3-nextjs/NEXTJS-003-Authentication.md, docs/deliverable-3-nextjs/NEXTJS-005-Component-Library.md.
```

**Feature 2: State & Forms**

```
/speckit.specify Implement state management: TanStack Query hooks for all Fineract endpoints, Zustand stores (theme, settings, sidebar, alerts), React Hook Form + Zod validation schemas, URL state with nuqs. See docs/deliverable-3-nextjs/NEXTJS-004-State-Management.md.
```

**Feature 3: Shared Components**

```
/speckit.specify Build shared components: DataTable (TanStack Table + status tabs + export), Wizard (multi-step form), ApprovalActionBar, StatusBadge, FileUpload, FormDialog, DeleteDialog, ConfirmationDialog. See docs/deliverable-3-nextjs/NEXTJS-005-Component-Library.md.
```

**Feature 4-10: Module Migration**

```
/speckit.specify Implement the Clients module: list page, create wizard (5 steps), detail layout with 14 tabs, edit, actions. See docs/deliverable-3-nextjs/NEXTJS-008-Module-Migration-Guide.md.
```

Repeat for: Groups, Loans, Accounting, Products, Organization, System.

**Feature 11-13: New M-SACCO Features**

```
/speckit.specify Implement Data Exports, Short Codes, and Client Transfers modules as new Next.js features.
```

---

## Step 10: Plan, Tasks & Implement (msacco-nextjs)

Same workflow as Angular:

```
/speckit.plan → /speckit.tasks → /speckit.implement
```

---

## Workflow Summary

```
For each repo (msacco-angular, msacco-nextjs):
  1. specify init . --here --ai claude --script ps
  2. /speckit.constitution  →  constitution.md
  3. For each feature:
     a. /speckit.specify    →  spec.md
     b. /speckit.clarify    →  (optional) refine spec
     c. /speckit.plan       →  plan.md
     d. /speckit.analyze    →  (optional) consistency check
     e. /speckit.tasks      →  tasks.md
     f. /speckit.implement  →  actual code
     g. /speckit.checklist  →  (optional) quality validation
     h. git commit + push
```

## Verification

- After each `/speckit.implement`: `npm run lint`, `npm run test`, `npm run build`
- Angular: verify spartan/ui components render, Fineract API calls work
- Next.js: verify `npm run dev` starts, pages render, auth works
- Push to respective GitHub repos after each feature
