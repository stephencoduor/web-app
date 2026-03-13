# TECH-011: CRB (Credit Reference Bureau) Integration Specification

## Overview

Integrate Credit Reference Bureau (CRB) checks into the M-SACCO system for credit scoring and risk assessment of clients before loan approval. This specification covers the CRB tab in client detail views, credit check integration in the loan approval workflow, loan product configuration for CRB requirements, and all supporting components and services.

**Target Modules:**

- Client detail view (new CRB tab)
- Loan application and approval workflow
- Loan product configuration
- Shared CRB components and service

---

## 1. Client Detail CRB Tab

### 1.1 CRB Account Tab

**Location:** `src/app/clients/clients-view/crb-account-tab/`

**Files:**

- `crb-account-tab.component.ts`
- `crb-account-tab.component.html`
- `crb-account-tab.component.scss`

### 1.2 Tab Component Implementation

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { CRBService, CreditCheck, CreditScore } from '../../../shared/crb/crb.service';
import { CRBCheckDialogComponent } from '../../../shared/crb/crb-check-dialog/crb-check-dialog.component';

@Component({
  selector: 'mifosx-crb-account-tab',
  templateUrl: './crb-account-tab.component.html',
  styleUrls: ['./crb-account-tab.component.scss']
})
export class CRBAccountTabComponent implements OnInit {
  clientId: number;
  currentScore: CreditScore | null = null;
  creditHistory: CreditCheck[] = [];
  isLoading: boolean = false;

  constructor(
    private route: ActivatedRoute,
    private crbService: CRBService,
    private dialog: MatDialog
  ) {}

  ngOnInit(): void {
    this.clientId = this.route.parent.snapshot.params['clientId'];
    this.loadCRBData();
  }

  loadCRBData(): void {
    this.isLoading = true;

    // Load current score and history in parallel
    this.crbService.getCreditScore(this.clientId).subscribe({
      next: (score) => {
        this.currentScore = score;
      },
      error: () => {
        this.currentScore = null;
      }
    });

    this.crbService.getCreditChecks(this.clientId).subscribe({
      next: (checks) => {
        this.creditHistory = checks;
        this.isLoading = false;
      },
      error: () => {
        this.creditHistory = [];
        this.isLoading = false;
      }
    });
  }

  openCreditCheckDialog(): void {
    const dialogRef = this.dialog.open(CRBCheckDialogComponent, {
      width: '500px',
      data: { clientId: this.clientId }
    });

    dialogRef.afterClosed().subscribe((result) => {
      if (result) {
        this.loadCRBData();
      }
    });
  }

  downloadReport(checkId: number): void {
    this.crbService.downloadReport(checkId).subscribe((blob) => {
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `crb-report-${checkId}.pdf`;
      a.click();
      window.URL.revokeObjectURL(url);
    });
  }
}
```

### 1.3 Tab Template

```html
<div class="crb-account-tab">
  <!-- Header with Action Button -->
  <div class="tab-header">
    <h2>Credit Reference Bureau</h2>
    <button mat-raised-button color="primary" (click)="openCreditCheckDialog()">
      <mat-icon>search</mat-icon>
      Request New Credit Check
    </button>
  </div>

  <!-- Loading -->
  <mat-progress-bar *ngIf="isLoading" mode="indeterminate"></mat-progress-bar>

  <!-- Credit Score Card -->
  <mifosx-crb-score-card *ngIf="currentScore" [score]="currentScore.currentScore" [rating]="currentScore.rating" [lastCheckDate]="currentScore.lastCheckDate" [bureau]="currentScore.bureau" [isExpired]="currentScore.isExpired"> </mifosx-crb-score-card>

  <!-- No Score State -->
  <mat-card *ngIf="!currentScore && !isLoading" class="no-score-card">
    <mat-card-content>
      <div class="empty-state">
        <mat-icon>credit_score</mat-icon>
        <h3>No Credit Score Available</h3>
        <p>This client has not had a credit check performed yet.</p>
        <button mat-raised-button color="primary" (click)="openCreditCheckDialog()">Request First Credit Check</button>
      </div>
    </mat-card-content>
  </mat-card>

  <!-- Credit History Table -->
  <mifosx-crb-history-table *ngIf="creditHistory.length > 0" [creditChecks]="creditHistory" (downloadReport)="downloadReport($event)"> </mifosx-crb-history-table>
</div>
```

### 1.4 Route Configuration

Add the CRB tab route to the client view routing:

```typescript
// In clients-view-routing.module.ts
{
  path: 'crb',
  component: CRBAccountTabComponent,
  data: { title: 'CRB', routeParamCondition: 'true' }
}
```

Add the tab link to the client view tabs:

```typescript
// In clients-view.component.ts - clientViewTabs array
{ label: 'CRB', link: 'crb', icon: 'credit_score' }
```

---

## 2. Loan Application Credit Check

### 2.1 Loan Approval Credit Check Gate

**File to modify:** `src/app/loans/loans-view/loan-account-actions/approve-loan/approve-loan.component.ts`

Before a loan can be approved, the system checks whether the loan product requires a CRB check and whether one has been completed.

### 2.2 Credit Check Gate Implementation

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { CRBService, CreditScore } from '../../../../shared/crb/crb.service';
import { CRBCheckDialogComponent } from '../../../../shared/crb/crb-check-dialog/crb-check-dialog.component';

@Component({
  selector: 'mifosx-approve-loan',
  templateUrl: './approve-loan.component.html',
  styleUrls: ['./approve-loan.component.scss']
})
export class ApproveLoanComponent implements OnInit {
  loanId: number;
  clientId: number;
  loanProductId: number;

  /** Whether the loan product requires CRB check before approval */
  crbCheckRequired: boolean = false;

  /** Current credit score for the client */
  creditScore: CreditScore | null = null;

  /** Whether CRB check has been completed and meets minimum threshold */
  crbCheckPassed: boolean = false;

  /** CRB status indicator color */
  crbStatusColor: 'green' | 'yellow' | 'red' | 'grey' = 'grey';

  /** Minimum credit score from loan product configuration */
  minimumCreditScore: number = 0;

  /** Whether CRB data is loading */
  crbLoading: boolean = false;

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private crbService: CRBService,
    private dialog: MatDialog
  ) {}

  ngOnInit(): void {
    this.loanId = this.route.snapshot.params['loanId'];
    // Extract from resolved data
    const loanData = this.route.snapshot.data['loanActionData'];
    this.clientId = loanData.clientId;
    this.loanProductId = loanData.loanProductId;

    this.checkCRBRequirement();
  }

  /**
   * Check if the loan product requires CRB check and evaluate the client's credit status.
   */
  checkCRBRequirement(): void {
    // Check loan product CRB configuration
    this.crbService.getLoanProductCRBConfig(this.loanProductId).subscribe((config) => {
      this.crbCheckRequired = config.crbCheckEnabled;
      this.minimumCreditScore = config.minimumCreditScore || 0;

      if (this.crbCheckRequired) {
        this.loadClientCreditScore();
      }
    });
  }

  loadClientCreditScore(): void {
    this.crbLoading = true;
    this.crbService.getCreditScore(this.clientId).subscribe({
      next: (score) => {
        this.creditScore = score;
        this.evaluateCreditStatus(score);
        this.crbLoading = false;
      },
      error: () => {
        this.creditScore = null;
        this.crbCheckPassed = false;
        this.crbStatusColor = 'grey';
        this.crbLoading = false;
      }
    });
  }

  /**
   * Evaluate credit status and set the traffic light indicator.
   * Green: Score meets or exceeds minimum, check is current.
   * Yellow: Score is below minimum but check exists, or check is expired.
   * Red: No credit check exists or check failed.
   */
  evaluateCreditStatus(score: CreditScore): void {
    if (!score || score.isExpired) {
      this.crbStatusColor = score ? 'yellow' : 'red';
      this.crbCheckPassed = false;
      return;
    }

    if (score.currentScore >= this.minimumCreditScore) {
      this.crbStatusColor = 'green';
      this.crbCheckPassed = true;
    } else {
      this.crbStatusColor = 'yellow';
      this.crbCheckPassed = false;
    }
  }

  /**
   * Get the display label for the traffic light status.
   */
  getCRBStatusLabel(): string {
    switch (this.crbStatusColor) {
      case 'green':
        return 'Approved - Credit score meets requirements';
      case 'yellow':
        return 'Review Required - Credit score below threshold or expired';
      case 'red':
        return 'No Credit Check - Credit check required before approval';
      case 'grey':
        return 'Loading credit status...';
    }
  }

  /**
   * Open dialog to request a new credit check for this client.
   */
  requestCreditCheck(): void {
    const dialogRef = this.dialog.open(CRBCheckDialogComponent, {
      width: '500px',
      data: { clientId: this.clientId }
    });

    dialogRef.afterClosed().subscribe((result) => {
      if (result) {
        this.loadClientCreditScore();
      }
    });
  }

  /**
   * Whether the approve button should be disabled.
   * Disabled when CRB check is required but not passed.
   */
  isApprovalBlocked(): boolean {
    return this.crbCheckRequired && !this.crbCheckPassed;
  }
}
```

### 2.3 Credit Check Gate Template

Add to the loan approval form template:

```html
<!-- CRB Check Section (shown when loan product requires CRB) -->
<div *ngIf="crbCheckRequired" class="crb-check-section">
  <mat-card>
    <mat-card-header>
      <mat-card-title>
        <mat-icon>credit_score</mat-icon>
        Credit Reference Bureau Check
      </mat-card-title>
    </mat-card-header>

    <mat-card-content>
      <!-- Loading -->
      <mat-progress-bar *ngIf="crbLoading" mode="indeterminate"></mat-progress-bar>

      <!-- Traffic Light Indicator -->
      <div class="crb-status-indicator" *ngIf="!crbLoading">
        <div class="traffic-light">
          <div class="light red" [class.active]="crbStatusColor === 'red'"></div>
          <div class="light yellow" [class.active]="crbStatusColor === 'yellow'"></div>
          <div class="light green" [class.active]="crbStatusColor === 'green'"></div>
        </div>
        <div class="status-details">
          <p class="status-label">{{ getCRBStatusLabel() }}</p>
          <p *ngIf="creditScore" class="score-value">
            Score: <strong>{{ creditScore.currentScore }}</strong>
            (Minimum required: {{ minimumCreditScore }})
          </p>
          <p *ngIf="creditScore" class="score-info">
            Bureau: {{ creditScore.bureau }} | Last checked: {{ creditScore.lastCheckDate | date:'mediumDate' }}
            <span *ngIf="creditScore.isExpired" class="expired-badge">EXPIRED</span>
          </p>
        </div>
      </div>

      <!-- Action: Request New Check -->
      <div class="crb-actions" *ngIf="!crbLoading && !crbCheckPassed">
        <button mat-raised-button color="primary" (click)="requestCreditCheck()">
          <mat-icon>refresh</mat-icon>
          Request New Credit Check
        </button>
        <p class="action-hint" *ngIf="crbStatusColor === 'red'">A credit check must be completed before this loan can be approved.</p>
        <p class="action-hint" *ngIf="crbStatusColor === 'yellow'">The credit score is below the minimum threshold. You may still proceed with manual override if authorized.</p>
      </div>
    </mat-card-content>
  </mat-card>
</div>

<!-- Existing Approve Button (with CRB gate) -->
<button mat-raised-button color="primary" [disabled]="isApprovalBlocked() || approvalForm.invalid" (click)="approveLoan()">Approve Loan</button>
```

### 2.4 Traffic Light Styles

```scss
.crb-check-section {
  margin-bottom: 24px;
}

.crb-status-indicator {
  display: flex;
  align-items: center;
  gap: 24px;
  padding: 16px 0;
}

.traffic-light {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 12px 8px;
  background: #333;
  border-radius: 8px;
  width: 40px;
  align-items: center;

  .light {
    width: 24px;
    height: 24px;
    border-radius: 50%;
    opacity: 0.2;
    transition: opacity 0.3s;

    &.red {
      background: #f44336;
    }
    &.yellow {
      background: #ff9800;
    }
    &.green {
      background: #4caf50;
    }

    &.active {
      opacity: 1;
      box-shadow: 0 0 12px currentColor;

      &.red {
        box-shadow: 0 0 12px #f44336;
      }
      &.yellow {
        box-shadow: 0 0 12px #ff9800;
      }
      &.green {
        box-shadow: 0 0 12px #4caf50;
      }
    }
  }
}

.status-details {
  .status-label {
    font-size: 16px;
    font-weight: 500;
    margin: 0 0 4px 0;
  }

  .score-value {
    font-size: 14px;
    margin: 0 0 4px 0;
  }

  .score-info {
    font-size: 13px;
    color: rgba(0, 0, 0, 0.54);
    margin: 0;
  }

  .expired-badge {
    display: inline-block;
    background: #ff9800;
    color: white;
    padding: 1px 6px;
    border-radius: 4px;
    font-size: 11px;
    font-weight: 600;
    margin-left: 4px;
  }
}

.crb-actions {
  margin-top: 16px;

  .action-hint {
    margin-top: 8px;
    font-size: 13px;
    color: rgba(0, 0, 0, 0.54);
  }
}
```

---

## 3. Loan Product Credit Check Configuration

### 3.1 CRB Step in Loan Product Wizard

Add CRB configuration to the loan product creation stepper at `src/app/products/loan-products/loan-product-stepper/`.

**Location:** `src/app/products/loan-products/loan-product-stepper/loan-product-crb-step/`

### 3.2 CRB Step Component

```typescript
import { Component, OnInit, Input, Output, EventEmitter } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';

export interface CRBProductConfig {
  crbCheckEnabled: boolean;
  minimumCreditScore: number;
  crbProviderId: number | null;
  autoCheckOnApplication: boolean;
  blockApprovalOnFailure: boolean;
  scoreExpiryDays: number;
}

@Component({
  selector: 'mifosx-loan-product-crb-step',
  templateUrl: './loan-product-crb-step.component.html',
  styleUrls: ['./loan-product-crb-step.component.scss']
})
export class LoanProductCRBStepComponent implements OnInit {
  @Input() crbConfig: CRBProductConfig;
  @Input() crbProviders: { id: number; name: string }[] = [];
  @Output() configChanged = new EventEmitter<CRBProductConfig>();

  crbForm: UntypedFormGroup;

  constructor(private fb: UntypedFormBuilder) {}

  ngOnInit(): void {
    this.crbForm = this.fb.group({
      crbCheckEnabled: [this.crbConfig?.crbCheckEnabled || false],
      minimumCreditScore: [
        this.crbConfig?.minimumCreditScore || 300,
        [
          Validators.min(0),
          Validators.max(900)
        ]
      ],
      crbProviderId: [this.crbConfig?.crbProviderId || null],
      autoCheckOnApplication: [this.crbConfig?.autoCheckOnApplication || false],
      blockApprovalOnFailure: [this.crbConfig?.blockApprovalOnFailure || true],
      scoreExpiryDays: [
        this.crbConfig?.scoreExpiryDays || 90,
        [
          Validators.min(1),
          Validators.max(365)
        ]
      ]
    });

    // Toggle validators
    this.crbForm.get('crbCheckEnabled').valueChanges.subscribe((enabled) => {
      const scoreControl = this.crbForm.get('minimumCreditScore');
      const providerControl = this.crbForm.get('crbProviderId');

      if (enabled) {
        scoreControl.setValidators([
          Validators.required,
          Validators.min(0),
          Validators.max(900)
        ]);
        providerControl.setValidators([Validators.required]);
      } else {
        scoreControl.clearValidators();
        providerControl.clearValidators();
      }
      scoreControl.updateValueAndValidity();
      providerControl.updateValueAndValidity();
    });

    this.crbForm.valueChanges.subscribe((value) => {
      this.configChanged.emit(value);
    });
  }
}
```

### 3.3 CRB Step Template

```html
<div class="crb-config-container">
  <h3>Credit Reference Bureau Configuration</h3>
  <p class="section-description">Configure credit check requirements for this loan product. When enabled, a CRB check must be completed before loans using this product can be approved.</p>

  <form [formGroup]="crbForm">
    <!-- Enable CRB Toggle -->
    <div class="toggle-row">
      <mat-slide-toggle formControlName="crbCheckEnabled" color="primary"> Enable CRB check for this loan product </mat-slide-toggle>
    </div>

    <div *ngIf="crbForm.get('crbCheckEnabled').value" class="crb-fields">
      <!-- CRB Provider Selection -->
      <mat-form-field class="full-width">
        <mat-label>CRB Provider</mat-label>
        <mat-select formControlName="crbProviderId">
          <mat-option *ngFor="let provider of crbProviders" [value]="provider.id"> {{ provider.name }} </mat-option>
        </mat-select>
        <mat-hint>Select the Credit Reference Bureau to use for credit checks</mat-hint>
        <mat-error *ngIf="crbForm.get('crbProviderId').hasError('required')"> CRB provider is required when credit checks are enabled </mat-error>
      </mat-form-field>

      <!-- Minimum Credit Score -->
      <mat-form-field class="full-width">
        <mat-label>Minimum Credit Score</mat-label>
        <input matInput type="number" formControlName="minimumCreditScore" />
        <mat-hint> Clients must have a credit score at or above this threshold (0-900) </mat-hint>
        <mat-error *ngIf="crbForm.get('minimumCreditScore').hasError('min')"> Score must be at least 0 </mat-error>
        <mat-error *ngIf="crbForm.get('minimumCreditScore').hasError('max')"> Score cannot exceed 900 </mat-error>
      </mat-form-field>

      <!-- Score Expiry Days -->
      <mat-form-field class="full-width">
        <mat-label>Score Expiry (Days)</mat-label>
        <input matInput type="number" formControlName="scoreExpiryDays" />
        <mat-hint> Number of days before a credit check is considered expired and must be refreshed </mat-hint>
      </mat-form-field>

      <!-- Auto-Check Toggle -->
      <div class="toggle-row">
        <mat-slide-toggle formControlName="autoCheckOnApplication" color="primary"> Automatically request credit check on loan application </mat-slide-toggle>
        <p class="toggle-hint">When enabled, a credit check is automatically initiated when a client applies for a loan using this product.</p>
      </div>

      <!-- Block Approval Toggle -->
      <div class="toggle-row">
        <mat-slide-toggle formControlName="blockApprovalOnFailure" color="warn"> Block loan approval if credit score is below minimum </mat-slide-toggle>
        <p class="toggle-hint">When enabled, the loan approval button is disabled until the client's credit score meets the minimum threshold. When disabled, a warning is shown but approval can proceed.</p>
      </div>
    </div>
  </form>
</div>
```

---

## 4. Shared CRB Components

### 4.1 CRB Score Card Component

**Location:** `src/app/shared/crb/crb-score-card/`

```typescript
import { Component, Input } from '@angular/core';

@Component({
  selector: 'mifosx-crb-score-card',
  templateUrl: './crb-score-card.component.html',
  styleUrls: ['./crb-score-card.component.scss']
})
export class CRBScoreCardComponent {
  @Input() score: number = 0;
  @Input() rating: string = '';
  @Input() lastCheckDate: string = '';
  @Input() bureau: string = '';
  @Input() isExpired: boolean = false;

  /**
   * Get the color for the score gauge based on score ranges.
   * 0-300: Red (Poor)
   * 301-500: Yellow/Orange (Fair)
   * 501-700: Green (Good)
   * 701+: Blue (Excellent)
   */
  getScoreColor(): string {
    if (this.score <= 300) return '#f44336';
    if (this.score <= 500) return '#ff9800';
    if (this.score <= 700) return '#4caf50';
    return '#2196f3';
  }

  /**
   * Get the rating label based on the score.
   */
  getRatingLabel(): string {
    if (this.rating) return this.rating;
    if (this.score <= 300) return 'Poor';
    if (this.score <= 500) return 'Fair';
    if (this.score <= 700) return 'Good';
    return 'Excellent';
  }

  /**
   * Get the score as a percentage for the circular gauge (0-100).
   * Assumes a maximum score of 900.
   */
  getScorePercentage(): number {
    return Math.min(100, Math.max(0, (this.score / 900) * 100));
  }

  /**
   * Calculate the stroke-dashoffset for the circular gauge SVG.
   * Circle circumference = 2 * PI * radius.
   * For r=54: circumference = 339.29
   */
  getGaugeOffset(): number {
    const circumference = 339.29;
    const percentage = this.getScorePercentage();
    return circumference - (percentage / 100) * circumference;
  }
}
```

### 4.2 Score Card Template

```html
<mat-card class="score-card">
  <mat-card-content>
    <div class="score-card-layout">
      <!-- Circular Score Gauge -->
      <div class="score-gauge">
        <svg viewBox="0 0 120 120" class="gauge-svg">
          <!-- Background circle -->
          <circle cx="60" cy="60" r="54" fill="none" stroke="#e0e0e0" stroke-width="8" />
          <!-- Score arc -->
          <circle cx="60" cy="60" r="54" fill="none" [attr.stroke]="getScoreColor()" stroke-width="8" stroke-linecap="round" [style.stroke-dasharray]="339.29" [style.stroke-dashoffset]="getGaugeOffset()" transform="rotate(-90 60 60)" />
        </svg>
        <div class="score-value">
          <span class="score-number" [style.color]="getScoreColor()">{{ score }}</span>
          <span class="score-max">/ 900</span>
        </div>
      </div>

      <!-- Score Details -->
      <div class="score-details">
        <div class="rating-badge" [style.background-color]="getScoreColor()">{{ getRatingLabel() }}</div>

        <div class="detail-rows">
          <div class="detail-row">
            <span class="detail-label">Bureau:</span>
            <span class="detail-value">{{ bureau }}</span>
          </div>
          <div class="detail-row">
            <span class="detail-label">Last Checked:</span>
            <span class="detail-value">
              {{ lastCheckDate | date:'mediumDate' }}
              <span *ngIf="isExpired" class="expired-badge">EXPIRED</span>
            </span>
          </div>
        </div>
      </div>
    </div>
  </mat-card-content>
</mat-card>
```

### 4.3 Score Card Styles

```scss
.score-card {
  margin-bottom: 24px;
}

.score-card-layout {
  display: flex;
  align-items: center;
  gap: 32px;
  padding: 16px;
}

.score-gauge {
  position: relative;
  width: 140px;
  height: 140px;
  flex-shrink: 0;

  .gauge-svg {
    width: 100%;
    height: 100%;
  }

  .score-value {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    text-align: center;

    .score-number {
      display: block;
      font-size: 32px;
      font-weight: 700;
      line-height: 1;
    }

    .score-max {
      display: block;
      font-size: 12px;
      color: rgba(0, 0, 0, 0.38);
      margin-top: 2px;
    }
  }
}

.score-details {
  flex: 1;

  .rating-badge {
    display: inline-block;
    color: white;
    padding: 4px 16px;
    border-radius: 16px;
    font-size: 14px;
    font-weight: 600;
    margin-bottom: 16px;
  }

  .detail-rows {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .detail-row {
    display: flex;
    gap: 8px;

    .detail-label {
      color: rgba(0, 0, 0, 0.54);
      min-width: 100px;
    }

    .detail-value {
      font-weight: 500;
    }
  }

  .expired-badge {
    display: inline-block;
    background: #ff9800;
    color: white;
    padding: 1px 6px;
    border-radius: 4px;
    font-size: 11px;
    font-weight: 600;
    margin-left: 4px;
  }
}
```

### 4.4 CRB Check Dialog Component

**Location:** `src/app/shared/crb/crb-check-dialog/`

```typescript
import { Component, Inject, OnInit } from '@angular/core';
import { UntypedFormBuilder, UntypedFormGroup, Validators } from '@angular/forms';
import { MatDialogRef, MAT_DIALOG_DATA } from '@angular/material/dialog';
import { CRBService } from '../crb.service';

@Component({
  selector: 'mifosx-crb-check-dialog',
  templateUrl: './crb-check-dialog.component.html',
  styleUrls: ['./crb-check-dialog.component.scss']
})
export class CRBCheckDialogComponent implements OnInit {
  checkForm: UntypedFormGroup;
  bureaus: { id: number; name: string }[] = [];
  isSubmitting: boolean = false;

  constructor(
    private fb: UntypedFormBuilder,
    private crbService: CRBService,
    private dialogRef: MatDialogRef<CRBCheckDialogComponent>,
    @Inject(MAT_DIALOG_DATA) public data: { clientId: number }
  ) {}

  ngOnInit(): void {
    this.checkForm = this.fb.group({
      bureauId: [
        '',
        Validators.required
      ],
      consentGiven: [
        false,
        Validators.requiredTrue
      ],
      note: ['']
    });

    // Load available CRB providers
    this.crbService.getCRBProviders().subscribe((providers) => {
      this.bureaus = providers;
      if (providers.length === 1) {
        this.checkForm.get('bureauId').setValue(providers[0].id);
      }
    });
  }

  submit(): void {
    if (this.checkForm.invalid) return;

    this.isSubmitting = true;
    const payload = {
      bureauId: this.checkForm.get('bureauId').value,
      consentGiven: this.checkForm.get('consentGiven').value,
      note: this.checkForm.get('note').value
    };

    this.crbService.requestCreditCheck(this.data.clientId, payload).subscribe({
      next: (result) => {
        this.isSubmitting = false;
        this.dialogRef.close(result);
      },
      error: (error) => {
        this.isSubmitting = false;
        // Error handling via notification service
      }
    });
  }

  cancel(): void {
    this.dialogRef.close(null);
  }
}
```

### 4.5 CRB Check Dialog Template

```html
<h2 mat-dialog-title>Request Credit Check</h2>

<mat-dialog-content>
  <form [formGroup]="checkForm">
    <!-- Bureau Selection -->
    <mat-form-field class="full-width">
      <mat-label>Credit Bureau</mat-label>
      <mat-select formControlName="bureauId">
        <mat-option *ngFor="let bureau of bureaus" [value]="bureau.id"> {{ bureau.name }} </mat-option>
      </mat-select>
      <mat-error *ngIf="checkForm.get('bureauId').hasError('required')"> Please select a credit bureau </mat-error>
    </mat-form-field>

    <!-- Note -->
    <mat-form-field class="full-width">
      <mat-label>Note (Optional)</mat-label>
      <textarea matInput formControlName="note" rows="3" placeholder="Add any notes regarding this credit check..."> </textarea>
    </mat-form-field>

    <!-- Consent Checkbox -->
    <div class="consent-section">
      <mat-checkbox formControlName="consentGiven" color="primary"> I confirm that the client has given consent for this credit check to be performed and understands that their credit information will be accessed from the selected bureau. </mat-checkbox>
      <mat-error *ngIf="checkForm.get('consentGiven').touched && checkForm.get('consentGiven').hasError('required')"> Client consent is required to perform a credit check </mat-error>
    </div>
  </form>
</mat-dialog-content>

<mat-dialog-actions align="end">
  <button mat-button (click)="cancel()">Cancel</button>
  <button mat-raised-button color="primary" [disabled]="checkForm.invalid || isSubmitting" (click)="submit()">
    <mat-spinner diameter="20" *ngIf="isSubmitting"></mat-spinner>
    <span *ngIf="!isSubmitting">Request Check</span>
  </button>
</mat-dialog-actions>
```

### 4.6 CRB History Table Component

**Location:** `src/app/shared/crb/crb-history-table/`

```typescript
import { Component, Input, Output, EventEmitter, ViewChild, OnInit, OnChanges, SimpleChanges } from '@angular/core';
import { MatPaginator } from '@angular/material/paginator';
import { MatSort } from '@angular/material/sort';
import { MatTableDataSource } from '@angular/material/table';
import { CreditCheck } from '../crb.service';

@Component({
  selector: 'mifosx-crb-history-table',
  templateUrl: './crb-history-table.component.html',
  styleUrls: ['./crb-history-table.component.scss']
})
export class CRBHistoryTableComponent implements OnInit, OnChanges {
  @Input() creditChecks: CreditCheck[] = [];
  @Output() downloadReport = new EventEmitter<number>();

  displayedColumns: string[] = [
    'checkDate',
    'bureauName',
    'score',
    'rating',
    'status',
    'actions'
  ];

  dataSource: MatTableDataSource<CreditCheck>;

  @ViewChild(MatPaginator) paginator: MatPaginator;
  @ViewChild(MatSort) sort: MatSort;

  ngOnInit(): void {
    this.initDataSource();
  }

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['creditChecks']) {
      this.initDataSource();
    }
  }

  private initDataSource(): void {
    this.dataSource = new MatTableDataSource(this.creditChecks);
    if (this.paginator) this.dataSource.paginator = this.paginator;
    if (this.sort) this.dataSource.sort = this.sort;
  }

  getRatingColor(rating: string): string {
    switch (rating) {
      case 'EXCELLENT':
        return '#2196f3';
      case 'GOOD':
        return '#4caf50';
      case 'FAIR':
        return '#ff9800';
      case 'POOR':
        return '#f44336';
      default:
        return 'inherit';
    }
  }

  getStatusIcon(status: string): string {
    switch (status) {
      case 'COMPLETED':
        return 'check_circle';
      case 'PENDING':
        return 'hourglass_empty';
      case 'FAILED':
        return 'error';
      default:
        return 'help';
    }
  }

  onDownloadReport(checkId: number): void {
    this.downloadReport.emit(checkId);
  }
}
```

### 4.7 History Table Template

```html
<div class="crb-history">
  <h3>Credit Check History</h3>

  <table mat-table [dataSource]="dataSource" matSort class="full-width">
    <!-- Check Date -->
    <ng-container matColumnDef="checkDate">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Date</th>
      <td mat-cell *matCellDef="let check">{{ check.checkDate | date:'mediumDate' }}</td>
    </ng-container>

    <!-- Bureau Name -->
    <ng-container matColumnDef="bureauName">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Bureau</th>
      <td mat-cell *matCellDef="let check">{{ check.bureauName }}</td>
    </ng-container>

    <!-- Score -->
    <ng-container matColumnDef="score">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Score</th>
      <td mat-cell *matCellDef="let check">
        <strong>{{ check.score }}</strong>
      </td>
    </ng-container>

    <!-- Rating -->
    <ng-container matColumnDef="rating">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Rating</th>
      <td mat-cell *matCellDef="let check">
        <span class="rating-chip" [style.background-color]="getRatingColor(check.rating)"> {{ check.rating }} </span>
      </td>
    </ng-container>

    <!-- Status -->
    <ng-container matColumnDef="status">
      <th mat-header-cell *matHeaderCellDef mat-sort-header>Status</th>
      <td mat-cell *matCellDef="let check">
        <span class="status-cell">
          <mat-icon>{{ getStatusIcon(check.status) }}</mat-icon>
          {{ check.status }}
        </span>
      </td>
    </ng-container>

    <!-- Actions -->
    <ng-container matColumnDef="actions">
      <th mat-header-cell *matHeaderCellDef>Report</th>
      <td mat-cell *matCellDef="let check">
        <button mat-icon-button *ngIf="check.reportUrl && check.status === 'COMPLETED'" (click)="onDownloadReport(check.id)" matTooltip="Download Report">
          <mat-icon>download</mat-icon>
        </button>
      </td>
    </ng-container>

    <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
    <tr mat-row *matRowDef="let row; columns: displayedColumns;"></tr>
  </table>

  <mat-paginator [pageSizeOptions]="[5, 10, 25]" showFirstLastButtons> </mat-paginator>
</div>
```

---

## 5. CRB Service

### 5.1 Service Implementation

**Location:** `src/app/shared/crb/crb.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface CreditCheck {
  id: number;
  clientId: number;
  bureauName: string;
  checkDate: string;
  score: number;
  rating: 'EXCELLENT' | 'GOOD' | 'FAIR' | 'POOR';
  status: 'COMPLETED' | 'PENDING' | 'FAILED';
  reportUrl?: string;
}

export interface CreditScore {
  currentScore: number;
  rating: string;
  lastCheckDate: string;
  bureau: string;
  isExpired: boolean;
}

export interface CRBCheckRequest {
  bureauId: number;
  consentGiven: boolean;
  note?: string;
}

export interface CRBProvider {
  id: number;
  name: string;
  apiUrl?: string;
}

export interface CRBProductConfig {
  crbCheckEnabled: boolean;
  minimumCreditScore: number;
  crbProviderId: number | null;
  autoCheckOnApplication: boolean;
  blockApprovalOnFailure: boolean;
  scoreExpiryDays: number;
}

@Injectable({ providedIn: 'root' })
export class CRBService {
  constructor(private http: HttpClient) {}

  // --- Credit Checks ---

  /**
   * Get all credit checks for a client.
   * Uses Fineract's credit check module if available,
   * or a custom API endpoint for external CRB integration.
   *
   * Fineract endpoint: GET /clients/{clientId}/creditchecks
   */
  getCreditChecks(clientId: number): Observable<CreditCheck[]> {
    return this.http.get<any[]>(`/clients/${clientId}/creditchecks`).pipe(map((checks) => checks.map(this.mapCreditCheck)));
  }

  /**
   * Request a new credit check for a client.
   * Initiates a CRB lookup with the specified bureau.
   * The check may complete synchronously or asynchronously depending on the bureau.
   *
   * Fineract endpoint: POST /clients/{clientId}/creditchecks
   */
  requestCreditCheck(clientId: number, payload: CRBCheckRequest): Observable<CreditCheck> {
    return this.http.post<any>(`/clients/${clientId}/creditchecks`, payload).pipe(map(this.mapCreditCheck));
  }

  /**
   * Get the current (most recent valid) credit score for a client.
   * Returns null if no valid score exists.
   */
  getCreditScore(clientId: number): Observable<CreditScore> {
    return this.http.get<any>(`/clients/${clientId}/creditchecks/latest`).pipe(
      map((data) => ({
        currentScore: data.score,
        rating: this.calculateRating(data.score),
        lastCheckDate: data.checkDate,
        bureau: data.bureauName,
        isExpired: this.isScoreExpired(data.checkDate, data.expiryDays || 90)
      }))
    );
  }

  /**
   * Download the credit check report PDF.
   */
  downloadReport(checkId: number): Observable<Blob> {
    return this.http.get(`/creditchecks/${checkId}/report`, {
      responseType: 'blob'
    });
  }

  // --- CRB Providers ---

  /**
   * Get available CRB providers configured in the system.
   */
  getCRBProviders(): Observable<CRBProvider[]> {
    return this.http.get<CRBProvider[]>('/creditbureaus');
  }

  // --- Loan Product CRB Config ---

  /**
   * Get the CRB configuration for a specific loan product.
   * This is stored as part of the loan product metadata.
   */
  getLoanProductCRBConfig(loanProductId: number): Observable<CRBProductConfig> {
    return this.http.get<any>(`/loanproducts/${loanProductId}`).pipe(
      map((product) => ({
        crbCheckEnabled: product.crbCheckEnabled || false,
        minimumCreditScore: product.minimumCreditScore || 0,
        crbProviderId: product.crbProviderId || null,
        autoCheckOnApplication: product.autoCheckOnApplication || false,
        blockApprovalOnFailure: product.blockApprovalOnFailure || true,
        scoreExpiryDays: product.scoreExpiryDays || 90
      }))
    );
  }

  // --- Helper Methods ---

  /**
   * Calculate the rating category based on the numeric score.
   */
  private calculateRating(score: number): string {
    if (score <= 300) return 'POOR';
    if (score <= 500) return 'FAIR';
    if (score <= 700) return 'GOOD';
    return 'EXCELLENT';
  }

  /**
   * Check if a credit score has expired based on the check date and expiry period.
   */
  private isScoreExpired(checkDate: string, expiryDays: number): boolean {
    const checkTime = new Date(checkDate).getTime();
    const expiryTime = checkTime + expiryDays * 24 * 60 * 60 * 1000;
    return Date.now() > expiryTime;
  }

  /**
   * Map raw API response to CreditCheck interface.
   */
  private mapCreditCheck(raw: any): CreditCheck {
    return {
      id: raw.id,
      clientId: raw.clientId,
      bureauName: raw.bureauName || raw.bureau?.name || '',
      checkDate: raw.checkDate || raw.createdDate,
      score: raw.score || 0,
      rating: raw.rating || 'POOR',
      status: raw.status || 'PENDING',
      reportUrl: raw.reportUrl || null
    };
  }
}
```

---

## 6. Data Models

### 6.1 TypeScript Interfaces

All data models used across CRB components:

```typescript
// src/app/shared/crb/crb.models.ts

/** Result of a credit check from a CRB provider */
export interface CreditCheck {
  id: number;
  clientId: number;
  bureauName: string;
  checkDate: string;
  score: number;
  rating: 'EXCELLENT' | 'GOOD' | 'FAIR' | 'POOR';
  status: 'COMPLETED' | 'PENDING' | 'FAILED';
  reportUrl?: string;
}

/** Current credit score summary for a client */
export interface CreditScore {
  currentScore: number;
  rating: string;
  lastCheckDate: string;
  bureau: string;
  isExpired: boolean;
}

/** Payload for requesting a new credit check */
export interface CRBCheckRequest {
  bureauId: number;
  consentGiven: boolean;
  note?: string;
}

/** CRB provider configuration */
export interface CRBProvider {
  id: number;
  name: string;
  apiUrl?: string;
  apiKey?: string;
}

/** CRB configuration on a loan product */
export interface CRBProductConfig {
  crbCheckEnabled: boolean;
  minimumCreditScore: number;
  crbProviderId: number | null;
  autoCheckOnApplication: boolean;
  blockApprovalOnFailure: boolean;
  scoreExpiryDays: number;
}

/** Score range definition for rating categories */
export interface ScoreRange {
  min: number;
  max: number;
  rating: string;
  color: string;
  label: string;
}

export const SCORE_RANGES: ScoreRange[] = [
  { min: 0, max: 300, rating: 'POOR', color: '#f44336', label: 'Poor' },
  { min: 301, max: 500, rating: 'FAIR', color: '#ff9800', label: 'Fair' },
  { min: 501, max: 700, rating: 'GOOD', color: '#4caf50', label: 'Good' },
  { min: 701, max: 900, rating: 'EXCELLENT', color: '#2196f3', label: 'Excellent' }
];
```

---

## 7. Environment Configuration

### 7.1 CRB Environment Settings

**File:** `src/environments/environment.ts`

```typescript
export const environment = {
  // ... existing configuration
  crb: {
    /** Whether CRB integration is enabled globally */
    enabled: true,

    /** Configured CRB providers */
    providers: [
      {
        id: 1,
        name: 'TransUnion',
        apiUrl: '',
        apiKey: ''
      },
      {
        id: 2,
        name: 'Metropol',
        apiUrl: '',
        apiKey: ''
      }
    ],

    /** Whether to auto-check CRB on loan application */
    autoCheckOnLoanApplication: true,

    /** Number of days before a credit score is considered expired */
    scoreExpiryDays: 90,

    /** Maximum score value (used for gauge rendering) */
    maxScore: 900
  }
};
```

### 7.2 Production Environment

**File:** `src/environments/environment.prod.ts`

```typescript
export const environment = {
  // ... existing production configuration
  crb: {
    enabled: true,
    providers: [
      {
        id: 1,
        name: 'TransUnion',
        apiUrl: '', // Set via deployment config
        apiKey: '' // Set via deployment config
      },
      {
        id: 2,
        name: 'Metropol',
        apiUrl: '', // Set via deployment config
        apiKey: '' // Set via deployment config
      }
    ],
    autoCheckOnLoanApplication: true,
    scoreExpiryDays: 90,
    maxScore: 900
  }
};
```

### 7.3 Configuration Notes

API keys for CRB providers must be set via environment variables at deployment time. The Fineract backend handles the actual CRB API communication. The Angular frontend configures the integration and displays the results.

---

## 8. CRB Module

### 8.1 Shared CRB Module

**Location:** `src/app/shared/crb/crb.module.ts`

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { ReactiveFormsModule } from '@angular/forms';
import { MaterialModule } from '../../material.module';

import { CRBScoreCardComponent } from './crb-score-card/crb-score-card.component';
import { CRBCheckDialogComponent } from './crb-check-dialog/crb-check-dialog.component';
import { CRBHistoryTableComponent } from './crb-history-table/crb-history-table.component';
import { CRBService } from './crb.service';

@NgModule({
  declarations: [
    CRBScoreCardComponent,
    CRBCheckDialogComponent,
    CRBHistoryTableComponent
  ],
  imports: [
    CommonModule,
    ReactiveFormsModule,
    MaterialModule
  ],
  exports: [
    CRBScoreCardComponent,
    CRBCheckDialogComponent,
    CRBHistoryTableComponent
  ],
  providers: [
    CRBService
  ]
})
export class CRBModule {}
```

---

## 9. Fineract API Integration

### 9.1 API Endpoints

| Method | Endpoint                                  | Purpose                            |
| ------ | ----------------------------------------- | ---------------------------------- |
| `GET`  | `/clients/{clientId}/creditchecks`        | List credit checks for a client    |
| `POST` | `/clients/{clientId}/creditchecks`        | Request a new credit check         |
| `GET`  | `/clients/{clientId}/creditchecks/latest` | Get most recent valid credit score |
| `GET`  | `/creditchecks/{checkId}/report`          | Download credit check report PDF   |
| `GET`  | `/creditbureaus`                          | List configured CRB providers      |
| `GET`  | `/loanproducts/{id}`                      | Get loan product with CRB config   |

### 9.2 Credit Check Request Payload

```json
{
  "bureauId": 1,
  "consentGiven": true,
  "note": "Pre-loan approval check"
}
```

### 9.3 Credit Check Response

```json
{
  "id": 42,
  "clientId": 123,
  "bureauName": "TransUnion",
  "checkDate": "13 March 2026",
  "score": 650,
  "rating": "GOOD",
  "status": "COMPLETED",
  "reportUrl": "/creditchecks/42/report"
}
```

### 9.4 Webhook for Async Results

For CRB providers that return results asynchronously, configure a webhook endpoint in Fineract:

```
POST /fineract-provider/api/v1/hooks
{
  "name": "CRB Result Webhook",
  "isActive": true,
  "displayName": "CRB Credit Check Result",
  "templateId": 1,
  "contentType": "json",
  "events": [
    { "actionName": "CREDIT_CHECK_COMPLETED", "entityName": "CLIENT" }
  ],
  "config": [
    { "fieldName": "Payload URL", "fieldValue": "https://your-app/api/crb/webhook" },
    { "fieldName": "Content Type", "fieldValue": "application/json" }
  ]
}
```

---

## 10. New Files Summary

| File Path                                                                                                        | Purpose                                |
| ---------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| `src/app/clients/clients-view/crb-account-tab/crb-account-tab.component.ts`                                      | CRB tab in client detail view          |
| `src/app/clients/clients-view/crb-account-tab/crb-account-tab.component.html`                                    | CRB tab template                       |
| `src/app/clients/clients-view/crb-account-tab/crb-account-tab.component.scss`                                    | CRB tab styles                         |
| `src/app/products/loan-products/loan-product-stepper/loan-product-crb-step/loan-product-crb-step.component.ts`   | CRB config step in loan product wizard |
| `src/app/products/loan-products/loan-product-stepper/loan-product-crb-step/loan-product-crb-step.component.html` | CRB step template                      |
| `src/app/products/loan-products/loan-product-stepper/loan-product-crb-step/loan-product-crb-step.component.scss` | CRB step styles                        |
| `src/app/shared/crb/crb.module.ts`                                                                               | CRB shared module                      |
| `src/app/shared/crb/crb.service.ts`                                                                              | CRB service                            |
| `src/app/shared/crb/crb.models.ts`                                                                               | CRB data models and interfaces         |
| `src/app/shared/crb/crb-score-card/crb-score-card.component.ts`                                                  | Score card with circular gauge         |
| `src/app/shared/crb/crb-score-card/crb-score-card.component.html`                                                | Score card template                    |
| `src/app/shared/crb/crb-score-card/crb-score-card.component.scss`                                                | Score card styles                      |
| `src/app/shared/crb/crb-check-dialog/crb-check-dialog.component.ts`                                              | Credit check request dialog            |
| `src/app/shared/crb/crb-check-dialog/crb-check-dialog.component.html`                                            | Dialog template                        |
| `src/app/shared/crb/crb-check-dialog/crb-check-dialog.component.scss`                                            | Dialog styles                          |
| `src/app/shared/crb/crb-history-table/crb-history-table.component.ts`                                            | Credit check history table             |
| `src/app/shared/crb/crb-history-table/crb-history-table.component.html`                                          | History table template                 |
| `src/app/shared/crb/crb-history-table/crb-history-table.component.scss`                                          | History table styles                   |

## 11. Modified Files Summary

| File Path                                                                                 | Changes                                          |
| ----------------------------------------------------------------------------------------- | ------------------------------------------------ |
| `src/app/clients/clients-view/clients-view.component.ts`                                  | Add CRB tab to client view tabs                  |
| `src/app/clients/clients-view/clients-view-routing.module.ts`                             | Add CRB tab route                                |
| `src/app/clients/clients.module.ts`                                                       | Import CRBModule, declare CRBAccountTabComponent |
| `src/app/products/loan-products/loan-product-stepper/loan-product-stepper.component.ts`   | Add CRB step to stepper                          |
| `src/app/products/loan-products/loan-product-stepper/loan-product-stepper.component.html` | Add CRB step in template                         |
| `src/app/products/loan-products/loan-products.module.ts`                                  | Register CRB step component                      |
| `src/app/loans/loans-view/loan-account-actions/approve-loan/approve-loan.component.ts`    | Add CRB check gate                               |
| `src/app/loans/loans-view/loan-account-actions/approve-loan/approve-loan.component.html`  | Add CRB status and traffic light                 |
| `src/environments/environment.ts`                                                         | Add CRB config                                   |
| `src/environments/environment.prod.ts`                                                    | Add CRB production config                        |

---

## 12. Testing Checklist

- [ ] CRB tab appears in client detail view navigation
- [ ] Score card displays circular gauge with correct colors for each score range
- [ ] Score card shows expired badge when credit check is past expiry period
- [ ] Credit check dialog opens with bureau selection dropdown
- [ ] Credit check dialog requires consent checkbox before submission
- [ ] Credit check dialog submits request and refreshes data on success
- [ ] Credit history table displays all past checks with correct columns
- [ ] History table supports sorting and pagination
- [ ] Report download button appears only for completed checks with reportUrl
- [ ] CRB step appears in loan product creation wizard
- [ ] CRB toggle enables/disables required fields (provider, minimum score)
- [ ] Minimum score validates within 0-900 range
- [ ] Score expiry days validates within 1-365 range
- [ ] Loan approval page shows CRB check section when product requires it
- [ ] Traffic light indicator shows correct color (red/yellow/green) based on score status
- [ ] Loan approval is blocked when CRB check required but not passed (and blockApprovalOnFailure is true)
- [ ] "Request New Credit Check" button opens dialog from loan approval page
- [ ] CRB status refreshes after a new credit check is completed
- [ ] Environment configuration loads correctly with provider list
- [ ] All Fineract API calls use correct endpoints and payload formats
