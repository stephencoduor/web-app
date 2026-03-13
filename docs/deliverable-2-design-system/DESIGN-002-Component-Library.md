# DESIGN-002: Component Library

> M-SACCO Design System -- Complete Component Specifications
>
> **Total components:** 42 (10 Base Material + 20 Existing Custom + 12 New M-SACCO)
>
> Each component documents: name, description, props/inputs, states, variants, usage guidelines, and Figma auto-layout specs.

---

## Part A: Base Material Components (10)

---

### A1. Button

**Description:** Primary interaction element. Wraps Angular Material `mat-button` family with M-SACCO theming.

**Variants:**

| Variant  | Directive            | Background  | Text Color | Border        | Shadow  |
| -------- | -------------------- | ----------- | ---------- | ------------- | ------- |
| Filled   | `mat-flat-button`    | `#1074b9`   | `white`    | none          | Level 0 |
| Outlined | `mat-stroked-button` | transparent | `#1074b9`  | 1px `#1074b9` | Level 0 |
| Elevated | `mat-raised-button`  | `#ffffff`   | `#1074b9`  | none          | Level 1 |
| Text     | `mat-button`         | transparent | `#1074b9`  | none          | Level 0 |
| Tonal    | `mat-flat-button`    | `#e3f0fa`   | `#004989`  | none          | Level 0 |

**Sizes:**

| Size | Height | Padding   | Font Size | Icon Size |
| ---- | ------ | --------- | --------- | --------- |
| sm   | 28px   | 4px 12px  | 12px      | 14px      |
| md   | 36px   | 8px 16px  | 14px      | 16px      |
| lg   | 44px   | 12px 24px | 14px      | 18px      |

**States:**

| State    | Filled                        | Outlined                         |
| -------- | ----------------------------- | -------------------------------- |
| Default  | bg `#1074b9`                  | border `#1074b9`                 |
| Hover    | bg `#0e66a5`                  | bg `rgba(16,116,185,0.04)`       |
| Active   | bg `#004989`                  | bg `rgba(16,116,185,0.12)`       |
| Focused  | bg `#0e66a5` + focus ring 2px | border 2px + focus ring          |
| Disabled | bg `#adb5bd`, text `#6c757d`  | border `#adb5bd`, text `#adb5bd` |

**Props/Inputs:**

| Input      | Type    | Default   | Description                 |
| ---------- | ------- | --------- | --------------------------- |
| `color`    | string  | `primary` | `primary`, `accent`, `warn` |
| `disabled` | boolean | `false`   | Disables interaction        |
| `type`     | string  | `button`  | `button`, `submit`, `reset` |

**Icon Spacing:** Icon placed before label with 4px gap (`margin-right: 4px; margin-bottom: 2px` per `.account-actions i` rule).

**Figma Auto-Layout:**

- Direction: Horizontal
- Padding: per size table
- Gap: 4px (icon to label)
- Alignment: Center / Center
- Border radius: 4px (filled), 20px (M3 variant)

**Usage Guidelines:**

- Use Filled for primary page action (one per page max).
- Use Outlined for secondary actions.
- Use Text for tertiary / inline actions.
- Destructive actions: `color="warn"`, Filled variant.
- Button label: 14px/500 uppercase.

---

### A2. Text Input

**Description:** Standard form input wrapping `mat-form-field` with `appearance="outline"`.

**Anatomy:**

```
┌─ Label (overline 10px/500, floats on focus) ──────────┐
│ [Prefix Icon]  Input Text (14px/400)  [Suffix Icon]  │
└───────────────────────────────────────────────────────┘
  Helper text or Error message (12px/400)
```

**States:**

| State    | Border Color | Label Color | Background |
| -------- | ------------ | ----------- | ---------- |
| Default  | `#e9ecef`    | `#6c757d`   | `#ffffff`  |
| Hover    | `#adb5bd`    | `#6c757d`   | `#ffffff`  |
| Focused  | `#1074b9`    | `#1074b9`   | `#ffffff`  |
| Error    | `#dc3545`    | `#dc3545`   | `#ffffff`  |
| Disabled | `#e9ecef`    | `#adb5bd`   | `#f5f5f5`  |

**Props/Inputs:**

| Input          | Type    | Default | Description             |
| -------------- | ------- | ------- | ----------------------- |
| `label`        | string  | --      | Float label text        |
| `placeholder`  | string  | --      | Placeholder when empty  |
| `type`         | string  | `text`  | Input type              |
| `required`     | boolean | `false` | Required validation     |
| `errorMessage` | string  | --      | Error text below field  |
| `hint`         | string  | --      | Helper text below field |
| `prefixIcon`   | string  | --      | FontAwesome icon class  |
| `suffixIcon`   | string  | --      | FontAwesome icon class  |
| `disabled`     | boolean | `false` | Disabled state          |

**Dimensions:**

- Min width: 200px
- Height: 56px (with outline appearance)
- Border: 1px solid
- Border radius: 4px
- Padding: 0 12px

**Figma Auto-Layout:**

- Direction: Vertical
- Padding: 0
- Gap: 4px (input to helper)
- Inner field: Horizontal, padding 0 12px, gap 8px

---

### A3. Select Dropdown

**Description:** Wraps `mat-select` with search and multi-select capabilities.

**Variants:**

| Variant    | Description                                    |
| ---------- | ---------------------------------------------- |
| Single     | Standard single-value select                   |
| Multi      | Checkbox-based multi-select                    |
| Searchable | Includes search input at top of dropdown panel |

**Dropdown Panel:**

- Max height: 256px (scrollable)
- Shadow: Level 2
- Border radius: 4px
- Background: `#ffffff` (light), `#303135` (dark)
- Option height: 40px
- Option hover: `rgba(0,0,0,0.04)` (light), `rgba(255,255,255,0.08)` (dark)

**Props/Inputs:**

| Input         | Type    | Default | Description            |
| ------------- | ------- | ------- | ---------------------- |
| `options`     | array   | `[]`    | List of option objects |
| `multiple`    | boolean | `false` | Enable multi-select    |
| `searchable`  | boolean | `false` | Enable search filter   |
| `placeholder` | string  | --      | Placeholder text       |
| `required`    | boolean | `false` | Required validation    |

---

### A4. Checkbox

**Description:** Wraps `mat-checkbox`. In dark mode inherits color via `color: inherit`.

**States:** Unchecked, Checked, Indeterminate, Disabled (each with hover/focus variants).

**Dimensions:**

- Touch target: 40px x 40px
- Checkbox box: 18px x 18px
- Label gap: 8px
- Border radius: 2px

**Props:** `checked`, `indeterminate`, `disabled`, `color`, `labelPosition` (`before` | `after`).

---

### A5. Radio Button

**Description:** Wraps `mat-radio-button` within `mat-radio-group`.

**Dimensions:**

- Touch target: 40px x 40px
- Radio circle: 20px outer, 10px inner dot
- Label gap: 8px
- Group gap: 16px between options

**Props:** `value`, `disabled`, `color`, `labelPosition`.

---

### A6. Date Picker

**Description:** Wraps `mat-datepicker` with calendar popup.

**Variants:**

| Variant | Description                                |
| ------- | ------------------------------------------ |
| Single  | Single date selection                      |
| Range   | Start date + End date with range highlight |

**Calendar Panel:**

- Background: `#ffffff` (light), `#303135` (dark)
- Border: none in dark mode
- Selected date: primary `#1074b9` circle
- Today: outlined circle

**Props:** `value`, `min`, `max`, `startView` (`month` | `year` | `multi-year`), `disabled`.

---

### A7. Toggle Switch

**Description:** Wraps `mat-slide-toggle` for boolean settings.

**Dimensions:**

- Track: 36px x 14px, border-radius 7px
- Thumb: 20px x 20px circle
- Touch target: 40px x 20px

**States:**

- Off: Track `#adb5bd`, Thumb `#ffffff`
- On: Track `rgba(16,116,185,0.5)`, Thumb `#1074b9`
- Disabled: Track `#e9ecef`, Thumb `#adb5bd`

---

### A8. Chip / Tag

**Description:** Wraps `mat-chip` for tags, filters, and selections.

**Variants:**

| Variant  | Background  | Text      | Border        |
| -------- | ----------- | --------- | ------------- |
| Default  | `#e9ecef`   | `#212529` | none          |
| Primary  | `#e3f0fa`   | `#1074b9` | none          |
| Success  | `#e8f5e9`   | `#28a745` | none          |
| Warning  | `#fff8e1`   | `#856404` | none          |
| Error    | `#ffebee`   | `#dc3545` | none          |
| Outlined | transparent | `#212529` | 1px `#e9ecef` |

**Dimensions:**

- Height: 24px
- Padding: 0 8px
- Border radius: 9999px (pill)
- Font: 12px/400
- Icon size: 14px, gap 4px

---

### A9. Tooltip

**Description:** Wraps `matTooltip` directive for hover/focus information.

**Specs:**

- Background: `#212529` (light), `#e9ecef` (dark)
- Text: `#ffffff` (light), `#212529` (dark)
- Font: 11px/400
- Padding: 4px 8px
- Border radius: 4px
- Shadow: Level 4
- Max width: 200px
- Show delay: 300ms
- Arrow: 6px triangle

---

### A10. Progress Bar / Spinner

**Description:** Wraps `mat-progress-bar` and `mat-spinner`.

**Variants:**

| Variant       | Component          | Usage                       |
| ------------- | ------------------ | --------------------------- |
| Linear        | `mat-progress-bar` | Page load, file upload      |
| Determinate   | `mat-progress-bar` | Known progress (percentage) |
| Indeterminate | `mat-progress-bar` | Unknown duration            |
| Circular      | `mat-spinner`      | Button loading, inline      |

**Dimensions:**

- Linear bar height: 4px
- Circular spinner: 40px (default), 20px (inline)
- Track color: `#e9ecef`
- Fill color: `#1074b9` (primary)

---

## Part B: Existing Custom Components (20)

---

### B1. M3Button

**Description:** Material Design 3 styled button component with 5 visual variants following M3 design language.

**Variants:** Filled, Outlined, Elevated, Text, Tonal (same as base Button A1 but with M3 border-radius of 20px).

**Figma Auto-Layout:**

- Same as Button A1 but with `border-radius: 20px`
- Ripple effect on click

---

### B2. FormDialog

**Description:** Reusable dialog that renders a dynamic form from a field configuration array. Used for create/edit actions across all modules.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│  Dialog Title (20px/500)            [X]  │
├──────────────────────────────────────────┤
│                                          │
│  [Dynamic Form Fields]                   │
│    - Text inputs                         │
│    - Selects                             │
│    - Date pickers                        │
│    - Checkboxes                          │
│                                          │
├──────────────────────────────────────────┤
│                    [Cancel]  [Submit]     │
└──────────────────────────────────────────┘
```

**Props/Inputs:**

| Input        | Type   | Description                    |
| ------------ | ------ | ------------------------------ |
| `title`      | string | Dialog header text             |
| `layout`     | object | Form field configuration array |
| `entityData` | object | Pre-filled data for edit mode  |

**Dimensions:**

- Width: 600px (medium)
- Padding: 24px
- Border radius: 12px
- Overlay: `rgba(0,0,0,0.32)`
- Background: `#ffffff` (light), `#303135` (dark)

---

### B3. DeleteDialog

**Description:** Confirmation dialog with danger styling for destructive delete operations.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│  ⚠ Delete [Entity]?                      │
├──────────────────────────────────────────┤
│                                          │
│  Are you sure you want to delete         │
│  [entity name]? This action cannot       │
│  be undone.                              │
│                                          │
├──────────────────────────────────────────┤
│                    [Cancel]  [Delete]     │
└──────────────────────────────────────────┘
```

**Styling:**

- Delete button: `color="warn"`, Filled variant (`#dc3545` background)
- Cancel button: Text variant
- Icon: `fa-exclamation-triangle` in `#dc3545`
- Width: 400px (small)

---

### B4. ConfirmationDialog

**Description:** Generic yes/no confirmation dialog for non-destructive actions.

**Props:**

| Input          | Type   | Description               |
| -------------- | ------ | ------------------------- |
| `title`        | string | Dialog title              |
| `message`      | string | Confirmation message body |
| `confirmLabel` | string | Confirm button text       |
| `cancelLabel`  | string | Cancel button text        |

**Styling:**

- Confirm button: Filled primary
- Cancel button: Outlined or Text
- Width: 400px

---

### B5. CancelDialog

**Description:** Specific confirmation dialog for cancel/abandon actions. Warns about unsaved changes.

**Styling:**

- Warning icon: `fa-exclamation-circle` in `#ffc107`
- Action button: Outlined warn
- Width: 400px

---

### B6. ErrorDialog

**Description:** Displays error information from API failures or validation errors.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│  Error                              [X]  │
├──────────────────────────────────────────┤
│  [fa-times-circle]                       │
│                                          │
│  Error message text                      │
│  Technical details (collapsible)         │
│                                          │
├──────────────────────────────────────────┤
│                              [Dismiss]   │
└──────────────────────────────────────────┘
```

**Styling:**

- Icon: `fa-times-circle` in `#dc3545`
- Error text: `#dc3545`
- Width: 500px

---

### B7. EnableDialog

**Description:** Confirmation dialog for enabling a resource (client, account, etc.).

**Styling:**

- Icon: `fa-check-circle` in `#28a745`
- Confirm button: Filled primary
- Width: 400px

---

### B8. DisableDialog

**Description:** Confirmation dialog for disabling a resource.

**Styling:**

- Icon: `fa-ban` in `#ffc107`
- Confirm button: Outlined warn
- Width: 400px

---

### B9. FileUpload

**Description:** File input with drag-and-drop zone, preview, and progress indicator.

**Anatomy:**

```
┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┐
│                                          │
│     [fa-cloud-upload-alt]                │
│     Drag & drop files here               │
│     or click to browse                   │
│                                          │
│     Accepted: .pdf, .jpg, .png           │
│     Max size: 5MB                        │
└─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘
```

**States:**

- Default: dashed border `#e9ecef`
- Hover: dashed border `#1074b9`, bg `#e3f0fa`
- Drag over: solid border `#1074b9`, bg `#e3f0fa`
- Uploading: progress bar overlay
- Error: dashed border `#dc3545`

**Dimensions:**

- Min height: 120px
- Padding: 24px
- Border: 2px dashed
- Border radius: 8px

---

### B10. EntityDatatableTab

**Description:** Dynamic data table for entity extension fields (custom datatables). Renders columns based on server-defined schema.

**Props:**

| Input           | Type   | Description                 |
| --------------- | ------ | --------------------------- |
| `entityId`      | number | Parent entity ID            |
| `datatableName` | string | Server datatable name       |
| `columns`       | array  | Column definitions from API |

**Table Specs:**

- Header bg: `#f5f5f5` (light), `#303135` (dark)
- Row height: 40px
- Cell padding: 6px 8px
- Even row: `#f8f9fa` (light), `#303135` (dark)
- Hover: `rgba(0,0,0,0.04)` (light), `rgba(255,255,255,0.08)` (dark)
- Border: 1px solid `#e9ecef`

---

### B11. EntityDocumentsTab

**Description:** Document list with upload capability. Displays document cards in a grid with thumbnail previews.

**Document Card:**

- Background: `#ffffff` (light), `#2d2d2d` (dark)
- Border: 1px solid `#e9ecef` (light), `#444` (dark)
- Shadow: Level 1 (light), `0 1px 4px rgba(0,0,0,0.30)` (dark)
- Hover shadow: Level 2 (light), `0 4px 12px rgba(0,0,0,0.50)` (dark)
- Thumbnail area bg: `#f5f5f5` (light), `#1a1a1a` (dark)
- Title: `#212529` (light), `rgba(255,255,255,0.87)` (dark)
- Meta text: `#6c757d` (light), `#aaa` (dark)

**Empty State:**

- Background: `#f5f5f5` (light), `#2d2d2d` (dark)
- Border: dashed `#adb5bd` (light), `#555` (dark)
- Muted text: `#6c757d` (light), `#888` (dark)

---

### B12. EntityNotesTab

**Description:** Notes list with add/edit functionality. Each note displayed as a card.

**Note Card:**

- Background: `#ffffff` (light), `#2d2d2d` (dark)
- Border: 1px solid `#e9ecef` (light), `#444` (dark)
- Shadow: `0 2px 4px rgba(0,0,0,0.08)` (light), `0 2px 4px rgba(0,0,0,0.30)` (dark)
- Hover shadow: `0 4px 8px rgba(0,0,0,0.12)` (light), `0 4px 8px rgba(0,0,0,0.50)` (dark)
- Content text: `#212529` (light), `rgba(255,255,255,0.87)` (dark)
- Footer border-top: `#e9ecef` (light), `#444` (dark)
- Created-by link: `#1074b9` (light), `#0098ff` (dark)
- Date text: `#6c757d` (light), `#bbb` (dark)

---

### B13. AccountNumber

**Description:** Formatted display of account numbers with copy-to-clipboard functionality.

**Specs:**

- Font: `body-1` (14px/400) monospace variant
- Copy icon: `fa-copy` 14px, hover color `#1074b9`
- Copy label color: `var(--md-sys-color-on-surface-variant, #c4c6d0)` (dark)

---

### B14. ExternalIdentifier

**Description:** Display component for external system identifiers.

**Specs:**

- Font: `body-2` (12px/400)
- Color: `#6c757d`
- Prefix label: overline style

---

### B15. SearchTool

**Description:** Global search input in the top navigation bar.

**Specs:**

- Width: 300px (desktop), 100% (mobile overlay)
- Height: 36px
- Background: `rgba(255,255,255,0.15)` on nav bar
- Border radius: 4px
- Icon: `fa-search` 16px
- Placeholder: "Search..." in `rgba(255,255,255,0.7)`

---

### B16. LanguageSelector

**Description:** Dropdown to switch application language.

**Specs:**

- Icon: `fa-globe` 16px
- Dropdown: Standard select panel
- Position: Top navigation bar, right section

---

### B17. ThemeToggle

**Description:** Toggle between light and dark mode. Toggles `.dark-theme` class on `<body>`.

**Specs:**

- Icons: `fa-sun` (light active), `fa-moon` (dark active)
- Size: 24px icon in 36px touch target
- Position: Top navigation bar

---

### B18. NotificationsTray

**Description:** Notification bell icon with dropdown panel showing recent notifications.

**Specs:**

- Bell icon: `fa-bell` 20px
- Badge: 16px circle, `#dc3545` bg, white text, positioned top-right
- Panel width: 320px
- Panel max height: 400px (scrollable)
- Panel shadow: Level 3

---

### B19. StepperButtons

**Description:** Previous/Next navigation buttons for multi-step wizard forms.

**Anatomy:**

```
[< Previous]                    [Next >]
```

**Specs:**

- Previous: Text button with left arrow icon
- Next: Filled primary button with right arrow icon
- Submit (last step): Filled primary "Submit" button
- Spacing: Flex space-between, full width
- Top margin: 24px

---

### B20. InputAmount

**Description:** Currency-formatted number input with locale-aware formatting.

**Specs:**

- Prefix: Currency symbol from locale
- Alignment: Right-aligned text
- Format: Thousands separator, 2 decimal places
- Input type: `text` (with masking)
- Width: 200px default

---

## Part C: New M-SACCO Components (12)

---

### C1. StatusTabs

**Description:** Horizontal tab bar with count badges, used at the top of list pages to filter by entity status.

**Anatomy:**

```
┌──────────────────────────────────────────────────────────┐
│  [All (156)]  [Pending (23)]  [Active (120)]  [Closed (13)] │
└──────────────────────────────────────────────────────────┘
```

**Specs:**

- Height: 40px
- Background: transparent
- Tab padding: 8px 16px
- Tab font: 14px/500
- Active tab: Primary color text + 2px bottom border `#1074b9`
- Inactive tab: `#6c757d` text
- Badge: 12px font, `#e9ecef` bg rounded pill, 4px left margin
- Active badge: `#e3f0fa` bg, `#1074b9` text
- Hover: `rgba(16,116,185,0.04)` bg
- Scrollable on mobile (horizontal overflow)

**Props:**

| Input      | Type   | Description                     |
| ---------- | ------ | ------------------------------- |
| `tabs`     | array  | `{label, count, value}` objects |
| `selected` | string | Currently selected tab value    |

**Figma Auto-Layout:**

- Direction: Horizontal
- Gap: 0
- Alignment: Bottom / Left
- Hug contents horizontally

---

### C2. DataTableWithExport

**Description:** Enhanced data table with export buttons (Copy, Excel, PDF) and "Showing X of Y" pagination footer.

**Anatomy:**

```
┌──────────────────────────────────────────────────────────┐
│  [Copy] [Excel] [PDF]                     [Search: ___]  │
├──────────────────────────────────────────────────────────┤
│  Col 1 ↕  │  Col 2 ↕  │  Col 3 ↕  │  Col 4 ↕  │ Actions│
├──────────────────────────────────────────────────────────┤
│  data      │  data      │  data      │  data      │  ...  │
│  data      │  data      │  data      │  data      │  ...  │
│  data      │  data      │  data      │  data      │  ...  │
├──────────────────────────────────────────────────────────┤
│  Showing 1 to 10 of 156 entries    [< 1 2 3 ... 16 >]   │
└──────────────────────────────────────────────────────────┘
```

**Export Buttons:**

- Style: Small outlined buttons, 28px height
- Copy: `fa-copy` icon
- Excel: `fa-file-excel` icon, green tint
- PDF: `fa-file-pdf` icon, red tint

**Pagination:**

- Text: "Showing X to Y of Z entries" -- 12px/400 `#6c757d`
- Page buttons: 28px square, 4px radius
- Active page: `#1074b9` bg, white text

**Table:**

- Header row height: 40px, bg `#f5f5f5`, font 12px/500 uppercase
- Data row height: 40px, font 14px/400
- Sort icon: `fa-sort` / `fa-sort-up` / `fa-sort-down` 12px
- Hover row: `rgba(0,0,0,0.04)`
- Even row: `#f8f9fa` (light), `#303135` (dark)

---

### C3. ApprovalActionBar

**Description:** Sticky bottom bar that appears on entity detail pages requiring approval actions.

**Anatomy:**

```
┌──────────────────────────────────────────────────────────┐
│  [Approve ✓]  [Reject ✗]  [More Actions ▾]              │
└──────────────────────────────────────────────────────────┘
```

**Specs:**

- Position: `sticky`, bottom 0, full width
- Height: 56px
- Background: `#ffffff`, shadow Level 2 (inverted, upward)
- Padding: 8px 24px
- Approve: Filled primary button, `fa-check` icon
- Reject: Outlined warn button, `fa-times` icon
- More: Text button with `fa-ellipsis-v` dropdown
- Gap: 8px between buttons
- Alignment: Left-aligned buttons

---

### C4. WizardStepIndicator

**Description:** Numbered circles connected by lines showing progress through a multi-step form.

**Anatomy:**

```
  (1)────────(2)────────(3)────────(4)────────(5)
General    Address     Family      IDs       Preview
```

**Specs:**

- Circle: 32px diameter
- Completed: `#1074b9` bg, white text, `fa-check`
- Current: `#1074b9` border 2px, white bg, `#1074b9` text
- Upcoming: `#e9ecef` bg, `#6c757d` text
- Connector line: 2px height
- Completed connector: `#1074b9`
- Upcoming connector: `#e9ecef`
- Label: 12px/400, 4px below circle
- Current label: 12px/500, `#1074b9`

**Responsive:**

- Desktop: Horizontal with labels
- Mobile: Hidden; replaced with "Step X of Y" text

**Figma Auto-Layout:**

- Direction: Horizontal
- Gap: 0 (circles fill with connectors)
- Each step: Vertical stack (circle + label)

---

### C5. StatusBadge

**Description:** Colored pill chip displaying entity status. Maps status strings to color variants.

**Status Mapping:**

| Status    | Background | Text      | Border    |
| --------- | ---------- | --------- | --------- |
| Active    | `#e8f5e9`  | `#28a745` | none      |
| Pending   | `#fff8e1`  | `#856404` | none      |
| Rejected  | `#ffebee`  | `#dc3545` | none      |
| Closed    | `#f5f5f5`  | `#6c757d` | none      |
| Overdue   | `#ffebee`  | `#dc3545` | none      |
| Draft     | `#f8f9fa`  | `#adb5bd` | `#adb5bd` |
| Submitted | `#e0f7fa`  | `#17a2b8` | none      |
| Approved  | `#e8f5e9`  | `#28a745` | none      |
| Disbursed | `#e3f0fa`  | `#1074b9` | none      |

**Specs:**

- Height: 24px
- Padding: 0 12px
- Border radius: 9999px
- Font: 12px/500 uppercase
- Optional dot indicator: 6px circle before text

**Props:**

| Input     | Type    | Description                   |
| --------- | ------- | ----------------------------- |
| `status`  | string  | Status key (mapped to colors) |
| `showDot` | boolean | Show colored dot before text  |

---

### C6. VariableInsertButton

**Description:** Button that opens a dropdown of SMS template variables for insertion into message fields.

**Anatomy:**

```
[Insert Variable ▾]
┌──────────────────┐
│ {{clientName}}   │
│ {{accountNo}}    │
│ {{loanAmount}}   │
│ {{dueDate}}      │
│ {{balance}}      │
│ ...              │
└──────────────────┘
```

**Specs:**

- Trigger: Small outlined button, 28px height
- Panel: 240px width, max-height 300px scrollable
- Option: 36px height, 14px/400 font, monospace for variable names
- Hover: `#e3f0fa` bg
- Click: Inserts variable at cursor position in target textarea

---

### C7. DragAndDropFieldList

**Description:** Reorderable list of fields for data export configuration. Items can be dragged to reorder.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│ ⠿  Field Name 1                    [×]  │
├──────────────────────────────────────────┤
│ ⠿  Field Name 2                    [×]  │
├──────────────────────────────────────────┤
│ ⠿  Field Name 3                    [×]  │
└──────────────────────────────────────────┘
```

**Specs:**

- Uses Angular CDK `cdkDragDrop`
- Item height: 40px
- Grip icon: `fa-grip-vertical` 16px, `#adb5bd`
- Remove icon: `fa-times` 14px
- Drag preview: Shadow Level 2, border-radius 4px, white bg (dark: white bg, `#303135` text)
- Drop placeholder: dashed border `#1074b9`, `#e3f0fa` bg

---

### C8. ClientTransferCard

**Description:** Summary card showing transfer details when moving a client between branches/officers.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│  Transfer Summary                        │
├──────────────────────────────────────────┤
│  From: [Branch A]  →  To: [Branch B]    │
│  Client: John Doe                        │
│  Transfer Date: 2026-03-13               │
│  Note: Optional transfer note            │
├──────────────────────────────────────────┤
│                  [Cancel]  [Confirm]     │
└──────────────────────────────────────────┘
```

**Specs:**

- Card: 8px radius, Level 1 shadow, 16px padding
- Arrow: `fa-arrow-right` 20px, `#1074b9`
- Labels: 12px/400 `#6c757d`
- Values: 14px/500 `#212529`

---

### C9. GroupMemberSelector

**Description:** Search-and-select component for adding members to a group. Includes search input and selected member chips.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│  Search members: [________________]       │
├──────────────────────────────────────────┤
│  Search results:                          │
│  ○ John Doe (#1234)                      │
│  ○ Jane Smith (#5678)                    │
│  ○ Bob Wilson (#9012)                    │
├──────────────────────────────────────────┤
│  Selected (3):                            │
│  [Alice Jones ×] [Tom Brown ×]           │
└──────────────────────────────────────────┘
```

**Specs:**

- Search input: Standard text input with `fa-search` prefix
- Result list: max-height 200px scrollable, 40px per item
- Selected chips: accent variant chips with remove `fa-times`

---

### C10. GuarantorSelector

**Description:** Three-mode guarantor selection component: existing client, external person, or group member.

**Modes:**

| Mode            | UI                                     |
| --------------- | -------------------------------------- |
| Existing Client | Client search autocomplete             |
| External        | Form fields (Name, DOB, Address, etc.) |
| Group Member    | Select from group members list         |

**Specs:**

- Mode selector: Radio group at top
- Each mode reveals different form content
- Card container: 8px radius, 16px padding

---

### C11. CRBScoreCard

**Description:** Credit Reference Bureau score visualization with gauge chart and summary.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│  Credit Score                            │
│                                          │
│        ╭───────────────╮                 │
│       ╱   Score: 725    ╲                │
│      ╱     ████████░░    ╲               │
│     ╰──────────────────────╯             │
│           Good Standing                  │
│                                          │
│  Last Updated: 2026-03-01                │
│  Bureau: TransUnion                      │
└──────────────────────────────────────────┘
```

**Score Ranges:**

| Range   | Color     | Label     |
| ------- | --------- | --------- |
| 0-300   | `#dc3545` | Poor      |
| 301-500 | `#ffc107` | Fair      |
| 501-700 | `#17a2b8` | Good      |
| 701-850 | `#28a745` | Excellent |

**Specs:**

- Gauge: SVG arc, 200px diameter
- Score text: 32px/700 centered
- Card: 8px radius, 16px padding

---

### C12. MPESAConfigPanel

**Description:** Configuration form panel for MPESA mobile payment channel integration.

**Anatomy:**

```
┌──────────────────────────────────────────┐
│  MPESA Configuration                     │
├──────────────────────────────────────────┤
│  Business Short Code: [____________]     │
│  App Key:             [____________]     │
│  App Secret:          [____________]     │
│  Passkey:             [____________]     │
│  Callback URL:        [____________]     │
│  Environment:         [Sandbox ▾]        │
│                                          │
│  ☑ Enable Paybill     ☑ Enable Till     │
├──────────────────────────────────────────┤
│               [Test Connection] [Save]   │
└──────────────────────────────────────────┘
```

**Specs:**

- Card: 8px radius, 24px padding
- Fields: Standard text input style
- Secret fields: Password type with show/hide toggle (`fa-eye` / `fa-eye-slash`)
- Test button: Outlined primary
- Save button: Filled primary
- Form layout: Single column, 16px field gap
