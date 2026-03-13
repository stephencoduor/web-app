# TECH-006: Client Transfers Module Specification

## Overview

This specification covers a dedicated Client Transfers module at `src/app/client-transfers/`. The existing application has individual transfer actions scattered within `src/app/clients/clients-view/client-actions/` (transfer-client, accept-client-transfer, reject-client-transfer, undo-client-transfer). This new module consolidates all transfer operations into a centralized management interface with pending approvals and history views.

**Existing patterns referenced:**

- `TransferClientComponent` at `src/app/clients/clients-view/client-actions/transfer-client/`
- `AcceptClientTransferComponent` at `src/app/clients/clients-view/client-actions/accept-client-transfer/`
- `ClientsService.executeClientCommand()` for transfer API calls

---

## 1. Module Architecture

### 1.1 File Structure

```
src/app/client-transfers/
  ├── client-transfers.module.ts
  ├── client-transfers-routing.module.ts
  ├── client-transfers.service.ts
  ├── models/
  │   └── client-transfer.model.ts
  ├── transfer-branch/
  │   ├── transfer-branch.component.ts
  │   ├── transfer-branch.component.html
  │   └── transfer-branch.component.scss
  ├── transfer-group/
  │   ├── transfer-group.component.ts
  │   ├── transfer-group.component.html
  │   └── transfer-group.component.scss
  ├── transfer-pending/
  │   ├── transfer-pending.component.ts
  │   ├── transfer-pending.component.html
  │   └── transfer-pending.component.scss
  └── transfer-history/
      ├── transfer-history.component.ts
      ├── transfer-history.component.html
      └── transfer-history.component.scss
```

### 1.2 Routing Configuration

```
/client-transfers
  /branch    -> TransferBranchComponent
  /group     -> TransferGroupComponent
  /pending   -> TransferPendingComponent
  /history   -> TransferHistoryComponent
```

---

## 2. Data Models

```typescript
// models/client-transfer.model.ts

export interface TransferRequest {
  clientId: number;
  clientName?: string;
  destinationOfficeId?: number;
  destinationGroupId?: number;
  transferDate: string;
  note?: string;
  locale: string;
  dateFormat: string;
}

export interface TransferRecord {
  id: number;
  clientId: number;
  clientName: string;
  sourceOfficeName: string;
  sourceOfficeId: number;
  destinationOfficeName: string;
  destinationOfficeId: number;
  transferDate: string;
  transferType: 'branch' | 'group';
  status: TransferStatus;
  initiatedBy: string;
  initiatedDate: string;
  note?: string;
  /** For group transfers */
  sourceGroupName?: string;
  destinationGroupName?: string;
}

export enum TransferStatus {
  PENDING = 'pending',
  APPROVED = 'approved',
  REJECTED = 'rejected',
  WITHDRAWN = 'withdrawn'
}

export interface ClientSearchResult {
  id: number;
  displayName: string;
  officeName: string;
  officeId: number;
  status: { value: string };
  /** Group memberships */
  groups?: Array<{ id: number; name: string }>;
}
```

---

## 3. Module and Routing

```typescript
// client-transfers.module.ts

import { NgModule } from '@angular/core';
import { ClientTransfersRoutingModule } from './client-transfers-routing.module';
import { ClientTransfersService } from './client-transfers.service';

@NgModule({
  imports: [
    ClientTransfersRoutingModule
  ],
  providers: [
    ClientTransfersService
  ]
})
export class ClientTransfersModule {}
```

```typescript
// client-transfers-routing.module.ts

import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { TransferBranchComponent } from './transfer-branch/transfer-branch.component';
import { TransferGroupComponent } from './transfer-group/transfer-group.component';
import { TransferPendingComponent } from './transfer-pending/transfer-pending.component';
import { TransferHistoryComponent } from './transfer-history/transfer-history.component';

const routes: Routes = [
  {
    path: '',
    children: [
      { path: 'branch', component: TransferBranchComponent, data: { title: 'Transfer Client Between Branches', breadcrumb: 'Branch Transfer' } },
      { path: 'group', component: TransferGroupComponent, data: { title: 'Transfer Client Between Groups', breadcrumb: 'Group Transfer' } },
      { path: 'pending', component: TransferPendingComponent, data: { title: 'Pending Transfer Approvals', breadcrumb: 'Pending Approvals' } },
      { path: 'history', component: TransferHistoryComponent, data: { title: 'Transfer History', breadcrumb: 'Transfer History' } },
      { path: '', redirectTo: 'branch', pathMatch: 'full' }
    ]
  }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class ClientTransfersRoutingModule {}
```

### App Routing Integration

In `src/app/app-routing.module.ts`, add the lazy-loaded route:

```typescript
{
  path: 'client-transfers',
  loadChildren: () => import('./client-transfers/client-transfers.module').then(m => m.ClientTransfersModule),
  data: { title: 'Client Transfers', breadcrumb: 'Client Transfers' }
}
```

---

## 4. Service

```typescript
// client-transfers.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { TransferRequest, TransferRecord } from './models/client-transfer.model';

@Injectable({ providedIn: 'root' })
export class ClientTransfersService {
  private http = inject(HttpClient);

  /**
   * Propose a client transfer between branches.
   * Fineract: POST /clients/{clientId}?command=proposeTransfer
   */
  proposeTransfer(clientId: number, data: TransferRequest): Observable<any> {
    const httpParams = new HttpParams().set('command', 'proposeTransfer');
    return this.http.post(`/clients/${clientId}`, data, { params: httpParams });
  }

  /**
   * Accept a pending client transfer.
   * Fineract: POST /clients/{clientId}?command=acceptTransfer
   */
  acceptTransfer(clientId: number, data: { transferDate: string; locale: string; dateFormat: string }): Observable<any> {
    const httpParams = new HttpParams().set('command', 'acceptTransfer');
    return this.http.post(`/clients/${clientId}`, data, { params: httpParams });
  }

  /**
   * Reject a pending client transfer.
   * Fineract: POST /clients/{clientId}?command=rejectTransfer
   */
  rejectTransfer(clientId: number, data: { transferDate: string; note?: string; locale: string; dateFormat: string }): Observable<any> {
    const httpParams = new HttpParams().set('command', 'rejectTransfer');
    return this.http.post(`/clients/${clientId}`, data, { params: httpParams });
  }

  /**
   * Withdraw a proposed client transfer.
   * Fineract: POST /clients/{clientId}?command=withdrawTransfer
   */
  withdrawTransfer(clientId: number, data: { transferDate: string; note?: string; locale: string; dateFormat: string }): Observable<any> {
    const httpParams = new HttpParams().set('command', 'withdrawTransfer');
    return this.http.post(`/clients/${clientId}`, data, { params: httpParams });
  }

  /**
   * Propose and accept a client transfer in one step (for group transfers).
   * Fineract: POST /clients/{clientId}?command=proposeAndAcceptTransfer
   */
  proposeAndAcceptTransfer(clientId: number, data: TransferRequest): Observable<any> {
    const httpParams = new HttpParams().set('command', 'proposeAndAcceptTransfer');
    return this.http.post(`/clients/${clientId}`, data, { params: httpParams });
  }

  /**
   * Get offices for destination selection.
   * Fineract: GET /offices
   */
  getOffices(): Observable<any> {
    return this.http.get('/offices');
  }

  /**
   * Get groups filtered by office for group transfer.
   * Fineract: GET /groups?officeId={officeId}&paged=true
   */
  getGroupsByOffice(officeId: number): Observable<any> {
    const httpParams = new HttpParams().set('officeId', officeId.toString()).set('paged', 'true').set('limit', '200');
    return this.http.get('/groups', { params: httpParams });
  }

  /**
   * Search clients by name or ID.
   * Fineract: GET /clients?displayName={name}&orderBy=displayName&sortOrder=ASC
   */
  searchClients(searchTerm: string): Observable<any> {
    const httpParams = new HttpParams().set('displayName', searchTerm).set('orderBy', 'displayName').set('sortOrder', 'ASC').set('orphansOnly', 'false').set('limit', '50');
    return this.http.get('/clients', { params: httpParams });
  }

  /**
   * Get clients with transfer-in-progress status for pending approvals.
   * Fineract: GET /clients?status=transferInProgress&limit=200
   */
  getPendingTransfers(): Observable<any> {
    const httpParams = new HttpParams()
      .set('status', '600') // 600 = Transfer In Progress
      .set('limit', '200');
    return this.http.get('/clients', { params: httpParams });
  }

  /**
   * Get client transfer data for proposal form.
   * Fineract: GET /clients/{clientId}?command=proposeTransfer&template=true
   */
  getTransferTemplate(clientId: number): Observable<any> {
    return this.http.get(`/clients/${clientId}/transfertemplate`);
  }
}
```

---

## 5. Page Components

### 5.1 Transfer Client Between Branches

```typescript
// transfer-branch/transfer-branch.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';
import { Router } from '@angular/router';
import { MatAutocompleteTrigger, MatAutocomplete } from '@angular/material/autocomplete';
import { CdkTextareaAutosize } from '@angular/cdk/text-field';
import { ClientTransfersService } from '../client-transfers.service';
import { SettingsService } from 'app/settings/settings.service';
import { Dates } from 'app/core/utils/dates';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-transfer-branch',
  templateUrl: './transfer-branch.component.html',
  styleUrls: ['./transfer-branch.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatAutocompleteTrigger,
    MatAutocomplete,
    CdkTextareaAutosize
  ]
})
export class TransferBranchComponent implements OnInit {
  private formBuilder = inject(UntypedFormBuilder);
  private transferService = inject(ClientTransfersService);
  private settingsService = inject(SettingsService);
  private dateUtils = inject(Dates);
  private router = inject(Router);

  transferForm: UntypedFormGroup;
  offices: any[] = [];
  clientsData: any[] = [];
  selectedClient: any = null;

  minDate = new Date(2000, 0, 1);
  maxDate = new Date();

  ngOnInit() {
    this.maxDate = this.settingsService.businessDate;
    this.createForm();
    this.loadOffices();
    this.setupClientSearch();
  }

  createForm() {
    this.transferForm = this.formBuilder.group({
      clientSearch: [
        '',
        Validators.required
      ],
      destinationOfficeId: [
        '',
        Validators.required
      ],
      transferDate: [
        '',
        Validators.required
      ],
      note: ['']
    });
  }

  loadOffices() {
    this.transferService.getOffices().subscribe((offices: any) => {
      this.offices = offices;
    });
  }

  setupClientSearch() {
    this.transferForm.get('clientSearch').valueChanges.subscribe((value: string) => {
      if (typeof value === 'string' && value.length >= 2) {
        this.transferService.searchClients(value).subscribe((data: any) => {
          this.clientsData = data.pageItems || [];
        });
      }
    });
  }

  displayClient(client: any): string | undefined {
    return client ? client.displayName : undefined;
  }

  onClientSelected(client: any) {
    this.selectedClient = client;
    // Filter out client's current office from destination list
  }

  submit() {
    if (!this.transferForm.valid || !this.selectedClient) return;

    const formData = this.transferForm.value;
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    const transferDate = formData.transferDate instanceof Date ? this.dateUtils.formatDate(formData.transferDate, dateFormat) : formData.transferDate;

    const payload = {
      destinationOfficeId: formData.destinationOfficeId,
      transferDate,
      note: formData.note || '',
      locale,
      dateFormat
    };

    this.transferService.proposeTransfer(this.selectedClient.id, payload as any).subscribe(() => {
      this.router.navigate(['/client-transfers/pending']);
    });
  }
}
```

**Template: `transfer-branch.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <h3 class="mat-h3">{{ 'labels.heading.Transfer Client Between Branches' | translate }}</h3>

      <div class="warning-banner">
        <fa-icon icon="exclamation-triangle" class="m-r-10"></fa-icon>
        {{ 'labels.text.transfer-warning' | translate }}
      </div>

      <form [formGroup]="transferForm" (ngSubmit)="submit()">
        <div class="layout-column">
          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Search Client' | translate }}</mat-label>
            <input matInput formControlName="clientSearch" [matAutocomplete]="clientsAutocomplete" />
            <mat-autocomplete autoActiveFirstOption #clientsAutocomplete="matAutocomplete" [displayWith]="displayClient" (optionSelected)="onClientSelected($event.option.value)">
              @for (client of clientsData; track client.id) {
              <mat-option [value]="client"> {{ client.displayName }} - {{ client.officeName }} </mat-option>
              }
            </mat-autocomplete>
            @if (transferForm.controls.clientSearch.hasError('required')) {
            <mat-error>
              {{ 'labels.inputs.Client' | translate }} {{ 'labels.commons.is' | translate }}
              <strong>{{ 'labels.commons.required' | translate }}</strong>
            </mat-error>
            }
          </mat-form-field>

          @if (selectedClient) {
          <div class="client-info mat-elevation-z1 padding-1 margin-b">
            <p><strong>{{ 'labels.inputs.Current Branch' | translate }}:</strong> {{ selectedClient.officeName }}</p>
            <p><strong>{{ 'labels.inputs.Status' | translate }}:</strong> {{ selectedClient.status?.value }}</p>
          </div>
          }

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Destination Branch' | translate }}</mat-label>
            <mat-select required formControlName="destinationOfficeId">
              @for (office of offices; track office.id) {
              <mat-option [value]="office.id" [disabled]="selectedClient && office.id === selectedClient.officeId"> {{ office.name }} </mat-option>
              }
            </mat-select>
            @if (transferForm.controls.destinationOfficeId.hasError('required')) {
            <mat-error>
              {{ 'labels.inputs.Destination Branch' | translate }} {{ 'labels.commons.is' | translate }}
              <strong>{{ 'labels.commons.required' | translate }}</strong>
            </mat-error>
            }
          </mat-form-field>

          <mat-form-field (click)="transferDatePicker.open()">
            <mat-label>{{ 'labels.inputs.Transfer Date' | translate }}</mat-label>
            <input matInput [min]="minDate" [max]="maxDate" [matDatepicker]="transferDatePicker" required formControlName="transferDate" />
            <mat-datepicker-toggle matSuffix [for]="transferDatePicker"></mat-datepicker-toggle>
            <mat-datepicker #transferDatePicker></mat-datepicker>
            @if (transferForm.controls.transferDate.hasError('required')) {
            <mat-error>
              {{ 'labels.inputs.Transfer Date' | translate }} {{ 'labels.commons.is' | translate }}
              <strong>{{ 'labels.commons.required' | translate }}</strong>
            </mat-error>
            }
          </mat-form-field>

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Note' | translate }}</mat-label>
            <textarea matInput formControlName="note" cdkTextareaAutosize cdkAutosizeMinRows="2"></textarea>
          </mat-form-field>
        </div>

        <mat-card-actions class="layout-row align-center gap-5px responsive-column">
          <button type="button" mat-raised-button [routerLink]="['/client-transfers/pending']">{{ 'labels.buttons.Cancel' | translate }}</button>
          <button mat-raised-button color="primary" [disabled]="!transferForm.valid || !selectedClient" *mifosxHasPermission="'PROPOSETRANSFER_CLIENT'">{{ 'labels.buttons.Submit' | translate }}</button>
        </mat-card-actions>
      </form>
    </mat-card-content>
  </mat-card>
</div>
```

**Warning message (add to translation file):**

```json
{
  "labels.text.transfer-warning": "When transferring clients, you will not be able to add, remove or adjust any transactions in the clients loans and savings that have happened before the transfer date."
}
```

---

### 5.2 Transfer Client Between Groups

```typescript
// transfer-group/transfer-group.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';
import { Router } from '@angular/router';
import { MatAutocompleteTrigger, MatAutocomplete } from '@angular/material/autocomplete';
import { ClientTransfersService } from '../client-transfers.service';
import { SettingsService } from 'app/settings/settings.service';
import { Dates } from 'app/core/utils/dates';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-transfer-group',
  templateUrl: './transfer-group.component.html',
  styleUrls: ['./transfer-group.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatAutocompleteTrigger,
    MatAutocomplete
  ]
})
export class TransferGroupComponent implements OnInit {
  private formBuilder = inject(UntypedFormBuilder);
  private transferService = inject(ClientTransfersService);
  private settingsService = inject(SettingsService);
  private dateUtils = inject(Dates);
  private router = inject(Router);

  transferForm: UntypedFormGroup;
  offices: any[] = [];
  clientsData: any[] = [];
  groups: any[] = [];
  selectedClient: any = null;
  selectedOfficeId: number | null = null;

  minDate = new Date(2000, 0, 1);
  maxDate = new Date();

  ngOnInit() {
    this.maxDate = this.settingsService.businessDate;
    this.createForm();
    this.loadOffices();
    this.setupClientSearch();
    this.setupOfficeChange();
  }

  createForm() {
    this.transferForm = this.formBuilder.group({
      clientSearch: [
        '',
        Validators.required
      ],
      destinationOfficeId: [
        '',
        Validators.required
      ],
      destinationGroupId: [
        '',
        Validators.required
      ],
      transferDate: [
        '',
        Validators.required
      ]
    });
  }

  loadOffices() {
    this.transferService.getOffices().subscribe((offices: any) => {
      this.offices = offices;
    });
  }

  setupClientSearch() {
    this.transferForm.get('clientSearch').valueChanges.subscribe((value: string) => {
      if (typeof value === 'string' && value.length >= 2) {
        this.transferService.searchClients(value).subscribe((data: any) => {
          this.clientsData = data.pageItems || [];
        });
      }
    });
  }

  setupOfficeChange() {
    this.transferForm.get('destinationOfficeId').valueChanges.subscribe((officeId: number) => {
      this.selectedOfficeId = officeId;
      this.groups = [];
      this.transferForm.get('destinationGroupId').reset();
      if (officeId) {
        this.transferService.getGroupsByOffice(officeId).subscribe((data: any) => {
          this.groups = data.pageItems || data || [];
        });
      }
    });
  }

  displayClient(client: any): string | undefined {
    return client ? client.displayName : undefined;
  }

  onClientSelected(client: any) {
    this.selectedClient = client;
  }

  submit() {
    if (!this.transferForm.valid || !this.selectedClient) return;

    const formData = this.transferForm.value;
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    const transferDate = formData.transferDate instanceof Date ? this.dateUtils.formatDate(formData.transferDate, dateFormat) : formData.transferDate;

    const payload = {
      destinationGroupId: formData.destinationGroupId,
      destinationOfficeId: formData.destinationOfficeId,
      transferDate,
      locale,
      dateFormat
    };

    this.transferService.proposeAndAcceptTransfer(this.selectedClient.id, payload as any).subscribe(() => {
      this.router.navigate(['/client-transfers/history']);
    });
  }
}
```

**Template: `transfer-group.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <h3 class="mat-h3">{{ 'labels.heading.Transfer Client Between Groups' | translate }}</h3>

      <form [formGroup]="transferForm" (ngSubmit)="submit()">
        <div class="layout-column">
          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Search Client' | translate }}</mat-label>
            <input matInput formControlName="clientSearch" [matAutocomplete]="clientsAutocomplete" />
            <mat-autocomplete autoActiveFirstOption #clientsAutocomplete="matAutocomplete" [displayWith]="displayClient" (optionSelected)="onClientSelected($event.option.value)">
              @for (client of clientsData; track client.id) {
              <mat-option [value]="client"> {{ client.displayName }} - {{ client.officeName }} </mat-option>
              }
            </mat-autocomplete>
          </mat-form-field>

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Destination Branch' | translate }}</mat-label>
            <mat-select required formControlName="destinationOfficeId">
              @for (office of offices; track office.id) {
              <mat-option [value]="office.id">{{ office.name }}</mat-option>
              }
            </mat-select>
          </mat-form-field>

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Destination Group' | translate }}</mat-label>
            <mat-select required formControlName="destinationGroupId" [disabled]="!selectedOfficeId || groups.length === 0">
              @for (group of groups; track group.id) {
              <mat-option [value]="group.id">{{ group.name }}</mat-option>
              }
            </mat-select>
            @if (selectedOfficeId && groups.length === 0) {
            <mat-hint>{{ 'labels.text.No groups in selected branch' | translate }}</mat-hint>
            }
          </mat-form-field>

          <mat-form-field (click)="transferDatePicker.open()">
            <mat-label>{{ 'labels.inputs.Transfer Date' | translate }}</mat-label>
            <input matInput [min]="minDate" [max]="maxDate" [matDatepicker]="transferDatePicker" required formControlName="transferDate" />
            <mat-datepicker-toggle matSuffix [for]="transferDatePicker"></mat-datepicker-toggle>
            <mat-datepicker #transferDatePicker></mat-datepicker>
          </mat-form-field>
        </div>

        <mat-card-actions class="layout-row align-center gap-5px responsive-column">
          <button type="button" mat-raised-button [routerLink]="['/client-transfers/pending']">{{ 'labels.buttons.Cancel' | translate }}</button>
          <button mat-raised-button color="primary" [disabled]="!transferForm.valid || !selectedClient" *mifosxHasPermission="'PROPOSETRANSFER_CLIENT'">{{ 'labels.buttons.Submit' | translate }}</button>
        </mat-card-actions>
      </form>
    </mat-card-content>
  </mat-card>
</div>
```

---

### 5.3 Pending Approvals

```typescript
// transfer-pending/transfer-pending.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { MatDialog } from '@angular/material/dialog';
import { MatTableDataSource } from '@angular/material/table';
import { Router } from '@angular/router';
import { MatTable, MatColumnDef, MatHeaderCellDef, MatHeaderCell, MatCellDef, MatCell, MatHeaderRowDef, MatHeaderRow, MatRowDef, MatRow } from '@angular/material/table';
import { MatIconButton } from '@angular/material/button';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { ClientTransfersService } from '../client-transfers.service';
import { SettingsService } from 'app/settings/settings.service';
import { Dates } from 'app/core/utils/dates';
import { DeleteDialogComponent } from 'app/shared/delete-dialog/delete-dialog.component';
import { DateFormatPipe } from 'app/pipes/date-format.pipe';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-transfer-pending',
  templateUrl: './transfer-pending.component.html',
  styleUrls: ['./transfer-pending.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    FaIconComponent,
    MatIconButton,
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
export class TransferPendingComponent implements OnInit {
  private transferService = inject(ClientTransfersService);
  private settingsService = inject(SettingsService);
  private dateUtils = inject(Dates);
  private dialog = inject(MatDialog);
  private router = inject(Router);

  pendingTransfers: any[] = [];
  dataSource: MatTableDataSource<any>;
  displayedColumns: string[] = [
    'clientName',
    'sourceBranch',
    'destinationBranch',
    'transferDate',
    'status',
    'actions'
  ];

  ngOnInit() {
    this.loadPendingTransfers();
  }

  loadPendingTransfers() {
    this.transferService.getPendingTransfers().subscribe((data: any) => {
      this.pendingTransfers = data.pageItems || [];
      this.dataSource = new MatTableDataSource(this.pendingTransfers);
    });
  }

  approveTransfer(client: any) {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    const transferDate = this.dateUtils.formatDate(this.settingsService.businessDate, dateFormat);

    this.transferService
      .acceptTransfer(client.id, {
        transferDate,
        locale,
        dateFormat
      })
      .subscribe(() => {
        this.loadPendingTransfers();
      });
  }

  rejectTransfer(client: any) {
    const deleteRef = this.dialog.open(DeleteDialogComponent, {
      data: { deleteContext: `transfer for ${client.displayName}` }
    });
    deleteRef.afterClosed().subscribe((response: any) => {
      if (response.delete) {
        const locale = this.settingsService.language.code;
        const dateFormat = this.settingsService.dateFormat;
        const transferDate = this.dateUtils.formatDate(this.settingsService.businessDate, dateFormat);
        this.transferService
          .rejectTransfer(client.id, {
            transferDate,
            locale,
            dateFormat
          })
          .subscribe(() => {
            this.loadPendingTransfers();
          });
      }
    });
  }

  withdrawTransfer(client: any) {
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;
    const transferDate = this.dateUtils.formatDate(this.settingsService.businessDate, dateFormat);
    this.transferService
      .withdrawTransfer(client.id, {
        transferDate,
        locale,
        dateFormat
      })
      .subscribe(() => {
        this.loadPendingTransfers();
      });
  }
}
```

**Template: `transfer-pending.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <div class="layout-row align-center gap-5px margin-b">
        <h3 class="mat-h3 flex">{{ 'labels.heading.Pending Transfer Approvals' | translate }}</h3>
        <button mat-raised-button color="primary" [routerLink]="['/client-transfers/branch']">
          <fa-icon icon="exchange-alt" class="m-r-10"></fa-icon>
          {{ 'labels.buttons.New Transfer' | translate }}
        </button>
      </div>

      <table mat-table [dataSource]="dataSource" class="mat-elevation-z1" [hidden]="!pendingTransfers || pendingTransfers.length === 0">
        <ng-container matColumnDef="clientName">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Client Name' | translate }}</th>
          <td mat-cell *matCellDef="let client">{{ client.displayName }}</td>
        </ng-container>

        <ng-container matColumnDef="sourceBranch">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Source Branch' | translate }}</th>
          <td mat-cell *matCellDef="let client">{{ client.officeName }}</td>
        </ng-container>

        <ng-container matColumnDef="destinationBranch">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Destination Branch' | translate }}</th>
          <td mat-cell *matCellDef="let client">{{ client.transferToOffice?.name || '-' }}</td>
        </ng-container>

        <ng-container matColumnDef="transferDate">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Transfer Date' | translate }}</th>
          <td mat-cell *matCellDef="let client">{{ client.timeline?.activatedOnDate | dateFormat }}</td>
        </ng-container>

        <ng-container matColumnDef="status">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Status' | translate }}</th>
          <td mat-cell *matCellDef="let client">
            <span class="status-badge status-pending">{{ client.status?.value }}</span>
          </td>
        </ng-container>

        <ng-container matColumnDef="actions">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Actions' | translate }}</th>
          <td mat-cell *matCellDef="let client">
            <button mat-icon-button color="primary" (click)="approveTransfer(client)" matTooltip="{{ 'labels.buttons.Approve' | translate }}" *mifosxHasPermission="'ACCEPTTRANSFER_CLIENT'">
              <fa-icon icon="check"></fa-icon>
            </button>
            <button mat-icon-button color="warn" (click)="rejectTransfer(client)" matTooltip="{{ 'labels.buttons.Reject' | translate }}" *mifosxHasPermission="'REJECTTRANSFER_CLIENT'">
              <fa-icon icon="times"></fa-icon>
            </button>
            <button mat-icon-button (click)="withdrawTransfer(client)" matTooltip="{{ 'labels.buttons.Withdraw' | translate }}" *mifosxHasPermission="'WITHDRAWTRANSFER_CLIENT'">
              <fa-icon icon="undo"></fa-icon>
            </button>
          </td>
        </ng-container>

        <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
        <tr mat-row *matRowDef="let row; columns: displayedColumns"></tr>
      </table>

      @if (!pendingTransfers || pendingTransfers.length === 0) {
      <p class="no-data">{{ 'labels.text.No pending transfers' | translate }}</p>
      }
    </mat-card-content>
  </mat-card>
</div>
```

---

### 5.4 Transfer History

```typescript
// transfer-history/transfer-history.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { MatTableDataSource } from '@angular/material/table';
import { MatTable, MatColumnDef, MatHeaderCellDef, MatHeaderCell, MatCellDef, MatCell, MatHeaderRowDef, MatHeaderRow, MatRowDef, MatRow } from '@angular/material/table';
import { MatPaginator } from '@angular/material/paginator';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { ClientTransfersService } from '../client-transfers.service';
import { DateFormatPipe } from 'app/pipes/date-format.pipe';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-transfer-history',
  templateUrl: './transfer-history.component.html',
  styleUrls: ['./transfer-history.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    FaIconComponent,
    MatPaginator,
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
export class TransferHistoryComponent implements OnInit {
  private transferService = inject(ClientTransfersService);

  transferHistory: any[] = [];
  dataSource: MatTableDataSource<any>;
  displayedColumns: string[] = [
    'clientName',
    'source',
    'destination',
    'date',
    'type',
    'status',
    'initiatedBy'
  ];

  ngOnInit() {
    this.loadHistory();
  }

  loadHistory() {
    // Fineract does not have a dedicated transfer history endpoint.
    // Implementation options:
    // 1. Use audit trail: GET /audits?actionName=PROPOSETRANSFER&entityName=CLIENT
    // 2. Use a custom datatable to store transfer history
    // 3. Aggregate from client timeline data
    //
    // For now, use the audit endpoint approach:
    this.transferService.getTransferAuditHistory().subscribe((data: any) => {
      this.transferHistory = data.pageItems || data || [];
      this.dataSource = new MatTableDataSource(this.transferHistory);
    });
  }
}
```

**Note:** The `getTransferAuditHistory()` method should be added to the service:

```typescript
// Add to client-transfers.service.ts

getTransferAuditHistory(): Observable<any> {
  const httpParams = new HttpParams()
    .set('actionName', 'PROPOSETRANSFER')
    .set('entityName', 'CLIENT')
    .set('orderBy', 'id')
    .set('sortOrder', 'DESC')
    .set('limit', '200');
  return this.http.get('/audits', { params: httpParams });
}
```

---

## 6. Fineract API Reference

| Endpoint                                               | Method | Command                    | Purpose                                |
| ------------------------------------------------------ | ------ | -------------------------- | -------------------------------------- |
| `/clients/{clientId}`                                  | POST   | `proposeTransfer`          | Initiate branch transfer               |
| `/clients/{clientId}`                                  | POST   | `acceptTransfer`           | Approve pending transfer               |
| `/clients/{clientId}`                                  | POST   | `rejectTransfer`           | Reject pending transfer                |
| `/clients/{clientId}`                                  | POST   | `withdrawTransfer`         | Withdraw proposed transfer             |
| `/clients/{clientId}`                                  | POST   | `proposeAndAcceptTransfer` | One-step group transfer                |
| `/clients/{clientId}/transfertemplate`                 | GET    | -                          | Get transfer template data             |
| `/offices`                                             | GET    | -                          | List all offices/branches              |
| `/groups?officeId={id}`                                | GET    | -                          | List groups in an office               |
| `/clients?status=600`                                  | GET    | -                          | List clients with transfer-in-progress |
| `/audits?actionName=PROPOSETRANSFER&entityName=CLIENT` | GET    | -                          | Transfer audit history                 |

### API Payloads

**Propose Transfer (Branch):**

```json
POST /clients/{clientId}?command=proposeTransfer
{
  "destinationOfficeId": 2,
  "transferDate": "15 March 2026",
  "note": "Relocating to another branch",
  "locale": "en",
  "dateFormat": "dd MMMM yyyy"
}
```

**Accept Transfer:**

```json
POST /clients/{clientId}?command=acceptTransfer
{
  "transferDate": "15 March 2026",
  "locale": "en",
  "dateFormat": "dd MMMM yyyy"
}
```

**Propose and Accept Transfer (Group):**

```json
POST /clients/{clientId}?command=proposeAndAcceptTransfer
{
  "destinationOfficeId": 2,
  "destinationGroupId": 5,
  "transferDate": "15 March 2026",
  "locale": "en",
  "dateFormat": "dd MMMM yyyy"
}
```

---

## 7. Navigation Integration

Add to the sidebar navigation in the app's navigation configuration:

```typescript
{
  name: 'Client Transfers',
  icon: 'exchange-alt',
  route: '/client-transfers',
  children: [
    { name: 'Branch Transfer', route: '/client-transfers/branch' },
    { name: 'Group Transfer', route: '/client-transfers/group' },
    { name: 'Pending Approvals', route: '/client-transfers/pending' },
    { name: 'Transfer History', route: '/client-transfers/history' }
  ]
}
```

---

## 8. Fineract Permissions

| Permission                | Used In                                       |
| ------------------------- | --------------------------------------------- |
| `PROPOSETRANSFER_CLIENT`  | Branch transfer submit, Group transfer submit |
| `ACCEPTTRANSFER_CLIENT`   | Approve button in Pending Approvals           |
| `REJECTTRANSFER_CLIENT`   | Reject button in Pending Approvals            |
| `WITHDRAWTRANSFER_CLIENT` | Withdraw button in Pending Approvals          |

Use the existing `*mifosxHasPermission` directive from `HasPermissionDirective` (included in `STANDALONE_SHARED_IMPORTS`).
