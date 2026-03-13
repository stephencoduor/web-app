# DESIGN-001: Design Foundations

> M-SACCO Design System -- Visual Foundation Tokens and Scales
>
> **Stack:** Angular Material 20.2.14, Tailwind CSS 3.3.3, FontAwesome 6.7.2
> **Theme file:** `src/theme/mifosx-theme.scss`
> **Palette file:** `src/theme/_material-palette.scss`

---

## 1. Color System

### 1.1 Primary Palette -- Blue (#1074b9)

| Token         | Hex       | Usage                                            |
| ------------- | --------- | ------------------------------------------------ |
| `primary-50`  | `#e3f0fa` | Hover tint on primary surfaces                   |
| `primary-100` | `#5ba2ec` | Light variant (from `$primary-palette 100`)      |
| `primary-200` | `#4a93dd` | Secondary highlights                             |
| `primary-300` | `#3984ce` | Active state backgrounds                         |
| `primary-400` | `#2478c4` | Focused borders                                  |
| `primary-500` | `#1074b9` | **Default primary** -- buttons, links, header bg |
| `primary-600` | `#0e66a5` | Hovered primary buttons                          |
| `primary-700` | `#004989` | Dark variant (from `$primary-palette 700`)       |
| `primary-800` | `#003a6e` | Pressed state                                    |
| `primary-900` | `#002b54` | High-emphasis text on light backgrounds          |

**Contrast:** 100 uses `rgba(0,0,0,0.87)`, 500 and 700 use `white`.

### 1.2 Accent Palette -- Green (#b4d575)

| Token        | Hex       | Usage                                      |
| ------------ | --------- | ------------------------------------------ |
| `accent-50`  | `#f4fce3` | Subtle success background                  |
| `accent-100` | `#e7ffa5` | Light variant (from `$accent-palette 100`) |
| `accent-200` | `#d5f08a` | Hover tint on accent elements              |
| `accent-300` | `#c5e580` | Tags, badges                               |
| `accent-400` | `#bcdd7a` | Accent icon color                          |
| `accent-500` | `#b4d575` | **Default accent** -- FABs, toggle active  |
| `accent-600` | `#9ec35e` | Hovered accent buttons                     |
| `accent-700` | `#83a447` | Dark variant (from `$accent-palette 700`)  |
| `accent-800` | `#6b8739` | Pressed state                              |
| `accent-900` | `#536a2c` | High-emphasis accent text                  |

**Contrast:** All levels use `rgba(0,0,0,0.87)`.

### 1.3 Warn Palette -- Material Red

Uses the built-in `mat.$m2-red-palette` from Angular Material.

| Token      | Hex       | Usage                            |
| ---------- | --------- | -------------------------------- |
| `warn-50`  | `#ffebee` | Error background tint            |
| `warn-100` | `#ffcdd2` | Light error highlight            |
| `warn-500` | `#f44336` | Default warn -- error states     |
| `warn-700` | `#d32f2f` | Dark warn -- destructive buttons |
| `warn-900` | `#b71c1c` | Critical alerts                  |

### 1.4 Neutral Palette

| Token                  | Hex       | Usage                                  |
| ---------------------- | --------- | -------------------------------------- |
| `neutral-text`         | `#212529` | Primary body text                      |
| `neutral-secondary`    | `#6c757d` | Secondary / helper text                |
| `neutral-disabled`     | `#adb5bd` | Disabled text, placeholder             |
| `neutral-border`       | `#e9ecef` | Dividers, card borders, input outlines |
| `neutral-table-header` | `#f5f5f5` | Table header row background            |
| `neutral-page-bg`      | `#f8f9fa` | Page background                        |
| `neutral-surface`      | `#ffffff` | Cards, dialogs, input backgrounds      |

### 1.5 Semantic Colors

| Token     | Hex       | Usage                                 |
| --------- | --------- | ------------------------------------- |
| `success` | `#28a745` | Success messages, positive indicators |
| `warning` | `#ffc107` | Warnings, caution states              |
| `error`   | `#dc3545` | Error messages, validation failures   |
| `info`    | `#17a2b8` | Informational alerts, help text       |

### 1.6 Entity Status Colors

| Status    | Color   | Hex       | Background Tint |
| --------- | ------- | --------- | --------------- |
| Active    | Success | `#28a745` | `#e8f5e9`       |
| Pending   | Warning | `#ffc107` | `#fff8e1`       |
| Rejected  | Error   | `#dc3545` | `#ffebee`       |
| Closed    | Gray    | `#6c757d` | `#f5f5f5`       |
| Overdue   | Error   | `#dc3545` | `#ffebee`       |
| Draft     | Muted   | `#adb5bd` | `#f8f9fa`       |
| Submitted | Info    | `#17a2b8` | `#e0f7fa`       |
| Approved  | Success | `#28a745` | `#e8f5e9`       |
| Disbursed | Primary | `#1074b9` | `#e3f0fa`       |

### 1.7 Dark Mode Variants

Dark mode is toggled via the `.dark-theme` CSS class on the `<body>` element. The dark theme uses the same primary/accent hue values with different contrast mappings.

| Token                    | Light Value        | Dark Value                |
| ------------------------ | ------------------ | ------------------------- |
| Page background          | `#f8f9fa`          | `#121212`                 |
| Surface (cards, dialogs) | `#ffffff`          | `#1e1e1e`                 |
| Elevated surface         | `#ffffff`          | `#2d2d2d`                 |
| Navigation tabs bg       | `#f2f2f2`          | `#303135`                 |
| Table even row           | `#f8f9fa`          | `#303135`                 |
| Primary text             | `#212529`          | `rgba(255,255,255,0.87)`  |
| Secondary text           | `#6c757d`          | `#aaa`                    |
| Disabled text            | `#adb5bd`          | `#666`                    |
| Border / divider         | `#e9ecef`          | `#444`                    |
| Link color               | `#1074b9`          | `#0098ff`                 |
| Primary contrast (500)   | `white`            | `rgba(255,255,255,0.87)`  |
| Accent contrast (all)    | `rgba(0,0,0,0.87)` | `white`                   |
| Hover on table row       | `rgba(0,0,0,0.04)` | `rgba(255,255,255,0.08)`  |
| Drag preview bg          | `white`            | `white` (text: `#303135`) |
| Business date footer     | inherited          | `#0098ff`                 |

**Dark palette definitions** (from `_material-palette.scss`):

```scss
$dark-primary-palette: (
  100: #5ba2ec,
  500: #1074b9,
  700: #004989,
  contrast: (
    100: white,
    500: rgba(255, 255, 255, 0.87),
    700: rgba(255, 255, 255, 0.87)
  )
);

$dark-accent-palette: (
  100: #e7ffa5,
  500: #b4d575,
  700: #83a447,
  contrast: (
    100: white,
    500: white,
    700: white
  )
);
```

---

## 2. Typography

**Font family:** `Roboto, "Helvetica Neue", sans-serif`

The theme defines a single base typography level applied uniformly across Angular Material typographic slots (`body-1`, `body-2`, `headline-1` through `headline-6`, `button`):

```scss
$mifosx-typography-level: mat.m2-define-typography-level(
  $font-family: Roboto,
  $font-weight: 400,
  $font-size: 14px,
  $line-height: 1.5,
  $letter-spacing: normal
);
```

### Recommended Type Scale

| Token      | Size | Weight | Line Height | Letter Spacing | Usage                                 |
| ---------- | ---- | ------ | ----------- | -------------- | ------------------------------------- |
| `display`  | 32px | 700    | 1.2         | -0.5px         | Dashboard headers only                |
| `h1`       | 24px | 600    | 1.2         | normal         | Page titles                           |
| `h2`       | 20px | 500    | 1.2         | normal         | Section headers                       |
| `h3`       | 16px | 500    | 1.3         | normal         | Card titles, tab labels               |
| `body-1`   | 14px | 400    | 1.5         | normal         | **Default** -- body text, table cells |
| `body-2`   | 12px | 400    | 1.5         | 0.1px          | Captions, helper text, timestamps     |
| `button`   | 14px | 500    | 1.5         | 0.5px          | Button labels, uppercase              |
| `caption`  | 11px | 400    | 1.4         | 0.2px          | Footnotes, fine print                 |
| `overline` | 10px | 500    | 1.4         | 1.5px          | Labels above form fields, uppercase   |

### Heading Colors

| Context       | Light Mode | Dark Mode                |
| ------------- | ---------- | ------------------------ |
| h2, h3, h4    | `#212529`  | `rgba(255,255,255,0.87)` |
| Account title | `white`    | `white` (on primary bg)  |
| Sub-heading   | `#6c757d`  | `#aaa`                   |

---

## 3. Spacing

**Base unit:** 4px

### Spacing Scale

| Token    | Value | Usage                                         |
| -------- | ----- | --------------------------------------------- |
| `sp-0`   | 0px   | No spacing                                    |
| `sp-0.5` | 2px   | Hairline gaps (icon-to-text micro adjustment) |
| `sp-1`   | 4px   | Minimum internal padding                      |
| `sp-2`   | 8px   | Tight component internal padding              |
| `sp-3`   | 12px  | Mobile page margins                           |
| `sp-4`   | 16px  | Tablet page margins, standard card padding    |
| `sp-5`   | 20px  | Section dividers                              |
| `sp-6`   | 24px  | Desktop page margins, card padding (large)    |
| `sp-8`   | 32px  | Section spacing                               |
| `sp-10`  | 40px  | Page section separation                       |
| `sp-12`  | 48px  | Large component heights (nav bar, pagination) |
| `sp-16`  | 64px  | Nav bar height (top bar)                      |
| `sp-20`  | 80px  | Hero / banner areas                           |
| `sp-24`  | 96px  | Major layout gaps                             |

### Component Spacing Rules

| Component           | Padding                       | Gap between items     |
| ------------------- | ----------------------------- | --------------------- |
| Button (sm)         | 4px 12px                      | --                    |
| Button (md)         | 8px 16px                      | --                    |
| Button (lg)         | 12px 24px                     | --                    |
| Card                | 16px (standard), 24px (large) | --                    |
| Form field group    | --                            | 16px vertical         |
| Form field (inline) | --                            | 12px horizontal       |
| Table cell          | 6px 8px                       | --                    |
| Table header cell   | 6px 8px                       | --                    |
| Dialog content      | 24px                          | 16px between sections |
| Dialog actions      | 8px 16px                      | 8px between buttons   |
| Toolbar             | 0 16px                        | 8px between items     |
| Breadcrumb          | 4px 0                         | 4px between segments  |
| Tab bar             | 0 16px per tab                | 0                     |

### Page Margins

| Breakpoint | Margin |
| ---------- | ------ |
| Desktop    | 24px   |
| Tablet     | 16px   |
| Mobile     | 12px   |

---

## 4. Elevation (Shadows)

Angular Material elevation classes are included globally via `@include mat.elevation-classes()`.

| Level | CSS `box-shadow`                                             | Usage                              |
| ----- | ------------------------------------------------------------ | ---------------------------------- |
| 0     | none                                                         | Flat elements, inline content      |
| 1     | `0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24)`     | Cards, raised buttons              |
| 2     | `0 3px 6px rgba(0,0,0,0.16), 0 3px 6px rgba(0,0,0,0.23)`     | Dropdowns, floating action buttons |
| 3     | `0 10px 20px rgba(0,0,0,0.19), 0 6px 6px rgba(0,0,0,0.23)`   | Dialogs, modals                    |
| 4     | `0 14px 28px rgba(0,0,0,0.25), 0 10px 10px rgba(0,0,0,0.22)` | Popovers, tooltips                 |

### Dark Mode Shadow Adjustments

In dark mode, shadows are more pronounced to maintain perceived depth:

| Level | Adjustment                                                                                          |
| ----- | --------------------------------------------------------------------------------------------------- |
| 1     | `0 1px 4px rgba(0,0,0,0.30)` (document card in dark theme)                                          |
| 2     | `0 4px 12px rgba(0,0,0,0.50)` (document card hover in dark)                                         |
| 3     | `0 2px 4px rgba(0,0,0,0.30)` / `0 4px 8px rgba(0,0,0,0.50)` hover (note card in dark)               |
| drag  | `0 5px 5px -3px rgba(0,0,0,0.20), 0 8px 10px 1px rgba(0,0,0,0.14), 0 3px 14px 2px rgba(0,0,0,0.12)` |

---

## 5. Border Radius

| Token         | Value  | Usage                                   |
| ------------- | ------ | --------------------------------------- |
| `radius-none` | 0px    | Tables, square elements                 |
| `radius-xs`   | 2px    | Subtle rounding on small elements       |
| `radius-sm`   | 4px    | Buttons (filled), inputs, drag previews |
| `radius-md`   | 8px    | Cards, containers                       |
| `radius-lg`   | 12px   | Dialogs, modals                         |
| `radius-xl`   | 16px   | Large panels                            |
| `radius-2xl`  | 20px   | Profile images, M3-style buttons        |
| `radius-pill` | 9999px | Badges, chips, pills                    |

### Component Radius Mapping

| Component     | Radius |
| ------------- | ------ |
| Filled button | 4px    |
| M3 button     | 20px   |
| Text input    | 4px    |
| Card          | 8px    |
| Dialog        | 12px   |
| Badge / Chip  | 9999px |
| Profile image | 20px   |
| Drag preview  | 4px    |

---

## 6. Icons -- FontAwesome 6.7.2 (Solid)

### Size Scale

| Token      | Size | Usage                             |
| ---------- | ---- | --------------------------------- |
| `icon-xs`  | 12px | Inline status indicators          |
| `icon-sm`  | 14px | Button icon prefix, table actions |
| `icon-md`  | 16px | **Default** -- most UI icons      |
| `icon-lg`  | 20px | Card header icons, nav icons      |
| `icon-xl`  | 24px | Page header icons, empty states   |
| `icon-2xl` | 32px | Dashboard feature icons           |

### Icon Inventory by Category

**Navigation (12 icons):**
`fa-home`, `fa-bars`, `fa-chevron-left`, `fa-chevron-right`, `fa-chevron-down`, `fa-chevron-up`, `fa-arrow-left`, `fa-arrow-right`, `fa-angle-double-left`, `fa-angle-double-right`, `fa-external-link-alt`, `fa-ellipsis-v`

**Actions (18 icons):**
`fa-plus`, `fa-plus-circle`, `fa-edit`, `fa-pencil-alt`, `fa-trash`, `fa-trash-alt`, `fa-save`, `fa-times`, `fa-times-circle`, `fa-check`, `fa-check-circle`, `fa-search`, `fa-filter`, `fa-sort`, `fa-sort-up`, `fa-sort-down`, `fa-download`, `fa-upload`

**Entities (20 icons):**
`fa-user`, `fa-users`, `fa-user-plus`, `fa-user-edit`, `fa-user-minus`, `fa-user-tie`, `fa-user-shield`, `fa-building`, `fa-landmark`, `fa-university`, `fa-money-bill`, `fa-money-bill-wave`, `fa-coins`, `fa-wallet`, `fa-credit-card`, `fa-hand-holding-usd`, `fa-file-invoice-dollar`, `fa-chart-line`, `fa-chart-bar`, `fa-chart-pie`

**Status (12 icons):**
`fa-check-circle`, `fa-exclamation-circle`, `fa-exclamation-triangle`, `fa-info-circle`, `fa-ban`, `fa-clock`, `fa-hourglass-half`, `fa-spinner`, `fa-sync`, `fa-redo`, `fa-lock`, `fa-unlock`

**Documents & Files (10 icons):**
`fa-file`, `fa-file-alt`, `fa-file-pdf`, `fa-file-excel`, `fa-file-csv`, `fa-file-image`, `fa-file-upload`, `fa-file-download`, `fa-folder`, `fa-folder-open`

**Communication (8 icons):**
`fa-envelope`, `fa-phone`, `fa-sms`, `fa-comment`, `fa-comments`, `fa-bell`, `fa-bell-slash`, `fa-paper-plane`

**Finance & Accounting (12 icons):**
`fa-calculator`, `fa-receipt`, `fa-balance-scale`, `fa-piggy-bank`, `fa-percentage`, `fa-exchange-alt`, `fa-calendar-alt`, `fa-calendar-check`, `fa-history`, `fa-undo`, `fa-clipboard-list`, `fa-tasks`

**Settings & System (10 icons):**
`fa-cog`, `fa-cogs`, `fa-sliders-h`, `fa-wrench`, `fa-key`, `fa-shield-alt`, `fa-globe`, `fa-language`, `fa-moon`, `fa-sun`

**Misc (8 icons):**
`fa-eye`, `fa-eye-slash`, `fa-copy`, `fa-paste`, `fa-grip-vertical`, `fa-expand`, `fa-compress`, `fa-question-circle`

### Dark Mode Icon Behavior

In dark mode, `fa-icon`, `mat-icon`, and `mat-checkbox` inherit color from their parent via `color: inherit`. Specific overrides:

- `.img-button`, `.app-user-photo`, `.profile-image` apply `filter: invert(100%)` to flip logos/images.

---

## 7. Breakpoints

| Token | Range          | Label            | Columns | Gutter | Margin |
| ----- | -------------- | ---------------- | ------- | ------ | ------ |
| `xs`  | 0 -- 599px     | Mobile           | 4       | 16px   | 12px   |
| `sm`  | 600 -- 959px   | Tablet portrait  | 8       | 16px   | 16px   |
| `md`  | 960 -- 1279px  | Tablet landscape | 12      | 24px   | 24px   |
| `lg`  | 1280 -- 1919px | Desktop          | 12      | 24px   | 24px   |
| `xl`  | 1920px+        | Large desktop    | 12      | 24px   | 24px   |

### Responsive Rules

| Behavior              | xs                        | sm                 | md             | lg         | xl         |
| --------------------- | ------------------------- | ------------------ | -------------- | ---------- | ---------- |
| Sidebar navigation    | Hidden (hamburger)        | Hidden (hamburger) | Collapsed rail | Expanded   | Expanded   |
| Data table            | Card list                 | Card list          | Full table     | Full table | Full table |
| Form columns          | 1                         | 1                  | 2              | 2          | 2          |
| Dialog width          | 100vw                     | 90vw               | 600px          | 600px      | 600px      |
| Wizard step indicator | Hidden (step X of Y text) | Horizontal         | Horizontal     | Horizontal | Horizontal |
| Max content width     | 100%                      | 100%               | 1280px         | 1400px     | 1400px     |
| Account card width    | 100%                      | 90%                | 90%            | 90%        | 90rem      |

### Tailwind Breakpoint Configuration

These breakpoints align with the default Tailwind CSS 3.3.3 configuration with custom overrides:

```
screens: {
  'xs': '0px',
  'sm': '600px',
  'md': '960px',
  'lg': '1280px',
  'xl': '1920px',
}
```

---

## Appendix: Token Quick Reference

```
/* Colors */
--color-primary:       #1074b9;
--color-primary-light:  #5ba2ec;
--color-primary-dark:   #004989;
--color-accent:        #b4d575;
--color-accent-light:   #e7ffa5;
--color-accent-dark:    #83a447;
--color-warn:          #f44336;
--color-success:       #28a745;
--color-warning:       #ffc107;
--color-error:         #dc3545;
--color-info:          #17a2b8;

/* Neutrals */
--color-text:          #212529;
--color-text-secondary: #6c757d;
--color-disabled:      #adb5bd;
--color-border:        #e9ecef;
--color-surface:       #ffffff;
--color-background:    #f8f9fa;

/* Typography */
--font-family:         Roboto, "Helvetica Neue", sans-serif;
--font-size-base:      14px;
--line-height-base:    1.5;

/* Spacing */
--space-unit:          4px;

/* Border Radius */
--radius-sm:           4px;
--radius-md:           8px;
--radius-lg:           12px;

/* Breakpoints */
--bp-sm:               600px;
--bp-md:               960px;
--bp-lg:               1280px;
--bp-xl:               1920px;
```
