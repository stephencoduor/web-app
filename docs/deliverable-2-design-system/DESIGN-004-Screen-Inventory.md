# DESIGN-004: Screen Inventory

> M-SACCO Design System -- Complete Screen Catalog (48 Screens)
>
> Organized by module. Each screen documents: layout pattern used, fields, table columns, action buttons, status indicators, navigation elements, and dialog triggers.

---

## Module 1: Clients (14 screens)

---

### 1.1 Client List

**Layout Pattern:** List Page

**Status Tabs:**

- All (count) | Pending Approval (count) | Closed (count) | Rejected (count)

**Table Columns:**

| Column     | Width | Sortable | Type        |
| ---------- | ----- | -------- | ----------- |
| Client ID  | 80px  | Yes      | Number link |
| Name       | 200px | Yes      | Text link   |
| Account No | 120px | Yes      | Monospace   |
| Branch     | 150px | Yes      | Text        |
| Status     | 100px | Yes      | StatusBadge |

**Actions:**

- Page action: "Create Client" (Filled primary, `fa-plus` icon)
- Row action: Click row to navigate to Client Detail
- Export: Copy, Excel, PDF

**Dialog Triggers:** None (navigates to wizard)

---

### 1.2 Create Client Step 1: General

**Layout Pattern:** Wizard Page (Step 1 of 5)

**Fields:**

| Field         | Type        | Required | Width | Validation              |
| ------------- | ----------- | -------- | ----- | ----------------------- |
| First Name    | Text input  | Yes      | Half  | Min 2 chars             |
| Last Name     | Text input  | Yes      | Half  | Min 2 chars             |
| Middle Name   | Text input  | No       | Half  | --                      |
| Gender        | Select      | Yes      | Half  | Male/Female             |
| Date of Birth | Date picker | No       | Half  | Cannot be future date   |
| Mobile Number | Text input  | No       | Half  | Numeric, country format |
| Email         | Text input  | No       | Full  | Email format            |

**Navigation:** [Next >]

---

### 1.3 Create Client Step 2: Address

**Layout Pattern:** Wizard Page (Step 2 of 5)

**Fields:**

| Field       | Type       | Required | Width |
| ----------- | ---------- | -------- | ----- |
| Address     | Textarea   | No       | Full  |
| City        | Text input | No       | Half  |
| County      | Text input | No       | Half  |
| Postal Code | Text input | No       | Half  |

**Navigation:** [< Previous] [Next >]

---

### 1.4 Create Client Step 3: Family

**Layout Pattern:** Wizard Page (Step 3 of 5)

**Repeatable Section** -- Add family members:

| Field         | Type        | Required | Width |
| ------------- | ----------- | -------- | ----- |
| Name          | Text input  | Yes      | Half  |
| Relationship  | Select      | Yes      | Half  |
| Date of Birth | Date picker | No       | Half  |
| Gender        | Select      | No       | Half  |
| Occupation    | Text input  | No       | Full  |

**Actions:**

- "Add Family Member" (Outlined button, `fa-plus`)
- Remove member (icon button, `fa-trash`)

**Navigation:** [< Previous] [Next >]

---

### 1.5 Create Client Step 4: Identification

**Layout Pattern:** Wizard Page (Step 4 of 5)

**Repeatable Section** -- Add identification documents:

| Field       | Type       | Required | Width |
| ----------- | ---------- | -------- | ----- |
| ID Type     | Select     | Yes      | Half  |
| ID Number   | Text input | Yes      | Half  |
| Description | Text input | No       | Full  |
| Attachment  | FileUpload | No       | Full  |

**Actions:**

- "Add Identification" (Outlined button, `fa-plus`)
- Remove ID (icon button, `fa-trash`)

**Navigation:** [< Previous] [Next >]

---

### 1.6 Create Client Step 5: Preview

**Layout Pattern:** Wizard Page (Step 5 of 5) -- Preview variant

**Sections displayed:**

1. General Information (read-only key-value list)
2. Address (read-only key-value list)
3. Family Members (read-only table)
4. Identification Documents (read-only table with attachment indicators)

**Navigation:** [< Previous] [Submit]

**Dialog Triggers:** ConfirmationDialog on Submit

---

### 1.7 Client Detail: General

**Layout Pattern:** Detail Page (General tab)

**Entity Header:**

- Profile image (80x80, border-radius 20px)
- Client name (20px/600, white on primary bg)
- Status badge
- Key info: Account No, Branch, Loan Officer, Activation Date
- Account summary: Savings count, Loans count, Shares count

**Tab Content -- General:**

- Personal information card (key-value pairs)
- Address card
- Activation details card

**Action Buttons:**

- Assign Staff, Transfer Client, Close Client, More (dropdown: Edit, Activate, Reject, Withdraw, Reactivate, Undo Transfer)

**Dialog Triggers:**

- Assign Staff: FormDialog
- Transfer: FormDialog (branch/officer selection)
- Close: ConfirmationDialog
- Activate: EnableDialog
- Reject: ConfirmationDialog with reason
- Delete: DeleteDialog

---

### 1.8 Client Detail: CRB Account

**Layout Pattern:** Detail Page (CRB tab)

**Content:**

- CRBScoreCard component (gauge visualization)
- Credit history table

**Credit History Table Columns:**

| Column | Width | Type        |
| ------ | ----- | ----------- |
| Date   | 120px | Date        |
| Bureau | 120px | Text        |
| Score  | 80px  | Number      |
| Status | 100px | StatusBadge |
| Report | 80px  | Link/Icon   |

---

### 1.9 Client Detail: Identification

**Layout Pattern:** Detail Page (Identification tab)

**Table Columns:**

| Column      | Width | Type      |
| ----------- | ----- | --------- |
| ID Type     | 150px | Text      |
| ID Number   | 150px | Monospace |
| Description | 200px | Text      |
| Status      | 100px | Badge     |
| Actions     | 100px | Icons     |

**Actions:**

- "Add Identification" button
- Row actions: View attachment, Edit, Delete

**Dialog Triggers:** FormDialog (add/edit), DeleteDialog (remove)

---

### 1.10 Client Detail: Business Details

**Layout Pattern:** Detail Page (Business tab)

**Content Cards:**

- Business Information: Name, Type, Start Date
- Business Address: Address, City, County

**Actions:** Edit (FormDialog), Add Business (FormDialog)

---

### 1.11 Client Detail: Documents

**Layout Pattern:** Detail Page (Documents tab)

**Content:** EntityDocumentsTab component (grid of document cards)

**Actions:**

- "Upload Document" (FileUpload dialog)
- Per document: Download, Delete

**Dialog Triggers:** FormDialog (upload), DeleteDialog (remove)

---

### 1.12 Client Detail: Family Details

**Layout Pattern:** Detail Page (Family tab)

**Table Columns:**

| Column        | Width | Type  |
| ------------- | ----- | ----- |
| Name          | 200px | Text  |
| Relationship  | 120px | Text  |
| Gender        | 80px  | Text  |
| Date of Birth | 120px | Date  |
| Occupation    | 150px | Text  |
| Actions       | 80px  | Icons |

**Actions:** Add Member, Edit, Delete

**Dialog Triggers:** FormDialog (add/edit), DeleteDialog (remove)

---

### 1.13 Client Detail: Financial Statements

**Layout Pattern:** Detail Page (Financials tab)

**Content:**

- Income table (source, amount, frequency)
- Expense table (category, amount, frequency)
- Net income summary card (highlighted)

**Table Columns (Income/Expense):**

| Column    | Width | Type     |
| --------- | ----- | -------- |
| Source    | 200px | Text     |
| Amount    | 120px | Currency |
| Frequency | 120px | Text     |
| Actions   | 80px  | Icons    |

**Summary Card:**

- Total Income: Currency, green text
- Total Expenses: Currency, red text
- Net Income: Currency, bold, primary or warn based on sign

---

### 1.14 Client Detail: Notes

**Layout Pattern:** Detail Page (Notes tab)

**Content:** EntityNotesTab component (list of note cards)

**Actions:**

- "Add Note" (Filled primary button)
- Per note: Edit, Delete

**Dialog Triggers:** FormDialog (add/edit), DeleteDialog (remove)

---

## Module 2: Groups (6 screens)

---

### 2.1 Group List

**Layout Pattern:** List Page

**Status Tabs:** All | Pending | Active | Closed

**Table Columns:**

| Column   | Width | Type        |
| -------- | ----- | ----------- |
| Group ID | 80px  | Number link |
| Name     | 200px | Text link   |
| Branch   | 150px | Text        |
| Staff    | 150px | Text        |
| Status   | 100px | StatusBadge |

**Actions:** Create Group button

---

### 2.2 Create Group

**Layout Pattern:** Wizard Page (3 steps)

**Step 1 -- General:**

- Group Name*, Office/Branch*, Submitted Date, Staff

**Step 2 -- Members:**

- GroupMemberSelector component
- Search and add existing clients

**Step 3 -- Preview & Submit**

---

### 2.3 Group Detail: General

**Layout Pattern:** Detail Page

**Header:** Group name, Branch, Staff, Status, Member count

**Tabs:** General | Members | Notes | Documents | Savings | Loans

**General Tab:** Group info card, meeting schedule

**Actions:** Assign Staff, Transfer, Close, Activate

---

### 2.4 Group Detail: Members

**Layout Pattern:** Detail Page (Members tab)

**Table Columns:**

| Column     | Width | Type        |
| ---------- | ----- | ----------- |
| Client ID  | 80px  | Number link |
| Name       | 200px | Text link   |
| Account No | 120px | Monospace   |
| Status     | 100px | StatusBadge |
| Actions    | 80px  | Icons       |

**Actions:** Add Member, Remove Member

---

### 2.5 Group Detail: Notes

**Layout Pattern:** Detail Page (Notes tab) -- uses EntityNotesTab

---

### 2.6 Group Detail: Documents

**Layout Pattern:** Detail Page (Documents tab) -- uses EntityDocumentsTab

---

## Module 3: Loans (10 screens)

---

### 3.1 Loan List

**Layout Pattern:** List Page

**Status Tabs:** All | Pending Approval | Approved | Active | Overpaid | Closed | Rejected

**Table Columns:**

| Column       | Width | Type        |
| ------------ | ----- | ----------- |
| Loan ID      | 80px  | Number link |
| Client Name  | 200px | Text link   |
| Loan Product | 150px | Text        |
| Principal    | 120px | Currency    |
| Disbursement | 120px | Date        |
| Status       | 100px | StatusBadge |

---

### 3.2 Create Loan Application Step 1: Details

**Layout Pattern:** Wizard Page (Step 1 of 5)

**Fields:**

- Loan Product (Select), Loan Purpose (Select)
- Principal Amount (InputAmount), Loan Term (Number + Select period)
- Interest Rate (Number with % suffix)
- Repayment Frequency (Select), Number of Repayments (Number)
- Expected Disbursement Date (Date picker)
- Submitted On (Date picker)

---

### 3.3 Create Loan Application Step 2: Terms

**Layout Pattern:** Wizard Page (Step 2 of 5)

**Fields:**

- Interest Method (Select: Flat/Declining), Amortization Type (Select)
- Interest Calculation Period (Select)
- Grace periods: Principal, Interest, Interest Payment (Number fields)
- Arrears tolerance (InputAmount)

---

### 3.4 Create Loan Application Step 3: Charges

**Layout Pattern:** Wizard Page (Step 3 of 5)

**Content:**

- Charge selection dropdown
- Added charges table (Name, Type, Amount, Due Date)
- Add/Remove charge actions

---

### 3.5 Create Loan Application Step 4: Guarantors

**Layout Pattern:** Wizard Page (Step 4 of 5)

**Content:** GuarantorSelector component (3 modes)

---

### 3.6 Create Loan Application Step 5: Preview

**Layout Pattern:** Wizard Page (Preview step)

**Sections:** Loan Details, Terms, Charges, Guarantors (all read-only)

---

### 3.7 Loan Detail: General

**Layout Pattern:** Detail Page

**Header:**

- Client name, Loan product, Principal amount
- Status badge, Loan officer
- Key metrics: Disbursed amount, Outstanding balance, Arrears

**Tabs:** General | Repayment Schedule | Transactions | Charges | Guarantors | Collateral | Documents | Notes

**General Tab:** Loan summary card, terms card, timeline card

**ApprovalActionBar:** Visible when status is "Pending Approval"

---

### 3.8 Loan Detail: Repayment Schedule

**Layout Pattern:** Detail Page (Repayment tab)

**Table Columns:**

| Column      | Width | Type     |
| ----------- | ----- | -------- |
| #           | 40px  | Number   |
| Date        | 120px | Date     |
| Principal   | 100px | Currency |
| Interest    | 100px | Currency |
| Fees        | 100px | Currency |
| Penalties   | 100px | Currency |
| Total Due   | 100px | Currency |
| Total Paid  | 100px | Currency |
| Outstanding | 100px | Currency |

Overdue rows highlighted with `#ffebee` background.

---

### 3.9 Loan Detail: Transactions

**Layout Pattern:** Detail Page (Transactions tab)

**Table Columns:**

| Column      | Width | Type     |
| ----------- | ----- | -------- |
| ID          | 60px  | Number   |
| Date        | 120px | Date     |
| Type        | 120px | Text     |
| Amount      | 100px | Currency |
| Principal   | 100px | Currency |
| Interest    | 100px | Currency |
| Outstanding | 100px | Currency |

---

### 3.10 Loan Detail: Charges

**Layout Pattern:** Detail Page (Charges tab)

**Table Columns:**

| Column      | Width | Type        |
| ----------- | ----- | ----------- |
| Name        | 200px | Text        |
| Type        | 120px | Text        |
| Due Date    | 120px | Date        |
| Amount      | 100px | Currency    |
| Amount Paid | 100px | Currency    |
| Outstanding | 100px | Currency    |
| Status      | 80px  | StatusBadge |

---

## Module 4: Accounting (6 screens)

---

### 4.1 Chart of Accounts

**Layout Pattern:** List Page (with tree hierarchy)

**Table Columns:**

| Column       | Width | Type      |
| ------------ | ----- | --------- |
| Account Code | 120px | Monospace |
| Account Name | 250px | Text link |
| Type         | 120px | Text      |
| Balance      | 120px | Currency  |
| Status       | 80px  | Badge     |

**Hierarchy:** Indent with `fa-chevron-right` expand/collapse icons.

**Actions:** Create GL Account, Import

---

### 4.2 Journal Entries

**Layout Pattern:** List Page

**Table Columns:**

| Column         | Width | Type     |
| -------------- | ----- | -------- |
| Entry ID       | 80px  | Number   |
| Date           | 120px | Date     |
| Office         | 150px | Text     |
| Debit Account  | 200px | Text     |
| Credit Account | 200px | Text     |
| Amount         | 120px | Currency |

**Actions:** Create Journal Entry, Search by date range

---

### 4.3 Create Journal Entry

**Layout Pattern:** Form Page (single step)

**Fields:**

- Office (Select), Currency (Select), Reference Number (Text)
- Transaction Date (Date picker), Payment Type (Select)
- Debit entries: Account (Select), Amount (InputAmount)
- Credit entries: Account (Select), Amount (InputAmount)
- Comments (Textarea)

**Validation:** Total debits must equal total credits

---

### 4.4 Financial Activity Mappings

**Layout Pattern:** List Page

**Table:** Financial Activity, GL Account, Actions (Edit)

---

### 4.5 Accounting Closures

**Layout Pattern:** List Page

**Table:** Office, Closing Date, Comments, Actions

---

### 4.6 Trial Balance

**Layout Pattern:** List Page (report view)

**Filters:** Office, Date Range, Currency

**Table Columns:**

| Column         | Width | Type      |
| -------------- | ----- | --------- |
| Account Code   | 100px | Monospace |
| Account Name   | 250px | Text      |
| Debit Balance  | 120px | Currency  |
| Credit Balance | 120px | Currency  |

**Footer:** Total row with summed debits and credits

---

## Module 5: Products (4 screens)

---

### 5.1 Loan Products List

**Layout Pattern:** List Page

**Table Columns:**

| Column      | Width | Type  |
| ----------- | ----- | ----- |
| Name        | 250px | Link  |
| Short Name  | 100px | Text  |
| Fund Source | 150px | Text  |
| Start Date  | 120px | Date  |
| Close Date  | 120px | Date  |
| Status      | 80px  | Badge |

---

### 5.2 Create/Edit Loan Product

**Layout Pattern:** Wizard Page (6 steps)

**Steps:** Details, Currency, Terms, Settings, Charges, Accounting

---

### 5.3 Savings Products List

**Layout Pattern:** List Page

**Table Columns:** Name, Short Name, Currency, Nominal Interest, Status

---

### 5.4 Create/Edit Savings Product

**Layout Pattern:** Wizard Page (5 steps)

**Steps:** Details, Currency, Terms, Charges, Accounting

---

## Module 6: Organization (4 screens)

---

### 6.1 Office List (Branch Management)

**Layout Pattern:** List Page (tree hierarchy)

**Table:** Office Name (hierarchical), External ID, Parent Office, Open Date, Actions

---

### 6.2 Staff List

**Layout Pattern:** List Page

**Status Tabs:** All | Active | Inactive

**Table:** Staff ID, Name, Office, Status, Actions

---

### 6.3 Currency Configuration

**Layout Pattern:** List Page

**Table:** Currency Code, Name, Decimal Places, Display Symbol, Actions

---

### 6.4 Payment Type Configuration

**Layout Pattern:** List Page

**Table:** Name, Description, Is Cash Payment, Position, Actions

---

## Module 7: New Features (4 screens)

---

### 7.1 SMS Campaigns

**Layout Pattern:** List Page + FormDialog for create

**Table:** Campaign Name, Type, Trigger, Status, Sent/Failed counts, Actions

**Create Dialog:** Name, Message (with VariableInsertButton), Provider, Schedule

---

### 7.2 Data Export Configuration

**Layout Pattern:** Form Page

**Content:**

- Entity type select
- DragAndDropFieldList for column selection and ordering
- Export format radio (CSV, Excel)
- Date range filters

---

### 7.3 Client Transfer

**Layout Pattern:** Form Page

**Content:**

- Client search (autocomplete)
- From Branch (read-only)
- To Branch (Select)
- Transfer Date (Date picker)
- Note (Textarea)
- ClientTransferCard (preview)

**Dialog Triggers:** ConfirmationDialog on submit

---

### 7.4 MPESA Integration Settings

**Layout Pattern:** Form Page

**Content:** MPESAConfigPanel component

**Actions:** Test Connection, Save Configuration

---

## Screen Count Summary

| Module       | Screens | Pattern Distribution       |
| ------------ | ------- | -------------------------- |
| Clients      | 14      | 1 List, 5 Wizard, 8 Detail |
| Groups       | 6       | 1 List, 1 Wizard, 4 Detail |
| Loans        | 10      | 1 List, 5 Wizard, 4 Detail |
| Accounting   | 6       | 4 List, 1 Form, 1 Report   |
| Products     | 4       | 2 List, 2 Wizard           |
| Organization | 4       | 4 List                     |
| New Features | 4       | 1 List, 3 Form             |
| **Total**    | **48**  |                            |

---

## Appendix: Common UI Elements Across All Screens

### Navigation Bar (present on all screens)

| Element        | Position | Icon/Component    |
| -------------- | -------- | ----------------- |
| Hamburger menu | Left     | `fa-bars`         |
| Logo           | Left     | Image             |
| Search         | Center   | SearchTool        |
| Language       | Right    | LanguageSelector  |
| Theme toggle   | Right    | ThemeToggle       |
| Notifications  | Right    | NotificationsTray |
| User menu      | Right    | Avatar + dropdown |

### Breadcrumbs (present on all screens except dashboard)

- Separator: `fa-chevron-right` 12px
- Link style: `#1074b9` (light), `#0098ff` (dark)
- Current page: `#6c757d`, no link

### Footer (present on all screens)

- Height: 48px
- Content: App version, Business Date, powered-by text
- Business Date: `#1074b9` (light), `#0098ff` (dark)
- Dark mode text: `white`
