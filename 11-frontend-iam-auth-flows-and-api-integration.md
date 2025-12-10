# Chapter 11 — Implementing IAM Auth Flows in Angular

In this chapter we finally **wire the IAM UI to the real backend**:

- Implement the **login + 2FA** flow against `IamService`.
- Implement **forgot password, reset password, email confirmation**.
- Implement **“My Security”** (change password, enable/disable 2FA).
- Add **JWT handling**, an **HTTP interceptor**, and basic **route guards**.
- Extend unit tests and E2E tests so the IAM portal behaves like a real banking system.

---

## 11.1 Goals & Scope

In the previous frontend chapters:

- **Chapter 9** gave us the IAM Employee Portal shell and routing, built with Angular 21 + Nx.
- **Chapter 10** defined the **Alvor Bank UI Design System** (design tokens, layout patterns, and CSS primitives).

In this chapter we move from **placeholder screens** to a **fully functional IAM authentication experience** wired to the real `IamService` backend.

### 11.1.1 What we will implement

From the perspective of a bank employee using the IAM portal, we will:

- Implement the full **login experience**:
  - Step 1: email + password (`POST /api/iam/auth/login`).
  - Step 2 (optional): 2FA code (`POST /api/iam/auth/2fa/verify`) when `requiresTwoFactor = true`.
- Implement the **password lifecycle flows**:
  - Forgot password (`POST /api/iam/auth/forgot-password`).
  - Reset password via emailed token (`POST /api/iam/auth/reset-password`).
  - Change password while logged in (`POST /api/iam/auth/change-password`).
- Implement **email confirmation**:
  - Confirm email via link (`GET /api/iam/auth/confirm-email`).
- Implement the **“My Security”** page:
  - Change password.
  - Enable/disable 2FA (`POST /api/iam/auth/2fa/enable` / `disable`).
- Put a minimal **session layer** in place:
  - Store and expose the current **JWT access token**.
  - Attach the token to outgoing HTTP calls via an **HTTP interceptor**.
  - Use **route guards** to protect authenticated-only routes (e.g. `/me/security`).

All of this must fit into the standards we have already established:

- Clean file/folder structure under `apps/iam-portal/src/app/core/...`.
- Models in their own files (e.g. `auth/models/login-request.model.ts`).
- Services, guards, and interceptors generated via **Nx CLI commands**, not by ad-hoc file creation.
- Consistent use of the **Alvor Bank design system**:
  - `.bs-btn`, `.bs-form`, `.bs-alert`, `.bs-tag`, `.page`, etc.

### 11.1.2 What we are _not_ implementing yet

To keep the chapter focused, we explicitly defer:

- **Admin employees management UI**:
  - Employees grid.
  - Employee details/edit.
  - Activate/deactivate flows.
- **Centralized app-wide state management**:
  - No NgRx, global signals store, or query libraries yet.
  - We start with a focused `AuthStateService` that manages only authentication concerns.
- **Multi-app shared UI libraries**:
  - Tokens and CSS primitives still live inside `iam-portal`.
  - We will extract them into `libs/ui/...` later when we add the customer-facing portal.

These decisions keep the scope tight: by the end of this chapter, the IAM portal behaves like a **real banking IAM front-end**, with production-style auth flows, but without mixing in admin grids or complex state concerns yet.

The next sections will:

1. Recap the backend auth contract from a UI perspective.
2. Set up a dedicated branch for this chapter’s work.
3. Use Nx generators to scaffold:
   - Auth models.
   - Auth API service.
   - Auth state service.
   - JWT interceptor.
   - Route guards.
4. Incrementally turn each IAM page (login, 2FA, forgot/reset password, email confirmation, My Security) into a real, tested flow.

---

## 11.2 Backend Contract Recap (UI Perspective)

Before we write any Angular code, we need a clear view of **what the IAM backend expects and returns**.  
This section is a quick, UI-focused summary of the `IamService` auth endpoints designed earlier.

> If you followed the backend chapters, nothing here is new — we’re just rephrasing it from the perspective of the Angular frontend.

---

### 11.2.1 Core auth endpoints

These are the endpoints the IAM portal will call in this chapter:

| Use case                | Method & URL                             | Request body (simplified)          | Response (simplified) |
| ----------------------- | ---------------------------------------- | ---------------------------------- | --------------------- |
| Login (step 1)          | `POST /api/iam/auth/login`               | `{ email, password }`              | `LoginResult`         |
| Verify 2FA (step 2)     | `POST /api/iam/auth/2fa/verify`          | `{ userId, code }`                 | `LoginResult`         |
| Confirm email           | `GET /api/iam/auth/confirm-email`        | Query: `userId`, `token`           | `204 No Content`      |
| Resend confirmation     | `POST /api/iam/auth/resend-confirmation` | `{ email }`                        | `{ sent: boolean }`   |
| Forgot password         | `POST /api/iam/auth/forgot-password`     | `{ email }`                        | `{ sent: boolean }`   |
| Reset password          | `POST /api/iam/auth/reset-password`      | `{ email, token, newPassword }`    | `204 No Content`      |
| Change password         | `POST /api/iam/auth/change-password`     | `{ currentPassword, newPassword }` | `204 No Content`      |
| Enable 2FA (logged-in)  | `POST /api/iam/auth/2fa/enable`          | `{ currentPassword }`              | `204 No Content`      |
| Disable 2FA (logged-in) | `POST /api/iam/auth/2fa/disable`         | `{ currentPassword }`              | `204 No Content`      |

A few important patterns:

- **All authentication is email-based** (no username).
- For security reasons, some responses are deliberately generic:
  - `forgot-password` and `resend-confirmation` respond with `{ sent: true }` even if the email does not exist.
- Email confirmation and password reset use **URL tokens** generated by the backend:
  - `Iam:EmailConfirmationBaseUrl` → frontend route for confirm email.
  - `Iam:PasswordResetBaseUrl` → frontend route for reset password.

---

### 11.2.2 The `LoginResult` contract

Both login and 2FA verification return a **`LoginResult`** object that drives the UI flow:

- For **login (step 1)** (`POST /api/iam/auth/login`):

  - If the user does **not** require 2FA:
    - `requiresTwoFactor = false`
    - `accessToken` is populated
    - `userId` may be null
  - If the user **does** require 2FA:
    - `requiresTwoFactor = true`
    - `userId` is populated (we’ll pass this to the 2FA screen)
    - `accessToken = null` (or absent)

- For **2FA verification (step 2)** (`POST /api/iam/auth/2fa/verify`):
  - On success:
    - `requiresTwoFactor = false`
    - `accessToken` is populated
    - `userId` may be null at this point
  - On failure:
    - API returns a 4xx error, and the UI must show a generic “invalid/expired code” message.

From the frontend’s perspective, we can think of `LoginResult` as:

```ts
// apps/iam-portal/src/app/core/auth/models/login-result.model.ts (to be created later)
//
// NOTE: This is just a conceptual view here; we'll generate models properly in 11.4.
export interface LoginResult {
  requiresTwoFactor: boolean;
  userId?: string | null;
  accessToken?: string | null;
  expiresAt?: string | null; // ISO timestamp
}
```

The key logic in the UI:

- If `requiresTwoFactor === false` and we have an `accessToken`:
  - Store token in our `AuthStateService`.
  - Navigate to an authenticated “home” route.
- If `requiresTwoFactor === true`:
  - Store `userId` in a **temporary 2FA session** (in memory, not localStorage).
  - Navigate to `/login/2fa`.

---

### 11.2.3 Security & error response behavior

The IAM backend is designed to **avoid leaking information**.  
The UI must respect this and show **generic, security-friendly messages**, not raw technical errors.

A few important behaviors:

- **Login errors**

  - Wrong email or password.
  - Inactive or deactivated user.
  - Email not confirmed.
  - Locked out due to too many failed attempts.
  - → Typically surface as `400`/`401` with a generic message.
  - UI should show something like:
    - “Invalid email or password.” or
    - “Your account is locked. Please try again later or contact support.”
  - We will standardize this in the `AuthApiService` and components.

- **2FA errors**

  - Wrong code.
  - Expired code.
  - Missing or invalid `userId`.
  - → Also surface as 4xx errors.
  - UI should show a generic message such as:
    - “Invalid or expired verification code. Please try again.”

- **Forgot password / resend confirmation**

  - Always return `{ sent: true }` from the backend.
  - UI always shows:
    - “If this email exists, we sent you an email with further instructions.”

- **Change password / enable/disable 2FA**
  - On success: `204 No Content`.
  - On failure (invalid current password, business rule):
    - 4xx error; the UI shows a clear but generic message (“Current password is incorrect.”).

These rules are important because they influence:

- How we design our **error handling pipeline** in Angular.
- How we present messages in a way that is both **user-friendly** and **safe** for a banking IAM context.

---

### 11.2.4 How the frontend will organize IAM auth code

To keep the IAM portal **bank-grade organized**, the next sections will:

- Use **Nx generators** to create a dedicated **auth core area**:

  - `apps/iam-portal/src/app/core/auth/models/...`
  - `apps/iam-portal/src/app/core/auth/services/...`
  - `apps/iam-portal/src/app/core/auth/guards/...`
  - `apps/iam-portal/src/app/core/auth/interceptors/...`

- Keep each TypeScript model in its **own file** under `models/`.
- Centralize all HTTP calls in an `AuthApiService`.
- Separate **API calls** from **state management** with an `AuthStateService`.

We’ll start wiring this up in **Section 11.3 (branch) and 11.4 (auth infrastructure)** using **Nx CLI commands**, not manual file creation.

---

## 11.3 Branch Strategy for Auth Flows

As with every significant chapter, we treat the IAM auth implementation as a **proper feature** in a real banking project, not a quick experiment.

That means:

- A dedicated **feature branch** for Chapter 11.
- All work scoped to that branch.
- CI-quality checks (lint, unit tests, E2E) before merging into `develop`.
- No direct commits to `develop` or `main`.

We keep the same repository structure:

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

---

### 11.3.1 Opening the Chapter 11 feature branch

From the repository root:

```bash
cd digital-banking-suite

# Make sure you're up to date with the latest develop
git checkout develop
git pull origin develop

# Create a dedicated feature branch for this chapter
git checkout -b feature/ch11-frontend-iam-auth-flows
```

We will stay on this branch for all IAM auth work until the end of the chapter.

---

### 11.3.2 Frontend working directory

All frontend work happens under `src/frontend`:

```bash
cd digital-banking-suite/src/frontend
```

From here, the main Nx commands you’ll use during this chapter are:

```bash
# Serve IAM portal locally
npx nx serve iam-portal

# Lint IAM portal
npx nx lint iam-portal

# Unit tests for IAM portal
npx nx test iam-portal

# E2E tests for IAM portal
npx nx e2e iam-portal-e2e
```

We will also use **Nx generators** to scaffold services, models (via libraries or shared code), and other building blocks rather than creating files manually.

---

### 11.3.3 Closing the chapter (sanity check & merge)

We will come back to this at the end, but the closing ritual is always the same:

1. From `src/frontend`, ensure all checks pass:

   ```bash
   npx nx lint iam-portal
   npx nx test iam-portal
   npx nx e2e iam-portal-e2e
   ```

2. From the repo root, commit and push:

   ```bash
   cd digital-banking-suite

   git status
   git add .
   git commit -m "feat(ch11): implement IAM auth flows (login, 2FA, password, security)"
   git push origin feature/ch11-frontend-iam-auth-flows
   ```

3. Open a Pull Request:

   - **From**: `feature/ch11-frontend-iam-auth-flows`
   - **To**: `develop`
   - Merge only after CI passes and the changes are reviewed.

We will explicitly remind you of these steps in **Section 11.13**, once all IAM auth functionality is in place.

Next, we start laying down the **auth infrastructure** in the frontend: models, services, state, interceptor, and guards — all generated via Nx and organized in a `core/auth` area suitable for a banking-grade application.

---

## 11.4 IAM Auth Infrastructure in the Frontend

Now we set up the **core auth layer** for the IAM portal — the pieces that every auth screen (login, 2FA, password flows, My Security) will rely on:

- Strongly typed **models** for all requests and responses.
- An `AuthApiService` that talks to `IamService`.
- An `AuthStateService` that manages the current session (access token).
- A **JWT interceptor** that attaches the token to outgoing HTTP requests.

Everything will live under a dedicated `core/auth` area inside the IAM app so it feels like a real banking front-end, not a demo.

Target structure:

```
apps/iam-portal/src/app/core/
  auth/
    models/
      login-request.model.ts
      login-result.model.ts
      two-factor-verify-request.model.ts
      forgot-password-request.model.ts
      reset-password-request.model.ts
      change-password-request.model.ts
      current-password-request.model.ts
      email-action-response.model.ts
      index.ts
    services/
      auth-api.service.ts
      auth-state.service.ts
    interceptors/
      auth-token.interceptor.ts
```

We will:

- Use **plain file creation** for models (Nx doesn’t have a dedicated “model” generator).
- Use **Nx generators** for the services and interceptor so the structure is consistent with a real Angular/Nx banking project.

---

### 11.4.1 Auth models (TypeScript)

From `src/frontend`, create the folders:

```bash
cd digital-banking-suite/src/frontend

mkdir -p apps/iam-portal/src/app/core/auth/models
mkdir -p apps/iam-portal/src/app/core/auth/services
mkdir -p apps/iam-portal/src/app/core/auth/interceptors
```

Now add the model files — **one interface per file**.

1. `LoginRequest`

   File: `apps/iam-portal/src/app/core/auth/models/login-request.model.ts`

   ```ts
   export interface LoginRequest {
     email: string;
     password: string;
   }
   ```

2. `LoginResult`

   File: `apps/iam-portal/src/app/core/auth/models/login-result.model.ts`

   ```ts
   export interface LoginResult {
     requiresTwoFactor: boolean;
     userId?: string | null;
     accessToken?: string | null;
     expiresAt?: string | null; // ISO-8601 timestamp
   }
   ```

3. `TwoFactorVerifyRequest`

   File: `apps/iam-portal/src/app/core/auth/models/two-factor-verify-request.model.ts`

   ```ts
   export interface TwoFactorVerifyRequest {
     userId: string;
     code: string;
   }
   ```

4. `ForgotPasswordRequest`

   File: `apps/iam-portal/src/app/core/auth/models/forgot-password-request.model.ts`

   ```ts
   export interface ForgotPasswordRequest {
     email: string;
   }
   ```

5. `ResetPasswordRequest`

   File: `apps/iam-portal/src/app/core/auth/models/reset-password-request.model.ts`

   ```ts
   export interface ResetPasswordRequest {
     email: string;
     token: string;
     newPassword: string;
   }
   ```

6. `ChangePasswordRequest`

   File: `apps/iam-portal/src/app/core/auth/models/change-password-request.model.ts`

   ```ts
   export interface ChangePasswordRequest {
     currentPassword: string;
     newPassword: string;
   }
   ```

7. `CurrentPasswordRequest` (for enable/disable 2FA)

   File: `apps/iam-portal/src/app/core/auth/models/current-password-request.model.ts`

   ```ts
   export interface CurrentPasswordRequest {
     currentPassword: string;
   }
   ```

8. `EmailActionResponse` (`forgot-password` / `resend-confirmation`)

   File: `apps/iam-portal/src/app/core/auth/models/email-action-response.model.ts`

   ```ts
   export interface EmailActionResponse {
     sent: boolean;
   }
   ```

9. Optional `index.ts` barrel for cleaner imports

   File: `apps/iam-portal/src/app/core/auth/models/index.ts`

   ```ts
   export * from "./login-request.model";
   export * from "./login-result.model";
   export * from "./two-factor-verify-request.model";
   export * from "./forgot-password-request.model";
   export * from "./reset-password-request.model";
   export * from "./change-password-request.model";
   export * from "./current-password-request.model";
   export * from "./email-action-response.model";
   ```

This keeps each model in its own file (bank-grade maintainability) while still allowing a single import path (`../models`) where convenient.

---

### 11.4.2 Auth API service

The `AuthApiService` is the **only place** that knows the exact URLs of IAM auth endpoints.  
We generate it with Nx so it lands under the right folder.

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:service \
  apps/iam-portal/src/app/core/auth/services/auth-api/auth-api \
  --flat=true
```

Nx creates:

- `apps/iam-portal/src/app/core/auth/services/auth-api.service.ts`
- `apps/iam-portal/src/app/core/auth/services/auth-api.service.spec.ts`

Update `auth-api.service.ts` to:

```ts
import { Injectable } from "@angular/core";
import { HttpClient, HttpParams } from "@angular/common/http";
import { Observable } from "rxjs";

import {
  LoginRequest,
  LoginResult,
  TwoFactorVerifyRequest,
  ForgotPasswordRequest,
  ResetPasswordRequest,
  ChangePasswordRequest,
  CurrentPasswordRequest,
  EmailActionResponse,
} from "../models";

// IAM_API_URL is injected at build time via Nx "define" config in project.json
const IAM_API_URL =
  (process.env["IAM_API_URL"] as string | undefined) ??
  "https://localhost:5001";

@Injectable({
  providedIn: "root",
})
export class AuthApiService {
  private readonly baseUrl = `${IAM_API_URL}/api/iam/auth`;

  constructor(private readonly http: HttpClient) {}

  login(request: LoginRequest): Observable<LoginResult> {
    return this.http.post<LoginResult>(`${this.baseUrl}/login`, request);
  }

  verifyTwoFactor(request: TwoFactorVerifyRequest): Observable<LoginResult> {
    return this.http.post<LoginResult>(`${this.baseUrl}/2fa/verify`, request);
  }

  confirmEmail(userId: string, token: string): Observable<void> {
    const params = new HttpParams().set("userId", userId).set("token", token);

    return this.http.get<void>(`${this.baseUrl}/confirm-email`, { params });
  }

  resendConfirmation(
    request: ForgotPasswordRequest
  ): Observable<EmailActionResponse> {
    return this.http.post<EmailActionResponse>(
      `${this.baseUrl}/resend-confirmation`,
      request
    );
  }

  forgotPassword(
    request: ForgotPasswordRequest
  ): Observable<EmailActionResponse> {
    return this.http.post<EmailActionResponse>(
      `${this.baseUrl}/forgot-password`,
      request
    );
  }

  resetPassword(request: ResetPasswordRequest): Observable<void> {
    return this.http.post<void>(`${this.baseUrl}/reset-password`, request);
  }

  changePassword(request: ChangePasswordRequest): Observable<void> {
    return this.http.post<void>(`${this.baseUrl}/change-password`, request);
  }

  enableTwoFactor(request: CurrentPasswordRequest): Observable<void> {
    return this.http.post<void>(`${this.baseUrl}/2fa/enable`, request);
  }

  disableTwoFactor(request: CurrentPasswordRequest): Observable<void> {
    return this.http.post<void>(`${this.baseUrl}/2fa/disable`, request);
  }
}
```

This keeps all IAM auth HTTP contracts in a single, testable service.

---

### 11.4.3 Auth state service

`AuthStateService` is our **session layer**:

- Holds the current access token in memory.
- Persists it in `localStorage` using a bank-specific key.
- Exposes:
  - `accessToken$` — observable of the token.
  - `isAuthenticated$` — observable of a boolean.
  - `getAccessToken()` — synchronous accessor (for the interceptor).

Generate the service via Nx:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:service \
  apps/iam-portal/src/app/core/auth/services/auth-state/auth-state \
  --flat=true
```

Nx creates:

- `apps/iam-portal/src/app/core/auth/services/auth-state.service.ts`
- `apps/iam-portal/src/app/core/auth/services/auth-state.service.spec.ts`

Update `auth-state.service.ts` to:

```ts
import { Injectable } from "@angular/core";
import { BehaviorSubject, Observable, map } from "rxjs";

@Injectable({
  providedIn: "root",
})
export class AuthStateService {
  private static readonly ACCESS_TOKEN_KEY = "alvor_iam_access_token";

  private readonly accessTokenSubject = new BehaviorSubject<string | null>(
    this.readInitialToken()
  );

  readonly accessToken$: Observable<string | null> =
    this.accessTokenSubject.asObservable();

  readonly isAuthenticated$: Observable<boolean> = this.accessToken$.pipe(
    map((token) => !!token)
  );

  constructor() {}

  private readInitialToken(): string | null {
    if (typeof window === "undefined") {
      return null;
    }

    try {
      return window.localStorage.getItem(AuthStateService.ACCESS_TOKEN_KEY);
    } catch {
      // In a real bank we might log this via a monitoring service.
      return null;
    }
  }

  getAccessToken(): string | null {
    return this.accessTokenSubject.value;
  }

  setSession(accessToken: string | null): void {
    this.accessTokenSubject.next(accessToken);

    if (typeof window === "undefined") {
      return;
    }

    try {
      if (accessToken) {
        window.localStorage.setItem(
          AuthStateService.ACCESS_TOKEN_KEY,
          accessToken
        );
      } else {
        window.localStorage.removeItem(AuthStateService.ACCESS_TOKEN_KEY);
      }
    } catch {
      // Intentionally swallowed; failure to persist should not crash the app.
    }
  }

  clearSession(): void {
    this.setSession(null);
  }
}
```

This service is intentionally **UI-agnostic**: it doesn’t know about routes, only about tokens. Components and guards decide where to redirect on login/logout.

---

### 11.4.4 JWT interceptor

The **auth token interceptor** attaches the JWT to outgoing HTTP requests targeting the IAM backend.

Generate it via Nx:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:interceptor \
  apps/iam-portal/src/app/core/auth/interceptors/auth-token/auth-token \
  --flat=true
```

Nx creates:

- `apps/iam-portal/src/app/core/auth/interceptors/auth-token.interceptor.ts`
- `apps/iam-portal/src/app/core/auth/interceptors/auth-token.interceptor.spec.ts`

Update `auth-token.interceptor.ts` to:

```ts
import { inject } from "@angular/core";
import { HttpInterceptorFn } from "@angular/common/http";

import { AuthStateService } from "../services/auth-state.service";

const IAM_API_URL =
  (process.env["IAM_API_URL"] as string | undefined) ??
  "https://localhost:5001";

export const authTokenInterceptor: HttpInterceptorFn = (req, next) => {
  const authState = inject(AuthStateService);
  const token = authState.getAccessToken();

  if (!token) {
    return next(req);
  }

  // If later we call multiple backends, we can restrict this to IAM URLs only.
  if (IAM_API_URL && !req.url.startsWith(IAM_API_URL)) {
    return next(req);
  }

  const authReq = req.clone({
    setHeaders: {
      Authorization: `Bearer ${token}`,
    },
  });

  return next(authReq);
};
```

Now register the interceptor in the IAM app config.

File: `apps/iam-portal/src/app/app.config.ts`

```ts
import { ApplicationConfig } from "@angular/core";
import { provideRouter } from "@angular/router";
import { provideHttpClient, withInterceptors } from "@angular/common/http";

import { routes } from "./app.routes";
import { authTokenInterceptor } from "./core/auth/interceptors/auth-token.interceptor";

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([authTokenInterceptor])),
  ],
};
```

At this point, the IAM portal has a **bank-grade auth infrastructure**:

- Models organized under `core/auth/models`.
- A single `AuthApiService` for all auth HTTP calls.
- An `AuthStateService` that manages tokens and exposes auth state.
- A JWT interceptor that automatically injects `Authorization: Bearer ...` for IAM API calls.

In the next sections we will:

- Add **route guards** on top of this infrastructure.
- Turn the IAM screens (login, 2FA, forgot/reset password, email confirmation, My Security) into fully functional flows that use these services.

---

## 11.5 Route Guards & Auth-Only Areas

With the auth infrastructure in place (models, `AuthApiService`, `AuthStateService`, interceptor), we now need **route guards** so that:

- Auth-only pages (like **My Security**) are not accessible without a valid session.
- Anonymous-only pages (like **Login** and **2FA**) cannot be accessed once the user is already logged in.

We will implement two guards:

- `authGuard` — protects routes that require authentication.
- `guestGuard` — protects routes that should only be accessible when **not** authenticated.

All guards go under:

```
apps/iam-portal/src/app/core/auth/guards/
  auth.guard.ts
  guest.guard.ts
```

We’ll use **functional guards** (`CanActivateFn`) and generate them via Nx.

---

### 11.5.1 Creating the guards with Nx

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

# AuthGuard: protects authenticated routes
npx nx g @nx/angular:guard \
  apps/iam-portal/src/app/core/auth/guards/auth \
  --flat=true --functional=true

# GuestGuard: protects anonymous-only routes (login, 2FA, etc.)
npx nx g @nx/angular:guard \
  apps/iam-portal/src/app/core/auth/guards/guest \
  --flat=true --functional=true
```

Nx generates:

- `apps/iam-portal/src/app/core/auth/guards/auth.guard.ts`
- `apps/iam-portal/src/app/core/auth/guards/auth.guard.spec.ts`
- `apps/iam-portal/src/app/core/auth/guards/guest.guard.ts`
- `apps/iam-portal/src/app/core/auth/guards/guest.guard.spec.ts`

We now update the implementations.

---

### 11.5.2 Implementing `authGuard` (authenticated-only)

`authGuard` ensures that a route can only be activated when the user has a valid access token.  
If not authenticated, it redirects to `/login` and (optionally) preserves a `returnUrl` query parameter.

File: `apps/iam-portal/src/app/core/auth/guards/auth.guard.ts`

```ts
import { inject } from "@angular/core";
import { CanActivateFn, Router, UrlTree } from "@angular/router";

import { AuthStateService } from "../services/auth-state.service";

export const authGuard: CanActivateFn = (route, state): boolean | UrlTree => {
  const authState = inject(AuthStateService);
  const router = inject(Router);

  const token = authState.getAccessToken();

  if (token) {
    // User is authenticated, allow route activation
    return true;
  }

  // Not authenticated — redirect to login, preserving returnUrl
  return router.createUrlTree(["/login"], {
    queryParams: { returnUrl: state.url },
  });
};
```

Notes:

- We intentionally use the **synchronous** `getAccessToken()` so the guard is simple and deterministic.
- `returnUrl` will be used later (optional) to navigate back after a successful login.

---

### 11.5.3 Implementing `guestGuard` (anonymous-only)

`guestGuard` is the inverse: it prevents authenticated users from going back to login/2FA pages.

File: `apps/iam-portal/src/app/core/auth/guards/guest.guard.ts`

```ts
import { inject } from "@angular/core";
import { CanActivateFn, Router, UrlTree } from "@angular/router";

import { AuthStateService } from "../services/auth-state.service";

export const guestGuard: CanActivateFn = (): boolean | UrlTree => {
  const authState = inject(AuthStateService);
  const router = inject(Router);

  const token = authState.getAccessToken();

  if (!token) {
    // User is not authenticated — allow access to login, 2FA, etc.
    return true;
  }

  // Already authenticated — send them to a sensible "home" route
  return router.createUrlTree(["/me/security"]);
};
```

Notes:

- For now our “authenticated home” is `/me/security`.  
  Later, if we add a dashboard or employees overview, we can switch the target route in one place.

---

### 11.5.4 Applying guards to IAM routes

Now we wire the guards into the router configuration.

File: `apps/iam-portal/src/app/app.routes.ts`

> Reminder: you may have already refactored your component class names (e.g. `LoginPage` instead of `LoginPageComponent`).  
> Adjust the imports below to match your actual class names — the important part here is the **guard usage**.

```ts
import { Routes } from "@angular/router";

import { LoginPage } from "./features/auth/login-page/login-page.component";
import { TwoFactorPage } from "./features/auth/two-factor-page/two-factor-page.component";
import { EmailConfirmationPage } from "./features/auth/email-confirmation-page/email-confirmation-page.component";
import { ForgotPasswordPage } from "./features/auth/forgot-password-page/forgot-password-page.component";
import { ResetPasswordPage } from "./features/auth/reset-password-page/reset-password-page.component";
import { MySecurityPage } from "./features/auth/my-security-page/my-security-page.component";

import { EmployeesPage } from "./features/admin/employees/employees-page/employees-page.component";
import { EmployeeDetailsPage } from "./features/admin/employees/employee-details-page/employee-details-page.component";

import { authGuard } from "./core/auth/guards/auth.guard";
import { guestGuard } from "./core/auth/guards/guest.guard";

export const routes: Routes = [
  // Auth flows (anonymous-only)
  {
    path: "login",
    component: LoginPage,
    canActivate: [guestGuard],
  },
  {
    path: "login/2fa",
    component: TwoFactorPage,
    canActivate: [guestGuard],
  },
  {
    path: "forgot-password",
    component: ForgotPasswordPage,
    canActivate: [guestGuard],
  },
  {
    path: "reset-password",
    component: ResetPasswordPage,
    canActivate: [guestGuard],
  },
  {
    path: "email-confirmation",
    component: EmailConfirmationPage,
    canActivate: [guestGuard],
  },

  // Logged-in user security (authenticated-only)
  {
    path: "me/security",
    component: MySecurityPage,
    canActivate: [authGuard],
  },

  // Admin employee management (will be fleshed out in a later chapter)
  {
    path: "admin/employees",
    component: EmployeesPage,
    canActivate: [authGuard],
  },
  {
    path: "admin/employees/:id",
    component: EmployeeDetailsPage,
    canActivate: [authGuard],
  },

  // Default + wildcard
  {
    path: "",
    pathMatch: "full",
    redirectTo: "login",
  },
  {
    path: "**",
    redirectTo: "login",
  },
];
```

If your generated components still end with `Component` (e.g. `LoginPageComponent`), simply adjust the imports and `component:` names accordingly; the guards remain the same.

---

### 11.5.5 Quick sanity check

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
```

Then run the app:

```bash
npx nx serve iam-portal
```

Manual check:

1. Navigate to `/me/security` without being logged in:
   - You should be redirected to `/login`.
2. After you later implement login and set a token:
   - Navigating to `/login` should redirect to `/me/security`.

With guards in place, we now have clear **auth boundaries** in the UI.  
Next, we’ll start turning the **Login page** into a real, reactive form that calls `AuthApiService` and uses `AuthStateService` to create a bank-grade sign-in experience.

---

## 11.6 Implementing Login Page (Step 1)

Now we turn the **Login page** from a placeholder into a real banking-grade sign-in screen:

- Reactive form with validation.
- Calls to `AuthApiService.login`.
- Session handling via `AuthStateService`.
- Branching into **2FA** when `requiresTwoFactor = true`.
- Clean error handling and UI using the Alvor Bank design system.

We’ll also introduce a small `TwoFactorSessionService` to keep the pending `userId` between **Login** and **2FA**.

> If you already generated the `LoginPage` component in Chapter 9, you’ll be **updating** it here.  
> If not, we include the Nx generator command so you can create it now.

---

### 11.6.1 TwoFactorSessionService (shared between Login & 2FA)

We never want to persist the pending 2FA `userId` in localStorage; it should live only in memory while the user is in the browser session.

Create the service with Nx:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:service \
  apps/iam-portal/src/app/core/auth/services/two-factor-session/two-factor-session \
  --flat=true
```

This creates:

- `apps/iam-portal/src/app/core/auth/services/two-factor-session.service.ts`
- `apps/iam-portal/src/app/core/auth/services/two-factor-session.service.spec.ts`

Update `two-factor-session.service.ts`:

```ts
import { Injectable } from "@angular/core";

@Injectable({
  providedIn: "root",
})
export class TwoFactorSessionService {
  private pendingUserId: string | null = null;

  setPendingUserId(userId: string): void {
    this.pendingUserId = userId;
  }

  getPendingUserId(): string | null {
    return this.pendingUserId;
  }

  hasPendingUserId(): boolean {
    return !!this.pendingUserId;
  }

  clear(): void {
    this.pendingUserId = null;
  }
}
```

The login page will set `pendingUserId` when the backend indicates that 2FA is required; the 2FA page will read and clear it.

---

### 11.6.2 Generating (or locating) the LoginPage component

If you did **Chapter 9**, you should already have:

- `apps/iam-portal/src/app/features/auth/login-page/login-page.component.ts`
- `login-page.component.html`
- `login-page.component.scss`

If not, generate it now with Nx (standalone component, external HTML/SCSS, OnPush):

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/auth/login-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

This will create the files under:

- `apps/iam-portal/src/app/features/auth/login-page/`

We will now replace the default implementation with a real login page.

---

### 11.6.3 LoginPage component class (TypeScript)

File:  
`apps/iam-portal/src/app/features/auth/login-page/login-page.component.ts`

Replace the content with:

```ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { FormBuilder, ReactiveFormsModule, Validators } from "@angular/forms";
import { ActivatedRoute, Router, RouterModule } from "@angular/router";
import { finalize } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

import { AuthApiService } from "../../../core/auth/services/auth-api.service";
import { AuthStateService } from "../../../core/auth/services/auth-state.service";
import { TwoFactorSessionService } from "../../../core/auth/services/two-factor-session.service";
import { LoginRequest } from "../../../core/auth/models";

@Component({
  standalone: true,
  selector: "bs-login-page",
  templateUrl: "./login-page.component.html",
  styleUrl: "./login-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
})
export class LoginPage {
  private readonly fb = inject(FormBuilder);
  private readonly authApi = inject(AuthApiService);
  private readonly authState = inject(AuthStateService);
  private readonly twoFactorSession = inject(TwoFactorSessionService);
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);
  private readonly destroyRef = inject(DestroyRef);

  readonly form = this.fb.nonNullable.group({
    email: ["", [Validators.required, Validators.email]],
    password: ["", [Validators.required, Validators.minLength(8)]],
  });

  isSubmitting = false;
  formError: string | null = null;

  get email() {
    return this.form.controls.email;
  }

  get password() {
    return this.form.controls.password;
  }

  onSubmit(): void {
    if (this.form.invalid || this.isSubmitting) {
      this.form.markAllAsTouched();
      return;
    }

    this.isSubmitting = true;
    this.formError = null;

    const payload: LoginRequest = this.form.getRawValue();

    this.authApi
      .login(payload)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isSubmitting = false;
        })
      )
      .subscribe({
        next: (result) => {
          if (result.requiresTwoFactor) {
            if (!result.userId) {
              // Defensive: backend should always send userId when 2FA is required
              this.formError =
                "We could not complete your sign-in. Please try again.";
              return;
            }

            this.twoFactorSession.setPendingUserId(result.userId);
            this.router.navigate(["/login/2fa"]);
            return;
          }

          if (!result.accessToken) {
            this.formError =
              "We could not complete your sign-in. Please try again.";
            return;
          }

          this.authState.setSession(result.accessToken);

          const returnUrl = this.route.snapshot.queryParamMap.get("returnUrl");

          const target =
            returnUrl && returnUrl !== "/login" ? returnUrl : "/me/security";

          this.router.navigateByUrl(target);
        },
        error: () => {
          // In a real bank, we'd inspect error codes and log details,
          // but the user-facing message stays generic.
          this.formError =
            "Invalid email or password. If the problem persists, contact support.";
        },
      });
  }
}
```

Key points:

- We use Angular’s `takeUntilDestroyed` + `DestroyRef` so subscriptions are automatically cleaned up.
- All routing logic stays in the component, not inside services.
- Error messages are **generic**, which is important for security.

---

### 11.6.4 LoginPage template (HTML)

We now apply the **Alvor Bank design system**: `page`, `bs-form`, `bs-form-field`, `bs-alert`, `bs-btn`, etc.

File:  
`apps/iam-portal/src/app/features/auth/login-page/login-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">Sign in to Alvor Bank</h1>
      <p class="page-header__subtitle">Employee access to the IAM Portal.</p>
    </div>
  </header>

  <form [formGroup]="form" class="bs-form" (ngSubmit)="onSubmit()">
    <!-- Form-level error -->
    <div *ngIf="formError" class="bs-alert bs-alert--danger">
      <div>
        <div class="bs-alert__title">Sign-in failed</div>
        <p class="bs-alert__body">{{ formError }}</p>
      </div>
    </div>

    <!-- Email -->
    <div class="bs-form-field">
      <label
        class="bs-form-field__label bs-form-field__label--required"
        for="email"
      >
        Email
      </label>
      <div class="bs-form-field__control">
        <input
          id="email"
          type="email"
          formControlName="email"
          autocomplete="email"
        />
      </div>
      <p class="bs-form-field__error" *ngIf="email.touched && email.invalid">
        Please enter a valid email address.
      </p>
    </div>

    <!-- Password -->
    <div class="bs-form-field">
      <label
        class="bs-form-field__label bs-form-field__label--required"
        for="password"
      >
        Password
      </label>
      <div class="bs-form-field__control">
        <input
          id="password"
          type="password"
          formControlName="password"
          autocomplete="current-password"
        />
      </div>
      <p
        class="bs-form-field__error"
        *ngIf="password.touched && password.invalid"
      >
        Your password must be at least 8 characters long.
      </p>
    </div>

    <!-- Actions -->
    <div class="login-actions">
      <button
        class="bs-btn bs-btn--primary"
        type="submit"
        [disabled]="isSubmitting"
      >
        <span *ngIf="!isSubmitting">Sign in</span>
        <span *ngIf="isSubmitting">Signing in…</span>
      </button>
    </div>
  </form>

  <div class="login-links">
    <a routerLink="/forgot-password">Forgot your password?</a>
  </div>
</div>
```

This template:

- Uses our design system styles from Chapter 10.
- Keeps the layout clean and banking-grade: minimal, clear, no gimmicks.

---

### 11.6.5 LoginPage styles (SCSS)

We keep page layout consistent with our global `.page` style but narrow it slightly for the login form and add a bit of spacing for the footer link.

File:  
`apps/iam-portal/src/app/features/auth/login-page/login-page.component.scss`

```scss
:host {
  display: block;
}

/* Narrower card for login than generic pages */
.page {
  max-width: 420px;
}

.login-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: var(--bs-space-3);
}

.login-actions .bs-btn {
  min-width: 120px;
}

.login-links {
  margin-top: var(--bs-space-3);
  font-size: var(--bs-font-size-xs);
  text-align: right;
}

.login-links a {
  color: var(--bs-color-primary);
  text-decoration: none;
}

.login-links a:hover {
  text-decoration: underline;
}
```

These styles are intentionally small; most of the visual identity comes from the **global design tokens and utilities** we already defined.

---

### 11.6.6 Sanity check for the Login page

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
npx nx serve iam-portal
```

Manual test:

1. Navigate to `http://localhost:4200/login`.
2. Verify:
   - The page uses the new layout and Alvor Bank design system.
   - Validation messages appear when fields are empty or invalid.
3. Enter a **known dev IAM user** (from backend seeding) and submit:
   - If the user **does not** require 2FA:
     - You should be redirected to `/me/security` (or `returnUrl` if provided).
   - If the user **does** require 2FA:
     - You should be redirected to `/login/2fa`.

With a real login flow now in place, the IAM portal feels like a genuine banking system entry point.  
Next, we will implement the **Two-Factor Verification page** that completes the sign-in flow when 2FA is enabled.

---

## 11.7 Implementing Two-Factor Page (Step 2)

For users with **Two-Factor Authentication enabled**, login is a **two-step** process:

1. Enter email + password on `/login`.
2. Enter the 2FA code on `/login/2fa`.

We already:

- Return `requiresTwoFactor = true` and `userId` from the backend.
- Store the `userId` in `TwoFactorSessionService`.
- Redirect to `/login/2fa`.

Now we implement the **TwoFactorPage**:

- Reactive form to capture the 2FA code.
- Call `AuthApiService.verifyTwoFactor`.
- On success:
  - Store the access token via `AuthStateService`.
  - Clear the pending 2FA session.
  - Navigate to an authenticated route.
- Protect the page from direct access without a pending 2FA session.

---

### 11.7.1 Generate (or locate) the TwoFactorPage component

If you already created `TwoFactorPage` in Chapter 9, you can reuse it and just replace the implementation.

If not, generate it now with Nx:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/auth/two-factor-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

This creates:

- `apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page.component.ts`
- `apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page.component.html`
- `apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page.component.scss`

We will now make this a real 2FA verification screen.

---

### 11.7.2 TwoFactorPage component class (TypeScript)

File:  
`apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page.component.ts`

Replace the content with:

```ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  OnInit,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { FormBuilder, ReactiveFormsModule, Validators } from "@angular/forms";
import { Router, RouterModule } from "@angular/router";
import { finalize } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

import { AuthApiService } from "../../../core/auth/services/auth-api.service";
import { AuthStateService } from "../../../core/auth/services/auth-state.service";
import { TwoFactorSessionService } from "../../../core/auth/services/two-factor-session.service";
import { TwoFactorVerifyRequest } from "../../../core/auth/models";

@Component({
  standalone: true,
  selector: "bs-two-factor-page",
  templateUrl: "./two-factor-page.component.html",
  styleUrl: "./two-factor-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
})
export class TwoFactorPage implements OnInit {
  private readonly fb = inject(FormBuilder);
  private readonly authApi = inject(AuthApiService);
  private readonly authState = inject(AuthStateService);
  private readonly twoFactorSession = inject(TwoFactorSessionService);
  private readonly router = inject(Router);
  private readonly destroyRef = inject(DestroyRef);

  readonly form = this.fb.nonNullable.group({
    code: ["", [Validators.required, Validators.minLength(4)]],
  });

  isSubmitting = false;
  formError: string | null = null;

  get code() {
    return this.form.controls.code;
  }

  ngOnInit(): void {
    // If there is no pending 2FA user, redirect back to login.
    if (!this.twoFactorSession.hasPendingUserId()) {
      this.router.navigate(["/login"]);
    }
  }

  onSubmit(): void {
    if (this.form.invalid || this.isSubmitting) {
      this.form.markAllAsTouched();
      return;
    }

    const pendingUserId = this.twoFactorSession.getPendingUserId();
    if (!pendingUserId) {
      // Defensive: if the user somehow lost the session, send them back to login.
      this.router.navigate(["/login"]);
      return;
    }

    this.isSubmitting = true;
    this.formError = null;

    const payload: TwoFactorVerifyRequest = {
      userId: pendingUserId,
      code: this.code.value,
    };

    this.authApi
      .verifyTwoFactor(payload)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isSubmitting = false;
        })
      )
      .subscribe({
        next: (result) => {
          if (!result.accessToken) {
            this.formError =
              "We could not complete your sign-in. Please try again.";
            return;
          }

          // 2FA completed successfully: set session and clear pending user.
          this.authState.setSession(result.accessToken);
          this.twoFactorSession.clear();

          // For now we always send the user to their security page.
          // Later we can preserve a returnUrl through the 2FA flow if needed.
          this.router.navigate(["/me/security"]);
        },
        error: () => {
          // Generic message: we don't reveal whether the code was wrong or expired.
          this.formError =
            "Invalid or expired verification code. Please try again.";
        },
      });
  }
}
```

Key behaviors:

- The page **refuses** to operate if there is no pending `userId` in `TwoFactorSessionService`.
- On success, we **always**:
  - Persist the token via `AuthStateService`.
  - Clear the 2FA session so the `userId` is not left hanging.
- On failure, the error message is generic and security-friendly.

---

### 11.7.3 TwoFactorPage template (HTML)

We reuse the same **page layout and design system primitives** as the Login page.

File:  
`apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">Two-step verification</h1>
      <p class="page-header__subtitle">
        Enter the verification code we sent to your registered email.
      </p>
    </div>
  </header>

  <form [formGroup]="form" class="bs-form" (ngSubmit)="onSubmit()">
    <!-- Form-level error -->
    <div *ngIf="formError" class="bs-alert bs-alert--danger">
      <div>
        <div class="bs-alert__title">Verification failed</div>
        <p class="bs-alert__body">{{ formError }}</p>
      </div>
    </div>

    <!-- Code -->
    <div class="bs-form-field">
      <label
        class="bs-form-field__label bs-form-field__label--required"
        for="code"
      >
        Verification code
      </label>
      <div class="bs-form-field__control">
        <input
          id="code"
          type="text"
          inputmode="numeric"
          autocomplete="one-time-code"
          formControlName="code"
        />
      </div>
      <p class="bs-form-field__hint" *ngIf="!code.invalid || !code.touched">
        Check your email for a one-time code.
      </p>
      <p class="bs-form-field__error" *ngIf="code.touched && code.invalid">
        Please enter the verification code you received.
      </p>
    </div>

    <!-- Actions -->
    <div class="two-factor-actions">
      <button
        class="bs-btn bs-btn--primary"
        type="submit"
        [disabled]="isSubmitting"
      >
        <span *ngIf="!isSubmitting">Verify and continue</span>
        <span *ngIf="isSubmitting">Verifying…</span>
      </button>
    </div>
  </form>

  <div class="two-factor-links">
    <a routerLink="/login">Back to sign in</a>
  </div>
</div>
```

This keeps the 2FA page visually aligned with the rest of the IAM portal and clearly communicates what the user must do.

---

### 11.7.4 TwoFactorPage styles (SCSS)

We keep the styles lightweight; most of the look-and-feel comes from global tokens.

File:  
`apps/iam-portal/src/app/features/auth/two-factor-page/two-factor-page.component.scss`

```scss
:host {
  display: block;
}

/* Same width as login card for consistency */
.page {
  max-width: 420px;
}

.two-factor-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: var(--bs-space-3);
}

.two-factor-actions .bs-btn {
  min-width: 160px;
}

.two-factor-links {
  margin-top: var(--bs-space-3);
  font-size: var(--bs-font-size-xs);
  text-align: right;
}

.two-factor-links a {
  color: var(--bs-color-primary);
  text-decoration: none;
}

.two-factor-links a:hover {
  text-decoration: underline;
}
```

---

### 11.7.5 Sanity check for the 2FA flow

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
npx nx serve iam-portal
```

Manual flow:

1. Use a dev IAM user with **2FA enabled**.
2. Go to `/login`, enter valid credentials.
3. Confirm you are redirected to `/login/2fa`.
4. Enter a **valid** code:
   - You should be redirected to `/me/security`.
   - The network tab should show a successful call to `/api/iam/auth/2fa/verify`.
5. Refresh the browser:
   - You should remain authenticated because the token is stored via `AuthStateService`.
6. Navigate directly to `/login/2fa` in a new tab:
   - You should be redirected back to `/login` because there is no pending 2FA session.

With the 2FA step implemented, **login flows are now fully aligned with the IAM backend**, and the IAM portal behaves like a real banking authentication system.

Next, we’ll implement the **Forgot Password and Reset Password** screens using the same design system and auth infrastructure.

---

## 11.8 Forgot Password & Reset Password

In a banking system, the **password lifecycle** must feel safe, predictable, and privacy-preserving:

- Users must be able to request a password reset without revealing whether an email exists.
- Reset links must flow through secure email templates.
- The UI must give clear feedback while still using **generic** wording.

We already have backend endpoints for:

- `POST /api/iam/auth/forgot-password` → `{ sent: boolean }`
- `POST /api/iam/auth/reset-password` → `204 No Content`

Now we implement two screens:

1. **Forgot Password** (`/forgot-password`)

   - Simple email form.
   - Always shows the same success message.

2. **Reset Password** (`/reset-password`)
   - Reads `email` and `token` from query parameters.
   - Form with new password and confirm password.
   - On success, shows a success banner with a link back to login.

Both use `AuthApiService` and the Alvor Bank design system.

---

### 11.8.1 ForgotPasswordPage component

If you already have a placeholder from Chapter 9, we will replace its implementation.  
If not, generate it with Nx:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/auth/forgot-password-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

This creates:

- `apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page.component.ts`
- `apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page.component.html`
- `apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page.component.scss`

#### ForgotPasswordPage TypeScript

File:  
`apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page.component.ts`

```ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { FormBuilder, ReactiveFormsModule, Validators } from "@angular/forms";
import { RouterModule } from "@angular/router";
import { finalize } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

import { AuthApiService } from "../../../core/auth/services/auth-api.service";
import { ForgotPasswordRequest } from "../../../core/auth/models";

@Component({
  standalone: true,
  selector: "bs-forgot-password-page",
  templateUrl: "./forgot-password-page.component.html",
  styleUrl: "./forgot-password-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
})
export class ForgotPasswordPage {
  private readonly fb = inject(FormBuilder);
  private readonly authApi = inject(AuthApiService);
  private readonly destroyRef = inject(DestroyRef);

  readonly form = this.fb.nonNullable.group({
    email: ["", [Validators.required, Validators.email]],
  });

  isSubmitting = false;
  hasSubmitted = false;
  formError: string | null = null;

  get email() {
    return this.form.controls.email;
  }

  onSubmit(): void {
    if (this.form.invalid || this.isSubmitting) {
      this.form.markAllAsTouched();
      return;
    }

    this.isSubmitting = true;
    this.formError = null;

    const payload: ForgotPasswordRequest = this.form.getRawValue();

    this.authApi
      .forgotPassword(payload)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isSubmitting = false;
        })
      )
      .subscribe({
        next: () => {
          // Backend always responds with { sent: true }.
          // We always show the same generic message.
          this.hasSubmitted = true;
        },
        error: () => {
          // We still show the same UX; only log would differ in a real system.
          this.hasSubmitted = true;
        },
      });
  }
}
```

#### ForgotPasswordPage template (HTML)

File:  
`apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">Forgot your password?</h1>
      <p class="page-header__subtitle">
        Enter your email and we&#8217;ll send you a link to reset your password.
      </p>
    </div>
  </header>

  <form [formGroup]="form" class="bs-form" (ngSubmit)="onSubmit()">
    <div
      *ngIf="hasSubmitted"
      class="bs-alert bs-alert--info"
      aria-live="polite"
    >
      <div>
        <div class="bs-alert__title">Check your email</div>
        <p class="bs-alert__body">
          If this email exists in our system, we sent you a link with
          instructions to reset your password.
        </p>
      </div>
    </div>

    <div class="bs-form-field">
      <label
        class="bs-form-field__label bs-form-field__label--required"
        for="email"
      >
        Email
      </label>
      <div class="bs-form-field__control">
        <input
          id="email"
          type="email"
          autocomplete="email"
          formControlName="email"
        />
      </div>
      <p class="bs-form-field__error" *ngIf="email.touched && email.invalid">
        Please enter a valid email address.
      </p>
    </div>

    <div class="forgot-actions">
      <button
        class="bs-btn bs-btn--primary"
        type="submit"
        [disabled]="isSubmitting"
      >
        <span *ngIf="!isSubmitting">Send reset link</span>
        <span *ngIf="isSubmitting">Sending…</span>
      </button>
    </div>
  </form>

  <div class="forgot-links">
    <a routerLink="/login">Back to sign in</a>
  </div>
</div>
```

#### ForgotPasswordPage styles (SCSS)

File:  
`apps/iam-portal/src/app/features/auth/forgot-password-page/forgot-password-page.component.scss`

```scss
:host {
  display: block;
}

.page {
  max-width: 420px;
}

.forgot-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: var(--bs-space-3);
}

.forgot-links {
  margin-top: var(--bs-space-3);
  font-size: var(--bs-font-size-xs);
  text-align: right;
}

.forgot-links a {
  color: var(--bs-color-primary);
  text-decoration: none;
}

.forgot-links a:hover {
  text-decoration: underline;
}
```

---

### 11.8.2 ResetPasswordPage component

The **Reset Password** page is opened via an email link like:

- `https://frontend-domain/password-reset?email=user%40bank.com&token=<url-encoded-token>`

IAM backend config:

- `Iam:PasswordResetBaseUrl` points to this Angular route.

Our UI responsibilities:

- Read `email` and `token` from the query string.
- Display a reset form (new password + confirm).
- Call `AuthApiService.resetPassword`.
- Provide clear success/failure feedback.

If you don’t already have the component, generate it:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/auth/reset-password-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

Files created:

- `apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page.component.ts`
- `apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page.component.html`
- `apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page.component.scss`

#### ResetPasswordPage TypeScript

File:  
`apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page.component.ts`

```ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  OnInit,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { FormBuilder, ReactiveFormsModule, Validators } from "@angular/forms";
import { ActivatedRoute, Router, RouterModule } from "@angular/router";
import { finalize } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

import { AuthApiService } from "../../../core/auth/services/auth-api.service";
import { ResetPasswordRequest } from "../../../core/auth/models";

@Component({
  standalone: true,
  selector: "bs-reset-password-page",
  templateUrl: "./reset-password-page.component.html",
  styleUrl: "./reset-password-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
})
export class ResetPasswordPage implements OnInit {
  private readonly fb = inject(FormBuilder);
  private readonly route = inject(ActivatedRoute);
  private readonly router = inject(Router);
  private readonly authApi = inject(AuthApiService);
  private readonly destroyRef = inject(DestroyRef);

  readonly form = this.fb.nonNullable.group(
    {
      password: ["", [Validators.required, Validators.minLength(8)]],
      confirmPassword: ["", [Validators.required]],
    },
    {
      validators: (group) => {
        const password = group.get("password")?.value;
        const confirm = group.get("confirmPassword")?.value;
        return password && confirm && password !== confirm
          ? { passwordMismatch: true }
          : null;
      },
    }
  );

  email: string | null = null;
  token: string | null = null;

  isSubmitting = false;
  isCompleted = false;
  formError: string | null = null;

  get password() {
    return this.form.controls.password;
  }

  get confirmPassword() {
    return this.form.controls.confirmPassword;
  }

  get hasPasswordMismatch(): boolean {
    return this.form.hasError("passwordMismatch");
  }

  ngOnInit(): void {
    const query = this.route.snapshot.queryParamMap;
    this.email = query.get("email");
    this.token = query.get("token");

    if (!this.email || !this.token) {
      // Missing parameters; show generic error and send user back to login.
      this.form.disable();
      this.formError =
        "This password reset link is invalid or has expired. Please request a new one.";
    }
  }

  onSubmit(): void {
    if (!this.email || !this.token) {
      return;
    }

    if (this.form.invalid || this.isSubmitting) {
      this.form.markAllAsTouched();
      return;
    }

    this.isSubmitting = true;
    this.formError = null;

    const payload: ResetPasswordRequest = {
      email: this.email,
      token: this.token,
      newPassword: this.password.value,
    };

    this.authApi
      .resetPassword(payload)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isSubmitting = false;
        })
      )
      .subscribe({
        next: () => {
          this.isCompleted = true;
          this.form.disable();
        },
        error: () => {
          this.formError =
            "We could not reset your password. Your link may have expired. Please request a new reset email.";
        },
      });
  }

  goToLogin(): void {
    this.router.navigate(["/login"]);
  }
}
```

#### ResetPasswordPage template (HTML)

File:  
`apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">Reset your password</h1>
      <p class="page-header__subtitle">
        Choose a new password for your Alvor Bank account.
      </p>
    </div>
  </header>

  <form [formGroup]="form" class="bs-form" (ngSubmit)="onSubmit()">
    <!-- Error banner -->
    <div *ngIf="formError" class="bs-alert bs-alert--danger">
      <div>
        <div class="bs-alert__title">Unable to reset password</div>
        <p class="bs-alert__body">{{ formError }}</p>
      </div>
    </div>

    <!-- Success banner -->
    <div *ngIf="isCompleted" class="bs-alert bs-alert--success">
      <div>
        <div class="bs-alert__title">Password updated</div>
        <p class="bs-alert__body">
          Your password has been changed successfully. You can now sign in with
          your new password.
        </p>
      </div>
    </div>

    <!-- New password -->
    <div class="bs-form-field">
      <label
        class="bs-form-field__label bs-form-field__label--required"
        for="password"
      >
        New password
      </label>
      <div class="bs-form-field__control">
        <input
          id="password"
          type="password"
          autocomplete="new-password"
          formControlName="password"
        />
      </div>
      <p class="bs-form-field__hint">
        Use at least 8 characters. Consider a mix of letters, numbers and
        symbols.
      </p>
      <p
        class="bs-form-field__error"
        *ngIf="password.touched && password.invalid"
      >
        Your password must be at least 8 characters long.
      </p>
    </div>

    <!-- Confirm password -->
    <div class="bs-form-field">
      <label
        class="bs-form-field__label bs-form-field__label--required"
        for="confirmPassword"
      >
        Confirm password
      </label>
      <div class="bs-form-field__control">
        <input
          id="confirmPassword"
          type="password"
          autocomplete="new-password"
          formControlName="confirmPassword"
        />
      </div>
      <p
        class="bs-form-field__error"
        *ngIf="confirmPassword.touched && confirmPassword.invalid"
      >
        Please confirm your new password.
      </p>
      <p class="bs-form-field__error" *ngIf="hasPasswordMismatch">
        The passwords do not match.
      </p>
    </div>

    <!-- Actions -->
    <div class="reset-actions">
      <button
        class="bs-btn bs-btn--primary"
        type="submit"
        [disabled]="isSubmitting || isCompleted"
      >
        <span *ngIf="!isSubmitting && !isCompleted">Update password</span>
        <span *ngIf="isSubmitting">Updating…</span>
        <span *ngIf="!isSubmitting && isCompleted">Updated</span>
      </button>
    </div>
  </form>

  <div class="reset-links">
    <button type="button" class="bs-btn bs-btn--ghost" (click)="goToLogin()">
      Back to sign in
    </button>
  </div>
</div>
```

#### ResetPasswordPage styles (SCSS)

File:  
`apps/iam-portal/src/app/features/auth/reset-password-page/reset-password-page.component.scss`

```scss
:host {
  display: block;
}

.page {
  max-width: 480px;
}

.reset-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: var(--bs-space-3);
}

.reset-links {
  margin-top: var(--bs-space-3);
  text-align: right;
}
```

---

### 11.8.3 Sanity check for password flows

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
npx nx serve iam-portal
```

Manual test:

1. Go to `/forgot-password`:

   - Submit a valid email → see the generic “Check your email” message.
   - Submit an invalid email format → see field validation error.

2. Use a **real reset link** from the IAM backend (local dev email sender):

   - Click the link → should land on `/reset-password?email=...&token=...`.
   - Enter a new password and confirm → see success banner.
   - Click “Back to sign in” → go to `/login` and log in with the new password.

3. Try opening `/reset-password` without query parameters:
   - You should see a generic error about an invalid/expired link.

With Forgot and Reset password implemented, **employees now have a complete self-service password recovery flow**, aligned with banking security practices.

Next, we’ll handle **email confirmation** and the **My Security** page (change password + 2FA toggle).

---

## 11.9 Email Confirmation Flow

When a new employee is created (or when they change their email), the IAM backend sends a **confirmation email** with a link like:

- `https://iam-portal.local/email-confirmation?userId=<GUID>&token=<URL-ENCODED-TOKEN>`

The backend uses `Iam:EmailConfirmationBaseUrl` to generate this URL.

Our job in the frontend:

- Read `userId` and `token` from the query string.
- Call `AuthApiService.confirmEmail`.
- Show one of two states:
  - **Success** — email confirmed, user can now sign in.
  - **Error** — invalid/expired link, user must request a new one or contact support.

We do **not** reveal whether the email exists or was already confirmed.  
We show clear but generic messages and a simple path back to `/login`.

---

### 11.9.1 EmailConfirmationPage component

If you already have a placeholder from Chapter 9, we’ll replace its implementation.  
If not, generate it now:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/auth/email-confirmation-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

This creates:

- `apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page.component.ts`
- `apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page.component.html`
- `apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page.component.scss`

---

### 11.9.2 EmailConfirmationPage TypeScript

File:  
`apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page.component.ts`

```ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  OnInit,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { ActivatedRoute, Router, RouterModule } from "@angular/router";
import { finalize } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

import { AuthApiService } from "../../../core/auth/services/auth-api.service";

@Component({
  standalone: true,
  selector: "bs-email-confirmation-page",
  templateUrl: "./email-confirmation-page.component.html",
  styleUrl: "./email-confirmation-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, RouterModule],
})
export class EmailConfirmationPage implements OnInit {
  private readonly route = inject(ActivatedRoute);
  private readonly router = inject(Router);
  private readonly authApi = inject(AuthApiService);
  private readonly destroyRef = inject(DestroyRef);

  isLoading = true;
  isSuccess = false;
  errorMessage: string | null = null;

  ngOnInit(): void {
    const query = this.route.snapshot.queryParamMap;
    const userId = query.get("userId");
    const token = query.get("token");

    if (!userId || !token) {
      this.isLoading = false;
      this.isSuccess = false;
      this.errorMessage =
        "This email confirmation link is invalid or has expired. Please request a new one or contact support.";
      return;
    }

    this.authApi
      .confirmEmail(userId, token)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isLoading = false;
        })
      )
      .subscribe({
        next: () => {
          this.isSuccess = true;
        },
        error: () => {
          this.isSuccess = false;
          this.errorMessage =
            "We could not confirm your email. Your link may have expired or already been used. Please try signing in or request a new confirmation email.";
        },
      });
  }

  goToLogin(): void {
    this.router.navigate(["/login"]);
  }
}
```

Key points:

- If `userId` or `token` is missing, we immediately show an error state.
- We never display raw technical errors; messages are user-friendly and generic.
- We always offer a way back to the login page.

---

### 11.9.3 EmailConfirmationPage template (HTML)

File:  
`apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">Email confirmation</h1>
      <p class="page-header__subtitle">
        Finalizing the setup of your Alvor Bank account.
      </p>
    </div>
  </header>

  <!-- Loading state -->
  <div *ngIf="isLoading" class="bs-alert bs-alert--info">
    <div>
      <div class="bs-alert__title">Checking your link…</div>
      <p class="bs-alert__body">
        Please wait while we confirm your email address.
      </p>
    </div>
  </div>

  <!-- Success state -->
  <div *ngIf="!isLoading && isSuccess" class="bs-alert bs-alert--success">
    <div>
      <div class="bs-alert__title">Email confirmed</div>
      <p class="bs-alert__body">
        Your email address has been confirmed successfully. You can now sign in
        to the IAM portal using your credentials.
      </p>
    </div>
  </div>

  <!-- Error state -->
  <div
    *ngIf="!isLoading && !isSuccess && errorMessage"
    class="bs-alert bs-alert--danger"
  >
    <div>
      <div class="bs-alert__title">Unable to confirm email</div>
      <p class="bs-alert__body">{{ errorMessage }}</p>
    </div>
  </div>

  <div class="confirmation-actions" *ngIf="!isLoading">
    <button type="button" class="bs-btn bs-btn--primary" (click)="goToLogin()">
      Go to sign in
    </button>
  </div>
</div>
```

This template:

- Uses the same `page` layout and `bs-alert` primitives as the rest of the IAM portal.
- Keeps the experience simple and clear: “checking”, “confirmed”, or “error”, with a single call to action.

---

### 11.9.4 EmailConfirmationPage styles (SCSS)

File:  
`apps/iam-portal/src/app/features/auth/email-confirmation-page/email-confirmation-page.component.scss`

```scss
:host {
  display: block;
}

.page {
  max-width: 480px;
}

.confirmation-actions {
  margin-top: var(--bs-space-4);
  display: flex;
  justify-content: flex-end;
}
```

---

### 11.9.5 Route & flow recap

We already wired the route in `app.routes.ts` with `guestGuard`:

```ts
{
  path: 'email-confirmation',
  component: EmailConfirmationPage,
  canActivate: [guestGuard],
}
```

End-to-end flow:

1. IAM backend sends a confirmation email using `Iam:EmailConfirmationBaseUrl`, e.g.:

   - `https://iam-portal.local/email-confirmation?userId=...&token=...`

2. Employee clicks the link.
3. IAM portal:
   - Loads `EmailConfirmationPage`.
   - Calls `AuthApiService.confirmEmail(userId, token)`.
   - Shows success or error state.
4. Employee clicks **“Go to sign in”** to reach `/login`.

With email confirmation implemented, new employees can finish their onboarding and sign in using the same IAM portal, completing a critical part of a real banking IAM lifecycle.

Next, we will implement the **“My Security” page**, allowing authenticated employees to:

- Change their password.
- Enable or disable 2FA on their own account.

---

## 11.10 “My Security” Page (Change Password & 2FA Toggle)

The **My Security** page is the primary self-service security screen for an employee:

- Change their own password.
- Enable or disable Two-Factor Authentication (2FA).

This page is only available to **authenticated users** and is already guarded by `authGuard` in our routes:

```ts
{
  path: 'me/security',
  component: MySecurityPage,
  canActivate: [authGuard],
}
```

In this section we:

- Implement the `MySecurityPage` component.
- Use `AuthApiService.changePassword`, `enableTwoFactor`, and `disableTwoFactor`.
- Use our **design system** (cards, alerts, buttons, tags) to make the UX consistent and bank-grade.

> For now, we maintain 2FA status locally in the component (`isTwoFactorEnabled`).  
> In a later chapter we’ll connect this to a proper `GET /api/iam/me` endpoint.

---

### 11.10.1 Generate the MySecurityPage component

If you already have a placeholder component from Chapter 9, you can replace its implementation.

If not, generate it now:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  features/auth/my-security-page \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

This creates:

- `apps/iam-portal/src/app/features/auth/my-security-page/my-security-page.component.ts`
- `apps/iam-portal/src/app/features/auth/my-security-page/my-security-page.component.html`
- `apps/iam-portal/src/app/features/auth/my-security-page/my-security-page.component.scss`

---

### 11.10.2 MySecurityPage TypeScript

We’ll build **two** reactive forms:

1. `changePasswordForm` — current + new + confirm password.
2. `twoFactorForm` — current password used to enable/disable 2FA.

We’ll track:

- `isTwoFactorEnabled` — local flag for 2FA status (stub until we have a `/me` endpoint).
- Separate `isSubmitting` and messages for each section.

File:  
`apps/iam-portal/src/app/features/auth/my-security-page/my-security-page.component.ts`

```ts
import {
  ChangeDetectionStrategy,
  Component,
  DestroyRef,
  OnInit,
  inject,
} from "@angular/core";
import { CommonModule } from "@angular/common";
import { FormBuilder, ReactiveFormsModule, Validators } from "@angular/forms";
import { RouterModule } from "@angular/router";
import { finalize } from "rxjs/operators";
import { takeUntilDestroyed } from "@angular/core/rxjs-interop";

import { AuthApiService } from "../../../core/auth/services/auth-api.service";
import {
  ChangePasswordRequest,
  CurrentPasswordRequest,
} from "../../../core/auth/models";

@Component({
  standalone: true,
  selector: "bs-my-security-page",
  templateUrl: "./my-security-page.component.html",
  styleUrl: "./my-security-page.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, ReactiveFormsModule, RouterModule],
})
export class MySecurityPage implements OnInit {
  private readonly fb = inject(FormBuilder);
  private readonly authApi = inject(AuthApiService);
  private readonly destroyRef = inject(DestroyRef);

  // In a later chapter we'll load this from a /me endpoint.
  isTwoFactorEnabled = false;

  // Change password form
  readonly changePasswordForm = this.fb.nonNullable.group(
    {
      currentPassword: ["", [Validators.required]],
      newPassword: ["", [Validators.required, Validators.minLength(8)]],
      confirmNewPassword: ["", [Validators.required]],
    },
    {
      validators: (group) => {
        const newPassword = group.get("newPassword")?.value;
        const confirm = group.get("confirmNewPassword")?.value;
        return newPassword && confirm && newPassword !== confirm
          ? { passwordMismatch: true }
          : null;
      },
    }
  );

  isChangingPassword = false;
  changePasswordSuccess: string | null = null;
  changePasswordError: string | null = null;

  // 2FA form (current password required for enable/disable)
  readonly twoFactorForm = this.fb.nonNullable.group({
    currentPassword: ["", [Validators.required]],
  });

  isUpdatingTwoFactor = false;
  twoFactorSuccess: string | null = null;
  twoFactorError: string | null = null;

  get currentPassword() {
    return this.changePasswordForm.controls.currentPassword;
  }

  get newPassword() {
    return this.changePasswordForm.controls.newPassword;
  }

  get confirmNewPassword() {
    return this.changePasswordForm.controls.confirmNewPassword;
  }

  get hasPasswordMismatch(): boolean {
    return this.changePasswordForm.hasError("passwordMismatch");
  }

  get twoFactorCurrentPassword() {
    return this.twoFactorForm.controls.currentPassword;
  }

  ngOnInit(): void {
    // Placeholder: in a real system, we'd load the current 2FA status from the backend.
    // this.loadCurrentSecuritySettings();
  }

  onChangePasswordSubmit(): void {
    if (this.changePasswordForm.invalid || this.isChangingPassword) {
      this.changePasswordForm.markAllAsTouched();
      return;
    }

    this.isChangingPassword = true;
    this.changePasswordSuccess = null;
    this.changePasswordError = null;

    const payload: ChangePasswordRequest = {
      currentPassword: this.currentPassword.value,
      newPassword: this.newPassword.value,
    };

    this.authApi
      .changePassword(payload)
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isChangingPassword = false;
        })
      )
      .subscribe({
        next: () => {
          this.changePasswordSuccess = "Your password has been updated.";
          this.changePasswordForm.reset();
        },
        error: () => {
          this.changePasswordError =
            "We could not update your password. Please check your current password and try again.";
        },
      });
  }

  onEnableTwoFactor(): void {
    this.updateTwoFactor(true);
  }

  onDisableTwoFactor(): void {
    this.updateTwoFactor(false);
  }

  private updateTwoFactor(enable: boolean): void {
    if (this.twoFactorForm.invalid || this.isUpdatingTwoFactor) {
      this.twoFactorForm.markAllAsTouched();
      return;
    }

    this.isUpdatingTwoFactor = true;
    this.twoFactorSuccess = null;
    this.twoFactorError = null;

    const payload: CurrentPasswordRequest = {
      currentPassword: this.twoFactorCurrentPassword.value,
    };

    const request$ = enable
      ? this.authApi.enableTwoFactor(payload)
      : this.authApi.disableTwoFactor(payload);

    request$
      .pipe(
        takeUntilDestroyed(this.destroyRef),
        finalize(() => {
          this.isUpdatingTwoFactor = false;
        })
      )
      .subscribe({
        next: () => {
          this.isTwoFactorEnabled = enable;
          this.twoFactorForm.reset();

          this.twoFactorSuccess = enable
            ? "Two-factor authentication has been enabled for your account."
            : "Two-factor authentication has been disabled for your account.";
        },
        error: () => {
          this.twoFactorError =
            "We could not update your two-factor settings. Please check your password and try again.";
        },
      });
  }
}
```

This implementation:

- Keeps **password** and **2FA** concerns separate but on the same page.
- Uses `ChangePasswordRequest` and `CurrentPasswordRequest` models to stay aligned with the backend.
- Uses generic error messages—no hint as to whether the password was correct or not beyond “check your current password”.

---

### 11.10.3 MySecurityPage template (HTML)

We now render two sections in a single `page`:

1. **Change password** card.
2. **Two-factor authentication** card.

File:  
`apps/iam-portal/src/app/features/auth/my-security-page/my-security-page.component.html`

```html
<div class="page">
  <header class="page-header">
    <div>
      <h1 class="page-header__title">My security</h1>
      <p class="page-header__subtitle">
        Manage your password and two-factor authentication for the IAM portal.
      </p>
    </div>
  </header>

  <!-- Change password section -->
  <section class="security-section">
    <div class="security-section__header">
      <h2 class="security-section__title">Change password</h2>
      <p class="security-section__subtitle">
        Update your password regularly to keep your Alvor Bank account secure.
      </p>
    </div>

    <form
      [formGroup]="changePasswordForm"
      class="bs-form"
      (ngSubmit)="onChangePasswordSubmit()"
    >
      <!-- Success -->
      <div
        *ngIf="changePasswordSuccess"
        class="bs-alert bs-alert--success"
        aria-live="polite"
      >
        <div>
          <div class="bs-alert__title">Password updated</div>
          <p class="bs-alert__body">{{ changePasswordSuccess }}</p>
        </div>
      </div>

      <!-- Error -->
      <div
        *ngIf="changePasswordError"
        class="bs-alert bs-alert--danger"
        aria-live="polite"
      >
        <div>
          <div class="bs-alert__title">Unable to update password</div>
          <p class="bs-alert__body">{{ changePasswordError }}</p>
        </div>
      </div>

      <!-- Current password -->
      <div class="bs-form-field">
        <label
          class="bs-form-field__label bs-form-field__label--required"
          for="currentPassword"
        >
          Current password
        </label>
        <div class="bs-form-field__control">
          <input
            id="currentPassword"
            type="password"
            autocomplete="current-password"
            formControlName="currentPassword"
          />
        </div>
        <p
          class="bs-form-field__error"
          *ngIf="currentPassword.touched && currentPassword.invalid"
        >
          Please enter your current password.
        </p>
      </div>

      <!-- New password -->
      <div class="bs-form-field">
        <label
          class="bs-form-field__label bs-form-field__label--required"
          for="newPassword"
        >
          New password
        </label>
        <div class="bs-form-field__control">
          <input
            id="newPassword"
            type="password"
            autocomplete="new-password"
            formControlName="newPassword"
          />
        </div>
        <p class="bs-form-field__hint">
          Use at least 8 characters and avoid reusing passwords from other
          systems.
        </p>
        <p
          class="bs-form-field__error"
          *ngIf="newPassword.touched && newPassword.invalid"
        >
          Your password must be at least 8 characters long.
        </p>
      </div>

      <!-- Confirm new password -->
      <div class="bs-form-field">
        <label
          class="bs-form-field__label bs-form-field__label--required"
          for="confirmNewPassword"
        >
          Confirm new password
        </label>
        <div class="bs-form-field__control">
          <input
            id="confirmNewPassword"
            type="password"
            autocomplete="new-password"
            formControlName="confirmNewPassword"
          />
        </div>
        <p
          class="bs-form-field__error"
          *ngIf="
            confirmNewPassword.touched && confirmNewPassword.invalid
          "
        >
          Please confirm your new password.
        </p>
        <p class="bs-form-field__error" *ngIf="hasPasswordMismatch">
          The passwords do not match.
        </p>
      </div>

      <div class="security-section__actions">
        <button
          class="bs-btn bs-btn--primary"
          type="submit"
          [disabled]="isChangingPassword"
        >
          <span *ngIf="!isChangingPassword">Update password</span>
          <span *ngIf="isChangingPassword">Updating…</span>
        </button>
      </div>
    </form>
  </section>

  <!-- Two-factor authentication section -->
  <section class="security-section">
    <div class="security-section__header security-section__header--with-tag">
      <div>
        <h2 class="security-section__title">Two-factor authentication</h2>
        <p class="security-section__subtitle">
          Add an extra layer of security by requiring a verification code during
          sign-in.
        </p>
      </div>

      <span
        class="bs-tag"
        [ngClass]="{
          'bs-tag--success': isTwoFactorEnabled,
          'bs-tag--neutral': !isTwoFactorEnabled
        }"
      >
        <ng-container *ngIf="isTwoFactorEnabled; else twoFaDisabled">
          2FA enabled
        </ng-container>
        <ng-template #twoFaDisabled>2FA disabled</ng-template>
      </span>
    </div>

    <!-- Success -->
    <div
      *ngIf="twoFactorSuccess"
      class="bs-alert bs-alert--success"
      aria-live="polite"
    >
      <div>
        <div class="bs-alert__title">Two-factor updated</div>
        <p class="bs-alert__body">{{ twoFactorSuccess }}</p>
      </div>
    </div>

    <!-- Error -->
    <div
      *ngIf="twoFactorError"
      class="bs-alert bs-alert--danger"
      aria-live="polite"
    >
      <div>
        <div class="bs-alert__title">Unable to update two-factor</div>
        <p class="bs-alert__body">{{ twoFactorError }}</p>
      </div>
    </div>

    <form
      [formGroup]="twoFactorForm"
      class="bs-form"
      (ngSubmit)="$event.preventDefault()"
    >
      <div class="bs-form-field">
        <label
          class="bs-form-field__label bs-form-field__label--required"
          for="twoFactorCurrentPassword"
        >
          Current password
        </label>
        <div class="bs-form-field__control">
          <input
            id="twoFactorCurrentPassword"
            type="password"
            autocomplete="current-password"
            formControlName="currentPassword"
          />
        </div>
        <p
          class="bs-form-field__error"
          *ngIf="
            twoFactorCurrentPassword.touched &&
            twoFactorCurrentPassword.invalid
          "
        >
          Please enter your current password to change two-factor settings.
        </p>
      </div>

      <div
        class="security-section__actions security-section__actions--two-factor"
      >
        <button
          class="bs-btn"
          [ngClass]="{
            'bs-btn--primary': !isTwoFactorEnabled,
            'bs-btn--danger': isTwoFactorEnabled
          }"
          type="button"
          (click)="isTwoFactorEnabled ? onDisableTwoFactor() : onEnableTwoFactor()"
          [disabled]="isUpdatingTwoFactor"
        >
          <span *ngIf="!isUpdatingTwoFactor && !isTwoFactorEnabled">
            Enable 2FA
          </span>
          <span *ngIf="!isUpdatingTwoFactor && isTwoFactorEnabled">
            Disable 2FA
          </span>
          <span *ngIf="isUpdatingTwoFactor">Updating…</span>
        </button>
      </div>
    </form>
  </section>
</div>
```

The layout treats each security concern as a **card-like section** inside the main `page`, using the same design system primitives everywhere.

---

### 11.10.4 MySecurityPage styles (SCSS)

File:  
`apps/iam-portal/src/app/features/auth/my-security-page/my-security-page.component.scss`

```scss
:host {
  display: block;
}

.page {
  max-width: 720px;
}

.security-section {
  margin-top: var(--bs-space-4);
  padding: var(--bs-space-4);
  border-radius: var(--bs-radius-lg);
  background-color: var(--bs-color-surface);
  box-shadow: var(--bs-shadow-soft);
  border: 1px solid var(--bs-color-border-subtle);
}

.security-section__header {
  margin-bottom: var(--bs-space-3);
}

.security-section__header--with-tag {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: var(--bs-space-3);
}

.security-section__title {
  font-size: var(--bs-font-size-lg);
  margin: 0 0 var(--bs-space-1) 0;
}

.security-section__subtitle {
  margin: 0;
  font-size: var(--bs-font-size-sm);
  color: var(--bs-color-text-muted);
}

.security-section__actions {
  display: flex;
  justify-content: flex-end;
  margin-top: var(--bs-space-3);
}

.security-section__actions--two-factor {
  margin-top: var(--bs-space-4);
}
```

These styles:

- Make each security section look like a **card** within the IAM portal.
- Keep typography and spacing aligned with the design tokens from Chapter 10.

---

### 11.10.5 Sanity check for My Security

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
npx nx serve iam-portal
```

Manual flow:

1. Log in as a dev IAM user (with or without 2FA).
2. Navigate to `/me/security`:
   - Confirm that the route is blocked if you are not logged in.
3. Test **Change password**:
   - Enter wrong current password → see generic error message.
   - Enter correct current password + valid new password → see success banner.
4. Test **Two-factor**:
   - Enter current password and click **Enable 2FA** → see success banner and tag change to “2FA enabled”.
   - Click **Disable 2FA** with current password → see success banner and tag change back to “2FA disabled”.

At this point, employees can manage their own security settings through the IAM portal, just like in a real banking system.

Next, we will tie all IAM auth flows together with **tests** (unit + E2E) and then close the chapter by running the full frontend quality gate before merging this work into `develop`.

---

## 11.11 Updating Navigation & Header Based on Auth State

Now that we have:

- `AuthStateService` (token + `isAuthenticated$`)
- Guards (`authGuard`, `guestGuard`)
- All auth flows (login, 2FA, password, email confirmation, My Security)

…the last piece to make the IAM portal feel like a **real banking system** is to reflect **auth state** in the **shell header**:

- Show different links when the user is **signed in** vs **signed out**.
- Provide a **Sign out** action in a predictable place.
- Use the same design system primitives for a polished, consistent look.

We assume you already have a **shell layout** from Chapter 9, something like:

- `ShellLayout` with header + sidebar + `<router-outlet>`.

If your implementation differs, you can still adapt this section — the key ideas are the same.

Target location for the shell:

```
apps/iam-portal/src/app/layout/shell-layout/
  shell-layout.component.ts
  shell-layout.component.html
  shell-layout.component.scss
```

If you do **not** have such a component yet, you can generate it now.

---

### 11.11.1 (Optional) Generate ShellLayout component

If you already created a shell layout in Chapter 9, **skip this command** and just update your existing files.

Otherwise, from `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx g @nx/angular:component \
  layout/shell-layout \
  --project=iam-portal \
  --standalone \
  --flat=false \
  --changeDetection=OnPush \
  --style=scss
```

This creates:

- `apps/iam-portal/src/app/layout/shell-layout/shell-layout.component.ts`
- `...html`
- `...scss`

In Chapter 9 you should already have wired this shell into `app.routes.ts` as the root layout hosting the IAM routes. If not, you can update the routes after this section; the core focus here is the **header auth state**.

---

### 11.11.2 ShellLayout component (TypeScript)

We update the shell to:

- Inject `AuthStateService` and expose `isAuthenticated$` to the template.
- Expose a `signOut()` method that:
  - Clears the session in `AuthStateService`.
  - Redirects to `/login`.

File:  
`apps/iam-portal/src/app/layout/shell-layout/shell-layout.component.ts`

```ts
import { ChangeDetectionStrategy, Component, inject } from "@angular/core";
import { CommonModule } from "@angular/common";
import { Router, RouterModule } from "@angular/router";

import { AuthStateService } from "../../core/auth/services/auth-state.service";

@Component({
  standalone: true,
  selector: "bs-shell-layout",
  templateUrl: "./shell-layout.component.html",
  styleUrl: "./shell-layout.component.scss",
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [CommonModule, RouterModule],
})
export class ShellLayout {
  private readonly router = inject(Router);
  private readonly authState = inject(AuthStateService);

  readonly isAuthenticated$ = this.authState.isAuthenticated$;

  signOut(): void {
    this.authState.clearSession();
    this.router.navigate(["/login"]);
  }
}
```

Even without a `/me` endpoint (yet), this gives us enough to:

- Show/hide navigation items based on `isAuthenticated$`.
- Provide a reliable **Sign out** action.

Later, when we implement a “current user” endpoint, we can extend this shell to show the current user’s email or role.

---

### 11.11.3 ShellLayout template (HTML)

Now we build a professional banking shell:

- Header with:
  - Bank brand (left).
  - Simple navigation (center/left).
  - Auth state/user actions (right).
- Sidebar (if you already have one).
- Page content (`<router-outlet>`).

File:  
`apps/iam-portal/src/app/layout/shell-layout/shell-layout.component.html`

```html
<div class="shell">
  <header class="shell-header">
    <div class="shell-header__brand">
      <span class="shell-header__logo">AB</span>
      <div class="shell-header__titles">
        <div class="shell-header__title">Alvor Bank</div>
        <div class="shell-header__subtitle">Employee IAM Portal</div>
      </div>
    </div>

    <nav class="shell-header__nav" *ngIf="isAuthenticated$ | async">
      <a routerLink="/me/security" routerLinkActive="is-active">
        My security
      </a>
      <a routerLink="/admin/employees" routerLinkActive="is-active">
        Employees
      </a>
    </nav>

    <div class="shell-header__user">
      <ng-container *ngIf="isAuthenticated$ | async; else guestLinks">
        <span class="shell-header__user-label">
          Signed in
          <!-- Placeholder for future user email/role -->
        </span>
        <button type="button" class="bs-btn bs-btn--ghost" (click)="signOut()">
          Sign out
        </button>
      </ng-container>

      <ng-template #guestLinks>
        <a routerLink="/login" class="shell-header__login-link"> Sign in </a>
      </ng-template>
    </div>
  </header>

  <div class="shell-body">
    <!-- Optional sidebar; if you already have one, integrate it here -->
    <!--
    <aside class="shell-sidebar">
      <nav>
        <a routerLink="/me/security" routerLinkActive="is-active">
          My security
        </a>
        <a routerLink="/admin/employees" routerLinkActive="is-active">
          Employees
        </a>
      </nav>
    </aside>
    -->

    <main class="shell-content">
      <router-outlet></router-outlet>
    </main>
  </div>
</div>
```

Notes:

- We show navigation links only when `isAuthenticated$ | async` is true.
- When signed out, the header just shows a **Sign in** link (or whatever the global behavior you prefer).
- The comment block for `<aside>` allows you to either:
  - Use a sidebar now.
  - Introduce it later when the portal grows.

---

### 11.11.4 ShellLayout styles (SCSS)

We’ll style the shell using the **design tokens** defined in Chapter 10.

File:  
`apps/iam-portal/src/app/layout/shell-layout/shell-layout.component.scss`

```scss
:host {
  display: block;
  min-height: 100vh;
}

.shell {
  min-height: 100vh;
  background-color: var(--bs-color-bg);
  color: var(--bs-color-text-main);
}

/* Header */

.shell-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--bs-space-4);
  padding: var(--bs-space-3) var(--bs-space-4);
  background: linear-gradient(
    135deg,
    var(--bs-color-primary),
    var(--bs-color-primary-soft)
  );
  color: var(--bs-color-text-on-primary);
  box-shadow: var(--bs-shadow-soft);
}

.shell-header__brand {
  display: flex;
  align-items: center;
  gap: var(--bs-space-2);
}

.shell-header__logo {
  width: 32px;
  height: 32px;
  border-radius: 999px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  font-size: var(--bs-font-size-sm);
  background-color: rgba(15, 23, 42, 0.2);
  border: 1px solid rgba(255, 255, 255, 0.4);
}

.shell-header__titles {
  display: flex;
  flex-direction: column;
}

.shell-header__title {
  font-size: var(--bs-font-size-lg);
  font-weight: 600;
}

.shell-header__subtitle {
  font-size: var(--bs-font-size-xs);
  opacity: 0.85;
}

.shell-header__nav {
  display: flex;
  gap: var(--bs-space-3);
  font-size: var(--bs-font-size-sm);
}

.shell-header__nav a {
  color: var(--bs-color-text-on-primary);
  text-decoration: none;
  opacity: 0.9;
}

.shell-header__nav a:hover {
  opacity: 1;
  text-decoration: underline;
}

.shell-header__nav a.is-active {
  font-weight: 600;
  text-decoration: underline;
}

.shell-header__user {
  display: flex;
  align-items: center;
  gap: var(--bs-space-2);
  font-size: var(--bs-font-size-xs);
}

.shell-header__user-label {
  opacity: 0.9;
}

.shell-header__login-link {
  color: var(--bs-color-text-on-primary);
  text-decoration: none;
}

.shell-header__login-link:hover {
  text-decoration: underline;
}

/* Body */

.shell-body {
  display: flex;
  min-height: calc(100vh - 64px);
}

/* Optional sidebar
.shell-sidebar {
  width: 220px;
  padding: var(--bs-space-4) var(--bs-space-3);
  background-color: var(--bs-color-surface);
  border-right: 1px solid var(--bs-color-border-subtle);
}

.shell-sidebar nav {
  display: flex;
  flex-direction: column;
  gap: var(--bs-space-2);
}

.shell-sidebar a {
  font-size: var(--bs-font-size-sm);
  color: var(--bs-color-text-main);
  text-decoration: none;
  padding: 0.35rem 0.5rem;
  border-radius: var(--bs-radius-md);
}

.shell-sidebar a:hover {
  background-color: #eef2ff;
}
*/

.shell-content {
  flex: 1;
  padding: var(--bs-space-4) var(--bs-space-4) var(--bs-space-6);
}
```

This gives the IAM portal a **professional banking shell**:

- Branded header.
- Navigation that respects auth state.
- Clean content area for all IAM pages.

---

### 11.11.5 Route wiring (if needed)

If your app is not yet using the shell as the root layout, you can **wrap existing routes** inside a `ShellLayout` route.

File:  
`apps/iam-portal/src/app/app.routes.ts`

**Conceptual example** (adapt to your existing setup; the guards from earlier sections stay the same):

```ts
import { Routes } from "@angular/router";

import { ShellLayout } from "./layout/shell-layout/shell-layout.component";
import { LoginPage } from "./features/auth/login-page/login-page.component";
import { TwoFactorPage } from "./features/auth/two-factor-page/two-factor-page.component";
import { ForgotPasswordPage } from "./features/auth/forgot-password-page/forgot-password-page.component";
import { ResetPasswordPage } from "./features/auth/reset-password-page/reset-password-page.component";
import { EmailConfirmationPage } from "./features/auth/email-confirmation-page/email-confirmation-page.component";
import { MySecurityPage } from "./features/auth/my-security-page/my-security-page.component";
import { EmployeesPage } from "./features/admin/employees/employees-page/employees-page.component";
import { EmployeeDetailsPage } from "./features/admin/employees/employee-details-page/employee-details-page.component";

import { authGuard } from "./core/auth/guards/auth.guard";
import { guestGuard } from "./core/auth/guards/guest.guard";

export const routes: Routes = [
  // Shell with main content
  {
    path: "",
    component: ShellLayout,
    children: [
      // Auth flows (guest-only)
      {
        path: "login",
        component: LoginPage,
        canActivate: [guestGuard],
      },
      {
        path: "login/2fa",
        component: TwoFactorPage,
        canActivate: [guestGuard],
      },
      {
        path: "forgot-password",
        component: ForgotPasswordPage,
        canActivate: [guestGuard],
      },
      {
        path: "reset-password",
        component: ResetPasswordPage,
        canActivate: [guestGuard],
      },
      {
        path: "email-confirmation",
        component: EmailConfirmationPage,
        canActivate: [guestGuard],
      },

      // Authenticated-only
      {
        path: "me/security",
        component: MySecurityPage,
        canActivate: [authGuard],
      },
      {
        path: "admin/employees",
        component: EmployeesPage,
        canActivate: [authGuard],
      },
      {
        path: "admin/employees/:id",
        component: EmployeeDetailsPage,
        canActivate: [authGuard],
      },

      // Default
      {
        path: "",
        pathMatch: "full",
        redirectTo: "login",
      },
      {
        path: "**",
        redirectTo: "login",
      },
    ],
  },
];
```

If your Chapter 9 routes already look similar, you only need to ensure the `ShellLayout` is used and that `authGuard`/`guestGuard` are applied as shown earlier.

---

### 11.11.6 Sanity check

From `src/frontend`:

```bash
cd digital-banking-suite/src/frontend

npx nx lint iam-portal
npx nx test iam-portal
npx nx serve iam-portal
```

Manual checks:

1. Not signed in:
   - Visit `/login` → header shows “Sign in” on the right.
   - `/me/security` should redirect to `/login`.
2. After successful login:
   - Header shows “Signed in” + “Sign out” button.
   - Navigation links (My security, Employees) appear.
3. Click **Sign out**:
   - Token cleared.
   - Redirected to `/login`.
   - Header returns to guest state.

With the shell now reflecting auth state, the IAM portal feels like a **cohesive banking application**, not just a set of isolated pages.

Next, we’ll focus on **testing the IAM auth flows** (unit + E2E) and closing this chapter with a full frontend quality gate.

---

## 11.12 Testing the IAM Auth Flows

A banking front-end without tests is not acceptable.  
In this section we add **targeted tests** for the IAM auth layer:

- Unit tests for:
  - `AuthApiService`
  - `AuthStateService`
- Optional tests for route guards (lightweight but valuable).
- E2E tests for the main auth flows in `iam-portal-e2e`.

> We assume the default Nx Jest + Cypress setup created in Chapter 9 is still in place.

---

### 11.12.1 Unit tests for AuthApiService

`AuthApiService` is pure HTTP; it’s straightforward to test using Angular’s `HttpClientTestingModule`.

File:  
`apps/iam-portal/src/app/core/auth/services/auth-api.service.spec.ts`

Replace contents with:

```ts
import { TestBed } from "@angular/core/testing";
import {
  HttpClientTestingModule,
  HttpTestingController,
} from "@angular/common/http/testing";

import { AuthApiService } from "./auth-api.service";
import {
  LoginRequest,
  LoginResult,
  ForgotPasswordRequest,
  EmailActionResponse,
} from "../models";

describe("AuthApiService", () => {
  let service: AuthApiService;
  let httpMock: HttpTestingController;

  const IAM_API_URL =
    (process.env["IAM_API_URL"] as string | undefined) ??
    "https://localhost:5001";
  const baseUrl = `${IAM_API_URL}/api/iam/auth`;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [AuthApiService],
    });

    service = TestBed.inject(AuthApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
  });

  it("should call /login with email and password", () => {
    const payload: LoginRequest = {
      email: "user@example.com",
      password: "Password123!",
    };

    const mockResponse: LoginResult = {
      requiresTwoFactor: false,
      accessToken: "jwt-token",
      userId: "some-id",
      expiresAt: "2024-01-01T00:00:00Z",
    };

    service.login(payload).subscribe((result) => {
      expect(result).toEqual(mockResponse);
    });

    const req = httpMock.expectOne(`${baseUrl}/login`);
    expect(req.request.method).toBe("POST");
    expect(req.request.body).toEqual(payload);

    req.flush(mockResponse);
  });

  it("should call /forgot-password and parse EmailActionResponse", () => {
    const payload: ForgotPasswordRequest = {
      email: "user@example.com",
    };

    const mockResponse: EmailActionResponse = {
      sent: true,
    };

    service.forgotPassword(payload).subscribe((result) => {
      expect(result).toEqual(mockResponse);
    });

    const req = httpMock.expectOne(`${baseUrl}/forgot-password`);
    expect(req.request.method).toBe("POST");
    expect(req.request.body).toEqual(payload);

    req.flush(mockResponse);
  });
});
```

You can add similar tests for:

- `verifyTwoFactor`
- `resetPassword`
- `changePassword`
- `enableTwoFactor` / `disableTwoFactor`

…but the above gives a solid example for the book.

---

### 11.12.2 Unit tests for AuthStateService

`AuthStateService` is pure state management with a small `localStorage` integration.  
We test:

- Initial value reading.
- `setSession` and `clearSession`.
- `isAuthenticated$` behavior (simplified with synchronous access).

File:  
`apps/iam-portal/src/app/core/auth/services/auth-state.service.spec.ts`

Replace contents with:

```ts
import { TestBed } from "@angular/core/testing";

import { AuthStateService } from "./auth-state.service";

describe("AuthStateService", () => {
  let service: AuthStateService;

  beforeEach(() => {
    // Mock localStorage for tests
    const store: Record<string, string> = {};

    spyOn(window.localStorage, "getItem").and.callFake((key: string) => {
      return store[key] ?? null;
    });

    spyOn(window.localStorage, "setItem").and.callFake(
      (key: string, value: string) => {
        store[key] = value;
      }
    );

    spyOn(window.localStorage, "removeItem").and.callFake((key: string) => {
      delete store[key];
    });

    TestBed.configureTestingModule({
      providers: [AuthStateService],
    });

    service = TestBed.inject(AuthStateService);
  });

  it("should start unauthenticated when no token is stored", (done) => {
    service.isAuthenticated$.subscribe((isAuthenticated) => {
      expect(isAuthenticated).toBeFalse();
      done();
    });
  });

  it("should become authenticated after setting a token", (done) => {
    service.setSession("fake-jwt-token");

    service.isAuthenticated$.subscribe((isAuthenticated) => {
      expect(isAuthenticated).toBeTrue();
      expect(service.getAccessToken()).toBe("fake-jwt-token");
      done();
    });
  });

  it("should clear token and become unauthenticated on clearSession", (done) => {
    service.setSession("fake-jwt-token");
    service.clearSession();

    service.isAuthenticated$.subscribe((isAuthenticated) => {
      expect(isAuthenticated).toBeFalse();
      expect(service.getAccessToken()).toBeNull();
      done();
    });
  });
});
```

This keeps the tests focused on behavior, not implementation details.

---

### 11.12.3 (Optional) Tests for route guards

Guard tests are not strictly required, but they are small and demonstrate the pattern.

Example for `authGuard`:

File:  
`apps/iam-portal/src/app/core/auth/guards/auth.guard.spec.ts`

```ts
import { TestBed } from "@angular/core/testing";
import { Router } from "@angular/router";
import { RouterTestingModule } from "@angular/router/testing";

import { authGuard } from "./auth.guard";
import { AuthStateService } from "../services/auth-state.service";

describe("authGuard", () => {
  let router: Router;
  let authState: jasmine.SpyObj<AuthStateService>;

  beforeEach(() => {
    authState = jasmine.createSpyObj<AuthStateService>("AuthStateService", [
      "getAccessToken",
    ]);

    TestBed.configureTestingModule({
      imports: [RouterTestingModule],
      providers: [{ provide: AuthStateService, useValue: authState }],
    });

    router = TestBed.inject(Router);
  });

  it("should allow navigation when token exists", () => {
    authState.getAccessToken.and.returnValue("token");

    const canActivate = authGuard({} as any, { url: "/me/security" } as any);

    expect(canActivate).toBeTrue();
  });

  it("should redirect to /login when not authenticated", () => {
    authState.getAccessToken.and.returnValue(null);

    const result = authGuard({} as any, { url: "/me/security" } as any);

    expect(router.serializeUrl(result as any)).toContain("/login");
  });
});
```

You can add a similar lightweight spec for `guestGuard` if desired.

---

### 11.12.4 E2E tests for IAM auth flows

For E2E, we update `iam-portal-e2e` (Cypress) to cover:

- Login page rendering.
- Successful login without 2FA (using a seeded dev user).
- Basic Forgot Password UX.

File:  
`apps/iam-portal-e2e/src/e2e/app.cy.ts`

Replace contents with:

```ts
describe("IAM Portal Auth Flows", () => {
  const baseUrl = "http://localhost:4200";

  // Adjust these to match your seeded IAM admin user in backend
  const devUserEmail = "iam.admin@alvorbank.test";
  const devUserPassword = "Admin123!";

  it("should display the login page", () => {
    cy.visit(`${baseUrl}/login`);

    cy.contains("Sign in to Alvor Bank").should("be.visible");
    cy.get('input[type="email"]').should("exist");
    cy.get('input[type="password"]').should("exist");
  });

  it("should show validation errors on empty submit", () => {
    cy.visit(`${baseUrl}/login`);

    cy.get('button[type="submit"]').click();

    cy.contains("Please enter a valid email address.").should("be.visible");
    cy.contains("Your password must be at least 8 characters long.").should(
      "be.visible"
    );
  });

  it("should allow login for a seeded user (no 2FA)", () => {
    cy.visit(`${baseUrl}/login`);

    cy.get('input[type="email"]').type(devUserEmail);
    cy.get('input[type="password"]').type(devUserPassword);
    cy.get('button[type="submit"]').click();

    // Expect redirect to /me/security
    cy.url().should("include", "/me/security");
    cy.contains("My security").should("be.visible");
  });

  it("should show generic success message on forgot password", () => {
    cy.visit(`${baseUrl}/forgot-password`);

    cy.get('input[type="email"]').type(devUserEmail);
    cy.get('button[type="submit"]').click();

    cy.contains("Check your email").should("be.visible");
    cy.contains(
      "If this email exists in our system, we sent you a link with instructions to reset your password."
    ).should("be.visible");
  });
});
```

Notes:

- This assumes you have a **seeded IAM user** with the given email/password and **no 2FA**.  
  Adjust `devUserEmail` and `devUserPassword` to match your backend seeding configuration.
- For 2FA-specific E2E tests, you can either:
  - Use Cypress `cy.intercept` to mock `/2fa/verify` responses, or
  - Drive a real flow using a dev 2FA code from the backend test email sender.  
    (This is advanced and can be added in an appendix if needed.)

Run E2E locally:

```bash
cd digital-banking-suite/src/frontend

npx nx e2e iam-portal-e2e
```

If the backend is running and seeded as expected, all tests should pass.

---

## 11.13 Local Sanity Check & Closing the Chapter

Before we consider the IAM auth chapter “complete” for a real bank, we must pass a **frontend quality gate**:

- Lint
- Unit tests
- E2E tests

…and then follow the **branch strategy** to merge safely into `develop`.

---

### 11.13.1 Run the full frontend quality gate

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

Fix any issues before proceeding. The expectation for a banking system is:

- **0 lint errors**
- **All unit tests passing**
- **All E2E tests passing** (against your dev IAM backend)

---

### 11.13.2 Commit & push the Chapter 11 work

From the repository root:

```bash
cd digital-banking-suite

git status
git add .
git commit -m "feat(ch11): implement IAM auth flows (login, 2FA, password, security)"
git push origin feature/ch11-frontend-iam-auth-flows
```

---

### 11.13.3 Open a Pull Request to develop

In your Git hosting (GitHub, Azure DevOps, etc.):

- **Source branch**: `feature/ch11-frontend-iam-auth-flows`
- **Target branch**: `develop`
- Title suggestion:
  - `feat(ch11): IAM auth flows in Angular (login, 2FA, password, security)`
- Summary:
  - Implemented full IAM auth flows in Angular 21 (login + 2FA + email confirmation + password flows).
  - Added `AuthApiService`, `AuthStateService`, JWT interceptor, guards.
  - Implemented My Security page (change password, 2FA toggle).
  - Added unit tests and E2E tests for core flows.

Once CI passes and the PR is reviewed, merge into `develop`.  
(As in previous chapters, you can keep the feature branch for historical reference.)

---

## 11.14 Summary & Next Steps

In this chapter we built a **bank-grade IAM authentication UI** on top of the Alvor Bank backend:

- **Auth infrastructure**:

  - Strongly typed models under `core/auth/models`.
  - `AuthApiService` encapsulating all IAM auth HTTP calls.
  - `AuthStateService` managing JWT sessions, with localStorage persistence.
  - JWT **HTTP interceptor** attaching `Authorization: Bearer ...` to IAM calls.
  - Route guards (`authGuard`, `guestGuard`) controlling access to routes.

- **Employee-facing auth flows**:

  - Login page with reactive form, validation, and `requiresTwoFactor` handling.
  - 2FA page for entering verification codes and completing login.
  - Forgot password and reset password flows with secure, generic messaging.
  - Email confirmation page driven by `userId` and `token` query params.
  - My Security page for:
    - Changing password.
    - Enabling/disabling 2FA with current password confirmation.

- **Shell & UX**:

  - A cohesive `ShellLayout` with Alvor Bank branding.
  - Navigation that responds to auth state.
  - Consistent visual language using the design system from Chapter 10.

- **Quality gates**:
  - Unit tests for `AuthApiService` and `AuthStateService`.
  - Optional guard tests.
  - Cypress E2E tests for login and forgot password flows.
  - A full frontend quality gate (lint + unit + E2E) before merging into `develop`.

With this in place, the IAM portal is no longer just a collection of screens — it behaves like a **real banking IAM front-end**, integrated with your .NET 10 FastEndpoints backend and protected by automated tests.

In the **next frontend chapter**, we will:

- Implement the **Admin Employees management UI**:
  - Employees grid with paging and search (`GET /api/iam/admin/employees`).
  - Employee details/edit screen (`GET/PUT /api/iam/admin/employee(s)`).
  - Activate/deactivate actions.
- Reuse the design system and auth infrastructure we’ve built here.
- Introduce more structured data-fetching patterns suitable for a growing banking portal.
