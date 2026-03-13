# TECH-012: Spartan/UI Migration — Replacing Angular Material with shadcn for Angular

## Context

The M-SACCO Angular app currently uses **Angular Material 20.2.14** with 34 imported modules and **82 distinct Material component tags** across the codebase. We will replace Angular Material with **[spartan/ui](https://www.spartan.ng/)** — the Angular port of shadcn/ui — to achieve a modern, clean, fully-customizable design consistent with the M-SACCO wireframes.

### Why spartan/ui?

| Criteria         | Angular Material              | spartan/ui                                |
| ---------------- | ----------------------------- | ----------------------------------------- |
| Design aesthetic | Material Design (opinionated) | shadcn/Tailwind (neutral, clean)          |
| Customization    | Theme overrides via SCSS      | Direct Tailwind classes you own           |
| Bundle size      | Large (entire component set)  | Tree-shakeable, only what you use         |
| Accessibility    | Built-in                      | Built-in (brain layer handles ARIA)       |
| Architecture     | NgModule + Standalone         | Standalone-first, signals, zoneless-ready |
| Styling          | SCSS + CSS variables          | Tailwind CSS utility classes              |
| Dark mode        | Theme class toggle            | Tailwind `dark:` variants                 |

### Architecture: Brain + Helm

spartan/ui uses a two-layer architecture:

- **Brain** (`@spartan-ng/brain/*`): Unstyled, accessible primitives (installed via npm). Handles ARIA attributes, keyboard navigation, focus management.
- **Helm** (`@spartan-ng/ui-*-helm`): Styled components using Tailwind CSS. Copied into your project — you own and edit the styling directly.

This means complex accessibility logic is maintained by the library, while you fully control the visual design.

---

## Prerequisites

### 1. Tailwind CSS v4 Upgrade

spartan/ui recommends Tailwind CSS v4. The current app uses Tailwind v3.3.3.

```bash
# Upgrade Tailwind CSS
npm install tailwindcss@latest @tailwindcss/forms@latest
```

Update `tailwind.config.js` for v4 compatibility or migrate to the new CSS-based config.

### 2. Install spartan/ui CLI

```bash
npm install -D @spartan-ng/cli
```

### 3. Initialize spartan/ui

```bash
npx @spartan-ng/cli init
```

This will:

- Add `@spartan-ng/ui-core` (class merge utilities, `hlm` function)
- Set up the `@spartan-ng/ui-core/hlm-tailwind-preset` in Tailwind config
- Create the helm component directory structure

### 4. Install Required Components

```bash
# Install all needed components at once
npx @spartan-ng/cli add \
  button accordion alert alert-dialog autocomplete avatar badge \
  breadcrumb calendar card checkbox collapsible combobox command \
  context-menu data-table dialog dropdown-menu form-field icon \
  input label menubar pagination popover progress radio-group \
  scroll-area select separator sheet sidebar skeleton slider \
  sonner spinner switch table tabs textarea toggle tooltip
```

---

## Component Migration Map

### Angular Material → spartan/ui (1:1 Mapping)

| Angular Material                                                                                              | spartan/ui Equivalent                                                                  | Notes                                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `mat-button`, `mat-raised-button`, `mat-flat-button`, `mat-stroked-button`, `mat-icon-button`, `mat-mini-fab` | `hlm-button`                                                                           | Single component with variants: `default`, `destructive`, `outline`, `secondary`, `ghost`, `link`. Sizes: `default`, `sm`, `lg`, `icon` |
| `mat-card`, `mat-card-title`, `mat-card-content`, `mat-card-actions`                                          | `hlm-card`, `hlm-card-header`, `hlm-card-title`, `hlm-card-content`, `hlm-card-footer` | Direct mapping                                                                                                                          |
| `mat-form-field`, `mat-label`, `mat-error`, `mat-hint`                                                        | `hlm-form-field`, `hlm-label`, `hlm-error`                                             | Uses `brn-form-field` brain                                                                                                             |
| `mat-input`                                                                                                   | `hlm-input`                                                                            | Directive-based, applies to `<input>`                                                                                                   |
| `mat-select`, `mat-option`                                                                                    | `brn-select` + `hlm-select`                                                            | Brain handles accessibility, helm handles styling                                                                                       |
| `mat-checkbox`                                                                                                | `hlm-checkbox`                                                                         | Uses `brn-checkbox` brain                                                                                                               |
| `mat-radio-group`, `mat-radio-button`                                                                         | `hlm-radio-group`                                                                      | Uses `brn-radio-group` brain                                                                                                            |
| `mat-datepicker`, `mat-datepicker-toggle`                                                                     | `hlm-calendar` + `hlm-popover`                                                         | Combine calendar with popover for datepicker behavior                                                                                   |
| `mat-slide-toggle`                                                                                            | `hlm-switch`                                                                           | Direct equivalent                                                                                                                       |
| `mat-slider`                                                                                                  | `hlm-slider`                                                                           | Direct equivalent                                                                                                                       |
| `mat-table`, `mat-header-row`, `mat-row`, `mat-cell`                                                          | `hlm-table`                                                                            | TanStack Table integration available via `hlm-data-table`                                                                               |
| `mat-paginator`                                                                                               | `hlm-pagination`                                                                       | Uses `brn-pagination` brain                                                                                                             |
| `mat-sort`, `mat-sort-header`                                                                                 | Built into `hlm-data-table`                                                            | TanStack Table handles sorting                                                                                                          |
| `mat-dialog`, `mat-dialog-title`, `mat-dialog-content`, `mat-dialog-actions`                                  | `hlm-dialog`, `hlm-dialog-header`, `hlm-dialog-content`, `hlm-dialog-footer`           | Uses `brn-dialog` brain                                                                                                                 |
| `mat-menu`, `mat-menu-item`                                                                                   | `hlm-dropdown-menu`                                                                    | Uses `brn-menu` brain                                                                                                                   |
| `mat-tab-group`, `mat-tab`                                                                                    | `hlm-tabs`, `hlm-tabs-list`, `hlm-tabs-trigger`, `hlm-tabs-content`                    | Uses `brn-tabs` brain                                                                                                                   |
| `mat-stepper`, `mat-step`                                                                                     | Custom `hlm-stepper` (see below)                                                       | No direct equivalent — build custom                                                                                                     |
| `mat-toolbar`                                                                                                 | Custom with Tailwind                                                                   | Simple div with flex + Tailwind classes                                                                                                 |
| `mat-sidenav`, `mat-sidenav-container`                                                                        | `hlm-sheet` or `hlm-sidebar`                                                           | Sheet for mobile overlay, sidebar for permanent                                                                                         |
| `mat-expansion-panel`                                                                                         | `hlm-accordion`                                                                        | Uses `brn-accordion` brain                                                                                                              |
| `mat-tooltip`                                                                                                 | `hlm-tooltip`                                                                          | Uses `brn-tooltip` brain                                                                                                                |
| `mat-progress-bar`                                                                                            | `hlm-progress`                                                                         | Direct equivalent                                                                                                                       |
| `mat-spinner`, `mat-progress-spinner`                                                                         | `hlm-spinner`                                                                          | Direct equivalent                                                                                                                       |
| `mat-snack-bar`                                                                                               | `hlm-sonner` (toast)                                                                   | Uses sonner for toast notifications                                                                                                     |
| `mat-chips`                                                                                                   | `hlm-badge` + `hlm-toggle-group`                                                       | Badge for display, toggle-group for selection                                                                                           |
| `mat-tree`, `mat-nested-tree-node`                                                                            | Custom with `hlm-collapsible`                                                          | Build tree from collapsible sections                                                                                                    |
| `mat-list`, `mat-list-item`, `mat-nav-list`                                                                   | Custom with Tailwind                                                                   | Simple list styling                                                                                                                     |
| `mat-grid-list`, `mat-grid-tile`                                                                              | Tailwind grid utilities                                                                | `grid grid-cols-{n} gap-{n}`                                                                                                            |
| `mat-divider`                                                                                                 | `hlm-separator`                                                                        | Direct equivalent                                                                                                                       |
| `mat-icon`                                                                                                    | `hlm-icon` + lucide-angular                                                            | Replace FontAwesome with lucide for consistency, or keep FA                                                                             |
| `mat-badge`                                                                                                   | `hlm-badge`                                                                            | Direct equivalent                                                                                                                       |
| `mat-button-toggle-group`                                                                                     | `hlm-toggle-group`                                                                     | Direct equivalent                                                                                                                       |
| `mat-autocomplete`                                                                                            | `hlm-combobox` or `hlm-autocomplete`                                                   | Uses `brn-combobox` brain                                                                                                               |
| `ngx-mat-select-search`                                                                                       | Built into `hlm-combobox`                                                              | Combobox has built-in search                                                                                                            |

### Components Requiring Custom Build

| Component                   | Strategy                                                                                                 |
| --------------------------- | -------------------------------------------------------------------------------------------------------- |
| `mat-stepper`               | Build `MsaccoStepperComponent` using `hlm-tabs` brain with numbered step indicators + content projection |
| `mat-tree`                  | Build `MsaccoTreeComponent` using nested `hlm-collapsible` + recursive template                          |
| `mat-grid-list`             | Replace with Tailwind CSS grid (`grid grid-cols-{n}`)                                                    |
| `mat-toolbar`               | Replace with semantic HTML + Tailwind (`<header class="flex items-center h-16 px-6 border-b">`)          |
| Datepicker (mat-datepicker) | Compose `hlm-calendar` + `hlm-popover` + `hlm-input` into `MsaccoDatepickerComponent`                    |

---

## Migration Strategy: 5 Phases

### Phase 1: Foundation Setup (Week 1)

**Goal:** Install spartan/ui, set up theme tokens, create the new design system foundation alongside existing Material.

**Steps:**

1. Upgrade Tailwind CSS to v4
2. Install spartan/ui CLI and core
3. Add all needed spartan/ui components via CLI
4. Configure Tailwind theme with M-SACCO design tokens:

```css
/* src/styles/spartan-theme.css */
:root {
  --background: 210 17% 98%; /* #f8f9fa */
  --foreground: 210 11% 15%; /* #212529 */
  --card: 0 0% 100%; /* #ffffff */
  --card-foreground: 210 11% 15%; /* #212529 */
  --popover: 0 0% 100%;
  --popover-foreground: 210 11% 15%;
  --primary: 205 84% 39%; /* #1074b9 */
  --primary-foreground: 0 0% 100%; /* white */
  --secondary: 82 50% 65%; /* #b4d575 */
  --secondary-foreground: 82 50% 20%;
  --muted: 210 16% 93%; /* #e9ecef */
  --muted-foreground: 215 14% 46%; /* #6c757d */
  --accent: 82 50% 65%; /* #b4d575 */
  --accent-foreground: 82 50% 20%;
  --destructive: 354 70% 54%; /* #dc3545 */
  --destructive-foreground: 0 0% 100%;
  --border: 220 13% 91%; /* #e9ecef */
  --input: 220 13% 91%;
  --ring: 205 84% 39%; /* #1074b9 */
  --radius: 0.5rem;

  /* M-SACCO Status Colors */
  --status-active: 134 61% 41%; /* #28a745 */
  --status-pending: 45 100% 51%; /* #ffc107 */
  --status-rejected: 354 70% 54%; /* #dc3545 */
  --status-closed: 208 7% 46%; /* #6c757d */
  --status-overdue: 354 70% 54%; /* #dc3545 */
  --status-disbursed: 205 84% 39%; /* #1074b9 */
}

.dark {
  --background: 222 47% 11%;
  --foreground: 210 40% 98%;
  --card: 222 47% 15%;
  --card-foreground: 210 40% 98%;
  --primary: 205 84% 55%;
  --primary-foreground: 0 0% 100%;
  --secondary: 82 50% 45%;
  --muted: 217 33% 17%;
  --muted-foreground: 215 20% 65%;
  --accent: 217 33% 17%;
  --border: 217 33% 17%;
  --input: 217 33% 17%;
  --ring: 205 84% 55%;
}
```

5. Create a `spartan-shared.module.ts` parallel to existing `standalone-shared.module.ts`
6. Verify both Material and spartan components can coexist during migration

**Key file:** `src/app/shared/spartan-shared.module.ts`

```typescript
// Shared imports for components migrated to spartan/ui
export const SPARTAN_SHARED_IMPORTS = [
  // Brain components (npm packages)
  BrnDialogModule,
  BrnMenuModule,
  BrnSelectModule,
  BrnTabsModule,
  BrnTooltipModule,
  BrnAccordionModule,
  BrnCheckboxModule,
  BrnRadioModule,
  BrnPopoverModule,
  BrnSheetModule,
  BrnAlertDialogModule,
  BrnCommandModule,
  BrnPaginationModule,

  // Helm components (local, editable)
  HlmButtonDirective,
  HlmCardDirective,
  HlmInputDirective,
  HlmLabelDirective,
  HlmBadgeDirective,
  HlmSeparatorDirective,
  HlmScrollAreaDirective,
  HlmIconComponent,
  HlmSpinnerComponent,
  HlmProgressDirective,
  HlmSwitchComponent,
  HlmSkeletonComponent,
  HlmToastComponent
  // ... add more as migration progresses
];
```

### Phase 2: Form Components (Weeks 2-3)

**Goal:** Migrate all form-related components — these are the most frequently used across all modules.

**Components to migrate:**

- `mat-form-field` + `mat-input` → `hlm-input` + `hlm-label` + `hlm-form-field`
- `mat-select` + `mat-option` → `brn-select` + `hlm-select`
- `mat-checkbox` → `hlm-checkbox`
- `mat-radio-group` → `hlm-radio-group`
- `mat-datepicker` → Custom `MsaccoDatepickerComponent` (hlm-calendar + hlm-popover)
- `mat-slide-toggle` → `hlm-switch`
- `mat-autocomplete` → `hlm-combobox`
- `ngx-mat-select-search` → Built into `hlm-combobox`

**Migration pattern for form fields:**

```html
<!-- BEFORE: Angular Material -->
<mat-form-field class="flex-fill">
  <mat-label>Product Name</mat-label>
  <mat-select required formControlName="productId">
    <mat-option *ngFor="let product of products" [value]="product.id"> {{ product.name }} </mat-option>
  </mat-select>
  <mat-error *ngIf="form.get('productId')?.hasError('required')"> Product Name is required </mat-error>
</mat-form-field>

<!-- AFTER: spartan/ui -->
<hlm-form-field class="flex-1">
  <label hlmLabel>Product Name</label>
  <brn-select formControlName="productId" required>
    <hlm-select-trigger>
      <hlm-select-value placeholder="Select product" />
    </hlm-select-trigger>
    <hlm-select-content>
      @for (product of products; track product.id) {
      <hlm-select-option [value]="product.id">{{ product.name }}</hlm-select-option>
      }
    </hlm-select-content>
  </brn-select>
  <hlm-error>Product Name is required</hlm-error>
</hlm-form-field>
```

**Files affected:** Every component with `mat-form-field` — estimated 200+ template files across all modules.

**Strategy:** Create a shared `MsaccoFormFieldComponent` that wraps spartan/ui form field to minimize per-file changes.

### Phase 3: Data Table & Navigation (Weeks 4-5)

**Goal:** Migrate tables, pagination, sorting, and navigation components.

**Components to migrate:**

- `mat-table` → `hlm-table` + TanStack Table integration
- `mat-paginator` → `hlm-pagination`
- `mat-sort` → TanStack Table sorting
- `mat-menu` → `hlm-dropdown-menu`
- `mat-tab-group` → `hlm-tabs`
- `mat-toolbar` → Semantic HTML + Tailwind
- `mat-sidenav` → `hlm-sidebar` / `hlm-sheet`
- `mat-expansion-panel` → `hlm-accordion`

**Data table migration pattern:**

```html
<!-- BEFORE: Angular Material Table -->
<table mat-table [dataSource]="dataSource" matSort>
  <ng-container matColumnDef="name">
    <th mat-header-cell *matHeaderCellDef mat-sort-header>Name</th>
    <td mat-cell *matCellDef="let row">{{ row.name }}</td>
  </ng-container>
  <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
  <tr mat-row *matRowDef="let row; columns: displayedColumns" (click)="onRowClick(row)"></tr>
</table>
<mat-paginator [pageSize]="25" [pageSizeOptions]="[10, 25, 50, 100]"></mat-paginator>

<!-- AFTER: spartan/ui Data Table -->
<hlm-table>
  <hlm-table-header>
    <hlm-table-row>
      @for (column of columns; track column.id) {
      <hlm-table-head [class]="column.class" [brnSortHeader]="column.id" (sortChange)="onSort($event)"> {{ column.label }} </hlm-table-head>
      }
    </hlm-table-row>
  </hlm-table-header>
  <hlm-table-body>
    @for (row of dataSource.data; track row.id) {
    <hlm-table-row class="cursor-pointer hover:bg-muted/50" (click)="onRowClick(row)">
      @for (column of columns; track column.id) {
      <hlm-table-cell>{{ row[column.id] }}</hlm-table-cell>
      }
    </hlm-table-row>
    }
  </hlm-table-body>
</hlm-table>
<brn-pagination [totalItems]="dataSource.data.length" [pageSize]="25" [pageSizeOptions]="[10, 25, 50, 100]" (pageChange)="onPageChange($event)" />
```

### Phase 4: Dialogs, Overlays & Feedback (Weeks 6-7)

**Components to migrate:**

- `mat-dialog` → `hlm-dialog`
- `mat-snack-bar` → `hlm-sonner` (toast notifications)
- `mat-tooltip` → `hlm-tooltip`
- `mat-progress-bar` → `hlm-progress`
- `mat-spinner` → `hlm-spinner`
- `mat-badge` → `hlm-badge`

**Dialog migration pattern:**

```html
<!-- BEFORE: Angular Material Dialog -->
<h1 mat-dialog-title>Delete Client</h1>
<div mat-dialog-content>
  <p>Are you sure you want to delete this client?</p>
</div>
<mat-dialog-actions align="end">
  <button mat-button mat-dialog-close>Cancel</button>
  <button mat-raised-button color="warn" [mat-dialog-close]="true">Delete</button>
</mat-dialog-actions>

<!-- AFTER: spartan/ui Dialog -->
<hlm-dialog-header>
  <h3 hlmDialogTitle>Delete Client</h3>
</hlm-dialog-header>
<hlm-dialog-content>
  <p class="text-sm text-muted-foreground">Are you sure you want to delete this client?</p>
</hlm-dialog-content>
<hlm-dialog-footer>
  <button hlmBtn variant="outline" (click)="dialogRef.close()">Cancel</button>
  <button hlmBtn variant="destructive" (click)="dialogRef.close(true)">Delete</button>
</hlm-dialog-footer>
```

### Phase 5: Layout, Stepper & Cleanup (Weeks 8-9)

**Components to build custom:**

- Custom `MsaccoStepperComponent` using spartan primitives
- Custom `MsaccoTreeComponent` for chart of accounts
- Custom `MsaccoDatepickerComponent` (calendar + popover)
- Remove all Angular Material imports
- Remove `@angular/material` and related packages from `package.json`
- Remove `material.module.ts`
- Remove Material theme files (`mifosx-theme.scss`, `_material-palette.scss`)
- Update `angular.json` to remove Material stylesheet references

**Custom Stepper Component:**

```typescript
@Component({
  selector: 'msacco-stepper',
  standalone: true,
  imports: [
    HlmButtonDirective,
    HlmSeparatorDirective,
    NgClass
  ],
  template: `
    <!-- Step indicators -->
    <div class="flex items-center justify-center gap-2 mb-8">
      @for (step of steps; track step.label; let i = $index) {
        <div class="flex items-center gap-2">
          <button (click)="goToStep(i)" [class]="stepClass(i)" class="w-10 h-10 rounded-full flex items-center justify-center text-sm font-medium transition-colors">
            @if (i < currentStep) {
              <lucide-icon name="check" class="w-5 h-5" />
            } @else {
              {{ i + 1 }}
            }
          </button>
          <span class="text-sm font-medium hidden sm:inline" [class.text-primary]="i === currentStep" [class.text-muted-foreground]="i !== currentStep">
            {{ step.label }}
          </span>
          @if (i < steps.length - 1) {
            <div hlmSeparator class="w-12 mx-2" />
          }
        </div>
      }
    </div>

    <!-- Step content -->
    <div class="min-h-[400px]">
      <ng-content />
    </div>

    <!-- Navigation buttons -->
    <div class="flex justify-between mt-6 pt-6 border-t">
      <button hlmBtn variant="outline" (click)="previous()" [disabled]="currentStep === 0">Previous</button>
      @if (currentStep < steps.length - 1) {
        <button hlmBtn (click)="next()" [disabled]="!canProceed">Next</button>
      } @else {
        <button hlmBtn (click)="submit.emit()">Submit</button>
      }
    </div>
  `
})
export class MsaccoStepperComponent {
  steps = input.required<StepDef[]>();
  currentStep = signal(0);
  canProceed = input(true);
  submit = output<void>();

  stepClass(index: number): string {
    if (index < this.currentStep()) return 'bg-primary text-primary-foreground';
    if (index === this.currentStep()) return 'bg-primary text-primary-foreground ring-2 ring-primary ring-offset-2';
    return 'bg-muted text-muted-foreground';
  }

  next() {
    if (this.canProceed()) this.currentStep.update((s) => Math.min(s + 1, this.steps().length - 1));
  }
  previous() {
    this.currentStep.update((s) => Math.max(s - 1, 0));
  }
  goToStep(i: number) {
    if (i <= this.currentStep()) this.currentStep.set(i);
  }
}
```

---

## Icon Strategy

### Option A: Keep FontAwesome (Recommended for initial migration)

- Less disruption, icons already registered in `icons.module.ts`
- FontAwesome works fine alongside spartan/ui

### Option B: Migrate to lucide-angular (Full shadcn consistency)

- `npm install lucide-angular`
- Replace `<fa-icon>` with `<lucide-icon>`
- 82 FontAwesome icons need mapping

**Recommended:** Keep FontAwesome during spartan/ui migration. Migrate to lucide later as a separate effort.

---

## Coexistence Strategy

During migration, Angular Material and spartan/ui will coexist. This is the recommended approach:

1. **Keep both in `package.json`** until migration is complete
2. **Migrate module by module** — each module fully migrates before moving to the next
3. **Use feature flags** — `environment.useSpartanUI` to toggle new UI per module during testing
4. **Shared imports:** Create `SPARTAN_SHARED_IMPORTS` array for migrated components, keep `STANDALONE_SHARED_IMPORTS` for Material ones
5. **Remove Material last** — only after ALL modules are migrated

### Migration order (by module):

1. **Shared components** — Dialogs, buttons, badges (used everywhere)
2. **Login** — Small, self-contained, good test case
3. **Home/Dashboard** — Cards, charts, simple layout
4. **Clients** — Most complex module, serves as reference
5. **Groups** — Similar patterns to Clients
6. **Loans** — Complex forms and steppers
7. **Accounting** — Tables and trees
8. **Products** — Complex steppers
9. **Organization** — Mixed components
10. **System** — Admin panels
11. **Remaining modules** — One at a time

---

## Package Changes

### Install

```bash
npm install @spartan-ng/ui-core
npm install @spartan-ng/brain
npm install lucide-angular  # if migrating icons
```

### Remove (after full migration)

```bash
npm uninstall @angular/material @angular/cdk @material/web ngx-mat-select-search
```

### Files to Delete (after full migration)

- `src/app/shared/material.module.ts`
- `src/theme/mifosx-theme.scss`
- `src/theme/_material-palette.scss`
- `src/theme/_material-web.scss`
- `src/theme/_content.scss`
- `src/theme/_dark_content.scss`
- `src/app/shared/m3-ui/` (entire directory — replaced by spartan/ui buttons)

### Files to Create

- `src/app/shared/spartan-shared.module.ts` — Shared spartan imports
- `src/styles/spartan-theme.css` — CSS variables for spartan/ui theme
- `src/app/shared/msacco-stepper/` — Custom stepper component
- `src/app/shared/msacco-datepicker/` — Custom datepicker (calendar + popover)
- `src/app/shared/msacco-tree/` — Custom tree component

---

## Testing Checklist

- [ ] All form fields render and validate correctly
- [ ] All data tables sort, paginate, and export
- [ ] All dialogs open, close, and return data
- [ ] All steppers navigate between steps with validation
- [ ] Toast notifications appear for success/error
- [ ] Dark mode toggles correctly with spartan/ui components
- [ ] Responsive layouts work at all breakpoints
- [ ] Keyboard navigation works (Tab, Enter, Escape, Arrow keys)
- [ ] Screen reader announces component states correctly
- [ ] No Angular Material imports remain in any module
- [ ] Bundle size is smaller than or equal to the Material version

---

## Estimated Timeline

| Phase                       | Duration     | Scope                                           |
| --------------------------- | ------------ | ----------------------------------------------- |
| Phase 1: Foundation         | 1 week       | Install, theme tokens, coexistence setup        |
| Phase 2: Forms              | 2 weeks      | 200+ template files with form fields            |
| Phase 3: Tables & Nav       | 2 weeks      | Data tables, tabs, menus, toolbar               |
| Phase 4: Dialogs & Feedback | 2 weeks      | 20+ dialog components, toasts, tooltips         |
| Phase 5: Layout & Cleanup   | 2 weeks      | Custom stepper/tree/datepicker, remove Material |
| **Total**                   | **~9 weeks** | Full Angular Material → spartan/ui migration    |

This can run in parallel with new feature development (TECH-001 through TECH-011) since new features should be built with spartan/ui from the start.
