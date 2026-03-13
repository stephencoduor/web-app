# TECH-010: MPESA Integration Specification

## Overview

Integrate M-PESA mobile money payment channels into the M-SACCO system for loan disbursements, repayments, and savings transactions. This specification covers the UI components, service layer, environment configuration, and Fineract API integration required to support M-PESA as a first-class payment method.

**Target Modules:**

- Loan product configuration (product creation wizard)
- Loan disbursement and repayment workflows
- Accounting rules and fund source mapping
- Transaction log and reconciliation

---

## 1. Loan Product Configuration

### 1.1 MPESA Step in Loan Product Wizard

Add a new step to the loan product creation stepper at `src/app/products/loan-products/loan-product-stepper/`.

**Location:** `src/app/products/loan-products/loan-product-stepper/loan-product-mpesa-step/`

**Files:**

- `loan-product-mpesa-step.component.ts`
- `loan-product-mpesa-step.component.html`
- `loan-product-mpesa-step.component.scss`

### 1.2 MPESA Step Component

```typescript
import { Component, OnInit, Input, Output, EventEmitter } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';

export interface MpesaProductConfig {
  mpesaEnabled: boolean;
  paybillNumber: string;
  accountReferenceFormat: 'CLIENT_ID' | 'LOAN_ACCOUNT_NO' | 'NATIONAL_ID';
  tillNumber?: string;
  b2cEnabled: boolean;
  c2bEnabled: boolean;
  autoReconcile: boolean;
}

@Component({
  selector: 'mifosx-loan-product-mpesa-step',
  templateUrl: './loan-product-mpesa-step.component.html',
  styleUrls: ['./loan-product-mpesa-step.component.scss']
})
export class LoanProductMpesaStepComponent implements OnInit {
  @Input() mpesaConfig: MpesaProductConfig;
  @Output() configChanged = new EventEmitter<MpesaProductConfig>();

  mpesaForm: UntypedFormGroup;

  accountReferenceOptions = [
    { value: 'CLIENT_ID', label: 'Client ID' },
    { value: 'LOAN_ACCOUNT_NO', label: 'Loan Account Number' },
    { value: 'NATIONAL_ID', label: 'National ID' }
  ];

  constructor(private fb: UntypedFormBuilder) {}

  ngOnInit(): void {
    this.mpesaForm = this.fb.group({
      mpesaEnabled: [this.mpesaConfig?.mpesaEnabled || false],
      paybillNumber: [
        this.mpesaConfig?.paybillNumber || '',
        []
      ],
      accountReferenceFormat: [this.mpesaConfig?.accountReferenceFormat || 'LOAN_ACCOUNT_NO'],
      tillNumber: [this.mpesaConfig?.tillNumber || ''],
      b2cEnabled: [this.mpesaConfig?.b2cEnabled || false],
      c2bEnabled: [this.mpesaConfig?.c2bEnabled || true],
      autoReconcile: [this.mpesaConfig?.autoReconcile || false]
    });

    // Toggle validators based on MPESA enabled state
    this.mpesaForm.get('mpesaEnabled').valueChanges.subscribe((enabled) => {
      const paybillControl = this.mpesaForm.get('paybillNumber');
      if (enabled) {
        paybillControl.setValidators([
          Validators.required,
          Validators.pattern(/^\d{5,7}$/)
        ]);
      } else {
        paybillControl.clearValidators();
      }
      paybillControl.updateValueAndValidity();
    });

    this.mpesaForm.valueChanges.subscribe((value) => {
      this.configChanged.emit(value);
    });
  }
}
```

### 1.3 MPESA Step Template

```html
<div class="mpesa-config-container">
  <h3>M-PESA Configuration</h3>
  <p class="section-description">Configure M-PESA mobile money as a payment channel for this loan product.</p>

  <form [formGroup]="mpesaForm">
    <!-- Enable MPESA Toggle -->
    <div class="toggle-row">
      <mat-slide-toggle formControlName="mpesaEnabled" color="primary"> Enable M-PESA for this loan product </mat-slide-toggle>
    </div>

    <div *ngIf="mpesaForm.get('mpesaEnabled').value" class="mpesa-fields" @fadeIn>
      <!-- Paybill Number -->
      <mat-form-field class="full-width">
        <mat-label>Paybill Number</mat-label>
        <input matInput formControlName="paybillNumber" placeholder="e.g., 123456" />
        <mat-hint>The M-PESA paybill number for receiving payments</mat-hint>
        <mat-error *ngIf="mpesaForm.get('paybillNumber').hasError('required')"> Paybill number is required when M-PESA is enabled </mat-error>
        <mat-error *ngIf="mpesaForm.get('paybillNumber').hasError('pattern')"> Enter a valid paybill number (5-7 digits) </mat-error>
      </mat-form-field>

      <!-- Account Reference Format -->
      <mat-form-field class="full-width">
        <mat-label>Account Reference Format</mat-label>
        <mat-select formControlName="accountReferenceFormat">
          <mat-option *ngFor="let opt of accountReferenceOptions" [value]="opt.value"> {{ opt.label }} </mat-option>
        </mat-select>
        <mat-hint> Determines how the account reference is generated for M-PESA transactions </mat-hint>
      </mat-form-field>

      <!-- Till Number (optional) -->
      <mat-form-field class="full-width">
        <mat-label>Till Number (Optional)</mat-label>
        <input matInput formControlName="tillNumber" placeholder="e.g., 5678901" />
        <mat-hint>Optional till number for Buy Goods transactions</mat-hint>
      </mat-form-field>

      <!-- Transaction Type Toggles -->
      <div class="transaction-types">
        <h4>Transaction Types</h4>
        <mat-slide-toggle formControlName="c2bEnabled" color="primary"> C2B (Customer to Business) - Loan Repayments </mat-slide-toggle>
        <mat-slide-toggle formControlName="b2cEnabled" color="primary"> B2C (Business to Customer) - Loan Disbursements </mat-slide-toggle>
      </div>

      <!-- Auto-Reconciliation -->
      <div class="auto-reconcile">
        <mat-slide-toggle formControlName="autoReconcile" color="primary"> Enable automatic transaction reconciliation </mat-slide-toggle>
        <p class="hint-text">When enabled, incoming M-PESA payments are automatically matched to client loan accounts using the account reference.</p>
      </div>
    </div>
  </form>
</div>
```

### 1.4 Integrating the Step into the Loan Product Stepper

In the loan product stepper component, add the MPESA step after the accounting step:

```typescript
// In loan-product-stepper.component.ts
// Add to the stepper steps array
steps = [
  // ... existing steps (Details, Currency, Terms, Settings, Charges, Accounting)
  { label: 'M-PESA', optional: true },
  // ... remaining steps
];

mpesaConfig: MpesaProductConfig = {
  mpesaEnabled: false,
  paybillNumber: '',
  accountReferenceFormat: 'LOAN_ACCOUNT_NO',
  b2cEnabled: false,
  c2bEnabled: true,
  autoReconcile: false
};

onMpesaConfigChanged(config: MpesaProductConfig): void {
  this.mpesaConfig = config;
}
```

In the stepper template, add:

```html
<!-- M-PESA Step -->
<mat-step [label]="'M-PESA'" [optional]="true">
  <mifosx-loan-product-mpesa-step [mpesaConfig]="mpesaConfig" (configChanged)="onMpesaConfigChanged($event)"> </mifosx-loan-product-mpesa-step>
  <div class="step-actions">
    <button mat-button matStepperPrevious>Back</button>
    <button mat-raised-button color="primary" matStepperNext>Next</button>
  </div>
</mat-step>
```

---

## 2. Accounting Rules - Fund Sources

### 2.1 Fund Source Mapping UI

Enhance the existing accounting step in the loan product wizard to support fund source mapping per payment channel.

**Wireframe Reference:** "Configure Fund sources for payment channels"

### 2.2 Fund Source Mapping Component

**Location:** `src/app/products/loan-products/loan-product-stepper/loan-product-accounting-step/fund-source-mapping/`

```typescript
import { Component, Input, OnInit, Output, EventEmitter } from '@angular/core';
import { UntypedFormArray, UntypedFormBuilder, UntypedFormGroup } from '@angular/forms';

export interface FundSourceMapping {
  paymentTypeId: number;
  paymentTypeName: string;
  fundSourceAccountId: number;
  fundSourceAccountName?: string;
}

@Component({
  selector: 'mifosx-fund-source-mapping',
  templateUrl: './fund-source-mapping.component.html',
  styleUrls: ['./fund-source-mapping.component.scss']
})
export class FundSourceMappingComponent implements OnInit {
  /** Available payment types from Fineract */
  @Input() paymentTypes: { id: number; name: string }[] = [];

  /** Available GL accounts for fund source selection */
  @Input() glAccounts: { id: number; name: string; glCode: string }[] = [];

  /** Existing fund source mappings (for edit mode) */
  @Input() existingMappings: FundSourceMapping[] = [];

  @Output() mappingsChanged = new EventEmitter<FundSourceMapping[]>();

  mappingsForm: UntypedFormArray;

  constructor(private fb: UntypedFormBuilder) {}

  ngOnInit(): void {
    this.initializeMappings();
  }

  private initializeMappings(): void {
    const groups = this.paymentTypes.map((pt) => {
      const existing = this.existingMappings.find((m) => m.paymentTypeId === pt.id);
      return this.fb.group({
        paymentTypeId: [pt.id],
        paymentTypeName: [pt.name],
        fundSourceAccountId: [existing?.fundSourceAccountId || null]
      });
    });
    this.mappingsForm = this.fb.array(groups);

    this.mappingsForm.valueChanges.subscribe((values) => {
      this.mappingsChanged.emit(values.filter((v: any) => v.fundSourceAccountId));
    });
  }

  getMappingGroup(index: number): UntypedFormGroup {
    return this.mappingsForm.at(index) as UntypedFormGroup;
  }
}
```

### 2.3 Fund Source Mapping Template

```html
<div class="fund-source-mapping">
  <h4>Payment Channel Fund Sources</h4>
  <p class="section-hint">Map each payment channel to a General Ledger (GL) account that serves as the fund source.</p>

  <table mat-table [dataSource]="mappingsForm.controls" class="full-width">
    <!-- Payment Type Column -->
    <ng-container matColumnDef="paymentType">
      <th mat-header-cell *matHeaderCellDef>Payment Type</th>
      <td mat-cell *matCellDef="let element; let i = index">{{ getMappingGroup(i).get('paymentTypeName').value }}</td>
    </ng-container>

    <!-- Fund Source Column -->
    <ng-container matColumnDef="fundSource">
      <th mat-header-cell *matHeaderCellDef>Fund Source (GL Account)</th>
      <td mat-cell *matCellDef="let element; let i = index">
        <mat-form-field class="table-select">
          <mat-select [formControl]="getMappingGroup(i).get('fundSourceAccountId')" placeholder="Select GL Account">
            <mat-option [value]="null">-- None --</mat-option>
            <mat-option *ngFor="let account of glAccounts" [value]="account.id"> {{ account.glCode }} - {{ account.name }} </mat-option>
          </mat-select>
        </mat-form-field>
      </td>
    </ng-container>

    <tr mat-header-row *matHeaderRowDef="['paymentType', 'fundSource']"></tr>
    <tr mat-row *matRowDef="let row; columns: ['paymentType', 'fundSource']"></tr>
  </table>
</div>
```

### 2.4 Expected Fund Source Mapping

| Payment Type  | Fund Source (GL Account) | GL Code Example |
| ------------- | ------------------------ | --------------- |
| Cash          | Bank Account             | 100001          |
| Cheques       | Bank Account             | 100001          |
| MPESA         | MPESA Account            | 100003          |
| Bank Transfer | Bank Account             | 100001          |

---

## 3. Loan Disbursement via MPESA

### 3.1 Disbursement Payment Type

When disbursing a loan where the loan product has MPESA enabled, show MPESA as a payment type option in the disbursement dialog.

**File to modify:** `src/app/loans/loans-view/loan-account-actions/disburse/disburse.component.ts`

### 3.2 Disbursement Component Enhancement

```typescript
// Add to the existing disburse component

/** Payment types from the loan product template */
paymentTypes: { id: number; name: string }[] = [];

/** Whether MPESA fields should be shown */
showMpesaFields: boolean = false;

ngOnInit(): void {
  // Existing initialization...

  // Load payment types from loan template data
  this.paymentTypes = this.loanTemplateData.paymentTypeOptions || [];

  // Watch for payment type changes to toggle MPESA fields
  this.disbursementForm.get('paymentTypeId').valueChanges.subscribe(paymentTypeId => {
    const selected = this.paymentTypes.find(pt => pt.id === paymentTypeId);
    this.showMpesaFields = selected?.name === 'MPESA';

    const mpesaTxnControl = this.disbursementForm.get('mpesaTransactionId');
    if (this.showMpesaFields) {
      mpesaTxnControl.setValidators([Validators.required, Validators.pattern(/^[A-Z0-9]{10}$/)]);
    } else {
      mpesaTxnControl.clearValidators();
      mpesaTxnControl.setValue('');
    }
    mpesaTxnControl.updateValueAndValidity();
  });
}

// Add to form builder
disbursementForm = this.fb.group({
  // ... existing fields
  paymentTypeId: [''],
  mpesaTransactionId: [''],
  mpesaPhoneNumber: ['']
});
```

### 3.3 Disbursement Template Addition

Add to the existing disbursement form template:

```html
<!-- Payment Type Selection -->
<mat-form-field class="full-width">
  <mat-label>Payment Type</mat-label>
  <mat-select formControlName="paymentTypeId">
    <mat-option *ngFor="let type of paymentTypes" [value]="type.id"> {{ type.name }} </mat-option>
  </mat-select>
</mat-form-field>

<!-- MPESA-specific fields (shown when MPESA payment type selected) -->
<div *ngIf="showMpesaFields" class="mpesa-fields" @fadeIn>
  <mat-form-field class="full-width">
    <mat-label>M-PESA Transaction ID</mat-label>
    <input matInput formControlName="mpesaTransactionId" placeholder="e.g., QJI5HFVE2M" style="text-transform: uppercase" />
    <mat-hint>The M-PESA receipt/confirmation code</mat-hint>
    <mat-error *ngIf="disbursementForm.get('mpesaTransactionId').hasError('required')"> M-PESA Transaction ID is required </mat-error>
    <mat-error *ngIf="disbursementForm.get('mpesaTransactionId').hasError('pattern')"> Enter a valid M-PESA transaction ID (10 alphanumeric characters) </mat-error>
  </mat-form-field>

  <mat-form-field class="full-width">
    <mat-label>Recipient Phone Number</mat-label>
    <input matInput formControlName="mpesaPhoneNumber" placeholder="e.g., 254712345678" />
    <mat-hint>Client M-PESA registered phone number</mat-hint>
  </mat-form-field>
</div>
```

### 3.4 Disbursement API Payload

When submitting a disbursement with MPESA as the payment type:

```json
{
  "dateFormat": "dd MMMM yyyy",
  "locale": "en",
  "actualDisbursementDate": "13 March 2026",
  "transactionAmount": 50000,
  "paymentTypeId": 3,
  "note": "Disbursed via M-PESA",
  "receiptNumber": "QJI5HFVE2M",
  "bankNumber": "",
  "accountNumber": "254712345678",
  "routingCode": ""
}
```

The `receiptNumber` field maps to the MPESA Transaction ID, and `accountNumber` maps to the recipient phone number.

**API Endpoint:** `POST /loans/{loanId}?command=disburse`

---

## 4. Loan Repayment via MPESA

### 4.1 Repayment Component Enhancement

**File to modify:** `src/app/loans/loans-view/loan-account-actions/make-repayment/make-repayment.component.ts`

The same payment type selection and MPESA-specific fields pattern applies to repayments.

### 4.2 MPESA Repayment Fields

```typescript
// Add to the repayment form
repaymentForm = this.fb.group({
  // ... existing fields (transactionDate, transactionAmount, etc.)
  paymentTypeId: [''],
  mpesaTransactionId: [''],
  mpesaPhoneNumber: [''],
  externalId: ['']
});

// Watch payment type to toggle MPESA fields
onPaymentTypeChange(paymentTypeId: number): void {
  const selected = this.paymentTypes.find(pt => pt.id === paymentTypeId);
  this.showMpesaFields = selected?.name === 'MPESA';
}
```

### 4.3 Auto-Matching for C2B Repayments

When M-PESA C2B (Customer to Business) payments are received via the callback URL, the system should auto-match them to client loan accounts using the account reference.

**Matching Logic:**

```typescript
/**
 * Match an incoming M-PESA C2B payment to a client loan account.
 *
 * The account reference in the M-PESA transaction is matched against
 * the configured account reference format on the loan product:
 * - CLIENT_ID: Match against client.id
 * - LOAN_ACCOUNT_NO: Match against loan.accountNo
 * - NATIONAL_ID: Match against client external ID (national ID)
 */
interface MpesaC2BPayment {
  transactionType: string;
  transactionId: string;
  transactionTime: string;
  transactionAmount: number;
  businessShortCode: string;
  billRefNumber: string; // The account reference provided by the customer
  invoiceNumber: string;
  orgAccountBalance: number;
  thirdPartyTransId: string;
  msisdn: string; // Customer phone number
  firstName: string;
  middleName: string;
  lastName: string;
}
```

### 4.4 Repayment API Payload

```json
{
  "dateFormat": "dd MMMM yyyy",
  "locale": "en",
  "transactionDate": "13 March 2026",
  "transactionAmount": 5000,
  "paymentTypeId": 3,
  "note": "M-PESA repayment",
  "receiptNumber": "QJI5HFVE2M",
  "accountNumber": "254712345678"
}
```

**API Endpoint:** `POST /loans/{loanId}/transactions?command=repayment`

---

## 5. MPESA Transaction Log

### 5.1 Transaction Log Component

**Location:** `src/app/shared/mpesa/mpesa-transaction-log/`

**Files:**

- `mpesa-transaction-log.component.ts`
- `mpesa-transaction-log.component.html`
- `mpesa-transaction-log.component.scss`

### 5.2 Component Implementation

```typescript
import { Component, Input, OnInit, ViewChild } from '@angular/core';
import { MatPaginator } from '@angular/material/paginator';
import { MatSort } from '@angular/material/sort';
import { MatTableDataSource } from '@angular/material/table';
import { MpesaService, MpesaTransaction } from '../mpesa.service';

@Component({
  selector: 'mifosx-mpesa-transaction-log',
  templateUrl: './mpesa-transaction-log.component.html',
  styleUrls: ['./mpesa-transaction-log.component.scss']
})
export class MpesaTransactionLogComponent implements OnInit {
  /** Context: either a client ID or a loan ID */
  @Input() clientId: number;
  @Input() loanId: number;

  displayedColumns: string[] = [
    'transactionId',
    'type',
    'amount',
    'date',
    'status',
    'phoneNumber',
    'reference'
  ];

  dataSource: MatTableDataSource<MpesaTransaction>;
  isLoading: boolean = false;

  @ViewChild(MatPaginator) paginator: MatPaginator;
  @ViewChild(MatSort) sort: MatSort;

  constructor(private mpesaService: MpesaService) {}

  ngOnInit(): void {
    this.loadTransactions();
  }

  loadTransactions(): void {
    this.isLoading = true;

    const observable = this.loanId ? this.mpesaService.getLoanTransactions(this.loanId) : this.mpesaService.getClientTransactions(this.clientId);

    observable.subscribe({
      next: (transactions) => {
        this.dataSource = new MatTableDataSource(transactions);
        this.dataSource.paginator = this.paginator;
        this.dataSource.sort = this.sort;
        this.isLoading = false;
      },
      error: () => {
        this.isLoading = false;
      }
    });
  }

  getStatusColor(status: string): string {
    switch (status) {
      case 'COMPLETED':
        return 'green';
      case 'PENDING':
        return 'orange';
      case 'FAILED':
        return 'red';
      case 'REVERSED':
        return 'grey';
      default:
        return 'inherit';
    }
  }

  getTypeIcon(type: string): string {
    switch (type) {
      case 'DISBURSEMENT':
        return 'arrow_upward';
      case 'REPAYMENT':
        return 'arrow_downward';
      case 'DEPOSIT':
        return 'savings';
      default:
        return 'swap_horiz';
    }
  }
}
```

### 5.3 Transaction Log Template

```html
<div class="mpesa-transaction-log">
  <div class="header-row">
    <h3>M-PESA Transactions</h3>
    <button mat-icon-button (click)="loadTransactions()" matTooltip="Refresh">
      <mat-icon>refresh</mat-icon>
    </button>
  </div>

  <!-- Loading State -->
  <mat-progress-bar *ngIf="isLoading" mode="indeterminate"></mat-progress-bar>

  <!-- Transaction Table -->
  <table mat-table [dataSource]="dataSource" matSort class="full-width" *ngIf="dataSource?.data?.length > 0">
    <!-- Transaction ID -->
    <ng-container matColumnDef="transactionId">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Transaction ID</th>
      <td mat-cell *matCellDef="let txn">
        <code class="txn-id">{{ txn.transactionId }}</code>
      </td>
    </ng-container>

    <!-- Type -->
    <ng-container matColumnDef="type">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Type</th>
      <td mat-cell *matCellDef="let txn">
        <span class="type-badge">
          <mat-icon>{{ getTypeIcon(txn.type) }}</mat-icon>
          {{ txn.type }}
        </span>
      </td>
    </ng-container>

    <!-- Amount -->
    <ng-container matColumnDef="amount">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Amount</th>
      <td mat-cell *matCellDef="let txn">{{ txn.amount | currency:txn.currency }}</td>
    </ng-container>

    <!-- Date -->
    <ng-container matColumnDef="date">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Date</th>
      <td mat-cell *matCellDef="let txn">{{ txn.transactionDate | date:'medium' }}</td>
    </ng-container>

    <!-- Status -->
    <ng-container matColumnDef="status">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Status</th>
      <td mat-cell *matCellDef="let txn">
        <span class="status-chip" [style.color]="getStatusColor(txn.status)"> {{ txn.status }} </span>
      </td>
    </ng-container>

    <!-- Phone Number -->
    <ng-container matColumnDef="phoneNumber">
      <th mat-header-cell *matHeaderCellDef>Phone Number</th>
      <td mat-cell *matCellDef="let txn">{{ txn.phoneNumber }}</td>
    </ng-container>

    <!-- Reference -->
    <ng-container matColumnDef="reference">
      <th mat-header-cell *matHeaderCellDef>Reference</th>
      <td mat-cell *matCellDef="let txn">{{ txn.accountReference }}</td>
    </ng-container>

    <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
    <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>
  </table>

  <!-- Paginator -->
  <mat-paginator [pageSizeOptions]="[10, 25, 50]" showFirstLastButtons *ngIf="dataSource?.data?.length > 0"> </mat-paginator>

  <!-- Empty State -->
  <div class="empty-state" *ngIf="!isLoading && (!dataSource || dataSource.data.length === 0)">
    <mat-icon>receipt_long</mat-icon>
    <p>No M-PESA transactions found.</p>
  </div>
</div>
```

### 5.4 Integrating the Transaction Log

Add the transaction log as a tab in the client detail view and loan detail view.

**Client Detail View** (`src/app/clients/clients-view/`):

```html
<!-- Add as a new tab in client-tabs -->
<mat-tab label="M-PESA Transactions">
  <mifosx-mpesa-transaction-log [clientId]="clientId"> </mifosx-mpesa-transaction-log>
</mat-tab>
```

**Loan Detail View** (`src/app/loans/loans-view/`):

```html
<!-- Add as a new tab in loan-tabs -->
<mat-tab label="M-PESA Transactions">
  <mifosx-mpesa-transaction-log [loanId]="loanId"> </mifosx-mpesa-transaction-log>
</mat-tab>
```

---

## 6. MPESA Service

### 6.1 Service Implementation

**Location:** `src/app/shared/mpesa/mpesa.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface MpesaTransaction {
  id: number;
  transactionId: string;
  type: 'DISBURSEMENT' | 'REPAYMENT' | 'DEPOSIT' | 'WITHDRAWAL';
  amount: number;
  currency: string;
  transactionDate: string;
  status: 'COMPLETED' | 'PENDING' | 'FAILED' | 'REVERSED';
  phoneNumber: string;
  accountReference: string;
  clientId?: number;
  loanId?: number;
  savingsId?: number;
  receiptNumber: string;
  resultDescription?: string;
}

export interface MpesaPaymentType {
  id: number;
  name: string;
  description: string;
  isCashPayment: boolean;
  position: number;
}

export interface MpesaBalanceResult {
  accountBalance: string;
  resultCode: number;
  resultDescription: string;
}

@Injectable({ providedIn: 'root' })
export class MpesaService {
  constructor(private http: HttpClient) {}

  // --- Transaction Queries ---

  /**
   * Get M-PESA transactions for a specific client.
   * Queries the Fineract loan/savings transaction history filtered by MPESA payment type.
   */
  getClientTransactions(clientId: number): Observable<MpesaTransaction[]> {
    return this.http.get<MpesaTransaction[]>(`/clients/${clientId}/transactions`, { params: new HttpParams().set('paymentType', 'MPESA') });
  }

  /**
   * Get M-PESA transactions for a specific loan.
   */
  getLoanTransactions(loanId: number): Observable<MpesaTransaction[]> {
    return this.http.get<MpesaTransaction[]>(`/loans/${loanId}/transactions`, { params: new HttpParams().set('paymentType', 'MPESA') });
  }

  /**
   * Get M-PESA transactions for a specific savings account.
   */
  getSavingsTransactions(savingsId: number): Observable<MpesaTransaction[]> {
    return this.http.get<MpesaTransaction[]>(`/savingsaccounts/${savingsId}/transactions`, { params: new HttpParams().set('paymentType', 'MPESA') });
  }

  // --- Payment Type Management ---

  /**
   * Get the MPESA payment type from Fineract.
   * If it does not exist, it must be created first via setupMpesaPaymentType().
   */
  getMpesaPaymentType(): Observable<MpesaPaymentType> {
    return this.http.get<MpesaPaymentType>('/paymenttypes/mpesa');
  }

  /**
   * Create the MPESA payment type in Fineract if it does not exist.
   */
  setupMpesaPaymentType(): Observable<MpesaPaymentType> {
    return this.http.post<MpesaPaymentType>('/paymenttypes', {
      name: 'MPESA',
      description: 'M-PESA Mobile Money',
      isCashPayment: false,
      position: 3
    });
  }

  // --- Transaction Verification ---

  /**
   * Verify an M-PESA transaction ID against the Safaricom API.
   * Used to confirm that a transaction is genuine before recording it.
   */
  verifyTransaction(transactionId: string): Observable<{ valid: boolean; details: any }> {
    return this.http.post<{ valid: boolean; details: any }>('/mpesa/verify', { transactionId });
  }

  // --- Reconciliation ---

  /**
   * Get unreconciled M-PESA transactions.
   * These are payments received via M-PESA that have not been matched to a loan or savings account.
   */
  getUnreconciledTransactions(): Observable<MpesaTransaction[]> {
    return this.http.get<MpesaTransaction[]>('/mpesa/unreconciled');
  }

  /**
   * Manually reconcile an M-PESA transaction with a loan or savings account.
   */
  reconcileTransaction(transactionId: string, targetType: 'loan' | 'savings', targetId: number): Observable<any> {
    return this.http.post('/mpesa/reconcile', {
      mpesaTransactionId: transactionId,
      targetType,
      targetId
    });
  }
}
```

---

## 7. Environment Configuration

### 7.1 Environment Settings

**File:** `src/environments/environment.ts`

Add the following MPESA configuration block to the environment files:

```typescript
export const environment = {
  // ... existing configuration
  mpesa: {
    /** Whether MPESA integration is enabled globally */
    enabled: true,

    /** MPESA Daraja API base URL */
    apiUrl: '',

    /** Organization paybill number (default, can be overridden per product) */
    paybillNumber: '',

    /** Daraja API consumer key */
    consumerKey: '',

    /** Daraja API consumer secret */
    consumerSecret: '',

    /** Lipa Na MPESA passkey */
    passKey: '',

    /** URL for MPESA callback notifications */
    callbackUrl: '',

    /** MPESA API environment */
    environment: 'sandbox' as 'sandbox' | 'production',

    /** Timeout for MPESA API calls in milliseconds */
    apiTimeout: 30000,

    /** Whether to enable auto-reconciliation of C2B payments */
    autoReconcile: false
  }
};
```

### 7.2 Production Environment

**File:** `src/environments/environment.prod.ts`

```typescript
export const environment = {
  // ... existing production configuration
  mpesa: {
    enabled: true,
    apiUrl: 'https://api.safaricom.co.ke',
    paybillNumber: '', // Set via deployment config
    consumerKey: '', // Set via deployment config
    consumerSecret: '', // Set via deployment config
    passKey: '', // Set via deployment config
    callbackUrl: '', // Set via deployment config
    environment: 'production' as 'sandbox' | 'production',
    apiTimeout: 30000,
    autoReconcile: true
  }
};
```

### 7.3 Configuration Notes

The sensitive fields (`consumerKey`, `consumerSecret`, `passKey`) should be set via environment variables at deployment time and never committed to source control. The Fineract backend handles the actual MPESA Daraja API communication; the Angular frontend only configures the payment type and captures transaction references.

---

## 8. Fineract Payment Type Setup

### 8.1 Create MPESA Payment Type

The MPESA payment type must exist in Fineract before it can be used in loan products.

**API Call:**

```
POST /fineract-provider/api/v1/paymenttypes
Content-Type: application/json

{
  "name": "MPESA",
  "description": "M-PESA Mobile Money",
  "isCashPayment": false,
  "position": 3
}
```

**Response:**

```json
{
  "resourceId": 3
}
```

### 8.2 Map GL Accounts for Financial Activities

Map the MPESA GL account to the appropriate financial activity:

```
POST /fineract-provider/api/v1/financialactivityaccounts
Content-Type: application/json

{
  "financialActivityId": 100,
  "glAccountId": <mpesa_gl_account_id>
}
```

### 8.3 Configure Fund Source for Loan Products

When creating or updating a loan product, include the fund source mapping:

```json
{
  "paymentChannelToFundSourceMappings": [
    {
      "paymentTypeId": 1,
      "fundSourceAccountId": 10
    },
    {
      "paymentTypeId": 2,
      "fundSourceAccountId": 10
    },
    {
      "paymentTypeId": 3,
      "fundSourceAccountId": 15
    }
  ]
}
```

Where `paymentTypeId: 3` is MPESA and `fundSourceAccountId: 15` is the MPESA GL account.

**API Endpoint:** `POST /fineract-provider/api/v1/loanproducts` or `PUT /fineract-provider/api/v1/loanproducts/{productId}`

---

## 9. New Files Summary

| File Path                                                                                                                                 | Purpose                             |
| ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| `src/app/products/loan-products/loan-product-stepper/loan-product-mpesa-step/loan-product-mpesa-step.component.ts`                        | MPESA configuration step component  |
| `src/app/products/loan-products/loan-product-stepper/loan-product-mpesa-step/loan-product-mpesa-step.component.html`                      | MPESA step template                 |
| `src/app/products/loan-products/loan-product-stepper/loan-product-mpesa-step/loan-product-mpesa-step.component.scss`                      | MPESA step styles                   |
| `src/app/products/loan-products/loan-product-stepper/loan-product-accounting-step/fund-source-mapping/fund-source-mapping.component.ts`   | Fund source mapping table component |
| `src/app/products/loan-products/loan-product-stepper/loan-product-accounting-step/fund-source-mapping/fund-source-mapping.component.html` | Fund source mapping template        |
| `src/app/products/loan-products/loan-product-stepper/loan-product-accounting-step/fund-source-mapping/fund-source-mapping.component.scss` | Fund source mapping styles          |
| `src/app/shared/mpesa/mpesa.service.ts`                                                                                                   | MPESA integration service           |
| `src/app/shared/mpesa/mpesa.module.ts`                                                                                                    | MPESA shared module                 |
| `src/app/shared/mpesa/mpesa-transaction-log/mpesa-transaction-log.component.ts`                                                           | Transaction log component           |
| `src/app/shared/mpesa/mpesa-transaction-log/mpesa-transaction-log.component.html`                                                         | Transaction log template            |
| `src/app/shared/mpesa/mpesa-transaction-log/mpesa-transaction-log.component.scss`                                                         | Transaction log styles              |

## 10. Modified Files Summary

| File Path                                                                                    | Changes                               |
| -------------------------------------------------------------------------------------------- | ------------------------------------- |
| `src/app/products/loan-products/loan-product-stepper/loan-product-stepper.component.ts`      | Add MPESA step to stepper             |
| `src/app/products/loan-products/loan-product-stepper/loan-product-stepper.component.html`    | Add MPESA step template               |
| `src/app/products/loan-products/loan-products.module.ts`                                     | Register MPESA step component         |
| `src/app/loans/loans-view/loan-account-actions/disburse/disburse.component.ts`               | Add MPESA payment type fields         |
| `src/app/loans/loans-view/loan-account-actions/disburse/disburse.component.html`             | Add MPESA fields to disbursement form |
| `src/app/loans/loans-view/loan-account-actions/make-repayment/make-repayment.component.ts`   | Add MPESA payment type fields         |
| `src/app/loans/loans-view/loan-account-actions/make-repayment/make-repayment.component.html` | Add MPESA fields to repayment form    |
| `src/app/clients/clients-view/clients-view.component.html`                                   | Add MPESA transactions tab            |
| `src/app/loans/loans-view/loans-view.component.html`                                         | Add MPESA transactions tab            |
| `src/environments/environment.ts`                                                            | Add MPESA config                      |
| `src/environments/environment.prod.ts`                                                       | Add MPESA production config           |

---

## 11. Testing Checklist

- [ ] MPESA step appears in loan product creation wizard when navigating to the step
- [ ] MPESA toggle enables/disables the paybill number required validation
- [ ] Paybill number validates as 5-7 digit number
- [ ] Account reference format dropdown shows all three options
- [ ] Fund source mapping table renders all payment types with GL account dropdowns
- [ ] MPESA payment type appears in loan disbursement form
- [ ] Selecting MPESA payment type shows transaction ID and phone number fields
- [ ] MPESA transaction ID validates as 10 alphanumeric characters
- [ ] Disbursement payload includes paymentTypeId, receiptNumber, and accountNumber
- [ ] MPESA payment type appears in loan repayment form
- [ ] Repayment payload includes correct MPESA fields
- [ ] Transaction log loads and displays MPESA transactions for a client
- [ ] Transaction log loads and displays MPESA transactions for a loan
- [ ] Transaction log pagination and sorting work correctly
- [ ] Empty state shows when no transactions exist
- [ ] Status colors render correctly (green/orange/red/grey)
- [ ] Environment configuration loads correctly in dev and production modes
