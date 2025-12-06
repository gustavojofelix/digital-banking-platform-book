## 8.2 Email Confirmation & Password Flows

In this file we add **email-driven security flows** to our IAM service:

- Email confirmation (after registration)
- Resend confirmation email
- Forgot password
- Reset password via email token
- Change password for logged-in employees

We stay consistent with our existing architecture:

- **FastEndpoints** on the edge
- **CQRS with MediatR** in the Application layer
- **ASP.NET Core Identity** for users, tokens and passwords
- Modern C# (records, primary constructors, collection expressions, etc.)

---

### 8.2.1 Branch Strategy for This Section

New section, new branch.

We keep:

- `main` → latest **stable, published** book state
- `develop` → integration branch for **Part II**
- Per-section branches as _permanent references_

For this section:

- **Branch name:** `part2-chapter08-iam-email-confirmation-password-flows`

From the repo root:

```bash
git checkout develop
git pull origin develop

git checkout -b part2-chapter08-iam-email-confirmation-password-flows
```

We’ll commit only the **email confirmation + password flow** changes into this branch and open a PR into `develop` at the end.

---

### 8.2.2 Identity Token Providers (One-Time Setup Check)

Email confirmation and password reset rely on **Identity token providers**.

In your IAM service startup (where Identity is configured), make sure you have:

```csharp
services.AddIdentity<ApplicationUser, ApplicationRole>(options =>
    {
        // existing Identity options, password rules, lockouts, etc.
    })
    .AddEntityFrameworkStores<IamDbContext>()
    .AddDefaultTokenProviders(); // <- important for email + password tokens
```

If you already added `AddDefaultTokenProviders()` in a previous chapter, there is nothing more to do here.

---

### 8.2.3 Email Sending Abstraction

Our flows need to send real emails (or at least log them in dev).

We keep things simple and define a minimal abstraction inside IAM’s Application layer (you can move this to BuildingBlocks later if you want it shared across services):

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Common/Interfaces/IEmailSender.cs`

```csharp
namespace BankingSuite.IAM.Application.Common.Interfaces;

public interface IEmailSender
{
    Task SendEmailAsync(string toEmail, string subject, string htmlBody, CancellationToken cancellationToken = default);
}
```

Your Infrastructure layer will provide a concrete implementation (SMTP, SendGrid, etc.).  
For development, you can have a `LoggingEmailSender` that writes to the logs or debug output.

---

## 8.2 Email Confirmation & Resend

We’ll implement two main flows:

1. **Confirm email** – when the user clicks the link we send
2. **Resend confirmation email** – if an employee lost or never received it

We assume that during registration (`POST /auth/employees`) you either:

- Already send a confirmation email, or
- Will plug in this new `ResendConfirmationEmailCommand` right after successfully creating the user.

Even if you don’t change registration yet, these endpoints will still work for manually testing and confirming users.

---

### 8.2.1 CQRS: Confirm Email

**Use case**

- Input: `userId` + `token` (usually from a URL query string)
- Behavior:
  - Find user
  - Call `UserManager.ConfirmEmailAsync(user, token)`
- Output: success or error

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ConfirmEmail/ConfirmEmailCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ConfirmEmail/ConfirmEmailCommandHandler.cs`

**ConfirmEmailCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.ConfirmEmail;

public sealed record ConfirmEmailCommand(Guid UserId, string Token) : IRequest;
```

**ConfirmEmailCommandHandler**

```csharp
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.ConfirmEmail;

public sealed class ConfirmEmailCommandHandler(UserManager<ApplicationUser> userManager)
    : IRequestHandler<ConfirmEmailCommand>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    public async Task<Unit> Handle(ConfirmEmailCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.FindByIdAsync(request.UserId.ToString());

        if (user is null)
            throw new KeyNotFoundException("User not found.");

        var result = await _userManager.ConfirmEmailAsync(user, request.Token);

        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to confirm email: {errors}");
        }

        return Unit.Value;
    }
}
```

---

### 8.2.2 CQRS: Resend Confirmation Email

**Use case**

- Input: employee email address
- Behavior:
  - Look up user by email
  - If not found → **do not** reveal that to the caller (security)
  - If already confirmed → no-op (or optionally return a validation error)
  - Generate email confirmation token
  - Build confirmation link (configurable base URL)
  - Send email via `IEmailSender`

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ResendConfirmation/ResendConfirmationEmailCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ResendConfirmation/ResendConfirmationEmailCommandHandler.cs`

**ResendConfirmationEmailCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.ResendConfirmation;

public sealed record ResendConfirmationEmailCommand(string Email, string ConfirmationBaseUrl) : IRequest;
```

We pass a `ConfirmationBaseUrl` so the Application layer can generate links like:

`{baseUrl}?userId={id}&token={urlEncodedToken}`

Later, the Angular frontend (or config) will control the base URL.

**ResendConfirmationEmailCommandHandler**

```csharp
using System.Web;
using BankingSuite.IAM.Application.Common.Interfaces;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.ResendConfirmation;

public sealed class ResendConfirmationEmailCommandHandler(
    UserManager<ApplicationUser> userManager,
    IEmailSender emailSender)
    : IRequestHandler<ResendConfirmationEmailCommand>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    private readonly IEmailSender _emailSender = emailSender
        ?? throw new ArgumentNullException(nameof(emailSender));

    public async Task<Unit> Handle(ResendConfirmationEmailCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.FindByEmailAsync(request.Email);

        // Security: do NOT reveal if user exists or not.
        if (user is null)
            return Unit.Value;

        if (user.EmailConfirmed)
            return Unit.Value;

        var token = await _userManager.GenerateEmailConfirmationTokenAsync(user);
        var encodedToken = HttpUtility.UrlEncode(token);

        var confirmationLink = $"{request.ConfirmationBaseUrl}?userId={user.Id}&token={encodedToken}";

        var subject = "Confirm your Alvor Bank employee account";
        var body = $"""
                    <p>Hello {user.FullName ?? user.Email},</p>
                    <p>Please confirm your employee account by clicking the link below:</p>
                    <p><a href="{confirmationLink}">Confirm my account</a></p>
                    <p>If you did not request this, you can safely ignore this email.</p>
                    """;

        await _emailSender.SendEmailAsync(user.Email!, subject, body, cancellationToken);

        return Unit.Value;
    }
}
```

---

### 8.2.3 FastEndpoints: Confirm Email & Resend Confirmation

We now expose these commands via HTTP.

Create:

- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/ConfirmEmailEndpoint.cs`
- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/ResendConfirmationEndpoint.cs`

**ConfirmEmailEndpoint**

Confirmation is usually triggered via a **GET link** from an email:

`GET /api/iam/auth/confirm-email?userId={id}&token={token}`

```csharp
using BankingSuite.IAM.Application.Auth.Commands.ConfirmEmail;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Auth;

public sealed class ConfirmEmailRequest
{
    public Guid UserId { get; init; }
    public string Token { get; init; } = string.Empty;
}

public sealed class ConfirmEmailEndpoint(IMediator mediator)
    : Endpoint<ConfirmEmailRequest>
{
    public override void Configure()
    {
        Get("/api/iam/auth/confirm-email");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Confirm employee email address using the link sent via email.";
        });
    }

    public override async Task HandleAsync(ConfirmEmailRequest req, CancellationToken ct)
    {
        var cmd = new ConfirmEmailCommand(req.UserId, req.Token);

        await mediator.Send(cmd, ct);

        // For now, just return 204. Later we can redirect to a frontend "Email confirmed" page.
        await SendNoContentAsync(ct);
    }
}
```

**ResendConfirmationEndpoint**

Admin, support users, or the employee themselves (unauthenticated) can call:

`POST /api/iam/auth/resend-confirmation`

```csharp
using BankingSuite.IAM.Application.Auth.Commands.ResendConfirmation;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Auth;

public sealed class ResendConfirmationRequest
{
    public string Email { get; init; } = string.Empty;
}

public sealed class ResendConfirmationResponse
{
    public bool Sent { get; init; }
}

public sealed class ResendConfirmationEndpoint(IMediator mediator, IConfiguration configuration)
    : Endpoint<ResendConfirmationRequest, ResendConfirmationResponse>
{
    private readonly IMediator _mediator = mediator;
    private readonly IConfiguration _configuration = configuration;

    public override void Configure()
    {
        Post("/api/iam/auth/resend-confirmation");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Resend email confirmation link to an employee.";
        });
    }

    public override async Task HandleAsync(ResendConfirmationRequest req, CancellationToken ct)
    {
        // This could come from configuration. Example:
        // "Iam:EmailConfirmationBaseUrl": "https://alvorbank.dev/iam/email-confirmation"
        var baseUrl = _configuration["Iam:EmailConfirmationBaseUrl"]
                      ?? "https://localhost:4200/email-confirmation";

        var cmd = new ResendConfirmationEmailCommand(
            Email: req.Email,
            ConfirmationBaseUrl: baseUrl);

        await _mediator.Send(cmd, ct);

        // We intentionally do NOT reveal whether the email exists.
        await SendOkAsync(new ResendConfirmationResponse { Sent = true }, ct);
    }
}
```

At this point, email confirmation is fully wired:

- Employee receives a link with `userId` + `token`
- Clicking it calls our ConfirmEmail endpoint
- We can resend the email as many times as we want

---

## 8.3 Password Flows (Forgot / Reset / Change)

Next we add password flows commonly seen in secure systems:

1. **Forgot password** – user provides email, we send reset link
2. **Reset password** – user uses email token to set a new password
3. **Change password** – logged-in employee changes password with the current one

All of them rely on Identity’s built-in methods:

- `GeneratePasswordResetTokenAsync`
- `ResetPasswordAsync`
- `ChangePasswordAsync`

---

### 8.3.1 CQRS: Forgot Password

**Use case**

- Input: email address
- Behavior:
  - Find user by email
  - If not found → no disclosure (just return)
  - Generate reset token
  - Build reset link
  - Send email

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ForgotPassword/ForgotPasswordCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ForgotPassword/ForgotPasswordCommandHandler.cs`

**ForgotPasswordCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.ForgotPassword;

public sealed record ForgotPasswordCommand(string Email, string ResetBaseUrl) : IRequest;
```

**ForgotPasswordCommandHandler**

```csharp
using System.Web;
using BankingSuite.IAM.Application.Common.Interfaces;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.ForgotPassword;

public sealed class ForgotPasswordCommandHandler(
    UserManager<ApplicationUser> userManager,
    IEmailSender emailSender)
    : IRequestHandler<ForgotPasswordCommand>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    private readonly IEmailSender _emailSender = emailSender
        ?? throw new ArgumentNullException(nameof(emailSender));

    public async Task<Unit> Handle(ForgotPasswordCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.FindByEmailAsync(request.Email);

        if (user is null || !user.EmailConfirmed)
        {
            // Do not reveal that user doesn't exist or isn't confirmed.
            return Unit.Value;
        }

        var token = await _userManager.GeneratePasswordResetTokenAsync(user);
        var encodedToken = HttpUtility.UrlEncode(token);

        var resetLink = $"{request.ResetBaseUrl}?email={HttpUtility.UrlEncode(user.Email)}&token={encodedToken}";

        var subject = "Reset your Alvor Bank employee password";
        var body = $"""
                    <p>Hello {user.FullName ?? user.Email},</p>
                    <p>You requested to reset your employee password.</p>
                    <p>Click the link below to choose a new password:</p>
                    <p><a href="{resetLink}">Reset my password</a></p>
                    <p>If you did not request this, you can safely ignore this email.</p>
                    """;

        await _emailSender.SendEmailAsync(user.Email!, subject, body, cancellationToken);

        return Unit.Value;
    }
}
```

---

### 8.3.2 CQRS: Reset Password

**Use case**

- Input: `email`, `token`, `newPassword`
- Behavior:
  - Find user by email
  - Call `ResetPasswordAsync`

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ResetPassword/ResetPasswordCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ResetPassword/ResetPasswordCommandHandler.cs`

**ResetPasswordCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.ResetPassword;

public sealed record ResetPasswordCommand(
    string Email,
    string Token,
    string NewPassword) : IRequest;
```

**ResetPasswordCommandHandler**

```csharp
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.ResetPassword;

public sealed class ResetPasswordCommandHandler(UserManager<ApplicationUser> userManager)
    : IRequestHandler<ResetPasswordCommand>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    public async Task<Unit> Handle(ResetPasswordCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.FindByEmailAsync(request.Email);

        if (user is null)
        {
            // Do not reveal.
            return Unit.Value;
        }

        var result = await _userManager.ResetPasswordAsync(user, request.Token, request.NewPassword);

        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to reset password: {errors}");
        }

        return Unit.Value;
    }
}
```

---

### 8.3.3 CQRS: Change Password for Logged-in User

For this, we need the current user’s identity.  
If you don’t have it yet, define a simple interface:

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Common/Interfaces/ICurrentUser.cs`

```csharp
namespace BankingSuite.IAM.Application.Common.Interfaces;

public interface ICurrentUser
{
    Guid UserId { get; }
    string? Email { get; }
    bool IsAuthenticated { get; }
}
```

You’ll implement this in the API project using `IHttpContextAccessor` and JWT claims. (We’ll assume that’s in place from earlier chapters.)

Now the command:

- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ChangePassword/ChangePasswordCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/ChangePassword/ChangePasswordCommandHandler.cs`

**ChangePasswordCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Auth.Commands.ChangePassword;

public sealed record ChangePasswordCommand(
    string CurrentPassword,
    string NewPassword) : IRequest;
```

**ChangePasswordCommandHandler**

```csharp
using BankingSuite.IAM.Application.Common.Interfaces;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Application.Auth.Commands.ChangePassword;

public sealed class ChangePasswordCommandHandler(
    UserManager<ApplicationUser> userManager,
    ICurrentUser currentUser)
    : IRequestHandler<ChangePasswordCommand>
{
    private readonly UserManager<ApplicationUser> _userManager = userManager
        ?? throw new ArgumentNullException(nameof(userManager));

    private readonly ICurrentUser _currentUser = currentUser
        ?? throw new ArgumentNullException(nameof(currentUser));

    public async Task<Unit> Handle(ChangePasswordCommand request, CancellationToken cancellationToken)
    {
        if (!_currentUser.IsAuthenticated)
            throw new InvalidOperationException("User must be authenticated to change password.");

        var user = await _userManager.FindByIdAsync(_currentUser.UserId.ToString());

        if (user is null)
            throw new KeyNotFoundException("Current user not found.");

        var result = await _userManager.ChangePasswordAsync(
            user,
            request.CurrentPassword,
            request.NewPassword);

        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to change password: {errors}");
        }

        return Unit.Value;
    }
}
```

---

### 8.3.4 FastEndpoints: Forgot / Reset / Change Password

Now we add endpoints that call these commands.

Create:

- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/ForgotPasswordEndpoint.cs`
- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/ResetPasswordEndpoint.cs`
- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/ChangePasswordEndpoint.cs`

**ForgotPasswordEndpoint**

```csharp
using BankingSuite.IAM.Application.Auth.Commands.ForgotPassword;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Auth;

public sealed class ForgotPasswordRequest
{
    public string Email { get; init; } = string.Empty;
}

public sealed class ForgotPasswordResponse
{
    public bool Sent { get; init; }
}

public sealed class ForgotPasswordEndpoint(IMediator mediator, IConfiguration configuration)
    : Endpoint<ForgotPasswordRequest, ForgotPasswordResponse>
{
    private readonly IMediator _mediator = mediator;
    private readonly IConfiguration _configuration = configuration;

    public override void Configure()
    {
        Post("/api/iam/auth/forgot-password");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Start the password reset flow for an employee.";
        });
    }

    public override async Task HandleAsync(ForgotPasswordRequest req, CancellationToken ct)
    {
        var baseUrl = _configuration["Iam:PasswordResetBaseUrl"]
                      ?? "https://localhost:4200/password-reset";

        var cmd = new ForgotPasswordCommand(
            Email: req.Email,
            ResetBaseUrl: baseUrl);

        await _mediator.Send(cmd, ct);

        await SendOkAsync(new ForgotPasswordResponse { Sent = true }, ct);
    }
}
```

**ResetPasswordEndpoint**

```csharp
using BankingSuite.IAM.Application.Auth.Commands.ResetPassword;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Auth;

public sealed class ResetPasswordRequest
{
    public string Email { get; init; } = string.Empty;
    public string Token { get; init; } = string.Empty;
    public string NewPassword { get; init; } = string.Empty;
}

public sealed class ResetPasswordEndpoint(IMediator mediator)
    : Endpoint<ResetPasswordRequest>
{
    public override void Configure()
    {
        Post("/api/iam/auth/reset-password");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Complete the password reset flow using the email token.";
        });
    }

    public override async Task HandleAsync(ResetPasswordRequest req, CancellationToken ct)
    {
        var cmd = new ResetPasswordCommand(
            Email: req.Email,
            Token: req.Token,
            NewPassword: req.NewPassword);

        await mediator.Send(cmd, ct);

        await SendNoContentAsync(ct);
    }
}
```

**ChangePasswordEndpoint**

```csharp
using BankingSuite.IAM.Application.Auth.Commands.ChangePassword;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Auth;

public sealed class ChangePasswordRequest
{
    public string CurrentPassword { get; init; } = string.Empty;
    public string NewPassword { get; init; } = string.Empty;
}

public sealed class ChangePasswordEndpoint(IMediator mediator)
    : Endpoint<ChangePasswordRequest>
{
    public override void Configure()
    {
        Post("/api/iam/auth/change-password");
        Roles("Employee", "IamAdmin", "SuperAdmin"); // adjust to match your roles
        Summary(s =>
        {
            s.Summary = "Change password for the currently logged-in employee.";
        });
    }

    public override async Task HandleAsync(ChangePasswordRequest req, CancellationToken ct)
    {
        var cmd = new ChangePasswordCommand(
            CurrentPassword: req.CurrentPassword,
            NewPassword: req.NewPassword);

        await mediator.Send(cmd, ct);

        await SendNoContentAsync(ct);
    }
}
```

---

### 8.3.5 Quick Manual Test and Commit

Before writing automated tests, do a quick manual sanity check:

1. **Register** a new employee (if you don’t have one yet) via `POST /auth/employees`.
2. Call `POST /api/iam/auth/resend-confirmation` with the employee’s email and confirm the link works.
3. Call `POST /api/iam/auth/forgot-password` and verify that a reset link is generated (and logged/sent).
4. Use that token with `POST /api/iam/auth/reset-password` to set a new password.
5. Log in, then call `POST /api/iam/auth/change-password` with the current password and a new one.

If everything works:

```bash
git status

git add \
  src/backend/services/iam/BankingSuite.IAM.Application/Common/Interfaces/IEmailSender.cs \
  src/backend/services/iam/BankingSuite.IAM.Application/Common/Interfaces/ICurrentUser.cs \
  src/backend/services/iam/BankingSuite.IAM.Application/Auth/Commands/** \
  src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Auth/**

git commit -m "Chapter 08: add email confirmation and password flows to IAM"
git push --set-upstream origin part2-chapter08-iam-email-confirmation-password-flows
```

In the next file (`08-03-iam-2fa-and-security-hardening.md`), we’ll take this even further by adding a **two-factor authentication (2FA) flow** using one-time codes sent via email and tightening the overall security posture of the IAM service.
