# TECH-008: Short Codes Module Specification

## Overview

This specification covers a new Short Codes module at `src/app/short-codes/` for managing SMS/USSD short codes within the M-SACCO system. The module provides full CRUD operations for short code entities, following the same standalone component patterns and Angular Material conventions used throughout the application.

---

## 1. Module Architecture

### 1.1 File Structure

```
src/app/short-codes/
  ├── short-codes.module.ts
  ├── short-codes-routing.module.ts
  ├── short-codes.service.ts
  ├── models/
  │   └── short-code.model.ts
  ├── short-codes-list/
  │   ├── short-codes-list.component.ts
  │   ├── short-codes-list.component.html
  │   └── short-codes-list.component.scss
  ├── create-short-code/
  │   ├── create-short-code.component.ts
  │   ├── create-short-code.component.html
  │   └── create-short-code.component.scss
  ├── view-short-code/
  │   ├── view-short-code.component.ts
  │   ├── view-short-code.component.html
  │   └── view-short-code.component.scss
  └── edit-short-code/
      ├── edit-short-code.component.ts
      ├── edit-short-code.component.html
      └── edit-short-code.component.scss
```

### 1.2 Routing

```
/short-codes
  /                    -> ShortCodesListComponent
  /create              -> CreateShortCodeComponent
  /:shortCodeId        -> ViewShortCodeComponent
  /:shortCodeId/edit   -> EditShortCodeComponent
```

---

## 2. Data Model

```typescript
// models/short-code.model.ts

export interface ShortCode {
  id: number;
  shortCode: string;
  name: string;
  description?: string;
  isActive: boolean;
  createdDate: string;
  lastModifiedDate?: string;
}

export interface ShortCodePayload {
  shortCode: string;
  name: string;
  description?: string;
  isActive: boolean;
}
```

---

## 3. Module and Routing

```typescript
// short-codes.module.ts

import { NgModule } from '@angular/core';
import { ShortCodesRoutingModule } from './short-codes-routing.module';
import { ShortCodesService } from './short-codes.service';

@NgModule({
  imports: [ShortCodesRoutingModule],
  providers: [ShortCodesService]
})
export class ShortCodesModule {}
```

```typescript
// short-codes-routing.module.ts

import { NgModule } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { ShortCodesListComponent } from './short-codes-list/short-codes-list.component';
import { CreateShortCodeComponent } from './create-short-code/create-short-code.component';
import { ViewShortCodeComponent } from './view-short-code/view-short-code.component';
import { EditShortCodeComponent } from './edit-short-code/edit-short-code.component';
import { ShortCodeResolver } from './resolvers/short-code.resolver';

const routes: Routes = [
  {
    path: '',
    children: [
      {
        path: '',
        component: ShortCodesListComponent,
        data: { title: 'Short Codes', breadcrumb: 'Short Codes' }
      },
      {
        path: 'create',
        component: CreateShortCodeComponent,
        data: { title: 'Create Short Code', breadcrumb: 'Create' }
      },
      {
        path: ':shortCodeId',
        component: ViewShortCodeComponent,
        resolve: { shortCode: ShortCodeResolver },
        data: { title: 'View Short Code', breadcrumb: 'View' }
      },
      {
        path: ':shortCodeId/edit',
        component: EditShortCodeComponent,
        resolve: { shortCode: ShortCodeResolver },
        data: { title: 'Edit Short Code', breadcrumb: 'Edit' }
      }
    ]
  }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class ShortCodesRoutingModule {}
```

### Resolver

```typescript
// resolvers/short-code.resolver.ts

import { Injectable, inject } from '@angular/core';
import { Resolve, ActivatedRouteSnapshot } from '@angular/router';
import { Observable } from 'rxjs';
import { ShortCodesService } from '../short-codes.service';
import { ShortCode } from '../models/short-code.model';

@Injectable({ providedIn: 'root' })
export class ShortCodeResolver implements Resolve<ShortCode> {
  private shortCodesService = inject(ShortCodesService);

  resolve(route: ActivatedRouteSnapshot): Observable<ShortCode> {
    const shortCodeId = +route.paramMap.get('shortCodeId')!;
    return this.shortCodesService.getShortCode(shortCodeId);
  }
}
```

### App Routing Integration

In `src/app/app-routing.module.ts`:

```typescript
{
  path: 'short-codes',
  loadChildren: () => import('./short-codes/short-codes.module').then(m => m.ShortCodesModule),
  data: { title: 'Short Codes', breadcrumb: 'Short Codes' }
}
```

---

## 4. Service

```typescript
// short-codes.service.ts

import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { ShortCode, ShortCodePayload } from './models/short-code.model';

/**
 * Service for managing SMS/USSD short codes.
 *
 * API Strategy: Short codes are stored in a custom Fineract datatable
 * registered against a global entity, or via a custom API extension.
 * The base path assumes a dedicated endpoint has been provisioned.
 * If using datatables, replace '/shortcodes' with '/datatables/short_codes/1'
 * (where 1 is a fixed app-table reference row).
 */
@Injectable({ providedIn: 'root' })
export class ShortCodesService {
  private http = inject(HttpClient);
  private basePath = '/shortcodes';

  /**
   * Get all short codes.
   * GET /shortcodes
   */
  getShortCodes(): Observable<ShortCode[]> {
    return this.http.get<ShortCode[]>(this.basePath);
  }

  /**
   * Get a single short code by ID.
   * GET /shortcodes/{id}
   */
  getShortCode(id: number): Observable<ShortCode> {
    return this.http.get<ShortCode>(`${this.basePath}/${id}`);
  }

  /**
   * Create a new short code.
   * POST /shortcodes
   */
  createShortCode(payload: ShortCodePayload): Observable<ShortCode> {
    return this.http.post<ShortCode>(this.basePath, payload);
  }

  /**
   * Update an existing short code.
   * PUT /shortcodes/{id}
   */
  updateShortCode(id: number, payload: ShortCodePayload): Observable<ShortCode> {
    return this.http.put<ShortCode>(`${this.basePath}/${id}`, payload);
  }

  /**
   * Delete a short code.
   * DELETE /shortcodes/{id}
   */
  deleteShortCode(id: number): Observable<void> {
    return this.http.delete<void>(`${this.basePath}/${id}`);
  }

  /**
   * Check if a short code value is unique.
   * GET /shortcodes?shortCode={value}
   * Returns true if no existing record has this code.
   */
  checkUniqueness(shortCodeValue: string, excludeId?: number): Observable<boolean> {
    return new Observable((observer) => {
      this.getShortCodes().subscribe(
        (codes: ShortCode[]) => {
          const exists = codes.some((c) => c.shortCode === shortCodeValue && c.id !== excludeId);
          observer.next(!exists);
          observer.complete();
        },
        (error) => {
          observer.error(error);
        }
      );
    });
  }
}
```

---

## 5. Form Validation

### Custom Validator for Short Code Format

```typescript
// validators/short-code.validator.ts

import { AbstractControl, ValidationErrors, AsyncValidatorFn } from '@angular/forms';
import { Observable, of, timer } from 'rxjs';
import { map, switchMap, catchError } from 'rxjs/operators';
import { ShortCodesService } from '../short-codes.service';

/**
 * Synchronous validator: short code must be 4-6 numeric digits.
 */
export function shortCodeFormatValidator(control: AbstractControl): ValidationErrors | null {
  const value = control.value;
  if (!value) return null;

  const pattern = /^\d{4,6}$/;
  if (!pattern.test(value)) {
    return {
      shortCodeFormat: {
        message: 'Short code must be 4-6 numeric digits'
      }
    };
  }
  return null;
}

/**
 * Async validator: checks uniqueness against existing short codes.
 * Debounced by 500ms to avoid excessive API calls.
 */
export function shortCodeUniquenessValidator(service: ShortCodesService, excludeId?: number): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    if (!control.value || control.value.length < 4) {
      return of(null);
    }

    return timer(500).pipe(
      switchMap(() => service.checkUniqueness(control.value, excludeId)),
      map((isUnique) => (isUnique ? null : { shortCodeNotUnique: true })),
      catchError(() => of(null))
    );
  };
}
```

---

## 6. Page Components

### 6.1 Short Codes List

```typescript
// short-codes-list/short-codes-list.component.ts

import { Component, OnInit, ViewChild, inject } from '@angular/core';
import { Router } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { MatTableDataSource } from '@angular/material/table';
import { MatPaginator } from '@angular/material/paginator';
import { MatSort, MatSortHeader } from '@angular/material/sort';
import { MatTable, MatColumnDef, MatHeaderCellDef, MatHeaderCell, MatCellDef, MatCell, MatHeaderRowDef, MatHeaderRow, MatRowDef, MatRow } from '@angular/material/table';
import { MatIconButton } from '@angular/material/button';
import { MatSlideToggle } from '@angular/material/slide-toggle';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { ShortCodesService } from '../short-codes.service';
import { ShortCode } from '../models/short-code.model';
import { DeleteDialogComponent } from 'app/shared/delete-dialog/delete-dialog.component';
import { DateFormatPipe } from 'app/pipes/date-format.pipe';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-short-codes-list',
  templateUrl: './short-codes-list.component.html',
  styleUrls: ['./short-codes-list.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    FaIconComponent,
    MatIconButton,
    MatSlideToggle,
    MatPaginator,
    MatSort,
    MatSortHeader,
    MatTable,
    MatColumnDef,
    MatHeaderCellDef,
    MatHeaderCell,
    MatCellDef,
    MatCell,
    MatHeaderRowDef,
    MatHeaderRow,
    MatRowDef,
    MatRow,
    DateFormatPipe
  ]
})
export class ShortCodesListComponent implements OnInit {
  private shortCodesService = inject(ShortCodesService);
  private router = inject(Router);
  private dialog = inject(MatDialog);

  @ViewChild(MatPaginator) paginator: MatPaginator;
  @ViewChild(MatSort) sort: MatSort;

  shortCodes: ShortCode[] = [];
  dataSource: MatTableDataSource<ShortCode>;
  displayedColumns: string[] = [
    'shortCode',
    'name',
    'description',
    'status',
    'createdDate',
    'actions'
  ];

  ngOnInit() {
    this.loadShortCodes();
  }

  loadShortCodes() {
    this.shortCodesService.getShortCodes().subscribe((data: ShortCode[]) => {
      this.shortCodes = data;
      this.dataSource = new MatTableDataSource(data);
      this.dataSource.paginator = this.paginator;
      this.dataSource.sort = this.sort;
    });
  }

  applyFilter(event: Event) {
    const filterValue = (event.target as HTMLInputElement).value;
    this.dataSource.filter = filterValue.trim().toLowerCase();
  }

  viewShortCode(shortCode: ShortCode) {
    this.router.navigate([
      '/short-codes',
      shortCode.id
    ]);
  }

  editShortCode(shortCode: ShortCode) {
    this.router.navigate([
      '/short-codes',
      shortCode.id,
      'edit'
    ]);
  }

  deleteShortCode(shortCode: ShortCode) {
    const deleteRef = this.dialog.open(DeleteDialogComponent, {
      data: { deleteContext: `short code "${shortCode.shortCode} - ${shortCode.name}"` }
    });
    deleteRef.afterClosed().subscribe((response: any) => {
      if (response.delete) {
        this.shortCodesService.deleteShortCode(shortCode.id).subscribe(() => {
          this.loadShortCodes();
        });
      }
    });
  }
}
```

**Template: `short-codes-list.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <div class="layout-row align-center gap-5px margin-b">
        <h3 class="mat-h3 flex">{{ 'labels.heading.Short Codes' | translate }}</h3>
        <button mat-raised-button color="primary" [routerLink]="['/short-codes/create']">
          <fa-icon icon="plus" class="m-r-10"></fa-icon>
          {{ 'labels.buttons.Create Short Code' | translate }}
        </button>
      </div>

      <!-- Search filter -->
      <mat-form-field class="full-width margin-b">
        <mat-label>{{ 'labels.inputs.Search' | translate }}</mat-label>
        <input matInput (keyup)="applyFilter($event)" placeholder="{{ 'labels.inputs.Filter short codes...' | translate }}" />
        <fa-icon matSuffix icon="search"></fa-icon>
      </mat-form-field>

      <table mat-table [dataSource]="dataSource" matSort class="mat-elevation-z1" [hidden]="!shortCodes || shortCodes.length === 0">
        <ng-container matColumnDef="shortCode">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>{{ 'labels.inputs.Short Code' | translate }}</th>
          <td mat-cell *matCellDef="let sc">
            <strong>{{ sc.shortCode }}</strong>
          </td>
        </ng-container>

        <ng-container matColumnDef="name">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>{{ 'labels.inputs.name' | translate }}</th>
          <td mat-cell *matCellDef="let sc">{{ sc.name }}</td>
        </ng-container>

        <ng-container matColumnDef="description">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Description' | translate }}</th>
          <td mat-cell *matCellDef="let sc">{{ sc.description || '-' }}</td>
        </ng-container>

        <ng-container matColumnDef="status">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>{{ 'labels.inputs.Status' | translate }}</th>
          <td mat-cell *matCellDef="let sc">
            <span [class]="sc.isActive ? 'status-badge status-active' : 'status-badge status-inactive'"> {{ sc.isActive ? 'Active' : 'Inactive' }} </span>
          </td>
        </ng-container>

        <ng-container matColumnDef="createdDate">
          <th mat-header-cell *matHeaderCellDef mat-sort-header>{{ 'labels.inputs.Created Date' | translate }}</th>
          <td mat-cell *matCellDef="let sc">{{ sc.createdDate | dateFormat }}</td>
        </ng-container>

        <ng-container matColumnDef="actions">
          <th mat-header-cell *matHeaderCellDef>{{ 'labels.inputs.Actions' | translate }}</th>
          <td mat-cell *matCellDef="let sc">
            <button mat-icon-button (click)="viewShortCode(sc)" matTooltip="View">
              <fa-icon icon="eye"></fa-icon>
            </button>
            <button mat-icon-button (click)="editShortCode(sc)" matTooltip="Edit">
              <fa-icon icon="pen"></fa-icon>
            </button>
            <button mat-icon-button color="warn" (click)="deleteShortCode(sc)" matTooltip="Delete">
              <fa-icon icon="trash"></fa-icon>
            </button>
          </td>
        </ng-container>

        <tr mat-header-row *matHeaderRowDef="displayedColumns"></tr>
        <tr mat-row *matRowDef="let row; columns: displayedColumns" class="clickable-row" (click)="viewShortCode(row)"></tr>
      </table>

      <mat-paginator [pageSizeOptions]="[10, 25, 50]" showFirstLastButtons> </mat-paginator>

      @if (!shortCodes || shortCodes.length === 0) {
      <p class="no-data">{{ 'labels.text.No short codes configured' | translate }}</p>
      }
    </mat-card-content>
  </mat-card>
</div>
```

---

### 6.2 Create Short Code

```typescript
// create-short-code/create-short-code.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';
import { Router } from '@angular/router';
import { MatSlideToggle } from '@angular/material/slide-toggle';
import { CdkTextareaAutosize } from '@angular/cdk/text-field';
import { ShortCodesService } from '../short-codes.service';
import { shortCodeFormatValidator, shortCodeUniquenessValidator } from '../validators/short-code.validator';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-create-short-code',
  templateUrl: './create-short-code.component.html',
  styleUrls: ['./create-short-code.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatSlideToggle,
    CdkTextareaAutosize
  ]
})
export class CreateShortCodeComponent implements OnInit {
  private formBuilder = inject(UntypedFormBuilder);
  private shortCodesService = inject(ShortCodesService);
  private router = inject(Router);

  shortCodeForm: UntypedFormGroup;

  ngOnInit() {
    this.shortCodeForm = this.formBuilder.group({
      shortCode: [
        '',
        [
          Validators.required,
          shortCodeFormatValidator
        ],
        [shortCodeUniquenessValidator(this.shortCodesService)]
      ],
      name: [
        '',
        Validators.required
      ],
      description: [''],
      isActive: [true]
    });
  }

  submit() {
    if (!this.shortCodeForm.valid) return;

    const payload = this.shortCodeForm.value;
    this.shortCodesService.createShortCode(payload).subscribe(() => {
      this.router.navigate(['/short-codes']);
    });
  }
}
```

**Template: `create-short-code.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <h3 class="mat-h3">{{ 'labels.heading.Create Short Code' | translate }}</h3>

      <form [formGroup]="shortCodeForm" (ngSubmit)="submit()">
        <div class="layout-column">
          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Short Code' | translate }}</mat-label>
            <input matInput required formControlName="shortCode" placeholder="e.g. 4567" maxlength="6" />
            @if (shortCodeForm.controls.shortCode.hasError('required')) {
            <mat-error>
              {{ 'labels.inputs.Short Code' | translate }} {{ 'labels.commons.is' | translate }}
              <strong>{{ 'labels.commons.required' | translate }}</strong>
            </mat-error>
            } @if (shortCodeForm.controls.shortCode.hasError('shortCodeFormat')) {
            <mat-error> {{ shortCodeForm.controls.shortCode.getError('shortCodeFormat').message }} </mat-error>
            } @if (shortCodeForm.controls.shortCode.hasError('shortCodeNotUnique')) {
            <mat-error> {{ 'labels.errors.Short code already exists' | translate }} </mat-error>
            }
            <mat-hint>{{ 'labels.hints.4-6 numeric digits' | translate }}</mat-hint>
          </mat-form-field>

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.name' | translate }}</mat-label>
            <input matInput required formControlName="name" />
            @if (shortCodeForm.controls.name.hasError('required')) {
            <mat-error>
              {{ 'labels.inputs.name' | translate }} {{ 'labels.commons.is' | translate }}
              <strong>{{ 'labels.commons.required' | translate }}</strong>
            </mat-error>
            }
          </mat-form-field>

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Description' | translate }}</mat-label>
            <textarea matInput formControlName="description" cdkTextareaAutosize cdkAutosizeMinRows="3"></textarea>
          </mat-form-field>

          <mat-slide-toggle formControlName="isActive" class="margin-b"> {{ 'labels.inputs.Active' | translate }} </mat-slide-toggle>
        </div>

        <mat-card-actions class="layout-row align-center gap-5px responsive-column">
          <button type="button" mat-raised-button [routerLink]="['/short-codes']">{{ 'labels.buttons.Cancel' | translate }}</button>
          <button mat-raised-button color="primary" [disabled]="!shortCodeForm.valid || shortCodeForm.pending">{{ 'labels.buttons.Submit' | translate }}</button>
        </mat-card-actions>
      </form>
    </mat-card-content>
  </mat-card>
</div>
```

---

### 6.3 View Short Code

```typescript
// view-short-code/view-short-code.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { MatDialog } from '@angular/material/dialog';
import { FaIconComponent } from '@fortawesome/angular-fontawesome';
import { ShortCodesService } from '../short-codes.service';
import { ShortCode } from '../models/short-code.model';
import { DeleteDialogComponent } from 'app/shared/delete-dialog/delete-dialog.component';
import { DateFormatPipe } from 'app/pipes/date-format.pipe';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-view-short-code',
  templateUrl: './view-short-code.component.html',
  styleUrls: ['./view-short-code.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    FaIconComponent,
    DateFormatPipe
  ]
})
export class ViewShortCodeComponent implements OnInit {
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private dialog = inject(MatDialog);
  private shortCodesService = inject(ShortCodesService);

  shortCode: ShortCode;

  ngOnInit() {
    this.route.data.subscribe((data: { shortCode: ShortCode }) => {
      this.shortCode = data.shortCode;
    });
  }

  editShortCode() {
    this.router.navigate([
      '/short-codes',
      this.shortCode.id,
      'edit'
    ]);
  }

  deleteShortCode() {
    const deleteRef = this.dialog.open(DeleteDialogComponent, {
      data: { deleteContext: `short code "${this.shortCode.shortCode} - ${this.shortCode.name}"` }
    });
    deleteRef.afterClosed().subscribe((response: any) => {
      if (response.delete) {
        this.shortCodesService.deleteShortCode(this.shortCode.id).subscribe(() => {
          this.router.navigate(['/short-codes']);
        });
      }
    });
  }
}
```

**Template: `view-short-code.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <div class="layout-row align-center gap-5px margin-b">
        <h3 class="mat-h3 flex">{{ 'labels.heading.Short Code Details' | translate }}</h3>
        <button mat-raised-button (click)="editShortCode()">
          <fa-icon icon="pen" class="m-r-10"></fa-icon>
          {{ 'labels.buttons.Edit' | translate }}
        </button>
        <button mat-raised-button color="warn" (click)="deleteShortCode()">
          <fa-icon icon="trash" class="m-r-10"></fa-icon>
          {{ 'labels.buttons.Delete' | translate }}
        </button>
      </div>

      <div class="detail-grid">
        <div class="detail-row">
          <span class="detail-label">{{ 'labels.inputs.Short Code' | translate }}</span>
          <span class="detail-value"><strong>{{ shortCode.shortCode }}</strong></span>
        </div>

        <div class="detail-row">
          <span class="detail-label">{{ 'labels.inputs.name' | translate }}</span>
          <span class="detail-value">{{ shortCode.name }}</span>
        </div>

        <div class="detail-row">
          <span class="detail-label">{{ 'labels.inputs.Description' | translate }}</span>
          <span class="detail-value">{{ shortCode.description || '-' }}</span>
        </div>

        <div class="detail-row">
          <span class="detail-label">{{ 'labels.inputs.Status' | translate }}</span>
          <span class="detail-value">
            <span [class]="shortCode.isActive ? 'status-badge status-active' : 'status-badge status-inactive'"> {{ shortCode.isActive ? 'Active' : 'Inactive' }} </span>
          </span>
        </div>

        <div class="detail-row">
          <span class="detail-label">{{ 'labels.inputs.Created Date' | translate }}</span>
          <span class="detail-value">{{ shortCode.createdDate | dateFormat }}</span>
        </div>

        @if (shortCode.lastModifiedDate) {
        <div class="detail-row">
          <span class="detail-label">{{ 'labels.inputs.Last Modified' | translate }}</span>
          <span class="detail-value">{{ shortCode.lastModifiedDate | dateFormat }}</span>
        </div>
        }
      </div>

      <mat-card-actions>
        <button mat-raised-button [routerLink]="['/short-codes']">
          <fa-icon icon="arrow-left" class="m-r-10"></fa-icon>
          {{ 'labels.buttons.Back to List' | translate }}
        </button>
      </mat-card-actions>
    </mat-card-content>
  </mat-card>
</div>
```

---

### 6.4 Edit Short Code

```typescript
// edit-short-code/edit-short-code.component.ts

import { Component, OnInit, inject } from '@angular/core';
import { UntypedFormGroup, UntypedFormBuilder, Validators } from '@angular/forms';
import { ActivatedRoute, Router } from '@angular/router';
import { MatSlideToggle } from '@angular/material/slide-toggle';
import { CdkTextareaAutosize } from '@angular/cdk/text-field';
import { ShortCodesService } from '../short-codes.service';
import { ShortCode } from '../models/short-code.model';
import { shortCodeFormatValidator, shortCodeUniquenessValidator } from '../validators/short-code.validator';
import { STANDALONE_SHARED_IMPORTS } from 'app/standalone-shared.module';

@Component({
  selector: 'mifosx-edit-short-code',
  templateUrl: './edit-short-code.component.html',
  styleUrls: ['./edit-short-code.component.scss'],
  imports: [
    ...STANDALONE_SHARED_IMPORTS,
    MatSlideToggle,
    CdkTextareaAutosize
  ]
})
export class EditShortCodeComponent implements OnInit {
  private formBuilder = inject(UntypedFormBuilder);
  private route = inject(ActivatedRoute);
  private router = inject(Router);
  private shortCodesService = inject(ShortCodesService);

  shortCodeForm: UntypedFormGroup;
  shortCode: ShortCode;

  ngOnInit() {
    this.route.data.subscribe((data: { shortCode: ShortCode }) => {
      this.shortCode = data.shortCode;
      this.createForm();
    });
  }

  createForm() {
    this.shortCodeForm = this.formBuilder.group({
      shortCode: [
        this.shortCode.shortCode,
        [
          Validators.required,
          shortCodeFormatValidator
        ],
        [shortCodeUniquenessValidator(this.shortCodesService, this.shortCode.id)]
      ],
      name: [
        this.shortCode.name,
        Validators.required
      ],
      description: [this.shortCode.description || ''],
      isActive: [this.shortCode.isActive]
    });
  }

  submit() {
    if (!this.shortCodeForm.valid) return;

    const payload = this.shortCodeForm.value;
    this.shortCodesService.updateShortCode(this.shortCode.id, payload).subscribe(() => {
      this.router.navigate([
        '/short-codes',
        this.shortCode.id
      ]);
    });
  }
}
```

**Template: `edit-short-code.component.html`**

```html
<div class="container">
  <mat-card>
    <mat-card-content>
      <h3 class="mat-h3">{{ 'labels.heading.Edit Short Code' | translate }}</h3>

      <form [formGroup]="shortCodeForm" (ngSubmit)="submit()">
        <div class="layout-column">
          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Short Code' | translate }}</mat-label>
            <input matInput required formControlName="shortCode" placeholder="e.g. 4567" maxlength="6" />
            @if (shortCodeForm.controls.shortCode.hasError('required')) {
            <mat-error>
              {{ 'labels.inputs.Short Code' | translate }} {{ 'labels.commons.is' | translate }}
              <strong>{{ 'labels.commons.required' | translate }}</strong>
            </mat-error>
            } @if (shortCodeForm.controls.shortCode.hasError('shortCodeFormat')) {
            <mat-error> {{ shortCodeForm.controls.shortCode.getError('shortCodeFormat').message }} </mat-error>
            } @if (shortCodeForm.controls.shortCode.hasError('shortCodeNotUnique')) {
            <mat-error> {{ 'labels.errors.Short code already exists' | translate }} </mat-error>
            }
            <mat-hint>{{ 'labels.hints.4-6 numeric digits' | translate }}</mat-hint>
          </mat-form-field>

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.name' | translate }}</mat-label>
            <input matInput required formControlName="name" />
            @if (shortCodeForm.controls.name.hasError('required')) {
            <mat-error>
              {{ 'labels.inputs.name' | translate }} {{ 'labels.commons.is' | translate }}
              <strong>{{ 'labels.commons.required' | translate }}</strong>
            </mat-error>
            }
          </mat-form-field>

          <mat-form-field>
            <mat-label>{{ 'labels.inputs.Description' | translate }}</mat-label>
            <textarea matInput formControlName="description" cdkTextareaAutosize cdkAutosizeMinRows="3"></textarea>
          </mat-form-field>

          <mat-slide-toggle formControlName="isActive" class="margin-b"> {{ 'labels.inputs.Active' | translate }} </mat-slide-toggle>
        </div>

        <mat-card-actions class="layout-row align-center gap-5px responsive-column">
          <button type="button" mat-raised-button [routerLink]="['/short-codes', shortCode.id]">{{ 'labels.buttons.Cancel' | translate }}</button>
          <button mat-raised-button color="primary" [disabled]="!shortCodeForm.valid || shortCodeForm.pending || shortCodeForm.pristine">{{ 'labels.buttons.Submit' | translate }}</button>
        </mat-card-actions>
      </form>
    </mat-card-content>
  </mat-card>
</div>
```

---

## 7. Styles

### Status Badge Styles

Add to a shared styles file or each component's scss:

```scss
.status-badge {
  display: inline-block;
  padding: 4px 12px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 500;

  &.status-active {
    background: #e8f5e9;
    color: #2e7d32;
  }

  &.status-inactive {
    background: #fce4ec;
    color: #c62828;
  }
}

.detail-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 16px;
  margin: 16px 0;
}

.detail-row {
  display: flex;
  gap: 16px;
  padding: 8px 0;
  border-bottom: 1px solid #f0f0f0;
}

.detail-label {
  min-width: 160px;
  color: #666;
  font-weight: 500;
}

.detail-value {
  flex: 1;
}

.clickable-row {
  cursor: pointer;
  &:hover {
    background: #f5f5f5;
  }
}

.full-width {
  width: 100%;
}
```

---

## 8. Navigation Integration

Add to the sidebar navigation configuration:

```typescript
{
  name: 'Short Codes',
  icon: 'hashtag',
  route: '/short-codes'
}
```

This could sit under a "System" or "Configuration" parent menu group depending on the navigation hierarchy.

---

## 9. API Endpoint Strategy

The short codes feature requires a backend endpoint. Two implementation approaches:

### Option A: Custom Fineract Datatable (No Backend Changes)

Register a datatable `short_codes` against the `m_appuser` table (or a standalone table):

```
POST /datatables
{
  "datatableName": "short_codes",
  "apptableName": "m_appuser",
  "columns": [
    { "name": "short_code", "type": "String", "length": 6, "mandatory": true, "unique": true },
    { "name": "name", "type": "String", "length": 100, "mandatory": true },
    { "name": "description", "type": "Text", "mandatory": false },
    { "name": "is_active", "type": "Boolean", "mandatory": true }
  ]
}
```

Then use the datatables CRUD API:

- `GET /datatables/short_codes` -- list
- `POST /datatables/short_codes/1` -- create (appTableId=1)
- `PUT /datatables/short_codes/1/{rowId}` -- update
- `DELETE /datatables/short_codes/1/{rowId}` -- delete

### Option B: Custom Fineract Extension (Recommended for Production)

Create a dedicated REST endpoint at `/fineract-provider/api/v1/shortcodes` via a Fineract module extension. This provides proper validation, indexing, and audit logging.

The `ShortCodesService` base path should be updated accordingly based on which approach is chosen.
