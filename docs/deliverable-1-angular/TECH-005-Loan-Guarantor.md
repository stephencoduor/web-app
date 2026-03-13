# TECH-005: Loan Guarantor Management Specification

## Overview

This specification covers adding an enhanced guarantor step to the loan creation wizard at `src/app/loans/loans-account-stepper/`. The existing application already has guarantor management for existing loans (create and view) at `src/app/loans/loans-view/loan-account-actions/create-guarantor/` and `view-guarantors/`. This new step integrates guarantor selection directly into the loan creation flow as a stepper step, supporting three selection modes aligned with M-SACCO wireframes.

**New component directory**: `src/app/loans/loans-account-stepper/loans-account-guarantor-step/`

---

## 1. TypeScript Interfaces

```typescript
// src/app/loans/models/guarantor.model.ts

/**
 * Guarantor type IDs as defined in Fineract:
 *   1 = Existing Client
 *   2 = Staff Member
 *   3 = External (New Guarantor)
 */
export enum GuarantorTypeId {
  EXISTING_CLIENT = 1,
  STAFF_MEMBER = 2,
  EXTERNAL = 3
}

export enum GuarantorRelationship {
  SPOUSE = 1,
  PARENT = 2,
  CHILD = 3,
  SIBLING = 4,
  BUSINESS_PARTNER = 5,
  OTHER = 6
}

export interface GuarantorEntry {
  /** Set after POST to Fineract; null while in-memory only */
  id?: number;
  guarantorTypeId: GuarantorTypeId;
  /** For existing client or group member mode */
  entityId?: number;
  /** Display name resolved from client search or group member list */
  displayName?: string;
  /** Relationship type ID from allowedClientRelationshipTypes */
  clientRelationshipTypeId: number;
  /** Relationship label for display */
  relationshipLabel?: string;
  /** For external (new) guarantor mode */
  firstname?: string;
  lastname?: string;
  dob?: string;
  addressLine1?: string;
  addressLine2?: string;
  city?: string;
  zip?: string;
  mobileNumber?: string;
  housePhoneNumber?: string;
  /** Savings account link for guarantee amount */
  savingsId?: number;
  amount?: number;
  /** Status from Fineract response */
  status?: string;
  /** Source mode for UI display */
  sourceMode: 'existing' | 'new' | 'group';
}

export interface GuarantorTemplate {
  guarantorTypeOptions: Array<{ id: number; value: string }>;
  allowedClientRelationshipTypes: Array<{ id: number; name: string }>;
}

export interface GroupMember {
  id: number;
  displayName: string;
  officeName: string;
  selected?: boolean;
}
```

---

## 2. Component Structure

### 2.1 Component Tree

```
GuarantorStepComponent (parent)
  ├── GuarantorModeTabsComponent (tab selector: Existing Client | New Guarantor | Group Members)
  ├── ExistingClientSearchComponent (search and select)
  ├── NewGuarantorFormComponent (inline form)
  ├── GroupMemberSelectorComponent (group member list - conditional)
  └── GuarantorListComponent (added guarantors table)
```

### 2.2 File Structure

```
loans-account-guarantor-step/
  ├── loans-account-guarantor-step.component.ts
  ├── loans-account-guarantor-step.component.html
  ├── loans-account-guarantor-step.component.scss
  ├── guarantor-mode-tabs/
  │   ├── guarantor-mode-tabs.component.ts
  │   └── guarantor-mode-tabs.component.html
  ├── existing-client-search/
  │   ├── existing-client-search.component.ts
  │   └── existing-client-search.component.html
  ├── new-guarantor-form/
  │   ├── new-guarantor-form.component.ts
  │   └── new-guarantor-form.component.html
  ├── group-member-selector/
  │   ├── group-member-selector.component.ts
  │   └── group-member-selector.component.html
  └── guarantor-list/
      ├── guarantor-list.component.ts
      └── guarantor-list.component.html
```

---

## 3. Guarantor Selection Modes

### 3.1 Mode 1: Existing Client

Search field to find existing clients within the system. Follows the same autocomplete pattern used in the existing `CreateGuarantorComponent`.

**Behavior:**

- Text input with `matAutocomplete` triggers client search after 2+ characters
- Uses `ClientsService.getFilteredClients('displayName', 'ASC', true, searchTerm)`
- Client search results display: Client ID, Display Name, Office Name
- On selection, user picks a relationship type from the dropdown
- Optional: Link a savings account for guarantee amount (fetched via `LoansService.guarantorAccountResource`)

**Component: `ExistingClientSearchComponent`**

```typescript
// existing-client-search/existing-client-search.component.ts

import { Component, OnInit, Output, EventEmitter, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';
import { MatAutocompleteTrigger, MatAutocomplete, MatOption } from '@angular/material/autocomplete';
import { ClientsService } from 'app/clients/clients.service';
import { LoansService } from 'app/loans/loans.service';
import { GuarantorEntry, GuarantorTypeId } from 'app/loans/models/guarantor.model';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-existing-client-search',
  templateUrl: './existing-client-search.component.html',
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatAutocompleteTrigger,
    MatAutocomplete
  ]
})
export class ExistingClientSearchComponent implements OnInit {
  private formBuilder = inject(UntypedFormBuilder);
  private clientsService = inject(ClientsService);
  private loansService = inject(LoansService);

  @Output() guarantorAdded = new EventEmitter<GuarantorEntry>();

  searchForm: UntypedFormGroup;
  clientsData: any[] = [];
  relationTypes: Array<{ id: number; name: string }> = [];
  accountOptions: any[] = [];
  selectedClient: any = null;
  loanId: string;

  ngOnInit() {
    this.searchForm = this.formBuilder.group({
      clientSearch: [
        '',
        Validators.required
      ],
      clientRelationshipTypeId: [
        '',
        Validators.required
      ],
      savingsId: [''],
      amount: ['']
    });

    this.searchForm.get('clientSearch').valueChanges.subscribe((value: string) => {
      if (typeof value === 'string' && value.length >= 2) {
        this.clientsService.getFilteredClients('displayName', 'ASC', true, value).subscribe((data: any) => {
          this.clientsData = data.pageItems;
        });
      }
    });
  }

  displayClient(client: any): string | undefined {
    return client ? client.displayName : undefined;
  }

  onClientSelected(client: any) {
    this.selectedClient = client;
    this.accountOptions = [];
    if (this.loanId) {
      this.loansService.guarantorAccountResource(this.loanId, client.id).subscribe((response: any) => {
        this.accountOptions = response.accountLinkingOptions || [];
      });
    }
  }

  addGuarantor() {
    if (!this.searchForm.valid || !this.selectedClient) return;

    const formData = this.searchForm.value;
    const relationLabel = this.relationTypes.find((r) => r.id === formData.clientRelationshipTypeId)?.name || '';

    const entry: GuarantorEntry = {
      guarantorTypeId: GuarantorTypeId.EXISTING_CLIENT,
      entityId: this.selectedClient.id,
      displayName: this.selectedClient.displayName,
      clientRelationshipTypeId: formData.clientRelationshipTypeId,
      relationshipLabel: relationLabel,
      savingsId: formData.savingsId || undefined,
      amount: formData.amount || undefined,
      sourceMode: 'existing'
    };

    this.guarantorAdded.emit(entry);
    this.resetForm();
  }

  private resetForm() {
    this.searchForm.reset();
    this.selectedClient = null;
    this.clientsData = [];
    this.accountOptions = [];
  }
}
```

**Template: `existing-client-search.component.html`**

```html
<div class="layout-column gap-2px">
  <mat-form-field>
    <mat-label>{{ 'labels.inputs.Search Client' | translate }}</mat-label>
    <input matInput formControlName="clientSearch" [matAutocomplete]="clientsAutocomplete" [formGroup]="searchForm" />
    <mat-autocomplete autoActiveFirstOption #clientsAutocomplete="matAutocomplete" [displayWith]="displayClient" (optionSelected)="onClientSelected($event.option.value)">
      @for (client of clientsData; track client.id) {
      <mat-option [value]="client"> {{ client.displayName }} ({{ client.officeName }}) </mat-option>
      }
    </mat-autocomplete>
  </mat-form-field>

  @if (selectedClient) {
  <div class="client-details mat-elevation-z1 padding-1">
    <p><strong>{{ 'labels.inputs.Client ID' | translate }}:</strong> {{ selectedClient.id }}</p>
    <p><strong>{{ 'labels.inputs.name' | translate }}:</strong> {{ selectedClient.displayName }}</p>
    <p><strong>{{ 'labels.inputs.Office' | translate }}:</strong> {{ selectedClient.officeName }}</p>
  </div>

  <form [formGroup]="searchForm">
    <mat-form-field>
      <mat-label>{{ 'labels.inputs.Relationship' | translate }}</mat-label>
      <mat-select formControlName="clientRelationshipTypeId" required>
        @for (relationType of relationTypes; track relationType.id) {
        <mat-option [value]="relationType.id">{{ relationType.name }}</mat-option>
        }
      </mat-select>
    </mat-form-field>

    @if (accountOptions.length > 0) {
    <mat-form-field>
      <mat-label>{{ 'labels.inputs.Savings Account' | translate }}</mat-label>
      <mat-select formControlName="savingsId">
        @for (account of accountOptions; track account.id) {
        <mat-option [value]="account.id"> {{ account.productName }} - {{ account.accountNo }} </mat-option>
        }
      </mat-select>
    </mat-form-field>
    <mat-form-field>
      <mat-label>{{ 'labels.inputs.Amount' | translate }}</mat-label>
      <input type="number" matInput formControlName="amount" />
    </mat-form-field>
    }

    <button mat-raised-button color="primary" type="button" (click)="addGuarantor()" [disabled]="!searchForm.valid">
      <fa-icon icon="plus" class="m-r-10"></fa-icon>
      {{ 'labels.buttons.Add Guarantor' | translate }}
    </button>
  </form>
  }
</div>
```

**Fineract API payload (Existing Client):**

```json
POST /loans/{loanId}/guarantors
{
  "guarantorTypeId": 1,
  "entityId": 42,
  "clientRelationshipTypeId": 1,
  "savingsId": 15,
  "amount": 50000,
  "locale": "en",
  "dateFormat": "dd MMMM yyyy"
}
```

---

### 3.2 Mode 2: New Guarantor (External)

Inline form for creating an external guarantor who is not a client in the system.

**Component: `NewGuarantorFormComponent`**

```typescript
// new-guarantor-form/new-guarantor-form.component.ts

import { Component, OnInit, Output, EventEmitter, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';
import { SettingsService } from 'app/settings/settings.service';
import { Dates } from 'app/core/utils/dates';
import { GuarantorEntry, GuarantorTypeId } from 'app/loans/models/guarantor.model';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-new-guarantor-form',
  templateUrl: './new-guarantor-form.component.html',
  imports: [...STANDALONE_SHARED_IMPORTS]
})
export class NewGuarantorFormComponent implements OnInit {
  private formBuilder = inject(UntypedFormBuilder);
  private settingsService = inject(SettingsService);
  private dateUtils = inject(Dates);

  @Output() guarantorAdded = new EventEmitter<GuarantorEntry>();

  newGuarantorForm: UntypedFormGroup;
  relationTypes: Array<{ id: number; name: string }> = [];
  minDate = new Date(1900, 0, 1);
  maxDate = new Date();

  ngOnInit() {
    this.maxDate = this.settingsService.businessDate;
    this.newGuarantorForm = this.formBuilder.group({
      firstname: [
        '',
        Validators.required
      ],
      lastname: [
        '',
        Validators.required
      ],
      clientRelationshipTypeId: [
        '',
        Validators.required
      ],
      dob: [''],
      addressLine1: [''],
      addressLine2: [''],
      mobileNumber: [''],
      city: [''],
      zip: ['']
    });
  }

  addGuarantor() {
    if (!this.newGuarantorForm.valid) return;

    const formData = this.newGuarantorForm.value;
    const dateFormat = this.settingsService.dateFormat;
    const relationLabel = this.relationTypes.find((r) => r.id === formData.clientRelationshipTypeId)?.name || '';

    const entry: GuarantorEntry = {
      guarantorTypeId: GuarantorTypeId.EXTERNAL,
      firstname: formData.firstname,
      lastname: formData.lastname,
      displayName: `${formData.firstname} ${formData.lastname}`,
      clientRelationshipTypeId: formData.clientRelationshipTypeId,
      relationshipLabel: relationLabel,
      dob: formData.dob instanceof Date ? this.dateUtils.formatDate(formData.dob, dateFormat) : formData.dob,
      addressLine1: formData.addressLine1,
      addressLine2: formData.addressLine2,
      mobileNumber: formData.mobileNumber,
      city: formData.city,
      zip: formData.zip,
      sourceMode: 'new'
    };

    this.guarantorAdded.emit(entry);
    this.newGuarantorForm.reset();
  }
}
```

**Template: `new-guarantor-form.component.html`**

```html
<form [formGroup]="newGuarantorForm" class="layout-column">
  <div class="layout-row-wrap gap-2px responsive-column">
    <mat-form-field class="flex-48">
      <mat-label>{{ 'labels.inputs.First Name' | translate }}</mat-label>
      <input matInput required formControlName="firstname" />
      @if (newGuarantorForm.controls.firstname.hasError('required')) {
      <mat-error>
        {{ 'labels.inputs.First Name' | translate }} {{ 'labels.commons.is' | translate }}
        <strong>{{ 'labels.commons.required' | translate }}</strong>
      </mat-error>
      }
    </mat-form-field>

    <mat-form-field class="flex-48">
      <mat-label>{{ 'labels.inputs.Last Name' | translate }}</mat-label>
      <input matInput required formControlName="lastname" />
      @if (newGuarantorForm.controls.lastname.hasError('required')) {
      <mat-error>
        {{ 'labels.inputs.Last Name' | translate }} {{ 'labels.commons.is' | translate }}
        <strong>{{ 'labels.commons.required' | translate }}</strong>
      </mat-error>
      }
    </mat-form-field>
  </div>

  <mat-form-field>
    <mat-label>{{ 'labels.inputs.Relationship' | translate }}</mat-label>
    <mat-select formControlName="clientRelationshipTypeId" required>
      @for (relationType of relationTypes; track relationType.id) {
      <mat-option [value]="relationType.id">{{ relationType.name }}</mat-option>
      }
    </mat-select>
  </mat-form-field>

  <mat-form-field (click)="dobDatePicker.open()">
    <mat-label>{{ 'labels.inputs.Date Of Birth' | translate }}</mat-label>
    <input matInput [min]="minDate" [max]="maxDate" [matDatepicker]="dobDatePicker" formControlName="dob" />
    <mat-datepicker-toggle matSuffix [for]="dobDatePicker"></mat-datepicker-toggle>
    <mat-datepicker #dobDatePicker></mat-datepicker>
  </mat-form-field>

  <mat-form-field>
    <mat-label>{{ 'labels.inputs.Address Line' | translate }} 1</mat-label>
    <input matInput formControlName="addressLine1" />
  </mat-form-field>

  <mat-form-field>
    <mat-label>{{ 'labels.inputs.Phone Number' | translate }}</mat-label>
    <input matInput formControlName="mobileNumber" />
  </mat-form-field>

  <div class="layout-row-wrap gap-2px responsive-column">
    <mat-form-field class="flex-48">
      <mat-label>{{ 'labels.inputs.City' | translate }}</mat-label>
      <input matInput formControlName="city" />
    </mat-form-field>

    <mat-form-field class="flex-48">
      <mat-label>{{ 'labels.inputs.County' | translate }}</mat-label>
      <input matInput formControlName="zip" />
    </mat-form-field>
  </div>

  <button mat-raised-button color="primary" type="button" (click)="addGuarantor()" [disabled]="!newGuarantorForm.valid">
    <fa-icon icon="plus" class="m-r-10"></fa-icon>
    {{ 'labels.buttons.Add Guarantor' | translate }}
  </button>
</form>
```

**Fineract API payload (External Guarantor):**

```json
POST /loans/{loanId}/guarantors
{
  "guarantorTypeId": 3,
  "firstname": "John",
  "lastname": "Doe",
  "dob": "15 March 1985",
  "addressLine1": "123 Main Street",
  "city": "Nairobi",
  "zip": "00100",
  "mobileNumber": "0722123456",
  "clientRelationshipTypeId": 6,
  "locale": "en",
  "dateFormat": "dd MMMM yyyy"
}
```

---

### 3.3 Mode 3: Group Members

Available only when `groupId` is present in the loan context. Displays group members excluding the loan applicant for multi-select.

**Component: `GroupMemberSelectorComponent`**

```typescript
// group-member-selector/group-member-selector.component.ts

import { Component, OnInit, Input, Output, EventEmitter, inject } from '@angular/core';
import { MatTableDataSource } from '@angular/material/table';
import { MatCheckbox } from '@angular/material/checkbox';
import { GroupsService } from 'app/groups/groups.service';
import { GuarantorEntry, GuarantorTypeId, GroupMember } from 'app/loans/models/guarantor.model';
import { MatTable, MatColumnDef, MatHeaderCellDef, MatHeaderCell, MatCellDef, MatCell, MatHeaderRowDef, MatHeaderRow, MatRowDef, MatRow } from '@angular/material/table';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-group-member-selector',
  templateUrl: './group-member-selector.component.html',
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatCheckbox,
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
export class GroupMemberSelectorComponent implements OnInit {
  private groupsService = inject(GroupsService);

  @Input() groupId: number;
  @Input() excludeClientId: number;
  @Input() relationTypes: Array<{ id: number; name: string }> = [];
  @Output() guarantorsAdded = new EventEmitter<GuarantorEntry[]>();

  members: GroupMember[] = [];
  dataSource: MatTableDataSource<GroupMember>;
  displayedColumns: string[] = [
    'select',
    'id',
    'displayName',
    'officeName'
  ];
  selectAll = false;

  /** ID for the 'Group Member' relationship type */
  private groupMemberRelationId: number;

  ngOnInit() {
    this.groupMemberRelationId = this.relationTypes.find((r) => r.name === 'Group Member')?.id || this.relationTypes[this.relationTypes.length - 1]?.id;

    this.groupsService.getGroupData(this.groupId).subscribe((group: any) => {
      this.members = (group.activeClientMembers || []).filter((m: any) => m.id !== this.excludeClientId).map((m: any) => ({ ...m, selected: false }));
      this.dataSource = new MatTableDataSource(this.members);
    });
  }

  toggleSelectAll() {
    for (const member of this.members) {
      member.selected = this.selectAll;
    }
  }

  toggleMember() {
    this.selectAll = this.members.length > 0 && this.members.every((m) => m.selected);
  }

  addSelectedGuarantors() {
    const selected = this.members.filter((m) => m.selected);
    const entries: GuarantorEntry[] = selected.map((member) => ({
      guarantorTypeId: GuarantorTypeId.EXISTING_CLIENT,
      entityId: member.id,
      displayName: member.displayName,
      clientRelationshipTypeId: this.groupMemberRelationId,
      relationshipLabel: 'Group Member',
      sourceMode: 'group' as const
    }));
    this.guarantorsAdded.emit(entries);
    // Deselect all after adding
    for (const member of this.members) {
      member.selected = false;
    }
    this.selectAll = false;
  }

  get hasSelection(): boolean {
    return this.members.some((m) => m.selected);
  }
}
```

**Template: `group-member-selector.component.html`**

```html
<div class="layout-column">
  <table mat-table [dataSource]="dataSource" class="mat-elevation-z1">
    <ng-container matColumnDef="select">
      <th mat-header-cell *matHeaderCellDef>
        <mat-checkbox [(ngModel)]="selectAll" (change)="toggleSelectAll()"> </mat-checkbox>
      </th>
      <td mat-cell *matCellDef="let member">
        <mat-checkbox [(ngModel)]="member.selected" (change)="toggleMember()"> </mat-checkbox>
      </td>
    </ng-container>

    <ng-container matColumnDef="id">
      <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Client ID' | translate }}</th>
      <td mat-cell *matCellDef="let member">{{ member.id }}</td>
    </ng-container>

    <ng-container matColumnDef="displayName">
      <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.name' | translate }}</th>
      <td mat-cell *matCellDef="let member">{{ member.displayName }}</td>
    </ng-container>

    <ng-container matColumnDef="officeName">
      <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Office' | translate }}</th>
      <td mat-cell *matCellDef="let member">{{ member.officeName }}</td>
    </ng-container>

    <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
    <tr mat-row *matRowDef="let row; columns: displayedColumns"></tr>
  </table>

  <button mat-raised-button color="primary" type="button" class="margin-t" (click)="addSelectedGuarantors()" [disabled]="!hasSelection">
    <fa-icon icon="plus" class="m-r-10"></fa-icon>
    {{ 'labels.buttons.Add Selected as Guarantors' | translate }}
  </button>
</div>
```

---

## 4. Guarantor List Display

**Component: `GuarantorListComponent`**

Displays all added guarantors in a summary table with remove action.

```typescript
// guarantor-list/guarantor-list.component.ts

import { Component, Input, Output, EventEmitter, inject } from '@angular/core';
import { MatDialog } from '@angular/material/dialog';
import { DeleteDialogComponent } from 'app/shared/delete-dialog/delete-dialog.component';
import { GuarantorEntry } from 'app/loans/models/guarantor.model';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { MatIconButton } from '@angular/material/button';
import { MatTable, MatColumnDef, MatHeaderCellDef, MatHeaderCell, MatCellDef, MatCell, MatHeaderRowDef, MatHeaderRow, MatRowDef, MatRow } from '@angular/material/table';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-guarantor-list',
  templateUrl: './guarantor-list.component.html',
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
    MatRow
  ]
})
export class GuarantorListComponent {
  private dialog = inject(MatDialog);

  @Input() guarantors: GuarantorEntry[] = [];
  @Input() minGuarantors = 0;
  @Output() guarantorRemoved = new EventEmitter<number>();

  displayedColumns: string[] = [
    'name',
    'type',
    'relationship',
    'status',
    'action'
  ];

  getTypeName(entry: GuarantorEntry): string {
    switch (entry.sourceMode) {
      case 'existing':
        return 'Existing Client';
      case 'new':
        return 'New Guarantor';
      case 'group':
        return 'Group Member';
      default:
        return 'Unknown';
    }
  }

  removeGuarantor(index: number) {
    const guarantor = this.guarantors[index];
    const deleteRef = this.dialog.open(DeleteDialogComponent, {
      data: { deleteContext: `guarantor ${guarantor.displayName}` }
    });
    deleteRef.afterClosed().subscribe((response: any) => {
      if (response.delete) {
        this.guarantorRemoved.emit(index);
      }
    });
  }

  get isValid(): boolean {
    return this.guarantors.length >= this.minGuarantors;
  }

  get validationMessage(): string {
    if (this.guarantors.length < this.minGuarantors) {
      return `Minimum ${this.minGuarantors} guarantor(s) required. Currently ${this.guarantors.length} added.`;
    }
    return '';
  }
}
```

**Template: `guarantor-list.component.html`**

```html
<h4 class="mat-h4">{{ 'labels.heading.Added Guarantors' | translate }} ({{ guarantors.length }})</h4>

@if (validationMessage) {
<p class="mat-error">{{ validationMessage }}</p>
}

<table mat-table [dataSource]="guarantors" class="mat-elevation-z1" [hidden]="guarantors.length === 0">
  <ng-container matColumnDef="name">
    <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.name' | translate }}</th>
    <td mat-cell *matCellDef="let g">{{ g.displayName }}</td>
  </ng-container>

  <ng-container matColumnDef="type">
    <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Type' | translate }}</th>
    <td mat-cell *matCellDef="let g">{{ getTypeName(g) }}</td>
  </ng-container>

  <ng-container matColumnDef="relationship">
    <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Relationship' | translate }}</th>
    <td mat-cell *matCellDef="let g">{{ g.relationshipLabel }}</td>
  </ng-container>

  <ng-container matColumnDef="status">
    <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Status' | translate }}</th>
    <td mat-cell *matCellDef="let g">{{ g.status || 'Pending' }}</td>
  </ng-container>

  <ng-container matColumnDef="action">
    <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Actions' | translate }}</th>
    <td mat-cell *matCellDef="let g; let i = index">
      <button mat-icon-button color="warn" (click)="removeGuarantor(i)">
        <fa-icon icon="trash"></fa-icon>
      </button>
    </td>
  </ng-container>

  <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
  <tr mat-row *matRowDef="let row; columns: displayedColumns"></tr>
</table>
```

---

## 5. Parent Step Component

**Component: `LoansAccountGuarantorStepComponent`**

```typescript
// loans-account-guarantor-step.component.ts

import { Component, OnInit, Input, inject } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { MatTabGroup, MatTab } from '@angular/material/tabs';
import { MatStepperPrevious, MatStepperNext } from '@angular/material/stepper';
import { MatDivider } from '@angular/material/divider';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { LoansService } from 'app/loans/loans.service';
import { GuarantorEntry, GuarantorTemplate } from 'app/loans/models/guarantor.model';
import { ExistingClientSearchComponent } from './existing-client-search/existing-client-search.component';
import { NewGuarantorFormComponent } from './new-guarantor-form/new-guarantor-form.component';
import { GroupMemberSelectorComponent } from './group-member-selector/group-member-selector.component';
import { GuarantorListComponent } from './guarantor-list/guarantor-list.component';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-loans-account-guarantor-step',
  templateUrl: './loans-account-guarantor-step.component.html',
  styleUrls: ['./loans-account-guarantor-step.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatTabGroup,
    MatTab,
    MatStepperPrevious,
    MatStepperNext,
    MatDivider,
    FaIconComponent,
    ExistingClientSearchComponent,
    NewGuarantorFormComponent,
    GroupMemberSelectorComponent,
    GuarantorListComponent
  ]
})
export class LoansAccountGuarantorStepComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private loansService = inject(LoansService);

  @Input() loansAccountProductTemplate: any;
  @Input() loansAccountFormValid: boolean;

  /** Group context - null when loan is not within a group */
  @Input() groupId: number | null = null;
  @Input() clientId: number | null = null;

  /** Minimum guarantors from loan product configuration */
  @Input() minGuarantors = 0;

  guarantors: GuarantorEntry[] = [];
  guarantorTemplate: GuarantorTemplate;
  loanId: string;

  ngOnInit() {
    this.loanId = this.route.snapshot.params['loanId'];

    // Load guarantor template for relationship types and guarantor type options
    if (this.loanId) {
      this.loansService.getGuarantorTemplate(this.loanId).subscribe((template: any) => {
        this.guarantorTemplate = template;
      });
    }
  }

  get showGroupMembersTab(): boolean {
    return this.groupId != null;
  }

  onGuarantorAdded(entry: GuarantorEntry) {
    this.guarantors = [
      ...this.guarantors,
      entry
    ];
  }

  onGuarantorsAdded(entries: GuarantorEntry[]) {
    // Filter out duplicates by entityId for existing/group members
    const existingIds = new Set(this.guarantors.filter((g) => g.entityId).map((g) => g.entityId));
    const newEntries = entries.filter((e) => !e.entityId || !existingIds.has(e.entityId));
    this.guarantors = [
      ...this.guarantors,
      ...newEntries
    ];
  }

  onGuarantorRemoved(index: number) {
    this.guarantors = this.guarantors.filter((_, i) => i !== index);
  }

  get isValid(): boolean {
    return this.guarantors.length >= this.minGuarantors;
  }

  /** Returns guarantor data for the loan submission payload */
  get loansAccountGuarantors(): { guarantors: GuarantorEntry[] } {
    return { guarantors: this.guarantors };
  }
}
```

**Template: `loans-account-guarantor-step.component.html`**

```html
<div class="layout-column gap-2px">
  <h3 class="mat-h3">{{ 'labels.heading.Guarantors' | translate }}</h3>

  <mat-tab-group>
    <mat-tab [label]="'labels.inputs.Existing Client' | translate">
      <div class="padding-1">
        <mifosx-existing-client-search [loanId]="loanId" [relationTypes]="guarantorTemplate?.allowedClientRelationshipTypes || []" (guarantorAdded)="onGuarantorAdded($event)"> </mifosx-existing-client-search>
      </div>
    </mat-tab>

    <mat-tab [label]="'labels.inputs.New Guarantor' | translate">
      <div class="padding-1">
        <mifosx-new-guarantor-form [relationTypes]="guarantorTemplate?.allowedClientRelationshipTypes || []" (guarantorAdded)="onGuarantorAdded($event)"> </mifosx-new-guarantor-form>
      </div>
    </mat-tab>

    @if (showGroupMembersTab) {
    <mat-tab [label]="'labels.inputs.Group Members' | translate">
      <div class="padding-1">
        <mifosx-group-member-selector [groupId]="groupId" [excludeClientId]="clientId" [relationTypes]="guarantorTemplate?.allowedClientRelationshipTypes || []" (guarantorsAdded)="onGuarantorsAdded($event)"> </mifosx-group-member-selector>
      </div>
    </mat-tab>
    }
  </mat-tab-group>

  <mat-divider></mat-divider>

  <mifosx-guarantor-list [guarantors]="guarantors" [minGuarantors]="minGuarantors" (guarantorRemoved)="onGuarantorRemoved($event)"> </mifosx-guarantor-list>
</div>

<div class="layout-row responsive-column align-center gap-2px margin-t stepper-buttons">
  <button mat-raised-button matStepperPrevious>
    <fa-icon icon="arrow-left" class="m-r-10"></fa-icon>
    {{ 'labels.buttons.Previous' | translate }}
  </button>
  <button mat-raised-button matStepperNext [disabled]="!isValid">
    {{ 'labels.buttons.Next' | translate }}
    <fa-icon icon="arrow-right" class="m-l-10"></fa-icon>
  </button>
</div>
```

---

## 6. Integration with Loan Creation Stepper

### 6.1 Stepper Parent Update

In `src/app/loans/loans-account-stepper/`, the parent stepper component must include the new step. The stepper currently contains:

1. Details Step
2. Terms Step
3. Charges Step
4. Schedule Step (optional)
5. Datatable Step (optional)
6. Preview Step

Insert the Guarantor Step between Charges and Schedule:

```typescript
// In the parent stepper component, add:
import { LoansAccountGuarantorStepComponent } from './loans-account-guarantor-step/loans-account-guarantor-step.component';

// In imports array:
LoansAccountGuarantorStepComponent

// In the component class, add:
@ViewChild(LoansAccountGuarantorStepComponent) guarantorStep: LoansAccountGuarantorStepComponent;
```

In the parent stepper template, add the step:

```html
<mat-step [stepControl]="guarantorStepControl" label="{{ 'labels.heading.Guarantors' | translate }}">
  <mifosx-loans-account-guarantor-step [loansAccountProductTemplate]="loansAccountProductTemplate" [loansAccountFormValid]="loansAccountFormValid" [groupId]="groupId" [clientId]="clientId" [minGuarantors]="loansAccountProductTemplate?.minGuarantors || 0"> </mifosx-loans-account-guarantor-step>
</mat-step>
```

### 6.2 Loan Submission Integration

When the loan is submitted, guarantors are posted individually after loan creation since the Fineract API requires a `loanId`:

```typescript
// In the loan submission handler:
async submitLoan() {
  // Step 1: Create the loan
  const loanResponse = await this.loansService.createLoansAccount(loanPayload).toPromise();
  const loanId = loanResponse.loanId;

  // Step 2: Add guarantors sequentially
  const guarantors = this.guarantorStep.loansAccountGuarantors.guarantors;
  for (const guarantor of guarantors) {
    const payload = this.buildGuarantorPayload(guarantor);
    await this.loansService.createNewGuarantor(loanId.toString(), payload).toPromise();
  }
}

private buildGuarantorPayload(entry: GuarantorEntry): any {
  const locale = this.settingsService.language.code;
  const dateFormat = this.settingsService.dateFormat;

  const base: any = {
    guarantorTypeId: entry.guarantorTypeId,
    clientRelationshipTypeId: entry.clientRelationshipTypeId,
    locale,
    dateFormat
  };

  if (entry.guarantorTypeId === GuarantorTypeId.EXISTING_CLIENT) {
    base.entityId = entry.entityId;
    if (entry.savingsId) {
      base.savingsId = entry.savingsId;
      base.amount = entry.amount;
    }
  } else if (entry.guarantorTypeId === GuarantorTypeId.EXTERNAL) {
    base.firstname = entry.firstname;
    base.lastname = entry.lastname;
    if (entry.dob) base.dob = entry.dob;
    if (entry.addressLine1) base.addressLine1 = entry.addressLine1;
    if (entry.addressLine2) base.addressLine2 = entry.addressLine2;
    if (entry.city) base.city = entry.city;
    if (entry.zip) base.zip = entry.zip;
    if (entry.mobileNumber) base.mobileNumber = entry.mobileNumber;
  }

  return base;
}
```

---

## 7. Fineract API Reference

| Endpoint                                                     | Method | Purpose                                             |
| ------------------------------------------------------------ | ------ | --------------------------------------------------- |
| `/loans/{loanId}/guarantors/template`                        | GET    | Fetch guarantor type options and relationship types |
| `/loans/{loanId}/guarantors`                                 | POST   | Create a new guarantor for a loan                   |
| `/loans/{loanId}/guarantors`                                 | GET    | List all guarantors for a loan                      |
| `/loans/{loanId}/guarantors/{guarantorId}`                   | DELETE | Remove a guarantor                                  |
| `/loans/{loanId}/guarantors/accounts/template?clientId={id}` | GET    | Fetch savings account options for a client          |

### Existing LoansService Methods (in `src/app/loans/loans.service.ts`)

```typescript
getGuarantorTemplate(loanId: string): Observable<any>
createNewGuarantor(loanId: string, data: any): Observable<any>
deleteGuarantor(loanId: any, guarantorId: any): Observable<any>
guarantorAccountResource(loanId: string, clientId: any): Observable<any>
```

No new service methods are needed. The existing `LoansService` already covers all required API calls.

---

## 8. Form Validation Rules

| Field                    | Rule                                      | Context               |
| ------------------------ | ----------------------------------------- | --------------------- |
| Guarantor count          | `>= minGuarantors` from loan product      | Step-level validation |
| Client search (existing) | Required, must select from autocomplete   | Existing Client mode  |
| Relationship type        | Required for all modes                    | All modes             |
| First Name (new)         | Required                                  | New Guarantor mode    |
| Last Name (new)          | Required                                  | New Guarantor mode    |
| Date of Birth (new)      | Optional, must be valid date if provided  | New Guarantor mode    |
| Phone Number (new)       | Optional, numeric format                  | New Guarantor mode    |
| Group member selection   | At least 1 selected to enable add button  | Group Members mode    |
| Duplicate check          | No duplicate `entityId` in guarantor list | Existing/Group modes  |

---

## 9. Permissions

The existing Fineract permission `CREATE_GUARANTOR` should be checked. Use the `*mifosxHasPermission="'CREATE_GUARANTOR'"` directive on the add/submit buttons, consistent with the existing `create-guarantor.component.html`.
