# TECH-009: SMS Campaign Enhancement Specification

## Overview

Enhance the existing SMS campaigns module at `src/app/organization/sms-campaigns/` to support template variables, campaign type configuration, and message preview capabilities as defined in the M-SACCO wireframes.

**Current State:** Basic CRUD for SMS campaigns exists with campaign name, message, type, and status fields. The module supports creating, viewing, editing, and deleting campaigns but lacks template variable insertion, message preview, and advanced campaign type configuration.

**Target State:** A fully featured SMS campaign system with click-to-insert template variables, real-time message preview with sample data, configurable business rules, and support for all M-SACCO campaign types.

---

## 1. Template Variable System

### 1.1 Available Variables

Variables are sourced from the Fineract reports API and map to client, loan, and office data. The full set of available variables:

| Variable              | Source | Description                           |
| --------------------- | ------ | ------------------------------------- |
| `{{firstname}}`       | Client | Client first name                     |
| `{{middlename}}`      | Client | Client middle name                    |
| `{{lastname}}`        | Client | Client last name                      |
| `{{FullName}}`        | Client | Full name (first + middle + last)     |
| `{{MobileNo}}`        | Client | Primary mobile number                 |
| `{{LoanAmount}}`      | Loan   | Approved loan principal amount        |
| `{{LoanOutstanding}}` | Loan   | Current outstanding loan balance      |
| `{{LoanDisbursed}}`   | Loan   | Disbursed loan amount                 |
| `{{PaymentDueDate}}`  | Loan   | Next repayment due date               |
| `{{TotalDue}}`        | Loan   | Total amount due for next installment |
| `{{officenumber}}`    | Office | Branch/office telephone number        |
| `{{id}}`              | Client | Client ID in the system               |

### 1.2 Variable Panel UI

Add an "Available Fields" panel to the right side of the message textarea in the campaign stepper component.

**Layout:**

```
+------------------------------------------+------------------+
| Message Textarea                         | Available Fields  |
|                                          |                   |
| Dear {{firstname}}, your loan of         | [Client]          |
| {{LoanAmount}} is due on                 |  firstname        |
| {{PaymentDueDate}}.                      |  middlename       |
|                                          |  lastname         |
|                                          |  FullName         |
|                                          |  MobileNo         |
|                                          |  id               |
|                                          |                   |
|                                          | [Loan]            |
|                                          |  LoanAmount       |
|                                          |  LoanOutstanding  |
|                                          |  LoanDisbursed    |
|                                          |  PaymentDueDate   |
|                                          |  TotalDue         |
|                                          |                   |
|                                          | [Office]          |
|                                          |  officenumber     |
+------------------------------------------+------------------+
```

Variables are grouped by category (Client, Loan, Office) with expandable/collapsible sections. Each variable is displayed as a clickable chip.

### 1.3 Click-to-Insert Implementation

**File:** `src/app/organization/sms-campaigns/sms-campaign-stepper/sms-campaign-stepper.component.ts`

Add a `@ViewChild` reference to the message textarea and implement the insertion method:

```typescript
import { Component, ViewChild, ElementRef } from '@angular/core';

@Component({
  selector: 'mifosx-sms-campaign-stepper',
  templateUrl: './sms-campaign-stepper.component.html',
  styleUrls: ['./sms-campaign-stepper.component.scss']
})
export class SmsCampaignStepperComponent {
  @ViewChild('messageTextarea') messageTextarea: ElementRef<HTMLTextAreaElement>;

  /** Template variables grouped by category */
  templateVariables: TemplateVariableGroup[] = [
    {
      category: 'Client',
      expanded: true,
      variables: [
        { key: 'firstname', label: 'First Name' },
        { key: 'middlename', label: 'Middle Name' },
        { key: 'lastname', label: 'Last Name' },
        { key: 'FullName', label: 'Full Name' },
        { key: 'MobileNo', label: 'Mobile Number' },
        { key: 'id', label: 'Client ID' }
      ]
    },
    {
      category: 'Loan',
      expanded: true,
      variables: [
        { key: 'LoanAmount', label: 'Loan Amount' },
        { key: 'LoanOutstanding', label: 'Loan Outstanding' },
        { key: 'LoanDisbursed', label: 'Loan Disbursed' },
        { key: 'PaymentDueDate', label: 'Payment Due Date' },
        { key: 'TotalDue', label: 'Total Due' }
      ]
    },
    {
      category: 'Office',
      expanded: false,
      variables: [
        { key: 'officenumber', label: 'Office Number' }]
    }
  ];

  /**
   * Inserts a template variable at the current cursor position in the message textarea.
   * Wraps the variable key in double curly braces: {{variableKey}}
   * Restores cursor position after insertion to allow continuous typing.
   */
  insertVariable(variable: string): void {
    const textarea = this.messageTextarea.nativeElement;
    const start = textarea.selectionStart;
    const end = textarea.selectionEnd;
    const currentValue = this.campaignForm.get('message').value || '';
    const insertText = `{{${variable}}}`;
    const newValue = currentValue.substring(0, start) + insertText + currentValue.substring(end);

    this.campaignForm.get('message').setValue(newValue);

    // Restore cursor position after the inserted variable
    const newCursorPos = start + insertText.length;
    setTimeout(() => {
      textarea.focus();
      textarea.setSelectionRange(newCursorPos, newCursorPos);
    });
  }

  /** Toggle expand/collapse of a variable category group */
  toggleCategory(group: TemplateVariableGroup): void {
    group.expanded = !group.expanded;
  }
}
```

### 1.4 Variable Highlighting in Textarea

Since native `<textarea>` elements do not support rich text, implement a highlight overlay approach:

**File:** `src/app/organization/sms-campaigns/sms-campaign-stepper/variable-highlight/variable-highlight.component.ts`

```typescript
import { Component, Input, OnChanges, SimpleChanges } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Component({
  selector: 'mifosx-variable-highlight',
  template: ` <div class="highlight-overlay" [innerHTML]="highlightedHtml"></div> `,
  styleUrls: ['./variable-highlight.component.scss']
})
export class VariableHighlightComponent implements OnChanges {
  @Input() message: string = '';
  highlightedHtml: SafeHtml;

  constructor(private sanitizer: DomSanitizer) {}

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['message']) {
      this.updateHighlight();
    }
  }

  private updateHighlight(): void {
    const escaped = this.escapeHtml(this.message);
    const highlighted = escaped.replace(/\{\{(\w+)\}\}/g, '<span class="variable-chip">{{$1}}</span>');
    this.highlightedHtml = this.sanitizer.bypassSecurityTrustHtml(highlighted);
  }

  private escapeHtml(text: string): string {
    const div = document.createElement('div');
    div.appendChild(document.createTextNode(text));
    return div.innerHTML;
  }
}
```

**Styles** (`variable-highlight.component.scss`):

```scss
.highlight-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  white-space: pre-wrap;
  word-wrap: break-word;
  font-family: inherit;
  font-size: inherit;
  line-height: inherit;
  padding: inherit;
  color: transparent;

  .variable-chip {
    background-color: rgba(33, 150, 243, 0.15);
    border: 1px solid rgba(33, 150, 243, 0.4);
    border-radius: 4px;
    padding: 0 2px;
    color: #1976d2;
  }
}
```

### 1.5 Template Variable Data Types

```typescript
interface TemplateVariable {
  /** Variable key used in the template (e.g., 'firstname') */
  key: string;
  /** Human-readable label for the UI (e.g., 'First Name') */
  label: string;
  /** Optional description shown on hover */
  description?: string;
}

interface TemplateVariableGroup {
  /** Category name (e.g., 'Client', 'Loan', 'Office') */
  category: string;
  /** Whether the group is expanded in the UI */
  expanded: boolean;
  /** Variables in this category */
  variables: TemplateVariable[];
}
```

---

## 2. Campaign Type Configuration

### 2.1 Business Rule Selection

Add a business rule dropdown to the first step of the campaign stepper. The dropdown options are populated from the Fineract API endpoint `GET /smscampaigns/template`.

**Business Rule Options:**

| Rule ID | Rule Name               | Description                               |
| ------- | ----------------------- | ----------------------------------------- |
| 1       | Client Arrears          | Clients with overdue loan repayments      |
| 2       | Loan Repayment Reminder | Upcoming loan repayment due dates         |
| 3       | Payment Reminder        | General payment reminders for individuals |
| 4       | Payment Due             | Payments due within a configured window   |
| 5       | Savings Activation      | Savings account activation notices        |

### 2.2 Business Rule Parameters

Each business rule may have dynamic parameters. When a rule is selected, fetch the parameter schema from the API and render form fields dynamically.

**File:** `src/app/organization/sms-campaigns/sms-campaign-stepper/business-rule-params/business-rule-params.component.ts`

```typescript
import { Component, Input, OnChanges, SimpleChanges, Output, EventEmitter } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';

interface BusinessRuleParam {
  name: string;
  type: 'TEXT' | 'NUMBER' | 'DATE' | 'SELECT';
  label: string;
  required: boolean;
  options?: { id: number; value: string }[];
  defaultValue?: any;
}

@Component({
  selector: 'mifosx-business-rule-params',
  templateUrl: './business-rule-params.component.html',
  styleUrls: ['./business-rule-params.component.scss']
})
export class BusinessRuleParamsComponent implements OnChanges {
  @Input() params: BusinessRuleParam[] = [];
  @Output() paramsChanged = new EventEmitter<Record<string, any>>();

  paramsForm: UntypedFormGroup;

  constructor(private fb: UntypedFormBuilder) {}

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['params']) {
      this.buildForm();
    }
  }

  private buildForm(): void {
    const controls: Record<string, any> = {};
    this.params.forEach((param) => {
      const validators = param.required ? [Validators.required] : [];
      controls[param.name] = [
        param.defaultValue || '',
        validators
      ];
    });
    this.paramsForm = this.fb.group(controls);
    this.paramsForm.valueChanges.subscribe((value) => this.paramsChanged.emit(value));
  }
}
```

**Template** (`business-rule-params.component.html`):

```html
<form [formGroup]="paramsForm" *ngIf="paramsForm" class="params-container">
  <ng-container *ngFor="let param of params">
    <!-- Text input -->
    <mat-form-field *ngIf="param.type === 'TEXT'" class="full-width">
      <mat-label>{{ param.label }}</mat-label>
      <input matInput [formControlName]="param.name" />
    </mat-form-field>

    <!-- Number input -->
    <mat-form-field *ngIf="param.type === 'NUMBER'" class="full-width">
      <mat-label>{{ param.label }}</mat-label>
      <input matInput type="number" [formControlName]="param.name" />
    </mat-form-field>

    <!-- Date picker -->
    <mat-form-field *ngIf="param.type === 'DATE'" class="full-width">
      <mat-label>{{ param.label }}</mat-label>
      <input matInput [matDatepicker]="picker" [formControlName]="param.name" />
      <mat-datepicker-toggle matSuffix [for]="picker"></mat-datepicker-toggle>
      <mat-datepicker #picker></mat-datepicker>
    </mat-form-field>

    <!-- Select dropdown -->
    <mat-form-field *ngIf="param.type === 'SELECT'" class="full-width">
      <mat-label>{{ param.label }}</mat-label>
      <mat-select [formControlName]="param.name">
        <mat-option *ngFor="let opt of param.options" [value]="opt.id"> {{ opt.value }} </mat-option>
      </mat-select>
    </mat-form-field>
  </ng-container>
</form>
```

### 2.3 Schedule Configuration

**Schedule Types:**

| Type      | Description                      | UI Fields                                                                  |
| --------- | -------------------------------- | -------------------------------------------------------------------------- |
| Immediate | Send immediately upon activation | None additional                                                            |
| Scheduled | Send at a specific date and time | Date picker, Time picker                                                   |
| Recurring | Send on a recurring schedule     | Recurrence pattern (Daily/Weekly/Monthly), Start date, End date (optional) |

**Trigger Types:**

| Type           | Description                                                                               |
| -------------- | ----------------------------------------------------------------------------------------- |
| Direct SMS     | Send SMS directly to recipients                                                           |
| Schedule-based | Campaign runs on schedule, selecting recipients matching business rules at execution time |

**Implementation in stepper form:**

```typescript
// Schedule step form controls
scheduleForm = this.fb.group({
  triggerType: ['DIRECT', Validators.required],
  scheduleType: ['IMMEDIATE', Validators.required],
  scheduledDate: [''],
  scheduledTime: [''],
  recurrencePattern: [''],
  recurrenceStartDate: [''],
  recurrenceEndDate: ['']
});

// Conditional validators based on schedule type
onScheduleTypeChange(type: string): void {
  const dateControl = this.scheduleForm.get('scheduledDate');
  const timeControl = this.scheduleForm.get('scheduledTime');
  const recurrenceControl = this.scheduleForm.get('recurrencePattern');

  if (type === 'SCHEDULED') {
    dateControl.setValidators([Validators.required]);
    timeControl.setValidators([Validators.required]);
    recurrenceControl.clearValidators();
  } else if (type === 'RECURRING') {
    recurrenceControl.setValidators([Validators.required]);
    this.scheduleForm.get('recurrenceStartDate').setValidators([Validators.required]);
    dateControl.clearValidators();
    timeControl.clearValidators();
  } else {
    dateControl.clearValidators();
    timeControl.clearValidators();
    recurrenceControl.clearValidators();
  }

  dateControl.updateValueAndValidity();
  timeControl.updateValueAndValidity();
  recurrenceControl.updateValueAndValidity();
}
```

---

## 3. Message Preview

### 3.1 Campaign Preview Component

**Location:** `src/app/organization/sms-campaigns/view-campaign/campaign-preview/`

**Files:**

- `campaign-preview.component.ts`
- `campaign-preview.component.html`
- `campaign-preview.component.scss`

### 3.2 Component Specification

```typescript
import { Component, Input, OnInit, OnChanges, SimpleChanges } from '@angular/core';
import { SmsCampaignsService } from '../../sms-campaigns.service';

interface PreviewRecipient {
  clientId: number;
  name: string;
  mobileNo: string;
  renderedMessage: string;
}

interface MessageStats {
  characterCount: number;
  segmentCount: number;
  hasVariables: boolean;
  unresolvedVariables: string[];
}

@Component({
  selector: 'mifosx-campaign-preview',
  templateUrl: './campaign-preview.component.html',
  styleUrls: ['./campaign-preview.component.scss']
})
export class CampaignPreviewComponent implements OnInit, OnChanges {
  /** Raw message template with {{variable}} placeholders */
  @Input() messageTemplate: string = '';

  /** Campaign ID for fetching matching recipients */
  @Input() campaignId: number;

  /** Business rule ID for recipient matching */
  @Input() businessRuleId: number;

  /** Business rule parameters */
  @Input() ruleParams: Record<string, any>;

  previewRecipients: PreviewRecipient[] = [];
  messageStats: MessageStats;
  selectedRecipientIndex: number = 0;
  isLoading: boolean = false;

  /** Maximum recipients to show in preview */
  private readonly MAX_PREVIEW_RECIPIENTS = 5;

  /** SMS segment size constants */
  private readonly GSM_SINGLE_LIMIT = 160;
  private readonly GSM_MULTI_LIMIT = 153;
  private readonly UNICODE_SINGLE_LIMIT = 70;
  private readonly UNICODE_MULTI_LIMIT = 67;

  constructor(private smsCampaignsService: SmsCampaignsService) {}

  ngOnInit(): void {
    this.loadPreviewData();
  }

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['messageTemplate'] || changes['ruleParams']) {
      this.calculateMessageStats();
      if (this.previewRecipients.length > 0) {
        this.renderPreviews();
      }
    }
  }

  /**
   * Load sample recipients matching the business rule criteria.
   * Fetches the first 5 matching clients with their data for preview rendering.
   */
  loadPreviewData(): void {
    if (!this.businessRuleId) return;

    this.isLoading = true;
    this.smsCampaignsService.getPreviewRecipients(this.businessRuleId, this.ruleParams, this.MAX_PREVIEW_RECIPIENTS).subscribe({
      next: (recipients) => {
        this.previewRecipients = recipients;
        this.renderPreviews();
        this.isLoading = false;
      },
      error: () => {
        this.isLoading = false;
      }
    });
  }

  /**
   * Render the message template for each preview recipient by substituting
   * their actual data into the template variables.
   */
  renderPreviews(): void {
    this.previewRecipients = this.previewRecipients.map((recipient) => ({
      ...recipient,
      renderedMessage: this.resolveTemplate(this.messageTemplate, recipient)
    }));
    this.calculateMessageStats();
  }

  /**
   * Resolve template variables in the message using recipient data.
   * Unresolved variables remain as-is and are flagged in stats.
   */
  resolveTemplate(template: string, data: Record<string, any>): string {
    return template.replace(/\{\{(\w+)\}\}/g, (match, key) => {
      return data[key] !== undefined && data[key] !== null ? String(data[key]) : match;
    });
  }

  /**
   * Calculate message statistics: character count, SMS segment count,
   * and any unresolved variables.
   */
  calculateMessageStats(): void {
    const message = this.previewRecipients.length > 0 ? this.previewRecipients[this.selectedRecipientIndex]?.renderedMessage || this.messageTemplate : this.messageTemplate;

    const charCount = message.length;
    const isUnicode = this.containsUnicode(message);
    const segmentCount = this.calculateSegments(charCount, isUnicode);

    const unresolvedMatches = message.match(/\{\{(\w+)\}\}/g) || [];
    const unresolvedVariables = unresolvedMatches.map((m) => m.replace(/[{}]/g, ''));

    this.messageStats = {
      characterCount: charCount,
      segmentCount,
      hasVariables: /\{\{\w+\}\}/.test(this.messageTemplate),
      unresolvedVariables
    };
  }

  /**
   * Calculate the number of SMS segments required for a message.
   * GSM 7-bit: 160 chars for single, 153 per segment for multi-part.
   * Unicode: 70 chars for single, 67 per segment for multi-part.
   */
  private calculateSegments(charCount: number, isUnicode: boolean): number {
    if (charCount === 0) return 0;

    const singleLimit = isUnicode ? this.UNICODE_SINGLE_LIMIT : this.GSM_SINGLE_LIMIT;
    const multiLimit = isUnicode ? this.UNICODE_MULTI_LIMIT : this.GSM_MULTI_LIMIT;

    if (charCount <= singleLimit) return 1;
    return Math.ceil(charCount / multiLimit);
  }

  /**
   * Check if a string contains characters outside the GSM 7-bit character set.
   */
  private containsUnicode(text: string): boolean {
    // GSM 7-bit basic character set regex (simplified)
    const gsmRegex = /^[@£$¥èéùìòÇ\nØø\rÅåΔ_ΦΓΛΩΠΨΣΘΞÆæßÉ !"#¤%&'()*+,\-.\/0-9:;<=>?¡A-ZÄÖÑܧ¿a-zäöñüà]*$/;
    return !gsmRegex.test(text);
  }

  selectRecipient(index: number): void {
    this.selectedRecipientIndex = index;
    this.calculateMessageStats();
  }
}
```

### 3.3 Preview Template

```html
<mat-card class="campaign-preview-card">
  <mat-card-header>
    <mat-card-title>Message Preview</mat-card-title>
    <mat-card-subtitle> Preview how the message will appear to recipients </mat-card-subtitle>
  </mat-card-header>

  <mat-card-content>
    <!-- Message Stats Bar -->
    <div class="message-stats">
      <span class="stat">
        <mat-icon>text_fields</mat-icon>
        {{ messageStats?.characterCount || 0 }} characters
      </span>
      <span class="stat">
        <mat-icon>sms</mat-icon>
        {{ messageStats?.segmentCount || 0 }} SMS segment(s)
      </span>
      <span class="stat warning" *ngIf="messageStats?.unresolvedVariables?.length > 0">
        <mat-icon>warning</mat-icon>
        {{ messageStats.unresolvedVariables.length }} unresolved variable(s)
      </span>
    </div>

    <!-- Recipient Tabs -->
    <div class="recipient-tabs" *ngIf="previewRecipients.length > 0">
      <mat-chip-listbox>
        <mat-chip-option *ngFor="let recipient of previewRecipients; let i = index" [selected]="i === selectedRecipientIndex" (click)="selectRecipient(i)"> {{ recipient.name }} </mat-chip-option>
      </mat-chip-listbox>
    </div>

    <!-- Message Preview Display -->
    <div class="preview-message-container">
      <div class="phone-frame">
        <div class="phone-screen">
          <div class="sms-bubble">
            <p class="sms-text" *ngIf="previewRecipients.length > 0">{{ previewRecipients[selectedRecipientIndex]?.renderedMessage }}</p>
            <p class="sms-text placeholder" *ngIf="previewRecipients.length === 0">{{ messageTemplate || 'Enter a message template to see preview...' }}</p>
          </div>
        </div>
      </div>
    </div>

    <!-- Recipient Details -->
    <div class="recipient-details" *ngIf="previewRecipients.length > 0">
      <p><strong>To:</strong> {{ previewRecipients[selectedRecipientIndex]?.name }}</p>
      <p><strong>Mobile:</strong> {{ previewRecipients[selectedRecipientIndex]?.mobileNo }}</p>
    </div>

    <!-- Loading State -->
    <div class="loading-container" *ngIf="isLoading">
      <mat-spinner diameter="32"></mat-spinner>
      <p>Loading preview recipients...</p>
    </div>

    <!-- No Recipients State -->
    <div class="empty-state" *ngIf="!isLoading && previewRecipients.length === 0 && businessRuleId">
      <mat-icon>people_outline</mat-icon>
      <p>No matching recipients found for the selected business rule.</p>
    </div>
  </mat-card-content>
</mat-card>
```

### 3.4 Preview Styles

```scss
.campaign-preview-card {
  margin-top: 16px;
}

.message-stats {
  display: flex;
  gap: 16px;
  padding: 8px 0;
  border-bottom: 1px solid rgba(0, 0, 0, 0.12);
  margin-bottom: 16px;

  .stat {
    display: flex;
    align-items: center;
    gap: 4px;
    font-size: 13px;
    color: rgba(0, 0, 0, 0.6);

    mat-icon {
      font-size: 18px;
      width: 18px;
      height: 18px;
    }

    &.warning {
      color: #f57c00;
    }
  }
}

.recipient-tabs {
  margin-bottom: 16px;
}

.preview-message-container {
  display: flex;
  justify-content: center;
  padding: 16px 0;
}

.phone-frame {
  width: 320px;
  border: 2px solid #ccc;
  border-radius: 24px;
  padding: 32px 16px;
  background: #f5f5f5;

  .phone-screen {
    background: #fff;
    border-radius: 8px;
    padding: 16px;
    min-height: 120px;
  }

  .sms-bubble {
    background: #e3f2fd;
    border-radius: 12px 12px 12px 0;
    padding: 12px;
    max-width: 90%;

    .sms-text {
      margin: 0;
      font-size: 14px;
      line-height: 1.5;
      word-break: break-word;

      &.placeholder {
        color: rgba(0, 0, 0, 0.38);
        font-style: italic;
      }
    }
  }
}

.recipient-details {
  margin-top: 16px;
  padding: 8px 12px;
  background: #fafafa;
  border-radius: 4px;

  p {
    margin: 4px 0;
    font-size: 13px;
  }
}

.loading-container,
.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 24px;
  color: rgba(0, 0, 0, 0.54);

  mat-icon {
    font-size: 48px;
    width: 48px;
    height: 48px;
    margin-bottom: 8px;
  }
}
```

---

## 4. Campaign Types from Wireframes

### 4.1 Predefined Campaign Templates

Each campaign type maps to a business rule and a default message template:

| Campaign Type                       | Business Rule           | Default Template                                                                                                                                                    |
| ----------------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Arrears Reminder SMS                | Client Arrears          | `Dear {{firstname}}, your loan repayment of {{TotalDue}} was due on {{PaymentDueDate}}. Please make payment to avoid penalties. Contact us at {{officenumber}}.`    |
| Loan Repayment Reminder SMS         | Loan Repayment Reminder | `Dear {{firstname}}, your upcoming loan repayment of {{TotalDue}} is due on {{PaymentDueDate}}. Please ensure sufficient funds. Ref: {{id}}.`                       |
| Payment Reminder (Individuals Only) | Payment Reminder        | `Dear {{FullName}}, this is a reminder that your payment of {{TotalDue}} is pending. Please make payment at your earliest convenience.`                             |
| Payment Due SMS                     | Payment Due             | `Dear {{firstname}}, a payment of {{TotalDue}} is now due. Outstanding balance: {{LoanOutstanding}}. Please pay promptly.`                                          |
| Clients in Arrears SMS              | Client Arrears          | `Dear {{FullName}}, your loan account is in arrears. Outstanding amount: {{LoanOutstanding}}. Please contact your branch at {{officenumber}} to make arrangements.` |

### 4.2 Campaign Template Selector

When creating a new campaign, provide a "Start from Template" option that pre-populates the business rule, message, and parameters based on the selected campaign type.

```typescript
interface CampaignTemplate {
  id: string;
  name: string;
  businessRuleId: number;
  defaultMessage: string;
  defaultParams: Record<string, any>;
  description: string;
}

// In the stepper component
selectTemplate(template: CampaignTemplate): void {
  this.campaignForm.patchValue({
    businessRuleId: template.businessRuleId,
    message: template.defaultMessage
  });
  this.ruleParams = template.defaultParams;
}
```

---

## 5. Service Enhancements

### 5.1 SMS Campaigns Service Updates

**File:** `src/app/organization/sms-campaigns/sms-campaigns.service.ts`

Add the following methods to the existing service:

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable({ providedIn: 'root' })
export class SmsCampaignsService {
  constructor(private http: HttpClient) {}

  /**
   * Get template data including business rules and available variables.
   * Calls: GET /smscampaigns/template
   */
  getCampaignTemplate(): Observable<CampaignTemplateData> {
    return this.http.get<CampaignTemplateData>('/smscampaigns/template');
  }

  /**
   * Get available template variables for a specific business rule.
   * Calls: GET /smscampaigns/template with businessRuleId
   */
  getTemplateVariables(businessRuleId: number): Observable<TemplateVariable[]> {
    const params = new HttpParams().set('businessRuleId', businessRuleId.toString());
    return this.http.get<any>('/smscampaigns/template', { params }).pipe(map((response) => response.templateVariables || []));
  }

  /**
   * Resolve a message template with sample data for preview.
   * Calls: POST /smscampaigns/preview
   */
  resolveTemplate(campaignId: number, messageTemplate: string): Observable<ResolvedMessage[]> {
    return this.http.post<ResolvedMessage[]>(`/smscampaigns/${campaignId}/preview`, {
      message: messageTemplate
    });
  }

  /**
   * Get preview recipients matching a business rule.
   * Returns the first N matching clients with their data for template resolution.
   */
  getPreviewRecipients(businessRuleId: number, ruleParams: Record<string, any>, limit: number): Observable<PreviewRecipient[]> {
    return this.http.post<PreviewRecipient[]>('/smscampaigns/preview/recipients', {
      businessRuleId,
      paramValue: ruleParams,
      limit
    });
  }

  /**
   * Get business rule parameters schema for dynamic form rendering.
   */
  getBusinessRuleParams(businessRuleId: number): Observable<BusinessRuleParam[]> {
    return this.http.get<any>(`/smscampaigns/template`).pipe(
      map((response) => {
        const rule = response.businessRules?.find((r: any) => r.id === businessRuleId);
        return rule?.params || [];
      })
    );
  }
}
```

### 5.2 Service Data Types

```typescript
interface CampaignTemplateData {
  businessRules: BusinessRule[];
  triggerTypes: { id: number; value: string }[];
  smsProviders: { id: number; providerName: string }[];
  campaignTypes: { id: number; value: string }[];
}

interface BusinessRule {
  id: number;
  name: string;
  description: string;
  params: BusinessRuleParam[];
}

interface ResolvedMessage {
  clientId: number;
  clientName: string;
  mobileNo: string;
  message: string;
}

interface PreviewRecipient {
  clientId: number;
  name: string;
  mobileNo: string;
  renderedMessage: string;
  [key: string]: any; // Additional data fields for template resolution
}
```

---

## 6. Files to Modify or Create

### 6.1 Modified Files

| File Path                                                                                     | Changes                                                                   |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/sms-campaign-stepper.component.ts`   | Add variable insertion toolbar, template selector, schedule configuration |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/sms-campaign-stepper.component.html` | Add variable panel, business rule params, schedule fields                 |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/sms-campaign-stepper.component.scss` | Styles for new UI elements                                                |
| `src/app/organization/sms-campaigns/sms-campaigns.service.ts`                                 | Add template resolution, preview recipients, business rule params methods |
| `src/app/organization/sms-campaigns/sms-campaigns.module.ts`                                  | Register new components                                                   |

### 6.2 New Files

| File Path                                                                                                          | Purpose                                             |
| ------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------- |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/variable-highlight/variable-highlight.component.ts`       | Overlay to highlight template variables in textarea |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/variable-highlight/variable-highlight.component.html`     | Template for highlight overlay                      |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/variable-highlight/variable-highlight.component.scss`     | Styles for variable chips                           |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/business-rule-params/business-rule-params.component.ts`   | Dynamic parameter form for business rules           |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/business-rule-params/business-rule-params.component.html` | Template for dynamic params form                    |
| `src/app/organization/sms-campaigns/sms-campaign-stepper/business-rule-params/business-rule-params.component.scss` | Styles for params form                              |
| `src/app/organization/sms-campaigns/view-campaign/campaign-preview/campaign-preview.component.ts`                  | Message preview with sample data                    |
| `src/app/organization/sms-campaigns/view-campaign/campaign-preview/campaign-preview.component.html`                | Preview template with phone frame mockup            |
| `src/app/organization/sms-campaigns/view-campaign/campaign-preview/campaign-preview.component.scss`                | Preview styles                                      |

---

## 7. Fineract API Endpoints

| Method   | Endpoint                                | Purpose                                                 |
| -------- | --------------------------------------- | ------------------------------------------------------- |
| `GET`    | `/smscampaigns/template`                | Get campaign template with business rules and variables |
| `GET`    | `/smscampaigns`                         | List all campaigns                                      |
| `POST`   | `/smscampaigns`                         | Create new campaign                                     |
| `GET`    | `/smscampaigns/{id}`                    | Get campaign details                                    |
| `PUT`    | `/smscampaigns/{id}`                    | Update campaign                                         |
| `DELETE` | `/smscampaigns/{id}`                    | Delete campaign                                         |
| `POST`   | `/smscampaigns/{id}?command=activate`   | Activate campaign                                       |
| `POST`   | `/smscampaigns/{id}?command=close`      | Close campaign                                          |
| `POST`   | `/smscampaigns/{id}?command=reactivate` | Reactivate campaign                                     |
| `GET`    | `/smscampaigns/{id}/messages`           | Get campaign messages/delivery status                   |

### 7.1 Create Campaign Payload

```json
{
  "campaignName": "Arrears Reminder - March 2026",
  "campaignType": 1,
  "triggerType": 1,
  "providerId": 1,
  "runReportId": 1,
  "paramValue": {
    "officeId": "1",
    "loanOfficerId": "-1"
  },
  "message": "Dear {{firstname}}, your loan repayment of {{TotalDue}} was due on {{PaymentDueDate}}.",
  "recurrenceStartDate": "13 March 2026",
  "recurrence": "FREQ=WEEKLY;INTERVAL=1;BYDAY=MO",
  "dateFormat": "dd MMMM yyyy",
  "locale": "en"
}
```

---

## 8. Testing Checklist

- [ ] Variable panel renders all variables grouped by category
- [ ] Clicking a variable inserts it at cursor position in textarea
- [ ] Variable highlighting displays correctly over textarea text
- [ ] Business rule selection loads dynamic parameters
- [ ] Schedule type switching shows/hides relevant date/time fields
- [ ] Message preview loads sample recipients from API
- [ ] Character count updates in real-time as message changes
- [ ] SMS segment count calculates correctly for GSM and Unicode messages
- [ ] Campaign template selector pre-populates form fields
- [ ] Preview renders correctly with resolved variable values
- [ ] Unresolved variables are flagged in the stats bar
- [ ] Phone frame preview displays message in SMS bubble format
- [ ] Recipient tabs allow switching between preview recipients
- [ ] All Fineract API calls use correct endpoints and payload formats
