# Chapter 12 — IAM Admin Employees UI in Angular

> Suggested filename: `12-frontend-iam-admin-employees.md`

## 12.1 Goals & Scope

With IAM authentication in place (login, 2FA, password, My Security), we can now start behaving like a real **IAM team inside a bank**:

- IAM admins must be able to **view and manage employees**.
- We already have backend endpoints for:
  - Listing employees with paging/search.
  - Viewing employee details.
  - Updating basic fields (name, phone, roles).
  - Activating/deactivating employees.

In this chapter we turn those backend capabilities into a **proper IAM Admin UI** inside the IAM portal.

---

### 12.1.1 What we will build

From the perspective of an IAM administrator, we will implement:

1. **Employees list page** (`/admin/employees`)

   - A banking-style **management table** that talks to:
     - `GET /api/iam/admin/employees?pageNumber=&pageSize=&search=&includeInactive=`
   - Features:
     - Search by email or full name.
     - Toggle to include inactive employees.
     - Paging controls wired to the backend `PagedResult<T>` model.
     - Status chips for:
       - Active vs inactive.
       - Email confirmed vs not.
       - 2FA enabled vs disabled.
     - Row click → navigates to employee details.

2. **Employee details / edit page** (`/admin/employees/:id`)

   - Uses:
     - `GET /api/iam/admin/employee/{id}`
     - `PUT /api/iam/admin/employees/{id}`
     - `POST /api/iam/admin/employees/{id}/activate`
     - `POST /api/iam/admin/employees/{id}/deactivate`
   - Features:
     - View and edit:
       - Full name.
       - Phone number.
       - Roles (multi-select: Employee, IamAdmin, SuperAdmin).
     - View-only fields:
       - Email.
       - Email confirmed status.
       - IsActive.
       - TwoFactorEnabled.
       - Last login (if available).
     - Actions:
       - Activate / Deactivate employee.
       - Save basic profile updates.
     - Clear messaging when changes succeed or fail.

3. **Supporting frontend infrastructure**

   - A dedicated **Admin Employees API service**, e.g.:

     - `apps/iam-portal/src/app/core/iam-admin/services/admin-employees-api.service.ts`

   - Models (one per file) for:
     - `EmployeeSummaryModel` (grid items).
     - `EmployeeDetailsModel` (details/edit page).
     - `PagedResultModel<T>` for frontend paging.
   - A small **EmployeesStore** or **EmployeesStateService** (local to IAM portal) to keep:
     - Current list query.
     - Current page of employees.
     - Loading/error flags.

   Everything is generated and structured like a real bank frontend:

   - **Nx generators** for services and components.
   - One file per TypeScript class/interface.
   - Clean folder organization under `apps/iam-portal/src/app/features/admin/employees/...` and `core/iam-admin/...`.

---

### 12.1.2 What we will **not** implement yet

To keep this chapter focused and realistic, we intentionally **defer**:

- **Creating new employees** from the UI:
  - We will assume employees are created by:
    - Another system, or
    - A separate “Create Employee” flow covered later.
- **Bulk operations**:
  - No bulk activate/deactivate yet.
  - No bulk role updates.
- **Advanced data/state management libraries**:
  - No NgRx / Akita / SignalStore yet.
  - We keep state local to an `EmployeesStateService` to avoid over-engineering too early.
- **Cross-app shared admin libraries**:
  - All code lives inside `iam-portal` for now.
  - Once we introduce other back-office apps, we can lift the admin UI into `libs/iam/admin-ui/...`.

---

### 12.1.3 How this chapter is structured

We will proceed in small, production-style steps:

1. **Branch strategy for Chapter 12** (feature branch + quality gate).
2. **Frontend admin models & API service** for employees.
3. **Employees list page**:
   - Component structure.
   - Table UI + paging + search.
4. **Employee details/edit page**:
   - Loading state.
   - Edit form + validation.
   - Activate/deactivate actions.
5. **Tests**:
   - Unit tests for the admin API service.
   - E2E tests: list + navigate to details + edit.
6. **Full frontend quality gate** (lint, tests, e2e) and merge to `develop`.

Throughout the chapter we will:

- Use **Nx generators** for services and components (no manual “new file” for Angular types).
- Keep **one model per file** under a dedicated `models/` folder.
- Reuse the **Alvor Bank design system** (tables, tags, alerts, buttons).
- Continue to behave like a **real banking team** working in a shared repo, with a disciplined branch strategy and CI expectations.

In the next section we’ll open the **Chapter 12 feature branch** and prepare the project for IAM Admin UI work.

---

## 12.2 Branch Strategy for Admin Employees UI

We treat the **IAM Admin Employees UI** as a serious feature in a real banking platform, not a quick UI tweak.

That means:

- A dedicated **feature branch** for Chapter 12.
- All work scoped to that branch under `src/frontend`.
- A clean **quality gate** (lint + unit tests + E2E tests) before merging back into `develop`.
- No direct commits to `develop` or `main`.

Our repository layout remains:

```
digital-banking-suite/
  src/
    backend/
    frontend/
      apps/
        iam-portal/
        iam-portal-e2e/
      ...
  infra/
  ...
```

All changes in this chapter live under:

- `src/frontend/apps/iam-portal/...`
- `src/frontend/apps/iam-portal-e2e/...`

Backend IAM admin endpoints are **already implemented and tested** in previous chapters; we consume them here.

---

### 12.2.1 Opening the Chapter 12 feature branch

From the repository root:

```bash
cd digital-banking-suite

# Ensure develop is up to date
git checkout develop
git pull origin develop

# Create a dedicated feature branch for IAM Admin Employees UI
git checkout -b feature/ch12-frontend-iam-admin-employees
```

We stay on this branch for all Chapter 12 work.

---

### 12.2.2 Frontend working directory

All Angular/Nx work happens under `src/frontend`:

```bash
cd digital-banking-suite/src/frontend
```

Core Nx commands you’ll use throughout this chapter:

```bash
# Serve IAM portal for local dev
npx nx serve iam-portal

# Lint
npx nx lint iam-portal

# Unit tests
npx nx test iam-portal

# E2E tests
npx nx e2e iam-portal-e2e
```

We will also use **Nx generators** to scaffold new pieces:

- Admin employees API service
- Admin employees state service (or store)
- Employees list page component
- Employee details page component

No ad-hoc Angular file creation — we keep the structure consistent with a real banking team’s standards.

---

### 12.2.3 Closing the chapter (merge ritual)

At the end of Chapter 12, once all Admin Employees UI features are implemented, we will:

1. Run the **frontend quality gate** from `src/frontend`:

   ```bash
   cd digital-banking-suite/src/frontend

   npx nx lint iam-portal
   npx nx test iam-portal
   npx nx e2e iam-portal-e2e
   ```

2. Commit and push the feature branch from the repo root:

   ```bash
   cd digital-banking-suite

   git status
   git add .
   git commit -m "feat(ch12): IAM admin employees UI (grid, details, actions)"
   git push origin feature/ch12-frontend-iam-admin-employees
   ```

3. Open a Pull Request to `develop`:

   - **Source**: `feature/ch12-frontend-iam-admin-employees`
   - **Target**: `develop`
   - Merge only after:
     - CI passes (lint, tests, E2E).
     - A peer review (or self-review if working solo) is completed.

We keep the **feature branch** for chapter-level historical reference, as with previous chapters.

Next, we will define the **frontend models and Admin Employees API service** that mirror the IAM admin endpoints and power the Employees list and details pages.

---

## 12.3 Admin Employees Models & API Service

Before we build grids and detail pages, we need a **clean frontend contract** that mirrors the IAM admin endpoints:

- Strongly typed models for:
  - `PagedResult<T>`
  - Employee list items (summaries)
  - Employee details (for the edit page)
  - Update request body
  - List query parameters
- A dedicated `AdminEmployeesApiService` that is the **only** place in the frontend that knows:
  - The URLs of IAM admin endpoints.
  - How to construct query strings for paging/search.

Everything will live under:

```
apps/iam-portal/src/app/core/iam-admin/
  models/
    paged-result.model.ts
    employee-summary.model.ts
    employee-details.model.ts
    update-employee-request.model.ts
    employee-list-query.model.ts
    index.ts
  services/
    admin-employees-api.service.ts
```

We’ll use **manual file creation** for models and an **Nx generator** for the service.

---

### 12.3.1 Create folders for IAM admin models & services

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

mkdir -p apps/iam-portal/src/app/core/iam-admin/models
mkdir -p apps/iam-portal/src/app/core/iam-admin/services
```

---

### 12.3.2 Frontend PagedResult model

Backend returns a `PagedResult<T>` with:

- Items
- PageNumber
- PageSize
- TotalCount
- TotalPages
- HasNextPage
- HasPreviousPage

In JSON (with default ASP.NET Core settings), these become camelCase properties.

File:  
`apps/iam-portal/src/app/core/iam-admin/models/paged-result.model.ts`

```ts
export interface PagedResult<T> {
  items: T[];
  pageNumber: number;
  pageSize: number;
  totalCount: number;
  totalPages: number;
  hasNextPage: boolean;
  hasPreviousPage: boolean;
}
```

We will reuse this model for any paged grid in the IAM portal.

---

### 12.3.3 Employee summary model (list items)

The list endpoint returns a collection of **summary DTOs**, which we map to a UI model:

Fields (from backend contract):

- `id: string`
- `email: string`
- `fullName: string | null`
- `emailConfirmed: boolean`
- `isActive: boolean`
- `twoFactorEnabled: boolean`
- `roles: string[]`

File:  
`apps/iam-portal/src/app/core/iam-admin/models/employee-summary.model.ts`

```ts
export interface EmployeeSummary {
  id: string;
  email: string;
  fullName: string | null;
  emailConfirmed: boolean;
  isActive: boolean;
  twoFactorEnabled: boolean;
  roles: string[];
}
```

This model will power the **Employees list grid** in `/admin/employees`.

---

### 12.3.4 Employee details model (details/edit page)

For the details/edit page we need more information:

Additional fields (from backend contract):

- `phoneNumber: string | null`
- `lockoutEnd: string | null` (ISO-8601 timestamp)
- `lastLoginAt: string | null` (ISO-8601 timestamp)

File:  
`apps/iam-portal/src/app/core/iam-admin/models/employee-details.model.ts`

```ts
import { EmployeeSummary } from "./employee-summary.model";

export interface EmployeeDetails extends EmployeeSummary {
  phoneNumber: string | null;
  lockoutEnd: string | null;
  lastLoginAt: string | null;
}
```

We use string timestamps in the model and will format them in the UI later when needed.

---

### 12.3.5 Update employee request model

The **update endpoint**:

- `PUT /api/iam/admin/employees/{id}`

Body (simplified for UI):

- `id: string`
- `fullName?: string | null`
- `phoneNumber?: string | null`
- `roles: string[]`

File:  
`apps/iam-portal/src/app/core/iam-admin/models/update-employee-request.model.ts`

```ts
export interface UpdateEmployeeRequest {
  id: string;
  fullName?: string | null;
  phoneNumber?: string | null;
  roles: string[];
}
```

We will build the edit form so that it always sends a complete set of roles.

---

### 12.3.6 Employee list query model

The list endpoint supports:

- `pageNumber`
- `pageSize`
- `search` (optional; matches email or full name)
- `includeInactive` (optional; boolean)

We keep the query parameters in a small model to make method signatures clear.

File:  
`apps/iam-portal/src/app/core/iam-admin/models/employee-list-query.model.ts`

```ts
export interface EmployeeListQuery {
  pageNumber: number;
  pageSize: number;
  search?: string | null;
  includeInactive?: boolean;
}
```

This will also be used by a small list state service in a later section.

---

### 12.3.7 Barrel file for IAM admin models (optional but convenient)

To keep imports tidy in services and components, we export all IAM admin models from one `index.ts`.

File:  
`apps/iam-portal/src/app/core/iam-admin/models/index.ts`

```ts
export * from "./paged-result.model";
export * from "./employee-summary.model";
export * from "./employee-details.model";
export * from "./update-employee-request.model";
export * from "./employee-list-query.model";
```

Now any consumer can simply:

```ts
import {
  PagedResult,
  EmployeeSummary,
  EmployeeDetails,
  UpdateEmployeeRequest,
  EmployeeListQuery,
} from "../../iam-admin/models";
```

---

### 12.3.8 Generate AdminEmployeesApiService with Nx

We now create a dedicated **Admin Employees API service** using Nx, to keep consistency with the rest of the app.

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:service \
  apps/iam-portal/src/app/core/iam-admin/services/admin-employees-api/admin-employees-api \
  --flat=true
```

Nx creates:

- `apps/iam-portal/src/app/core/iam-admin/services/admin-employees-api.service.ts`
- `apps/iam-portal/src/app/core/iam-admin/services/admin-employees-api.service.spec.ts`

We’ll update the service implementation next.

---

### 12.3.9 AdminEmployeesApiService implementation

This service wraps the IAM admin endpoints:

- `GET /api/iam/admin/employees`
- `GET /api/iam/admin/employee/{id}`
- `PUT /api/iam/admin/employees/{id}`
- `POST /api/iam/admin/employees/{id}/activate`
- `POST /api/iam/admin/employees/{id}/deactivate`

File:  
`apps/iam-portal/src/app/core/iam-admin/services/admin-employees-api.service.ts`

```ts
import { Injectable } from "@angular/core";
import { HttpClient, HttpParams } from "@angular/common/http";
import { Observable } from "rxjs";

import {
  EmployeeDetails,
  EmployeeListQuery,
  EmployeeSummary,
  PagedResult,
  UpdateEmployeeRequest,
} from "../models";

// Same pattern used in AuthApiService
const IAM_API_URL =
  (process.env["IAM_API_URL"] as string | undefined) ??
  "https://localhost:5001";

@Injectable({
  providedIn: "root",
})
export class AdminEmployeesApiService {
  private readonly baseUrl = `${IAM_API_URL}/api/iam/admin`;

  constructor(private readonly http: HttpClient) {}

  getEmployees(
    query: EmployeeListQuery
  ): Observable<PagedResult<EmployeeSummary>> {
    let params = new HttpParams()
      .set("pageNumber", query.pageNumber.toString())
      .set("pageSize", query.pageSize.toString());

    if (query.search) {
      params = params.set("search", query.search);
    }

    if (typeof query.includeInactive === "boolean") {
      params = params.set("includeInactive", query.includeInactive.toString());
    }

    return this.http.get<PagedResult<EmployeeSummary>>(
      `${this.baseUrl}/employees`,
      { params }
    );
  }

  getEmployeeById(id: string): Observable<EmployeeDetails> {
    return this.http.get<EmployeeDetails>(`${this.baseUrl}/employee/${id}`);
  }

  updateEmployee(request: UpdateEmployeeRequest): Observable<void> {
    return this.http.put<void>(
      `${this.baseUrl}/employees/${request.id}`,
      request
    );
  }

  activateEmployee(id: string): Observable<void> {
    return this.http.post<void>(`${this.baseUrl}/employees/${id}/activate`, {});
  }

  deactivateEmployee(id: string): Observable<void> {
    return this.http.post<void>(
      `${this.baseUrl}/employees/${id}/deactivate`,
      {}
    );
  }
}
```

Notes:

- We reuse the same `IAM_API_URL` pattern used in `AuthApiService` so environments are consistent.
- The service methods are **fully typed**, returning:
  - `Observable<PagedResult<EmployeeSummary>>`
  - `Observable<EmployeeDetails>`
  - `Observable<void>` for commands.
- All query-string construction happens here, not in components.  
  Components will only speak in terms of `EmployeeListQuery`.

---

### 12.3.10 Quick sanity check

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
```

We haven’t created any new components yet, so there should be no UI changes — only additional TypeScript types and a service.

With the **models and AdminEmployeesApiService** in place, we’re ready to build the **Employees list page** using a proper management table, paging, search, and status tags in the next section.

---

## 12.4 Employees List State & Page Setup

Now we build the **Admin Employees list** (`/admin/employees`) as a real management screen:

- A stateful list backed by the IAM admin endpoints.
- Search by email or full name.
- Toggle to include inactive employees.
- Paging controls wired to backend paging.
- A table that looks and behaves like a back-office banking tool.

We will:

1. Add an `AdminEmployeesStateService` to manage list state.
2. Implement the `EmployeesPage` component that:
   - Renders the filters.
   - Displays the management table.
   - Handles paging.
   - Navigates to the details page on row click.

---

### 12.4.1 AdminEmployeesStateService

This service sits on top of `AdminEmployeesApiService` and owns:

- The **current query** (`EmployeeListQuery`).
- The **current page** (`PagedResult<EmployeeSummary>`).
- Loading and error flags.

Target location:

- `apps/iam-portal/src/app/core/iam-admin/services/admin-employees-state.service.ts`

Generate with Nx:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:service \
  apps/iam-portal/src/app/core/iam-admin/services/admin-employees-state/admin-employees-state \
  --flat=true
```

Nx creates:

- `apps/iam-portal/src/app/core/iam-admin/services/admin-employees-state.service.ts`
- `apps/iam-portal/src/app/core/iam-admin/services/admin-employees-state.service.spec.ts`

Replace `admin-employees-state.service.ts` with:

```ts
import { Injectable } from "@angular/core";
import { BehaviorSubject, Observable } from "rxjs";

import { EmployeeListQuery, EmployeeSummary, PagedResult } from "../models";
import { AdminEmployeesApiService } from "./admin-employees-api.service";

@Injectable({
  providedIn: "root",
})
export class AdminEmployeesStateService {
  private readonly defaultQuery: EmployeeListQuery = {
    pageNumber: 1,
    pageSize: 20,
    search: null,
    includeInactive: false,
  };

  private readonly querySubject = new BehaviorSubject<EmployeeListQuery>(
    this.defaultQuery
  );

  private readonly employeesPageSubject =
    new BehaviorSubject<PagedResult<EmployeeSummary> | null>(null);

  private readonly loadingSubject = new BehaviorSubject<boolean>(false);
  private readonly errorSubject = new BehaviorSubject<string | null>(null);

  readonly query$: Observable<EmployeeListQuery> =
    this.querySubject.asObservable();

  readonly employeesPage$: Observable<PagedResult<EmployeeSummary> | null> =
    this.employeesPageSubject.asObservable();

  readonly loading$: Observable<boolean> = this.loadingSubject.asObservable();

  readonly error$: Observable<string | null> = this.errorSubject.asObservable();

  constructor(private readonly api: AdminEmployeesApiService) {}

  getCurrentQuery(): EmployeeListQuery {
    return this.querySubject.value;
  }

  updateQuery(patch: Partial<EmployeeListQuery>): void {
    const current = this.querySubject.value;

    const next: EmployeeListQuery = {
      ...current,
      ...patch,
      pageNumber: Math.max(1, patch.pageNumber ?? current.pageNumber),
    };

    this.querySubject.next(next);
  }

  loadEmployees(): void {
    const query = this.querySubject.value;

    this.loadingSubject.next(true);
    this.errorSubject.next(null);

    this.api.getEmployees(query).subscribe({
      next: (page) => {
        this.employeesPageSubject.next(page);
        this.loadingSubject.next(false);
      },
      error: () => {
        this.loadingSubject.next(false);
        this.errorSubject.next(
          "We could not load employees. Please try again or contact support."
        );
      },
    });
  }

  reset(): void {
    this.querySubject.next(this.defaultQuery);
    this.employeesPageSubject.next(null);
    this.errorSubject.next(null);
  }
}
```

This gives us a small list store:

- Components call `updateQuery(...)` and `loadEmployees()`.
- Components read `employeesPage$`, `loading$`, `error$`, `query$`.

---

### 12.4.2 EmployeesPage component scaffold

We now implement the **Employees list** page that:

- Uses `AdminEmployeesStateService`.
- Renders a management table.
- Navigates to `/admin/employees/:id` on row click.

If you already created `EmployeesPage` in earlier chapters, you can reuse it and replace the content.  
Otherwise, generate it now:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/admin/employees/employees-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

Files:

- `apps/iam-portal/src/app/features/admin/employees/employees-page/employees-page.component.ts`
- `...html`
- `...scss`

We’ll keep the class name `EmployeesPage` (no `Component` suffix) to align with our routing style.

---

### 12.4.3 EmployeesPage TypeScript

File:  
`apps/iam-portal/src/app/features/admin/employees/employees-page/employees-page.component.ts`

```ts
import {
  ChangeDetectionStrategy,
  Component,
  OnInit,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { FormBuilder, ReactiveFormsModule } from "@angular/forms";
import { Router, RouterModule } from "@angular/router";

import { AdminEmployeesStateService } from "../../../core/iam-admin/services/admin-employees-state.service";
import { EmployeeSummary } from "../../../core/iam-admin/models";

@Component({
  standalone: true,
  selector: "bs-employees-page",
  templateUrl: "./employees-page.component.html",
  styleUrl: "./employees-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
})
export class EmployeesPage implements OnInit {
  private readonly fb = inject(FormBuilder);
  private readonly state = inject(AdminEmployeesStateService);
  private readonly router = inject(Router);

  readonly employeesPage$ = this.state.employeesPage$;
  readonly loading$ = this.state.loading$;
  readonly error$ = this.state.error$;
  readonly query$ = this.state.query$;

  readonly filterForm = this.fb.nonNullable.group({
    search: [""],
    includeInactive: [false],
  });

  ngOnInit(): void {
    const query = this.state.getCurrentQuery();

    this.filterForm.patchValue({
      search: query.search ?? "",
      includeInactive: !!query.includeInactive,
    });

    this.state.loadEmployees();
  }

  onSearch(): void {
    const { search, includeInactive } = this.filterForm.getRawValue();

    this.state.updateQuery({
      pageNumber: 1,
      search: search?.trim() || null,
      includeInactive: !!includeInactive,
    });

    this.state.loadEmployees();
  }

  onClearSearch(): void {
    this.filterForm.patchValue({
      search: "",
      includeInactive: false,
    });

    this.state.updateQuery({
      pageNumber: 1,
      search: null,
      includeInactive: false,
    });

    this.state.loadEmployees();
  }

  onPageChange(targetPage: number): void {
    this.state.updateQuery({ pageNumber: targetPage });
    this.state.loadEmployees();
  }

  trackByEmployeeId(_: number, employee: EmployeeSummary): string {
    return employee.id;
  }

  openEmployeeDetails(employee: EmployeeSummary): void {
    this.router.navigate(["/admin/employees", employee.id]);
  }
}
```

---

### 12.4.4 EmployeesPage template (HTML)

We now build the management table using our design system:

- Page header.
- Filter bar (search + include inactive).
- Data table.
- Paging controls.

File:  
`apps/iam-portal/src/app/features/admin/employees/employees-page/employees-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">Employees</h1>
      <p class="page-header__subtitle">
        Manage IAM users for Alvor Bank: view status, roles, and security
        attributes.
      </p>
    </div>
  </header>

  <!-- Filters -->
  <section class="employees-filters">
    <form
      [formGroup]="filterForm"
      class="employees-filters__form"
      (ngSubmit)="onSearch()"
    >
      <div class="employees-filters__search">
        <label for="search" class="employees-filters__label"> Search </label>
        <input
          id="search"
          type="text"
          formControlName="search"
          placeholder="Search by email or name"
        />
      </div>

      <div class="employees-filters__include-inactive">
        <label class="employees-filters__checkbox-label">
          <input type="checkbox" formControlName="includeInactive" />
          <span>Include inactive employees</span>
        </label>
      </div>

      <div class="employees-filters__actions">
        <button
          class="bs-btn bs-btn--ghost"
          type="button"
          (click)="onClearSearch()"
        >
          Clear
        </button>
        <button class="bs-btn bs-btn--primary" type="submit">Apply</button>
      </div>
    </form>
  </section>

  <!-- Error message -->
  <div *ngIf="error$ | async as error" class="bs-alert bs-alert--danger">
    <div>
      <div class="bs-alert__title">Unable to load employees</div>
      <p class="bs-alert__body">{{ error }}</p>
    </div>
  </div>

  <!-- Table -->
  <section class="employees-table">
    <div
      class="employees-table__wrapper"
      *ngIf="employeesPage$ | async as page"
    >
      <table>
        <thead>
          <tr>
            <th>Email</th>
            <th>Full name</th>
            <th>Status</th>
            <th>Email</th>
            <th>2FA</th>
            <th>Roles</th>
          </tr>
        </thead>
        <tbody>
          <tr
            *ngFor="let employee of page.items; trackBy: trackByEmployeeId"
            (click)="openEmployeeDetails(employee)"
          >
            <td class="employees-table__email">{{ employee.email }}</td>
            <td>{{ employee.fullName || '—' }}</td>
            <td>
              <span
                class="bs-tag"
                [ngClass]="{
                  'bs-tag--success': employee.isActive,
                  'bs-tag--danger': !employee.isActive
                }"
              >
                {{ employee.isActive ? 'Active' : 'Inactive' }}
              </span>
            </td>
            <td>
              <span
                class="bs-tag"
                [ngClass]="{
                  'bs-tag--success': employee.emailConfirmed,
                  'bs-tag--neutral': !employee.emailConfirmed
                }"
              >
                {{ employee.emailConfirmed ? 'Confirmed' : 'Unconfirmed' }}
              </span>
            </td>
            <td>
              <span
                class="bs-tag"
                [ngClass]="{
                  'bs-tag--success': employee.twoFactorEnabled,
                  'bs-tag--neutral': !employee.twoFactorEnabled
                }"
              >
                {{ employee.twoFactorEnabled ? 'Enabled' : 'Disabled' }}
              </span>
            </td>
            <td>
              <span
                *ngFor="let role of employee.roles; let i = index"
                class="bs-tag bs-tag--neutral employees-table__role-tag"
              >
                {{ role }}
              </span>
              <span *ngIf="employee.roles.length === 0">—</span>
            </td>
          </tr>

          <tr *ngIf="page.items.length === 0">
            <td colspan="6" class="employees-table__empty">
              No employees found for the current filters.
            </td>
          </tr>
        </tbody>
      </table>

      <!-- Pagination -->
      <div class="employees-pagination">
        <button
          class="bs-btn bs-btn--ghost"
          type="button"
          (click)="onPageChange(page.pageNumber - 1)"
          [disabled]="!page.hasPreviousPage"
        >
          Previous
        </button>

        <span class="employees-pagination__info">
          Page {{ page.pageNumber }} of {{ page.totalPages }} &middot; {{
          page.totalCount }} employees
        </span>

        <button
          class="bs-btn bs-btn--ghost"
          type="button"
          (click)="onPageChange(page.pageNumber + 1)"
          [disabled]="!page.hasNextPage"
        >
          Next
        </button>
      </div>
    </div>

    <!-- Loading overlay -->
    <div class="employees-table__loading" *ngIf="loading$ | async">
      <div class="employees-table__loading-indicator">Loading employees…</div>
    </div>
  </section>
</div>
```

---

### 12.4.5 EmployeesPage styles (SCSS)

File:  
`apps/iam-portal/src/app/features/admin/employees/employees-page/employees-page.component.scss`

```scss
:host {
  display: block;
}

.page {
  max-width: 960px;
}

/* Filters */

.employees-filters {
  margin-top: var(--bs-space-4);
  margin-bottom: var(--bs-space-3);
}

.employees-filters__form {
  display: grid;
  grid-template-columns: minmax(0, 2fr) auto auto;
  gap: var(--bs-space-3);
  align-items: end;
}

.employees-filters__label {
  display: block;
  font-size: var(--bs-font-size-xs);
  margin-bottom: var(--bs-space-1);
  color: var(--bs-color-text-muted);
}

.employees-filters__search input {
  width: 100%;
  padding: 0.45rem 0.6rem;
  border-radius: var(--bs-radius-md);
  border: 1px solid var(--bs-color-border-subtle);
  font-size: var(--bs-font-size-sm);
}

.employees-filters__include-inactive {
  display: flex;
  align-items: center;
}

.employees-filters__checkbox-label {
  display: inline-flex;
  align-items: center;
  gap: var(--bs-space-1);
  font-size: var(--bs-font-size-sm);
}

.employees-filters__actions {
  display: flex;
  justify-content: flex-end;
  gap: var(--bs-space-2);
}

/* Table */

.employees-table {
  position: relative;
  margin-top: var(--bs-space-3);
}

.employees-table__wrapper {
  border-radius: var(--bs-radius-lg);
  background-color: var(--bs-color-surface);
  box-shadow: var(--bs-shadow-soft);
  border: 1px solid var(--bs-color-border-subtle);
  overflow: hidden;
}

.employees-table table {
  width: 100%;
  border-collapse: collapse;
  font-size: var(--bs-font-size-sm);
}

.employees-table thead {
  background-color: #f1f5f9;
}

.employees-table th,
.employees-table td {
  padding: 0.6rem 0.8rem;
  text-align: left;
  border-bottom: 1px solid var(--bs-color-border-subtle);
}

.employees-table th {
  font-weight: 600;
  color: var(--bs-color-text-muted);
}

.employees-table tbody tr {
  cursor: pointer;
}

.employees-table tbody tr:hover {
  background-color: #f8fafc;
}

.employees-table__email {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    "Liberation Mono", "Courier New", monospace;
}

.employees-table__role-tag {
  margin-right: 0.25rem;
  margin-bottom: 0.15rem;
}

.employees-table__empty {
  text-align: center;
  padding: 1.5rem 0.8rem;
  color: var(--bs-color-text-muted);
}

/* Pagination */

.employees-pagination {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--bs-space-2) var(--bs-space-3);
}

.employees-pagination__info {
  font-size: var(--bs-font-size-xs);
  color: var(--bs-color-text-muted);
}

/* Loading overlay */

.employees-table__loading {
  position: absolute;
  inset: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  pointer-events: none;
}

.employees-table__loading-indicator {
  padding: 0.5rem 0.9rem;
  border-radius: 999px;
  background-color: rgba(15, 23, 42, 0.85);
  color: white;
  font-size: var(--bs-font-size-xs);
}
```

---

### 12.4.6 Sanity check

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
npx nx serve iam-portal
```

Manual test:

1. Log in as an IAM admin, navigate to `/admin/employees`.
2. Verify:
   - Table displays seeded employees (from backend).
   - Search filters the list (backend-driven).
   - “Include inactive” toggle works.
   - “Previous/Next” paging buttons move through pages.
   - Clicking a row navigates to `/admin/employees/{id}` (details page placeholder).

With the **Employees list state and page** in place, we are ready to implement the **Employee details/edit page** in the next section.

---

## 12.5 Employee Details & Edit Page

The list page now gives IAM admins a good overview.  
Next, we need a **details/edit page** for a single employee:

- View key status flags (Active, Email Confirmed, 2FA, Roles).
- Edit full name and phone number.
- Update roles.
- Activate / deactivate the employee.

The route is already reserved:

- `/admin/employees/:id` → guarded by `authGuard`.

We’ll:

1. Implement the `EmployeeDetailsPage` component.
2. Use `AdminEmployeesApiService` to load, update, activate, and deactivate.
3. Keep the UI aligned with the design system and management-table style.

---

### 12.5.1 Generate (or reuse) EmployeeDetailsPage

If you already created `EmployeeDetailsPage` in earlier chapters, reuse it and replace the implementation below.

If not, from `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/admin/employees/employee-details-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

Files:

- `apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page.component.ts`
- `apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page.component.html`
- `apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page.component.scss`

---

### 12.5.2 EmployeeDetailsPage TypeScript

We will:

- Read `id` from the route.
- Load the employee via `AdminEmployeesApiService.getEmployeeById`.
- Keep `availableRoles` as a fixed list (must match backend seeds).
- Use a reactive form:

  - `fullName`
  - `phoneNumber`
  - `roles` as a `FormArray<boolean>` (one checkbox per role).

- Provide handlers to:
  - Save profile updates.
  - Activate/deactivate the employee.

File:  
`apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page.component.ts`

```ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  OnInit,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import {
  FormArray,
  FormBuilder,
  ReactiveFormsModule,
  Validators,
} from "@angular/forms";
import { ActivatedRoute, Router, RouterModule } from "@angular/router";
import { finalize } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

import {
  EmployeeDetails,
  UpdateEmployeeRequest,
} from "../../../core/iam-admin/models";
import { AdminEmployeesApiService } from "../../../core/iam-admin/services/admin-employees-api.service";

@Component({
  standalone: true,
  selector: "bs-employee-details-page",
  templateUrl: "./employee-details-page.component.html",
  styleUrl: "./employee-details-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
})
export class EmployeeDetailsPage implements OnInit {
  private readonly fb = inject(FormBuilder);
  private readonly route = inject(ActivatedRoute);
  private readonly router = inject(Router);
  private readonly api = inject(AdminEmployeesApiService);
  private readonly destroyRef = inject(DestroyRef);

  // These should match the IAM roles seeded in the backend
  readonly availableRoles = ["Employee", "IamAdmin", "SuperAdmin"] as const;

  employeeId: string | null = null;
  employee: EmployeeDetails | null = null;

  isLoading = true;
  isSaving = false;
  isTogglingStatus = false;

  loadError: string | null = null;
  saveSuccess: string | null = null;
  saveError: string | null = null;
  statusError: string | null = null;

  readonly profileForm = this.fb.nonNullable.group({
    fullName: ["", [Validators.maxLength(200)]],
    phoneNumber: ["", [Validators.maxLength(50)]],
    roles: this.fb.array([] as boolean[]),
  });

  get rolesArray(): FormArray {
    return this.profileForm.get("roles") as FormArray;
  }

  ngOnInit(): void {
    this.employeeId = this.route.snapshot.paramMap.get("id");

    if (!this.employeeId) {
      this.loadError =
        "Employee not found. Please use the Employees list to select a valid user.";
      this.isLoading = false;
      return;
    }

    // Initialize roles FormArray (one control per available role)
    this.availableRoles.forEach(() => {
      this.rolesArray.push(this.fb.control(false, { nonNullable: true }));
    });

    this.loadEmployee(this.employeeId);
  }

  private loadEmployee(id: string): void {
    this.isLoading = true;
    this.loadError = null;
    this.saveSuccess = null;
    this.saveError = null;
    this.statusError = null;

    this.api
      .getEmployeeById(id)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isLoading = false;
        })
      )
      .subscribe({
        next: (employee) => {
          this.employee = employee;

          this.profileForm.patchValue({
            fullName: employee.fullName ?? "",
            phoneNumber: employee.phoneNumber ?? "",
          });

          // Map roles -> checkbox booleans
          const currentRoles = new Set(employee.roles);
          this.availableRoles.forEach((role, index) => {
            const hasRole = currentRoles.has(role);
            this.rolesArray.at(index).setValue(hasRole);
          });
        },
        error: () => {
          this.loadError =
            "We could not load this employee. Please go back to the list and try again.";
        },
      });
  }

  onBackToList(): void {
    this.router.navigate(["/admin/employees"]);
  }

  onSaveProfile(): void {
    if (!this.employeeId || !this.employee) {
      return;
    }

    if (this.profileForm.invalid || this.isSaving) {
      this.profileForm.markAllAsTouched();
      return;
    }

    this.isSaving = true;
    this.saveSuccess = null;
    this.saveError = null;

    const formValue = this.profileForm.getRawValue();

    const selectedRoles: string[] = this.availableRoles.filter(
      (_role, index) => !!formValue.roles[index]
    );

    const request: UpdateEmployeeRequest = {
      id: this.employeeId,
      fullName: formValue.fullName?.trim() || null,
      phoneNumber: formValue.phoneNumber?.trim() || null,
      roles: selectedRoles,
    };

    this.api
      .updateEmployee(request)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isSaving = false;
        })
      )
      .subscribe({
        next: () => {
          // Keep local copy in sync
          this.employee = {
            ...this.employee!,
            fullName: request.fullName,
            phoneNumber: request.phoneNumber,
            roles: selectedRoles,
          };

          this.saveSuccess = "Employee profile has been updated.";
        },
        error: () => {
          this.saveError =
            "We could not save changes for this employee. Please try again.";
        },
      });
  }

  onActivate(): void {
    if (!this.employeeId || !this.employee || this.isTogglingStatus) {
      return;
    }

    this.isTogglingStatus = true;
    this.statusError = null;

    this.api
      .activateEmployee(this.employeeId)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isTogglingStatus = false;
        })
      )
      .subscribe({
        next: () => {
          this.employee = { ...this.employee!, isActive: true };
        },
        error: () => {
          this.statusError =
            "We could not activate this employee. Please try again.";
        },
      });
  }

  onDeactivate(): void {
    if (!this.employeeId || !this.employee || this.isTogglingStatus) {
      return;
    }

    this.isTogglingStatus = true;
    this.statusError = null;

    this.api
      .deactivateEmployee(this.employeeId)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isTogglingStatus = false;
        })
      )
      .subscribe({
        next: () => {
          this.employee = { ...this.employee!, isActive: false };
        },
        error: () => {
          this.statusError =
            "We could not deactivate this employee. Please try again.";
        },
      });
  }

  isRoleChecked(index: number): boolean {
    return !!this.rolesArray.at(index).value;
  }
}
```

---

### 12.5.3 EmployeeDetailsPage template (HTML)

The UI:

- Shows identity (email) and core status tags.
- Displays technical info (Last login, Lockout).
- Provides a **Profile** section (name, phone, roles).
- Provides **Status actions** (Activate / Deactivate).

File:  
`apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">Employee details</h1>
      <p class="page-header__subtitle">
        View and manage access for a single IAM user.
      </p>
    </div>

    <button type="button" class="bs-btn bs-btn--ghost" (click)="onBackToList()">
      Back to employees
    </button>
  </header>

  <ng-container *ngIf="isLoading">
    <div class="bs-alert bs-alert--info">
      <div>
        <div class="bs-alert__title">Loading employee…</div>
        <p class="bs-alert__body">
          Please wait while we retrieve this employee&#8217;s details.
        </p>
      </div>
    </div>
  </ng-container>

  <ng-container *ngIf="!isLoading && loadError">
    <div class="bs-alert bs-alert--danger">
      <div>
        <div class="bs-alert__title">Unable to load employee</div>
        <p class="bs-alert__body">{{ loadError }}</p>
      </div>
    </div>
  </ng-container>

  <ng-container *ngIf="!isLoading && !loadError && employee as emp">
    <!-- Header card with status -->
    <section class="employee-header-card">
      <div class="employee-header-card__main">
        <div class="employee-header-card__email">{{ emp.email }}</div>
        <div class="employee-header-card__name">
          {{ emp.fullName || 'No name on record' }}
        </div>
        <div class="employee-header-card__status-tags">
          <span
            class="bs-tag"
            [ngClass]="{
              'bs-tag--success': emp.isActive,
              'bs-tag--danger': !emp.isActive
            }"
          >
            {{ emp.isActive ? 'Active' : 'Inactive' }}
          </span>

          <span
            class="bs-tag"
            [ngClass]="{
              'bs-tag--success': emp.emailConfirmed,
              'bs-tag--neutral': !emp.emailConfirmed
            }"
          >
            {{ emp.emailConfirmed ? 'Email confirmed' : 'Email not confirmed' }}
          </span>

          <span
            class="bs-tag"
            [ngClass]="{
              'bs-tag--success': emp.twoFactorEnabled,
              'bs-tag--neutral': !emp.twoFactorEnabled
            }"
          >
            {{ emp.twoFactorEnabled ? '2FA enabled' : '2FA disabled' }}
          </span>
        </div>
      </div>

      <div class="employee-header-card__meta">
        <div class="employee-header-card__meta-item">
          <div class="employee-header-card__meta-label">Last login</div>
          <div class="employee-header-card__meta-value">
            {{ emp.lastLoginAt || '—' }}
          </div>
        </div>
        <div class="employee-header-card__meta-item">
          <div class="employee-header-card__meta-label">Lockout</div>
          <div class="employee-header-card__meta-value">
            {{ emp.lockoutEnd || '—' }}
          </div>
        </div>
      </div>
    </section>

    <!-- Save/status messages -->
    <section class="employee-messages">
      <div
        *ngIf="saveSuccess"
        class="bs-alert bs-alert--success"
        aria-live="polite"
      >
        <div>
          <div class="bs-alert__title">Profile saved</div>
          <p class="bs-alert__body">{{ saveSuccess }}</p>
        </div>
      </div>

      <div
        *ngIf="saveError"
        class="bs-alert bs-alert--danger"
        aria-live="polite"
      >
        <div>
          <div class="bs-alert__title">Unable to save profile</div>
          <p class="bs-alert__body">{{ saveError }}</p>
        </div>
      </div>

      <div
        *ngIf="statusError"
        class="bs-alert bs-alert--danger"
        aria-live="polite"
      >
        <div>
          <div class="bs-alert__title">Unable to update status</div>
          <p class="bs-alert__body">{{ statusError }}</p>
        </div>
      </div>
    </section>

    <!-- Profile & roles -->
    <section class="employee-section">
      <div class="employee-section__header">
        <h2 class="employee-section__title">Profile</h2>
        <p class="employee-section__subtitle">
          Basic contact details and access roles for this employee.
        </p>
      </div>

      <form
        [formGroup]="profileForm"
        class="bs-form"
        (ngSubmit)="onSaveProfile()"
      >
        <div class="bs-form-field">
          <label class="bs-form-field__label" for="fullName"> Full name </label>
          <div class="bs-form-field__control">
            <input
              id="fullName"
              type="text"
              formControlName="fullName"
              maxlength="200"
            />
          </div>
        </div>

        <div class="bs-form-field">
          <label class="bs-form-field__label" for="phoneNumber">
            Phone number
          </label>
          <div class="bs-form-field__control">
            <input
              id="phoneNumber"
              type="tel"
              formControlName="phoneNumber"
              maxlength="50"
            />
          </div>
        </div>

        <div class="bs-form-field">
          <label class="bs-form-field__label">Roles</label>
          <div class="employee-roles">
            <label
              class="employee-roles__item"
              *ngFor="
                let role of availableRoles;
                let i = index
              "
            >
              <input
                type="checkbox"
                [formControlName]="i"
                [checked]="isRoleChecked(i)"
                [formControl]="rolesArray.at(i)"
              />
              <span>{{ role }}</span>
            </label>
          </div>
        </div>

        <div class="employee-section__actions">
          <button
            class="bs-btn bs-btn--primary"
            type="submit"
            [disabled]="isSaving"
          >
            <span *ngIf="!isSaving">Save changes</span>
            <span *ngIf="isSaving">Saving…</span>
          </button>
        </div>
      </form>
    </section>

    <!-- Status actions -->
    <section class="employee-section">
      <div class="employee-section__header employee-section__header--status">
        <div>
          <h2 class="employee-section__title">Account status</h2>
          <p class="employee-section__subtitle">
            Activate or deactivate this employee&#8217;s IAM access.
          </p>
        </div>

        <span
          class="bs-tag"
          [ngClass]="{
            'bs-tag--success': emp.isActive,
            'bs-tag--danger': !emp.isActive
          }"
        >
          {{ emp.isActive ? 'Active' : 'Inactive' }}
        </span>
      </div>

      <div class="employee-section__actions">
        <button
          *ngIf="!emp.isActive"
          class="bs-btn bs-btn--primary"
          type="button"
          (click)="onActivate()"
          [disabled]="isTogglingStatus"
        >
          <span *ngIf="!isTogglingStatus">Activate employee</span>
          <span *ngIf="isTogglingStatus">Updating…</span>
        </button>

        <button
          *ngIf="emp.isActive"
          class="bs-btn bs-btn--danger"
          type="button"
          (click)="onDeactivate()"
          [disabled]="isTogglingStatus"
        >
          <span *ngIf="!isTogglingStatus">Deactivate employee</span>
          <span *ngIf="isTogglingStatus">Updating…</span>
        </button>
      </div>
    </section>
  </ng-container>
</div>
```

---

### 12.5.4 EmployeeDetailsPage styles (SCSS)

File:  
`apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page.component.scss`

```scss
:host {
  display: block;
}

.page {
  max-width: 960px;
}

/* Header card */

.employee-header-card {
  margin-top: var(--bs-space-4);
  padding: var(--bs-space-4);
  border-radius: var(--bs-radius-lg);
  background-color: var(--bs-color-surface);
  box-shadow: var(--bs-shadow-soft);
  border: 1px solid var(--bs-color-border-subtle);

  display: flex;
  justify-content: space-between;
  gap: var(--bs-space-4);
  flex-wrap: wrap;
}

.employee-header-card__main {
  display: flex;
  flex-direction: column;
  gap: var(--bs-space-1);
}

.employee-header-card__email {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    "Liberation Mono", "Courier New", monospace;
  font-size: var(--bs-font-size-sm);
}

.employee-header-card__name {
  font-size: var(--bs-font-size-lg);
  font-weight: 600;
}

.employee-header-card__status-tags {
  margin-top: var(--bs-space-2);
  display: flex;
  flex-wrap: wrap;
  gap: var(--bs-space-1);
}

.employee-header-card__meta {
  display: flex;
  flex-direction: column;
  gap: var(--bs-space-2);
  min-width: 200px;
}

.employee-header-card__meta-item {
  font-size: var(--bs-font-size-xs);
}

.employee-header-card__meta-label {
  color: var(--bs-color-text-muted);
  margin-bottom: 0.15rem;
}

.employee-header-card__meta-value {
  font-weight: 500;
}

/* Messages */

.employee-messages {
  margin-top: var(--bs-space-3);
  display: flex;
  flex-direction: column;
  gap: var(--bs-space-2);
}

/* Sections */

.employee-section {
  margin-top: var(--bs-space-4);
  padding: var(--bs-space-4);
  border-radius: var(--bs-radius-lg);
  background-color: var(--bs-color-surface);
  box-shadow: var(--bs-shadow-soft);
  border: 1px solid var(--bs-color-border-subtle);
}

.employee-section__header {
  margin-bottom: var(--bs-space-3);
}

.employee-section__header--status {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: var(--bs-space-3);
}

.employee-section__title {
  margin: 0 0 var(--bs-space-1) 0;
  font-size: var(--bs-font-size-lg);
}

.employee-section__subtitle {
  margin: 0;
  font-size: var(--bs-font-size-sm);
  color: var(--bs-color-text-muted);
}

.employee-section__actions {
  margin-top: var(--bs-space-3);
  display: flex;
  justify-content: flex-end;
  gap: var(--bs-space-2);
}

/* Roles */

.employee-roles {
  display: flex;
  flex-wrap: wrap;
  gap: var(--bs-space-2);
}

.employee-roles__item {
  display: inline-flex;
  align-items: center;
  gap: var(--bs-space-1);
  font-size: var(--bs-font-size-sm);
}
```

---

### 12.5.5 Sanity check

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
npx nx serve iam-portal
```

Manual flow:

1. Log in as an IAM admin.
2. Go to `/admin/employees`.
3. Click any employee row → you should be navigated to `/admin/employees/{id}`.
4. Confirm that:
   - Employee header card shows email, name, and tags.
   - Profile form is populated with full name, phone, and current roles.
   - Changing name/phone/roles and clicking **Save changes** updates the user.
   - Clicking **Deactivate employee** marks the user as inactive (and vice versa for **Activate**).

With the **Employee details & edit page** implemented, the IAM portal now gives admins end-to-end control over employee accounts, matching the capabilities of the backend.

Next, we’ll add **tests** (unit + E2E) for the Admin Employees UI and close the chapter with the usual frontend quality gate and branch-merge ritual.

---

## 12.6 Testing the Admin Employees UI

We now add tests so the **Admin Employees UI** is safe to rely on:

- Unit tests for:
  - `AdminEmployeesApiService` (HTTP contracts).
  - `AdminEmployeesStateService` (list state behavior).
- E2E tests for:
  - Listing employees as an IAM admin.
  - Navigating from list → details page.
  - Performing a basic profile update.

As before, we use:

- **Jest** for unit tests (`iam-portal`).
- **Cypress** for E2E (`iam-portal-e2e`).

---

### 12.6.1 Unit tests for AdminEmployeesApiService

We validate:

- Correct URLs and methods.
- Querystring composition for paging & filters.
- Response types.

File:  
`apps/iam-portal/src/app/core/iam-admin/services/admin-employees-api.service.spec.ts`

```ts
import { TestBed } from "@angular/core/testing";
import {
  HttpClientTestingModule,
  HttpTestingController,
} from "@angular/common/http/testing";

import { AdminEmployeesApiService } from "./admin-employees-api.service";
import {
  EmployeeListQuery,
  EmployeeSummary,
  PagedResult,
  EmployeeDetails,
  UpdateEmployeeRequest,
} from "../models";

describe("AdminEmployeesApiService", () => {
  let service: AdminEmployeesApiService;
  let httpMock: HttpTestingController;

  const IAM_API_URL =
    (process.env["IAM_API_URL"] as string | undefined) ??
    "https://localhost:5001";
  const baseUrl = `${IAM_API_URL}/api/iam/admin`;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [AdminEmployeesApiService],
    });

    service = TestBed.inject(AdminEmployeesApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it("should call GET /employees with paging and filters", () => {
    const query: EmployeeListQuery = {
      pageNumber: 2,
      pageSize: 20,
      search: "john",
      includeInactive: true,
    };

    const mockPage: PagedResult<EmployeeSummary> = {
      items: [],
      pageNumber: 2,
      pageSize: 20,
      totalCount: 0,
      totalPages: 0,
      hasNextPage: false,
      hasPreviousPage: true,
    };

    service.getEmployees(query).subscribe((page) => {
      expect(page).toEqual(mockPage);
    });

    const req = httpMock.expectOne((r) => {
      return (
        r.method === "GET" &&
        r.url === `${baseUrl}/employees` &&
        r.params.get("pageNumber") === "2" &&
        r.params.get("pageSize") === "20" &&
        r.params.get("search") === "john" &&
        r.params.get("includeInactive") === "true"
      );
    });

    req.flush(mockPage);
  });

  it("should call GET /employee/{id} for details", () => {
    const id = "employee-id-123";

    const mockDetails: EmployeeDetails = {
      id,
      email: "user@example.com",
      fullName: "Test User",
      emailConfirmed: true,
      isActive: true,
      twoFactorEnabled: false,
      roles: ["Employee"],
      phoneNumber: "+27...",
      lockoutEnd: null,
      lastLoginAt: null,
    };

    service.getEmployeeById(id).subscribe((details) => {
      expect(details).toEqual(mockDetails);
    });

    const req = httpMock.expectOne(`${baseUrl}/employee/${id}`);
    expect(req.request.method).toBe("GET");

    req.flush(mockDetails);
  });

  it("should call PUT /employees/{id} to update employee", () => {
    const request: UpdateEmployeeRequest = {
      id: "employee-id-123",
      fullName: "Updated Name",
      phoneNumber: "+27...",
      roles: ["Employee", "IamAdmin"],
    };

    service.updateEmployee(request).subscribe((result) => {
      expect(result).toBeUndefined();
    });

    const req = httpMock.expectOne(`${baseUrl}/employees/${request.id}`);
    expect(req.request.method).toBe("PUT");
    expect(req.request.body).toEqual(request);

    req.flush(null);
  });

  it("should call POST /employees/{id}/activate", () => {
    const id = "employee-id-123";

    service.activateEmployee(id).subscribe((result) => {
      expect(result).toBeUndefined();
    });

    const req = httpMock.expectOne(`${baseUrl}/employees/${id}/activate`);
    expect(req.request.method).toBe("POST");
    expect(req.request.body).toEqual({});

    req.flush(null);
  });

  it("should call POST /employees/{id}/deactivate", () => {
    const id = "employee-id-123";

    service.deactivateEmployee(id).subscribe((result) => {
      expect(result).toBeUndefined();
    });

    const req = httpMock.expectOne(`${baseUrl}/employees/${id}/deactivate`);
    expect(req.request.method).toBe("POST");
    expect(req.request.body).toEqual({});

    req.flush(null);
  });
});
```

---

### 12.6.2 Unit tests for AdminEmployeesStateService

We ensure the state service:

- Starts with the default query.
- Updates query correctly.
- Calls the API and populates `employeesPage$`.
- Handles errors.

File:  
`apps/iam-portal/src/app/core/iam-admin/services/admin-employees-state.service.spec.ts`

```ts
import { TestBed } from "@angular/core/testing";
import { of, throwError } from "rxjs";

import { AdminEmployeesStateService } from "./admin-employees-state.service";
import { AdminEmployeesApiService } from "./admin-employees-api.service";
import { EmployeeListQuery, EmployeeSummary, PagedResult } from "../models";

describe("AdminEmployeesStateService", () => {
  let service: AdminEmployeesStateService;
  let apiSpy: jasmine.SpyObj<AdminEmployeesApiService>;

  const mockPage: PagedResult<EmployeeSummary> = {
    items: [],
    pageNumber: 1,
    pageSize: 20,
    totalCount: 0,
    totalPages: 0,
    hasNextPage: false,
    hasPreviousPage: false,
  };

  beforeEach(() => {
    apiSpy = jasmine.createSpyObj<AdminEmployeesApiService>(
      "AdminEmployeesApiService",
      ["getEmployees"]
    );

    TestBed.configureTestingModule({
      providers: [
        AdminEmployeesStateService,
        { provide: AdminEmployeesApiService, useValue: apiSpy },
      ],
    });

    service = TestBed.inject(AdminEmployeesStateService);
  });

  it("should expose default query on init", (done) => {
    service.query$.subscribe((query: EmployeeListQuery) => {
      expect(query.pageNumber).toBe(1);
      expect(query.pageSize).toBe(20);
      expect(query.search).toBeNull();
      expect(query.includeInactive).toBeFalse();
      done();
    });
  });

  it("should update query and call API when loadEmployees is invoked", (done) => {
    apiSpy.getEmployees.and.returnValue(of(mockPage));

    service.updateQuery({ pageNumber: 2, search: "john" });
    service.loadEmployees();

    expect(apiSpy.getEmployees).toHaveBeenCalled();

    service.employeesPage$.subscribe((page) => {
      expect(page).toEqual(mockPage);
      done();
    });
  });

  it("should set error when API call fails", (done) => {
    apiSpy.getEmployees.and.returnValue(throwError(() => new Error("fail")));

    service.loadEmployees();

    service.error$.subscribe((err) => {
      expect(err).toContain("We could not load employees");
      done();
    });
  });
});
```

These tests focus on behavior, not implementation details.

---

### 12.6.3 E2E tests for Admin Employees flows

We extend the Cypress E2E suite to cover IAM admin flows:

- Login as an IAM admin.
- Visit the employees list page and see the table.
- Click a row to open details.
- Perform a small edit and see a success message.

Create a new E2E spec so the tests stay focused:

File:  
`apps/iam-portal-e2e/src/e2e/admin-employees.cy.ts`

```ts
describe("IAM Admin - Employees Management", () => {
  const baseUrl = "http://localhost:4200";

  // Must match a seeded IAM admin user in the backend
  const adminEmail = "iam.admin@alvorbank.test";
  const adminPassword = "Admin123!";

  function loginAsAdmin() {
    cy.visit(`${baseUrl}/login`);

    cy.get('input[type="email"]').type(adminEmail);
    cy.get('input[type="password"]').type(adminPassword);
    cy.get('button[type="submit"]').click();

    // No 2FA in this baseline test
    cy.url().should("include", "/me/security");
  }

  it("should display the employees list for an IAM admin", () => {
    loginAsAdmin();

    cy.visit(`${baseUrl}/admin/employees`);

    cy.contains("Employees").should("be.visible");
    cy.get("table tbody tr").its("length").should("be.greaterThan", 0);
  });

  it("should navigate from list to employee details", () => {
    loginAsAdmin();

    cy.visit(`${baseUrl}/admin/employees`);

    cy.get("table tbody tr").first().click();

    cy.url().should("match", /\/admin\/employees\/.+/);
    cy.contains("Employee details").should("be.visible");
  });

  it("should allow updating an employee profile (UI success)", () => {
    loginAsAdmin();

    cy.visit(`${baseUrl}/admin/employees`);

    cy.get("table tbody tr").first().click();

    cy.url().should("match", /\/admin\/employees\/.+/);

    // Append a marker to ensure we change something
    cy.get("input#fullName").clear().type("Test User Updated");

    cy.contains("button", "Save changes").click();

    cy.contains("Profile saved").should("be.visible");
  });
});
```

Notes:

- These tests assume the IAM backend is running and has a seeded admin user with the given credentials.
- We do **not** assert on the exact employee count or data values; we simply confirm that:
  - The list loads.
  - Navigation works.
  - The UI acknowledges profile updates.

---

## 12.7 Local Frontend Quality Gate & Closing the Chapter

As with previous chapters, we close by running a **full frontend quality gate** for the IAM portal:

1. **Lint** (`iam-portal`).
2. **Unit tests** (`iam-portal`).
3. **E2E tests** (`iam-portal-e2e`).

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

# 1. Lint
npx nx lint iam-portal

# 2. Unit tests
npx nx test iam-portal

# 3. E2E tests
npx nx e2e iam-portal-e2e
```

All commands should pass before you move on.

---

### 12.7.1 Commit & push Chapter 12 work

From the repository root:

```bash
cd digital-banking-suite

git status
git add .
git commit -m "feat(ch12): IAM admin employees UI (list, details, tests)"
git push origin feature/ch12-frontend-iam-admin-employees
```

---

### 12.7.2 Pull Request to develop

Open a PR in your Git hosting:

- **Source branch**: `feature/ch12-frontend-iam-admin-employees`
- **Target branch**: `develop`
- Suggested title:
  - `feat(ch12): IAM admin employees UI in Angular`
- Suggested description:
  - Implemented IAM Admin Employees list page with paging, search and status tags.
  - Implemented Employee details/edit page with profile update and activate/deactivate actions.
  - Added AdminEmployees API + state services.
  - Added unit tests and E2E tests for admin flows.
  - All frontend quality checks (lint, unit, E2E) passing.

Wait for CI to confirm all checks are green, perform a quick self-review (or team review), and then merge into `develop`.

You may keep the feature branch for chapter-level reference, as we’ve done with previous chapters.

---

## 12.8 Chapter Summary & What’s Next

In this chapter we turned the IAM Admin capabilities of **Alvor Bank** into a full Angular UI:

- **Infrastructure:**

  - IAM admin models under `core/iam-admin/models`:
    - `EmployeeSummary`, `EmployeeDetails`, `UpdateEmployeeRequest`,
      `EmployeeListQuery`, `PagedResult<T>`.
  - `AdminEmployeesApiService` encapsulating `/api/iam/admin/...` endpoints.
  - `AdminEmployeesStateService` managing list state (query, page, loading, error).

- **Admin UI:**

  - `EmployeesPage` (`/admin/employees`):
    - Management table with search, include-inactive toggle, and paging.
    - Status tags for Active/Inactive, Email Confirmed, 2FA.
    - Row click → navigates to details.
  - `EmployeeDetailsPage` (`/admin/employees/:id`):
    - Header card with identity and status.
    - Profile form for full name, phone, and roles.
    - Account status section with Activate/Deactivate actions.

- **Quality:**
  - Unit tests for `AdminEmployeesApiService` and `AdminEmployeesStateService`.
  - Cypress E2E tests for admin login, list, navigation, and profile update.
  - Full frontend quality gate before merging to `develop`.

At this point the IAM portal feels like a **real back-office tool** used by IAM administrators in a bank, not a demo. They can:

- Authenticate securely (Chapter 11).
- Manage their own security settings (My Security).
- View, update, and control the status of other employees (Chapter 12).

In the next frontend chapter we can:

- Extend the IAM portal with richer **audit visibility** (login history, recent actions), or
- Move to another bounded context (e.g., **Accounts** or **Customers**) and reuse the same Angular + Nx foundation, design system, and quality gates.

Either way, you now have a solid, tested IAM Admin layer for Alvor Bank’s UI.
