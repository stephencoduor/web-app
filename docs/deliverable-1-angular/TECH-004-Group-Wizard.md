# TECH-004: Group Creation Wizard Specification

**Status:** Draft
**Priority:** Medium-High
**Estimated Effort:** 3-5 days
**Dependencies:** TECH-002 (MsaccoWizardComponent)

---

## Overview

Convert the current single-page group creation form (`src/app/groups/create-group/create-group.component.ts`) into a 3-step wizard using Angular Material stepper.

### Current Implementation Analysis

**File:** `src/app/groups/create-group/create-group.component.ts`

The current `CreateGroupComponent`:

- Uses a single `UntypedFormGroup` (`groupForm`) with fields: `name`, `officeId`, `submittedOnDate`, `staffId`, `externalId`, `active` (with conditional `activationDate`)
- Office data comes from route resolver (`OfficesResolver`)
- Staff data loads dynamically when office is selected (`groupService.getStaff(officeId)`)
- Client search uses `clientsService.getFilteredClients()` with office filtering
- Client members stored in a `clientMembers: any[]` array
- Submits via `POST /groups` through `groupService.createGroup()`

**Route configuration** in `src/app/groups/groups-routing.module.ts`:

```typescript
{
  path: 'create',
  component: CreateGroupComponent,
  data: { title: 'Create Group', breadcrumb: 'Create', routeParamBreadcrumb: false },
  resolve: { offices: OfficesResolver }
}
```

---

## Target Architecture

**New directory:** `src/app/groups/group-stepper/`

The wizard will use the `MsaccoWizardComponent` (TECH-002) wrapper or directly use `mat-stepper` for finer control. Given the form validation requirements per step, direct `mat-stepper` is recommended with the M-SACCO styling applied via CSS.

### Component Tree

```
GroupStepperComponent                    (orchestrator)
  +-- GroupStepInfoComponent             (Step 1: Group Info form)
  +-- GroupStepClientsComponent          (Step 2: Select Clients)
  +-- GroupStepOverviewComponent         (Step 3: Read-only summary)
```

---

## Step 1: Group Info

### Fields

| Field               | Control Type       | Required | Validation                                                | Notes                                                       |
| ------------------- | ------------------ | -------- | --------------------------------------------------------- | ----------------------------------------------------------- |
| Group Name          | `<input matInput>` | Yes      | `Validators.required`, `Validators.pattern('(^[A-z]).*')` | Must start with a letter                                    |
| Branch Name         | `<mat-select>`     | Yes      | `Validators.required`                                     | Loaded from `OfficesResolver`                               |
| Loan Officer        | `<mat-select>`     | No       | None                                                      | Depends on Branch selection; disabled until branch selected |
| Registration Number | `<input matInput>` | No       | None                                                      | Maps to `externalId`                                        |
| Registration Date   | `<mat-datepicker>` | Yes      | `Validators.required`                                     | Maps to `submittedOnDate`; max date = business date         |
| Meeting Location    | `<input matInput>` | No       | None                                                      | M-SACCO specific field                                      |
| Meeting Days        | `<mat-select>`     | No       | None                                                      | Options: Monday-Sunday                                      |
| Meeting Frequency   | `<mat-select>`     | No       | None                                                      | Options: Weekly, Biweekly, Monthly                          |

### Dependencies

When the **Branch Name** selection changes:

1. Call `groupService.getStaff(officeId)` to load loan officers for that branch
2. If no staff available, disable the Loan Officer dropdown
3. Reset the Loan Officer selection
4. The selected branch ID will be passed to Step 2 for client filtering

### Component

```typescript
// src/app/groups/group-stepper/group-step-info/group-step-info.component.ts

import { Component, OnInit, Input, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators, ReactiveFormsModule } from '@angular/forms';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { GroupsService } from '../../groups.service';
import { SettingsService } from 'app/settings/settings.service';

@Component({
  selector: 'mifosx-group-step-info',
  templateUrl: './group-step-info.component.html',
  styleUrls: ['./group-step-info.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    ReactiveFormsModule
  ]
})
export class GroupStepInfoComponent implements OnInit {
  private fb = inject(UntypedFormBuilder);
  private groupService = inject(GroupsService);
  private settingsService = inject(SettingsService);

  /** Office data from resolver */
  @Input() officeData: any[] = [];

  /** The form group (passed in from parent for validation gating) */
  groupInfoForm: UntypedFormGroup;

  /** Staff options loaded based on selected office */
  staffData: any[] = [];

  /** Meeting day options */
  meetingDays: string[] = [
    'Monday',
    'Tuesday',
    'Wednesday',
    'Thursday',
    'Friday',
    'Saturday',
    'Sunday'
  ];

  /** Meeting frequency options */
  meetingFrequencies: string[] = [
    'Weekly',
    'Biweekly',
    'Monthly'
  ];

  /** Max date for datepicker */
  maxDate: Date;
  minDate = new Date(2000, 0, 1);

  ngOnInit(): void {
    this.maxDate = this.settingsService.businessDate;
    this.createForm();
    this.setupDependencies();
  }

  private createForm(): void {
    this.groupInfoForm = this.fb.group({
      name: [
        '',
        [
          Validators.required,
          Validators.pattern('(^[A-z]).*')
        ]
      ],
      officeId: [
        '',
        Validators.required
      ],
      staffId: [{ value: '', disabled: true }],
      externalId: [''],
      submittedOnDate: [
        this.settingsService.businessDate,
        Validators.required
      ],
      meetingLocation: [''],
      meetingDay: [''],
      meetingFrequency: ['']
    });
  }

  private setupDependencies(): void {
    this.groupInfoForm.get('officeId').valueChanges.subscribe((officeId: number) => {
      // Reset staff selection
      this.groupInfoForm.get('staffId').setValue('');
      this.groupInfoForm.get('staffId').disable();
      this.staffData = [];

      if (officeId) {
        this.groupService.getStaff(officeId).subscribe((data: any) => {
          this.staffData = data?.staffOptions || [];
          if (this.staffData.length > 0) {
            this.groupInfoForm.get('staffId').enable();
          }
        });
      }
    });
  }

  /** Expose the selected office ID for Step 2 client filtering */
  get selectedOfficeId(): number {
    return this.groupInfoForm.get('officeId')?.value;
  }
}
```

### Template

```html
<!-- src/app/groups/group-stepper/group-step-info/group-step-info.component.html -->

<div class="step-info-form">
  <h3 class="step-title">Group Information</h3>
  <p class="step-description">Enter the basic details for the new group.</p>

  <form [formGroup]="groupInfoForm" class="info-form">
    <div class="form-row">
      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Group Name *</mat-label>
        <input matInput formControlName="name" placeholder="Enter group name" />
        @if (groupInfoForm.get('name')?.hasError('required') && groupInfoForm.get('name')?.touched) {
        <mat-error>Group name is required</mat-error>
        } @if (groupInfoForm.get('name')?.hasError('pattern')) {
        <mat-error>Group name must start with a letter</mat-error>
        }
      </mat-form-field>

      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Branch Name *</mat-label>
        <mat-select formControlName="officeId">
          @for (office of officeData; track office.id) {
          <mat-option [value]="office.id">{{ office.name }}</mat-option>
          }
        </mat-select>
        @if (groupInfoForm.get('officeId')?.hasError('required') && groupInfoForm.get('officeId')?.touched) {
        <mat-error>Branch is required</mat-error>
        }
      </mat-form-field>
    </div>

    <div class="form-row">
      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Loan Officer</mat-label>
        <mat-select formControlName="staffId">
          <mat-option value="">-- None --</mat-option>
          @for (staff of staffData; track staff.id) {
          <mat-option [value]="staff.id">{{ staff.displayName }}</mat-option>
          }
        </mat-select>
        @if (staffData.length === 0 && groupInfoForm.get('officeId')?.value) {
        <mat-hint>No loan officers available for this branch</mat-hint>
        }
      </mat-form-field>

      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Registration Number</mat-label>
        <input matInput formControlName="externalId" placeholder="Optional" />
      </mat-form-field>
    </div>

    <div class="form-row">
      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Registration Date *</mat-label>
        <input matInput [matDatepicker]="regDatePicker" formControlName="submittedOnDate" />
        <mat-datepicker-toggle matSuffix [for]="regDatePicker"></mat-datepicker-toggle>
        <mat-datepicker #regDatePicker></mat-datepicker>
        @if (groupInfoForm.get('submittedOnDate')?.hasError('required') && groupInfoForm.get('submittedOnDate')?.touched) {
        <mat-error>Registration date is required</mat-error>
        }
      </mat-form-field>

      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Meeting Location</mat-label>
        <input matInput formControlName="meetingLocation" placeholder="Optional" />
      </mat-form-field>
    </div>

    <div class="form-row">
      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Meeting Days</mat-label>
        <mat-select formControlName="meetingDay">
          <mat-option value="">-- None --</mat-option>
          @for (day of meetingDays; track day) {
          <mat-option [value]="day">{{ day }}</mat-option>
          }
        </mat-select>
      </mat-form-field>

      <mat-form-field appearance="outline" class="form-field">
        <mat-label>Meeting Frequency</mat-label>
        <mat-select formControlName="meetingFrequency">
          <mat-option value="">-- None --</mat-option>
          @for (freq of meetingFrequencies; track freq) {
          <mat-option [value]="freq">{{ freq }}</mat-option>
          }
        </mat-select>
      </mat-form-field>
    </div>
  </form>
</div>
```

### Styles

```scss
// src/app/groups/group-stepper/group-step-info/group-step-info.component.scss

.step-info-form {
  max-width: 800px;
}

.step-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}

.step-description {
  font-size: 13px;
  color: #666;
  margin-bottom: 24px;
}

.info-form {
  display: flex;
  flex-direction: column;
  gap: 0;
}

.form-row {
  display: flex;
  gap: 16px;

  @media (max-width: 600px) {
    flex-direction: column;
    gap: 0;
  }
}

.form-field {
  flex: 1;
}
```

---

## Step 2: Select Clients

### UI Behavior

1. A search input at the top allows filtering clients by name
2. Search triggers when 2+ characters are entered
3. Client results appear in a table below the search (Client ID, Client Name)
4. Clicking a row or the "Add" button adds the client to the selected list
5. The selected clients list appears on the right (or below on mobile) with remove buttons
6. Only clients from the selected branch (Step 1) are shown
7. Validation: at least 1 client must be selected to proceed

### Fineract API

```
GET /fineract-provider/api/v1/clients?officeId={branchId}&orphansOnly=true&displayName={searchTerm}&sortOrder=ASC&orderBy=displayName
```

This uses the existing `clientsService.getFilteredClients()` method already used by the current `CreateGroupComponent`.

### Component

```typescript
// src/app/groups/group-stepper/group-step-clients/group-step-clients.component.ts

import { Component, OnInit, Input, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, UntypedFormControl, Validators, ReactiveFormsModule } from '@angular/forms';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { ClientsService } from 'app/clients/clients.service';
import { MatTableDataSource } from '@angular/material/table';

export interface SelectedClient {
  id: number;
  displayName: string;
  accountNo: string;
}

@Component({
  selector: 'mifosx-group-step-clients',
  templateUrl: './group-step-clients.component.html',
  styleUrls: ['./group-step-clients.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    ReactiveFormsModule
  ]
})
export class GroupStepClientsComponent implements OnInit {
  private fb = inject(UntypedFormBuilder);
  private clientsService = inject(ClientsService);

  /** Office ID from Step 1, used to filter clients */
  @Input() officeId: number;

  /** Form group for step validation (must have at least 1 client) */
  clientsForm: UntypedFormGroup;

  /** Search input control */
  searchControl = new UntypedFormControl('');

  /** Search results */
  searchResults: any[] = [];
  searchResultsDataSource = new MatTableDataSource<any>();
  searchColumns = [
    'accountNo',
    'displayName',
    'actions'
  ];

  /** Selected client members */
  selectedClients: SelectedClient[] = [];

  /** Loading state for search */
  isSearching = false;

  ngOnInit(): void {
    // The form group holds a hidden validator that checks selectedClients.length > 0
    this.clientsForm = this.fb.group({
      _clientCount: [
        0,
        Validators.min(1)
      ]
    });

    this.setupSearch();
  }

  private setupSearch(): void {
    this.searchControl.valueChanges.subscribe((value: string) => {
      if (typeof value === 'string' && value.length >= 2 && this.officeId) {
        this.isSearching = true;
        this.clientsService.getFilteredClients('displayName', 'ASC', true, value, this.officeId).subscribe({
          next: (data: any) => {
            // Filter out already-selected clients
            const selectedIds = new Set(this.selectedClients.map((c) => c.id));
            this.searchResults = (data.pageItems || []).filter((c: any) => !selectedIds.has(c.id));
            this.searchResultsDataSource.data = this.searchResults;
            this.isSearching = false;
          },
          error: () => {
            this.isSearching = false;
          }
        });
      } else {
        this.searchResults = [];
        this.searchResultsDataSource.data = [];
      }
    });
  }

  addClient(client: any): void {
    if (!this.selectedClients.some((c) => c.id === client.id)) {
      this.selectedClients.push({
        id: client.id,
        displayName: client.displayName,
        accountNo: client.accountNo
      });
      this.updateValidation();

      // Remove from search results
      this.searchResults = this.searchResults.filter((c) => c.id !== client.id);
      this.searchResultsDataSource.data = this.searchResults;
    }
  }

  removeClient(index: number): void {
    this.selectedClients.splice(index, 1);
    this.updateValidation();
  }

  private updateValidation(): void {
    this.clientsForm.get('_clientCount')?.setValue(this.selectedClients.length);
    this.clientsForm.get('_clientCount')?.markAsTouched();
  }

  /** Get the array of client IDs for submission */
  getClientMemberIds(): number[] {
    return this.selectedClients.map((c) => c.id);
  }
}
```

### Template

```html
<!-- src/app/groups/group-stepper/group-step-clients/group-step-clients.component.html -->

<div class="step-clients">
  <h3 class="step-title">Select Group Members</h3>
  <p class="step-description">
    Search and add clients to this group. At least one client is required. @if (officeId) { Showing clients from the selected branch. } @else {
    <strong>Please select a branch in Step 1 first.</strong>
    }
  </p>

  <div class="clients-layout">
    <!-- Search and Results -->
    <div class="search-panel">
      <mat-form-field appearance="outline" class="search-field">
        <mat-label>Search clients by name</mat-label>
        <mat-icon matPrefix>search</mat-icon>
        <input matInput [formControl]="searchControl" placeholder="Type at least 2 characters..." />
        @if (isSearching) {
        <mat-spinner matSuffix diameter="20"></mat-spinner>
        }
      </mat-form-field>

      @if (searchResults.length > 0) {
      <div class="search-results">
        <table mat-table [dataSource]="searchResultsDataSource" class="results-table">
          <ng-container matColumnDef="accountNo">
            <th mat-header-cell *matHeaderCellDef>Client ID</th>
            <td mat-cell *matCellDef="let row">{{ row.accountNo }}</td>
          </ng-container>
          <ng-container matColumnDef="displayName">
            <th mat-header-cell *matHeaderCellDef>Client Name</th>
            <td mat-cell *matCellDef="let row">{{ row.displayName }}</td>
          </ng-container>
          <ng-container matColumnDef="actions">
            <th mat-header-cell *matHeaderCellDef></th>
            <td mat-cell *matCellDef="let row">
              <button mat-icon-button color="primary" (click)="addClient(row)" matTooltip="Add to group">
                <mat-icon>add_circle</mat-icon>
              </button>
            </td>
          </ng-container>
          <tr mat-header-row *matHeaderRowDef="searchColumns"></tr>
          <tr mat-row *matRowDef="let row; columns: searchColumns" (click)="addClient(row)" class="clickable-row"></tr>
        </table>
      </div>
      } @if (searchControl.value?.length >= 2 && searchResults.length === 0 && !isSearching) {
      <div class="no-results">
        <mat-icon>search_off</mat-icon>
        <span>No matching clients found in this branch.</span>
      </div>
      }
    </div>

    <!-- Selected Clients List -->
    <div class="selected-panel">
      <div class="selected-header">
        <h4>Selected Members ({{ selectedClients.length }})</h4>
      </div>

      @if (selectedClients.length > 0) {
      <div class="selected-list">
        @for (client of selectedClients; track client.id; let i = $index) {
        <div class="selected-client">
          <div class="client-info">
            <span class="client-name">{{ client.displayName }}</span>
            <span class="client-id">{{ client.accountNo }}</span>
          </div>
          <button mat-icon-button color="warn" (click)="removeClient(i)" matTooltip="Remove from group">
            <mat-icon>remove_circle</mat-icon>
          </button>
        </div>
        }
      </div>
      } @else {
      <div class="empty-selected">
        <mat-icon>group_add</mat-icon>
        <p>No clients selected yet.</p>
        <p class="hint">Use the search to find and add clients.</p>
      </div>
      } @if (clientsForm.get('_clientCount')?.touched && selectedClients.length === 0) {
      <mat-error class="validation-error"> At least one client must be selected. </mat-error>
      }
    </div>
  </div>
</div>
```

### Styles

```scss
// src/app/groups/group-stepper/group-step-clients/group-step-clients.component.scss

.step-clients {
  max-width: 1000px;
}

.step-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}

.step-description {
  font-size: 13px;
  color: #666;
  margin-bottom: 24px;
}

.clients-layout {
  display: flex;
  gap: 24px;

  @media (max-width: 768px) {
    flex-direction: column;
  }
}

.search-panel {
  flex: 1;
}

.search-field {
  width: 100%;
}

.search-results {
  max-height: 400px;
  overflow-y: auto;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
}

.results-table {
  width: 100%;

  .clickable-row {
    cursor: pointer;
    &:hover {
      background-color: #f0f7ff;
    }
  }
}

.no-results {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 24px;
  color: #999;
  font-size: 14px;
}

.selected-panel {
  width: 320px;
  min-width: 280px;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  background: #fafafa;

  @media (max-width: 768px) {
    width: 100%;
  }
}

.selected-header {
  padding: 12px 16px;
  border-bottom: 1px solid #e0e0e0;
  background: #f5f5f5;

  h4 {
    margin: 0;
    font-size: 14px;
    font-weight: 600;
    color: #333;
  }
}

.selected-list {
  max-height: 400px;
  overflow-y: auto;
}

.selected-client {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 16px;
  border-bottom: 1px solid #eee;

  &:last-child {
    border-bottom: none;
  }

  .client-info {
    display: flex;
    flex-direction: column;

    .client-name {
      font-size: 13px;
      font-weight: 500;
      color: #333;
    }
    .client-id {
      font-size: 12px;
      color: #999;
    }
  }
}

.empty-selected {
  padding: 32px 16px;
  text-align: center;
  color: #999;

  mat-icon {
    font-size: 40px;
    width: 40px;
    height: 40px;
    color: #ccc;
  }
  p {
    margin: 8px 0 0;
    font-size: 13px;
  }
  .hint {
    font-size: 12px;
    color: #bbb;
  }
}

.validation-error {
  padding: 8px 16px;
  font-size: 12px;
}
```

---

## Step 3: Overview

### Purpose

Read-only summary of all data entered in Steps 1 and 2. The user reviews before submitting.

### Component

```typescript
// src/app/groups/group-stepper/group-step-overview/group-step-overview.component.ts

import { Component, Input } from '@angular/core';
import { UntypedFormGroup } from '@angular/forms';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { SelectedClient } from '../group-step-clients/group-step-clients.component';

@Component({
  selector: 'mifosx-group-step-overview',
  templateUrl: './group-step-overview.component.html',
  styleUrls: ['./group-step-overview.component.scss'],
  standalone: true,
  imports: [...STANDALONE_SHARED_IMPORTS]
})
export class GroupStepOverviewComponent {
  /** Group info form from Step 1 (read values) */
  @Input() groupInfoForm: UntypedFormGroup;

  /** Selected clients from Step 2 */
  @Input() selectedClients: SelectedClient[] = [];

  /** Office name for display (resolved from officeId) */
  @Input() officeName: string = '';

  /** Staff name for display (resolved from staffId) */
  @Input() staffName: string = '';
}
```

### Template

```html
<!-- src/app/groups/group-stepper/group-step-overview/group-step-overview.component.html -->

<div class="step-overview">
  <h3 class="step-title">Review Group Details</h3>
  <p class="step-description">Please review the group information before submitting.</p>

  <!-- Group Info Section -->
  <div class="overview-section">
    <h4 class="section-title">Group Information</h4>
    <div class="overview-grid">
      <div class="overview-item">
        <span class="item-label">Group Name</span>
        <span class="item-value">{{ groupInfoForm.get('name')?.value }}</span>
      </div>
      <div class="overview-item">
        <span class="item-label">Branch</span>
        <span class="item-value">{{ officeName }}</span>
      </div>
      <div class="overview-item">
        <span class="item-label">Loan Officer</span>
        <span class="item-value">{{ staffName || 'Not assigned' }}</span>
      </div>
      <div class="overview-item">
        <span class="item-label">Registration Number</span>
        <span class="item-value">{{ groupInfoForm.get('externalId')?.value || 'N/A' }}</span>
      </div>
      <div class="overview-item">
        <span class="item-label">Registration Date</span>
        <span class="item-value">{{ groupInfoForm.get('submittedOnDate')?.value | date }}</span>
      </div>
      @if (groupInfoForm.get('meetingLocation')?.value) {
      <div class="overview-item">
        <span class="item-label">Meeting Location</span>
        <span class="item-value">{{ groupInfoForm.get('meetingLocation')?.value }}</span>
      </div>
      } @if (groupInfoForm.get('meetingDay')?.value) {
      <div class="overview-item">
        <span class="item-label">Meeting Day</span>
        <span class="item-value">{{ groupInfoForm.get('meetingDay')?.value }}</span>
      </div>
      } @if (groupInfoForm.get('meetingFrequency')?.value) {
      <div class="overview-item">
        <span class="item-label">Meeting Frequency</span>
        <span class="item-value">{{ groupInfoForm.get('meetingFrequency')?.value }}</span>
      </div>
      }
    </div>
  </div>

  <!-- Client Members Section -->
  <div class="overview-section">
    <h4 class="section-title">Group Members ({{ selectedClients.length }})</h4>
    <table mat-table [dataSource]="selectedClients" class="members-table">
      <ng-container matColumnDef="index">
        <th mat-header-cell *matHeaderCellDef>#</th>
        <td mat-cell *matCellDef="let row; let i = index">{{ i + 1 }}</td>
      </ng-container>
      <ng-container matColumnDef="accountNo">
        <th mat-header-cell *matHeaderCellDef>Client ID</th>
        <td mat-cell *matCellDef="let row">{{ row.accountNo }}</td>
      </ng-container>
      <ng-container matColumnDef="displayName">
        <th mat-header-cell *matHeaderCellDef>Client Name</th>
        <td mat-cell *matCellDef="let row">{{ row.displayName }}</td>
      </ng-container>
      <tr mat-header-row *matHeaderRowDef="['index', 'accountNo', 'displayName']"></tr>
      <tr mat-row *matRowDef="let row; columns: ['index', 'accountNo', 'displayName']"></tr>
    </table>
  </div>
</div>
```

### Styles

```scss
// src/app/groups/group-stepper/group-step-overview/group-step-overview.component.scss

.step-overview {
  max-width: 800px;
}

.step-title {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}

.step-description {
  font-size: 13px;
  color: #666;
  margin-bottom: 24px;
}

.overview-section {
  margin-bottom: 24px;
  border: 1px solid #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.section-title {
  margin: 0;
  padding: 12px 16px;
  font-size: 14px;
  font-weight: 600;
  color: #333;
  background-color: #f8f9fa;
  border-bottom: 1px solid #e0e0e0;
}

.overview-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0;

  @media (max-width: 600px) {
    grid-template-columns: 1fr;
  }
}

.overview-item {
  display: flex;
  flex-direction: column;
  padding: 12px 16px;
  border-bottom: 1px solid #f0f0f0;

  &:last-child {
    border-bottom: none;
  }

  .item-label {
    font-size: 12px;
    color: #999;
    margin-bottom: 2px;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  }

  .item-value {
    font-size: 14px;
    color: #333;
    font-weight: 500;
  }
}

.members-table {
  width: 100%;
}
```

---

## GroupStepperComponent (Orchestrator)

### Component

```typescript
// src/app/groups/group-stepper/group-stepper.component.ts

import { Component, OnInit, ViewChild, inject } from '@angular/core';
import { Router, ActivatedRoute } from '@angular/router';
import { MatStepper } from '@angular/material/stepper';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';
import { GroupsService } from '../groups.service';
import { SettingsService } from 'app/settings/settings.service';
import { Dates } from 'app/core/utils/dates';

import { GroupStepInfoComponent } from './group-step-info/group-step-info.component';
import { GroupStepClientsComponent } from './group-step-clients/group-step-clients.component';
import { GroupStepOverviewComponent } from './group-step-overview/group-step-overview.component';

@Component({
  selector: 'mifosx-group-stepper',
  templateUrl: './group-stepper.component.html',
  styleUrls: ['./group-stepper.component.scss'],
  standalone: true,
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatStepper,
    GroupStepInfoComponent,
    GroupStepClientsComponent,
    GroupStepOverviewComponent
  ]
})
export class GroupStepperComponent implements OnInit {
  private router = inject(Router);
  private route = inject(ActivatedRoute);
  private groupService = inject(GroupsService);
  private settingsService = inject(SettingsService);
  private dateUtils = inject(Dates);

  @ViewChild('stepper') stepper: MatStepper;
  @ViewChild(GroupStepInfoComponent) stepInfo: GroupStepInfoComponent;
  @ViewChild(GroupStepClientsComponent) stepClients: GroupStepClientsComponent;

  /** Office data from resolver */
  officeData: any[] = [];

  /** Resolved display names for the overview step */
  selectedOfficeName: string = '';
  selectedStaffName: string = '';

  /** Submission state */
  isSubmitting = false;

  ngOnInit(): void {
    this.route.data.subscribe((data: { offices: any }) => {
      this.officeData = data.offices;
    });
  }

  /**
   * Called when the stepper moves to a new step.
   * Used to resolve display names for the overview.
   */
  onStepChange(event: any): void {
    if (event.selectedIndex === 2) {
      // Moving to Overview step -- resolve display names
      this.resolveDisplayNames();
    }
  }

  private resolveDisplayNames(): void {
    const officeId = this.stepInfo?.groupInfoForm.get('officeId')?.value;
    const staffId = this.stepInfo?.groupInfoForm.get('staffId')?.value;

    const office = this.officeData.find((o: any) => o.id === officeId);
    this.selectedOfficeName = office?.name || '';

    if (staffId && this.stepInfo?.staffData) {
      const staff = this.stepInfo.staffData.find((s: any) => s.id === staffId);
      this.selectedStaffName = staff?.displayName || '';
    } else {
      this.selectedStaffName = '';
    }
  }

  /**
   * Submit the group creation request.
   * Called when the user clicks Submit on the Overview step.
   */
  submit(): void {
    if (this.isSubmitting) return;
    this.isSubmitting = true;

    const infoForm = this.stepInfo.groupInfoForm;
    const locale = this.settingsService.language.code;
    const dateFormat = this.settingsService.dateFormat;

    // Format date
    let submittedOnDate = infoForm.get('submittedOnDate')?.value;
    if (submittedOnDate instanceof Date) {
      submittedOnDate = this.dateUtils.formatDate(submittedOnDate, dateFormat);
    }

    // Build payload
    const payload: any = {
      name: infoForm.get('name')?.value,
      officeId: infoForm.get('officeId')?.value,
      submittedOnDate,
      externalId: infoForm.get('externalId')?.value || undefined,
      active: false,
      dateFormat,
      locale,
      clientMembers: this.stepClients.getClientMemberIds()
    };

    // Include staffId only if selected
    const staffId = infoForm.get('staffId')?.value;
    if (staffId) {
      payload.staffId = staffId;
    }

    this.groupService.createGroup(payload).subscribe({
      next: (response: any) => {
        this.isSubmitting = false;
        this.router.navigate([
          '../groups',
          response.resourceId,
          'general'
        ]);
      },
      error: () => {
        this.isSubmitting = false;
      }
    });
  }
}
```

### Template

```html
<!-- src/app/groups/group-stepper/group-stepper.component.html -->

<div class="group-stepper-container">
  <div class="stepper-header">
    <h2>Create New Group</h2>
  </div>

  <mat-stepper #stepper [linear]="true" (selectionChange)="onStepChange($event)" class="msacco-stepper">
    <!-- Step 1: Group Info -->
    <mat-step [stepControl]="stepInfo?.groupInfoForm" label="Group Info" [editable]="true">
      <ng-template matStepLabel>
        <mat-icon class="step-icon">info</mat-icon>
        Group Info
      </ng-template>

      <mifosx-group-step-info [officeData]="officeData"></mifosx-group-step-info>

      <div class="step-actions">
        <span></span>
        <button mat-flat-button color="primary" matStepperNext type="button">Next <mat-icon>arrow_forward</mat-icon></button>
      </div>
    </mat-step>

    <!-- Step 2: Select Clients -->
    <mat-step [stepControl]="stepClients?.clientsForm" label="Select Clients" [editable]="true">
      <ng-template matStepLabel>
        <mat-icon class="step-icon">people</mat-icon>
        Select Clients
      </ng-template>

      <mifosx-group-step-clients [officeId]="stepInfo?.selectedOfficeId"></mifosx-group-step-clients>

      <div class="step-actions">
        <button mat-stroked-button matStepperPrevious type="button"><mat-icon>arrow_back</mat-icon> Back</button>
        <button mat-flat-button color="primary" matStepperNext type="button">Next <mat-icon>arrow_forward</mat-icon></button>
      </div>
    </mat-step>

    <!-- Step 3: Overview -->
    <mat-step label="Overview" [editable]="true">
      <ng-template matStepLabel>
        <mat-icon class="step-icon">preview</mat-icon>
        Overview
      </ng-template>

      <mifosx-group-step-overview [groupInfoForm]="stepInfo?.groupInfoForm" [selectedClients]="stepClients?.selectedClients || []" [officeName]="selectedOfficeName" [staffName]="selectedStaffName"></mifosx-group-step-overview>

      <div class="step-actions">
        <button mat-stroked-button matStepperPrevious type="button"><mat-icon>arrow_back</mat-icon> Back</button>
        <button mat-flat-button color="primary" type="button" (click)="submit()" [disabled]="isSubmitting">
          @if (isSubmitting) {
          <mat-spinner diameter="20" class="inline-spinner"></mat-spinner>
          } @else {
          <mat-icon>check</mat-icon>
          } {{ isSubmitting ? 'Creating...' : 'Create Group' }}
        </button>
      </div>
    </mat-step>
  </mat-stepper>
</div>
```

### Styles

```scss
// src/app/groups/group-stepper/group-stepper.component.scss

$primary-blue: #1074b9;

.group-stepper-container {
  max-width: 1100px;
  margin: 0 auto;
  padding: 16px;
}

.stepper-header {
  margin-bottom: 16px;

  h2 {
    font-size: 22px;
    font-weight: 600;
    color: #333;
    margin: 0;
  }
}

.msacco-stepper {
  background: transparent;

  ::ng-deep {
    .mat-step-header .mat-step-icon {
      width: 36px;
      height: 36px;
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

    .mat-step-header .mat-step-label {
      font-size: 13px;
      font-weight: 500;

      &.mat-step-label-active,
      &.mat-step-label-selected {
        color: $primary-blue;
        font-weight: 600;
      }
    }

    .mat-stepper-horizontal-line {
      border-color: #e0e0e0;
    }

    .mat-horizontal-content-container {
      padding: 24px 16px;
    }
  }

  .step-icon {
    font-size: 18px;
    margin-right: 4px;
    vertical-align: middle;
  }
}

.step-actions {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-top: 24px;
  margin-top: 24px;
  border-top: 1px solid #eee;

  .inline-spinner {
    display: inline-block;
    margin-right: 8px;
  }
}
```

---

## Route Configuration Updates

### Update `src/app/groups/groups-routing.module.ts`

Replace the existing `create` route:

```typescript
// Before:
{
  path: 'create',
  component: CreateGroupComponent,
  data: { title: 'Create Group', breadcrumb: 'Create', routeParamBreadcrumb: false },
  resolve: { offices: OfficesResolver }
}

// After:
{
  path: 'create',
  component: GroupStepperComponent,
  data: { title: 'Create Group', breadcrumb: 'Create', routeParamBreadcrumb: false },
  resolve: { offices: OfficesResolver }
}
```

**Import the new component:**

```typescript
import { GroupStepperComponent } from './group-stepper/group-stepper.component';
```

The `OfficesResolver` is already in use and does not need to change.

### Keep the Old Component

The old `CreateGroupComponent` at `src/app/groups/create-group/` should be kept temporarily and can be removed after the wizard is verified. No changes to the old component are needed.

---

## Fineract API Payload

### `POST /fineract-provider/api/v1/groups`

```json
{
  "name": "Kilimani Savings Group",
  "officeId": 2,
  "staffId": 5,
  "externalId": "REG-2026-001",
  "active": false,
  "submittedOnDate": "13 March 2026",
  "dateFormat": "dd MMMM yyyy",
  "locale": "en",
  "clientMembers": [
    14,
    27,
    33,
    45
  ]
}
```

**Response:**

```json
{
  "officeId": 2,
  "groupId": 12,
  "resourceId": 12
}
```

On success, the wizard navigates to `/groups/12/general`.

---

## Service Methods Used

All service methods already exist in the codebase:

| Service          | Method                                                                       | Location                             |
| ---------------- | ---------------------------------------------------------------------------- | ------------------------------------ |
| `GroupsService`  | `getStaff(officeId)`                                                         | `src/app/groups/groups.service.ts`   |
| `GroupsService`  | `createGroup(data)`                                                          | `src/app/groups/groups.service.ts`   |
| `ClientsService` | `getFilteredClients(orderBy, sortOrder, orphansOnly, displayName, officeId)` | `src/app/clients/clients.service.ts` |

No new service methods are required.

---

## File Checklist

### New Files

| File                                                                                  | Purpose                |
| ------------------------------------------------------------------------------------- | ---------------------- |
| `src/app/groups/group-stepper/group-stepper.component.ts`                             | Orchestrator component |
| `src/app/groups/group-stepper/group-stepper.component.html`                           | Orchestrator template  |
| `src/app/groups/group-stepper/group-stepper.component.scss`                           | Orchestrator styles    |
| `src/app/groups/group-stepper/group-step-info/group-step-info.component.ts`           | Step 1 component       |
| `src/app/groups/group-stepper/group-step-info/group-step-info.component.html`         | Step 1 template        |
| `src/app/groups/group-stepper/group-step-info/group-step-info.component.scss`         | Step 1 styles          |
| `src/app/groups/group-stepper/group-step-clients/group-step-clients.component.ts`     | Step 2 component       |
| `src/app/groups/group-stepper/group-step-clients/group-step-clients.component.html`   | Step 2 template        |
| `src/app/groups/group-stepper/group-step-clients/group-step-clients.component.scss`   | Step 2 styles          |
| `src/app/groups/group-stepper/group-step-overview/group-step-overview.component.ts`   | Step 3 component       |
| `src/app/groups/group-stepper/group-step-overview/group-step-overview.component.html` | Step 3 template        |
| `src/app/groups/group-stepper/group-step-overview/group-step-overview.component.scss` | Step 3 styles          |

### Files to Modify

| File                                      | Change                                                                            |
| ----------------------------------------- | --------------------------------------------------------------------------------- |
| `src/app/groups/groups-routing.module.ts` | Replace `CreateGroupComponent` with `GroupStepperComponent` in the `create` route |

### Files to Keep (no changes)

| File                                 | Reason                                                  |
| ------------------------------------ | ------------------------------------------------------- |
| `src/app/groups/create-group/*`      | Keep as fallback until wizard is verified; remove later |
| `src/app/groups/groups.service.ts`   | Existing methods are sufficient                         |
| `src/app/clients/clients.service.ts` | Existing `getFilteredClients()` is reused               |

---

## Validation Summary

| Step                   | Validation Rules                                                                      | Gate                                                |
| ---------------------- | ------------------------------------------------------------------------------------- | --------------------------------------------------- |
| Step 1: Group Info     | Group Name required + starts with letter; Branch required; Registration Date required | Form must be valid to proceed to Step 2             |
| Step 2: Select Clients | At least 1 client selected                                                            | Hidden `_clientCount` field with `min(1)` validator |
| Step 3: Overview       | Read-only, no validation                                                              | Submit button triggers API call                     |

## Accessibility

- All form fields have `<mat-label>` for screen readers
- Step labels include icons but also text labels
- Tab order follows natural reading order within each step
- Error messages use `<mat-error>` which is announced by screen readers
- The stepper keyboard navigation (left/right arrows) is handled by `mat-stepper` natively
