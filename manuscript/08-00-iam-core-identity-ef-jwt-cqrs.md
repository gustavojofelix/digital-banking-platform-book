# Chapter 07 — IAM Service Core: Identity, EF Core, JWT & CQRS

By now you have:

- A **backend solution** (`BankingSuite.Backend.sln`)
- Shared **BuildingBlocks** projects
- An **IAM service skeleton** with a `/health` endpoint using FastEndpoints
- **Backend tests + coverage + CI** from the previous chapters

In this chapter we turn the IAM service into a **real backend microservice** using:

- **ASP.NET Core Identity** for users, roles and security
- **EF Core + PostgreSQL** for persistence
- **JWT tokens** for authentication
- **CQRS + MediatR** to keep application logic clean and testable
- A couple of **core endpoints**:
  - `POST /auth/login` – issue a JWT
  - `POST /auth/employees` – create a bank employee user
- **EF Core migrations** wired to the `iam-db` PostgreSQL container

We’ll keep the IAM feature set small for now (login + create employee) but **structure it correctly** with CQRS and Application-layer handlers. In the **next chapter** we’ll add advanced IAM flows: user listing, activation/deactivation, email confirmation, forgot/reset password, and 2FA.

> **IDE assumption:** All steps here assume you are primarily using **Visual Studio 2026**.  
> If you are using another IDE (Rider, VS Code, etc.), perform the equivalent steps or use `dotnet` CLI.

---

## 7.1 Goals for this chapter

When you finish Chapter 07, you should be able to:

- Start `iam-db` via Docker
- Run the IAM API with **Identity + EF Core + JWT** configured
- Use **CQRS + MediatR** for IAM commands/handlers
- Call `/auth/employees` to create an employee user
- Call `/auth/login` to obtain a **JWT** for that user
- See Identity tables and data in the `iam_service_db` PostgreSQL database

This chapter gives you a **clean, CQRS-based IAM core** that we’ll extend in Chapter 08.

### Branch strategy reminder

As in previous chapters, we follow a **feature-branch per chapter** strategy:

1. Ensure you are on `develop` and up to date:

   ```bash
   git checkout develop
   git pull
   ```

2. Create a feature branch for this chapter:

   ```bash
   git checkout -b feature/ch07-iam-core
   ```

3. Do **all work for Chapter 07** on this branch.
4. At the end of the chapter, you will:
   - Commit your changes
   - Push `feature/ch07-iam-core`
   - Open a Pull Request into `develop`
   - Ensure the **Backend Tests & Coverage** GitHub Actions workflow passes
   - Merge and tag the chapter state

We’ll repeat this pattern for every chapter.

---

## 7.2 Quick recap of the backend structure

Your backend layout should look something like this:

```text
digital-banking-suite/
└─ src/
   └─ backend/
      ├─ BankingSuite.Backend.sln
      ├─ BuildingBlocks/
      │  ├─ BankingSuite.BuildingBlocks.Domain/
      │  ├─ BankingSuite.BuildingBlocks.Application/
      │  └─ BankingSuite.BuildingBlocks.Infrastructure/
      └─ Services/
         └─ IamService/
            ├─ BankingSuite.IamService.Domain/
            ├─ BankingSuite.IamService.Application/
            ├─ BankingSuite.IamService.Infrastructure/
            └─ BankingSuite.IamService.API/

tests/
├─ BankingSuite.IamService.UnitTests/
└─ BankingSuite.IamService.IntegrationTests/
```

In this chapter we focus on **these four projects**:

- `BankingSuite.IamService.Domain`
- `BankingSuite.IamService.Application`
- `BankingSuite.IamService.Infrastructure`
- `BankingSuite.IamService.API`

and on adding **MediatR & CQRS** into:

- `BankingSuite.BuildingBlocks.Application`
- `BankingSuite.IamService.Application`
- `BankingSuite.IamService.Infrastructure` (for DI registration)

---

## 7.3 Introduce CQRS & MediatR

We want:

- **Application layer** to expose **commands/queries** as small, composable record types
- **Handlers** to contain IAM logic (login, create employee, etc.)
- **Endpoints** to be very thin: map HTTP → command → HTTP

### 7.3.1 Add MediatR to BuildingBlocks.Application

In Visual Studio 2026:

1. Right-click `BankingSuite.BuildingBlocks.Application` → **Manage NuGet Packages…**
2. On the **Browse** tab, search for `MediatR`
3. Install **`MediatR`** (for .NET 10)

Now add CQRS base interfaces.

In `BankingSuite.BuildingBlocks.Application`:

1. Right-click the project → **Add → New Folder…** → `CQRS`
2. Right-click `CQRS` → **Add → Class…** → name it `CqrsAbstractions.cs`

Replace with:

```csharp
using MediatR;

namespace BankingSuite.BuildingBlocks.Application.CQRS;

public interface ICommand<out TResponse> : IRequest<TResponse>;

public interface IQuery<out TResponse> : IRequest<TResponse>;
```

We’ll use these in IAM (and later in other services) for commands and queries.

> These are intentionally lightweight; they simply wrap `IRequest<TResponse>` from MediatR.

---

### 7.3.2 Add MediatR to IAM Application and Infrastructure

Install MediatR into:

- `BankingSuite.IamService.Application`
- `BankingSuite.IamService.Infrastructure`

In Visual Studio:

1. Right-click `BankingSuite.IamService.Application` → **Manage NuGet Packages…** → install `MediatR`
2. Right-click `BankingSuite.IamService.Infrastructure` → **Manage NuGet Packages…** → install `MediatR`

We’ll register MediatR from the Infrastructure project so handlers in the Application assembly are automatically discovered.

---

## 7.4 IAM Domain: ApplicationUser & ApplicationRole

IAM is responsible for **users** (employees, admins, and later customers) and **roles**.

We’ll use ASP.NET Core Identity and extend it with our own properties.

### 7.4.1 Install Identity base package in Domain

In `BankingSuite.IamService.Domain`:

1. Right-click the project → **Manage NuGet Packages…**
2. Install:

- `Microsoft.AspNetCore.Identity`

This gives us `IdentityUser`, `IdentityRole`, etc.

### 7.4.2 UserType enum

In `BankingSuite.IamService.Domain`:

1. Right-click → **Add → New Folder…** → name it `Users`
2. Right-click `Users` → **Add → Class…** → name it `UserType.cs`

Replace with:

```csharp
namespace BankingSuite.IamService.Domain.Users;

public enum UserType
{
    Admin = 1,
    Employee = 2,
    Customer = 3
}
```

### 7.4.3 ApplicationUser

In the same `Users` folder, add `ApplicationUser.cs`:

```csharp
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IamService.Domain.Users;

public class ApplicationUser : IdentityUser<Guid>
{
    public string FirstName { get; set; } = string.Empty;

    public string LastName { get; set; } = string.Empty;

    public bool IsActive { get; set; } = true;

    public UserType UserType { get; set; } = UserType.Employee;

    public DateTime CreatedAtUtc { get; set; } = DateTime.UtcNow;

    public DateTime? DeactivatedAtUtc { get; set; }

    public string FullName => $"{FirstName} {LastName}".Trim();
}
```

### 7.4.4 ApplicationRole

Add `ApplicationRole.cs` under `Users`:

```csharp
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IamService.Domain.Users;

public class ApplicationRole : IdentityRole<Guid>
{
    public ApplicationRole() : base()
    {
    }

    public ApplicationRole(string roleName) : base(roleName)
    {
    }
}
```

Now IAM’s **Domain** has the core user and role types used by Identity.

---

## 7.5 IAM Infrastructure: DbContext, JWT options & DI

We’ll configure:

- `IamDbContext` using EF Core + Identity
- `JwtOptions` for token settings
- `JwtTokenGenerator` implementation
- Infrastructure dependency injection, including MediatR registration

### 7.5.1 EF Core + Identity + Npgsql packages

In `BankingSuite.IamService.Infrastructure`, install:

- `Microsoft.EntityFrameworkCore`
- `Microsoft.EntityFrameworkCore.Design`
- `Microsoft.AspNetCore.Identity.EntityFrameworkCore`
- `Npgsql.EntityFrameworkCore.PostgreSQL`

### 7.5.2 IamDbContext

In `BankingSuite.IamService.Infrastructure`:

1. Right-click → **Add → New Folder…** → `Persistence`
2. Right-click `Persistence` → **Add → Class…** → `IamDbContext.cs`

Replace with:

```csharp
using BankingSuite.IamService.Domain.Users;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IamService.Infrastructure.Persistence;

public class IamDbContext : IdentityDbContext<ApplicationUser, ApplicationRole, Guid>
{
    public IamDbContext(DbContextOptions<IamDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        builder.Entity<ApplicationUser>(b =>
        {
            b.ToTable("Users");
        });

        builder.Entity<ApplicationRole>(b =>
        {
            b.ToTable("Roles");
        });

        builder.Entity<IdentityUserRole<Guid>>().ToTable("UserRoles");
        builder.Entity<IdentityUserClaim<Guid>>().ToTable("UserClaims");
        builder.Entity<IdentityUserLogin<Guid>>().ToTable("UserLogins");
        builder.Entity<IdentityRoleClaim<Guid>>().ToTable("RoleClaims");
        builder.Entity<IdentityUserToken<Guid>>().ToTable("UserTokens");
    }
}
```

---

## 7.6 JWT options & token generator (Application + Infrastructure)

To keep the architecture clean:

- The **interface** for generating tokens belongs in the **Application** layer
- The **implementation** lives in the **Infrastructure** layer

### 7.6.1 IJwtTokenGenerator (Application)

In `BankingSuite.IamService.Application`:

1. Right-click project → **Add → New Folder…** → `Auth`
2. Right-click `Auth` → **Add → Class…** → `IJwtTokenGenerator.cs`

Replace with:

```csharp
using BankingSuite.IamService.Domain.Users;

namespace BankingSuite.IamService.Application.Auth;

public interface IJwtTokenGenerator
{
    string GenerateToken(ApplicationUser user, IList<string> roles);
}
```

> This keeps the Application layer depending **only on an abstraction**, not on any specific JWT library.

### 7.6.2 JwtOptions & JwtTokenGenerator (Infrastructure)

In `BankingSuite.IamService.Infrastructure`:

1. Right-click project → **Add → New Folder…** → `Security`

Add `JwtOptions.cs`:

```csharp
namespace BankingSuite.IamService.Infrastructure.Security;

public class JwtOptions
{
    public const string SectionName = "Jwt";

    public string Issuer { get; set; } = string.Empty;

    public string Audience { get; set; } = string.Empty;

    public string Key { get; set; } = string.Empty;

    public int AccessTokenMinutes { get; set; } = 60;
}
```

Now add `JwtTokenGenerator.cs` under the same `Security` folder:

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using BankingSuite.IamService.Application.Auth;
using BankingSuite.IamService.Domain.Users;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;

namespace BankingSuite.IamService.Infrastructure.Security;

public class JwtTokenGenerator : IJwtTokenGenerator
{
    private readonly JwtOptions _options;

    public JwtTokenGenerator(IOptions<JwtOptions> options)
    {
        _options = options.Value;
    }

    public string GenerateToken(ApplicationUser user, IList<string> roles)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email ?? string.Empty),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new("fullName", user.FullName),
            new("userType", user.UserType.ToString())
        };

        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var keyBytes = Encoding.UTF8.GetBytes(_options.Key);
        var key = new SymmetricSecurityKey(keyBytes);
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _options.Issuer,
            audience: _options.Audience,
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: DateTime.UtcNow.AddMinutes(_options.AccessTokenMinutes),
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

> **Important:** `JwtOptions.Key` must be long enough (at least 32+ characters) to avoid HS256 key-size errors.

---

## 7.7 Infrastructure dependency injection (+ MediatR registration)

We’ll wire up:

- EF Core + PostgreSQL
- Identity
- JWT authentication
- MediatR (scanning the Application assembly)
- `IJwtTokenGenerator` implementation

In `BankingSuite.IamService.Infrastructure`, add or update `DependencyInjection.cs`:

```csharp
using System.Reflection;
using System.Text;
using BankingSuite.IamService.Application.Auth;
using BankingSuite.IamService.Domain.Users;
using BankingSuite.IamService.Infrastructure.Persistence;
using BankingSuite.IamService.Infrastructure.Security;
using MediatR;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.IdentityModel.Tokens;

namespace BankingSuite.IamService.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddIamInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Database
        var connectionString = configuration.GetConnectionString("IamDatabase");

        services.AddDbContext<IamDbContext>(options =>
            options.UseNpgsql(connectionString));

        // Identity
        services
            .AddIdentityCore<ApplicationUser>(options =>
            {
                options.User.RequireUniqueEmail = true;

                options.Password.RequiredLength = 8;
                options.Password.RequireDigit = true;
                options.Password.RequireLowercase = true;
                options.Password.RequireUppercase = false;
                options.Password.RequireNonAlphanumeric = false;

                options.Lockout.MaxFailedAccessAttempts = 3;
                options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(10);
                options.Lockout.AllowedForNewUsers = true;
            })
            .AddRoles<ApplicationRole>()
            .AddEntityFrameworkStores<IamDbContext>()
            .AddSignInManager<SignInManager<ApplicationUser>>();

        // JWT options
        services.Configure<JwtOptions>(configuration.GetSection(JwtOptions.SectionName));

        var jwtOptions = configuration.GetSection(JwtOptions.SectionName).Get<JwtOptions>() ?? new JwtOptions();
        var keyBytes = Encoding.UTF8.GetBytes(jwtOptions.Key);

        // Authentication + JWT bearer
        services
            .AddAuthentication(options =>
            {
                options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
                options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
            })
            .AddJwtBearer(options =>
            {
                options.RequireHttpsMetadata = false;
                options.SaveToken = true;
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidIssuer = jwtOptions.Issuer,
                    ValidAudience = jwtOptions.Audience,
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKey = new SymmetricSecurityKey(keyBytes),
                    ValidateLifetime = true,
                    ClockSkew = TimeSpan.FromMinutes(1)
                };
            });

        // Authorization
        services.AddAuthorization();

        // JWT service
        services.AddScoped<IJwtTokenGenerator, JwtTokenGenerator>();

        // MediatR - scan IAM Application assembly
        var applicationAssembly = Assembly.Load("BankingSuite.IamService.Application");

        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(applicationAssembly);
        });

        return services;
    }
}
```

Now IAM infrastructure fully wires Identity, JWT and MediatR.

---

## 7.8 IAM Application layer: commands & handlers (CQRS + Result)

We’ll move business logic into **Application commands/handlers** and return a `Result<T>` from our BuildingBlocks.

We’ll implement two commands:

- `LoginCommand` → `LoginResult`
- `CreateEmployeeCommand` → `CreateEmployeeResult`

### 7.8.1 BuildingBlocks `Result` (already present)

You should already have a `Result` type in `BankingSuite.BuildingBlocks.Domain.Abstractions`. We’ll re-use it.

Example (for reference):

```csharp
namespace BankingSuite.BuildingBlocks.Domain.Abstractions;

public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public string? Error { get; }

    protected Result(bool isSuccess, string? error)
    {
        IsSuccess = isSuccess;
        Error = error;
    }

    public static Result Success() => new(true, null);

    public static Result Failure(string error) =>
        new(false, error);

    public static Result<T> Success<T>(T value) => Result<T>.Success(value);

    public static Result<T> Failure<T>(string error) => Result<T>.Failure(error);
}

public class Result<T> : Result
{
    public T? Value { get; }

    protected Result(bool isSuccess, T? value, string? error)
        : base(isSuccess, error)
    {
        Value = value;
    }

    public static Result<T> Success(T value) =>
        new(true, value, null);

    public new static Result<T> Failure(string error) =>
        new(false, default, error);
}
```

Application layer will return `Result<T>` from handlers instead of throwing exceptions for normal validation errors.

---

### 7.8.2 LoginCommand & handler

In `BankingSuite.IamService.Application`:

1. Right-click → **Add → New Folder…** → `Auth`
2. Under `Auth`, add another folder: `Commands`
3. Under `Commands`, add folder `Login`

So you get:

```text
BankingSuite.IamService.Application/
  Auth/
    IJwtTokenGenerator.cs
    Commands/
      Login/
```

Add `LoginCommand.cs`:

```csharp
using BankingSuite.BuildingBlocks.Application.CQRS;
using BankingSuite.BuildingBlocks.Domain.Abstractions;

namespace BankingSuite.IamService.Application.Auth.Commands.Login;

public sealed record LoginCommand(string Email, string Password)
    : ICommand<Result<LoginResult>>;

public sealed record LoginResult(string AccessToken, DateTime ExpiresAtUtc);
```

Add `LoginCommandHandler.cs` in the same folder:

```csharp
using BankingSuite.BuildingBlocks.Domain.Abstractions;
using BankingSuite.IamService.Application.Auth;
using BankingSuite.IamService.Domain.Users;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IamService.Application.Auth.Commands.Login;

public sealed class LoginCommandHandler
    : IRequestHandler<LoginCommand, Result<LoginResult>>
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly IJwtTokenGenerator _jwtTokenGenerator;

    public LoginCommandHandler(
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        IJwtTokenGenerator jwtTokenGenerator)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _jwtTokenGenerator = jwtTokenGenerator;
    }

    public async Task<Result<LoginResult>> Handle(LoginCommand request, CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(request.Email) || string.IsNullOrWhiteSpace(request.Password))
        {
            return Result.Failure<LoginResult>("Email and password are required.");
        }

        var user = await _userManager.FindByEmailAsync(request.Email);
        if (user is null || !user.IsActive)
        {
            return Result.Failure<LoginResult>("Invalid credentials.");
        }

        var signInResult = await _signInManager.CheckPasswordSignInAsync(
            user,
            request.Password,
            lockoutOnFailure: true);

        if (!signInResult.Succeeded)
        {
            return Result.Failure<LoginResult>("Invalid credentials.");
        }

        var roles = await _userManager.GetRolesAsync(user);
        var token = _jwtTokenGenerator.GenerateToken(user, roles);

        // For now, keep in sync with JwtOptions.AccessTokenMinutes (60 minutes by default)
        var expiresAt = DateTime.UtcNow.AddMinutes(60);

        return Result.Success(new LoginResult(token, expiresAt));
    }
}
```

---

### 7.8.3 CreateEmployeeCommand & handler

Under `BankingSuite.IamService.Application/Auth/Commands`, add folder `CreateEmployee`.

Add `CreateEmployeeCommand.cs`:

```csharp
using BankingSuite.BuildingBlocks.Application.CQRS;
using BankingSuite.BuildingBlocks.Domain.Abstractions;

namespace BankingSuite.IamService.Application.Auth.Commands.CreateEmployee;

public sealed record CreateEmployeeCommand(
    string Email,
    string Password,
    string FirstName,
    string LastName)
    : ICommand<Result<CreateEmployeeResult>>;

public sealed record CreateEmployeeResult(
    Guid Id,
    string Email,
    string FullName);
```

Add `CreateEmployeeCommandHandler.cs`:

```csharp
using BankingSuite.BuildingBlocks.Domain.Abstractions;
using BankingSuite.IamService.Domain.Users;
using MediatR;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IamService.Application.Auth.Commands.CreateEmployee;

public sealed class CreateEmployeeCommandHandler
    : IRequestHandler<CreateEmployeeCommand, Result<CreateEmployeeResult>>
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly RoleManager<ApplicationRole> _roleManager;

    public CreateEmployeeCommandHandler(
        UserManager<ApplicationUser> userManager,
        RoleManager<ApplicationRole> roleManager)
    {
        _userManager = userManager;
        _roleManager = roleManager;
    }

    public async Task<Result<CreateEmployeeResult>> Handle(
        CreateEmployeeCommand request,
        CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(request.Email) ||
            string.IsNullOrWhiteSpace(request.Password) ||
            string.IsNullOrWhiteSpace(request.FirstName))
        {
            return Result.Failure<CreateEmployeeResult>("Email, password and first name are required.");
        }

        var existing = await _userManager.FindByEmailAsync(request.Email);
        if (existing is not null)
        {
            return Result.Failure<CreateEmployeeResult>("A user with this email already exists.");
        }

        var user = new ApplicationUser
        {
            Id = Guid.NewGuid(),
            Email = request.Email,
            UserName = request.Email,
            FirstName = request.FirstName,
            LastName = request.LastName,
            UserType = UserType.Employee,
            EmailConfirmed = true,
            IsActive = true
        };

        var result = await _userManager.CreateAsync(user, request.Password);

        if (!result.Succeeded)
        {
            var errorMessage = string.Join("; ", result.Errors.Select(e => e.Description));
            return Result.Failure<CreateEmployeeResult>(errorMessage);
        }

        const string employeeRole = "Employee";

        if (!await _roleManager.RoleExistsAsync(employeeRole))
        {
            await _roleManager.CreateAsync(new ApplicationRole(employeeRole));
        }

        await _userManager.AddToRoleAsync(user, employeeRole);

        var dto = new CreateEmployeeResult(
            user.Id,
            user.Email ?? string.Empty,
            user.FullName);

        return Result.Success(dto);
    }
}
```

Now the **Application layer** implements IAM core features via CQRS and returns `Result<T>`.

---

## 7.9 API project: Program.cs + endpoints using MediatR

We now update the API:

- `Program.cs` wires FastEndpoints, Swagger, IAM infrastructure, auth
- FastEndpoints **endpoints** become thin adapters that call MediatR commands

### 7.9.1 Program.cs

Open `src/backend/Services/IamService/BankingSuite.IamService.API/Program.cs` and update it to:

```csharp
using BankingSuite.IamService.Infrastructure;
using FastEndpoints;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddFastEndpoints();

// Needed for FastEndpoints + Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerDoc(config =>
{
    config.Title = "Alvor Bank - IAM Service";
    config.Version = "v1";
});

// IAM infrastructure (Identity + EF Core + JWT + MediatR)
builder.Services.AddIamInfrastructure(builder.Configuration);

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.UseFastEndpoints();

if (app.Environment.IsDevelopment())
{
    app.UseOpenApi();
    app.UseSwaggerUi3(settings =>
    {
        settings.Path = "/swagger";
        settings.DocumentPath = "/swagger/v1/swagger.json";
    });
}

app.Run();

// For WebApplicationFactory in tests
public partial class Program;
```

---

### 7.9.2 Login endpoint (FastEndpoints + MediatR)

In `BankingSuite.IamService.API`:

1. Ensure you have `Endpoints/Auth` folders:
   - Right-click project → **Add → New Folder…** → `Endpoints`
   - Right-click `Endpoints` → **Add → New Folder…** → `Auth`

Add `LoginEndpoint.cs`:

```csharp
using BankingSuite.IamService.Application.Auth.Commands.Login;
using FastEndpoints;
using MediatR;

namespace BankingSuite.IamService.API.Endpoints.Auth;

public class LoginRequest
{
    public string Email { get; set; } = string.Empty;

    public string Password { get; set; } = string.Empty;
}

public class LoginResponse
{
    public string AccessToken { get; set; } = string.Empty;

    public DateTime ExpiresAtUtc { get; set; }
}

public class LoginEndpoint : Endpoint<LoginRequest, LoginResponse>
{
    private readonly IMediator _mediator;

    public LoginEndpoint(IMediator mediator)
    {
        _mediator = mediator;
    }

    public override void Configure()
    {
        Post("/auth/login");
        AllowAnonymous();

        Summary(s =>
        {
            s.Summary = "Login and receive a JWT access token.";
            s.Description = "Validates credentials and issues a JWT used by other microservices.";
        });
    }

    public override async Task HandleAsync(LoginRequest req, CancellationToken ct)
    {
        var result = await _mediator.Send(
            new LoginCommand(req.Email, req.Password),
            ct);

        if (result.IsFailure)
        {
            AddError(result.Error ?? "Login failed.");
            await SendErrorsAsync(cancellation: ct);
            return;
        }

        var value = result.Value!;

        await Send.OkAsync(new LoginResponse
        {
            AccessToken = value.AccessToken,
            ExpiresAtUtc = value.ExpiresAtUtc
        }, cancellation: ct);
    }
}
```

---

### 7.9.3 Create employee endpoint

Add `CreateEmployeeEndpoint.cs` under `Endpoints/Auth`:

```csharp
using BankingSuite.IamService.Application.Auth.Commands.CreateEmployee;
using FastEndpoints;
using MediatR;

namespace BankingSuite.IamService.API.Endpoints.Auth;

public class CreateEmployeeRequest
{
    public string Email { get; set; } = string.Empty;

    public string Password { get; set; } = string.Empty;

    public string FirstName { get; set; } = string.Empty;

    public string LastName { get; set; } = string.Empty;
}

public class CreateEmployeeResponse
{
    public Guid Id { get; set; }

    public string Email { get; set; } = string.Empty;

    public string FullName { get; set; } = string.Empty;
}

public class CreateEmployeeEndpoint : Endpoint<CreateEmployeeRequest, CreateEmployeeResponse>
{
    private readonly IMediator _mediator;

    public CreateEmployeeEndpoint(IMediator mediator)
    {
        _mediator = mediator;
    }

    public override void Configure()
    {
        Post("/auth/employees");

        // In Chapter 08 we will restrict this to Admins:
        // Roles("Admin");

        Summary(s =>
        {
            s.Summary = "Create a new employee user.";
            s.Description = "Creates a bank employee login. In production this would be restricted to Admins.";
        });
    }

    public override async Task HandleAsync(CreateEmployeeRequest req, CancellationToken ct)
    {
        var command = new CreateEmployeeCommand(
            req.Email,
            req.Password,
            req.FirstName,
            req.LastName);

        var result = await _mediator.Send(command, ct);

        if (result.IsFailure)
        {
            AddError(result.Error ?? "Could not create employee.");
            await SendErrorsAsync(cancellation: ct);
            return;
        }

        var value = result.Value!;

        await Send.OkAsync(new CreateEmployeeResponse
        {
            Id = value.Id,
            Email = value.Email,
            FullName = value.FullName
        }, cancellation: ct);
    }
}
```

---

### 7.9.4 Health endpoint (FastEndpoints-style)

Ensure your health endpoint still exists and uses FastEndpoints (from earlier chapters). For example:

```csharp
using FastEndpoints;

namespace BankingSuite.IamService.API.Endpoints.Health;

public class HealthCheckEndpoint : EndpointWithoutRequest
{
    public override void Configure()
    {
        Get("/health");
        AllowAnonymous();

        Summary(s =>
        {
            s.Summary = "Simple health check for the IAM service.";
            s.Description = "Returns a basic payload that tells if the IAM service is running.";
        });
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        await Send.OkAsync(new
        {
            Service = "IamService",
            Status = "Healthy",
            TimestampUtc = DateTime.UtcNow
        }, cancellation: ct);
    }
}
```

---

## 7.10 Configuration: appsettings.Development.json

We need:

- Connection string to the IAM database (`iam-db` container)
- JWT settings

Open `src/backend/Services/IamService/BankingSuite.IamService.API/appsettings.Development.json` and ensure it contains:

```json
{
  "ConnectionStrings": {
    "IamDatabase": "Host=localhost;Port=5433;Database=iam_service_db;Username=iam_user;Password=iam_password"
  },
  "Jwt": {
    "Issuer": "AlvorBank.IamService",
    "Audience": "AlvorBank.Backend",
    "Key": "THIS-IS-A-VERY-LONG-DEV-ONLY-SECRET-KEY-CHANGE-ME-1234567890",
    "AccessTokenMinutes": 60
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

Notes:

- IAM API runs on your host, so it connects to `localhost:5433`, which is the port we exposed for `iam-db` in `docker-compose.dev.yml`.
- The `Key` is intentionally long to avoid HS256 key-size errors.
- Inside Docker later, we’ll switch host to `iam-db` (container name).

---

## 7.11 EF Core migrations for IAM

Now we’ll create and apply the initial migration so Identity tables are created.

> Make sure `iam-db` is running:
>
> ```bash
> docker compose -f infra/docker-compose.dev.yml up -d iam-db
> ```

### 7.11.1 Install `dotnet-ef` (if you haven’t already)

```bash
dotnet tool install --global dotnet-ef
```

Verify:

```bash
dotnet ef --help
```

### 7.11.2 Add the migration

From the IAM API project folder:

```bash
cd src/backend/Services/IamService/BankingSuite.IamService.API
```

Run:

```bash
dotnet ef migrations add InitialIdentitySchema ^
  --project ../BankingSuite.IamService.Infrastructure ^
  --startup-project . ^
  --output-dir Persistence/Migrations
```

(on Linux/macOS, replace `^` with `\`)

This will create a migration under:

```text
BankingSuite.IamService.Infrastructure/
  Persistence/
    Migrations/
      20xxxxxxxxxxxx_InitialIdentitySchema.cs
```

### 7.11.3 Update the database

Still from the API project folder:

```bash
dotnet ef database update ^
  --project ../BankingSuite.IamService.Infrastructure ^
  --startup-project .
```

This applies the migration to `iam_service_db` on `localhost:5433`.

Use your DB client to confirm tables like `Users`, `Roles`, `UserRoles`, etc. exist.

---

## 7.12 Manual sanity check

1. Ensure `iam-db` is running:

   ```bash
   docker compose -f infra/docker-compose.dev.yml up -d iam-db
   ```

2. In Visual Studio 2026, set **`BankingSuite.IamService.API`** as the Startup Project.
3. Press **F5** or **Ctrl+F5**.

Check:

- Swagger UI:

  ```text
  http://localhost:5101/swagger
  ```

  You should see endpoints:

  - `GET /health`
  - `POST /auth/login`
  - `POST /auth/employees`

### 7.12.1 Create your first employee

In Swagger:

1. Open `POST /auth/employees`
2. Click **Try it out**
3. Use:

   ```json
   {
     "email": "admin@alvorbank.test",
     "password": "Admin1234!",
     "firstName": "System",
     "lastName": "Admin"
   }
   ```

4. Execute.

You should get `200 OK` with:

- `id` (GUID)
- `email`
- `fullName`

Check the database `iam_service_db` to confirm:

- New row in `Users`
- `Employee` role exists in `Roles`
- Mapping in `UserRoles`

### 7.12.2 Login and get a JWT

In Swagger:

1. Open `POST /auth/login`
2. **Try it out**
3. Use:

   ```json
   {
     "email": "admin@alvorbank.test",
     "password": "Admin1234!"
   }
   ```

4. Execute.

You should see:

```json
{
  "accessToken": "<long-jwt-token>",
  "expiresAtUtc": "2026-01-01T12:34:56.000Z"
}
```

Copy the `accessToken` and inspect it on a JWT tool (e.g. jwt.io) to see:

- Subject and email
- `fullName`
- `userType`
- Role claim (`Employee`)

---

## 7.13 Tests & CI still green

Run the tests from the repo root:

```bash
dotnet test src/backend/BankingSuite.Backend.sln
```

Then run with coverage if you like:

```bash
dotnet test src/backend/BankingSuite.Backend.sln ^
  /p:CollectCoverage=true ^
  /p:CoverletOutput=../../lcov ^
  /p:CoverletOutputFormat=lcov

reportgenerator ^
  -reports:lcov.info ^
  -targetdir:src/backend/coverage-report ^
  "-reporttypes:Html;TextSummary"
```

Open `src/backend/coverage-report/index.html` to see updated coverage. Our IAM logic now lives in **Application handlers**, which is easier to test thoroughly in Chapter 08.

GitHub Actions (`backend-tests.yml`) will continue to run tests and publish the coverage report artifact.

---

## 7.14 Commit & tag Chapter 07

From the repo root:

```bash
git status
git add src/backend/Services/IamService src/backend/BankingSuite.Backend.sln
git commit -m "ch07: add IAM core with Identity, EF Core, JWT and CQRS"
```

If you’re using a feature branch:

```bash
git push -u origin feature/ch07-iam-core
```

After merging into `develop`, tag the chapter:

```bash
git checkout develop
git pull

git tag -a chapter-07-iam-core -m "Chapter 07: IAM core with Identity, EF Core, JWT and CQRS"
git push origin chapter-07-iam-core
```

---

## 7.15 What’s next

You now have a **clean IAM core**:

- Identity + EF Core + PostgreSQL
- JWT authentication
- CQRS + MediatR for core flows
- Minimal endpoints for login & employee creation

In **Chapter 08 — Advanced IAM with CQRS, Security Flows & 2FA** we will:

- Add **admin-level user management** endpoints:
  - List users
  - Get user details
  - Update user
  - Activate/deactivate users
- Implement **email confirmation** and **resend confirmation**
- Implement **forgot password / reset password / change password**
- Add a pragmatic **2FA** flow (starting with email-based one-time codes)
- Extend tests to raise IAM coverage towards **80%+**, as a real banking team would do

Only after that will we move to the **Angular 21 + Nx frontend** and start wiring a real login screen and IAM admin UI on top of this backend.
