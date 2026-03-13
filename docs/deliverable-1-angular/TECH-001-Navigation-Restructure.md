# TECH-001: Navigation Restructure Specification

**Status:** Draft
**Priority:** High
**Estimated Effort:** 3-5 days
**Dependencies:** TECH-012 (spartan/ui migration — navigation components should use spartan/ui from the start)
**UI Library:** spartan/ui (Angular port of shadcn/ui) — see TECH-012 for full migration plan

---

## 1. Current Navigation Structure Analysis

The existing navigation lives in `src/app/core/shell/` and uses a **mat-sidenav-container** shell with a collapsible sidebar and a primary-colored toolbar containing mega-menu dropdowns.

### Current Toolbar (`toolbar.component.html`)

The toolbar is a `<mat-toolbar color="primary">` with id `mifosx-toolbar`. After spartan/ui migration (TECH-012), this will be replaced with a semantic `<header>` element styled with Tailwind classes. It currently contains:

| Element                  | Type                                    | Behavior                                                             |
| ------------------------ | --------------------------------------- | -------------------------------------------------------------------- |
| Hamburger button         | `mat-icon-button`                       | Toggles sidenav open/closed                                          |
| Collapse chevron         | `mat-icon-button`                       | Toggles sidenav between full and compact modes                       |
| **Institution**          | `[matMenuTriggerFor]="institutionMenu"` | Opens dropdown: Clients, Groups, Centers, Accounting, Reports, Admin |
| **Accounting**           | `[routerLink]="['/accounting']"`        | Direct link (visible `>768px`)                                       |
| **Reports**              | `[matMenuTriggerFor]="reportsMenu"`     | Dropdown: All, Clients, Loans, Savings, Funds, Accounting            |
| **Admin**                | `[matMenuTriggerFor]="adminMenu"`       | Dropdown: Users, Organization, System, Products, Templates           |
| **Configuration Wizard** | `(click)="openDialog()"`                | Opens wizard dialog                                                  |
| Search                   | `<mifosx-search-tool>`                  | Global search component                                              |
| Language selector        | `<mifosx-language-selector>`            | Language picker                                                      |
| Notifications            | `<mifosx-notifications-tray>`           | Notification bell                                                    |
| Theme toggle             | `<mifosx-theme-toggle>`                 | Dark/light mode                                                      |
| User menu                | `[matMenuTriggerFor]="applicationMenu"` | Help, Profile, Settings, Sign Out                                    |

### Current Shell (`shell.component.html`)

```
mat-sidenav-container
  mat-sidenav (sidebar-panel)
    mifosx-sidenav              <-- Full sidebar navigation tree
  mat-sidenav-content
    mifosx-toolbar              <-- Top toolbar with mega-menus
    mifosx-breadcrumb
    mifosx-content (router-outlet)
    mifosx-footer
```

The sidenav is **open by default on desktop** (`[opened]="(isHandset$ | async) === false"`) and uses `mode="side"` on desktop, `mode="over"` on handset.

### Current Toolbar Component (`toolbar.component.ts`)

Key logic to remove:

- `PopoverService` and `ConfigurationWizardService` imports and injection
- All `@ViewChild` references for popover targets (`institution`, `templateInstitution`, `appMenu`, etc.)
- `showPopover()` method
- `nextStep()` method
- `openDialog()` method (configuration wizard)
- `toggleSidenav()` and `toggleSidenavCollapse()` sidenav management
- All `ng-template` blocks for configuration wizard popovers (12 templates)

### Current Toolbar Styles (`toolbar.component.scss`)

- Primary colored background (indigo/dark) set by `color="primary"` on `mat-toolbar`
- `.tab-link` styled with white/70% opacity text, white on hover
- `.toolbar-spacer` pushes right-side items
- White text overrides for search and language selector via `::ng-deep`

---

## 2. Target M-SACCO Navigation

The M-SACCO wireframes specify a **flat horizontal navigation bar** with no dropdown menus. The navigation items are displayed as direct links in a single row.

### Target Nav Items

| Position | Label         | Route                 | Permission Guard     |
| -------- | ------------- | --------------------- | -------------------- |
| 1        | Clients       | `/clients`            | `READ_CLIENT`        |
| 2        | Groups        | `/groups`             | `READ_GROUP`         |
| 3        | Products      | `/products`           | `READ_PRODUCT`       |
| 4        | Reports       | `/reports`            | `READ_REPORT`        |
| 5        | Accounting    | `/accounting`         | `READ_JOURNALENTRY`  |
| 6        | Configuration | `/organization`       | `READ_CONFIGURATION` |
| 7        | Search        | (inline search input) | Authenticated        |

### Visual Design

- **Background:** White (`#ffffff`)
- **Text color:** Primary blue (`#1074b9`)
- **Active indicator:** 3px bottom border in primary blue on the active nav item
- **Hover state:** Light blue background (`#f0f7ff`) with transition
- **Font:** 14px, font-weight 500 (medium)
- **Height:** 48px nav bar
- **Logo:** "Msacco" logo left-aligned before nav items

---

## 3. Files to Modify

### 3.1 `src/app/core/shell/toolbar/toolbar.component.html`

**Remove:**

- Hamburger toggle button and collapse chevron button (lines 10-36)
- The entire `institutionMenu` mat-menu and its trigger (lines 39-49, 138-175)
- The `reportsMenu` mat-menu and trigger (lines 68-78, 178-197)
- The `adminMenu` mat-menu and trigger (lines 79-89, 199-215)
- Configuration Wizard link (line 90-93)
- All 12 `ng-template` blocks for configuration wizard popovers (lines 245-446)
- Theme toggle section (lines 122-124)
- Notifications tray (lines 118-120) -- move to utility bar or keep as icon

**Replace with:**

```html
<nav class="msacco-nav" id="msacco-navbar">
  <!-- Logo -->
  <a class="msacco-logo" routerLink="/home">
    <img src="assets/images/msacco-logo.svg" alt="M-Sacco" height="32" />
  </a>

  <!-- Primary Navigation Links -->
  <div class="msacco-nav-links" role="navigation" aria-label="Primary navigation">
    @for (item of navItems; track item.route) {
    <a class="msacco-nav-link" [routerLink]="[item.route]" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: item.exact }" *mifosxHasPermission="item.permissions"> {{ item.label | translate }} </a>
    }
  </div>

  <span class="nav-spacer"></span>

  <!-- Search -->
  <div class="msacco-search">
    <mifosx-search-tool tabindex="0"></mifosx-search-tool>
  </div>

  <!-- Utility Icons -->
  <div class="msacco-utility-icons">
    <mifosx-notifications-tray tabindex="0"></mifosx-notifications-tray>

    <button mat-icon-button class="user-menu-btn" [matMenuTriggerFor]="userMenu" tabindex="0">
      <img src="assets/images/user_placeholder.png" alt="User Profile" class="user-avatar" />
    </button>
  </div>

  <!-- Mobile hamburger (visible < 960px) -->
  <button mat-icon-button class="msacco-hamburger" (click)="toggleMobileMenu()" aria-label="Toggle navigation menu">
    <mat-icon>menu</mat-icon>
  </button>
</nav>

<!-- Mobile menu overlay -->
@if (mobileMenuOpen) {
<div class="msacco-mobile-menu" role="navigation" aria-label="Mobile navigation">
  @for (item of navItems; track item.route) {
  <a class="msacco-mobile-link" [routerLink]="[item.route]" routerLinkActive="active" (click)="closeMobileMenu()" *mifosxHasPermission="item.permissions"> {{ item.label | translate }} </a>
  }
</div>
}

<!-- User dropdown menu -->
<mat-menu #userMenu="matMenu" [overlapTrigger]="false">
  <button mat-menu-item [routerLink]="['/profile']">
    <mat-icon>person</mat-icon>
    <span>{{ 'labels.menus.Profile' | translate }}</span>
  </button>
  <button mat-menu-item [routerLink]="['/settings']">
    <mat-icon>settings</mat-icon>
    <span>{{ 'labels.menus.Settings' | translate }}</span>
  </button>
  <mat-divider></mat-divider>
  <button mat-menu-item (click)="logout()">
    <mat-icon>logout</mat-icon>
    <span>{{ 'labels.menus.Sign Out' | translate }}</span>
  </button>
</mat-menu>
```

### 3.2 `src/app/core/shell/toolbar/toolbar.component.ts`

**Remove:**

- `PopoverService` import and injection
- `ConfigurationWizardService` import and injection
- `ConfigurationWizardComponent` import
- All `@ViewChild` decorators for popover targets
- `showPopover()` method
- `nextStep()` method
- `openDialog()` method
- `toggleSidenav()` method
- `toggleSidenavCollapse()` method
- `@Input() sidenav` and `@Output() collapse`
- `ngAfterViewInit` popover logic
- `ngAfterContentChecked` change detection hack

**Add:**

```typescript
import { Component, OnInit, inject } from '@angular/core';
import { Router } from '@angular/router';
import { BreakpointObserver } from '@angular/cdk/layout';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

import { AuthenticationService } from '../../authentication/authentication.service';
import { DocumentationLinksService } from 'app/shared/services/documentation-links.service';

export interface NavItem {
  label: string;
  route: string;
  permissions: string | string[];
  exact: boolean;
}

@Component({
  selector: 'mifosx-toolbar',
  templateUrl: './toolbar.component.html',
  styleUrls: ['./toolbar.component.scss'],
  imports: [
    // ... reduced imports, remove MatMenuTrigger for nav dropdowns,
    // keep MatMenu only for user menu
  ]
})
export class ToolbarComponent implements OnInit {
  private router = inject(Router);
  private authenticationService = inject(AuthenticationService);
  private breakpointObserver = inject(BreakpointObserver);
  private documentationLinks = inject(DocumentationLinksService);

  mobileMenuOpen = false;

  /** Flat navigation items -- no dropdowns */
  navItems: NavItem[] = [
    { label: 'labels.menus.Clients', route: '/clients', permissions: 'READ_CLIENT', exact: false },
    { label: 'labels.menus.Groups', route: '/groups', permissions: 'READ_GROUP', exact: false },
    { label: 'labels.menus.Products', route: '/products', permissions: 'READ_PRODUCT', exact: false },
    { label: 'labels.menus.Reports', route: '/reports', permissions: 'READ_REPORT', exact: false },
    { label: 'labels.menus.Accounting', route: '/accounting', permissions: [
        'READ_JOURNALENTRY',
        'READ_GLACCOUNT'
      ], exact: false },
    { label: 'labels.menus.Configuration', route: '/organization', permissions: 'READ_CONFIGURATION', exact: false }
  ];

  /** Detect mobile breakpoint */
  isMobile$: Observable<boolean> = this.breakpointObserver.observe(['(max-width: 959px)']).pipe(map((result) => result.matches));

  ngOnInit(): void {
    // Close mobile menu on route change
    this.router.events.subscribe(() => this.closeMobileMenu());
  }

  toggleMobileMenu(): void {
    this.mobileMenuOpen = !this.mobileMenuOpen;
  }

  closeMobileMenu(): void {
    this.mobileMenuOpen = false;
  }

  logout(): void {
    this.authenticationService
      .logout()
      .pipe(
        take(1),
        catchError(() => of(void 0)),
        finalize(() => this.router.navigate(['/login'], { replaceUrl: true }))
      )
      .subscribe();
  }

  help(): void {
    this.documentationLinks.open('userManual');
  }
}
```

### 3.3 `src/app/core/shell/toolbar/toolbar.component.scss`

**Replace entirely with:**

```scss
// ==========================================================
// M-SACCO Navigation Bar Styles
// ==========================================================

$msacco-primary: #1074b9;
$msacco-primary-light: #f0f7ff;
$msacco-white: #ffffff;
$msacco-text: #333333;
$msacco-border: #e0e0e0;
$msacco-nav-height: 48px;
$msacco-mobile-breakpoint: 959px;

.msacco-nav {
  display: flex;
  align-items: center;
  height: $msacco-nav-height;
  background-color: $msacco-white;
  border-bottom: 1px solid $msacco-border;
  padding: 0 16px;
  position: sticky;
  top: 0;
  z-index: 1000;
  box-shadow: 0 1px 3px rgb(0 0 0 / 8%);
}

// Logo
.msacco-logo {
  display: flex;
  align-items: center;
  margin-right: 24px;
  text-decoration: none;

  img {
    height: 32px;
    width: auto;
  }
}

// Navigation links container
.msacco-nav-links {
  display: flex;
  align-items: center;
  height: 100%;
  gap: 0;

  @media (max-width: $msacco-mobile-breakpoint) {
    display: none;
  }
}

// Individual nav link
.msacco-nav-link {
  display: flex;
  align-items: center;
  height: 100%;
  padding: 0 16px;
  font-size: 14px;
  font-weight: 500;
  color: $msacco-text;
  text-decoration: none;
  border-bottom: 3px solid transparent;
  transition: all 0.2s ease;
  white-space: nowrap;

  &:hover {
    background-color: $msacco-primary-light;
    color: $msacco-primary;
  }

  &.active {
    color: $msacco-primary;
    border-bottom-color: $msacco-primary;
    font-weight: 600;
  }
}

// Spacer pushes utility items to the right
.nav-spacer {
  flex: 1 1 auto;
}

// Search area
.msacco-search {
  margin-right: 8px;

  ::ng-deep mifosx-search-tool {
    .mat-mdc-form-field {
      font-size: 13px;
    }
  }
}

// Utility icons area (notifications, user menu)
.msacco-utility-icons {
  display: flex;
  align-items: center;
  gap: 4px;
}

.user-menu-btn {
  .user-avatar {
    width: 32px;
    height: 32px;
    border-radius: 50%;
  }
}

// Hamburger menu button (mobile only)
.msacco-hamburger {
  display: none;

  @media (max-width: $msacco-mobile-breakpoint) {
    display: inline-flex;
  }
}

// Mobile menu overlay
.msacco-mobile-menu {
  position: fixed;
  top: $msacco-nav-height;
  left: 0;
  right: 0;
  background: $msacco-white;
  border-bottom: 1px solid $msacco-border;
  box-shadow: 0 4px 6px rgb(0 0 0 / 10%);
  z-index: 999;
  padding: 8px 0;
}

.msacco-mobile-link {
  display: block;
  padding: 12px 24px;
  font-size: 15px;
  font-weight: 500;
  color: $msacco-text;
  text-decoration: none;
  border-left: 3px solid transparent;

  &:hover {
    background-color: $msacco-primary-light;
  }

  &.active {
    color: $msacco-primary;
    border-left-color: $msacco-primary;
    background-color: $msacco-primary-light;
  }
}
```

### 3.4 `src/app/core/shell/shell.component.html`

**Change:** Hide sidenav on desktop entirely. Only show as a slide-over on mobile if the hamburger menu needs a sidebar-style nav (the flat nav replaces it on desktop).

```html
<!-- Application Shell -->
<mat-sidenav-container id="mifosx-shell-container" autosize>
  <!-- Sidenav: hidden on desktop, slide-over on mobile only -->
  <mat-sidenav #sidenav class="sidebar-panel" [attr.role]="'dialog'" mode="over" [opened]="false">
    <mifosx-sidenav [sidenavCollapsed]="false"></mifosx-sidenav>
  </mat-sidenav>

  <!-- Content -->
  <mat-sidenav-content class="sidenav">
    <!-- Toolbar (new flat nav) -->
    <mifosx-toolbar></mifosx-toolbar>
    <!-- Progress Bar -->
    @if (progressBarMode !== 'none') {
    <div>
      <div class="loading"></div>
    </div>
    }

    <!-- Breadcrumb -->
    <mifosx-breadcrumb></mifosx-breadcrumb>
    <!-- Content -->
    <mifosx-content></mifosx-content>
    <!-- Footer -->
    <mifosx-footer [styleClass]="'main-page'"></mifosx-footer>
  </mat-sidenav-content>
</mat-sidenav-container>
```

Key changes:

- Remove `[sidenav]="sidenav"` and `(collapse)` bindings from `<mifosx-toolbar>` since the toolbar no longer manages the sidenav
- Set `mode="over"` always (no longer `mode="side"` on desktop)
- Set `[opened]="false"` always (the flat nav replaces the sidebar on desktop)
- Keep the sidenav component in DOM for potential mobile hamburger use

---

## 4. Route Mapping Table

| Nav Item      | Angular Route   | Module               | Primary Component                |
| ------------- | --------------- | -------------------- | -------------------------------- |
| Clients       | `/clients`      | `ClientsModule`      | `ClientsComponent` (list view)   |
| Groups        | `/groups`       | `GroupsModule`       | `GroupsComponent` (list view)    |
| Products      | `/products`     | `ProductsModule`     | `ProductsComponent`              |
| Reports       | `/reports`      | `ReportsModule`      | `ReportsComponent` (all reports) |
| Accounting    | `/accounting`   | `AccountingModule`   | `AccountingComponent`            |
| Configuration | `/organization` | `OrganizationModule` | `OrganizationComponent`          |

### Sub-routes Accessible from Former Dropdowns

These routes were previously hidden inside mega-menu dropdowns and now need to be accessible via their parent module's internal navigation or breadcrumbs:

| Former Menu Location  | Route                 | New Access Method                                                  |
| --------------------- | --------------------- | ------------------------------------------------------------------ |
| Institution > Centers | `/centers`            | Link within Clients or Groups module, or add as nav item if needed |
| Admin > Users         | `/appusers`           | Configuration landing page link                                    |
| Admin > Organization  | `/organization`       | Direct (this IS the Configuration nav item)                        |
| Admin > System        | `/system`             | Configuration landing page link                                    |
| Admin > Templates     | `/templates`          | Configuration landing page link                                    |
| Reports > Client      | `/reports/Client`     | Reports landing page filter tabs                                   |
| Reports > Loan        | `/reports/Loan`       | Reports landing page filter tabs                                   |
| Reports > Savings     | `/reports/Savings`    | Reports landing page filter tabs                                   |
| Reports > Fund        | `/reports/Fund`       | Reports landing page filter tabs                                   |
| Reports > Accounting  | `/reports/Accounting` | Reports landing page filter tabs                                   |

---

## 5. Responsive Behavior

### Breakpoints

| Breakpoint             | Behavior                                                                                                                     |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| >= 960px (desktop)     | Full horizontal nav bar with all items visible. Sidenav hidden.                                                              |
| 600px - 959px (tablet) | Hamburger menu replaces nav links. Tapping hamburger opens vertical mobile menu overlay. Search icon collapses to icon-only. |
| < 600px (mobile)       | Same as tablet. User avatar may collapse to icon. Search moves into mobile menu.                                             |

### Desktop (>= 960px)

```
[Logo] [Clients] [Groups] [Products] [Reports] [Accounting] [Configuration]    [Search...] [Bell] [Avatar]
```

### Tablet/Mobile (< 960px)

```
[Logo]                                                                          [Search] [Bell] [Hamburger]
```

Tapping hamburger reveals:

```
------------------------------
  Clients
  Groups
  Products
  Reports
  Accounting
  Configuration
------------------------------
```

---

## 6. Branding Changes

### Logo

- Replace the current Mifos "m" icon with the M-Sacco logo
- File: `src/assets/images/msacco-logo.svg` (new asset to be provided by design team)
- Fallback: Text-based "Msacco" in primary blue, bold, 20px
- Logo links to `/home`

### Color Scheme

| Element            | Current                      | Target                                        |
| ------------------ | ---------------------------- | --------------------------------------------- |
| Toolbar background | Primary theme color (indigo) | White `#ffffff`                               |
| Nav text           | White/70% opacity            | Dark gray `#333333`                           |
| Nav active         | N/A (dropdown-based)         | Primary blue `#1074b9` with 3px bottom border |
| Nav hover          | White 100% opacity           | Light blue bg `#f0f7ff`                       |
| Overall accent     | Material indigo              | M-SACCO blue `#1074b9`                        |

### Theme Customization

Update the Angular Material theme in `src/styles/theme.scss` (or equivalent):

```scss
$msacco-primary-palette: (
  50: #e3f0f9,
  100: #b9d9f0,
  200: #8bc0e6,
  300: #5da7dc,
  400: #3a94d5,
  500: #1074b9,
  // Primary
  600: #0e66a8,
  700: #0b5592,
  800: #09457d,
  900: #052d5a,
  contrast: (
    50: #000000,
    100: #000000,
    200: #000000,
    300: #000000,
    400: #ffffff,
    500: #ffffff,
    600: #ffffff,
    700: #ffffff,
    800: #ffffff,
    900: #ffffff
  )
);
```

---

## 7. Footer Redesign

### Current Footer (`src/app/shared/footer/footer.component.html`)

The current footer displays a table with version information: App name, Mifos version, Fineract version, server URL, username, render time, and business date.

### Target Footer

The M-SACCO wireframes show a simple single-line footer:

```
Help  *  Support  *  Logout  *  (c) M-Sacco
```

### New Footer Template

```html
<footer class="msacco-footer" [ngClass]="styleClass">
  <div class="msacco-footer-content">
    <a href="javascript:void(0)" (click)="openHelp()" class="footer-link"> {{ 'labels.menus.Help' | translate }} </a>
    <span class="footer-separator">&bull;</span>
    <a href="javascript:void(0)" (click)="openSupport()" class="footer-link"> {{ 'labels.menus.Support' | translate }} </a>
    <span class="footer-separator">&bull;</span>
    <a href="javascript:void(0)" (click)="logout()" class="footer-link"> {{ 'labels.menus.Sign Out' | translate }} </a>
    <span class="footer-separator">&bull;</span>
    <span class="footer-copyright">&copy; M-Sacco {{ currentYear }}</span>
  </div>
</footer>
```

### Footer Styles

```scss
.msacco-footer {
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 12px 16px;
  border-top: 1px solid #e0e0e0;
  background-color: #fafafa;
  font-size: 13px;
  color: #666666;
}

.msacco-footer-content {
  display: flex;
  align-items: center;
  gap: 8px;
}

.footer-link {
  color: #1074b9;
  text-decoration: none;
  font-weight: 400;

  &:hover {
    text-decoration: underline;
  }
}

.footer-separator {
  color: #cccccc;
}

.footer-copyright {
  color: #999999;
}
```

### Footer Component Updates

Add to `footer.component.ts`:

```typescript
currentYear = new Date().getFullYear();

openHelp(): void {
  this.documentationLinks.open('userManual');
}

openSupport(): void {
  // Navigate to support page or open external link
  window.open('https://support.msacco.com', '_blank');
}

logout(): void {
  this.authenticationService.logout().pipe(
    take(1),
    finalize(() => this.router.navigate(['/login'], { replaceUrl: true }))
  ).subscribe();
}
```

---

## 8. Breadcrumb Updates

The existing `<mifosx-breadcrumb>` component at `src/app/core/shell/breadcrumb/` should continue to work as-is since it reads from the router data `{ breadcrumb: '...' }` configuration.

### Adjustments needed:

1. **Verify breadcrumb root:** The breadcrumb currently may show "Home" as root. Confirm it shows "Home > Clients > Client Name" pattern correctly with the new flat nav.

2. **Style update:** Match the breadcrumb to M-SACCO visual style:

```scss
.msacco-breadcrumb {
  padding: 8px 16px;
  font-size: 13px;
  color: #666666;
  background-color: #f8f9fa;
  border-bottom: 1px solid #eee;

  a {
    color: #1074b9;
    text-decoration: none;

    &:hover {
      text-decoration: underline;
    }
  }

  .separator {
    margin: 0 4px;
    color: #999;
  }

  .current {
    color: #333;
    font-weight: 500;
  }
}
```

3. **Configuration sub-pages:** Since the former Admin dropdown items (Users, Organization, System, Products, Templates) are now accessed through the "Configuration" nav item, ensure breadcrumbs read correctly:
   - Configuration > Users
   - Configuration > System > Manage Data Tables
   - Configuration > Products > Loan Products

---

## 9. Migration Checklist

- [ ] Create `msacco-logo.svg` asset in `src/assets/images/`
- [ ] Update `toolbar.component.html` with flat nav template
- [ ] Rewrite `toolbar.component.ts` to remove dropdown/popover/sidenav logic
- [ ] Replace `toolbar.component.scss` with M-SACCO nav styles
- [ ] Update `shell.component.html` to hide sidenav on desktop
- [ ] Update `shell.component.ts` to remove sidenav collapse management
- [ ] Rewrite `footer.component.html` with simple link footer
- [ ] Update `footer.component.ts` with help/support/logout methods
- [ ] Update Angular Material theme colors to M-SACCO palette
- [ ] Update breadcrumb styles
- [ ] Remove `ConfigurationWizardComponent` dialog trigger from toolbar
- [ ] Test all nav routes resolve correctly
- [ ] Test responsive behavior at 960px, 600px breakpoints
- [ ] Test permission guards on each nav item
- [ ] Verify keyboard navigation (Tab, Enter) works on all nav links
- [ ] Run full e2e test suite
