# TECH-002: Shared Components Specification

**Status:** Draft
**Priority:** High
**Estimated Effort:** 5-7 days
**Dependencies:** TECH-001 (branding/color variables), TECH-012 (spartan/ui migration)
**UI Library:** spartan/ui (Angular port of shadcn/ui) — see TECH-012 for setup

---

## Overview

This specification defines 5 new shared components that implement M-SACCO's UI patterns. These components are reusable building blocks consumed by feature modules (Clients, Groups, Loans, Savings, etc.).

All components will be standalone Angular components built with **spartan/ui** (brain + helm architecture) and Tailwind CSS. They will use `SPARTAN_SHARED_IMPORTS` (defined in TECH-012) and be placed under `src/app/shared/`.

**Key spartan/ui components leveraged:**

- `hlm-table` / `hlm-data-table` — For MsaccoDataTableComponent
- `hlm-tabs` / `brn-tabs` — For StatusTabsComponent
- `hlm-button` — For all button variants (replaces mat-button, mat-raised-button, etc.)
- `hlm-badge` — For StatusBadgeComponent
- Custom composition — For MsaccoWizardComponent and ApprovalActionBarComponent

---

## 1. MsaccoDataTableComponent

**Location:** `src/app/shared/msacco-data-table/`

### Purpose

A configurable data table with status tab filtering, search, pagination display ("Showing X to Y of Z"), and export buttons (Copy, Excel, PDF). Replaces ad-hoc table implementations across the application.

### TypeScript Interfaces

```typescript
// src/app/shared/msacco-data-table/msacco-data-table.interfaces.ts

export type ColumnType = 'text' | 'date' | 'currency' | 'status' | 'action';

export interface ColumnDef {
  /** Property name on the data object */
  name: string;
  /** Display header text (supports i18n keys) */
  header: string;
  /** Template function: receives row data, returns display string */
  cell: (row: any) => string;
  /** Whether this column is sortable. Default: true */
  sortable?: boolean;
  /** Column display type. Default: 'text' */
  type?: ColumnType;
  /** Column width (CSS value, e.g., '120px', '15%'). Optional. */
  width?: string;
  /** For type='action': action template reference. */
  actionTemplate?: TemplateRef<any>;
}

export interface StatusTab {
  /** Display label (supports i18n keys) */
  label: string;
  /** Filter value to match against the statusField */
  value: string;
  /** Count of records matching this status */
  count: number;
}

export interface ExportConfig {
  /** Filename prefix for exports (without extension) */
  filenamePrefix: string;
  /** Title shown in PDF header */
  title?: string;
  /** Columns to include in export (defaults to all non-action columns) */
  exportColumns?: string[];
}
```

### Component API

```typescript
// src/app/shared/msacco-data-table/msacco-data-table.component.ts

import { Component, Input, Output, EventEmitter, ViewChild, OnInit, OnChanges, SimpleChanges, TemplateRef, ContentChild, inject } from '@angular/core';
import { MatTableDataSource } from '@angular/material/table';
import { MatPaginator } from '@angular/material/paginator';
import { MatSort } from '@angular/material/sort';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { ColumnDef, StatusTab, ExportConfig } from './msacco-data-table.interfaces';
import { MsaccoDataTableExportService } from './msacco-data-table-export.service';
import { StatusTabsComponent } from '../status-tabs/status-tabs.component';
import { StatusBadgeComponent } from '../status-badge/status-badge.component';

@Component({
  selector: 'msacco-data-table',
  templateUrl: './msacco-data-table.component.html',
  styleUrls: ['./msacco-data-table.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatPaginator,
    MatSort,
    StatusTabsComponent,
    StatusBadgeComponent
  ]
})
export class MsaccoDataTableComponent implements OnInit, OnChanges {
  private exportService = inject(MsaccoDataTableExportService);

  /** Column definitions */
  @Input() columns: ColumnDef[] = [];

  /** Data source (MatTableDataSource or raw array) */
  @Input() dataSource: MatTableDataSource<any> | any[];

  /** Field name on the data object used for status tab filtering */
  @Input() statusField: string = '';

  /** Status tabs to display above the table */
  @Input() statusTabs: StatusTab[] = [];

  /** Whether to show the export buttons (Copy, Excel, PDF) */
  @Input() exportEnabled: boolean = false;

  /** Export configuration */
  @Input() exportConfig: ExportConfig = { filenamePrefix: 'export' };

  /** Whether to show the search/filter input above the table */
  @Input() searchEnabled: boolean = true;

  /** Page size options for the paginator */
  @Input() pageSizeOptions: number[] = [
    10,
    25,
    50,
    100
  ];

  /** Default page size */
  @Input() pageSize: number = 10;

  /** Whether to show the "Showing X to Y of Z" text */
  @Input() showPaginationSummary: boolean = true;

  /** Template for custom action column cells */
  @ContentChild('actionTemplate') actionTemplate: TemplateRef<any>;

  /** Emitted when a row is clicked */
  @Output() rowClick = new EventEmitter<any>();

  /** Emitted when status tab changes */
  @Output() statusTabChange = new EventEmitter<string>();

  @ViewChild(MatPaginator) paginator: MatPaginator;
  @ViewChild(MatSort) sort: MatSort;

  internalDataSource: MatTableDataSource<any>;
  displayedColumns: string[] = [];
  activeStatusTab: string = '';
  searchValue: string = '';

  ngOnInit(): void {
    this.initDataSource();
    this.displayedColumns = this.columns.map((c) => c.name);
  }

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['dataSource'] || changes['columns']) {
      this.initDataSource();
      this.displayedColumns = this.columns.map((c) => c.name);
    }
  }

  private initDataSource(): void {
    if (this.dataSource instanceof MatTableDataSource) {
      this.internalDataSource = this.dataSource;
    } else if (Array.isArray(this.dataSource)) {
      this.internalDataSource = new MatTableDataSource(this.dataSource);
    }
    // Defer paginator/sort binding to after view init
    setTimeout(() => {
      if (this.internalDataSource) {
        this.internalDataSource.paginator = this.paginator;
        this.internalDataSource.sort = this.sort;
      }
    });
  }

  /** Apply text filter from search input */
  applyFilter(filterValue: string): void {
    this.searchValue = filterValue;
    this.internalDataSource.filter = filterValue.trim().toLowerCase();
    if (this.internalDataSource.paginator) {
      this.internalDataSource.paginator.firstPage();
    }
  }

  /** Handle status tab selection */
  onStatusTabChange(tabValue: string): void {
    this.activeStatusTab = tabValue;
    if (tabValue === '' || tabValue === 'all') {
      this.internalDataSource.filterPredicate = () => true;
      this.internalDataSource.filter = this.searchValue;
    } else {
      this.internalDataSource.filterPredicate = (data: any) => {
        return data[this.statusField]?.toString().toLowerCase() === tabValue.toLowerCase();
      };
      this.internalDataSource.filter = tabValue;
    }
    this.statusTabChange.emit(tabValue);
  }

  /** Pagination summary text */
  get paginationSummary(): string {
    if (!this.internalDataSource || !this.paginator) return '';
    const total = this.internalDataSource.filteredData.length;
    const start = this.paginator.pageIndex * this.paginator.pageSize + 1;
    const end = Math.min(start + this.paginator.pageSize - 1, total);
    return `Showing ${start} to ${end} of ${total}`;
  }

  /** Export: Copy to clipboard */
  exportCopy(): void {
    this.exportService.copyToClipboard(this.internalDataSource.filteredData, this.getExportColumns());
  }

  /** Export: Excel */
  exportExcel(): void {
    this.exportService.exportToExcel(this.internalDataSource.filteredData, this.getExportColumns(), this.exportConfig.filenamePrefix);
  }

  /** Export: PDF */
  exportPdf(): void {
    this.exportService.exportToPdf(this.internalDataSource.filteredData, this.getExportColumns(), this.exportConfig.filenamePrefix, this.exportConfig.title);
  }

  private getExportColumns(): ColumnDef[] {
    const exportNames = this.exportConfig.exportColumns;
    const nonActionCols = this.columns.filter((c) => c.type !== 'action');
    if (exportNames?.length) {
      return nonActionCols.filter((c) => exportNames.includes(c.name));
    }
    return nonActionCols;
  }

  onRowClick(row: any): void {
    this.rowClick.emit(row);
  }
}
```

### Export Service

```typescript
// src/app/shared/msacco-data-table/msacco-data-table-export.service.ts

import { Injectable } from '@angular/core';
import { ColumnDef } from './msacco-data-table.interfaces';

@Injectable({ providedIn: 'root' })
export class MsaccoDataTableExportService {
  /** Copy table data to clipboard as tab-separated text */
  copyToClipboard(data: any[], columns: ColumnDef[]): void {
    const headers = columns.map((c) => c.header).join('\t');
    const rows = data.map((row) => columns.map((c) => c.cell(row) ?? '').join('\t')).join('\n');
    const text = `${headers}\n${rows}`;
    navigator.clipboard.writeText(text);
  }

  /** Export to Excel using ExcelJS */
  async exportToExcel(data: any[], columns: ColumnDef[], filename: string): Promise<void> {
    const ExcelJS = await import('exceljs');
    const workbook = new ExcelJS.Workbook();
    const worksheet = workbook.addWorksheet('Data');

    // Header row
    worksheet.addRow(columns.map((c) => c.header));

    // Style header row
    const headerRow = worksheet.getRow(1);
    headerRow.font = { bold: true };
    headerRow.fill = {
      type: 'pattern',
      pattern: 'solid',
      fgColor: { argb: 'FF1074B9' }
    };
    headerRow.font = { bold: true, color: { argb: 'FFFFFFFF' } };

    // Data rows
    data.forEach((row) => {
      worksheet.addRow(columns.map((c) => c.cell(row) ?? ''));
    });

    // Auto-fit column widths
    columns.forEach((col, i) => {
      const maxLength = Math.max(col.header.length, ...data.map((row) => (c.cell(row) ?? '').toString().length));
      worksheet.getColumn(i + 1).width = Math.min(maxLength + 4, 40);
    });

    // Generate and download
    const buffer = await workbook.xlsx.writeBuffer();
    const blob = new Blob([buffer], {
      type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
    });
    this.downloadBlob(blob, `${filename}.xlsx`);
  }

  /** Export to PDF using jsPDF + autoTable */
  async exportToPdf(data: any[], columns: ColumnDef[], filename: string, title?: string): Promise<void> {
    const { default: jsPDF } = await import('jspdf');
    await import('jspdf-autotable');

    const doc = new jsPDF('l', 'mm', 'a4');

    if (title) {
      doc.setFontSize(16);
      doc.text(title, 14, 15);
    }

    const headers = columns.map((c) => c.header);
    const body = data.map((row) => columns.map((c) => c.cell(row) ?? ''));

    (doc as any).autoTable({
      head: [headers],
      body: body,
      startY: title ? 22 : 10,
      styles: { fontSize: 9 },
      headStyles: { fillColor: [
          16,
          116,
          185
        ] }, // #1074b9
      alternateRowStyles: { fillColor: [
          245,
          245,
          245
        ] }
    });

    doc.save(`${filename}.pdf`);
  }

  private downloadBlob(blob: Blob, filename: string): void {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = filename;
    a.click();
    URL.revokeObjectURL(url);
  }
}
```

### Template

```html
<!-- src/app/shared/msacco-data-table/msacco-data-table.component.html -->

<div class="msacco-table-container">
  <!-- Status Tabs -->
  @if (statusTabs.length > 0) {
  <msacco-status-tabs [tabs]="statusTabs" [activeTab]="activeStatusTab" (tabChange)="onStatusTabChange($event)"></msacco-status-tabs>
  }

  <!-- Toolbar: Search + Export -->
  <div class="msacco-table-toolbar">
    @if (searchEnabled) {
    <mat-form-field appearance="outline" class="msacco-table-search">
      <mat-icon matPrefix>search</mat-icon>
      <input matInput [placeholder]="'labels.inputs.Filter' | translate" (keyup)="applyFilter($event.target.value)" />
    </mat-form-field>
    }
    <span class="toolbar-spacer"></span>
    @if (exportEnabled) {
    <div class="msacco-export-buttons">
      <button mat-stroked-button (click)="exportCopy()" class="export-btn"><mat-icon>content_copy</mat-icon> Copy</button>
      <button mat-stroked-button (click)="exportExcel()" class="export-btn"><mat-icon>table_chart</mat-icon> Excel</button>
      <button mat-stroked-button (click)="exportPdf()" class="export-btn"><mat-icon>picture_as_pdf</mat-icon> PDF</button>
    </div>
    }
  </div>

  <!-- Data Table -->
  <div class="msacco-table-wrapper">
    <table mat-table [dataSource]="internalDataSource" matSort class="msacco-table">
      @for (col of columns; track col.name) {
      <ng-container [matColumnDef]="col.name">
        <th mat-header-cell *matHeaderCellDef [mat-sort-header]="col.sortable !== false ? col.name : null">{{ col.header | translate }}</th>
        <td mat-cell *matCellDef="let row">
          @switch (col.type) { @case ('status') {
          <msacco-status-badge [status]="col.cell(row)"></msacco-status-badge>
          } @case ('currency') {
          <span class="currency-cell">{{ col.cell(row) | number:'1.2-2' }}</span>
          } @case ('date') {
          <span>{{ col.cell(row) }}</span>
          } @case ('action') {
          <ng-container [ngTemplateOutlet]="actionTemplate" [ngTemplateOutletContext]="{ $implicit: row }"></ng-container>
          } @default {
          <span>{{ col.cell(row) }}</span>
          } }
        </td>
      </ng-container>
      }

      <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
      <tr mat-row *matRowDef="let row; columns: displayedColumns" (click)="onRowClick(row)" class="msacco-table-row"></tr>
    </table>
  </div>

  <!-- Pagination -->
  <div class="msacco-table-footer">
    @if (showPaginationSummary) {
    <span class="pagination-summary">{{ paginationSummary }}</span>
    }
    <mat-paginator [pageSizeOptions]="pageSizeOptions" [pageSize]="pageSize" showFirstLastButtons></mat-paginator>
  </div>
</div>
```

### Styles

```scss
// src/app/shared/msacco-data-table/msacco-data-table.component.scss

.msacco-table-container {
  background: #ffffff;
  border-radius: 4px;
  border: 1px solid #e0e0e0;
  overflow: hidden;
}

.msacco-table-toolbar {
  display: flex;
  align-items: center;
  padding: 8px 16px;
  gap: 8px;
  border-bottom: 1px solid #eee;
}

.msacco-table-search {
  max-width: 300px;
  font-size: 13px;

  ::ng-deep .mat-mdc-form-field-subscript-wrapper {
    display: none;
  }
}

.toolbar-spacer {
  flex: 1 1 auto;
}

.msacco-export-buttons {
  display: flex;
  gap: 4px;

  .export-btn {
    font-size: 12px;
    padding: 0 8px;
    height: 32px;
    line-height: 32px;

    mat-icon {
      font-size: 16px;
      width: 16px;
      height: 16px;
      margin-right: 4px;
    }
  }
}

.msacco-table-wrapper {
  overflow-x: auto;
}

.msacco-table {
  width: 100%;

  th.mat-mdc-header-cell {
    font-weight: 600;
    font-size: 13px;
    color: #333;
    background-color: #f8f9fa;
  }

  td.mat-mdc-cell {
    font-size: 13px;
    color: #555;
  }

  .currency-cell {
    font-variant-numeric: tabular-nums;
    text-align: right;
  }
}

.msacco-table-row {
  cursor: pointer;

  &:hover {
    background-color: #f0f7ff;
  }
}

.msacco-table-footer {
  display: flex;
  align-items: center;
  padding: 0 16px;
  border-top: 1px solid #eee;

  .pagination-summary {
    font-size: 13px;
    color: #666;
    white-space: nowrap;
  }
}
```

### Usage Example: Client List

```typescript
// In clients.component.ts

clientColumns: ColumnDef[] = [
  { name: 'accountNo', header: 'Account No', cell: (row) => row.accountNo, sortable: true },
  { name: 'displayName', header: 'Client Name', cell: (row) => row.displayName, sortable: true },
  { name: 'officeName', header: 'Branch', cell: (row) => row.officeName, sortable: true },
  { name: 'status', header: 'Status', cell: (row) => row.status?.value, type: 'status', sortable: false },
  { name: 'activationDate', header: 'Activation Date', cell: (row) => row.activationDate, type: 'date' },
];

clientStatusTabs: StatusTab[] = [
  { label: 'All', value: '', count: 0 },
  { label: 'Active', value: 'Active', count: 0 },
  { label: 'Pending', value: 'Pending', count: 0 },
  { label: 'Closed', value: 'Closed', count: 0 },
];

clientExportConfig: ExportConfig = {
  filenamePrefix: 'clients-list',
  title: 'Client List Report'
};
```

```html
<!-- In clients.component.html -->
<msacco-data-table [columns]="clientColumns" [dataSource]="clientsDataSource" statusField="status.value" [statusTabs]="clientStatusTabs" [exportEnabled]="true" [exportConfig]="clientExportConfig" [searchEnabled]="true" (rowClick)="onClientClick($event)">
  <ng-template #actionTemplate let-row>
    <button mat-icon-button [routerLink]="['/clients', row.id, 'general']">
      <mat-icon>visibility</mat-icon>
    </button>
  </ng-template>
</msacco-data-table>
```

---

## 2. StatusTabsComponent

**Location:** `src/app/shared/status-tabs/`

### Purpose

A horizontal tab bar showing status categories with count badges. Selecting a tab emits the filter value.

### Component

```typescript
// src/app/shared/status-tabs/status-tabs.component.ts

import { Component, Input, Output, EventEmitter } from '@angular/core';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { StatusTab } from '../msacco-data-table/msacco-data-table.interfaces';

@Component({
  selector: 'msacco-status-tabs',
  templateUrl: './status-tabs.component.html',
  styleUrls: ['./status-tabs.component.scss'],
  standalone: true,
  imports: [...STANDALONE_SHARED_IMPORTS]
})
export class StatusTabsComponent {
  /** Array of status tabs to display */
  @Input() tabs: StatusTab[] = [];

  /** Currently active tab value */
  @Input() activeTab: string = '';

  /** Emits the selected tab value when user clicks a tab */
  @Output() tabChange = new EventEmitter<string>();

  selectTab(tabValue: string): void {
    this.tabChange.emit(tabValue);
  }

  isActive(tabValue: string): boolean {
    return this.activeTab === tabValue;
  }
}
```

### Template

```html
<!-- src/app/shared/status-tabs/status-tabs.component.html -->

<div class="msacco-status-tabs" role="tablist">
  @for (tab of tabs; track tab.value) {
  <button class="status-tab" [class.active]="isActive(tab.value)" (click)="selectTab(tab.value)" role="tab" [attr.aria-selected]="isActive(tab.value)">
    <span class="tab-label">{{ tab.label | translate }}</span>
    @if (tab.count !== undefined && tab.count !== null) {
    <span class="tab-count" [class.active]="isActive(tab.value)">{{ tab.count }}</span>
    }
  </button>
  }
</div>
```

### Styles

```scss
// src/app/shared/status-tabs/status-tabs.component.scss

$primary-blue: #1074b9;

.msacco-status-tabs {
  display: flex;
  align-items: center;
  gap: 0;
  padding: 0 16px;
  border-bottom: 2px solid #e0e0e0;
  background-color: #ffffff;
  overflow-x: auto;
}

.status-tab {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 10px 16px;
  font-size: 13px;
  font-weight: 500;
  color: #666;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  margin-bottom: -2px;
  cursor: pointer;
  white-space: nowrap;
  transition: all 0.2s ease;

  &:hover {
    color: $primary-blue;
    background-color: #f0f7ff;
  }

  &.active {
    color: $primary-blue;
    border-bottom-color: $primary-blue;
    font-weight: 600;
  }
}

.tab-count {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 20px;
  height: 20px;
  padding: 0 6px;
  border-radius: 10px;
  font-size: 11px;
  font-weight: 600;
  background-color: #e0e0e0;
  color: #666;

  &.active {
    background-color: $primary-blue;
    color: #ffffff;
  }
}
```

---

## 3. MsaccoWizardComponent

**Location:** `src/app/shared/msacco-wizard/`

### Purpose

Wraps Angular Material `mat-stepper` with M-SACCO visual styling: numbered circles connected by lines, current step highlighted in primary blue.

### TypeScript Interfaces

```typescript
// src/app/shared/msacco-wizard/msacco-wizard.interfaces.ts

import { TemplateRef } from '@angular/core';

export interface WizardStep {
  /** Step label text */
  label: string;
  /** Optional icon name (Material icon) */
  icon?: string;
  /** Template reference for the step content */
  contentTemplate?: TemplateRef<any>;
  /** Whether this step is optional */
  optional?: boolean;
  /** Whether the step is editable after completion */
  editable?: boolean;
}
```

### Component

```typescript
// src/app/shared/msacco-wizard/msacco-wizard.component.ts

import { Component, Input, Output, EventEmitter, ViewChild, ContentChildren, QueryList, AfterContentInit, TemplateRef } from '@angular/core';
import { MatStepper, MatStep, MatStepLabel, MatStepContent } from '@angular/material/stepper';
import { MatButton } from '@angular/material/button';
import { MatIcon } from '@angular/material/icon';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { STEPPER_GLOBAL_OPTIONS } from '@angular/cdk/stepper';
import { WizardStep } from './msacco-wizard.interfaces';
import { MsaccoWizardStepDirective } from './msacco-wizard-step.directive';

@Component({
  selector: 'msacco-wizard',
  templateUrl: './msacco-wizard.component.html',
  styleUrls: ['./msacco-wizard.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatStepper,
    MatStep,
    MatStepLabel,
    MatButton,
    MatIcon
  ],
  providers: [
    { provide: STEPPER_GLOBAL_OPTIONS, useValue: { displayDefaultIndicatorType: false } }]
})
export class MsaccoWizardComponent implements AfterContentInit {
  /** Step definitions (used when steps are defined via Input, not content projection) */
  @Input() steps: WizardStep[] = [];

  /** Whether the wizard enforces linear step progression */
  @Input() linear: boolean = true;

  /** Whether completed steps can be re-edited */
  @Input() editable: boolean = true;

  /** Orientation: 'horizontal' or 'vertical' */
  @Input() orientation: 'horizontal' | 'vertical' = 'horizontal';

  /** Emitted when the user clicks Submit on the final step */
  @Output() wizardComplete = new EventEmitter<void>();

  /** Emitted on each step change, payload is the new step index */
  @Output() stepChange = new EventEmitter<number>();

  /** Reference to the internal mat-stepper for programmatic control */
  @ViewChild('stepper') stepper: MatStepper;

  /** Content-projected step templates */
  @ContentChildren(MsaccoWizardStepDirective) stepTemplates: QueryList<MsaccoWizardStepDirective>;

  projectedSteps: MsaccoWizardStepDirective[] = [];

  ngAfterContentInit(): void {
    this.projectedSteps = this.stepTemplates?.toArray() ?? [];
  }

  onStepChange(event: any): void {
    this.stepChange.emit(event.selectedIndex);
  }

  goToStep(index: number): void {
    if (this.stepper) {
      this.stepper.selectedIndex = index;
    }
  }

  next(): void {
    this.stepper?.next();
  }

  previous(): void {
    this.stepper?.previous();
  }

  reset(): void {
    this.stepper?.reset();
  }

  complete(): void {
    this.wizardComplete.emit();
  }
}
```

### Step Directive (for content projection)

```typescript
// src/app/shared/msacco-wizard/msacco-wizard-step.directive.ts

import { Directive, Input, TemplateRef, inject } from '@angular/core';
import { UntypedFormGroup } from '@angular/forms';

@Directive({
  selector: '[msaccoWizardStep]',
  standalone: true
})
export class MsaccoWizardStepDirective {
  templateRef = inject(TemplateRef<any>);

  /** Step label */
  @Input('msaccoWizardStep') label: string = '';

  /** Optional icon */
  @Input() stepIcon: string = '';

  /** Optional form group for validation gating */
  @Input() stepFormGroup: UntypedFormGroup | null = null;

  /** Whether step is optional */
  @Input() stepOptional: boolean = false;
}
```

### Template

```html
<!-- src/app/shared/msacco-wizard/msacco-wizard.component.html -->

<mat-stepper #stepper [linear]="linear" [orientation]="orientation" (selectionChange)="onStepChange($event)" class="msacco-wizard-stepper">
  <!-- Content-projected steps -->
  @for (step of projectedSteps; track step.label) {
  <mat-step [stepControl]="step.stepFormGroup" [label]="step.label" [editable]="editable" [optional]="step.stepOptional">
    <ng-template matStepLabel>
      @if (step.stepIcon) {
      <mat-icon class="step-icon">{{ step.stepIcon }}</mat-icon>
      } {{ step.label | translate }}
    </ng-template>

    <ng-container [ngTemplateOutlet]="step.templateRef"></ng-container>

    <!-- Step navigation buttons -->
    <div class="msacco-wizard-actions">
      @if (!$first) {
      <button mat-stroked-button matStepperPrevious type="button">
        <mat-icon>arrow_back</mat-icon>
        {{ 'labels.buttons.Back' | translate }}
      </button>
      } @if (!$last) {
      <button mat-flat-button color="primary" matStepperNext type="button">
        {{ 'labels.buttons.Next' | translate }}
        <mat-icon>arrow_forward</mat-icon>
      </button>
      } @if ($last) {
      <button mat-flat-button color="primary" type="button" (click)="complete()">
        <mat-icon>check</mat-icon>
        {{ 'labels.buttons.Submit' | translate }}
      </button>
      }
    </div>
  </mat-step>
  }
</mat-stepper>
```

### Styles

```scss
// src/app/shared/msacco-wizard/msacco-wizard.component.scss

$primary-blue: #1074b9;
$step-circle-size: 36px;

.msacco-wizard-stepper {
  background: transparent;

  // Override Material stepper indicator styles
  ::ng-deep {
    .mat-step-header {
      .mat-step-icon {
        width: $step-circle-size;
        height: $step-circle-size;
        font-size: 14px;
        font-weight: 600;
        background-color: #e0e0e0;
        color: #666;

        &.mat-step-icon-selected,
        &.mat-step-icon-state-edit {
          background-color: $primary-blue;
          color: #ffffff;
        }

        &.mat-step-icon-state-done {
          background-color: #28a745;
          color: #ffffff;
        }
      }

      .mat-step-label {
        font-size: 13px;
        font-weight: 500;
        color: #666;

        &.mat-step-label-active {
          color: $primary-blue;
          font-weight: 600;
        }

        &.mat-step-label-selected {
          color: $primary-blue;
          font-weight: 600;
        }
      }
    }

    // Connector line between steps
    .mat-stepper-horizontal-line {
      border-color: #e0e0e0;
      min-width: 32px;
    }

    .mat-horizontal-content-container {
      padding: 16px 24px;
    }
  }

  .step-icon {
    font-size: 18px;
    margin-right: 4px;
    vertical-align: middle;
  }
}

.msacco-wizard-actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  padding: 16px 0 0;
  border-top: 1px solid #eee;
  margin-top: 16px;
}
```

### Usage Example

```html
<msacco-wizard [linear]="true" (wizardComplete)="onSubmit()">
  <ng-template msaccoWizardStep="Group Info" stepIcon="info" [stepFormGroup]="groupInfoForm">
    <!-- Step 1 content -->
    <form [formGroup]="groupInfoForm">...</form>
  </ng-template>

  <ng-template msaccoWizardStep="Select Clients" stepIcon="people" [stepFormGroup]="clientsForm">
    <!-- Step 2 content -->
  </ng-template>

  <ng-template msaccoWizardStep="Overview" stepIcon="preview">
    <!-- Step 3 content (read-only summary) -->
  </ng-template>
</msacco-wizard>
```

---

## 4. ApprovalActionBarComponent

**Location:** `src/app/shared/approval-action-bar/`

### Purpose

A sticky bottom bar displaying context-specific action buttons (Approve, Reject, Disburse, etc.) with permission-based visibility. Used on entity detail pages for workflow actions.

### TypeScript Interfaces

```typescript
// src/app/shared/approval-action-bar/approval-action-bar.interfaces.ts

export interface ActionDef {
  /** Button label text (supports i18n keys) */
  label: string;
  /** Material icon name */
  icon: string;
  /** Button color: 'primary' | 'accent' | 'warn' | '' */
  color: 'primary' | 'accent' | 'warn' | '';
  /** Callback function when button is clicked */
  action: () => void;
  /** Permission string required to show this button (uses *mifosxHasPermission) */
  permission?: string;
  /** Whether the button is disabled */
  disabled?: boolean;
  /** Tooltip text */
  tooltip?: string;
}
```

### Component

```typescript
// src/app/shared/approval-action-bar/approval-action-bar.component.ts

import { Component, Input } from '@angular/core';
import { MatButton } from '@angular/material/button';
import { MatIcon } from '@angular/material/icon';
import { MatTooltip } from '@angular/material/tooltip';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { ActionDef } from './approval-action-bar.interfaces';

@Component({
  selector: 'msacco-approval-action-bar',
  templateUrl: './approval-action-bar.component.html',
  styleUrls: ['./approval-action-bar.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatButton,
    MatIcon,
    MatTooltip
  ]
})
export class ApprovalActionBarComponent {
  /** Array of action button definitions */
  @Input() actions: ActionDef[] = [];

  /** Entity context name displayed in the bar (e.g., "Loan Application #12345") */
  @Input() entityName: string = '';

  /** Whether the bar is visible */
  @Input() visible: boolean = true;

  executeAction(action: ActionDef): void {
    if (!action.disabled && action.action) {
      action.action();
    }
  }
}
```

### Template

```html
<!-- src/app/shared/approval-action-bar/approval-action-bar.component.html -->

@if (visible && actions.length > 0) {
<div class="msacco-approval-bar">
  @if (entityName) {
  <span class="entity-name">{{ entityName }}</span>
  }
  <span class="bar-spacer"></span>
  <div class="action-buttons">
    @for (action of actions; track action.label) {
    <button mat-flat-button [color]="action.color" [disabled]="action.disabled" [matTooltip]="action.tooltip || ''" (click)="executeAction(action)" *mifosxHasPermission="action.permission" class="action-btn">
      <mat-icon>{{ action.icon }}</mat-icon>
      {{ action.label | translate }}
    </button>
    }
  </div>
</div>
}
```

### Styles

```scss
// src/app/shared/approval-action-bar/approval-action-bar.component.scss

.msacco-approval-bar {
  position: sticky;
  bottom: 0;
  left: 0;
  right: 0;
  display: flex;
  align-items: center;
  padding: 8px 24px;
  background-color: #ffffff;
  border-top: 2px solid #e0e0e0;
  box-shadow: 0 -2px 6px rgb(0 0 0 / 8%);
  z-index: 100;
  min-height: 56px;
}

.entity-name {
  font-size: 14px;
  font-weight: 500;
  color: #333;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 300px;
}

.bar-spacer {
  flex: 1 1 auto;
}

.action-buttons {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;

  .action-btn {
    font-size: 13px;
    height: 36px;

    mat-icon {
      font-size: 18px;
      width: 18px;
      height: 18px;
      margin-right: 4px;
    }
  }
}
```

### Usage Example: Loan Detail Page

```typescript
loanActions: ActionDef[] = [
  {
    label: 'Approve',
    icon: 'check_circle',
    color: 'primary',
    action: () => this.approveLoan(),
    permission: 'APPROVE_LOAN'
  },
  {
    label: 'Reject',
    icon: 'cancel',
    color: 'warn',
    action: () => this.rejectLoan(),
    permission: 'REJECT_LOAN'
  },
  {
    label: 'Disburse',
    icon: 'payments',
    color: 'primary',
    action: () => this.disburseLoan(),
    permission: 'DISBURSE_LOAN',
    disabled: !this.loanData.isApproved
  }
];
```

```html
<msacco-approval-action-bar [actions]="loanActions" [entityName]="'Loan #' + loanData.accountNo"></msacco-approval-action-bar>
```

---

## 5. StatusBadgeComponent

**Location:** `src/app/shared/status-badge/`

### Purpose

A small colored badge/pill that displays an entity's status (Active, Pending, Rejected, etc.) with consistent color coding across the application.

### Component

```typescript
// src/app/shared/status-badge/status-badge.component.ts

import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

export interface StatusConfig {
  /** Background color (CSS color string) */
  color: string;
  /** Optional text color override. Default: #ffffff */
  textColor?: string;
  /** Display label. If not provided, the raw status string is used. */
  label?: string;
}

/** Default status-to-color mapping used across the application */
export const DEFAULT_STATUS_MAP: Record<string, StatusConfig> = {
  active: { color: '#28a745', label: 'Active' },
  approved: { color: '#28a745', label: 'Approved' },
  pending: { color: '#ffc107', textColor: '#333333', label: 'Pending' },
  submitted: { color: '#ffc107', textColor: '#333333', label: 'Submitted' },
  pendingapproval: { color: '#ffc107', textColor: '#333333', label: 'Pending Approval' },
  rejected: { color: '#dc3545', label: 'Rejected' },
  withdrawn: { color: '#6c757d', label: 'Withdrawn' },
  closed: { color: '#6c757d', label: 'Closed' },
  inactive: { color: '#6c757d', label: 'Inactive' },
  overdue: { color: '#dc3545', label: 'Overdue' },
  disbursed: { color: '#17a2b8', label: 'Disbursed' },
  writtenoff: { color: '#343a40', label: 'Written Off' },
  overpaid: { color: '#007bff', label: 'Overpaid' },
  transfer_in_progress: { color: '#fd7e14', label: 'Transfer In Progress' },
  transfer_on_hold: { color: '#fd7e14', label: 'Transfer On Hold' }
};

@Component({
  selector: 'msacco-status-badge',
  templateUrl: './status-badge.component.html',
  styleUrls: ['./status-badge.component.scss'],
  standalone: true,
  imports: [...STANDALONE_SHARED_IMPORTS]
})
export class StatusBadgeComponent implements OnChanges {
  /** The status value (matched case-insensitively against the statusMap) */
  @Input() status: string = '';

  /** Custom status-to-config map. Merged with defaults (custom takes priority). */
  @Input() statusMap: Record<string, StatusConfig> = {};

  /** Size variant */
  @Input() size: 'sm' | 'md' = 'md';

  displayLabel: string = '';
  backgroundColor: string = '#6c757d';
  textColor: string = '#ffffff';

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['status'] || changes['statusMap']) {
      this.resolveStatus();
    }
  }

  private resolveStatus(): void {
    if (!this.status) {
      this.displayLabel = '';
      return;
    }

    const mergedMap = { ...DEFAULT_STATUS_MAP, ...this.statusMap };
    const normalizedKey = this.status
      .toLowerCase()
      .replace(/[\s_-]+/g, '')
      .trim();

    // Try exact match, then normalized match
    const config = mergedMap[this.status.toLowerCase()] || mergedMap[normalizedKey] || Object.values(mergedMap).find((_v, _i, _a) => false); // fallback to default

    if (config) {
      this.displayLabel = config.label || this.status;
      this.backgroundColor = config.color;
      this.textColor = config.textColor || '#ffffff';
    } else {
      // Unknown status: use gray with the raw status text
      this.displayLabel = this.status;
      this.backgroundColor = '#6c757d';
      this.textColor = '#ffffff';
    }
  }
}
```

### Template

```html
<!-- src/app/shared/status-badge/status-badge.component.html -->

@if (displayLabel) {
<span class="msacco-status-badge" [class.sm]="size === 'sm'" [style.background-color]="backgroundColor" [style.color]="textColor"> {{ displayLabel }} </span>
}
```

### Styles

```scss
// src/app/shared/status-badge/status-badge.component.scss

.msacco-status-badge {
  display: inline-flex;
  align-items: center;
  padding: 2px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
  line-height: 1.5;
  white-space: nowrap;
  letter-spacing: 0.02em;
  text-transform: capitalize;

  &.sm {
    padding: 1px 8px;
    font-size: 11px;
    border-radius: 10px;
  }
}
```

### Default Status Color Reference

| Status           | Color  | Hex       | Text Color  |
| ---------------- | ------ | --------- | ----------- |
| Active           | Green  | `#28a745` | White       |
| Approved         | Green  | `#28a745` | White       |
| Pending          | Yellow | `#ffc107` | Dark `#333` |
| Submitted        | Yellow | `#ffc107` | Dark `#333` |
| Pending Approval | Yellow | `#ffc107` | Dark `#333` |
| Rejected         | Red    | `#dc3545` | White       |
| Closed           | Gray   | `#6c757d` | White       |
| Inactive         | Gray   | `#6c757d` | White       |
| Withdrawn        | Gray   | `#6c757d` | White       |
| Overdue          | Red    | `#dc3545` | White       |
| Disbursed        | Teal   | `#17a2b8` | White       |
| Written Off      | Dark   | `#343a40` | White       |
| Overpaid         | Blue   | `#007bff` | White       |

### Usage Examples

```html
<!-- Basic usage -->
<msacco-status-badge [status]="client.status.value"></msacco-status-badge>

<!-- Small variant -->
<msacco-status-badge [status]="'Active'" size="sm"></msacco-status-badge>

<!-- Custom status map (overrides defaults for specific statuses) -->
<msacco-status-badge
  [status]="loan.status"
  [statusMap]="{
    'in arrears': { color: '#e65100', label: 'In Arrears' },
    'restructured': { color: '#9c27b0', label: 'Restructured' }
  }"
></msacco-status-badge>
```

---

## npm Dependencies

The following packages need to be installed for export functionality:

```bash
npm install exceljs --save
npm install jspdf jspdf-autotable --save
npm install @types/jspdf --save-dev
```

---

## Shared Module Registration

All 5 components are standalone and do not need to be added to a shared NgModule. They can be imported directly by consuming components:

```typescript
// In any feature component that uses the data table:
import { MsaccoDataTableComponent } from 'app/shared/msacco-data-table/msacco-data-table.component';
import { StatusTabsComponent } from 'app/shared/status-tabs/status-tabs.component';
import { StatusBadgeComponent } from 'app/shared/status-badge/status-badge.component';
import { MsaccoWizardComponent } from 'app/shared/msacco-wizard/msacco-wizard.component';
import { MsaccoWizardStepDirective } from 'app/shared/msacco-wizard/msacco-wizard-step.directive';
import { ApprovalActionBarComponent } from 'app/shared/approval-action-bar/approval-action-bar.component';

@Component({
  imports: [
    MsaccoDataTableComponent,
    StatusBadgeComponent,
    MsaccoWizardComponent,
    MsaccoWizardStepDirective,
    ApprovalActionBarComponent
  ]
})
```

---

## File Checklist

| #   | Component         | Files to Create                                                                                                              |
| --- | ----------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 1   | MsaccoDataTable   | `msacco-data-table.interfaces.ts`, `msacco-data-table.component.ts`, `.html`, `.scss`, `msacco-data-table-export.service.ts` |
| 2   | StatusTabs        | `status-tabs.component.ts`, `.html`, `.scss`                                                                                 |
| 3   | MsaccoWizard      | `msacco-wizard.interfaces.ts`, `msacco-wizard.component.ts`, `.html`, `.scss`, `msacco-wizard-step.directive.ts`             |
| 4   | ApprovalActionBar | `approval-action-bar.interfaces.ts`, `approval-action-bar.component.ts`, `.html`, `.scss`                                    |
| 5   | StatusBadge       | `status-badge.component.ts`, `.html`, `.scss`                                                                                |
