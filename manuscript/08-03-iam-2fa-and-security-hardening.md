## 8.4 Two-Factor Authentication (2FA) with Email Codes

In this section we add a **pragmatic 2FA flow** on top of our existing IAM service:

- If an employee has **2FA enabled**, login becomes a **two-step flow**:
  1. Enter email + password → system emails a one-time code
  2. Enter the one-time code → system returns a JWT access token

We keep the architecture consistent:

- **FastEndpoints** on the edge
- **CQRS with MediatR** in the Application layer
- **ASP.NET Core Identity** for user, password, and token management
- `IEmailSender` from the previous section for sending codes
- `ICurrentUser` for enabling/disabling 2FA for the current employee

---

### 8.4.1 Branch Strategy for This Section

New big section, new branch.

We still use:

- `main` → latest **stable, published** state
- `develop` → integration branch for ongoing **Part II** backend work
- One branch per section as a **permanent reference**

For this section:

- **Branch name:** `part2-chapter08-iam-2fa-and-security-hardening`

From the repo root:

```bash
git checkout develop
git pull origin develop

git checkout -b part2-chapter08-iam-2fa-and-security-hardening
```

We’ll commit only the **2FA + security hardening** changes to this branch and open a PR into `develop` at the end.

---

### 8.4.2 2FA Flow Overview

We’ll implement a **simple email-based 2FA**:

1. **Login Step (email + password)**

   - If `TwoFactorEnabled == false` → behave exactly like before (return JWT).
   - If `TwoFactorEnabled == true`:
     - Generate a 2FA code using Identity’s token provider.
     - Email the code to the employee.
     - Return: `requiresTwoFactor = true` and the `userId` (no JWT yet).

2. **Verify 2FA Code**

   - Endpoint receives `userId` + `code`.
   - Verifies the 2FA code via `UserManager.VerifyTwoFactorTokenAsync`.
   - If valid:
     - Generate JWT for that user.
     - Return it to the client.

3. **Enable / Disable 2FA**
   - Endpoints for the **currently logged-in employee**:
     - `POST /api/iam/auth/2fa/enable`
     - `POST /api/iam/auth/2fa/disable`
   - We require the current password to avoid accidental or malicious toggling.

This design is intentionally **pragmatic**:

- No extra tables or cache to store 2FA sessions.
- We reuse Identity’s built-in token provider for one-time codes.
- All sensitive checks (`IsActive`, `EmailConfirmed`) are still enforced.

---

### 8.4.3 Login Response Shape with 2FA

We extend the login flow to be able to represent **two types of result**:

- **Normal login** → JWT token (no 2FA required)
- **2FA required** → no token yet, but we know which user we’re verifying

If you already have a `LoginResult` / `LoginResponse` type from a previous section, **add** the new properties to it.  
For completeness, here is a canonical shape you can use or adapt.

Create/update:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/Login/LoginResult.cs`

```csharp
namespace BankingSuite.IAM.Application.Auth.Commands.Login;

public sealed class LoginResult
{
    // When false and AccessToken is set => normal login success.
    // When true => 2FA still pending; AccessToken is null.
    public bool RequiresTwoFactor { get; init; }

    // Used by the frontend to call the 2FA verify endpoint.
    public Guid? UserId { get; init; }

    // Populated only when 2FA is not required or already completed.
    public string? AccessToken { get; init; }

    public DateTimeOffset? ExpiresAt { get; init; }

    // Optional: add RefreshToken, roles, etc if your app uses them.
}
```

The **Login endpoint** will simply return this shape to the frontend.

---

### 8.4.4 Updating LoginCommandHandler for 2FA

We now update the `LoginCommandHandler` to:

- Check if the user has 2FA enabled.
- If not → return normal JWT as before.
- If yes → send a 2FA code via email and return `RequiresTwoFactor = true`.

We assume you already have:

- `LoginCommand` (e.g. `record LoginCommand(string Email, string Password) : IRequest<LoginResult>`)
- A token generator service used in the previous section (replace `IJwtTokenGenerator` with your actual interface name if different).

Example:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/Login/LoginCommandHandler.cs`

```csharp
using BankingSuite.IAM.Application.Auth.Commands.Login;
using BankingSuite.IAM.Application.Common.Interfaces;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.Login;

public sealed class LoginCommandHandler(
    UserManager<ApplicationUser> userManager,
    IJwtTokenGenerator jwtTokenGenerator,
    IEmailSender emailSender)
    : IRequestHandler<LoginCommand, LoginResult>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    private readonly IJwtTokenGenerator _jwtTokenGenerator = jwtTokenGenerator
        ?? throw new ArgumentNullException(nameof(jwtTokenGenerator));

    private readonly IEmailSender _emailSender = emailSender
        ?? throw new ArgumentNullException(nameof(emailSender));

    // Identity's default email provider name
    private const string TwoFactorProvider = TokenOptions.DefaultEmailProvider;

    public async Task<LoginResult> Handle(LoginCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.FindByEmailAsync(request.Email);

        if (user is null || !user.IsActive)
            throw new InvalidOperationException("Invalid credentials.");

        if (!user.EmailConfirmed)
            throw new InvalidOperationException("Email address must be confirmed before logging in.");

        var passwordValid = await _userManager.CheckPasswordAsync(user, request.Password);
        if (!passwordValid)
            throw new InvalidOperationException("Invalid credentials.");

        // If 2FA is not enabled, behave as before.
        if (!user.TwoFactorEnabled)
        {
            var token = await _jwtTokenGenerator.GenerateTokenAsync(user, cancellationToken);

            return new LoginResult
            {
                RequiresTwoFactor = false,
                UserId = user.Id,
                AccessToken = token.AccessToken,
                ExpiresAt = token.ExpiresAt
            };
        }

        // 2FA enabled: generate a one-time code and email it.
        var twoFactorCode = await _userManager.GenerateTwoFactorTokenAsync(user, TwoFactorProvider);

        var subject = "Your Alvor Bank 2FA code";
        var body = $"""
                    <p>Hello {user.FullName ?? user.Email},</p>
                    <p>Your one-time 2FA code is:</p>
                    <p><strong>{twoFactorCode}</strong></p>
                    <p>This code will expire shortly. If you did not attempt to log in, please contact support.</p>
                    """;

        await _emailSender.SendEmailAsync(user.Email!, subject, body, cancellationToken);

        // Return a result that indicates 2FA is still pending.
        return new LoginResult
        {
            RequiresTwoFactor = true,
            UserId = user.Id,
            AccessToken = null,
            ExpiresAt = null
        };
    }
}
```

> **Note:**  
> `IJwtTokenGenerator` is an example interface. Use whatever token service you created in the login section (e.g. `ITokenService`, `IAccessTokenFactory`) and adapt the code accordingly.

Your existing **Login endpoint** (FastEndpoint) only needs to return this `LoginResult` object; the frontend will branch based on `RequiresTwoFactor`.

---

### 8.4.5 CQRS: Verify 2FA Code

Now we create a dedicated command to:

- Verify a 2FA code for a given user
- Return a normal `LoginResult` with a JWT if successful

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/VerifyTwoFactor/VerifyTwoFactorCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/VerifyTwoFactor/VerifyTwoFactorCommandHandler.cs`

**VerifyTwoFactorCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.VerifyTwoFactor;

public sealed record VerifyTwoFactorCommand(
    Guid UserId,
    string Code) : IRequest<LoginResult>;
```

**VerifyTwoFactorCommandHandler**

```csharp
using BankingSuite.IAM.Application.Auth.Commands.Login;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.VerifyTwoFactor;

public sealed class VerifyTwoFactorCommandHandler(
    UserManager<ApplicationUser> userManager,
    IJwtTokenGenerator jwtTokenGenerator)
    : IRequestHandler<VerifyTwoFactorCommand, LoginResult>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    private readonly IJwtTokenGenerator _jwtTokenGenerator = jwtTokenGenerator
        ?? throw new ArgumentNullException(nameof(jwtTokenGenerator));

    private const string TwoFactorProvider = TokenOptions.DefaultEmailProvider;

    public async Task<LoginResult> Handle(VerifyTwoFactorCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.FindByIdAsync(request.UserId.ToString());

        if (user is null || !user.IsActive)
            throw new InvalidOperationException("Invalid 2FA request.");

        if (!user.TwoFactorEnabled)
            throw new InvalidOperationException("Two-factor authentication is not enabled for this user.");

        if (!user.EmailConfirmed)
            throw new InvalidOperationException("Email must be confirmed.");

        var isValid = await _userManager.VerifyTwoFactorTokenAsync(
            user,
            TwoFactorProvider,
            request.Code);

        if (!isValid)
            throw new InvalidOperationException("Invalid or expired 2FA code.");

        var token = await _jwtTokenGenerator.GenerateTokenAsync(user, cancellationToken);

        return new LoginResult
        {
            RequiresTwoFactor = false,
            UserId = user.Id,
            AccessToken = token.AccessToken,
            ExpiresAt = token.ExpiresAt
        };
    }
}
```

---

### 8.4.6 FastEndpoints: 2FA Verify Endpoint

We now expose this as an HTTP endpoint the frontend can call after the user enters the code.

Create:

- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/VerifyTwoFactorEndpoint.cs`

```csharp
using BankingSuite.IAM.Application.Auth.Commands.Login;
using BankingSuite.IAM.Application.Auth.Commands.VerifyTwoFactor;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Auth;

public sealed class VerifyTwoFactorRequest
{
    public Guid UserId { get; init; }
    public string Code { get; init; } = string.Empty;
}

public sealed class VerifyTwoFactorEndpoint(IMediator mediator)
    : Endpoint<VerifyTwoFactorRequest, LoginResult>
{
    public override void Configure()
    {
        Post("/api/iam/auth/2fa/verify");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Verify a 2FA code sent via email and issue a JWT for the employee.";
        });
    }

    public override async Task HandleAsync(VerifyTwoFactorRequest req, CancellationToken ct)
    {
        var cmd = new VerifyTwoFactorCommand(
            UserId: req.UserId,
            Code: req.Code);

        var result = await mediator.Send(cmd, ct);

        await SendOkAsync(result, ct);
    }
}
```

From the frontend’s perspective, the flow becomes:

1. Call `POST /api/iam/auth/login` with email + password.
2. If `RequiresTwoFactor == true`, show a screen asking for the code.
3. Call `POST /api/iam/auth/2fa/verify` with `userId` and `code`.
4. On success, store `AccessToken` as usual.

---

### 8.4.7 CQRS: Enable / Disable 2FA

Employees should be able to enable or disable 2FA for their own accounts.

We’ll:

- Use `ICurrentUser` to know who is logged in.
- Require the **current password** to avoid unauthorized changes.

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/TwoFactor/EnableTwoFactorCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/TwoFactor/EnableTwoFactorCommandHandler.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/TwoFactor/DisableTwoFactorCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/TwoFactor/DisableTwoFactorCommandHandler.cs`

**EnableTwoFactorCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.TwoFactor;

public sealed record EnableTwoFactorCommand(string CurrentPassword) : IRequest;
```

**EnableTwoFactorCommandHandler**

```csharp
using BankingSuite.IAM.Application.Common.Interfaces;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.TwoFactor;

public sealed class EnableTwoFactorCommandHandler(
    UserManager<ApplicationUser> userManager,
    ICurrentUser currentUser)
    : IRequestHandler<EnableTwoFactorCommand>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    private readonly ICurrentUser _currentUser = currentUser
        ?? throw new ArgumentNullException(nameof(currentUser));

    public async Task<Unit> Handle(EnableTwoFactorCommand request, CancellationToken cancellationToken)
    {
        if (!_currentUser.IsAuthenticated)
            throw new InvalidOperationException("User must be authenticated.");

        var user = await _userManager.FindByIdAsync(_currentUser.UserId.ToString());

        if (user is null)
            throw new KeyNotFoundException("Current user not found.");

        var passwordValid = await _userManager.CheckPasswordAsync(user, request.CurrentPassword);
        if (!passwordValid)
            throw new InvalidOperationException("Invalid password.");

        user.TwoFactorEnabled = true;

        var result = await _userManager.UpdateAsync(user);
        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to enable 2FA: {errors}");
        }

        return Unit.Value;
    }
}
```

**DisableTwoFactorCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.TwoFactor;

public sealed record DisableTwoFactorCommand(string CurrentPassword) : IRequest;
```

**DisableTwoFactorCommandHandler**

```csharp
using BankingSuite.IAM.Application.Common.Interfaces;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.TwoFactor;

public sealed class DisableTwoFactorCommandHandler(
    UserManager<ApplicationUser> userManager,
    ICurrentUser currentUser)
    : IRequestHandler<DisableTwoFactorCommand>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    private readonly ICurrentUser _currentUser = currentUser
        ?? throw new ArgumentNullException(nameof(currentUser));

    public async Task<Unit> Handle(DisableTwoFactorCommand request, CancellationToken cancellationToken)
    {
        if (!_currentUser.IsAuthenticated)
            throw new InvalidOperationException("User must be authenticated.");

        var user = await _userManager.FindByIdAsync(_currentUser.UserId.ToString());

        if (user is null)
            throw new KeyNotFoundException("Current user not found.");

        var passwordValid = await _userManager.CheckPasswordAsync(user, request.CurrentPassword);
        if (!passwordValid)
            throw new InvalidOperationException("Invalid password.");

        user.TwoFactorEnabled = false;

        var result = await _userManager.UpdateAsync(user);
        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to disable 2FA: {errors}");
        }

        return Unit.Value;
    }
}
```

---

### 8.4.8 FastEndpoints: Enable / Disable 2FA

Create:

- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/TwoFactorSettingsEndpoints.cs`

```csharp
using BankingSuite.IAM.Application.Auth.Commands.TwoFactor;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Auth;

public sealed class TwoFactorUpdateRequest
{
    public string CurrentPassword { get; init; } = string.Empty;
}

public sealed class EnableTwoFactorEndpoint(IMediator mediator)
    : Endpoint<TwoFactorUpdateRequest>
{
    public override void Configure()
    {
        Post("/api/iam/auth/2fa/enable");
        Roles("Employee", "IamAdmin", "SuperAdmin"); // adjust role names as needed
        Summary(s =>
        {
            s.Summary = "Enable 2FA for the current employee using their password.";
        });
    }

    public override async Task HandleAsync(TwoFactorUpdateRequest req, CancellationToken ct)
    {
        await mediator.Send(new EnableTwoFactorCommand(req.CurrentPassword), ct);
        await SendNoContentAsync(ct);
    }
}

public sealed class DisableTwoFactorEndpoint(IMediator mediator)
    : Endpoint<TwoFactorUpdateRequest>
{
    public override void Configure()
    {
        Post("/api/iam/auth/2fa/disable");
        Roles("Employee", "IamAdmin", "SuperAdmin");
        Summary(s =>
        {
            s.Summary = "Disable 2FA for the current employee using their password.";
        });
    }

    public override async Task HandleAsync(TwoFactorUpdateRequest req, CancellationToken ct)
    {
        await mediator.Send(new DisableTwoFactorCommand(req.CurrentPassword), ct);
        await SendNoContentAsync(ct);
    }
}
```

Now the IAM service supports:

- Optional 2FA per employee
- Email-based 2FA codes during login
- Separate 2FA verification endpoint
- Simple enable/disable endpoints, protected by the employee’s password

---

## 8.5 Security Hardening & Tests

We’ll close this chapter with a few **hardening steps** and testing recommendations so IAM behaves like a real banking system, not a demo.

---

### 8.5.1 Identity Lockout & 2FA Failure Handling

Identity already supports lockout:

- Max failed access attempts
- Lockout time span

Ensure in your Identity configuration (where `AddIdentity` is called) you have sane values, e.g.:

```csharp
services.AddIdentity<ApplicationUser, ApplicationRole>(options =>
    {
        options.Lockout.MaxFailedAccessAttempts = 5;
        options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
        options.Lockout.AllowedForNewUsers = true;

        // Password & user options already configured in previous chapters.
    })
    .AddEntityFrameworkStores<IamDbContext>()
    .AddDefaultTokenProviders();
```

This protects both:

- Normal password logins
- 2FA token verifications

Because Identity increments access failed count on those operations too (depending on how you call them).

---

### 8.5.2 Consistent Checks in Login + 2FA

Double-check that **both**:

- `LoginCommandHandler`
- `VerifyTwoFactorCommandHandler`

apply the same basic rules:

- `user.IsActive == true`
- `user.EmailConfirmed == true`
- `user.LockoutEnd` not in the future (handled by Identity lockout)

This prevents edge cases where a deactivated or unconfirmed user might still complete login via 2FA.

---

### 8.5.3 Logging & Auditing (Optional but Recommended)

For a real banking environment, you’d also want structured logging around IAM events:

- Login success / failure
- 2FA requested / verified / failed
- 2FA enabled / disabled
- Password changed / reset

At minimum, consider adding logging statements in your handlers (e.g. using `ILogger<T>`).  
Full audit trails can be added later (e.g. an `AuditLog` table or external SIEM integration).

---

### 8.5.4 Integration Tests to Push Coverage Toward 80%+

To push IAM coverage high (80%+), add tests for:

1. **Admin Employee Management** (from `08-01`):

   - List employees (with active/inactive filters)
   - Get employee details
   - Update name/phone/roles
   - Activate / deactivate

2. **Email Flows** (from `08-02`):

   - Confirm email success and failure
   - Resend confirmation (user exists / does not exist / already confirmed)
   - Forgot password (confirmed vs unconfirmed users)
   - Reset password (valid token / invalid token)
   - Change password (correct vs incorrect current password)

3. **2FA Flows** (this file):
   - Login with 2FA disabled → token returned, `RequiresTwoFactor = false`
   - Login with 2FA enabled → email code generated, `RequiresTwoFactor = true`
   - Verify 2FA with valid code → token returned
   - Verify 2FA with invalid code → failure, no token
   - Enable 2FA with correct password
   - Enable 2FA with wrong password
   - Disable 2FA with correct/wrong password
   - Ensure inactive or unconfirmed users cannot log in or complete 2FA

You can reuse the same **test harness** from earlier chapters:

- Spin up IAM API in-memory (or with Testcontainers + PostgreSQL)
- Seed one or more `ApplicationUser` records
- Call endpoints via HTTP client using FastEndpoints’ testing utilities or plain `HttpClient`
- Assert HTTP status codes, response shapes, and side effects (e.g. `TwoFactorEnabled` flag)

Once tests are passing and coverage is improved:

```bash
git status

git add \
  src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/Login/LoginResult.cs \
  src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/Login/LoginCommandHandler.cs \
  src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/VerifyTwoFactor/** \
  src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/TwoFactor/** \
  src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/** \
  tests/** # your IAM test projects, if updated

git commit -m "Chapter 08: add 2FA and security hardening to IAM"
git push --set-upstream origin part2-chapter08-iam-2fa-and-security-hardening
```

At this point, our IAM service is ready for a **real Angular 21 + Nx** frontend:

- Admins can manage employees (Status, roles, 2FA flag)
- Employees can confirm email, reset password, change password
- Optional email-based 2FA is supported during login

In the next part of the book, we’ll build the **frontend IAM UI** that sits on top of these endpoints and wires everything into a real banking login experience.
