# TECH-007: Data Exports Module Specification

## Overview

This specification covers a new Data Exports module at `src/app/data-exports/`. The module provides a wizard-based interface for building custom data exports from Fineract entities, with drag-and-drop field selection using Angular CDK, configurable filters, and multi-format export (CSV, Excel, PDF).

---

## 1. Module Architecture

### 1.1 File Structure

```
src/app/data-exports/
  ├── data-exports.module.ts
  ├── data-exports-routing.module.ts
  ├── data-exports.service.ts
  ├── models/
  │   └── data-export.model.ts
  ├── data-export-list/
  │   ├── data-export-list.component.ts
  │   ├── data-export-list.component.html
  │   └── data-export-list.component.scss
  └── create-data-export/
      ├── create-data-export.component.ts
      ├── create-data-export.component.html
      ├── create-data-export.component.scss
      ├── step-basic-info/
      │   ├── step-basic-info.component.ts
      │   └── step-basic-info.component.html
      ├── step-field-selection/
      │   ├── step-field-selection.component.ts
      │   └── step-field-selection.component.html
      ├── step-filters/
      │   ├── step-filters.component.ts
      │   └── step-filters.component.html
      └── step-preview/
          ├── step-preview.component.ts
          └── step-preview.component.html
```

### 1.2 Routing

```
/data-exports
  / (list)                       -> DataExportListComponent
  /create                        -> CreateDataExportComponent
  /:exportId/edit                -> CreateDataExportComponent (edit mode)
```

---

## 2. Data Models

```typescript
// models/data-export.model.ts

export interface DataExport {
  id: number;
  name: string;
  baseEntity: BaseEntity;
  selectedFields: ExportField[];
  filters: ExportFilter[];
  format: ExportFormat;
  createdBy: string;
  createdDate: string;
  lastRunDate?: string;
}

export enum BaseEntity {
  CLIENTS = 'clients',
  LOANS = 'loans',
  SAVINGS = 'savings',
  GROUPS = 'groups',
  JOURNAL_ENTRIES = 'journalEntries',
  SHARE_ACCOUNTS = 'shareAccounts',
  OFFICES = 'offices',
  STAFF = 'staff'
}

export interface ExportField {
  /** Column/field name from the entity */
  name: string;
  /** Human-readable label */
  label: string;
  /** Data type for filter operator selection */
  type: 'string' | 'number' | 'date' | 'boolean' | 'enum';
  /** Ordinal position in export (set via drag-and-drop) */
  order: number;
  /** For enum types, the set of allowed values */
  enumValues?: string[];
}

export interface ExportFilter {
  field: string;
  operator: FilterOperator;
  value: any;
  /** For 'between' operator */
  valueTo?: any;
  /** For combining multiple filters */
  conjunction: 'AND' | 'OR';
}

export enum FilterOperator {
  EQUALS = 'equals',
  CONTAINS = 'contains',
  GREATER_THAN = 'greaterThan',
  LESS_THAN = 'lessThan',
  BETWEEN = 'between',
  IN = 'in'
}

export enum ExportFormat {
  CSV = 'csv',
  EXCEL = 'xlsx',
  PDF = 'pdf'
}

export interface EntityFieldMetadata {
  entityName: string;
  fields: ExportField[];
}
```

---

## 3. Module and Routing

```typescript
// data-exports.module.ts

import { NgModule } from '@angular/core';
import { DataExportsRoutingModule } from './data-exports-routing.module';
import { DataExportsService } from './data-exports.service';

@NgModule({
  imports: [DataExportsRoutingModule],
  providers: [DataExportsService]
})
export class DataExportsModule {}
```

```typescript
// data-exports-routing.module.ts

import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { DataExportListComponent } from './data-export-list/data-export-list.component';
import { CreateDataExportComponent } from './create-data-export/create-data-export.component';

const routes: Routes = [
  {
    path: '',
    children: [
      { path: '', component: DataExportListComponent, data: { title: 'Data Exports', breadcrumb: 'Data Exports' } },
      { path: 'create', component: CreateDataExportComponent, data: { title: 'Create Data Export', breadcrumb: 'Create' } },
      { path: ':exportId/edit', component: CreateDataExportComponent, data: { title: 'Edit Data Export', breadcrumb: 'Edit' } }
    ]
  }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class DataExportsRoutingModule {}
```

---

## 4. Service

```typescript
// data-exports.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, of } from 'rxjs';
import { map } from 'rxjs/operators';
import { DataExport, BaseEntity, ExportField, EntityFieldMetadata, ExportFormat } from './models/data-export.model';

@Injectable({ providedIn: 'root' })
export class DataExportsService {
  private http = inject(HttpClient);

  /**
   * Entity-to-Fineract-API mapping for field metadata discovery.
   * Fineract does not expose a generic field metadata endpoint,
   * so we derive fields from datatable definitions and template endpoints.
   */
  private entityEndpoints: Record<BaseEntity, string> = {
    [BaseEntity.CLIENTS]: '/clients/template',
    [BaseEntity.LOANS]: '/loans/template?templateType=individual',
    [BaseEntity.SAVINGS]: '/savingsaccounts/template',
    [BaseEntity.GROUPS]: '/groups/template',
    [BaseEntity.JOURNAL_ENTRIES]: '/journalentries/template',
    [BaseEntity.SHARE_ACCOUNTS]: '/accounts/share/template',
    [BaseEntity.OFFICES]: '/offices',
    [BaseEntity.STAFF]: '/staff'
  };

  /**
   * Get available fields for a base entity.
   * Combines hard-coded core fields with dynamic datatable columns.
   */
  getEntityFields(entity: BaseEntity): Observable<EntityFieldMetadata> {
    return new Observable((observer) => {
      // First get core fields (hard-coded per entity)
      const coreFields = this.getCoreFields(entity);

      // Then try to load additional datatable fields
      this.getDatatableFields(entity).subscribe(
        (dtFields) => {
          observer.next({
            entityName: entity,
            fields: [
              ...coreFields,
              ...dtFields
            ]
          });
          observer.complete();
        },
        () => {
          observer.next({ entityName: entity, fields: coreFields });
          observer.complete();
        }
      );
    });
  }

  /**
   * Get registered datatables for an entity.
   * Fineract: GET /datatables?apptable={entityTable}
   */
  getDatatableFields(entity: BaseEntity): Observable<ExportField[]> {
    const tableMapping: Partial<Record<BaseEntity, string>> = {
      [BaseEntity.CLIENTS]: 'm_client',
      [BaseEntity.LOANS]: 'm_loan',
      [BaseEntity.SAVINGS]: 'm_savings_account',
      [BaseEntity.GROUPS]: 'm_group'
    };

    const table = tableMapping[entity];
    if (!table) return of([]);

    const httpParams = new HttpParams().set('apptable', table);
    return this.http.get<any[]>('/datatables', { params: httpParams }).pipe(
      map((datatables) => {
        const fields: ExportField[] = [];
        for (const dt of datatables) {
          for (const col of dt.columnHeaderData || []) {
            if (col.columnName === 'id') continue;
            fields.push({
              name: `${dt.registeredTableName}.${col.columnName}`,
              label: `${dt.registeredTableName}: ${col.columnDisplayType || col.columnName}`,
              type: this.mapColumnType(col.columnDisplayType),
              order: 0
            });
          }
        }
        return fields;
      })
    );
  }

  /**
   * Core fields per entity. These represent the standard columns
   * available from the Fineract search/list endpoints.
   */
  private getCoreFields(entity: BaseEntity): ExportField[] {
    const fieldSets: Record<BaseEntity, ExportField[]> = {
      [BaseEntity.CLIENTS]: [
        { name: 'id', label: 'Client ID', type: 'number', order: 0 },
        { name: 'accountNo', label: 'Account Number', type: 'string', order: 0 },
        { name: 'displayName', label: 'Display Name', type: 'string', order: 0 },
        { name: 'firstname', label: 'First Name', type: 'string', order: 0 },
        { name: 'lastname', label: 'Last Name', type: 'string', order: 0 },
        { name: 'officeName', label: 'Office', type: 'string', order: 0 },
        { name: 'status', label: 'Status', type: 'enum', order: 0, enumValues: [
            'Active',
            'Pending',
            'Closed',
            'Rejected',
            'Withdrawn'
          ] },
        { name: 'activationDate', label: 'Activation Date', type: 'date', order: 0 },
        { name: 'dateOfBirth', label: 'Date of Birth', type: 'date', order: 0 },
        { name: 'gender', label: 'Gender', type: 'enum', order: 0, enumValues: [
            'Male',
            'Female'
          ] },
        { name: 'mobileNo', label: 'Mobile Number', type: 'string', order: 0 },
        { name: 'externalId', label: 'External ID', type: 'string', order: 0 },
        { name: 'submittedOnDate', label: 'Submitted Date', type: 'date', order: 0 }
      ],
      [BaseEntity.LOANS]: [
        { name: 'id', label: 'Loan ID', type: 'number', order: 0 },
        { name: 'accountNo', label: 'Account Number', type: 'string', order: 0 },
        { name: 'clientName', label: 'Client Name', type: 'string', order: 0 },
        { name: 'productName', label: 'Product', type: 'string', order: 0 },
        { name: 'principal', label: 'Principal', type: 'number', order: 0 },
        { name: 'status', label: 'Status', type: 'enum', order: 0, enumValues: [
            'Submitted',
            'Approved',
            'Active',
            'Overpaid',
            'Closed',
            'Written-Off'
          ] },
        { name: 'disbursementDate', label: 'Disbursement Date', type: 'date', order: 0 },
        { name: 'maturityDate', label: 'Maturity Date', type: 'date', order: 0 },
        { name: 'interestRatePerPeriod', label: 'Interest Rate', type: 'number', order: 0 },
        { name: 'numberOfRepayments', label: 'Number of Repayments', type: 'number', order: 0 },
        { name: 'totalOutstandingAmount', label: 'Outstanding Amount', type: 'number', order: 0 },
        { name: 'totalRepaymentAmount', label: 'Total Repayment', type: 'number', order: 0 },
        { name: 'loanOfficerName', label: 'Loan Officer', type: 'string', order: 0 }
      ],
      [BaseEntity.SAVINGS]: [
        { name: 'id', label: 'Savings ID', type: 'number', order: 0 },
        { name: 'accountNo', label: 'Account Number', type: 'string', order: 0 },
        { name: 'clientName', label: 'Client Name', type: 'string', order: 0 },
        { name: 'productName', label: 'Product', type: 'string', order: 0 },
        { name: 'accountBalance', label: 'Account Balance', type: 'number', order: 0 },
        { name: 'status', label: 'Status', type: 'enum', order: 0, enumValues: [
            'Submitted',
            'Approved',
            'Active',
            'Closed',
            'Withdrawn'
          ] },
        { name: 'activatedOnDate', label: 'Activation Date', type: 'date', order: 0 },
        { name: 'nominalAnnualInterestRate', label: 'Interest Rate', type: 'number', order: 0 }
      ],
      [BaseEntity.GROUPS]: [
        { name: 'id', label: 'Group ID', type: 'number', order: 0 },
        { name: 'name', label: 'Group Name', type: 'string', order: 0 },
        { name: 'officeName', label: 'Office', type: 'string', order: 0 },
        { name: 'status', label: 'Status', type: 'enum', order: 0, enumValues: [
            'Pending',
            'Active',
            'Closed'
          ] },
        { name: 'activationDate', label: 'Activation Date', type: 'date', order: 0 },
        { name: 'externalId', label: 'External ID', type: 'string', order: 0 }
      ],
      [BaseEntity.JOURNAL_ENTRIES]: [
        { name: 'id', label: 'Entry ID', type: 'number', order: 0 },
        { name: 'officeId', label: 'Office ID', type: 'number', order: 0 },
        { name: 'officeName', label: 'Office', type: 'string', order: 0 },
        { name: 'glAccountName', label: 'GL Account', type: 'string', order: 0 },
        { name: 'entryDate', label: 'Entry Date', type: 'date', order: 0 },
        { name: 'amount', label: 'Amount', type: 'number', order: 0 },
        { name: 'transactionId', label: 'Transaction ID', type: 'string', order: 0 },
        { name: 'type', label: 'Type', type: 'enum', order: 0, enumValues: [
            'DEBIT',
            'CREDIT'
          ] }
      ],
      [BaseEntity.SHARE_ACCOUNTS]: [
        { name: 'id', label: 'Share Account ID', type: 'number', order: 0 },
        { name: 'accountNo', label: 'Account Number', type: 'string', order: 0 },
        { name: 'clientName', label: 'Client Name', type: 'string', order: 0 },
        { name: 'productName', label: 'Product', type: 'string', order: 0 },
        { name: 'totalApprovedShares', label: 'Approved Shares', type: 'number', order: 0 }
      ],
      [BaseEntity.OFFICES]: [
        { name: 'id', label: 'Office ID', type: 'number', order: 0 },
        { name: 'name', label: 'Office Name', type: 'string', order: 0 },
        { name: 'hierarchy', label: 'Hierarchy', type: 'string', order: 0 },
        { name: 'openingDate', label: 'Opening Date', type: 'date', order: 0 },
        { name: 'externalId', label: 'External ID', type: 'string', order: 0 }
      ],
      [BaseEntity.STAFF]: [
        { name: 'id', label: 'Staff ID', type: 'number', order: 0 },
        { name: 'displayName', label: 'Display Name', type: 'string', order: 0 },
        { name: 'officeName', label: 'Office', type: 'string', order: 0 },
        { name: 'isLoanOfficer', label: 'Is Loan Officer', type: 'boolean', order: 0 },
        { name: 'isActive', label: 'Is Active', type: 'boolean', order: 0 },
        { name: 'joiningDate', label: 'Joining Date', type: 'date', order: 0 }
      ]
    };

    return fieldSets[entity] || [];
  }

  private mapColumnType(displayType: string): 'string' | 'number' | 'date' | 'boolean' | 'enum' {
    if (!displayType) return 'string';
    const lower = displayType.toLowerCase();
    if (lower.includes('integer') || lower.includes('decimal') || lower.includes('number')) return 'number';
    if (lower.includes('date') || lower.includes('datetime')) return 'date';
    if (lower.includes('boolean') || lower.includes('codelookup')) return 'boolean';
    return 'string';
  }

  /**
   * Save a data export configuration.
   * Uses a custom Fineract datatable to store export definitions.
   */
  saveExport(exportConfig: Partial<DataExport>): Observable<any> {
    return this.http.post('/datatables/data_exports', {
      name: exportConfig.name,
      baseEntity: exportConfig.baseEntity,
      selectedFields: JSON.stringify(exportConfig.selectedFields),
      filters: JSON.stringify(exportConfig.filters),
      format: exportConfig.format
    });
  }

  updateExport(id: number, exportConfig: Partial<DataExport>): Observable<any> {
    return this.http.put(`/datatables/data_exports/${id}`, {
      name: exportConfig.name,
      baseEntity: exportConfig.baseEntity,
      selectedFields: JSON.stringify(exportConfig.selectedFields),
      filters: JSON.stringify(exportConfig.filters),
      format: exportConfig.format
    });
  }

  getExports(): Observable<DataExport[]> {
    return this.http.get<any[]>('/datatables/data_exports').pipe(
      map((rows) =>
        rows.map((row) => ({
          ...row,
          selectedFields: JSON.parse(row.selectedFields || '[]'),
          filters: JSON.parse(row.filters || '[]')
        }))
      )
    );
  }

  getExport(id: number): Observable<DataExport> {
    return this.http.get<any>(`/datatables/data_exports/${id}`).pipe(
      map((row) => ({
        ...row,
        selectedFields: JSON.parse(row.selectedFields || '[]'),
        filters: JSON.parse(row.filters || '[]')
      }))
    );
  }

  deleteExport(id: number): Observable<void> {
    return this.http.delete<void>(`/datatables/data_exports/${id}`);
  }

  /**
   * Run an export and download the result.
   * This builds a report query from the export config and fetches data.
   */
  runExport(exportConfig: DataExport): Observable<Blob> {
    const params = this.buildExportParams(exportConfig);
    const endpoint = this.getEntityListEndpoint(exportConfig.baseEntity);
    const responseType = exportConfig.format === ExportFormat.PDF ? 'blob' : 'blob';

    return this.http.get(endpoint, {
      params,
      responseType: 'blob' as 'json'
    }) as Observable<Blob>;
  }

  private getEntityListEndpoint(entity: BaseEntity): string {
    const endpoints: Record<BaseEntity, string> = {
      [BaseEntity.CLIENTS]: '/clients',
      [BaseEntity.LOANS]: '/loans',
      [BaseEntity.SAVINGS]: '/savingsaccounts',
      [BaseEntity.GROUPS]: '/groups',
      [BaseEntity.JOURNAL_ENTRIES]: '/journalentries',
      [BaseEntity.SHARE_ACCOUNTS]: '/accounts/share',
      [BaseEntity.OFFICES]: '/offices',
      [BaseEntity.STAFF]: '/staff'
    };
    return endpoints[entity];
  }

  private buildExportParams(exportConfig: DataExport): HttpParams {
    let params = new HttpParams().set('limit', '10000');
    for (const filter of exportConfig.filters) {
      params = params.set(filter.field, filter.value);
    }
    return params;
  }
}
```

---

## 5. Page Components

### 5.1 Data Export List

```typescript
// data-export-list/data-export-list.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { Router } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { MatTableDataSource } from '@angular/material/table';
import { MatTabGroup, MatTab } from '@angular/material/tabs';
import { MatTable, MatColumnDef, MatHeaderCellDef, MatHeaderCell, MatCellDef, MatCell, MatHeaderRowDef, MatHeaderRow, MatRowDef, MatRow } from '@angular/material/table';
import { MatIconButton } from '@angular/material/button';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { DataExportsService } from '../data-exports.service';
import { DataExport } from '../models/data-export.model';
import { DeleteDialogComponent } from 'app/shared/delete-dialog/delete-dialog.component';
import { DateFormatPipe } from 'app/pipes/date-format.pipe';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-data-export-list',
  templateUrl: './data-export-list.component.html',
  styleUrls: ['./data-export-list.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    FaIconComponent,
    MatIconButton,
    MatTabGroup,
    MatTab,
    MatTable,
    MatColumnDef,
    MatHeaderCellDef,
    MatHeaderCell,
    MatCellDef,
    MatCell,
    MatHeaderRowDef,
    MatHeaderRow,
    MatRowDef,
    MatRow,
    DateFormatPipe
  ]
})
export class DataExportListComponent implements OnInit {
  private exportService = inject(DataExportsService);
  private router = inject(Router);
  private dialog = inject(MatDialog);

  exports: DataExport[] = [];
  dataSource: MatTableDataSource<DataExport>;
  displayedColumns: string[] = [
    'name',
    'baseEntity',
    'fieldsCount',
    'createdBy',
    'createdDate',
    'lastRunDate',
    'actions'
  ];
  activeTab = 0;

  ngOnInit() {
    this.loadExports();
  }

  loadExports() {
    this.exportService.getExports().subscribe((data: DataExport[]) => {
      this.exports = data;
      this.dataSource = new MatTableDataSource(this.exports);
    });
  }

  runExport(exportConfig: DataExport) {
    this.exportService.runExport(exportConfig).subscribe((blob: Blob) => {
      const extension = exportConfig.format || 'csv';
      const url = window.URL.createObjectURL(blob);
      const anchor = document.createElement('a');
      anchor.href = url;
      anchor.download = `${exportConfig.name}.${extension}`;
      anchor.click();
      window.URL.revokeObjectURL(url);
    });
  }

  editExport(exportConfig: DataExport) {
    this.router.navigate([
      '/data-exports',
      exportConfig.id,
      'edit'
    ]);
  }

  deleteExport(exportConfig: DataExport) {
    const deleteRef = this.dialog.open(DeleteDialogComponent, {
      data: { deleteContext: `export "${exportConfig.name}"` }
    });
    deleteRef.afterClosed().subscribe((response: any) => {
      if (response.delete) {
        this.exportService.deleteExport(exportConfig.id).subscribe(() => {
          this.loadExports();
        });
      }
    });
  }

  filterByOwner(tabIndex: number) {
    this.activeTab = tabIndex;
    // Tab 0 = All, Tab 1 = My Exports
    // Filtering by current user would use AuthenticationService
  }
}
```

**Template: `data-export-list.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <div class="layout-row align-center gap-5px margin-b">
        <h3 class="mat-h3 flex">{{ 'labels.heading.Data Exports' | translate }}</h3>
        <button mat-raised-button color="primary" [routerLink]="['/data-exports/create']">
          <fa-icon icon="plus" class="m-r-10"></fa-icon>
          {{ 'labels.buttons.Create Data Export' | translate }}
        </button>
      </div>

      <mat-tab-group (selectedTabChange)="filterByOwner($event.index)">
        <mat-tab [label]="'labels.tabs.All' | translate"></mat-tab>
        <mat-tab [label]="'labels.tabs.My Exports' | translate"></mat-tab>
      </mat-tab-group>

      <table mat-table [dataSource]="dataSource" class="mat-elevation-z1" [hidden]="!exports || exports.length === 0">
        <ng-container matColumnDef="name">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Export Name' | translate }}</th>
          <td mat-cell *matCellDef="let e">{{ e.name }}</td>
        </ng-container>

        <ng-container matColumnDef="baseEntity">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Entity' | translate }}</th>
          <td mat-cell *matCellDef="let e">{{ e.baseEntity }}</td>
        </ng-container>

        <ng-container matColumnDef="fieldsCount">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Fields' | translate }}</th>
          <td mat-cell *matCellDef="let e">{{ e.selectedFields?.length || 0 }}</td>
        </ng-container>

        <ng-container matColumnDef="createdBy">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Created By' | translate }}</th>
          <td mat-cell *matCellDef="let e">{{ e.createdBy }}</td>
        </ng-container>

        <ng-container matColumnDef="createdDate">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Created Date' | translate }}</th>
          <td mat-cell *matCellDef="let e">{{ e.createdDate | dateFormat }}</td>
        </ng-container>

        <ng-container matColumnDef="lastRunDate">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Last Run' | translate }}</th>
          <td mat-cell *matCellDef="let e">{{ (e.lastRunDate | dateFormat) || '-' }}</td>
        </ng-container>

        <ng-container matColumnDef="actions">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Actions' | translate }}</th>
          <td mat-cell *matCellDef="let e">
            <button mat-icon-button color="primary" (click)="runExport(e)" matTooltip="Run">
              <fa-icon icon="play"></fa-icon>
            </button>
            <button mat-icon-button (click)="editExport(e)" matTooltip="Edit">
              <fa-icon icon="pen"></fa-icon>
            </button>
            <button mat-icon-button color="warn" (click)="deleteExport(e)" matTooltip="Delete">
              <fa-icon icon="trash"></fa-icon>
            </button>
          </td>
        </ng-container>

        <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
        <tr mat-row *matRowDef="let row; columns: displayedColumns"></tr>
      </table>

      @if (!exports || exports.length === 0) {
      <p class="no-data">{{ 'labels.text.No data exports configured' | translate }}</p>
      }
    </mat-card-content>
  </mat-card>
</div>
```

---

### 5.2 Create Data Export (Stepper Wizard)

```typescript
// create-data-export/create-data-export.component.ts

import { Component, OnInit, ViewChild, inject } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { MatStepper, MatStep, MatStepLabel, MatStepperNext, MatStepperPrevious } from '@angular/material/stepper';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { DataExportsService } from '../data-exports.service';
import { DataExport, BaseEntity, ExportField, ExportFilter, ExportFormat } from '../models/data-export.model';
import { StepBasicInfoComponent } from './step-basic-info/step-basic-info.component';
import { StepFieldSelectionComponent } from './step-field-selection/step-field-selection.component';
import { StepFiltersComponent } from './step-filters/step-filters.component';
import { StepPreviewComponent } from './step-preview/step-preview.component';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-create-data-export',
  templateUrl: './create-data-export.component.html',
  styleUrls: ['./create-data-export.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatStepper,
    MatStep,
    MatStepLabel,
    MatStepperNext,
    MatStepperPrevious,
    FaIconComponent,
    StepBasicInfoComponent,
    StepFieldSelectionComponent,
    StepFiltersComponent,
    StepPreviewComponent
  ]
})
export class CreateDataExportComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private exportService = inject(DataExportsService);

  @ViewChild('stepper') stepper: MatStepper;

  exportId: number | null = null;
  isEditMode = false;

  // Step 1 data
  exportName = '';
  baseEntity: BaseEntity | null = null;

  // Step 2 data
  availableFields: ExportField[] = [];
  selectedFields: ExportField[] = [];

  // Step 3 data
  filters: ExportFilter[] = [];

  // Step 4 data
  exportFormat: ExportFormat = ExportFormat.CSV;
  previewData: any[] = [];

  ngOnInit() {
    const paramId = this.route.snapshot.params['exportId'];
    if (paramId) {
      this.exportId = +paramId;
      this.isEditMode = true;
      this.loadExistingExport();
    }
  }

  loadExistingExport() {
    this.exportService.getExport(this.exportId!).subscribe((exp: DataExport) => {
      this.exportName = exp.name;
      this.baseEntity = exp.baseEntity;
      this.selectedFields = exp.selectedFields;
      this.filters = exp.filters;
      this.exportFormat = exp.format;
      this.onEntityChanged(this.baseEntity);
    });
  }

  onEntityChanged(entity: BaseEntity) {
    this.baseEntity = entity;
    this.exportService.getEntityFields(entity).subscribe((metadata) => {
      // Available = all fields minus already selected
      const selectedNames = new Set(this.selectedFields.map((f) => f.name));
      this.availableFields = metadata.fields.filter((f) => !selectedNames.has(f.name));
    });
  }

  save() {
    // Assign order based on position
    this.selectedFields.forEach((f, i) => (f.order = i));

    const config: Partial<DataExport> = {
      name: this.exportName,
      baseEntity: this.baseEntity!,
      selectedFields: this.selectedFields,
      filters: this.filters,
      format: this.exportFormat
    };

    const op = this.isEditMode ? this.exportService.updateExport(this.exportId!, config) : this.exportService.saveExport(config);

    op.subscribe(() => {
      this.router.navigate(['/data-exports']);
    });
  }
}
```

**Template: `create-data-export.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <h3 class="mat-h3">{{ (isEditMode ? 'labels.heading.Edit Data Export' : 'labels.heading.Create Data Export') | translate }}</h3>

      <mat-stepper linear #stepper>
        <!-- Step 1: Basic Info -->
        <mat-step [label]="'labels.steps.Basic Info' | translate">
          <mifosx-step-basic-info [(exportName)]="exportName" [(baseEntity)]="baseEntity" (entityChanged)="onEntityChanged($event)"> </mifosx-step-basic-info>
          <div class="stepper-buttons layout-row gap-5px margin-t">
            <button mat-raised-button matStepperNext [disabled]="!exportName || !baseEntity">
              {{ 'labels.buttons.Next' | translate }}
              <fa-icon icon="arrow-right" class="m-l-10"></fa-icon>
            </button>
          </div>
        </mat-step>

        <!-- Step 2: Field Selection -->
        <mat-step [label]="'labels.steps.Select Fields' | translate">
          <mifosx-step-field-selection [availableFields]="availableFields" [(selectedFields)]="selectedFields"> </mifosx-step-field-selection>
          <div class="stepper-buttons layout-row gap-5px margin-t">
            <button mat-raised-button matStepperPrevious>
              <fa-icon icon="arrow-left" class="m-r-10"></fa-icon>
              {{ 'labels.buttons.Previous' | translate }}
            </button>
            <button mat-raised-button matStepperNext [disabled]="selectedFields.length === 0">
              {{ 'labels.buttons.Next' | translate }}
              <fa-icon icon="arrow-right" class="m-l-10"></fa-icon>
            </button>
          </div>
        </mat-step>

        <!-- Step 3: Filters -->
        <mat-step [label]="'labels.steps.Filters' | translate" [optional]="true">
          <mifosx-step-filters [availableFields]="selectedFields" [(filters)]="filters"> </mifosx-step-filters>
          <div class="stepper-buttons layout-row gap-5px margin-t">
            <button mat-raised-button matStepperPrevious>
              <fa-icon icon="arrow-left" class="m-r-10"></fa-icon>
              {{ 'labels.buttons.Previous' | translate }}
            </button>
            <button mat-raised-button matStepperNext>
              {{ 'labels.buttons.Next' | translate }}
              <fa-icon icon="arrow-right" class="m-l-10"></fa-icon>
            </button>
          </div>
        </mat-step>

        <!-- Step 4: Preview & Save -->
        <mat-step [label]="'labels.steps.Preview & Save' | translate">
          <mifosx-step-preview [selectedFields]="selectedFields" [filters]="filters" [(exportFormat)]="exportFormat" [baseEntity]="baseEntity"> </mifosx-step-preview>
          <div class="stepper-buttons layout-row gap-5px margin-t">
            <button mat-raised-button matStepperPrevious>
              <fa-icon icon="arrow-left" class="m-r-10"></fa-icon>
              {{ 'labels.buttons.Previous' | translate }}
            </button>
            <button mat-raised-button color="primary" (click)="save()">
              <fa-icon icon="save" class="m-r-10"></fa-icon>
              {{ 'labels.buttons.Save' | translate }}
            </button>
          </div>
        </mat-step>
      </mat-stepper>
    </mat-card-content>
  </mat-card>
</div>
```

---

### 5.3 Step 2: Field Selection with Angular CDK DragDrop

This is the core UI innovation of the module. Two side-by-side panels connected via `cdkDropListGroup` allow transferring fields from available to selected and reordering within selected.

```typescript
// step-field-selection/step-field-selection.component.ts

import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CdkDragDrop, CdkDrag, CdkDropList, CdkDropListGroup, CdkDragHandle, moveItemInArray, transferArrayItem } from '@angular/cdk/drag-drop';
import { MatIconButton } from '@angular/material/button';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { ExportField } from '../../models/data-export.model';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-step-field-selection',
  templateUrl: './step-field-selection.component.html',
  styleUrls: ['./step-field-selection.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    CdkDrag,
    CdkDropList,
    CdkDropListGroup,
    CdkDragHandle,
    MatIconButton,
    FaIconComponent
  ]
})
export class StepFieldSelectionComponent {
  @Input() availableFields: ExportField[] = [];
  @Input() selectedFields: ExportField[] = [];
  @Output() selectedFieldsChange = new EventEmitter<ExportField[]>();

  /**
   * Handles drag-and-drop events for both panels.
   * - Within same list: reorder
   * - Between lists: transfer
   */
  onDrop(event: CdkDragDrop<ExportField[]>) {
    if (event.previousContainer === event.container) {
      // Reorder within the same list
      moveItemInArray(event.container.data, event.previousIndex, event.currentIndex);
    } else {
      // Transfer between lists
      transferArrayItem(event.previousContainer.data, event.container.data, event.previousIndex, event.currentIndex);
    }
    this.emitChanges();
  }

  /**
   * Add a single field from available to selected (click handler).
   */
  addField(field: ExportField, index: number) {
    this.availableFields.splice(index, 1);
    this.selectedFields.push(field);
    this.emitChanges();
  }

  /**
   * Remove a field from selected back to available.
   */
  removeField(field: ExportField, index: number) {
    this.selectedFields.splice(index, 1);
    this.availableFields.push(field);
    this.emitChanges();
  }

  /**
   * Add all available fields to selected.
   */
  addAll() {
    this.selectedFields.push(...this.availableFields);
    this.availableFields = [];
    this.emitChanges();
  }

  /**
   * Remove all selected fields back to available.
   */
  removeAll() {
    this.availableFields.push(...this.selectedFields);
    this.selectedFields = [];
    this.emitChanges();
  }

  private emitChanges() {
    this.selectedFieldsChange.emit([...this.selectedFields]);
  }
}
```

**Template: `step-field-selection.component.html`**

```html
<div class="field-selection-container layout-row gap-2px responsive-column">
  <!-- Available Fields Panel (left) -->
  <div class="field-panel flex-48">
    <div class="panel-header layout-row align-center gap-5px">
      <h4 class="mat-h4 flex">{{ 'labels.heading.Available Fields' | translate }} ({{ availableFields.length }})</h4>
      <button mat-icon-button (click)="addAll()" matTooltip="Add all" [disabled]="availableFields.length === 0">
        <fa-icon icon="angle-double-right"></fa-icon>
      </button>
    </div>

    <div cdkDropList #availableList="cdkDropList" [cdkDropListData]="availableFields" [cdkDropListConnectedTo]="[selectedList]" class="field-list" (cdkDropListDropped)="onDrop($event)">
      @for (field of availableFields; track field.name; let i = $index) {
      <div class="field-item" cdkDrag>
        <div class="field-item-content layout-row align-center">
          <fa-icon icon="grip-vertical" class="drag-handle m-r-10" cdkDragHandle></fa-icon>
          <span class="flex">{{ field.label }}</span>
          <span class="field-type-badge">{{ field.type }}</span>
          <button mat-icon-button (click)="addField(field, i)" class="add-btn">
            <fa-icon icon="plus"></fa-icon>
          </button>
        </div>
        <!-- Drag placeholder -->
        <div class="field-placeholder" *cdkDragPlaceholder></div>
      </div>
      }
    </div>
  </div>

  <!-- Selected Fields Panel (right) -->
  <div class="field-panel flex-48">
    <div class="panel-header layout-row align-center gap-5px">
      <h4 class="mat-h4 flex">{{ 'labels.heading.Selected Fields' | translate }} ({{ selectedFields.length }})</h4>
      <button mat-icon-button (click)="removeAll()" matTooltip="Remove all" [disabled]="selectedFields.length === 0">
        <fa-icon icon="angle-double-left"></fa-icon>
      </button>
    </div>

    <div cdkDropList #selectedList="cdkDropList" [cdkDropListData]="selectedFields" [cdkDropListConnectedTo]="[availableList]" class="field-list selected-list" (cdkDropListDropped)="onDrop($event)">
      @for (field of selectedFields; track field.name; let i = $index) {
      <div class="field-item selected" cdkDrag>
        <div class="field-item-content layout-row align-center">
          <fa-icon icon="grip-vertical" class="drag-handle m-r-10" cdkDragHandle></fa-icon>
          <span class="field-order">{{ i + 1 }}.</span>
          <span class="flex m-l-5">{{ field.label }}</span>
          <button mat-icon-button color="warn" (click)="removeField(field, i)">
            <fa-icon icon="times"></fa-icon>
          </button>
        </div>
        <div class="field-placeholder" *cdkDragPlaceholder></div>
      </div>
      } @if (selectedFields.length === 0) {
      <div class="empty-state">
        <p>{{ 'labels.text.Drag fields here or click + to add' | translate }}</p>
      </div>
      }
    </div>
  </div>
</div>
```

**Styles: `step-field-selection.component.scss`**

```scss
.field-selection-container {
  min-height: 400px;
}

.field-panel {
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.panel-header {
  background: #f5f5f5;
  padding: 8px 16px;
  border-bottom: 1px solid #e0e0e0;
}

.field-list {
  min-height: 300px;
  max-height: 500px;
  overflow-y: auto;
  padding: 4px;
}

.field-item {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  margin: 4px 0;
  cursor: move;

  &.selected {
    border-left: 3px solid #3f51b5;
  }
}

.field-item-content {
  padding: 8px 12px;
}

.drag-handle {
  color: #9e9e9e;
  cursor: grab;
}

.field-type-badge {
  font-size: 11px;
  padding: 2px 6px;
  border-radius: 10px;
  background: #e8eaf6;
  color: #3f51b5;
  margin-right: 8px;
}

.field-order {
  font-weight: 500;
  color: #3f51b5;
  min-width: 24px;
}

.field-placeholder {
  background: #e8eaf6;
  border: 2px dashed #3f51b5;
  border-radius: 4px;
  min-height: 40px;
}

.empty-state {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 200px;
  color: #9e9e9e;
}

/* CDK Drag & Drop animations */
.cdk-drag-preview {
  box-sizing: border-box;
  border-radius: 4px;
  box-shadow:
    0 5px 5px -3px rgba(0, 0, 0, 0.2),
    0 8px 10px 1px rgba(0, 0, 0, 0.14),
    0 3px 14px 2px rgba(0, 0, 0, 0.12);
}

.cdk-drag-animating {
  transition: transform 250ms cubic-bezier(0, 0, 0.2, 1);
}

.field-list.cdk-drop-list-dragging .field-item:not(.cdk-drag-placeholder) {
  transition: transform 250ms cubic-bezier(0, 0, 0.2, 1);
}
```

---

### 5.4 Step 3: Filter Builder

```typescript
// step-filters/step-filters.component.ts

import { Component, Input, Output, EventEmitter } from '@angular/core';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { MatIconButton } from '@angular/material/button';
import { ExportField, ExportFilter, FilterOperator } from '../../models/data-export.model';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-step-filters',
  templateUrl: './step-filters.component.html',
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    FaIconComponent,
    MatIconButton
  ]
})
export class StepFiltersComponent {
  @Input() availableFields: ExportField[] = [];
  @Input() filters: ExportFilter[] = [];
  @Output() filtersChange = new EventEmitter<ExportFilter[]>();

  operatorOptions = Object.values(FilterOperator);
  conjunctionOptions: Array<'AND' | 'OR'> = [
    'AND',
    'OR'
  ];

  /**
   * Returns operators valid for the given field type.
   */
  getOperatorsForType(fieldType: string): FilterOperator[] {
    switch (fieldType) {
      case 'number':
      case 'date':
        return [
          FilterOperator.EQUALS,
          FilterOperator.GREATER_THAN,
          FilterOperator.LESS_THAN,
          FilterOperator.BETWEEN
        ];
      case 'string':
        return [
          FilterOperator.EQUALS,
          FilterOperator.CONTAINS,
          FilterOperator.IN
        ];
      case 'enum':
        return [
          FilterOperator.EQUALS,
          FilterOperator.IN
        ];
      case 'boolean':
        return [FilterOperator.EQUALS];
      default:
        return this.operatorOptions;
    }
  }

  getFieldByName(name: string): ExportField | undefined {
    return this.availableFields.find((f) => f.name === name);
  }

  addFilter() {
    const newFilter: ExportFilter = {
      field: '',
      operator: FilterOperator.EQUALS,
      value: '',
      conjunction: this.filters.length > 0 ? 'AND' : 'AND'
    };
    this.filters = [
      ...this.filters,
      newFilter
    ];
    this.filtersChange.emit(this.filters);
  }

  removeFilter(index: number) {
    this.filters = this.filters.filter((_, i) => i !== index);
    this.filtersChange.emit(this.filters);
  }

  onFilterChanged() {
    this.filtersChange.emit([...this.filters]);
  }
}
```

**Template: `step-filters.component.html`**

```html
<div class="layout-column">
  <h4 class="mat-h4">{{ 'labels.heading.Export Filters' | translate }}</h4>
  <p class="mat-body-1">{{ 'labels.text.Add filters to narrow down exported data' | translate }}</p>

  @for (filter of filters; track $index; let i = $index) {
  <div class="filter-row layout-row-wrap gap-2px align-center margin-b">
    @if (i > 0) {
    <mat-form-field class="flex-10">
      <mat-select [(ngModel)]="filter.conjunction" (ngModelChange)="onFilterChanged()">
        @for (conj of conjunctionOptions; track conj) {
        <mat-option [value]="conj">{{ conj }}</mat-option>
        }
      </mat-select>
    </mat-form-field>
    }

    <mat-form-field class="flex-25">
      <mat-label>{{ 'labels.inputs.Field' | translate }}</mat-label>
      <mat-select [(ngModel)]="filter.field" (ngModelChange)="onFilterChanged()">
        @for (field of availableFields; track field.name) {
        <mat-option [value]="field.name">{{ field.label }}</mat-option>
        }
      </mat-select>
    </mat-form-field>

    <mat-form-field class="flex-20">
      <mat-label>{{ 'labels.inputs.Operator' | translate }}</mat-label>
      <mat-select [(ngModel)]="filter.operator" (ngModelChange)="onFilterChanged()">
        @for (op of getOperatorsForType(getFieldByName(filter.field)?.type || 'string'); track op) {
        <mat-option [value]="op">{{ op }}</mat-option>
        }
      </mat-select>
    </mat-form-field>

    <!-- Dynamic value input based on field type -->
    @if (getFieldByName(filter.field)?.type === 'date') {
    <mat-form-field class="flex-25" (click)="filterDatePicker.open()">
      <mat-label>{{ 'labels.inputs.Value' | translate }}</mat-label>
      <input matInput [(ngModel)]="filter.value" [matDatepicker]="filterDatePicker" (ngModelChange)="onFilterChanged()" />
      <mat-datepicker-toggle matSuffix [for]="filterDatePicker"></mat-datepicker-toggle>
      <mat-datepicker #filterDatePicker></mat-datepicker>
    </mat-form-field>
    } @else if (getFieldByName(filter.field)?.type === 'enum') {
    <mat-form-field class="flex-25">
      <mat-label>{{ 'labels.inputs.Value' | translate }}</mat-label>
      <mat-select [(ngModel)]="filter.value" [multiple]="filter.operator === 'in'" (ngModelChange)="onFilterChanged()">
        @for (val of getFieldByName(filter.field)?.enumValues || []; track val) {
        <mat-option [value]="val">{{ val }}</mat-option>
        }
      </mat-select>
    </mat-form-field>
    } @else if (getFieldByName(filter.field)?.type === 'boolean') {
    <mat-form-field class="flex-25">
      <mat-label>{{ 'labels.inputs.Value' | translate }}</mat-label>
      <mat-select [(ngModel)]="filter.value" (ngModelChange)="onFilterChanged()">
        <mat-option [value]="true">True</mat-option>
        <mat-option [value]="false">False</mat-option>
      </mat-select>
    </mat-form-field>
    } @else if (getFieldByName(filter.field)?.type === 'number') {
    <mat-form-field class="flex-25">
      <mat-label>{{ 'labels.inputs.Value' | translate }}</mat-label>
      <input type="number" matInput [(ngModel)]="filter.value" (ngModelChange)="onFilterChanged()" />
    </mat-form-field>
    } @else {
    <mat-form-field class="flex-25">
      <mat-label>{{ 'labels.inputs.Value' | translate }}</mat-label>
      <input matInput [(ngModel)]="filter.value" (ngModelChange)="onFilterChanged()" />
    </mat-form-field>
    } @if (filter.operator === 'between') {
    <mat-form-field class="flex-20">
      <mat-label>{{ 'labels.inputs.To' | translate }}</mat-label>
      <input matInput [(ngModel)]="filter.valueTo" [type]="getFieldByName(filter.field)?.type === 'number' ? 'number' : 'text'" (ngModelChange)="onFilterChanged()" />
    </mat-form-field>
    }

    <button mat-icon-button color="warn" (click)="removeFilter(i)">
      <fa-icon icon="trash"></fa-icon>
    </button>
  </div>
  }

  <button mat-raised-button type="button" (click)="addFilter()">
    <fa-icon icon="plus" class="m-r-10"></fa-icon>
    {{ 'labels.buttons.Add Filter' | translate }}
  </button>
</div>
```

---

### 5.5 Step 4: Preview and Save

```typescript
// step-preview/step-preview.component.ts

import { Component, Input, Output, EventEmitter, OnInit, inject } from '@angular/core';
import { MatTableDataSource } from '@angular/material/table';
import { MatTable, MatColumnDef, MatHeaderCellDef, MatHeaderCell, MatCellDef, MatCell, MatHeaderRowDef, MatHeaderRow, MatRowDef, MatRow } from '@angular/material/table';
import { MatRadioGroup, MatRadioButton } from '@angular/material/radio';
import { DataExportsService } from '../../data-exports.service';
import { ExportField, ExportFilter, ExportFormat, BaseEntity } from '../../models/data-export.model';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-step-preview',
  templateUrl: './step-preview.component.html',
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatRadioGroup,
    MatRadioButton,
    MatTable,
    MatColumnDef,
    MatHeaderCellDef,
    MatHeaderCell,
    MatCellDef,
    MatCell,
    MatHeaderRowDef,
    MatHeaderRow,
    MatRowDef,
    MatRow
  ]
})
export class StepPreviewComponent implements OnInit {
  private exportService = inject(DataExportsService);

  @Input() selectedFields: ExportField[] = [];
  @Input() filters: ExportFilter[] = [];
  @Input() baseEntity: BaseEntity | null = null;
  @Input() exportFormat: ExportFormat = ExportFormat.CSV;
  @Output() exportFormatChange = new EventEmitter<ExportFormat>();

  formatOptions = Object.values(ExportFormat);
  previewData: any[] = [];
  previewColumns: string[] = [];
  dataSource: MatTableDataSource<any>;
  isLoading = false;

  ngOnInit() {
    this.loadPreview();
  }

  loadPreview() {
    if (!this.baseEntity || this.selectedFields.length === 0) return;

    this.isLoading = true;
    this.previewColumns = this.selectedFields.map((f) => f.name);

    // Fetch first 10 rows from entity endpoint
    this.exportService.getPreviewData(this.baseEntity, this.filters, 10).subscribe(
      (data: any[]) => {
        this.previewData = data;
        this.dataSource = new MatTableDataSource(data);
        this.isLoading = false;
      },
      () => {
        this.isLoading = false;
      }
    );
  }

  onFormatChanged(format: ExportFormat) {
    this.exportFormat = format;
    this.exportFormatChange.emit(format);
  }
}
```

**Template: `step-preview.component.html`**

```html
<div class="layout-column">
  <h4 class="mat-h4">{{ 'labels.heading.Export Preview' | translate }}</h4>

  <!-- Format selection -->
  <div class="margin-b">
    <h5 class="mat-h5">{{ 'labels.inputs.Export Format' | translate }}</h5>
    <mat-radio-group [value]="exportFormat" (change)="onFormatChanged($event.value)">
      @for (format of formatOptions; track format) {
      <mat-radio-button [value]="format" class="m-r-20"> {{ format | uppercase }} </mat-radio-button>
      }
    </mat-radio-group>
  </div>

  <!-- Summary -->
  <div class="export-summary margin-b">
    <p><strong>{{ 'labels.inputs.Fields' | translate }}:</strong> {{ selectedFields.length }}</p>
    <p><strong>{{ 'labels.inputs.Filters' | translate }}:</strong> {{ filters.length }}</p>
    <p><strong>{{ 'labels.inputs.Format' | translate }}:</strong> {{ exportFormat | uppercase }}</p>
  </div>

  <!-- Preview table -->
  <h5 class="mat-h5">{{ 'labels.heading.Preview (first 10 rows)' | translate }}</h5>

  @if (isLoading) {
  <p>{{ 'labels.text.Loading preview...' | translate }}</p>
  } @if (!isLoading && previewData.length > 0) {
  <div class="table-container">
    <table mat-table [dataSource]="dataSource" class="mat-elevation-z1">
      @for (field of selectedFields; track field.name) {
      <ng-container [matColumnDef]="field.name">
        <th mat-header-cell *matHeaderCellDef>{{ field.label }}</th>
        <td mat-cell *matCellDef="let row">{{ row[field.name] }}</td>
      </ng-container>
      }

      <tr mat-header-row *matHeaderRowDef="previewColumns"></tr>
      <tr mat-row *matRowDef="let row; columns: previewColumns"></tr>
    </table>
  </div>
  } @if (!isLoading && previewData.length === 0) {
  <p class="no-data">{{ 'labels.text.No data available for preview' | translate }}</p>
  }
</div>
```

Add this method to the service:

```typescript
// Add to data-exports.service.ts

getPreviewData(entity: BaseEntity, filters: ExportFilter[], limit: number): Observable<any[]> {
  let params = new HttpParams().set('limit', limit.toString());
  for (const filter of filters) {
    if (filter.field && filter.value) {
      params = params.set(filter.field, filter.value.toString());
    }
  }
  const endpoint = this.getEntityListEndpoint(entity);
  return this.http.get<any>(endpoint, { params }).pipe(
    map(response => response.pageItems || response || [])
  );
}
```

---

## 6. Fineract API Reference

| Endpoint                                  | Method   | Purpose                                              |
| ----------------------------------------- | -------- | ---------------------------------------------------- |
| `/datatables?apptable={table}`            | GET      | Get registered datatables for an entity              |
| `/clients/template`                       | GET      | Client field metadata                                |
| `/loans/template?templateType=individual` | GET      | Loan field metadata                                  |
| `/savingsaccounts/template`               | GET      | Savings field metadata                               |
| `/clients?limit=10&...`                   | GET      | Client list with filters (preview)                   |
| `/loans?limit=10&...`                     | GET      | Loan list with filters (preview)                     |
| `/datatables/data_exports`                | POST/GET | CRUD for saved export definitions (custom datatable) |

---

## 7. Navigation Integration

Add to the sidebar navigation:

```typescript
{
  name: 'Data Exports',
  icon: 'file-export',
  route: '/data-exports'
}
```

Add the lazy-loaded route in `src/app/app-routing.module.ts`:

```typescript
{
  path: 'data-exports',
  loadChildren: () => import('./data-exports/data-exports.module').then(m => m.DataExportsModule),
  data: { title: 'Data Exports', breadcrumb: 'Data Exports' }
}
```
