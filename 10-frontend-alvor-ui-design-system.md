# Chapter 10 — Alvor Bank UI Design System

In Chapter 9 we set up the **IAM Employee Portal** with Angular 21 + Nx and a clean route structure.  
The app works, but visually it still looks like a starter project.

In this chapter we define the **Alvor Bank UI Design System**:

- A consistent **visual language** (colors, typography, spacing).
- A set of **design tokens** encoded as CSS variables / SCSS.
- Reusable **layout patterns** for pages and management tables.
- Core **UI primitives** (buttons, tags, alerts) that we’ll reuse across all apps.

We will implement enough code so you can see the design system live in the IAM portal, but we will **not** yet wire real backend data or complex forms. That starts in the next chapter when we apply this system to real IAM flows.

---

## 10.1 Why a Design System for a Bank?

Banks need more than a pretty UI:

- They carry **high-risk actions** (payments, access, approvals).
- They operate under **regulation and audit**.
- They are built by multiple teams in parallel over many years.

Without a design system you quickly get:

- Inconsistent forms and tables.
- Different colors for the same status.
- Copy-paste CSS that is impossible to refactor.

Alvor Bank’s design system aims to:

- Look **trustworthy and calm**, not flashy.
- Give engineers a **single source of truth** for colors, spacing, and components.
- Scale from the IAM portal to:
  - Operations dashboards.
  - Back-office tools.
  - Online banking for customers.

We will implement this via **design tokens** and a small set of reusable CSS classes that sit on top of Angular components.

---

## 10.2 UX Principles for Internal Banking Tools

Before we touch code, we agree on a few UX principles. These will guide decisions later when we design screens.

1. **Trust & clarity over flashiness**

   - Conservative use of animation.
   - High contrast for text and critical actions.
   - Simple, readable layouts.

2. **Consistency & predictability**

   - Same patterns for forms, tables, tags, and alerts across all screens.
   - Users should not have to “re-learn” the UI per module.

3. **Speed for experts**

   - Clean tab order and keyboard-friendly forms.
   - Clear default actions (primary button) for common tasks.

4. **Safety**

   - Destructive actions (e.g. deactivate employee) are visually distinct.
   - Critical flows have confirmation patterns.

5. **Feedback**
   - Success/error banners or toasts after important actions.
   - Loading states for long operations.

We will reference these principles when designing IAM screens like login, 2FA, and employee management.

---

## 10.3 Alvor Bank Visual Identity

Alvor Bank’s visual identity is:

- **Modern corporate** — serious, not playful.
- **Tech-forward** — clean typography, good use of whitespace.
- **Blue-centric** — a deep navy primary, with a precise accent for actions.

### 10.3.1 Color palette (semantic tokens)

Instead of scattering hex values through the codebase, we use **semantic tokens**.

The table below shows the core palette:

| Token                      | Purpose                             | Example Hex |
| -------------------------- | ----------------------------------- | ----------- |
| `--bs-color-primary`       | Brand/navy, primary actions         | `#0b1f3b`   |
| `--bs-color-primary-soft`  | Hover states, subtle emphasis       | `#163a63`   |
| `--bs-color-accent`        | Secondary actions, highlights       | `#0ea5e9`   |
| `--bs-color-accent-soft`   | Accent backgrounds, info tags       | `#e0f2fe`   |
| `--bs-color-bg`            | App background                      | `#f3f4f6`   |
| `--bs-color-surface`       | Cards, page surfaces                | `#ffffff`   |
| `--bs-color-border-subtle` | Standard borders                    | `#e5e7eb`   |
| `--bs-color-border-strong` | Emphasized borders (panels, tables) | `#cbd5f5`   |
| `--bs-color-text-main`     | Main text                           | `#111827`   |
| `--bs-color-text-muted`    | Secondary text                      | `#6b7280`   |
| `--bs-color-success`       | Success states                      | `#16a34a`   |
| `--bs-color-warning`       | Warning states                      | `#d97706`   |
| `--bs-color-danger`        | Error/destructive states            | `#b91c1c`   |
| `--bs-color-info`          | Informational states                | `#0284c7`   |

In the manuscript, we reference this diagram:

> `![Alvor Bank Color Palette](images/10-ui-design-system/alvor-bank-color-palette.png)`

### 10.3.2 Typography

We use a **system font stack** for performance and native feel:

- `system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif`

Sizes (relative to a 16px base):

- `--bs-font-size-xs`: `0.75rem` (12px)
- `--bs-font-size-sm`: `0.875rem` (14px)
- `--bs-font-size-md`: `1rem` (16px)
- `--bs-font-size-lg`: `1.125rem` (18px)
- `--bs-font-size-xl`: `1.25rem` (20px)

Typical usage:

- Page titles: `lg` or `xl`.
- Section headers: `md` or `lg`.
- Body text: `sm` or `md`.
- Captions and meta: `xs`.

We keep headings **modest** — this is a bank, not a marketing landing page.

### 10.3.3 Spacing, radius, and elevation

We standardize spacing on a 4px grid:

- `--bs-space-0`: `0`
- `--bs-space-1`: `0.25rem` (4px)
- `--bs-space-2`: `0.5rem` (8px)
- `--bs-space-3`: `0.75rem` (12px)
- `--bs-space-4`: `1rem` (16px)
- `--bs-space-6`: `1.5rem` (24px)
- `--bs-space-8`: `2rem` (32px)

Radius:

- `--bs-radius-sm`: `0.25rem`
- `--bs-radius-md`: `0.5rem`
- `--bs-radius-lg`: `0.75rem`

Shadows:

- `--bs-shadow-soft`: `0 1px 2px rgba(15, 23, 42, 0.05)`
- `--bs-shadow-strong`: `0 8px 30px rgba(15, 23, 42, 0.12)`

### 10.3.4 Breakpoints

We design IAM as desktop-first but keep it usable down to tablet size:

- `--bs-breakpoint-sm`: `640px`
- `--bs-breakpoint-md`: `768px`
- `--bs-breakpoint-lg`: `1024px`
- `--bs-breakpoint-xl`: `1280px`

Later we can refine layout for smaller devices as needed.

---

## 10.4 Encoding Design Tokens in Code

All visual decisions become **tokens**, defined once and reused everywhere.

For now, we keep tokens in the IAM app. In a later refactor we’ll move them into a shared library.

### 10.4.1 Creating `_tokens.scss`

Create a `styles` subfolder inside the IAM app (if it doesn’t exist yet):

- `apps/iam-portal/src/styles/_tokens.scss`

Add:

```scss
:root {
  /* Brand & palette */
  --bs-color-primary: #0b1f3b;
  --bs-color-primary-strong: #061024;
  --bs-color-primary-soft: #163a63;

  --bs-color-accent: #0ea5e9;
  --bs-color-accent-soft: #e0f2fe;

  --bs-color-bg: #f3f4f6;
  --bs-color-surface: #ffffff;
  --bs-color-border-subtle: #e5e7eb;
  --bs-color-border-strong: #cbd5f5;

  --bs-color-text-main: #111827;
  --bs-color-text-muted: #6b7280;
  --bs-color-text-on-primary: #ffffff;

  --bs-color-success: #16a34a;
  --bs-color-warning: #d97706;
  --bs-color-danger: #b91c1c;
  --bs-color-info: #0284c7;

  /* Typography */
  --bs-font-size-xs: 0.75rem;
  --bs-font-size-sm: 0.875rem;
  --bs-font-size-md: 1rem;
  --bs-font-size-lg: 1.125rem;
  --bs-font-size-xl: 1.25rem;

  /* Spacing */
  --bs-space-0: 0;
  --bs-space-1: 0.25rem;
  --bs-space-2: 0.5rem;
  --bs-space-3: 0.75rem;
  --bs-space-4: 1rem;
  --bs-space-6: 1.5rem;
  --bs-space-8: 2rem;

  /* Radius */
  --bs-radius-sm: 0.25rem;
  --bs-radius-md: 0.5rem;
  --bs-radius-lg: 0.75rem;

  /* Shadows */
  --bs-shadow-soft: 0 1px 2px rgba(15, 23, 42, 0.05);
  --bs-shadow-strong: 0 8px 30px rgba(15, 23, 42, 0.12);
}
```

If we ever want to add a dark theme, we can introduce a `.theme-dark` class that overrides these tokens.

### 10.4.2 Wiring tokens into `styles.scss`

Open `apps/iam-portal/src/styles.scss` and update it to:

```scss
@use "./styles/tokens";

html,
body {
  height: 100%;
  margin: 0;
  font-family: system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  background-color: var(--bs-color-bg);
  color: var(--bs-color-text-main);
}

*,
*::before,
*::after {
  box-sizing: border-box;
}

/* Page container used in IAM placeholders (from Chapter 9),
   now refactored to use tokens. */
.page {
  max-width: 640px;
  margin: var(--bs-space-6) auto;
  padding: var(--bs-space-4);
  border-radius: var(--bs-radius-lg);
  background-color: var(--bs-color-surface);
  box-shadow: var(--bs-shadow-soft);
}

.page h1 {
  font-size: var(--bs-font-size-xl);
  margin-bottom: var(--bs-space-2);
}

.page p {
  font-size: var(--bs-font-size-sm);
  color: var(--bs-color-text-muted);
}

.page ul {
  padding-left: 1.2rem;
  color: var(--bs-color-text-muted);
}
```

> In Manuscript:  
> `![Design Tokens Overview Map](images/10-ui-design-system/design-tokens-overview-map.png)`

This diagram helps the reader visualise how colors, spacing, and typography are controlled via tokens.

### 10.4.3 Future shared UI library

Later, when we introduce a customer-facing online banking portal, we’ll move `_tokens.scss` into a shared library:

- `libs/ui/styles/src/_tokens.scss`

Then each app (IAM, online banking, etc.) will import the shared tokens.

For now, keeping tokens inside `iam-portal` keeps the chapter focused.

---

## 10.5 Layout System

The IAM portal already has an application shell:

- Header with bank brand and user info.
- Left sidebar for navigation.
- Main content area with `router-outlet`.

Here we formalize the **page layouts** we will use across all screens.

### 10.5.1 Application shell recap

The shell follows this wireframe:

> `![Standard Page Layout Wireframe](images/10-ui-design-system/standard-page-layout-wireframe.png)`

Rules:

- **Header** contains:
  - Bank logo / name: “Alvor Bank”.
  - Portal subtitle: “Employee IAM Portal”.
  - Current user info and quick links (future).
- **Sidebar** contains:
  - IAM navigation grouped by area (Auth, Admin, etc.).
  - It is stable; we do not hide/show it per route.
- **Content** hosts page layouts — we’ll standardize those next.

### 10.5.2 Standard page template

Most IAM pages (login, my security, employee details) follow the same pattern:

- Page title.
- Optional subtitle/description.
- Primary action area (usually at the top-right or within the header).
- Main content card.

We can express a standard header as utility classes that live in `styles.scss`:

```scss
.page-header {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin-bottom: var(--bs-space-4);
}

.page-header__title {
  font-size: var(--bs-font-size-xl);
  font-weight: 600;
}

.page-header__subtitle {
  margin-top: var(--bs-space-1);
  font-size: var(--bs-font-size-sm);
  color: var(--bs-color-text-muted);
}

.page-header__actions {
  display: flex;
  gap: var(--bs-space-2);
}
```

We deliberately implement this as plain CSS classes so any Angular component template can use them.

### 10.5.3 Management table layout

Admin screens such as **Employees list** use a slightly different layout:

- Toolbar with filters (search, toggles).
- Dense table.
- Pagination bar at the bottom.

Wireframe:

> `![Management Table Layout Wireframe](images/10-ui-design-system/management-table-layout-wireframe.png)`

We’ll improve the table visually, but the key concept is:

- Filters and actions above.
- Rows in the middle.
- Pagination controls at the bottom.

We can sketch some utilities now:

```scss
.table-page {
  max-width: 1000px;
  margin: var(--bs-space-4) auto;
}

.table-page__header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: var(--bs-space-3);
}

.table-page__filters {
  display: flex;
  flex-wrap: wrap;
  gap: var(--bs-space-2);
  margin-bottom: var(--bs-space-3);
}

.table-page__footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: var(--bs-space-3);
  font-size: var(--bs-font-size-sm);
  color: var(--bs-color-text-muted);
}
```

The actual table styles and pagination buttons will be defined later when we build the Employees list.

---

## 10.6 UI Component Primitives (CSS-Level)

Before building Angular components, we define **CSS primitives** that make our UI look consistent.

### 10.6.1 Buttons

We standardize buttons with semantic variants:

- `bs-btn bs-btn--primary`
- `bs-btn bs-btn--secondary`
- `bs-btn bs-btn--ghost`
- `bs-btn bs-btn--danger`

Add to `styles.scss`:

```scss
.bs-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.4rem;
  padding: 0.45rem 0.9rem;
  border-radius: var(--bs-radius-md);
  border: 1px solid transparent;
  font-size: var(--bs-font-size-sm);
  font-weight: 500;
  cursor: pointer;
  background-color: transparent;
  color: var(--bs-color-text-main);
  transition: background-color 120ms ease-out, color 120ms ease-out,
    border-color 120ms ease-out, box-shadow 120ms ease-out;
}

.bs-btn:disabled {
  cursor: not-allowed;
  opacity: 0.6;
}

/* Primary */
.bs-btn--primary {
  background-color: var(--bs-color-primary);
  color: var(--bs-color-text-on-primary);
  border-color: var(--bs-color-primary);
}

.bs-btn--primary:hover:not(:disabled) {
  background-color: var(--bs-color-primary-soft);
}

/* Secondary */
.bs-btn--secondary {
  background-color: var(--bs-color-surface);
  border-color: var(--bs-color-border-strong);
  color: var(--bs-color-primary);
}

.bs-btn--secondary:hover:not(:disabled) {
  background-color: #e5edff;
}

/* Ghost */
.bs-btn--ghost {
  background-color: transparent;
  border-color: transparent;
  color: var(--bs-color-primary);
}

.bs-btn--ghost:hover:not(:disabled) {
  background-color: rgba(15, 23, 42, 0.04);
}

/* Danger */
.bs-btn--danger {
  background-color: var(--bs-color-danger);
  border-color: var(--bs-color-danger);
  color: var(--bs-color-text-on-primary);
}

.bs-btn--danger:hover:not(:disabled) {
  background-color: #7f1d1d;
}

/* Focus outline */
.bs-btn:focus-visible {
  outline: 2px solid var(--bs-color-accent);
  outline-offset: 2px;
}
```

Any Angular template can now use:

```html
<button class="bs-btn bs-btn--primary">Save</button>
<button class="bs-btn bs-btn--secondary">Cancel</button>
<button class="bs-btn bs-btn--danger">Deactivate</button>
```

### 10.6.2 Form fields

We create a couple of generic form layout helpers; the actual Angular form controls will use them in the next chapter.

Add to `styles.scss`:

```scss
.bs-form {
  display: flex;
  flex-direction: column;
  gap: var(--bs-space-3);
}

.bs-form-field {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.bs-form-field__label {
  font-size: var(--bs-font-size-sm);
  font-weight: 500;
}

.bs-form-field__label--required::after {
  content: " *";
  color: var(--bs-color-danger);
}

.bs-form-field__control input,
.bs-form-field__control select {
  width: 100%;
  padding: 0.45rem 0.6rem;
  border-radius: var(--bs-radius-md);
  border: 1px solid var(--bs-color-border-subtle);
  font-size: var(--bs-font-size-sm);
  background-color: var(--bs-color-surface);
  color: var(--bs-color-text-main);
}

.bs-form-field__control input:focus-visible,
.bs-form-field__control select:focus-visible {
  outline: 2px solid var(--bs-color-accent);
  outline-offset: 1px;
  border-color: var(--bs-color-accent);
}

.bs-form-field__hint {
  font-size: var(--bs-font-size-xs);
  color: var(--bs-color-text-muted);
}

.bs-form-field__error {
  font-size: var(--bs-font-size-xs);
  color: var(--bs-color-danger);
}
```

We will use these utilities for IAM login, 2FA, and password forms.

### 10.6.3 Alerts & banners

Alerts provide feedback at the top of the page or near a form.

Add:

```scss
.bs-alert {
  padding: var(--bs-space-2) var(--bs-space-3);
  border-radius: var(--bs-radius-md);
  border: 1px solid transparent;
  font-size: var(--bs-font-size-sm);
  display: flex;
  align-items: flex-start;
  gap: var(--bs-space-2);
}

.bs-alert__title {
  font-weight: 600;
  margin-bottom: 0.1rem;
}

.bs-alert__body {
  margin: 0;
}

/* Variants */
.bs-alert--info {
  background-color: var(--bs-color-accent-soft);
  border-color: var(--bs-color-accent);
  color: var(--bs-color-text-main);
}

.bs-alert--success {
  background-color: #dcfce7;
  border-color: var(--bs-color-success);
}

.bs-alert--warning {
  background-color: #fffbeb;
  border-color: var(--bs-color-warning);
}

.bs-alert--danger {
  background-color: #fef2f2;
  border-color: var(--bs-color-danger);
}
```

Usage example (later for password resets, etc.):

```html
<div class="bs-alert bs-alert--success">
  <div>
    <div class="bs-alert__title">Password changed</div>
    <p class="bs-alert__body">Your password was updated successfully.</p>
  </div>
</div>
```

### 10.6.4 Status tags / chips

Status tags help quickly scan employee rows (active/inactive, 2FA enabled, etc.).

Add:

```scss
.bs-tag {
  display: inline-flex;
  align-items: center;
  padding: 0.1rem 0.45rem;
  border-radius: 999px;
  font-size: var(--bs-font-size-xs);
  font-weight: 500;
  border: 1px solid transparent;
}

/* Variants */
.bs-tag--success {
  background-color: #dcfce7;
  color: #166534;
  border-color: #bbf7d0;
}

.bs-tag--danger {
  background-color: #fee2e2;
  color: #b91c1c;
  border-color: #fecaca;
}

.bs-tag--warning {
  background-color: #fef3c7;
  color: #92400e;
  border-color: #fde68a;
}

.bs-tag--info {
  background-color: var(--bs-color-accent-soft);
  color: var(--bs-color-primary);
  border-color: #bae6fd;
}

.bs-tag--neutral {
  background-color: #e5e7eb;
  color: #374151;
  border-color: #d1d5db;
}
```

IAM examples:

- `IsActive = true` → `bs-tag bs-tag--success` (“Active”).
- `EmailConfirmed = false` → `bs-tag bs-tag--warning` (“Pending confirmation”).
- `TwoFactorEnabled = true` → `bs-tag bs-tag--info` (“2FA enabled”).

### 10.6.5 Table basics & pagination

We won’t build a full grid component yet, but we define base table styles and pagination that the Employees list will reuse.

Add:

```scss
.bs-table {
  width: 100%;
  border-collapse: collapse;
  background-color: var(--bs-color-surface);
  border-radius: var(--bs-radius-lg);
  overflow: hidden;
  box-shadow: var(--bs-shadow-soft);
  font-size: var(--bs-font-size-sm);
}

.bs-table th,
.bs-table td {
  padding: 0.6rem 0.75rem;
  border-bottom: 1px solid var(--bs-color-border-subtle);
  text-align: left;
}

.bs-table thead {
  background-color: #f9fafb;
}

.bs-table th {
  font-weight: 600;
  font-size: var(--bs-font-size-xs);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  color: var(--bs-color-text-muted);
}

.bs-table tbody tr:hover {
  background-color: #f3f4ff;
}

/* Pagination */
.bs-pagination {
  display: inline-flex;
  align-items: center;
  border-radius: var(--bs-radius-md);
  border: 1px solid var(--bs-color-border-subtle);
  overflow: hidden;
}

.bs-pagination__button {
  min-width: 2.25rem;
  padding: 0.3rem 0.6rem;
  text-align: center;
  font-size: var(--bs-font-size-sm);
  border-right: 1px solid var(--bs-color-border-subtle);
  background-color: var(--bs-color-surface);
  cursor: pointer;
}

.bs-pagination__button:last-child {
  border-right: none;
}

.bs-pagination__button--active {
  background-color: var(--bs-color-primary);
  color: var(--bs-color-text-on-primary);
  border-color: var(--bs-color-primary);
}

.bs-pagination__button--disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

The wireframe with pagination:

> `![Management Table Layout Wireframe](images/10-ui-design-system/management-table-layout-wireframe.png)`

In the Employees list chapter, we will bind this pagination UI to the `PagedResult` returned by the IamService.

---

## 10.7 Accessibility & Error Handling Guidelines

### 10.7.1 Accessibility basics

- **Color contrast**
  - Primary text and buttons use combinations that meet WCAG AA.
- **Focus outlines**
  - `.bs-btn` and form controls use clear focus styles.
- **Keyboard navigation**
  - Every interactive element must be reachable via Tab.
  - Default button for forms (e.g. pressing Enter submits login).

We will enforce these by:

- Using `:focus-visible` styles on buttons and inputs.
- Avoiding “outline: none” unless replaced by a better outline.

### 10.7.2 Error patterns

IAM is security-sensitive. We standardise how errors appear:

- **Field-level errors**
  - Shown below the input using `.bs-form-field__error`.
- **Form-level errors**
  - Shown using `.bs-alert--danger` near the top.

Message style:

- Clear but generic enough to avoid information leaks for security-sensitive flows:
  - “Invalid email or password.”
  - “Your account is locked. Please try again later or contact support.”
- For email existence (`forgot-password`, `resend-confirmation`):
  - Always “If this email exists, we sent a link.” — no difference between valid/invalid email.

We’ll implement these patterns concretely in the IAM login and password flows.

---

## 10.8 Applying the Design System to IAM (Conceptual)

Before we write Angular code, we describe how IAM screens will **use** the design system.

### 10.8.1 Login & 2FA

- **Layout**

  - Uses the `Standard Page` template, centered.
  - Page header: “Sign in to Alvor Bank”.
  - Form with `.bs-form` and `.bs-form-field`.
  - Primary action button: `bs-btn bs-btn--primary`.

- **Errors**

  - Invalid credentials: `.bs-alert bs-alert--danger` at top.
  - Locked account: similar alert with tailored copy.

- **2FA**
  - Same base layout.
  - Field for 2FA code.
  - Info alert telling user where the code was sent.

### 10.8.2 Forgot & reset password

- **Forgot password**

  - Single email field in a `.page`.
  - After submission, always show a `.bs-alert--info` success banner.

- **Reset password**
  - New password + confirm password.
  - Password hint in `.bs-form-field__hint`.
  - Errors as field + form messages.

### 10.8.3 Email confirmation

- Success state:
  - `bs-alert--success` explaining email confirmed and linking to login.
- Error state:
  - `bs-alert--danger` for invalid/expired token.

### 10.8.4 My Security (change password & 2FA toggle)

- Uses the standard page template with two sections:
  - Change password form.
  - 2FA settings (current status tag + enable/disable button).

### 10.8.5 Employees list & details

- **List**

  - Management table layout with:
    - `bs-table` table styling.
    - `bs-tag` for `IsActive`, `EmailConfirmed`, `TwoFactorEnabled`.
    - Filters in `table-page__filters`.

- **Details**
  - Standard page with sections (Profile, Roles, Security).
  - Activate/deactivate actions using `bs-btn--danger` with confirmation.

All of this is now **design-complete**, ready for implementation.

---

## 10.9 Technical Implementation Plan

In the next chapter we will:

- Refactor IAM templates to use:
  - `.page`, `.page-header`, `.bs-btn`, `.bs-form`, `.bs-alert`, `.bs-tag`, `.bs-table`, and `.bs-pagination`.
- Start with:
  - Login + 2FA flows.
  - My Security page.
  - Employees list overview.

We will also ensure:

- Unit tests continue to pass (`nx test iam-portal`).
- E2E tests are updated to assert key UX elements (e.g., presence of “Sign in to Alvor Bank”, visible error alerts).

---

## 10.10 Branch Strategy & CI for Design System Work

As before, we treat this as a feature branch:

- Branch name suggestion:

  ```bash
  git checkout develop
  git checkout -b feature/ch10-alvor-ui-design-system
  ```

- As you implement the tokens and CSS utilities in this chapter:

  ```bash
  cd digital-banking-suite/src/frontend

  npx nx lint iam-portal
  npx nx test iam-portal
  npx nx e2e iam-portal-e2e
  ```

- Commit and push regularly:

  ```bash
  git add apps/iam-portal/src/styles.scss \
          apps/iam-portal/src/styles/_tokens.scss
  git commit -m "feat(ui): add Alvor Bank design tokens and base UI primitives"
  git push origin feature/ch10-alvor-ui-design-system
  ```

Once CI passes, merge back into `develop`.

---

## 10.11 Images & Assets for This Chapter

Place the generated diagrams under:

- `images/10-ui-design-system/alvor-bank-color-palette.png`
- `images/10-ui-design-system/design-tokens-overview-map.png`
- `images/10-ui-design-system/standard-page-layout-wireframe.png`
- `images/10-ui-design-system/management-table-layout-wireframe.png`

Refer to them inside the manuscript as shown earlier so Leanpub picks them up.

---

## 10.12 Summary & Next Steps

In this chapter we:

- Defined **Alvor Bank’s visual identity**:
  - Color palette, typography, spacing, radius, shadows, breakpoints.
- Encoded these decisions in **design tokens** using CSS variables in `_tokens.scss`.
- Created **layout patterns** for standard pages and management tables.
- Implemented **CSS primitives**:
  - Buttons, form layout, alerts, status tags, table and pagination styles.
- Established **accessibility** and **error-handling** guidelines for IAM.

In the **next chapter** we will:

- Apply this design system to real IAM screens.
- Implement the login + 2FA flow UI using the tokens and primitives defined here.
- Wire the screens to the IamService endpoints and start handling real responses and errors.

This is the point where Alvor Bank’s UI starts looking and behaving like a **real banking system**, not just an Angular starter app.
