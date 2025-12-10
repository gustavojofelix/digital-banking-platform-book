# Chapter 9 — Frontend Foundation: Angular 21, Nx & the IAM Employee Portal

In this chapter, we bring the **frontend** to life.

We will:

- Set up an **Angular 21 + Nx** workspace under `src/frontend`.
- Create the **IAM Employee Portal** (`iam-portal` app) for Alvor Bank staff.
- Put in place a **clean folder structure** and a **route skeleton** for all IAM flows.
- Fix and align unit tests and E2E tests with our new app shell.

> Many readers will jump straight into the UI.  
> This chapter is written so that you can:
>
> - Either follow from previous backend chapters, **or**
> - Clone the repo with backend ready and start from the frontend only.

In the **next chapter**, we will focus on **visual design & real IAM flows**:
a proper design system (colors, typography, spacing) and real authentication UI.

---

## 9.1 Where the Frontend Fits in the Overall Solution

By now, our repository looks like this:

```
digital-banking-suite/
  src/
    backend/
      BankingSuite.sln
      services/
        IamService/
        ...
    frontend/
      (empty for now — we’ll create the Nx workspace here)
  infra/
    ...
  README.md
```

We have two main frontend audiences:

1. **Bank employees** — internal portals:
   - IAM (identity & access management)
   - Operations dashboards
   - Back-office tooling
2. **Customers** — an **online banking portal** (internet banking).

In this chapter, we focus on the **Employee IAM Portal**, but we set up the workspace so that later we can add:

- `iam-portal` (this chapter)
- `online-banking-portal` (later)
- Additional admin tools if needed.

---

## 9.2 Branch Strategy for Frontend Work

We keep the same Git practices used for the backend:

- `main` — production-ready.
- `develop` — integration branch.
- `feature/*` — one branch per chapter or feature.

From the repository root:

```bash
cd digital-banking-suite

git checkout develop
git checkout -b feature/ch09-frontend-iam-portal-setup
```

We’ll merge this branch back into `develop` at the end of the chapter, after running tests and a local sanity check.

---

## 9.3 Prerequisites & Tooling

Before following this chapter, you should have:

- **Node.js** (20+ recommended)
- **npm**
- **Git**
- **Visual Studio Code**, plus recommended extensions:
  - Angular Language Service
  - ESLint
  - Nx Console (optional, but convenient)

Backend-wise, you can:

- Either build `IamService` following previous chapters, **or**
- Clone the repo at a tag/commit that represents “backend ready for IAM”.

You do **not** need to understand every backend detail to follow this chapter, but you do need to know:

- IAM endpoints live under `/api/iam/...`.
- `IamService` will be reachable from the frontend (directly or via API gateway).

---

## 9.4 Creating the Nx Angular Workspace

We keep backend and frontend clearly separated under `src/`.

From the repository root:

```bash
cd digital-banking-suite
cd src

npx create-nx-workspace@latest frontend --preset=angular-monorepo
```

When prompted (or via flags), choose:

- **Workspace name**: `frontend` (already set by the command).
- **Application name**: `iam-portal`
- **Styles**: `scss`
- **Standalone Components**: `Yes`
- **Unit test runner**: `Vitest` or `Jest` (your choice; we treat tests as mandatory)
- **E2E test runner**: `Cypress` or `Playwright`
- **Nx Cloud**: optional

This will create:

```
digital-banking-suite/
  src/
    backend/
      ...
    frontend/
      apps/
        iam-portal/
        iam-portal-e2e/
      nx.json
      package.json
      tsconfig.base.json
      ...
```

If dependencies were not installed automatically:

```bash
cd digital-banking-suite/src/frontend
npm install
```

---

## 9.5 VS Code Workflow for Frontend Development

Open the **root** folder (`digital-banking-suite`) in VS Code (not just `src/frontend`). This gives you:

- Backend and frontend side by side.
- Separate terminals for backend (`src/backend`) and frontend (`src/frontend`).

Common frontend commands (run from `src/frontend`):

```bash
# Serve the IAM portal locally
npx nx serve iam-portal

# Run unit tests for the IAM portal
npx nx test iam-portal

# Run linting for the IAM portal
npx nx lint iam-portal
```

You can also use **Nx Console** in VS Code, but the CLI remains our source of truth for documentation and CI.

---

## 9.6 First Run: Serving the IAM Portal

From `src/frontend`:

```bash
npx nx serve iam-portal
```

Open `http://localhost:4200` in your browser.

You should see the default Nx/Angular starter page. This confirms:

- Angular 21 tooling works.
- The Nx workspace is healthy.
- Unit test and lint plumbing exist and run.

> ✅ Commit suggestion
>
> ```bash
> git status
> git add .
> git commit -m "chore(frontend): scaffold Nx Angular workspace with iam-portal app"
> ```

---

## 9.7 Structuring the IAM App Like a Real Bank Project

We want a frontend structure that looks like something a real team would maintain.

We use this pattern inside the IAM app:

- `app.component.*` — shell (header + sidebar + `router-outlet`).
- `app.routes.ts` — route configuration.
- `features/` — feature screens grouped by domain.

Target structure:

```
apps/iam-portal/src/app/
  app.component.ts
  app.component.html
  app.component.scss
  app.config.ts
  app.routes.ts
  features/
    auth/
      login-page/
      two-factor-page/
      email-confirmation-page/
      forgot-password-page/
      reset-password-page/
      my-security-page/
    admin/
      employees/
        employees-page/
        employee-details-page/
```

In this chapter we focus on structure and routing.  
Styling will be functional and minimal — in the next chapter we design a proper visual system.

---

## 9.8 The IAM Shell Layout (App Component)

### 9.8.1 Update `AppComponent` to use external HTML & SCSS

Nx initially created `AppComponent` with an inline template. We refactor it to reference external files.

File: `apps/iam-portal/src/app/app.component.ts`

```ts
import { Component } from "@angular/core";
import { RouterOutlet } from "@angular/router";

@Component({
  selector: "bs-iam-root",
  standalone: true,
  imports: [RouterOutlet],
  templateUrl: "./app.component.html",
  styleUrls: ["./app.component.scss"],
})
export class AppComponent {}
```

### 9.8.2 Define the shell HTML

File: `apps/iam-portal/src/app/app.component.html`

```html
<div class="iam-layout">
  <header class="iam-header">
    <div class="iam-header__brand">
      <div class="iam-header__logo">Alvor Bank</div>
      <div class="iam-header__subtitle">Employee IAM Portal</div>
    </div>

    <div class="iam-header__user">
      <!-- In a later chapter this will be bound to the authenticated user -->
      <span class="iam-header__user-name">Not signed in</span>
      <span class="iam-header__user-role">Please log in</span>
    </div>
  </header>

  <div class="iam-body">
    <aside class="iam-sidebar">
      <nav class="iam-nav">
        <a routerLink="/login" routerLinkActive="active">Login</a>
        <a routerLink="/forgot-password" routerLinkActive="active">
          Forgot password
        </a>

        <div class="iam-nav__section">Admin</div>

        <a routerLink="/admin/employees" routerLinkActive="active">
          Employees
        </a>
      </nav>
    </aside>

    <main class="iam-content">
      <router-outlet></router-outlet>
    </main>
  </div>
</div>
```

### 9.8.3 Define basic layout styles

For now, we keep the styling simple and focus on structure.

File: `apps/iam-portal/src/app/app.component.scss`

```scss
.iam-layout {
  display: flex;
  flex-direction: column;
  height: 100vh;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

.iam-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.75rem 1.5rem;
  border-bottom: 1px solid #e5e7eb;
  background-color: #ffffff;
}

.iam-header__brand {
  display: flex;
  flex-direction: column;
}

.iam-header__logo {
  font-weight: 700;
  font-size: 0.9rem;
  text-transform: uppercase;
  letter-spacing: 0.12em;
}

.iam-header__subtitle {
  font-size: 0.85rem;
  color: #6b7280;
}

.iam-header__user {
  text-align: right;
  font-size: 0.85rem;
  color: #4b5563;
}

.iam-header__user-name {
  display: block;
  font-weight: 600;
}

.iam-header__user-role {
  font-size: 0.8rem;
}

.iam-body {
  display: grid;
  grid-template-columns: 220px 1fr;
  flex: 1;
  min-height: 0;
}

.iam-sidebar {
  border-right: 1px solid #e5e7eb;
  padding: 1rem;
  background-color: #f9fafb;
}

.iam-nav {
  display: flex;
  flex-direction: column;
  gap: 0.4rem;
}

.iam-nav a {
  display: block;
  padding: 0.5rem 0.75rem;
  border-radius: 0.5rem;
  text-decoration: none;
  color: #111827;
  font-size: 0.9rem;
}

.iam-nav a:hover {
  background-color: #e5e7eb;
}

.iam-nav a.active {
  background-color: #111827;
  color: #ffffff;
}

.iam-nav__section {
  margin-top: 1rem;
  margin-bottom: 0.25rem;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  color: #6b7280;
}

.iam-content {
  padding: 1.5rem;
  overflow: auto;
  background-color: #f3f4f6;
}
```

---

## 9.9 Generating Feature Components via Nx (No Inline Templates)

We now create dedicated components for each IAM flow.  
They live under `features/` and use separate `.html` + `.scss` files.

We group them by domain:

- `features/auth`
- `features/admin/employees`

> Note: in Nx 21, `@nx/angular:component` does **not** accept `--project`.  
> We pass a **path** instead and Nx infers the project.

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend
```

### 9.9.1 Generate auth feature components

```bash
# Login (step 1: email + password)
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/auth/login-page/login-page \
  --standalone --style=scss

# Two-factor verification (step 2)
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page \
  --standalone --style=scss

# Email confirmation page
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page \
  --standalone --style=scss

# Forgot password page
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page \
  --standalone --style=scss

# Reset password page
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page \
  --standalone --style=scss

# "My Security" page (change password, toggle 2FA)
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/auth/my-security-page/my-security-page \
  --standalone --style=scss
```

NX will create:

```
apps/iam-portal/src/app/features/auth/
  login-page/
    login-page.component.ts
    login-page.component.html
    login-page.component.scss
  two-factor-page/
  email-confirmation-page/
  forgot-password-page/
  reset-password-page/
  my-security-page/
```

### 9.9.2 Generate admin employee feature components

```bash
# Employees list page (grid + filters + paging)
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/admin/employees/employees-page/employees-page \
  --standalone --style=scss

# Employee details/edit page
npx nx g @nx/angular:component \
  apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page \
  --standalone --style=scss
```

Resulting structure:

```
apps/iam-portal/src/app/features/admin/employees/
  employees-page/
    employees-page.component.ts
    employees-page.component.html
    employees-page.component.scss
  employee-details-page/
    employee-details-page.component.ts
    employee-details-page.component.html
    employee-details-page.component.scss
```

---

## 9.10 Shared Page Layout Style

To keep feature pages consistent, we define a small `.page` layout class in the app’s global stylesheet.

File: `apps/iam-portal/src/styles.scss`

```scss
.page {
  max-width: 640px;
  margin: 2rem auto;
  background-color: #ffffff;
  padding: 1.5rem;
  border-radius: 0.75rem;
  box-shadow: 0 1px 2px rgba(15, 23, 42, 0.05);
}

.page h1 {
  font-size: 1.4rem;
  margin-bottom: 0.5rem;
}

.page p {
  font-size: 0.9rem;
  color: #4b5563;
}

.page ul {
  padding-left: 1.2rem;
  color: #4b5563;
}
```

We’ll refine the visual design (colors, components, spacing) in the next chapter.  
For now, this gives us clean, readable placeholder pages.

---

## 9.11 Placeholder Templates for IAM Pages

For this chapter, feature pages are **text-only placeholders** explaining which backend endpoints they will use. Real forms, tables and logic come later.

We only change the `.html` files; generated `.ts` and `.scss` files can stay minimal.

### 9.11.1 Login page (step 1)

File:  
`apps/iam-portal/src/app/features/auth/login-page/login-page.component.html`

```html
<section class="page">
  <h1>Login</h1>
  <p>Step 1 of authentication: email and password.</p>
  <p>
    In the next chapter, this page will call
    <code>POST /api/iam/auth/login</code> and branch on
    <code>requiresTwoFactor</code> to decide whether to:
  </p>
  <ul>
    <li>Complete login directly, or</li>
    <li>Redirect to the two-factor verification page.</li>
  </ul>
</section>
```

### 9.11.2 Two-factor page (step 2)

File:  
`apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page.component.html`

```html
<section class="page">
  <h1>Two-Factor Verification</h1>
  <p>Step 2 of authentication: enter the one-time 2FA code.</p>
  <p>
    This page will call <code>POST /api/iam/auth/2fa/verify</code> using the
    <code>userId</code> provided by the login response when
    <code>requiresTwoFactor = true</code>.
  </p>
</section>
```

### 9.11.3 Email confirmation page

File:  
`apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page.component.html`

```html
<section class="page">
  <h1>Email Confirmation</h1>
  <p>
    This page will read <code>userId</code> and <code>token</code> from the
    query string and call <code>GET /api/iam/auth/confirm-email</code>.
  </p>
  <p>
    The URL used in the email is configured by
    <code>Iam:EmailConfirmationBaseUrl</code> on the backend.
  </p>
</section>
```

### 9.11.4 Forgot password page

File:  
`apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page.component.html`

```html
<section class="page">
  <h1>Forgot Password</h1>
  <p>
    Employees will enter their email address here. The frontend will call
    <code>POST /api/iam/auth/forgot-password</code>.
  </p>
  <p>
    The backend always returns a generic “If this email exists, we sent a link”
    message to avoid leaking whether the account is valid.
  </p>
</section>
```

### 9.11.5 Reset password page

File:  
`apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page.component.html`

```html
<section class="page">
  <h1>Reset Password</h1>
  <p>
    This page will read <code>email</code> and <code>token</code> from the URL
    and call <code>POST /api/iam/auth/reset-password</code> with the new
    password.
  </p>
  <p>
    The link is generated using
    <code>Iam:PasswordResetBaseUrl</code> on the backend.
  </p>
</section>
```

### 9.11.6 My security settings page

File:  
`apps/iam-portal/src/app/features/auth/my-security-page/my-security-page.component.html`

```html
<section class="page">
  <h1>My Security Settings</h1>
  <p>
    An authenticated employee will use this screen to manage their own
    credentials:
  </p>
  <ul>
    <li>Change password (POST <code>/api/iam/auth/change-password</code>)</li>
    <li>Enable 2FA (POST <code>/api/iam/auth/2fa/enable</code>)</li>
    <li>Disable 2FA (POST <code>/api/iam/auth/2fa/disable</code>)</li>
  </ul>
</section>
```

### 9.11.7 Employees list page (admin)

File:  
`apps/iam-portal/src/app/features/admin/employees/employees-page/employees-page.component.html`

```html
<section class="page">
  <h1>Employees</h1>
  <p>
    This screen will show a grid backed by
    <code>GET /api/iam/admin/employees</code> returning
    <code>PagedResult&lt;EmployeeSummaryDto&gt;</code>.
  </p>
  <p>It will support:</p>
  <ul>
    <li>Paging via <code>pageNumber</code> and <code>pageSize</code></li>
    <li>Search by email or full name via <code>search</code></li>
    <li>An “Include inactive” toggle via <code>includeInactive</code></li>
  </ul>
</section>
```

### 9.11.8 Employee details page (admin)

File:  
`apps/iam-portal/src/app/features/admin/employees/employee-details-page/employee-details-page.component.html`

```html
<section class="page">
  <h1>Employee Details</h1>
  <p>
    This screen will load a single employee using
    <code>GET /api/iam/admin/employee/{id}</code> and allow:
  </p>
  <ul>
    <li>
      Updating core fields via <code>PUT /api/iam/admin/employees/{id}</code>
    </li>
    <li>Activating the user (POST <code>/activate</code>)</li>
    <li>Deactivating the user (POST <code>/deactivate</code>)</li>
  </ul>
  <p>
    In a later chapter we will use the <code>EmployeeDetailsDto</code> model to
    bind and validate this form.
  </p>
</section>
```

---

## 9.12 Wiring Routes in `app.routes.ts`

Now we connect these components to the Angular router.

File: `apps/iam-portal/src/app/app.routes.ts`

```ts
import { Routes } from "@angular/router";

import { LoginPageComponent } from "./features/auth/login-page/login-page.component";
import { TwoFactorPageComponent } from "./features/auth/two-factor-page/two-factor-page.component";
import { EmailConfirmationPageComponent } from "./features/auth/email-confirmation-page/email-confirmation-page.component";
import { ForgotPasswordPageComponent } from "./features/auth/forgot-password-page/forgot-password-page.component";
import { ResetPasswordPageComponent } from "./features/auth/reset-password-page/reset-password-page.component";
import { MySecurityPageComponent } from "./features/auth/my-security-page/my-security-page.component";

import { EmployeesPageComponent } from "./features/admin/employees/employees-page/employees-page.component";
import { EmployeeDetailsPageComponent } from "./features/admin/employees/employee-details-page/employee-details-page.component";

export const routes: Routes = [
  // Auth flows
  { path: "login", component: LoginPageComponent },
  { path: "login/2fa", component: TwoFactorPageComponent },
  { path: "email-confirmation", component: EmailConfirmationPageComponent },
  { path: "forgot-password", component: ForgotPasswordPageComponent },
  { path: "reset-password", component: ResetPasswordPageComponent },

  // Logged-in user security
  { path: "me/security", component: MySecurityPageComponent },

  // Admin employee management
  { path: "admin/employees", component: EmployeesPageComponent },
  { path: "admin/employees/:id", component: EmployeeDetailsPageComponent },

  // Default + wildcard
  { path: "", pathMatch: "full", redirectTo: "login" },
  { path: "**", redirectTo: "login" },
];
```

Ensure `app.config.ts` wires the router and HttpClient:

File: `apps/iam-portal/src/app/app.config.ts`

```ts
import { ApplicationConfig } from "@angular/core";
import { provideRouter } from "@angular/router";
import { provideHttpClient } from "@angular/common/http";
import { routes } from "./app.routes";

export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes), provideHttpClient()],
};
```

---

## 9.13 Configuring the IAM API Base URL (Build-Time)

We want a clean way to point the frontend at the IAM backend without hardcoding URLs in services.

We’ll use Nx’s `define` option to set a build-time constant.

File: `apps/iam-portal/project.json` (excerpt):

```json
{
  "targets": {
    "build": {
      "executor": "@nx/angular:application",
      "options": {
        "outputPath": "dist/apps/iam-portal",
        "index": "apps/iam-portal/src/index.html",
        "main": "apps/iam-portal/src/main.ts",
        "polyfills": ["zone.js"],
        "tsConfig": "apps/iam-portal/tsconfig.app.json",
        "assets": [
          "apps/iam-portal/src/favicon.ico",
          "apps/iam-portal/src/assets"
        ],
        "styles": ["apps/iam-portal/src/styles.scss"],
        "scripts": [],
        "define": {
          "process.env.IAM_API_URL": "\"https://localhost:5001\""
        }
      }
    },
    "serve": {
      "executor": "@nx/angular:dev-server",
      "options": {
        "buildTarget": "iam-portal:build"
      }
    }
  }
}
```

Later, in our Angular services, we’ll read:

```ts
const IAM_API_URL = process.env.IAM_API_URL ?? "https://localhost:5001";
```

We can override this per environment (CI, staging, production) by adjusting the `define` configuration for each build configuration.

---

## 9.14 Frontend Quality Gates: Lint & Unit Tests

We treat the frontend like the backend: nothing merges into `develop` without passing basic quality checks.

From `src/frontend`:

```bash
# Lint IAM portal
npx nx lint iam-portal

# Unit tests for IAM portal
npx nx test iam-portal
```

Initially, Nx generates a simple unit test for `AppComponent`.  
After our refactor, that test needs to be updated.

### 9.14.1 Fixing `app.component.spec.ts` (router dependencies)

Because `AppComponent` uses `RouterOutlet`, we must provide router testing utilities in the spec.

File: `apps/iam-portal/src/app/app.component.spec.ts`

```ts
import { TestBed } from "@angular/core/testing";
import { RouterTestingModule } from "@angular/router/testing";
import { AppComponent } from "./app.component";

describe("AppComponent", () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [
        AppComponent,
        RouterTestingModule, // provides router services used by RouterOutlet
      ],
    }).compileComponents();
  });

  it("should create the app", () => {
    const fixture = TestBed.createComponent(AppComponent);
    const app = fixture.componentInstance;
    expect(app).toBeTruthy();
  });

  it("should render IAM subtitle in header", () => {
    const fixture = TestBed.createComponent(AppComponent);
    fixture.detectChanges();

    const compiled = fixture.nativeElement as HTMLElement;
    const subtitle = compiled.querySelector(
      ".iam-header__subtitle"
    )?.textContent;

    expect(subtitle).toContain("Employee IAM Portal");
  });
});
```

Re-run:

```bash
npx nx test iam-portal
```

You should see the tests passing without `ActivatedRoute` errors.

---

## 9.15 E2E Tests: Aligning With the IAM Portal

Nx also scaffolds an E2E test (e.g. with Cypress) that expects a “Welcome” heading on the starter page. That no longer matches our app.

We update the E2E spec to:

- Visit `/`
- Confirm redirect to `/login`
- Assert that the login page shows `<h1>Login</h1>`

Open the E2E spec:

File: `apps/iam-portal-e2e/src/e2e/app.cy.ts`

```ts
describe("iam-portal", () => {
  beforeEach(() => {
    cy.visit("/");
  });

  it("should redirect to login page and show login title", () => {
    cy.url().should("include", "/login");
    cy.contains("h1", "Login");
  });
});
```

Run the E2E tests:

```bash
npx nx e2e iam-portal-e2e
```

You should now see the spec passing with no “Welcome” assertion error.

---

## 9.16 CI Overview: Adding the Frontend to the Pipeline

We will fully wire up CI/CD in a later chapter, but at this stage, our pipeline should at least:

- Install Node dependencies under `src/frontend`.
- Run:

  - `npx nx lint iam-portal`
  - `npx nx test iam-portal`
  - `npx nx build iam-portal --configuration=production`

Conceptually, a GitHub Actions job could look like:

```yaml
frontend:
  runs-on: ubuntu-latest

  steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Node
      uses: actions/setup-node@v4
      with:
        node-version: "22"

    - name: Install frontend deps
      working-directory: src/frontend
      run: npm ci

    - name: Lint iam-portal
      working-directory: src/frontend
      run: npx nx lint iam-portal

    - name: Test iam-portal
      working-directory: src/frontend
      run: npx nx test iam-portal --configuration=ci

    - name: Build iam-portal
      working-directory: src/frontend
      run: npx nx build iam-portal --configuration=production
```

The idea is simple:

> A bank does not merge frontend code that doesn’t build, doesn’t lint, or doesn’t pass tests.

Treat the Angular/Nx layer with the same seriousness as the .NET layer.

---

## 9.17 Local Sanity Check & Merge

Before closing this chapter:

1. **Serve the app**

   From `src/frontend`:

   ```bash
   npx nx serve iam-portal
   ```

   Visit:

   - `/login`
   - `/login/2fa`
   - `/email-confirmation`
   - `/forgot-password`
   - `/reset-password`
   - `/me/security`
   - `/admin/employees`
   - `/admin/employees/123`

   All routes should show the placeholder pages we defined, inside the IAM shell.

2. **Run lint, unit tests, and E2E**

   ```bash
   npx nx lint iam-portal
   npx nx test iam-portal
   npx nx e2e iam-portal-e2e
   ```

3. **Commit and push**

   ```bash
   git status
   git add .
   git commit -m "feat(ch09): set up Angular Nx IAM portal shell, routes and tests"
   git push origin feature/ch09-frontend-iam-portal-setup
   ```

4. **Open a pull request** from  
   `feature/ch09-frontend-iam-portal-setup` → `develop`, and merge after review.

At this point you have:

- A clean **Angular 21 + Nx** workspace under `src/frontend`.
- An **IAM Employee Portal** app with a professional folder structure.
- A route skeleton that mirrors the IAM backend contract.
- Frontend lint, unit tests, and E2E tests aligned with the new app shell.

---

## 9.18 What’s Next: Visual Design & Real IAM Flows

In the next chapter we will:

- Define a **UI design system** for Alvor Bank:
  - Color palette
  - Typography
  - Spacing and layout tokens
  - Reusable layout components (shell, page headers, tables, forms, buttons)
- Apply that system to the IAM portal so it looks like a **professional banking system**, not a starter app.
- Start implementing real IAM flows:
  - Login + 2FA
  - JWT handling (storage + interceptors)
  - Auth guards and role-aware navigation
  - Employees grid and details screens bound to `IamService`.

From this point on, every UI we build — internal portals and customer online banking — will reuse the same visual system and component patterns.
