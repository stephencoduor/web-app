# DESIGN-005: Figma File Organization

> M-SACCO Design System -- Figma File Structure, Token Setup, and Handoff Guidelines
>
> This document defines how the Figma design file should be organized to support efficient design, prototyping, and developer handoff.

---

## 1. File Structure -- 11 Pages

The Figma file is organized into 11 top-level pages. Each page has a specific purpose and naming convention.

| #   | Page Name           | Purpose                                             |
| --- | ------------------- | --------------------------------------------------- |
| 1   | Cover               | File cover art, version info, table of contents     |
| 2   | Foundations         | Color, typography, spacing, elevation, icons tokens |
| 3   | Components          | All 42 component definitions with variants          |
| 4   | Patterns            | 4 layout patterns (List, Detail, Wizard, Dialog)    |
| 5   | Clients Module      | 14 client screens (desktop + tablet + mobile)       |
| 6   | Groups Module       | 6 group screens                                     |
| 7   | Loans Module        | 10 loan screens                                     |
| 8   | Accounting Module   | 6 accounting screens                                |
| 9   | Products Module     | 4 product screens                                   |
| 10  | Organization Module | 4 organization screens                              |
| 11  | New Features Module | 4 new feature screens                               |

### Page Naming Convention

```
[Number] [Page Name]
```

Examples: `01 Cover`, `02 Foundations`, `03 Components`, `05 Clients Module`

### Section Organization Within Pages

Each module page (5-11) follows this internal structure:

```
[Module Name] Module
├── Desktop (1440px frames)
│   ├── [Screen Name] -- Desktop
│   ├── [Screen Name] -- Desktop
│   └── ...
├── Tablet (768px frames)
│   ├── [Screen Name] -- Tablet
│   └── ...
└── Mobile (375px frames)
    ├── [Screen Name] -- Mobile
    └── ...
```

---

## 2. Component Organization

### 2.1 Component Page Layout

Page `03 Components` is organized into three horizontal sections:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Section A: Base Material Components (10)                           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │ Button  │ │  Input  │ │ Select  │ │Checkbox │ │  Radio  │     │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘     │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐     │
│  │DatePick │ │ Toggle  │ │  Chip   │ │ Tooltip │ │Progress │     │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘     │
├─────────────────────────────────────────────────────────────────────┤
│  Section B: Existing Custom Components (20)                         │
│  (4 rows of 5 components)                                           │
├─────────────────────────────────────────────────────────────────────┤
│  Section C: New M-SACCO Components (12)                             │
│  (3 rows of 4 components)                                           │
└─────────────────────────────────────────────────────────────────────┘
```

Each component block contains:

1. Component name label (16px/500)
2. All variants laid out horizontally
3. All states laid out vertically per variant
4. Spacing annotations

### 2.2 Variant Naming Convention

Figma component variants use a structured property naming system:

```
[Property]=[Value]
```

**Standard Properties:**

| Property | Values                                           |
| -------- | ------------------------------------------------ |
| State    | Default, Hover, Active, Focused, Disabled, Error |
| Size     | SM, MD, LG                                       |
| Style    | Filled, Outlined, Elevated, Text, Tonal          |
| Color    | Primary, Accent, Warn, Neutral                   |
| Theme    | Light, Dark                                      |

**Examples:**

```
Button / State=Default, Size=MD, Style=Filled, Color=Primary
Button / State=Hover, Size=MD, Style=Outlined, Color=Primary
TextInput / State=Focused, Size=MD
TextInput / State=Error, Size=MD
StatusBadge / Status=Active
StatusBadge / Status=Pending
StatusBadge / Status=Rejected
```

### 2.3 Component Properties (Figma Component Properties)

Each component exposes these Figma-native properties:

**Boolean Properties:**

- `Show Icon` -- toggles icon visibility
- `Show Helper Text` -- toggles helper/error text below inputs
- `Show Badge` -- toggles count badge on tabs
- `Disabled` -- toggles disabled visual state

**Text Properties:**

- `Label` -- button/input label text
- `Placeholder` -- input placeholder text
- `Helper Text` -- text below input fields
- `Error Message` -- error text below input fields

**Instance Swap Properties:**

- `Icon` -- swappable icon instance
- `Status` -- swappable StatusBadge variant

### 2.4 Auto-Layout Specifications per Component

**Button:**

```
Auto-Layout: Horizontal
Padding: [varies by size - see DESIGN-002]
Gap: 4 (icon to label)
Alignment: Center / Center
Min width: 64
Border radius: 4 (or 20 for M3)
```

**Text Input:**

```
Auto-Layout: Vertical
Padding: 0
Gap: 4 (field to helper text)
  Inner Field:
    Auto-Layout: Horizontal
    Padding: 0 12
    Gap: 8 (prefix to text to suffix)
    Height: 56
```

**Card:**

```
Auto-Layout: Vertical
Padding: 16 (or 24 for large)
Gap: 16
Border radius: 8
Fill: White
Effect: Level 1 shadow
```

**Dialog:**

```
Auto-Layout: Vertical
Padding: 0
Gap: 0
Border radius: 12
  Title: H-pad 24, V-pad 16, H: 56
  Content: Padding 24, Gap 16
  Actions: H-pad 16, V-pad 8, Gap 8, Align: End
```

**StatusTabs:**

```
Auto-Layout: Horizontal
Padding: 0
Gap: 0
  Each Tab:
    Auto-Layout: Horizontal
    Padding: 8 16
    Gap: 4 (label to badge)
```

**DataTable:**

```
Auto-Layout: Vertical
Padding: 0
Gap: 0
  Header Row:
    Auto-Layout: Horizontal
    Height: 40
    Fill: #f5f5f5
    Each Cell: Padding 6 8
  Data Row:
    Auto-Layout: Horizontal
    Height: 40
    Each Cell: Padding 6 8
```

---

## 3. Naming Conventions

### 3.1 Frame Naming (BEM-Inspired)

Pattern: `Module/Component/Variant`

**Screen Frames:**

```
Clients/ClientList/Desktop
Clients/ClientList/Tablet
Clients/ClientList/Mobile
Clients/CreateClient/Step1-General/Desktop
Clients/ClientDetail/General/Desktop
Loans/LoanDetail/RepaymentSchedule/Desktop
Loans/CreateLoan/Step3-Charges/Mobile
Accounting/ChartOfAccounts/Desktop
```

**Component Frames:**

```
Base/Button/Filled-Primary-MD
Base/TextInput/Outlined-Default
Base/Select/Single-Default
Custom/FormDialog/Medium
Custom/DeleteDialog/Default
MSACCO/StatusTabs/Default
MSACCO/StatusBadge/Active
MSACCO/DataTableWithExport/Default
MSACCO/WizardStepIndicator/Step2of5
MSACCO/ApprovalActionBar/Default
```

### 3.2 Layer Naming

| Layer Type | Convention          | Example              |
| ---------- | ------------------- | -------------------- |
| Frame      | PascalCase          | `HeaderCard`         |
| Text       | camelCase           | `pageTitle`          |
| Icon       | icon/[name]         | `icon/search`        |
| Image      | img/[description]   | `img/profilePhoto`   |
| Divider    | divider/[direction] | `divider/horizontal` |
| Spacer     | spacer/[size]       | `spacer/16`          |
| Background | bg                  | `bg`                 |

### 3.3 Color Style Naming

```
Primary/50
Primary/100
Primary/500 (Default)
Primary/700
Primary/900
Accent/50
Accent/500 (Default)
Accent/700
Warn/500
Neutral/Text
Neutral/Secondary
Neutral/Disabled
Neutral/Border
Neutral/TableHeader
Neutral/PageBg
Neutral/Surface
Semantic/Success
Semantic/Warning
Semantic/Error
Semantic/Info
Status/Active
Status/Pending
Status/Rejected
...
Dark/Surface
Dark/ElevatedSurface
Dark/Text
Dark/Secondary
Dark/Border
Dark/Link
```

### 3.4 Text Style Naming

```
Typography/Display
Typography/H1
Typography/H2
Typography/H3
Typography/Body1
Typography/Body2
Typography/Button
Typography/Caption
Typography/Overline
```

---

## 4. Design Tokens -- Figma Variables

### 4.1 Variable Collections

Create 4 Figma Variable collections:

**Collection 1: Colors**

| Variable Name            | Light Mode | Dark Mode                |
| ------------------------ | ---------- | ------------------------ |
| `color/primary/500`      | `#1074b9`  | `#1074b9`                |
| `color/primary/100`      | `#5ba2ec`  | `#5ba2ec`                |
| `color/primary/700`      | `#004989`  | `#004989`                |
| `color/accent/500`       | `#b4d575`  | `#b4d575`                |
| `color/accent/100`       | `#e7ffa5`  | `#e7ffa5`                |
| `color/accent/700`       | `#83a447`  | `#83a447`                |
| `color/warn/500`         | `#f44336`  | `#f44336`                |
| `color/text/primary`     | `#212529`  | `rgba(255,255,255,0.87)` |
| `color/text/secondary`   | `#6c757d`  | `#aaa`                   |
| `color/text/disabled`    | `#adb5bd`  | `#666`                   |
| `color/bg/page`          | `#f8f9fa`  | `#121212`                |
| `color/bg/surface`       | `#ffffff`  | `#1e1e1e`                |
| `color/bg/elevated`      | `#ffffff`  | `#2d2d2d`                |
| `color/bg/navTabs`       | `#f2f2f2`  | `#303135`                |
| `color/bg/tableHeader`   | `#f5f5f5`  | `#303135`                |
| `color/bg/tableEvenRow`  | `#f8f9fa`  | `#303135`                |
| `color/border/default`   | `#e9ecef`  | `#444`                   |
| `color/link`             | `#1074b9`  | `#0098ff`                |
| `color/semantic/success` | `#28a745`  | `#28a745`                |
| `color/semantic/warning` | `#ffc107`  | `#ffc107`                |
| `color/semantic/error`   | `#dc3545`  | `#dc3545`                |
| `color/semantic/info`    | `#17a2b8`  | `#17a2b8`                |

**Collection 2: Spacing**

| Variable Name | Value |
| ------------- | ----- |
| `space/0`     | 0     |
| `space/0.5`   | 2     |
| `space/1`     | 4     |
| `space/2`     | 8     |
| `space/3`     | 12    |
| `space/4`     | 16    |
| `space/5`     | 20    |
| `space/6`     | 24    |
| `space/8`     | 32    |
| `space/10`    | 40    |
| `space/12`    | 48    |
| `space/16`    | 64    |
| `space/20`    | 80    |
| `space/24`    | 96    |

**Collection 3: Border Radius**

| Variable Name | Value |
| ------------- | ----- |
| `radius/none` | 0     |
| `radius/xs`   | 2     |
| `radius/sm`   | 4     |
| `radius/md`   | 8     |
| `radius/lg`   | 12    |
| `radius/xl`   | 16    |
| `radius/2xl`  | 20    |
| `radius/pill` | 9999  |

**Collection 4: Typography**

| Variable Name             | Value  |
| ------------------------- | ------ |
| `font/family`             | Roboto |
| `font/size/display`       | 32     |
| `font/size/h1`            | 24     |
| `font/size/h2`            | 20     |
| `font/size/h3`            | 16     |
| `font/size/body1`         | 14     |
| `font/size/body2`         | 12     |
| `font/size/caption`       | 11     |
| `font/size/overline`      | 10     |
| `font/weight/regular`     | 400    |
| `font/weight/medium`      | 500    |
| `font/weight/semibold`    | 600    |
| `font/weight/bold`        | 700    |
| `font/lineHeight/default` | 1.5    |
| `font/lineHeight/heading` | 1.2    |

### 4.2 Variable Modes

Configure two modes on the Colors collection:

| Mode  | Description                                 |
| ----- | ------------------------------------------- |
| Light | Default mode -- all light theme values      |
| Dark  | Dark theme values from `_dark_content.scss` |

Switching the mode on a top-level frame instantly applies dark theme to all children.

---

## 5. Auto-Layout Rules

### 5.1 Global Rules

| Rule                           | Specification                                  |
| ------------------------------ | ---------------------------------------------- |
| All containers use auto-layout | No absolute positioning except overlays        |
| Spacing uses variables         | Reference `space/*` variables, not raw numbers |
| Fill vs Hug                    | Containers: Fill width. Content: Hug height    |
| Min width on inputs            | 200px                                          |
| Max width on content           | 1400px                                         |

### 5.2 Per Component Type

**Page-Level Frames:**

```
Direction: Vertical
Padding: 0
Gap: 0
Width: Fill container
Height: Hug contents
Max width: viewport width
```

**Content Containers:**

```
Direction: Vertical
Padding: 24 (desktop) / 16 (tablet) / 12 (mobile)
Gap: 16
Width: Fill container
Max width: 1400px
Alignment: Top / Center
```

**Cards:**

```
Direction: Vertical
Padding: space/4 (16px)
Gap: space/4 (16px)
Width: Fill container
Border radius: radius/md (8)
Fill: color/bg/surface
Effect: shadow Level 1
```

**Form Layouts:**

```
Direction: Vertical (wrapper)
Gap: space/4 (16px)
  Field Row:
    Direction: Horizontal (wrap)
    Gap: space/4 (16px)
    Each field: Min width 200, Fill container
```

**Button Groups:**

```
Direction: Horizontal
Gap: space/2 (8px)
Alignment: Center / End (right-aligned for dialog actions)
```

**Table:**

```
Direction: Vertical
Gap: 0
Width: Fill container
  Row: Horizontal, Gap: 0, Height: 40
    Cell: Padding 6 8, text truncate
```

---

## 6. Responsive Frames

### 6.1 Frame Sizes

Each screen is designed at three breakpoints:

| Frame Name | Width  | Height | Usage                 |
| ---------- | ------ | ------ | --------------------- |
| Desktop    | 1440px | Hug    | Primary design target |
| Tablet     | 768px  | Hug    | iPad portrait         |
| Mobile     | 375px  | Hug    | iPhone SE / standard  |

### 6.2 Responsive Layout Rules

**Desktop (1440px):**

- Sidebar: Expanded (240px width)
- Content max-width: 1400px, centered
- Forms: 2 columns
- Tables: Full columns visible
- Dialogs: Centered at specified width

**Tablet (768px):**

- Sidebar: Collapsed rail (64px) or hidden
- Content: Full width with 16px margins
- Forms: 2 columns (but narrower)
- Tables: Horizontal scroll if needed, some columns hidden
- Dialogs: 90vw width

**Mobile (375px):**

- Sidebar: Hidden (hamburger menu)
- Content: Full width with 12px margins
- Forms: 1 column
- Tables: Card list layout
- Dialogs: Full screen (100vw, no border-radius)
- Step indicator: Text only ("Step X of Y")
- Tabs: Horizontal scroll

### 6.3 Frame Organization on Canvas

```
┌───────────────────────────────────────────────────────────────────────┐
│ Page: "05 Clients Module"                                             │
│                                                                       │
│ ┌─── Desktop (1440px) ──────────────────────────────────┐             │
│ │                                                        │             │
│ │  [Client List]  [Create Step1]  [Create Step2]  ...   │             │
│ │                                                        │             │
│ └────────────────────────────────────────────────────────┘             │
│                                                                       │
│ ┌─── Tablet (768px) ──────────────────────────┐                       │
│ │                                              │                       │
│ │  [Client List]  [Create Step1]  ...          │                       │
│ │                                              │                       │
│ └──────────────────────────────────────────────┘                       │
│                                                                       │
│ ┌─── Mobile (375px) ────────────┐                                     │
│ │                                │                                     │
│ │  [Client List]  [Create]  ...  │                                     │
│ │                                │                                     │
│ └────────────────────────────────┘                                     │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

Spacing between frames: 100px horizontal, 200px between breakpoint sections.

---

## 7. Prototype Flows

### 7.1 Client Creation Wizard Flow

```
[Client List] ──(Click "Create Client")──→ [Step 1: General]
  ──(Click "Next")──→ [Step 2: Address]
  ──(Click "Next")──→ [Step 3: Family]
  ──(Click "Next")──→ [Step 4: Identification]
  ──(Click "Next")──→ [Step 5: Preview]
  ──(Click "Submit")──→ [Confirmation Dialog]
  ──(Click "Confirm")──→ [Client Detail: General]
```

**Interactions:**

- Page transitions: Smart Animate, 300ms ease
- Dialog open: Dissolve in, 200ms
- Dialog close: Dissolve out, 150ms
- Back navigation: Slide right, 300ms

### 7.2 Loan Application and Approval Flow

```
[Loan List] ──(Click "Create Loan")──→ [Step 1: Details]
  ──→ [Step 2: Terms]
  ──→ [Step 3: Charges]
  ──→ [Step 4: Guarantors]
  ──→ [Step 5: Preview]
  ──→ [Submit] ──→ [Loan Detail: General (Pending)]
  ──(Click "Approve" on ApprovalActionBar)──→ [Approval Dialog]
  ──(Confirm)──→ [Loan Detail: General (Approved)]
  ──(Click "Disburse")──→ [Disburse Dialog]
  ──(Confirm)──→ [Loan Detail: General (Disbursed)]
```

### 7.3 Group Creation and Member Management Flow

```
[Group List] ──(Click "Create Group")──→ [Step 1: General]
  ──→ [Step 2: Members (GroupMemberSelector)]
  ──→ [Step 3: Preview]
  ──→ [Submit] ──→ [Group Detail: General]
  ──(Click Members tab)──→ [Group Detail: Members]
  ──(Click "Add Member")──→ [Member Search Dialog]
  ──(Select client + Confirm)──→ [Group Detail: Members (updated)]
```

### 7.4 Client Transfer Between Branches Flow

```
[Client Detail] ──(Click "Transfer")──→ [Transfer Form]
  ──(Select target branch)──→ [ClientTransferCard preview appears]
  ──(Click "Confirm Transfer")──→ [Confirmation Dialog]
  ──(Confirm)──→ [Client Detail (updated branch)]
```

### 7.5 Prototype Settings

| Setting            | Value                             |
| ------------------ | --------------------------------- |
| Device frame       | None (responsive)                 |
| Starting frame     | Module List page (desktop)        |
| Transition default | Smart Animate, 300ms, Ease In Out |
| Overlay (dialogs)  | Manual overlay, centered          |
| Scroll behavior    | Vertical scroll on all pages      |
| Fixed elements     | Nav bar (top), Footer (bottom)    |

---

## 8. Handoff Notes

### 8.1 What to Include for Developer Handoff

Every screen frame should include the following annotations:

**Spacing Annotations:**

- Red dimension lines showing padding and gaps
- Reference spacing variable names (e.g., `space/4 = 16px`)
- Annotate non-obvious spacing (between sections, within cards)

**Color References:**

- Color variable names next to colored elements
- Opacity values where applicable
- Dark mode variant note where colors change

**Typography:**

- Text style name reference (e.g., `Typography/H2`)
- Font size, weight, line-height for non-standard text
- Color variable for text elements

**Component Annotations:**

- Component name and variant used
- Any overridden properties
- State-specific notes (hover colors, focus rings, etc.)

**Interaction Notes:**

- Click targets and their actions
- Hover states
- Keyboard navigation notes
- Form validation rules
- Loading states

### 8.2 Annotation Format

Use a consistent annotation box style:

```
┌─ Annotation ──────────────────────┐
│ Component: StatusBadge/Active      │
│ Font: 12px/500 uppercase           │
│ Color: color/semantic/success      │
│ BG: #e8f5e9                        │
│ Radius: radius/pill                │
│ Padding: 0 12                      │
└────────────────────────────────────┘
```

### 8.3 Export Settings

| Asset Type    | Format  | Scale      | Naming                     |
| ------------- | ------- | ---------- | -------------------------- |
| Icons         | SVG     | 1x         | `icon-[name].svg`          |
| Logos         | SVG+PNG | 1x, 2x     | `logo-[variant].svg`       |
| Illustrations | SVG+PNG | 1x, 2x, 3x | `illus-[name]@[scale].png` |
| Screenshots   | PNG     | 2x         | `screen-[name]-[bp].png`   |

### 8.4 CSS Variable Mapping

Include a reference sheet mapping Figma variable names to CSS custom properties:

| Figma Variable       | CSS Custom Property       |
| -------------------- | ------------------------- |
| `color/primary/500`  | `--color-primary`         |
| `color/text/primary` | `--color-text`            |
| `color/bg/surface`   | `--color-surface`         |
| `space/4`            | `--space-4` (16px)        |
| `radius/md`          | `--radius-md` (8px)       |
| `font/size/body1`    | `--font-size-base` (14px) |

### 8.5 Angular Material Class Mapping

| Figma Component | Angular Material Directive/Class          |
| --------------- | ----------------------------------------- |
| Button/Filled   | `mat-flat-button`                         |
| Button/Outlined | `mat-stroked-button`                      |
| Button/Elevated | `mat-raised-button`                       |
| Button/Text     | `mat-button`                              |
| TextInput       | `mat-form-field` + `appearance="outline"` |
| Select          | `mat-select`                              |
| Checkbox        | `mat-checkbox`                            |
| Radio           | `mat-radio-button`                        |
| DatePicker      | `mat-datepicker`                          |
| Toggle          | `mat-slide-toggle`                        |
| Chip            | `mat-chip`                                |
| Tooltip         | `matTooltip`                              |
| ProgressBar     | `mat-progress-bar`                        |
| Spinner         | `mat-spinner`                             |
| Dialog          | `MatDialog` service                       |
| Tab             | `mat-tab-group` / `mat-tab-link`          |

---

## 9. Version Control

### 9.1 Branch Naming

Figma branching follows this convention:

```
[type]/[ticket-id]-[description]
```

**Types:**

- `feature/` -- New screens or components
- `fix/` -- Design corrections
- `update/` -- Modifications to existing designs
- `explore/` -- Exploratory / experimental designs

**Examples:**

```
feature/WEB-100-client-creation-wizard
fix/WEB-201-button-spacing-correction
update/WEB-302-dark-mode-refinement
explore/new-dashboard-layout
```

### 9.2 Version History Practices

| Action                  | When                                     |
| ----------------------- | ---------------------------------------- |
| Save to version history | After completing each screen             |
| Name the version        | Descriptive: "Added Client List desktop" |
| Create branch           | Before major changes or experiments      |
| Merge to main           | After review and approval                |
| Archive old branches    | After merge, prefix with `archive/`      |

### 9.3 Review Checklist

Before merging a design branch, verify:

- [ ] All screens have Desktop, Tablet, and Mobile variants
- [ ] All components use design tokens (variables), not hard-coded values
- [ ] Dark mode variant works correctly (switch variable mode)
- [ ] All interactive elements have hover, focus, and disabled states
- [ ] Spacing annotations are present on key elements
- [ ] Typography uses text styles, not custom overrides
- [ ] Component naming follows BEM convention
- [ ] Prototype flows are functional
- [ ] Handoff annotations are complete

---

## Appendix: Quick Setup Checklist

When starting a new Figma file for M-SACCO:

1. **Create variable collections** (Colors, Spacing, Radius, Typography) per Section 4
2. **Set up color modes** (Light and Dark) on the Colors collection
3. **Create text styles** matching the type scale in DESIGN-001
4. **Create color styles** for all semantic and status colors
5. **Build base components first** (Button, Input, Select) with all variants
6. **Build custom components** using base components as building blocks
7. **Create layout pattern frames** (List, Detail, Wizard, Dialog) as templates
8. **Design screens** by duplicating pattern frames and populating with real content
9. **Add responsive variants** (Tablet, Mobile) for each screen
10. **Wire prototype flows** for key user journeys
11. **Add handoff annotations** to all final screens
12. **Save version** and create branch for review
