# DESIGN-003: Layout Patterns

> M-SACCO Design System -- Page Layout Templates and Responsive Behavior
>
> **Patterns covered:** List Page, Detail Page, Wizard/Form Page, Dialog
>
> Each pattern includes ASCII wireframes, spacing specs, responsive behavior, and Figma auto-layout configuration.

---

## 1. List Page Pattern

The standard layout for browsable, filterable entity lists (Clients, Loans, Groups, Savings, Charges, etc.).

### 1.1 Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                  Top Navigation Bar (64px)                       │
│  [≡ Logo]  [Search ___________]  [🌐] [🌙] [🔔] [User ▾]     │
├─────────────────────────────────────────────────────────────────┤
│  Home > Module > Entity List                    (Breadcrumbs 32px)
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Entity Name                               [+ Create Entity]   │
│  (Page Title 24px/600)                     (Filled primary btn) │
│                                                    (48px row)   │
├─────────────────────────────────────────────────────────────────┤
│  [All (156)] [Pending (23)] [Active (120)] [Closed (13)]       │
│                                              (StatusTabs 40px)  │
├─────────────────────────────────────────────────────────────────┤
│  [🔍 Search ____________]  [Filter ▾] [Date Range]  [Export ▾] │
│                                              (Toolbar 48px)     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────┬───────────┬──────────┬─────────┬────────┬──────────┐  │
│  │  #  │  Name     │ Acct No  │ Branch  │ Status │ Actions  │  │
│  ├─────┼───────────┼──────────┼─────────┼────────┼──────────┤  │
│  │  1  │  John Doe │ 000123   │ Main    │ Active │ [View]   │  │
│  │  2  │  Jane S.  │ 000124   │ Branch2 │ Pending│ [View]   │  │
│  │  3  │  Bob W.   │ 000125   │ Main    │ Closed │ [View]   │  │
│  │ ... │  ...      │ ...      │ ...     │ ...    │ ...      │  │
│  └─────┴───────────┴──────────┴─────────┴────────┴──────────┘  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Showing 1 to 10 of 156 entries         [< 1 2 3 ... 16 >]    │
│                                              (Pagination 48px)  │
├─────────────────────────────────────────────────────────────────┤
│  Footer: M-SACCO v1.0 | Business Date: 2026-03-13   (48px)    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Grid Specifications

| Property           | Value                                        |
| ------------------ | -------------------------------------------- |
| Container          | Single column, centered                      |
| Max width          | 1400px                                       |
| Account card width | 90% (capped at 90rem)                        |
| Page padding       | 24px (desktop), 16px (tablet), 12px (mobile) |
| Content gap        | 0 (sections stacked flush)                   |

### 1.3 Section Heights

| Section            | Height | Padding          |
| ------------------ | ------ | ---------------- |
| Top Navigation Bar | 64px   | 0 16px           |
| Breadcrumbs        | 32px   | 4px 0            |
| Page Title Row     | 48px   | 8px 0            |
| Status Tabs        | 40px   | 0                |
| Search/Filter Bar  | 48px   | 8px 0            |
| Table Header Row   | 40px   | 6px 8px per cell |
| Table Data Row     | 40px   | 6px 8px per cell |
| Pagination         | 48px   | 8px 16px         |
| Footer             | 48px   | 12px 16px        |

### 1.4 Responsive Behavior

| Breakpoint | Behavior                                                    |
| ---------- | ----------------------------------------------------------- |
| xl (1920+) | Full table, all columns visible, max-width 1400px centered  |
| lg (1280+) | Full table, all columns visible                             |
| md (960+)  | Table with horizontal scroll if needed, some columns hidden |
| sm (600+)  | Table converts to card list (one card per row)              |
| xs (0-599) | Card list, single column, stacked actions                   |

**Mobile Card List Layout (xs/sm):**

```
┌────────────────────────────┐
│  John Doe          Active  │
│  Acct: 000123              │
│  Branch: Main              │
│  [View Details →]          │
└────────────────────────────┘
┌────────────────────────────┐
│  Jane Smith        Pending │
│  ...                       │
└────────────────────────────┘
```

### 1.5 Figma Auto-Layout

```
Frame: "List Page" (Fill container)
├── Direction: Vertical
├── Padding: 0
├── Gap: 0
├── Alignment: Top / Center
│
├── Navigation Bar (Fixed, Fill width, H: 64)
│   ├── Direction: Horizontal
│   ├── Padding: 0 16
│   ├── Gap: 8
│   └── Alignment: Center / Space Between
│
├── Content Area (Fill, max-width 1400)
│   ├── Breadcrumbs (Fill width, H: 32)
│   ├── Title Row (Fill width, H: 48)
│   │   ├── Direction: Horizontal
│   │   ├── Alignment: Center / Space Between
│   │   ├── Title (Hug)
│   │   └── Action Button (Hug)
│   ├── Status Tabs (Fill width, H: 40)
│   ├── Search Bar (Fill width, H: 48)
│   ├── Data Table (Fill width, Hug height)
│   └── Pagination (Fill width, H: 48)
│
└── Footer (Fixed bottom, Fill width, H: 48)
```

---

## 2. Detail Page Pattern

The standard layout for viewing a single entity (Client, Loan, Savings Account, Group, Center, etc.).

### 2.1 Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                  Top Navigation Bar (64px)                       │
├─────────────────────────────────────────────────────────────────┤
│  Home > Clients > John Doe                      (Breadcrumbs)   │
├─────────────────────────────────────────────────────────────────┤
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │                    Entity Header Card                       │ │
│ │  ┌─────────┐                                                │ │
│ │  │ Profile │  John Doe                         [Active]     │ │
│ │  │  Image  │  Account No: 000123                            │ │
│ │  │  80x80  │  Branch: Main Branch                           │ │
│ │  │         │  Loan Officer: Jane Smith                      │ │
│ │  └─────────┘  Activation Date: 2025-01-15                   │ │
│ │                                                             │ │
│ │  ┌──────────────────────────────────────────────┐           │ │
│ │  │ Savings: 3 | Loans: 2 | Shares: 1           │           │ │
│ │  └──────────────────────────────────────────────┘           │ │
│ │                                                             │ │
│ │  [Assign Staff] [Transfer] [Close] [More ▾]                │ │
│ └─────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────┤
│  [General] [Accounts] [CRB] [IDs] [Documents] [Family] [Notes]│
│                                      (Tab Navigation bar)       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tab Content Area                                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  Content cards, tables, forms specific to selected tab    │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Footer (48px)                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Entity Header Card

| Element                | Spec                                                         |
| ---------------------- | ------------------------------------------------------------ |
| Card background        | Primary `#1074b9` (from theme `header` class)                |
| Card padding           | 1% (per existing `_content.scss`)                            |
| Card text color        | `white`                                                      |
| Profile image          | 80x80px, `border-radius: 20px`, `object-fit: cover`          |
| Entity name            | 20px/600, white                                              |
| Key info labels        | 12px/400, `rgba(255,255,255,0.7)`                            |
| Key info values        | 14px/400, white                                              |
| Status badge           | StatusBadge component (see DESIGN-002 C5)                    |
| Account overview table | Max-width 240px, 14px font, no border, transparent even rows |
| Action buttons area    | Flex, align-self flex-end, margin 0 1%                       |
| Action button icons    | margin-bottom 2px, margin-right 4px                          |

### 2.3 Tab Navigation

| Property           | Value                                  |
| ------------------ | -------------------------------------- |
| Background         | `#f2f2f2` (light), `#303135` (dark)    |
| Height             | 48px                                   |
| Tab text           | 14px/500                               |
| Active tab         | Primary color underline 2px            |
| Dark mode tab link | `white`                                |
| Active tab (dark)  | border-bottom `lightgray`              |
| Overflow           | `overflow: auto` (horizontal scroll)   |
| Scroll behavior    | Smooth scroll, no scrollbar on desktop |

### 2.4 Tab Content Area

| Property     | Value                                    |
| ------------ | ---------------------------------------- |
| Padding      | 16px                                     |
| Background   | Page background (`#f8f9fa` light)        |
| Cards within | 8px radius, 16px padding, Level 1 shadow |
| Card gap     | 16px vertical between cards              |

### 2.5 Responsive Behavior

| Breakpoint | Behavior                                                         |
| ---------- | ---------------------------------------------------------------- |
| lg+        | Full header with image, overview table, and actions side by side |
| md         | Header stacks: image + info above, actions below                 |
| sm         | Profile image hidden, info stacks vertically                     |
| xs         | Compact header, tabs become scrollable horizontal list           |

### 2.6 Figma Auto-Layout

```
Frame: "Detail Page" (Fill container)
├── Direction: Vertical
├── Padding: 0
├── Gap: 0
│
├── Navigation Bar (Fixed, H: 64)
├── Breadcrumbs (Fill width, H: 32)
├── Entity Header Card (Fill width, Hug height)
│   ├── Direction: Horizontal (wraps on mobile)
│   ├── Padding: 16 (maps to 1% at 1400px)
│   ├── Gap: 16
│   ├── Profile Image (Fixed 80x80)
│   ├── Info Section (Fill, Vertical)
│   │   ├── Name + Status (Horizontal, Space Between)
│   │   ├── Key Info Grid (2 columns)
│   │   └── Account Summary Row (Horizontal)
│   └── Actions Section (Hug, Vertical, align-end)
│
├── Tab Bar (Fill width, H: 48)
│   ├── Direction: Horizontal
│   ├── Padding: 0
│   ├── Gap: 0
│   ├── Overflow: Scroll
│   └── Each Tab (Hug, padding 12 16)
│
├── Tab Content (Fill width, Fill height)
│   ├── Padding: 16
│   ├── Gap: 16
│   └── Content Cards (Fill width, Hug height each)
│
└── Footer (H: 48)
```

---

## 3. Wizard / Form Page Pattern

Multi-step form flow for creating entities (Create Client, Create Loan Application, Create Group, etc.).

### 3.1 Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                  Top Navigation Bar (64px)                       │
├─────────────────────────────────────────────────────────────────┤
│  Home > Clients > Create Client                 (Breadcrumbs)   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Create Client                                                  │
│  (Page Title 24px/600)                                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    (1)─────────(2)─────────(3)─────────(4)─────────(5)         │
│  General     Address     Family       IDs       Preview         │
│                                                                 │
│                    (WizardStepIndicator)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Step 1: General Information                              │  │
│  │                                                           │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐        │  │
│  │  │ First Name *        │  │ Last Name *          │        │  │
│  │  └─────────────────────┘  └─────────────────────┘        │  │
│  │                                                           │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐        │  │
│  │  │ Middle Name         │  │ Gender *      [▾]   │        │  │
│  │  └─────────────────────┘  └─────────────────────┘        │  │
│  │                                                           │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐        │  │
│  │  │ Date of Birth [📅]  │  │ Mobile Number       │        │  │
│  │  └─────────────────────┘  └─────────────────────┘        │  │
│  │                                                           │  │
│  │  ┌───────────────────────────────────────────────┐        │  │
│  │  │ Email Address                                 │        │  │
│  │  └───────────────────────────────────────────────┘        │  │
│  │                                                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  [< Previous]                                     [Next >]      │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Footer (48px)                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Step Indicator Specs

See WizardStepIndicator in DESIGN-002 (C4) for full component specs.

| Property         | Value                           |
| ---------------- | ------------------------------- |
| Container height | 80px (including labels)         |
| Margin top       | 16px                            |
| Margin bottom    | 24px                            |
| Alignment        | Center                          |
| Max width        | 600px (centered within content) |

### 3.3 Form Layout

| Property          | Desktop (md+)  | Mobile (xs/sm) |
| ----------------- | -------------- | -------------- |
| Columns           | 2 columns      | 1 column       |
| Column gap        | 16px           | 0              |
| Row gap           | 16px           | 16px           |
| Full-width fields | Span 2 columns | Span 1 column  |
| Form card padding | 24px           | 16px           |
| Form card radius  | 8px            | 8px            |
| Form card shadow  | Level 1        | Level 0        |
| Field min width   | 200px          | 100%           |

**Field Layout Rules:**

- Short fields (name, phone, date): Half width (1 column)
- Long fields (email, address, description): Full width (2 columns)
- Related fields (first/last name, city/postal code): Same row
- Checkbox groups: Full width, vertical stack
- File uploads: Full width

### 3.4 Navigation Buttons (StepperButtons)

| Property        | Value                                          |
| --------------- | ---------------------------------------------- |
| Container       | Flex, space-between, full width                |
| Margin top      | 24px                                           |
| Padding         | 0 24px (inside form card)                      |
| Previous button | Text variant, `fa-chevron-left` icon prefix    |
| Next button     | Filled primary, `fa-chevron-right` icon suffix |
| Submit (last)   | Filled primary, `fa-check` icon prefix         |
| First step      | Previous button hidden                         |

### 3.5 Preview Step (Final Step)

The last step renders a read-only summary of all entered data:

```
┌───────────────────────────────────────────────────────────┐
│  Review & Submit                                          │
│                                                           │
│  General Information                                      │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ First Name:    John                                 │  │
│  │ Last Name:     Doe                                  │  │
│  │ Gender:        Male                                 │  │
│  │ DOB:           1990-05-15                           │  │
│  │ Mobile:        +254712345678                        │  │
│  │ Email:         john@example.com                     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  Address                                                  │
│  ┌─────────────────────────────────────────────────────┐  │
│  │ ...                                                 │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
│  [< Previous]                              [Submit]       │
└───────────────────────────────────────────────────────────┘
```

**Preview Layout:**

- Section header: `h3` (16px/500)
- Key-value pairs: Label 12px/400 `#6c757d`, Value 14px/400 `#212529`
- Section card: `#f8f9fa` bg, 8px radius, 16px padding
- Gap between sections: 16px

### 3.6 Responsive Behavior

| Breakpoint | Behavior                                            |
| ---------- | --------------------------------------------------- |
| lg+        | 2-column form, horizontal step indicator            |
| md         | 2-column form, horizontal step indicator            |
| sm         | 1-column form, text step indicator ("Step 2 of 5")  |
| xs         | 1-column form, text step indicator, compact padding |

### 3.7 Figma Auto-Layout

```
Frame: "Wizard Page" (Fill container)
├── Direction: Vertical
├── Gap: 0
│
├── Navigation Bar (H: 64)
├── Breadcrumbs (H: 32)
├── Page Title (Fill width, H: 48, padding 8 24)
├── Step Indicator (Fill width, H: 80, max-w 600, centered)
│   ├── Direction: Horizontal
│   ├── Alignment: Center / Center
│   └── Equal width step slots
│
├── Form Card (Fill width, Hug height, margin 0 24)
│   ├── Padding: 24
│   ├── Gap: 16
│   ├── Form Grid (CSS Grid or Horizontal wrap)
│   │   ├── Columns: 2 (desktop), 1 (mobile)
│   │   ├── Column gap: 16
│   │   └── Row gap: 16
│   └── Each Field (Hug height)
│
├── Stepper Buttons (Fill width, H: 56, padding 0 24)
│   ├── Direction: Horizontal
│   ├── Alignment: Center / Space Between
│   ├── Previous Button (Hug)
│   └── Next/Submit Button (Hug)
│
└── Footer (H: 48)
```

---

## 4. Dialog Pattern

Modal overlay pattern used by all dialog components (FormDialog, DeleteDialog, ConfirmationDialog, etc.).

### 4.1 Structure

```
┌─────────────────────────────── Viewport ───────────────────────────────┐
│                                                                        │
│                     ░░░░░░░░░░░░░░░░░░░░░░░░░░░                       │
│                     ░                          ░                       │
│                     ░  ┌──────────────────────┐░                       │
│                     ░  │  Dialog Title    [X] │░                       │
│                     ░  ├──────────────────────┤░                       │
│                     ░  │                      │░                       │
│                     ░  │  Dialog Content      │░                       │
│                     ░  │                      │░                       │
│                     ░  │  (Forms, messages,   │░                       │
│                     ░  │   confirmations)     │░                       │
│                     ░  │                      │░                       │
│                     ░  ├──────────────────────┤░                       │
│                     ░  │     [Cancel] [OK]    │░                       │
│                     ░  └──────────────────────┘░                       │
│                     ░                          ░                       │
│                     ░░░░░░░░░░░░░░░░░░░░░░░░░░░                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Overlay

| Property   | Value                                 |
| ---------- | ------------------------------------- |
| Background | `rgba(0,0,0,0.32)`                    |
| Position   | Fixed, full viewport                  |
| z-index    | 1000                                  |
| Click      | Closes dialog (unless `disableClose`) |
| Animation  | Fade in 150ms ease-in                 |

### 4.3 Dialog Card

| Property      | Value                                      |
| ------------- | ------------------------------------------ |
| Background    | `#ffffff` (light), `#303135` (dark)        |
| Border radius | 12px                                       |
| Shadow        | Level 3                                    |
| Position      | Centered vertically and horizontally       |
| Animation     | Scale from 0.95 + fade, 200ms ease-out     |
| Max height    | 90vh (scrollable content)                  |
| Overflow      | Content area scrolls, header/actions fixed |

### 4.4 Dialog Sizes

| Size       | Width | Usage                                      |
| ---------- | ----- | ------------------------------------------ |
| Small      | 400px | Confirmations, delete, enable/disable      |
| Medium     | 600px | Forms, data entry                          |
| Large      | 800px | Complex forms, multi-section content       |
| Full-width | 90vw  | Data tables, large data views              |
| Mobile     | 100vw | All dialogs on xs breakpoint (full screen) |

### 4.5 Dialog Sections

**Title Bar:**

| Property      | Value                                              |
| ------------- | -------------------------------------------------- |
| Height        | 56px                                               |
| Padding       | 16px 24px                                          |
| Font          | 20px/500                                           |
| Color         | `#212529` (light), `rgba(255,255,255,0.87)` (dark) |
| Close button  | 24px icon, `fa-times`, right-aligned               |
| Border bottom | 1px solid `#e9ecef`                                |

**Content Area:**

| Property   | Value                                          |
| ---------- | ---------------------------------------------- |
| Padding    | 24px                                           |
| Font       | 14px/400 body-1                                |
| Max height | calc(90vh - 56px - 56px) (minus title+actions) |
| Overflow   | auto (vertical scroll)                         |
| Gap        | 16px between form fields or sections           |

**Actions Bar:**

| Property      | Value                                    |
| ------------- | ---------------------------------------- |
| Height        | 56px                                     |
| Padding       | 8px 16px                                 |
| Alignment     | Right-aligned (flex-end)                 |
| Gap           | 8px between buttons                      |
| Border top    | 1px solid `#e9ecef`                      |
| Cancel button | Text or Outlined variant                 |
| Submit button | Filled primary variant                   |
| Danger submit | Filled warn variant (for delete dialogs) |

### 4.6 Dialog Variants

| Variant            | Title Icon                      | Submit Style   |
| ------------------ | ------------------------------- | -------------- |
| FormDialog         | none                            | Filled primary |
| DeleteDialog       | `fa-exclamation-triangle` warn  | Filled warn    |
| ConfirmationDialog | none                            | Filled primary |
| CancelDialog       | `fa-exclamation-circle` warning | Outlined warn  |
| ErrorDialog        | `fa-times-circle` error         | Text "Dismiss" |
| EnableDialog       | `fa-check-circle` success       | Filled primary |
| DisableDialog      | `fa-ban` warning                | Outlined warn  |

### 4.7 Responsive Behavior

| Breakpoint | Behavior                                             |
| ---------- | ---------------------------------------------------- |
| lg+        | Centered dialog at specified width                   |
| md         | Centered dialog at specified width                   |
| sm         | Dialog width: 90vw                                   |
| xs         | Full-screen dialog (100vw x 100vh, no border radius) |

### 4.8 Figma Auto-Layout

```
Frame: "Dialog Overlay" (Fixed, Fill viewport)
├── Background: rgba(0,0,0,0.32)
├── Alignment: Center / Center
│
└── Dialog Card (Hug height, Fixed width per size)
    ├── Direction: Vertical
    ├── Padding: 0
    ├── Gap: 0
    ├── Border radius: 12
    ├── Shadow: Level 3
    │
    ├── Title Bar (Fill width, H: 56)
    │   ├── Direction: Horizontal
    │   ├── Padding: 16 24
    │   ├── Alignment: Center / Space Between
    │   ├── Title Text (Hug)
    │   └── Close Icon Button (24x24)
    │
    ├── Content Area (Fill width, Hug height, max-h constrained)
    │   ├── Padding: 24
    │   ├── Gap: 16
    │   └── [Form fields / Message / Custom content]
    │
    └── Actions Bar (Fill width, H: 56)
        ├── Direction: Horizontal
        ├── Padding: 8 16
        ├── Gap: 8
        ├── Alignment: Center / End (right-aligned)
        ├── Cancel Button (Hug)
        └── Submit Button (Hug)
```

---

## Appendix: Layout Token Summary

```
/* Page Layout */
--layout-max-width:        1400px;
--layout-nav-height:       64px;
--layout-breadcrumb-height: 32px;
--layout-footer-height:    48px;

/* Content */
--content-padding-desktop: 24px;
--content-padding-tablet:  16px;
--content-padding-mobile:  12px;

/* Cards */
--card-padding:            16px;
--card-padding-large:      24px;
--card-radius:             8px;
--card-shadow:             0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24);

/* Dialogs */
--dialog-overlay:          rgba(0,0,0,0.32);
--dialog-radius:           12px;
--dialog-title-height:     56px;
--dialog-actions-height:   56px;
--dialog-padding:          24px;

/* Tables */
--table-header-height:     40px;
--table-row-height:        40px;
--table-cell-padding:      6px 8px;
--table-header-bg:         #f5f5f5;
--table-even-row-bg:       #f8f9fa;

/* Forms */
--form-field-gap:          16px;
--form-columns-desktop:    2;
--form-columns-mobile:     1;
```
