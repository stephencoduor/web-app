# TECH-003: Client Detail New Tabs Specification

**Status:** Draft
**Priority:** Medium-High
**Estimated Effort:** 8-12 days
**Dependencies:** TECH-002 (StatusBadgeComponent, MsaccoDataTableComponent)

---

## Overview

The client detail view at `src/app/clients/clients-view/` currently uses `mat-tab-nav-bar` with tabs like General, Address, Family Members, Identities, Documents, Notes, and dynamic datatable tabs. This specification adds 7 new tabs to the client detail view.

### Current Tab Architecture

The `ClientsViewComponent` renders tabs using `MatTabNav` / `MatTabLink` with `routerLinkActive`. Each tab is a child route of `/clients/:clientId/` defined in `src/app/clients/clients-routing.module.ts`. The tab content is rendered via `<router-outlet>` inside `<mat-tab-nav-panel>`.

### Existing Tabs (for reference)

| Tab                  | Route                       | Component                   |
| -------------------- | --------------------------- | --------------------------- |
| General              | `general`                   | `GeneralTabComponent`       |
| Address              | `address`                   | `AddressTabComponent`       |
| Family Members       | `family-members`            | `FamilyMembersTabComponent` |
| Identities           | `identities`                | `IdentitiesTabComponent`    |
| Documents            | `documents`                 | `DocumentsTabComponent`     |
| Notes                | `notes`                     | `NotesTabComponent`         |
| Personal Data        | `personal-data`             | `PersonalDataTabComponent`  |
| Datatables (dynamic) | `datatables/:datatableName` | `DatatableTabComponent`     |

---

## 1. CRB Account Tab

**Directory:** `src/app/clients/clients-view/crb-account-tab/`
**Route:** `/clients/:clientId/crb-account`

### Purpose

Displays Credit Reference Bureau (CRB) data for the client, including credit score, report history, and the ability to request new credit checks.

### Fineract API

```
GET  /fineract-provider/api/v1/clients/{clientId}/creditChecks
POST /fineract-provider/api/v1/clients/{clientId}/creditChecks
```

If the backend does not have a dedicated CRB endpoint, this can be backed by a custom datatable `m_client_crb_data` or a plugin endpoint. The service layer abstracts this.

### Service

```typescript
// src/app/clients/clients-view/crb-account-tab/crb.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { SettingsService } from 'app/settings/settings.service';

export interface CreditCheck {
  id: number;
  bureauName: string;
  creditScore: number;
  reportDate: string;
  status: string; // 'Completed' | 'Pending' | 'Failed'
  reportSummary?: string;
  requestedBy?: string;
  requestedDate?: string;
}

@Injectable({ providedIn: 'root' })
export class CRBService {
  private http = inject(HttpClient);
  private settingsService = inject(SettingsService);

  private get apiBase(): string {
    return `${this.settingsService.serverUrl}/api/v1`;
  }

  getCreditChecks(clientId: number): Observable<CreditCheck[]> {
    return this.http.get<CreditCheck[]>(`${this.apiBase}/clients/${clientId}/creditChecks`);
  }

  requestCreditCheck(clientId: number, bureauName: string): Observable<any> {
    return this.http.post(`${this.apiBase}/clients/${clientId}/creditChecks`, { bureauName });
  }

  getCreditCheckDetail(clientId: number, checkId: number): Observable<CreditCheck> {
    return this.http.get<CreditCheck>(`${this.apiBase}/clients/${clientId}/creditChecks/${checkId}`);
  }
}
```

### Resolver

```typescript
// src/app/clients/clients-view/crb-account-tab/crb-account-tab.resolver.ts

import { Injectable, inject } from '@angular/core';
import { Resolve, ActivatedRouteSnapshot } from '@angular/router';
import { Observable } from 'rxjs';
import { CRBService, CreditCheck } from './crb.service';

@Injectable({ providedIn: 'root' })
export class CRBAccountResolver implements Resolve<CreditCheck[]> {
  private crbService = inject(CRBService);

  resolve(route: ActivatedRouteSnapshot): Observable<CreditCheck[]> {
    const clientId = Number(route.parent.paramMap.get('clientId'));
    return this.crbService.getCreditChecks(clientId);
  }
}
```

### Component

```typescript
// src/app/clients/clients-view/crb-account-tab/crb-account-tab.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { MsaccoDataTableComponent } from 'app/shared/msacco-data-table/msacco-data-table.component';
import { StatusBadgeComponent } from 'app/shared/status-badge/status-badge.component';
import { CRBService, CreditCheck } from './crb.service';
import { ColumnDef } from 'app/shared/msacco-data-table/msacco-data-table.interfaces';

@Component({
  selector: 'mifosx-crb-account-tab',
  templateUrl: './crb-account-tab.component.html',
  styleUrls: ['./crb-account-tab.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MsaccoDataTableComponent,
    StatusBadgeComponent
  ]
})
export class CRBAccountTabComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private crbService = inject(CRBService);
  private dialog = inject(MatDialog);

  clientId: number;
  creditChecks: CreditCheck[] = [];
  latestScore: number | null = null;
  latestBureau: string = '';
  latestDate: string = '';

  columns: ColumnDef[] = [
    { name: 'bureauName', header: 'Bureau Name', cell: (row) => row.bureauName },
    { name: 'creditScore', header: 'Credit Score', cell: (row) => row.creditScore?.toString() },
    { name: 'reportDate', header: 'Report Date', cell: (row) => row.reportDate, type: 'date' },
    { name: 'status', header: 'Status', cell: (row) => row.status, type: 'status' },
    { name: 'requestedBy', header: 'Requested By', cell: (row) => row.requestedBy || '-' }
  ];

  ngOnInit(): void {
    this.clientId = Number(this.route.parent.snapshot.paramMap.get('clientId'));
    this.route.data.subscribe((data: { creditChecks: CreditCheck[] }) => {
      this.creditChecks = data.creditChecks || [];
      this.updateLatestScore();
    });
  }

  private updateLatestScore(): void {
    if (this.creditChecks.length > 0) {
      const latest = this.creditChecks.filter((c) => c.status === 'Completed').sort((a, b) => new Date(b.reportDate).getTime() - new Date(a.reportDate).getTime())[0];
      if (latest) {
        this.latestScore = latest.creditScore;
        this.latestBureau = latest.bureauName;
        this.latestDate = latest.reportDate;
      }
    }
  }

  requestNewCheck(): void {
    // Open a dialog to select bureau, then call service
    this.crbService.requestCreditCheck(this.clientId, 'TransUnion').subscribe(() => {
      // Refresh data
      this.crbService.getCreditChecks(this.clientId).subscribe((checks) => {
        this.creditChecks = checks;
        this.updateLatestScore();
      });
    });
  }
}
```

### Template

```html
<!-- src/app/clients/clients-view/crb-account-tab/crb-account-tab.component.html -->

<div class="crb-account-tab">
  <!-- Credit Score Summary Card -->
  <div class="crb-score-card">
    <div class="score-display">
      @if (latestScore !== null) {
      <div class="score-value" [class.good]="latestScore >= 700" [class.fair]="latestScore >= 500 && latestScore < 700" [class.poor]="latestScore < 500">{{ latestScore }}</div>
      <div class="score-meta">
        <span class="score-label">Credit Score</span>
        <span class="score-bureau">{{ latestBureau }}</span>
        <span class="score-date">{{ latestDate }}</span>
      </div>
      } @else {
      <div class="no-score">
        <mat-icon>credit_score</mat-icon>
        <span>No credit check data available</span>
      </div>
      }
    </div>
    <button mat-flat-button color="primary" (click)="requestNewCheck()" *mifosxHasPermission="'CREATE_CREDITCHECK'">
      <mat-icon>add</mat-icon>
      Request New Check
    </button>
  </div>

  <!-- Credit Check History Table -->
  <msacco-data-table [columns]="columns" [dataSource]="creditChecks" [searchEnabled]="false" [exportEnabled]="true" [exportConfig]="{ filenamePrefix: 'crb-history' }"></msacco-data-table>
</div>
```

### Styles

```scss
.crb-account-tab {
  padding: 16px;
}

.crb-score-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 24px;
  margin-bottom: 16px;
  background: #ffffff;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
}

.score-display {
  display: flex;
  align-items: center;
  gap: 16px;
}

.score-value {
  font-size: 48px;
  font-weight: 700;
  line-height: 1;

  &.good {
    color: #28a745;
  }
  &.fair {
    color: #ffc107;
  }
  &.poor {
    color: #dc3545;
  }
}

.score-meta {
  display: flex;
  flex-direction: column;
  gap: 2px;

  .score-label {
    font-size: 14px;
    font-weight: 600;
    color: #333;
  }
  .score-bureau {
    font-size: 13px;
    color: #666;
  }
  .score-date {
    font-size: 12px;
    color: #999;
  }
}

.no-score {
  display: flex;
  align-items: center;
  gap: 8px;
  color: #999;
  font-size: 14px;
}
```

---

## 2. Business Details Tab

**Directory:** `src/app/clients/clients-view/business-details-tab/`
**Route:** `/clients/:clientId/business-details`

### Purpose

Displays and manages business information associated with a client. Supports view, add, edit, and delete operations.

### Fineract API

```
GET    /fineract-provider/api/v1/datatables/m_client_business_details/{clientId}
POST   /fineract-provider/api/v1/datatables/m_client_business_details/{clientId}
PUT    /fineract-provider/api/v1/datatables/m_client_business_details/{clientId}/{entryId}
DELETE /fineract-provider/api/v1/datatables/m_client_business_details/{clientId}/{entryId}
```

This uses the Fineract custom datatable pattern. The datatable `m_client_business_details` must be registered on the backend with the appropriate columns.

### Service

```typescript
// src/app/clients/clients-view/business-details-tab/business-details.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { SettingsService } from 'app/settings/settings.service';

export interface BusinessDetail {
  id?: number;
  businessName: string;
  businessType: string;
  startDate: string;
  address: string;
  postalCode: string;
  county: string;
  description: string;
}

@Injectable({ providedIn: 'root' })
export class BusinessDetailsService {
  private http = inject(HttpClient);
  private settingsService = inject(SettingsService);

  private readonly DATATABLE_NAME = 'm_client_business_details';

  private get apiBase(): string {
    return `${this.settingsService.serverUrl}/api/v1`;
  }

  getBusinessDetails(clientId: number): Observable<any> {
    return this.http.get(`${this.apiBase}/datatables/${this.DATATABLE_NAME}/${clientId}`, {
      params: new HttpParams().set('genericResultSet', 'false')
    });
  }

  addBusinessDetail(clientId: number, data: BusinessDetail): Observable<any> {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    return this.http.post(`${this.apiBase}/datatables/${this.DATATABLE_NAME}/${clientId}`, {
      ...data,
      locale,
      dateFormat
    });
  }

  updateBusinessDetail(clientId: number, entryId: number, data: BusinessDetail): Observable<any> {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    return this.http.put(`${this.apiBase}/datatables/${this.DATATABLE_NAME}/${clientId}/${entryId}`, {
      ...data,
      locale,
      dateFormat
    });
  }

  deleteBusinessDetail(clientId: number, entryId: number): Observable<any> {
    return this.http.delete(`${this.apiBase}/datatables/${this.DATATABLE_NAME}/${clientId}/${entryId}`);
  }
}
```

### Component

```typescript
// src/app/clients/clients-view/business-details-tab/business-details-tab.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { UntypedFormBuilder, UntypedFormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { BusinessDetailsService, BusinessDetail } from './business-details.service';
import { Dates } from 'app/core/utils/dates';
import { SettingsService } from 'app/settings/settings.service';

@Component({
  selector: 'mifosx-business-details-tab',
  templateUrl: './business-details-tab.component.html',
  styleUrls: ['./business-details-tab.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    ReactiveFormsModule
  ]
})
export class BusinessDetailsTabComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private fb = inject(UntypedFormBuilder);
  private businessService = inject(BusinessDetailsService);
  private dateUtils = inject(Dates);
  private settingsService = inject(SettingsService);

  clientId: number;
  businessDetails: BusinessDetail[] = [];
  editingDetail: BusinessDetail | null = null;
  isFormVisible: boolean = false;
  businessForm: UntypedFormGroup;

  businessTypes: string[] = [
    'Retail',
    'Agriculture',
    'Manufacturing',
    'Services',
    'Transport',
    'Construction',
    'Education',
    'Other'
  ];

  ngOnInit(): void {
    this.clientId = Number(this.route.parent.snapshot.paramMap.get('clientId'));
    this.initForm();
    this.loadData();
  }

  private initForm(): void {
    this.businessForm = this.fb.group({
      businessName: [
        '',
        Validators.required
      ],
      businessType: [
        '',
        Validators.required
      ],
      startDate: [''],
      address: [''],
      postalCode: [''],
      county: [''],
      description: ['']
    });
  }

  private loadData(): void {
    this.businessService.getBusinessDetails(this.clientId).subscribe((data: any) => {
      this.businessDetails = Array.isArray(data) ? data : [];
    });
  }

  showAddForm(): void {
    this.editingDetail = null;
    this.businessForm.reset();
    this.isFormVisible = true;
  }

  editDetail(detail: BusinessDetail): void {
    this.editingDetail = detail;
    this.businessForm.patchValue(detail);
    this.isFormVisible = true;
  }

  cancelForm(): void {
    this.isFormVisible = false;
    this.editingDetail = null;
    this.businessForm.reset();
  }

  saveDetail(): void {
    if (this.businessForm.invalid) return;

    const formData = { ...this.businessForm.value };
    const dateFormat = this.settingsService.dateFormat;
    if (formData.startDate instanceof Date) {
      formData.startDate = this.dateUtils.formatDate(formData.startDate, dateFormat);
    }

    if (this.editingDetail?.id) {
      this.businessService.updateBusinessDetail(this.clientId, this.editingDetail.id, formData).subscribe(() => {
        this.cancelForm();
        this.loadData();
      });
    } else {
      this.businessService.addBusinessDetail(this.clientId, formData).subscribe(() => {
        this.cancelForm();
        this.loadData();
      });
    }
  }

  deleteDetail(detail: BusinessDetail): void {
    if (detail.id && confirm('Are you sure you want to delete this business record?')) {
      this.businessService.deleteBusinessDetail(this.clientId, detail.id).subscribe(() => this.loadData());
    }
  }
}
```

### Template

```html
<!-- src/app/clients/clients-view/business-details-tab/business-details-tab.component.html -->

<div class="business-details-tab">
  <div class="tab-header">
    <h3>{{ 'labels.headings.Business Details' | translate }}</h3>
    <button mat-flat-button color="primary" (click)="showAddForm()" *mifosxHasPermission="'CREATE_DATATABLE'"><mat-icon>add</mat-icon> Add Business</button>
  </div>

  <!-- Business Form (Add/Edit) -->
  @if (isFormVisible) {
  <mat-card class="business-form-card">
    <mat-card-content>
      <form [formGroup]="businessForm" class="business-form">
        <div class="form-row">
          <mat-form-field appearance="outline">
            <mat-label>Business Name *</mat-label>
            <input matInput formControlName="businessName" />
          </mat-form-field>
          <mat-form-field appearance="outline">
            <mat-label>Business Type *</mat-label>
            <mat-select formControlName="businessType">
              @for (type of businessTypes; track type) {
              <mat-option [value]="type">{{ type }}</mat-option>
              }
            </mat-select>
          </mat-form-field>
        </div>
        <div class="form-row">
          <mat-form-field appearance="outline">
            <mat-label>Start Date</mat-label>
            <input matInput [matDatepicker]="startPicker" formControlName="startDate" />
            <mat-datepicker-toggle matSuffix [for]="startPicker"></mat-datepicker-toggle>
            <mat-datepicker #startPicker></mat-datepicker>
          </mat-form-field>
          <mat-form-field appearance="outline">
            <mat-label>County</mat-label>
            <input matInput formControlName="county" />
          </mat-form-field>
        </div>
        <div class="form-row">
          <mat-form-field appearance="outline">
            <mat-label>Address</mat-label>
            <input matInput formControlName="address" />
          </mat-form-field>
          <mat-form-field appearance="outline">
            <mat-label>Postal Code</mat-label>
            <input matInput formControlName="postalCode" />
          </mat-form-field>
        </div>
        <mat-form-field appearance="outline" class="full-width">
          <mat-label>Description</mat-label>
          <textarea matInput formControlName="description" rows="3"></textarea>
        </mat-form-field>
        <div class="form-actions">
          <button mat-stroked-button type="button" (click)="cancelForm()">Cancel</button>
          <button mat-flat-button color="primary" type="button" (click)="saveDetail()" [disabled]="businessForm.invalid">{{ editingDetail ? 'Update' : 'Save' }}</button>
        </div>
      </form>
    </mat-card-content>
  </mat-card>
  }

  <!-- Business Details List -->
  @if (businessDetails.length > 0) { @for (detail of businessDetails; track detail.id) {
  <mat-card class="business-card">
    <mat-card-content>
      <div class="business-info">
        <div class="info-row">
          <span class="info-label">Business Name:</span>
          <span class="info-value">{{ detail.businessName }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">Type:</span>
          <span class="info-value">{{ detail.businessType }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">Start Date:</span>
          <span class="info-value">{{ detail.startDate }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">Address:</span>
          <span class="info-value">{{ detail.address }} {{ detail.postalCode }}</span>
        </div>
        <div class="info-row">
          <span class="info-label">County:</span>
          <span class="info-value">{{ detail.county }}</span>
        </div>
        @if (detail.description) {
        <div class="info-row">
          <span class="info-label">Description:</span>
          <span class="info-value">{{ detail.description }}</span>
        </div>
        }
      </div>
      <div class="business-actions">
        <button mat-icon-button (click)="editDetail(detail)" *mifosxHasPermission="'UPDATE_DATATABLE'">
          <mat-icon>edit</mat-icon>
        </button>
        <button mat-icon-button color="warn" (click)="deleteDetail(detail)" *mifosxHasPermission="'DELETE_DATATABLE'">
          <mat-icon>delete</mat-icon>
        </button>
      </div>
    </mat-card-content>
  </mat-card>
  } } @else if (!isFormVisible) {
  <div class="empty-state">
    <mat-icon>store</mat-icon>
    <p>No business details recorded for this client.</p>
  </div>
  }
</div>
```

---

## 3. PPI Tab

**Directory:** `src/app/clients/clients-view/ppi-tab/`
**Route:** `/clients/:clientId/ppi`

### Purpose

Displays Progress out of Poverty Index (PPI) survey results for the client, with a score trend chart.

### Fineract API

```
GET /fineract-provider/api/v1/surveys/scorecards/{surveyId}/clients/{clientId}
GET /fineract-provider/api/v1/surveys
```

### Component Structure

```typescript
// src/app/clients/clients-view/ppi-tab/ppi-tab.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { MsaccoDataTableComponent } from 'app/shared/msacco-data-table/msacco-data-table.component';
import { ColumnDef } from 'app/shared/msacco-data-table/msacco-data-table.interfaces';

// Chart.js imports (lazy loaded)
import { BaseChartDirective } from 'ng2-charts';
import { ChartConfiguration } from 'chart.js';

export interface PPISurvey {
  id: number;
  surveyName: string;
  dateTaken: string;
  score: number;
  povertyLikelihood: string;
  completedBy: string;
}

@Component({
  selector: 'mifosx-ppi-tab',
  templateUrl: './ppi-tab.component.html',
  styleUrls: ['./ppi-tab.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MsaccoDataTableComponent,
    BaseChartDirective
  ]
})
export class PPITabComponent implements OnInit {
  private route = inject(ActivatedRoute);

  clientId: number;
  surveys: PPISurvey[] = [];

  columns: ColumnDef[] = [
    { name: 'surveyName', header: 'Survey', cell: (row) => row.surveyName },
    { name: 'dateTaken', header: 'Date', cell: (row) => row.dateTaken, type: 'date' },
    { name: 'score', header: 'Score', cell: (row) => row.score?.toString() },
    { name: 'povertyLikelihood', header: 'Poverty Likelihood', cell: (row) => row.povertyLikelihood },
    { name: 'completedBy', header: 'Completed By', cell: (row) => row.completedBy }
  ];

  // Chart.js configuration for PPI trend
  chartData: ChartConfiguration<'line'>['data'] = {
    labels: [],
    datasets: [
      {
        label: 'PPI Score',
        data: [],
        borderColor: '#1074b9',
        backgroundColor: 'rgba(16, 116, 185, 0.1)',
        fill: true,
        tension: 0.3
      }
    ]
  };

  chartOptions: ChartConfiguration<'line'>['options'] = {
    responsive: true,
    plugins: {
      legend: { display: false },
      title: { display: true, text: 'PPI Score Trend' }
    },
    scales: {
      y: { beginAtZero: true, max: 100, title: { display: true, text: 'Score' } },
      x: { title: { display: true, text: 'Date' } }
    }
  };

  ngOnInit(): void {
    this.clientId = Number(this.route.parent.snapshot.paramMap.get('clientId'));
    this.route.data.subscribe((data: { ppiSurveys: PPISurvey[] }) => {
      this.surveys = data.ppiSurveys || [];
      this.buildChart();
    });
  }

  private buildChart(): void {
    const sorted = [...this.surveys].sort((a, b) => new Date(a.dateTaken).getTime() - new Date(b.dateTaken).getTime());
    this.chartData.labels = sorted.map((s) => s.dateTaken);
    this.chartData.datasets[0].data = sorted.map((s) => s.score);
  }
}
```

### Template

```html
<div class="ppi-tab">
  <!-- PPI Score Trend Chart -->
  @if (surveys.length > 1) {
  <div class="ppi-chart-card">
    <canvas baseChart [data]="chartData" [options]="chartOptions" type="line"></canvas>
  </div>
  }

  <!-- Survey History Table -->
  <msacco-data-table [columns]="columns" [dataSource]="surveys" [searchEnabled]="false" [exportEnabled]="true" [exportConfig]="{ filenamePrefix: 'ppi-surveys', title: 'PPI Survey History' }"></msacco-data-table>
</div>
```

---

## 4. Financial Statements Tab

**Directory:** `src/app/clients/clients-view/financial-statements-tab/`
**Route:** `/clients/:clientId/financial-statements`

### Purpose

Records income and expense data for the client, with automatic net income calculation.

### Fineract API

Uses custom datatables:

```
GET/POST/PUT/DELETE /fineract-provider/api/v1/datatables/m_client_income/{clientId}
GET/POST/PUT/DELETE /fineract-provider/api/v1/datatables/m_client_expenses/{clientId}
```

### Service

```typescript
// src/app/clients/clients-view/financial-statements-tab/financial-statements.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable, forkJoin } from 'rxjs';
import { map } from 'rxjs/operators';
import { SettingsService } from 'app/settings/settings.service';

export interface FinancialItem {
  id?: number;
  source: string; // income source or expense category
  amount: number;
  frequency: string; // 'Monthly' | 'Weekly' | 'Annually' | 'One-time'
}

export interface FinancialStatement {
  incomeItems: FinancialItem[];
  expenseItems: FinancialItem[];
  totalIncome: number;
  totalExpenses: number;
  netIncome: number;
}

@Injectable({ providedIn: 'root' })
export class FinancialStatementsService {
  private http = inject(HttpClient);
  private settingsService = inject(SettingsService);

  private get apiBase(): string {
    return `${this.settingsService.serverUrl}/api/v1`;
  }

  getFinancialStatement(clientId: number): Observable<FinancialStatement> {
    const params = new HttpParams().set('genericResultSet', 'false');
    return forkJoin({
      income: this.http.get<FinancialItem[]>(`${this.apiBase}/datatables/m_client_income/${clientId}`, { params }),
      expenses: this.http.get<FinancialItem[]>(`${this.apiBase}/datatables/m_client_expenses/${clientId}`, { params })
    }).pipe(
      map(({ income, expenses }) => {
        const incomeItems = Array.isArray(income) ? income : [];
        const expenseItems = Array.isArray(expenses) ? expenses : [];
        const totalIncome = this.calcMonthlyTotal(incomeItems);
        const totalExpenses = this.calcMonthlyTotal(expenseItems);
        return { incomeItems, expenseItems, totalIncome, totalExpenses, netIncome: totalIncome - totalExpenses };
      })
    );
  }

  addIncomeItem(clientId: number, item: FinancialItem): Observable<any> {
    const locale = this.settingsService.language.code;
    return this.http.post(`${this.apiBase}/datatables/m_client_income/${clientId}`, { ...item, locale });
  }

  addExpenseItem(clientId: number, item: FinancialItem): Observable<any> {
    const locale = this.settingsService.language.code;
    return this.http.post(`${this.apiBase}/datatables/m_client_expenses/${clientId}`, { ...item, locale });
  }

  deleteIncomeItem(clientId: number, itemId: number): Observable<any> {
    return this.http.delete(`${this.apiBase}/datatables/m_client_income/${clientId}/${itemId}`);
  }

  deleteExpenseItem(clientId: number, itemId: number): Observable<any> {
    return this.http.delete(`${this.apiBase}/datatables/m_client_expenses/${clientId}/${itemId}`);
  }

  /** Normalize all frequencies to a monthly equivalent */
  private calcMonthlyTotal(items: FinancialItem[]): number {
    return items.reduce((sum, item) => {
      switch (item.frequency) {
        case 'Weekly':
          return sum + item.amount * 4.33;
        case 'Annually':
          return sum + item.amount / 12;
        case 'One-time':
          return sum; // excluded from recurring total
        case 'Monthly':
        default:
          return sum + item.amount;
      }
    }, 0);
  }
}
```

### Component

```typescript
// src/app/clients/clients-view/financial-statements-tab/financial-statements-tab.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { UntypedFormBuilder, UntypedFormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { FinancialStatementsService, FinancialStatement, FinancialItem } from './financial-statements.service';

@Component({
  selector: 'mifosx-financial-statements-tab',
  templateUrl: './financial-statements-tab.component.html',
  styleUrls: ['./financial-statements-tab.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    ReactiveFormsModule
  ]
})
export class FinancialStatementsTabComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private fb = inject(UntypedFormBuilder);
  private financialService = inject(FinancialStatementsService);

  clientId: number;
  statement: FinancialStatement;
  incomeForm: UntypedFormGroup;
  expenseForm: UntypedFormGroup;
  showIncomeForm = false;
  showExpenseForm = false;

  frequencies = [
    'Monthly',
    'Weekly',
    'Annually',
    'One-time'
  ];
  incomeCategories = [
    'Employment',
    'Business',
    'Agriculture',
    'Rental',
    'Remittance',
    'Other'
  ];
  expenseCategories = [
    'Rent',
    'Food',
    'Transport',
    'Education',
    'Healthcare',
    'Utilities',
    'Loan Repayment',
    'Other'
  ];

  ngOnInit(): void {
    this.clientId = Number(this.route.parent.snapshot.paramMap.get('clientId'));
    this.incomeForm = this.fb.group({
      source: [
        '',
        Validators.required
      ],
      amount: [
        '',
        [
          Validators.required,
          Validators.min(0)
        ]
      ],
      frequency: [
        'Monthly',
        Validators.required
      ]
    });
    this.expenseForm = this.fb.group({
      source: [
        '',
        Validators.required
      ],
      amount: [
        '',
        [
          Validators.required,
          Validators.min(0)
        ]
      ],
      frequency: [
        'Monthly',
        Validators.required
      ]
    });
    this.loadData();
  }

  private loadData(): void {
    this.financialService.getFinancialStatement(this.clientId).subscribe((stmt) => {
      this.statement = stmt;
    });
  }

  addIncome(): void {
    if (this.incomeForm.invalid) return;
    this.financialService.addIncomeItem(this.clientId, this.incomeForm.value).subscribe(() => {
      this.showIncomeForm = false;
      this.incomeForm.reset({ frequency: 'Monthly' });
      this.loadData();
    });
  }

  addExpense(): void {
    if (this.expenseForm.invalid) return;
    this.financialService.addExpenseItem(this.clientId, this.expenseForm.value).subscribe(() => {
      this.showExpenseForm = false;
      this.expenseForm.reset({ frequency: 'Monthly' });
      this.loadData();
    });
  }

  deleteIncome(item: FinancialItem): void {
    if (item.id) {
      this.financialService.deleteIncomeItem(this.clientId, item.id).subscribe(() => this.loadData());
    }
  }

  deleteExpense(item: FinancialItem): void {
    if (item.id) {
      this.financialService.deleteExpenseItem(this.clientId, item.id).subscribe(() => this.loadData());
    }
  }
}
```

### Template

```html
<div class="financial-statements-tab">
  @if (statement) {
  <!-- Net Income Summary -->
  <div class="summary-bar">
    <div class="summary-item income">
      <span class="summary-label">Total Monthly Income</span>
      <span class="summary-value">{{ statement.totalIncome | number:'1.2-2' }}</span>
    </div>
    <div class="summary-item expense">
      <span class="summary-label">Total Monthly Expenses</span>
      <span class="summary-value">{{ statement.totalExpenses | number:'1.2-2' }}</span>
    </div>
    <div class="summary-item net" [class.positive]="statement.netIncome >= 0" [class.negative]="statement.netIncome < 0">
      <span class="summary-label">Net Monthly Income</span>
      <span class="summary-value">{{ statement.netIncome | number:'1.2-2' }}</span>
    </div>
  </div>

  <!-- Income Section -->
  <div class="section">
    <div class="section-header">
      <h4>Income</h4>
      <button mat-stroked-button (click)="showIncomeForm = true" *mifosxHasPermission="'CREATE_DATATABLE'"><mat-icon>add</mat-icon> Add Income</button>
    </div>
    @if (showIncomeForm) { /* inline form for income */ }
    <table mat-table [dataSource]="statement.incomeItems" class="section-table">
      <ng-container matColumnDef="source"
        ><th mat-header-cell *matHeaderCellDef>Source</th>
        <td mat-cell *matCellDef="let row">{{ row.source }}</td></ng-container
      >
      <ng-container matColumnDef="amount"
        ><th mat-header-cell *matHeaderCellDef>Amount</th>
        <td mat-cell *matCellDef="let row">{{ row.amount | number:'1.2-2' }}</td></ng-container
      >
      <ng-container matColumnDef="frequency"
        ><th mat-header-cell *matHeaderCellDef>Frequency</th>
        <td mat-cell *matCellDef="let row">{{ row.frequency }}</td></ng-container
      >
      <ng-container matColumnDef="actions"
        ><th mat-header-cell *matHeaderCellDef></th>
        <td mat-cell *matCellDef="let row">
          <button mat-icon-button color="warn" (click)="deleteIncome(row)"><mat-icon>delete</mat-icon></button>
        </td></ng-container
      >
      <tr mat-header-row *matHeaderRowDef="['source','amount','frequency','actions']"></tr>
      <tr mat-row *matRowDef="let row; columns: ['source','amount','frequency','actions']"></tr>
    </table>
  </div>

  <!-- Expense Section (same pattern as Income) -->
  <div class="section">
    <div class="section-header">
      <h4>Expenses</h4>
      <button mat-stroked-button (click)="showExpenseForm = true" *mifosxHasPermission="'CREATE_DATATABLE'"><mat-icon>add</mat-icon> Add Expense</button>
    </div>
    <table mat-table [dataSource]="statement.expenseItems" class="section-table">
      <ng-container matColumnDef="source"
        ><th mat-header-cell *matHeaderCellDef>Category</th>
        <td mat-cell *matCellDef="let row">{{ row.source }}</td></ng-container
      >
      <ng-container matColumnDef="amount"
        ><th mat-header-cell *matHeaderCellDef>Amount</th>
        <td mat-cell *matCellDef="let row">{{ row.amount | number:'1.2-2' }}</td></ng-container
      >
      <ng-container matColumnDef="frequency"
        ><th mat-header-cell *matHeaderCellDef>Frequency</th>
        <td mat-cell *matCellDef="let row">{{ row.frequency }}</td></ng-container
      >
      <ng-container matColumnDef="actions"
        ><th mat-header-cell *matHeaderCellDef></th>
        <td mat-cell *matCellDef="let row">
          <button mat-icon-button color="warn" (click)="deleteExpense(row)"><mat-icon>delete</mat-icon></button>
        </td></ng-container
      >
      <tr mat-header-row *matHeaderRowDef="['source','amount','frequency','actions']"></tr>
      <tr mat-row *matRowDef="let row; columns: ['source','amount','frequency','actions']"></tr>
    </table>
  </div>
  }
</div>
```

---

## 5. Multi-funding Tab

**Directory:** `src/app/clients/clients-view/multi-funding-tab/`
**Route:** `/clients/:clientId/multi-funding`

### Purpose

Tracks multiple funding sources associated with a client (e.g., different donors, government programs, internal funds).

### Fineract API

```
GET/POST/PUT/DELETE /fineract-provider/api/v1/datatables/m_client_funding_sources/{clientId}
```

### Component

```typescript
// src/app/clients/clients-view/multi-funding-tab/multi-funding-tab.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { MsaccoDataTableComponent } from 'app/shared/msacco-data-table/msacco-data-table.component';
import { StatusBadgeComponent } from 'app/shared/status-badge/status-badge.component';
import { ColumnDef } from 'app/shared/msacco-data-table/msacco-data-table.interfaces';
import { MultiFundingService, FundingSource } from './multi-funding.service';

@Component({
  selector: 'mifosx-multi-funding-tab',
  templateUrl: './multi-funding-tab.component.html',
  styleUrls: ['./multi-funding-tab.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MsaccoDataTableComponent,
    StatusBadgeComponent
  ]
})
export class MultiFundingTabComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private dialog = inject(MatDialog);
  private fundingService = inject(MultiFundingService);

  clientId: number;
  fundingSources: FundingSource[] = [];

  columns: ColumnDef[] = [
    { name: 'funderName', header: 'Funder Name', cell: (row) => row.funderName },
    { name: 'amount', header: 'Amount', cell: (row) => row.amount?.toString(), type: 'currency' },
    { name: 'date', header: 'Date', cell: (row) => row.date, type: 'date' },
    { name: 'status', header: 'Status', cell: (row) => row.status, type: 'status' },
    { name: 'actions', header: '', cell: () => '', type: 'action' }
  ];

  ngOnInit(): void {
    this.clientId = Number(this.route.parent.snapshot.paramMap.get('clientId'));
    this.loadData();
  }

  private loadData(): void {
    this.fundingService.getFundingSources(this.clientId).subscribe((data) => {
      this.fundingSources = data;
    });
  }

  addFundingSource(): void {
    // Open AddFundingDialogComponent, on close refresh data
  }

  editFundingSource(source: FundingSource): void {
    // Open EditFundingDialogComponent with source data
  }

  deleteFundingSource(source: FundingSource): void {
    if (source.id) {
      this.fundingService.deleteFundingSource(this.clientId, source.id).subscribe(() => this.loadData());
    }
  }
}
```

### Service

```typescript
// src/app/clients/clients-view/multi-funding-tab/multi-funding.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { SettingsService } from 'app/settings/settings.service';

export interface FundingSource {
  id?: number;
  funderName: string;
  amount: number;
  date: string;
  status: string; // 'Active' | 'Completed' | 'Pending'
  notes?: string;
}

@Injectable({ providedIn: 'root' })
export class MultiFundingService {
  private http = inject(HttpClient);
  private settingsService = inject(SettingsService);

  private readonly TABLE = 'm_client_funding_sources';

  private get apiBase(): string {
    return `${this.settingsService.serverUrl}/api/v1`;
  }

  getFundingSources(clientId: number): Observable<FundingSource[]> {
    return this.http.get<FundingSource[]>(`${this.apiBase}/datatables/${this.TABLE}/${clientId}`, {
      params: new HttpParams().set('genericResultSet', 'false')
    });
  }

  addFundingSource(clientId: number, source: FundingSource): Observable<any> {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    return this.http.post(`${this.apiBase}/datatables/${this.TABLE}/${clientId}`, { ...source, locale, dateFormat });
  }

  updateFundingSource(clientId: number, entryId: number, source: FundingSource): Observable<any> {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    return this.http.put(`${this.apiBase}/datatables/${this.TABLE}/${clientId}/${entryId}`, { ...source, locale, dateFormat });
  }

  deleteFundingSource(clientId: number, entryId: number): Observable<any> {
    return this.http.delete(`${this.apiBase}/datatables/${this.TABLE}/${clientId}/${entryId}`);
  }
}
```

---

## 6. Client Tasks Tab

**Directory:** `src/app/clients/clients-view/client-tasks-tab/`
**Route:** `/clients/:clientId/tasks`

### Purpose

Manage task assignments related to the client (follow-up calls, document collection, etc.).

### Fineract API

```
GET/POST/PUT/DELETE /fineract-provider/api/v1/datatables/m_client_tasks/{clientId}
```

### Component

```typescript
// src/app/clients/clients-view/client-tasks-tab/client-tasks-tab.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { MsaccoDataTableComponent } from 'app/shared/msacco-data-table/msacco-data-table.component';
import { StatusBadgeComponent } from 'app/shared/status-badge/status-badge.component';
import { ColumnDef, StatusTab } from 'app/shared/msacco-data-table/msacco-data-table.interfaces';
import { ClientTasksService, ClientTask } from './client-tasks.service';

@Component({
  selector: 'mifosx-client-tasks-tab',
  templateUrl: './client-tasks-tab.component.html',
  styleUrls: ['./client-tasks-tab.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MsaccoDataTableComponent,
    StatusBadgeComponent
  ]
})
export class ClientTasksTabComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private dialog = inject(MatDialog);
  private tasksService = inject(ClientTasksService);

  clientId: number;
  tasks: ClientTask[] = [];

  columns: ColumnDef[] = [
    { name: 'title', header: 'Task', cell: (row) => row.title },
    { name: 'assignedTo', header: 'Assigned To', cell: (row) => row.assignedTo },
    { name: 'dueDate', header: 'Due Date', cell: (row) => row.dueDate, type: 'date' },
    { name: 'priority', header: 'Priority', cell: (row) => row.priority, type: 'status' },
    { name: 'status', header: 'Status', cell: (row) => row.status, type: 'status' },
    { name: 'actions', header: '', cell: () => '', type: 'action' }
  ];

  statusTabs: StatusTab[] = [
    { label: 'All', value: '', count: 0 },
    { label: 'Open', value: 'Open', count: 0 },
    { label: 'In Progress', value: 'In Progress', count: 0 },
    { label: 'Completed', value: 'Completed', count: 0 }
  ];

  priorityStatusMap = {
    high: { color: '#dc3545', label: 'High' },
    medium: { color: '#ffc107', textColor: '#333', label: 'Medium' },
    low: { color: '#28a745', label: 'Low' }
  };

  ngOnInit(): void {
    this.clientId = Number(this.route.parent.snapshot.paramMap.get('clientId'));
    this.loadData();
  }

  private loadData(): void {
    this.tasksService.getTasks(this.clientId).subscribe((tasks) => {
      this.tasks = tasks;
      this.updateTabCounts();
    });
  }

  private updateTabCounts(): void {
    this.statusTabs[0].count = this.tasks.length;
    this.statusTabs[1].count = this.tasks.filter((t) => t.status === 'Open').length;
    this.statusTabs[2].count = this.tasks.filter((t) => t.status === 'In Progress').length;
    this.statusTabs[3].count = this.tasks.filter((t) => t.status === 'Completed').length;
  }

  createTask(): void {
    // Open CreateTaskDialogComponent, on close refresh data
  }
}
```

### Service

```typescript
// src/app/clients/clients-view/client-tasks-tab/client-tasks.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { SettingsService } from 'app/settings/settings.service';

export interface ClientTask {
  id?: number;
  title: string;
  description?: string;
  assignedTo: string;
  dueDate: string;
  status: string; // 'Open' | 'In Progress' | 'Completed' | 'Cancelled'
  priority: string; // 'High' | 'Medium' | 'Low'
}

@Injectable({ providedIn: 'root' })
export class ClientTasksService {
  private http = inject(HttpClient);
  private settingsService = inject(SettingsService);

  private readonly TABLE = 'm_client_tasks';

  private get apiBase(): string {
    return `${this.settingsService.serverUrl}/api/v1`;
  }

  getTasks(clientId: number): Observable<ClientTask[]> {
    return this.http.get<ClientTask[]>(`${this.apiBase}/datatables/${this.TABLE}/${clientId}`, {
      params: new HttpParams().set('genericResultSet', 'false')
    });
  }

  createTask(clientId: number, task: ClientTask): Observable<any> {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    return this.http.post(`${this.apiBase}/datatables/${this.TABLE}/${clientId}`, { ...task, locale, dateFormat });
  }

  updateTask(clientId: number, taskId: number, task: Partial<ClientTask>): Observable<any> {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    return this.http.put(`${this.apiBase}/datatables/${this.TABLE}/${clientId}/${taskId}`, { ...task, locale, dateFormat });
  }

  deleteTask(clientId: number, taskId: number): Observable<any> {
    return this.http.delete(`${this.apiBase}/datatables/${this.TABLE}/${clientId}/${taskId}`);
  }
}
```

---

## 7. Guarantor For Tab

**Directory:** `src/app/clients/clients-view/guarantor-for-tab/`
**Route:** `/clients/:clientId/guarantor-for`

### Purpose

Shows all loans where the current client is acting as a guarantor for another borrower.

### Fineract API

There is no direct Fineract endpoint that queries "loans where client X is a guarantor." Two approaches:

1. **Custom endpoint (preferred):** Backend adds `GET /clients/{clientId}/guarantorships` returning loans where this client appears as a guarantor.
2. **Client-side derivation:** Search loan guarantor data tables, but this is expensive. Use approach 1 if possible.

Fallback: Use custom datatable `m_client_guarantor_for` maintained by triggers or batch processes.

### Service

```typescript
// src/app/clients/clients-view/guarantor-for-tab/guarantor-for.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { SettingsService } from 'app/settings/settings.service';

export interface GuarantorshipRecord {
  loanId: number;
  loanAccountNo: string;
  borrowerName: string;
  borrowerClientId: number;
  loanAmount: number;
  guaranteeAmount: number;
  loanStatus: string;
  relationship: string; // 'Self' | 'Spouse' | 'Parent' | 'Child' | 'Other'
}

@Injectable({ providedIn: 'root' })
export class GuarantorForService {
  private http = inject(HttpClient);
  private settingsService = inject(SettingsService);

  private get apiBase(): string {
    return `${this.settingsService.serverUrl}/api/v1`;
  }

  getGuarantorships(clientId: number): Observable<GuarantorshipRecord[]> {
    return this.http.get<GuarantorshipRecord[]>(`${this.apiBase}/clients/${clientId}/guarantorships`);
  }
}
```

### Component

```typescript
// src/app/clients/clients-view/guarantor-for-tab/guarantor-for-tab.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { MsaccoDataTableComponent } from 'app/shared/msacco-data-table/msacco-data-table.component';
import { StatusBadgeComponent } from 'app/shared/status-badge/status-badge.component';
import { ColumnDef } from 'app/shared/msacco-data-table/msacco-data-table.interfaces';
import { GuarantorForService, GuarantorshipRecord } from './guarantor-for.service';

@Component({
  selector: 'mifosx-guarantor-for-tab',
  templateUrl: './guarantor-for-tab.component.html',
  styleUrls: ['./guarantor-for-tab.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MsaccoDataTableComponent,
    StatusBadgeComponent
  ]
})
export class GuarantorForTabComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private guarantorService = inject(GuarantorForService);

  clientId: number;
  guarantorships: GuarantorshipRecord[] = [];

  columns: ColumnDef[] = [
    { name: 'loanAccountNo', header: 'Loan Account', cell: (row) => row.loanAccountNo },
    { name: 'borrowerName', header: 'Borrower Name', cell: (row) => row.borrowerName },
    { name: 'loanAmount', header: 'Loan Amount', cell: (row) => row.loanAmount?.toString(), type: 'currency' },
    { name: 'guaranteeAmount', header: 'Guarantee Amount', cell: (row) => row.guaranteeAmount?.toString(), type: 'currency' },
    { name: 'loanStatus', header: 'Loan Status', cell: (row) => row.loanStatus, type: 'status' },
    { name: 'relationship', header: 'Relationship', cell: (row) => row.relationship }
  ];

  ngOnInit(): void {
    this.clientId = Number(this.route.parent.snapshot.paramMap.get('clientId'));
    this.loadData();
  }

  private loadData(): void {
    this.guarantorService.getGuarantorships(this.clientId).subscribe((data) => {
      this.guarantorships = data;
    });
  }

  viewLoan(record: GuarantorshipRecord): void {
    this.router.navigate([
      '/clients',
      record.borrowerClientId,
      'loans-accounts',
      record.loanId
    ]);
  }
}
```

### Template

```html
<div class="guarantor-for-tab">
  <div class="tab-header">
    <h3>Loans Guaranteed by This Client</h3>
  </div>

  @if (guarantorships.length > 0) {
  <msacco-data-table [columns]="columns" [dataSource]="guarantorships" [searchEnabled]="true" [exportEnabled]="true" [exportConfig]="{ filenamePrefix: 'guarantorships', title: 'Guarantorship Records' }" (rowClick)="viewLoan($event)"></msacco-data-table>
  } @else {
  <div class="empty-state">
    <mat-icon>security</mat-icon>
    <p>This client is not a guarantor for any loans.</p>
  </div>
  }
</div>
```

---

## Route Configuration

Add the following child routes to `src/app/clients/clients-routing.module.ts` under the `:clientId` route's children:

```typescript
// Add to the children array of the :clientId route

{
  path: 'crb-account',
  component: CRBAccountTabComponent,
  data: { title: 'CRB Account', breadcrumb: 'CRB Account', routeParamBreadcrumb: false },
  resolve: { creditChecks: CRBAccountResolver }
},
{
  path: 'business-details',
  component: BusinessDetailsTabComponent,
  data: { title: 'Business Details', breadcrumb: 'Business Details', routeParamBreadcrumb: false }
},
{
  path: 'ppi',
  component: PPITabComponent,
  data: { title: 'PPI', breadcrumb: 'PPI', routeParamBreadcrumb: false },
  resolve: { ppiSurveys: PPISurveyResolver }
},
{
  path: 'financial-statements',
  component: FinancialStatementsTabComponent,
  data: { title: 'Financial Statements', breadcrumb: 'Financial Statements', routeParamBreadcrumb: false }
},
{
  path: 'multi-funding',
  component: MultiFundingTabComponent,
  data: { title: 'Multi-funding', breadcrumb: 'Multi-funding', routeParamBreadcrumb: false }
},
{
  path: 'tasks',
  component: ClientTasksTabComponent,
  data: { title: 'Tasks', breadcrumb: 'Tasks', routeParamBreadcrumb: false }
},
{
  path: 'guarantor-for',
  component: GuarantorForTabComponent,
  data: { title: 'Guarantor For', breadcrumb: 'Guarantor For', routeParamBreadcrumb: false }
}
```

### Tab Links in clients-view.component.html

Add the following tab links to the `mat-tab-nav-bar` in `clients-view.component.html`:

```html
<a mat-tab-link [routerLink]="['crb-account']" routerLinkActive #crbTab="routerLinkActive" [active]="crbTab.isActive"> {{ formatTabLabel('CRB Account') }} </a>
<a mat-tab-link [routerLink]="['business-details']" routerLinkActive #bizTab="routerLinkActive" [active]="bizTab.isActive"> {{ formatTabLabel('Business Details') }} </a>
<a mat-tab-link [routerLink]="['ppi']" routerLinkActive #ppiTab="routerLinkActive" [active]="ppiTab.isActive"> {{ formatTabLabel('PPI') }} </a>
<a mat-tab-link [routerLink]="['financial-statements']" routerLinkActive #finTab="routerLinkActive" [active]="finTab.isActive"> {{ formatTabLabel('Financial Statements') }} </a>
<a mat-tab-link [routerLink]="['multi-funding']" routerLinkActive #fundTab="routerLinkActive" [active]="fundTab.isActive"> {{ formatTabLabel('Multi-funding') }} </a>
<a mat-tab-link [routerLink]="['tasks']" routerLinkActive #taskTab="routerLinkActive" [active]="taskTab.isActive"> {{ formatTabLabel('Tasks') }} </a>
<a mat-tab-link [routerLink]="['guarantor-for']" routerLinkActive #guarTab="routerLinkActive" [active]="guarTab.isActive"> {{ formatTabLabel('Guarantor For') }} </a>
```

---

## Backend Requirements

### Custom Datatables to Register

The following Fineract datatables must be created on the backend to support tabs 2, 4, 5, and 6:

| Datatable Name              | Entity     | Columns                                                                                                                                      |
| --------------------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `m_client_business_details` | `m_client` | businessName (varchar), businessType (varchar), startDate (date), address (text), postalCode (varchar), county (varchar), description (text) |
| `m_client_income`           | `m_client` | source (varchar), amount (decimal), frequency (varchar)                                                                                      |
| `m_client_expenses`         | `m_client` | source (varchar), amount (decimal), frequency (varchar)                                                                                      |
| `m_client_funding_sources`  | `m_client` | funderName (varchar), amount (decimal), date (date), status (varchar), notes (text)                                                          |
| `m_client_tasks`            | `m_client` | title (varchar), description (text), assignedTo (varchar), dueDate (date), status (varchar), priority (varchar)                              |

### Custom API Endpoints Needed

| Endpoint                           | Purpose                |
| ---------------------------------- | ---------------------- |
| `GET /clients/{id}/creditChecks`   | CRB tab data           |
| `POST /clients/{id}/creditChecks`  | Request new CRB check  |
| `GET /clients/{id}/guarantorships` | Guarantor For tab data |

---

## File Checklist

| Tab                  | Files to Create                                                                                   |
| -------------------- | ------------------------------------------------------------------------------------------------- |
| CRB Account          | `crb-account-tab.component.ts`, `.html`, `.scss`, `crb.service.ts`, `crb-account-tab.resolver.ts` |
| Business Details     | `business-details-tab.component.ts`, `.html`, `.scss`, `business-details.service.ts`              |
| PPI                  | `ppi-tab.component.ts`, `.html`, `.scss` (uses existing survey service)                           |
| Financial Statements | `financial-statements-tab.component.ts`, `.html`, `.scss`, `financial-statements.service.ts`      |
| Multi-funding        | `multi-funding-tab.component.ts`, `.html`, `.scss`, `multi-funding.service.ts`                    |
| Client Tasks         | `client-tasks-tab.component.ts`, `.html`, `.scss`, `client-tasks.service.ts`                      |
| Guarantor For        | `guarantor-for-tab.component.ts`, `.html`, `.scss`, `guarantor-for.service.ts`                    |

### Files to Modify

| File                                                       | Change                                         |
| ---------------------------------------------------------- | ---------------------------------------------- |
| `src/app/clients/clients-routing.module.ts`                | Add 7 new child routes under `:clientId`       |
| `src/app/clients/clients-view/clients-view.component.html` | Add 7 new `mat-tab-link` entries               |
| `src/app/clients/clients-view/clients-view.component.ts`   | Add imports for new tab components (if needed) |
