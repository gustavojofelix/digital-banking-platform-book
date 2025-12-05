# Chapter 07 — Final Backend Structure, Building Blocks & IAM Service Implementation

In Part 1, we experimented with several ideas: clean architecture, CQRS, Docker, testing, CI, and a first microservice.

In this chapter we **freeze the backend architecture** for the entire book and for the real Alvor Bank system:

- A **repeatable microservice structure** (Domain, Application, Infrastructure, API)
- Shared **BuildingBlocks** for DDD, CQRS, EF Core and web setup
- A complete **IAM Service** implemented end-to-end on top of that template
- Everything running in **Docker**, with **tests and ≥80% coverage** and a **PR-based workflow**

From now on, every backend chapter follows the same rhythm:

1. Create a **feature branch** for the chapter
2. Implement the service (**Domain → Application → Infrastructure → API**)
3. **Dockerize** it and run via `docker compose`
4. Add/extend **unit + integration tests** and ensure **≥80% coverage**
5. Open a **PR to `develop`** and let **CI** validate everything

Later chapters will repeat this pattern for `CustomerService`, `AccountService`, and more.

---

## 7.1 Working Like a Real Banking Team

We’re treating this book like a real project. That means:

- No coding directly on `develop` or `main`
- All services must run via **Docker**
- We don’t merge changes if tests are red or if coverage drops below **80%**

### 7.1.1 Create the chapter branch

From the root of the repo:

```bash
cd digital-banking-suite

git checkout develop
git pull origin develop

git checkout -b feature/chapter-07-iam-backend
```

All changes in this chapter are done on this branch.

---

## 7.2 Final Backend Solution Structure

Our backend lives in `src/backend`, with a single solution that knows about all microservices.

```text
digital-banking-suite/
└─ src/
   └─ backend/
      ├─ BankingSuite.Backend.sln
      ├─ BuildingBlocks/
      │  ├─ BankingSuite.BuildingBlocks.Domain/
      │  ├─ BankingSuite.BuildingBlocks.Application/
      │  ├─ BankingSuite.BuildingBlocks.Infrastructure/
      │  └─ BankingSuite.BuildingBlocks.Web/
      └─ Services/
         ├─ IamService/
         │  ├─ BankingSuite.IamService.Domain/
         │  ├─ BankingSuite.IamService.Application/
         │  ├─ BankingSuite.IamService.Infrastructure/
         │  └─ BankingSuite.IamService.API/
         ├─ CustomerService/
         │  ├─ BankingSuite.CustomerService.Domain/
         │  ├─ BankingSuite.CustomerService.Application/
         │  ├─ BankingSuite.CustomerService.Infrastructure/
         │  └─ BankingSuite.CustomerService.API/
         ├─ AccountService/
         │  ├─ BankingSuite.AccountService.Domain/
         │  ├─ BankingSuite.AccountService.Application/
         │  ├─ BankingSuite.AccountService.Infrastructure/
         │  └─ BankingSuite.AccountService.API/
         └─ ApiGateway/
            └─ BankingSuite.ApiGateway/
```

For **every microservice**, we use the same 4–project pattern:

- `*.Domain` — DDD concepts: entities, value objects, domain services, invariants.
- `*.Application` — CQRS: commands, queries, handlers, DTOs, interfaces.
- `*.Infrastructure` — EF Core, Identity, messaging, external resource integrations.
- `*.API` — FastEndpoints + HTTP pipeline.

**Dependencies always flow inward:**

- `API → Infrastructure → Application → Domain`
- Each project can also depend on the shared **BuildingBlocks** projects.

We’ll now build the **BuildingBlocks** once, and then use them to implement the IAM service.

---

## 7.3 BuildingBlocks: Shared Kernel and Cross–Cutting Concerns

Our goal is to avoid copy–pasting common patterns into every service. Instead, we centralize them in a small set of shared projects.

### 7.3.0 Cleaning up the old BuildingBlocks project

In Part 1 you may have created a **single** project like `BankingSuite.BuildingBlocks` as a sandbox.

We’re now replacing that with four focused projects:

- `BankingSuite.BuildingBlocks.Domain`
- `BankingSuite.BuildingBlocks.Application`
- `BankingSuite.BuildingBlocks.Infrastructure`
- `BankingSuite.BuildingBlocks.Web`

If you still have the old single project, remove it from the solution (and optionally delete the folder):

```bash
cd src/backend

# Adjust the path/name if your previous project is different
dotnet sln BankingSuite.Backend.sln remove \
  BuildingBlocks/BankingSuite.BuildingBlocks/BankingSuite.BuildingBlocks.csproj

# Optional: delete the old project folder if you no longer need it
rm -rf BuildingBlocks/BankingSuite.BuildingBlocks
```

From here on, we only use the **four** new BuildingBlocks projects.

### 7.3.1 BankingSuite.BuildingBlocks.Domain

Create a class library:

```bash
cd src/backend/BuildingBlocks
dotnet new classlib -n BankingSuite.BuildingBlocks.Domain
```

Add base DDD primitives:

```csharp
// Entities/Entity.cs
namespace BankingSuite.BuildingBlocks.Domain.Entities;

public abstract class Entity<TId>
{
    public TId Id { get; protected set; } = default!;
}
```

```csharp
// Entities/AggregateRoot.cs
using BankingSuite.BuildingBlocks.Domain.Events;

namespace BankingSuite.BuildingBlocks.Domain.Entities;

public abstract class AggregateRoot<TId> : Entity<TId>
{
    private readonly List<IDomainEvent> _domainEvents = new();

    public IReadOnlyCollection<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    protected void AddDomainEvent(IDomainEvent @event) => _domainEvents.Add(@event);

    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

```csharp
// Events/IDomainEvent.cs
namespace BankingSuite.BuildingBlocks.Domain.Events;

public interface IDomainEvent
{
    DateTime OccurredOn { get; }
}
```

We’ll extend this later with base `ValueObject` and domain exceptions when the need arises.

### 7.3.2 BankingSuite.BuildingBlocks.Application

Create the project:

```bash
dotnet new classlib -n BankingSuite.BuildingBlocks.Application
```

Add references:

```bash
dotnet add BankingSuite.BuildingBlocks.Application reference \
  BankingSuite.BuildingBlocks.Domain
```

Add basic abstractions:

```csharp
// Abstractions/IUnitOfWork.cs
namespace BankingSuite.BuildingBlocks.Application.Abstractions;

public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

And a simple `Result<T>` type:

```csharp
// Results/Result.cs
namespace BankingSuite.BuildingBlocks.Application.Results;

public sealed record Result<T>(T? Value, bool IsSuccess, string? Error)
{
    public static Result<T> Success(T value) => new(value, true, null);
    public static Result<T> Failure(string error) => new(default, false, error);
}
```

Later we’ll add:

- Validation and logging pipeline behaviors for MediatR.
- Error codes & exceptions to keep API responses consistent.

### 7.3.3 BankingSuite.BuildingBlocks.Infrastructure

Create:

```bash
dotnet new classlib -n BankingSuite.BuildingBlocks.Infrastructure
dotnet add BankingSuite.BuildingBlocks.Infrastructure reference \
  BankingSuite.BuildingBlocks.Application \
  BankingSuite.BuildingBlocks.Domain
```

For now we keep it minimal:

```csharp
// Persistence/RepositoryBase.cs
using BankingSuite.BuildingBlocks.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.BuildingBlocks.Infrastructure.Persistence;

// Generic EF repository for most services (Customer, Account, etc.)
public class RepositoryBase<TAggregate> where TAggregate : AggregateRoot<Guid>
{
    protected readonly DbContext DbContext;
    protected readonly DbSet<TAggregate> DbSet;

    protected RepositoryBase(DbContext dbContext)
    {
        DbContext = dbContext;
        DbSet = dbContext.Set<TAggregate>();
    }

    public Task<TAggregate?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
        => DbSet.FirstOrDefaultAsync(x => x.Id.Equals(id), cancellationToken);

    public Task AddAsync(TAggregate aggregate, CancellationToken cancellationToken = default)
        => DbSet.AddAsync(aggregate, cancellationToken).AsTask();

    public void Remove(TAggregate aggregate) => DbSet.Remove(aggregate);
}
```

IAM will mostly use ASP.NET Identity APIs for users, but other services will reuse this base.

### 7.3.4 BankingSuite.BuildingBlocks.Web

Create:

```bash
dotnet new classlib -n BankingSuite.BuildingBlocks.Web
dotnet add BankingSuite.BuildingBlocks.Web package FastEndpoints
dotnet add BankingSuite.BuildingBlocks.Web package FastEndpoints.Swagger
```

Add helper extensions:

```csharp
// Extensions/WebApplicationBuilderExtensions.cs
using FastEndpoints;
using FastEndpoints.Swagger;
using Microsoft.Extensions.DependencyInjection;

namespace BankingSuite.BuildingBlocks.Web.Extensions;

public static class WebApplicationBuilderExtensions
{
    public static IServiceCollection AddBankingFastEndpoints(this IServiceCollection services)
    {
        services.AddFastEndpoints();
        services.AddSwaggerDoc(); // FastEndpoints Swagger
        return services;
    }
}
```

```csharp
// Extensions/WebApplicationExtensions.cs
using FastEndpoints;
using Microsoft.AspNetCore.Builder;

namespace BankingSuite.BuildingBlocks.Web.Extensions;

public static class WebApplicationExtensions
{
    public static WebApplication UseBankingFastEndpoints(this WebApplication app)
    {
        app.UseFastEndpoints(c =>
        {
            c.Endpoints.RoutePrefix = "api";
        });

        app.UseOpenApi();
        app.UseSwaggerUi3(s => s.ConfigureDefaults());

        return app;
    }
}
```

We’ll later add a global exception handler and ProblemDetails mapping here, and all APIs will get them for free.

### 7.3.5 Add BuildingBlocks projects to the solution

Finally, register all four BuildingBlocks projects in the main backend solution:

```bash
cd ../   # back to src/backend

dotnet sln BankingSuite.Backend.sln add \
  BuildingBlocks/BankingSuite.BuildingBlocks.Domain/BankingSuite.BuildingBlocks.Domain.csproj \
  BuildingBlocks/BankingSuite.BuildingBlocks.Application/BankingSuite.BuildingBlocks.Application.csproj \
  BuildingBlocks/BankingSuite.BuildingBlocks.Infrastructure/BankingSuite.BuildingBlocks.Infrastructure.csproj \
  BuildingBlocks/BankingSuite.BuildingBlocks.Web/BankingSuite.BuildingBlocks.Web.csproj
```

Now the solution knows about the new BuildingBlocks projects and we can reference them from the IAM service.

---

## 7.4 Scaffolding the IAM Service Projects

We now create IAM with the 4–project pattern.

From `src/backend/Services`:

```bash
mkdir -p IamService
cd IamService

dotnet new classlib -n BankingSuite.IamService.Domain
dotnet new classlib -n BankingSuite.IamService.Application
dotnet new classlib -n BankingSuite.IamService.Infrastructure
dotnet new web -n BankingSuite.IamService.API
```

### 7.4.1 Project References

Inside `IamService`:

```bash
# Domain uses BuildingBlocks.Domain
dotnet add BankingSuite.IamService.Domain reference \
  ../../BuildingBlocks/BankingSuite.BuildingBlocks.Domain

# Application depends on Domain + BuildingBlocks.Application
dotnet add BankingSuite.IamService.Application reference \
  BankingSuite.IamService.Domain \
  ../../BuildingBlocks/BankingSuite.BuildingBlocks.Application

# Infrastructure depends on Domain + Application + BuildingBlocks.Infrastructure
dotnet add BankingSuite.IamService.Infrastructure reference \
  BankingSuite.IamService.Domain \
  BankingSuite.IamService.Application \
  ../../BuildingBlocks/BankingSuite.BuildingBlocks.Infrastructure

# API depends on Application + Infrastructure + BuildingBlocks.Web
dotnet add BankingSuite.IamService.API reference \
  BankingSuite.IamService.Application \
  BankingSuite.IamService.Infrastructure \
  ../../BuildingBlocks/BankingSuite.BuildingBlocks.Web
```

Add Identity, EF Core & JWT packages to Infrastructure and API as needed (we omit explicit versions so central package management or NuGet picks appropriate ones):

```bash
dotnet add BankingSuite.IamService.Infrastructure package Microsoft.EntityFrameworkCore
dotnet add BankingSuite.IamService.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add BankingSuite.IamService.Infrastructure package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add BankingSuite.IamService.Infrastructure package Microsoft.Extensions.Options.ConfigurationExtensions

dotnet add BankingSuite.IamService.API package FastEndpoints
dotnet add BankingSuite.IamService.API package FastEndpoints.Swagger
dotnet add BankingSuite.IamService.API package FastEndpoints.Security
dotnet add BankingSuite.IamService.API package Microsoft.AspNetCore.Authentication.JwtBearer
```

### 7.4.2 Add IAM projects to the solution

Back in `src/backend`, add all four IAM projects to the solution:

```bash
cd ../../  # from Services/IamService back to src/backend

dotnet sln BankingSuite.Backend.sln add \
  Services/IamService/BankingSuite.IamService.Domain/BankingSuite.IamService.Domain.csproj \
  Services/IamService/BankingSuite.IamService.Application/BankingSuite.IamService.Application.csproj \
  Services/IamService/BankingSuite.IamService.Infrastructure/BankingSuite.IamService.Infrastructure.csproj \
  Services/IamService/BankingSuite.IamService.API/BankingSuite.IamService.API.csproj
```

Now `BankingSuite.Backend.sln` includes both the **BuildingBlocks** and **IAM** projects, so your IDE and CI pipeline can build everything together.

---

## 7.5 IAM Domain: User Types and Roles

IAM’s Domain layer models IAM-specific concepts that are independent of any particular persistence or web framework.

### 7.5.1 UserType

`BankingSuite.IamService.Domain/Users/UserType.cs`:

```csharp
namespace BankingSuite.IamService.Domain.Users;

public enum UserType
{
    Employee = 1,
    Customer = 2
}
```

Alvor Bank has two user categories:

- **Employee** — internal staff (SystemAdmin, BackOfficeAgent, KycOfficer, etc.)
- **Customer** — external users accessing online banking

### 7.5.2 RoleNames

`BankingSuite.IamService.Domain/Users/RoleNames.cs`:

```csharp
namespace BankingSuite.IamService.Domain.Users;

public static class RoleNames
{
    public const string SystemAdmin     = "SystemAdmin";
    public const string BackOfficeAgent = "BackOfficeAgent";
    public const string KycOfficer      = "KycOfficer";
    public const string Teller          = "Teller";
    public const string SupportAgent    = "SupportAgent";
    public const string Customer        = "Customer";

    public static readonly IReadOnlyList<string> All = new[]
    {
        SystemAdmin,
        BackOfficeAgent,
        KycOfficer,
        Teller,
        SupportAgent,
        Customer
    };
}
```

**Business rules (we’ll enforce in Application/Infrastructure):**

- Employees have `UserType.Employee` and at least one non–Customer role.
- Customers have `UserType.Customer` and the `Customer` role only.

---

## 7.6 IAM Application: CQRS and Interfaces

IAM’s Application layer defines use cases and interfaces without knowing about EF Core or Identity.

### 7.6.1 DTOs

`BankingSuite.IamService.Application/Auth/Dto/LoginResultDto.cs`:

```csharp
namespace BankingSuite.IamService.Application.Auth.Dto;

public sealed record LoginResultDto(
    string AccessToken,
    DateTime ExpiresAt
);
```

`BankingSuite.IamService.Application/Users/Dto/UserDto.cs`:

```csharp
using BankingSuite.IamService.Domain.Users;

namespace BankingSuite.IamService.Application.Users.Dto;

public sealed record UserDto(
    Guid Id,
    string Email,
    string DisplayName,
    UserType UserType,
    bool IsActive,
    IReadOnlyList<string> Roles
);
```

### 7.6.2 IIdentityService Interface

`BankingSuite.IamService.Application/Common/Interfaces/IIdentityService.cs`:

```csharp
using BankingSuite.IamService.Application.Auth.Dto;
using BankingSuite.IamService.Application.Users.Dto;
using BankingSuite.IamService.Domain.Users;

namespace BankingSuite.IamService.Application.Common.Interfaces;

public interface IIdentityService
{
    Task<LoginResultDto?> LoginAsync(
        string email,
        string password,
        CancellationToken cancellationToken = default);

    Task<UserDto> CreateUserAsync(
        string email,
        string displayName,
        string password,
        UserType userType,
        IReadOnlyList<string> roles,
        CancellationToken cancellationToken = default);
}
```

The Application layer only cares about these **high–level operations**. It doesn’t know anything about `UserManager`, `JwtSecurityToken`, etc.

### 7.6.3 CQRS: Commands and Handlers (MediatR)

Add MediatR:

```bash
dotnet add Services/IamService/BankingSuite.IamService.Application package MediatR
```

#### LoginCommand

`BankingSuite.IamService.Application/Auth/LoginCommand.cs`:

```csharp
using BankingSuite.IamService.Application.Auth.Dto;
using MediatR;

namespace BankingSuite.IamService.Application.Auth;

public sealed record LoginCommand(
    string Email,
    string Password
) : IRequest<LoginResultDto?>;
```

`BankingSuite.IamService.Application/Auth/LoginCommandHandler.cs`:

```csharp
using BankingSuite.IamService.Application.Auth.Dto;
using BankingSuite.IamService.Application.Common.Interfaces;
using MediatR;

namespace BankingSuite.IamService.Application.Auth;

public sealed class LoginCommandHandler : IRequestHandler<LoginCommand, LoginResultDto?>
{
    private readonly IIdentityService _identityService;

    public LoginCommandHandler(IIdentityService identityService)
    {
        _identityService = identityService;
    }

    public Task<LoginResultDto?> Handle(LoginCommand request, CancellationToken cancellationToken)
        => _identityService.LoginAsync(request.Email, request.Password, cancellationToken);
}
```

#### CreateUserCommand

`BankingSuite.IamService.Application/Users/CreateUserCommand.cs`:

```csharp
using BankingSuite.IamService.Application.Users.Dto;
using BankingSuite.IamService.Domain.Users;
using MediatR;

namespace BankingSuite.IamService.Application.Users;

public sealed record CreateUserCommand(
    string Email,
    string DisplayName,
    string Password,
    UserType UserType,
    IReadOnlyList<string> Roles
) : IRequest<UserDto>;
```

`BankingSuite.IamService.Application/Users/CreateUserCommandHandler.cs`:

```csharp
using BankingSuite.IamService.Application.Common.Interfaces;
using BankingSuite.IamService.Application.Users.Dto;
using MediatR;

namespace BankingSuite.IamService.Application.Users;

public sealed class CreateUserCommandHandler
    : IRequestHandler<CreateUserCommand, UserDto>
{
    private readonly IIdentityService _identityService;

    public CreateUserCommandHandler(IIdentityService identityService)
    {
        _identityService = identityService;
    }

    public Task<UserDto> Handle(CreateUserCommand request, CancellationToken cancellationToken)
        => _identityService.CreateUserAsync(
            request.Email,
            request.DisplayName,
            request.Password,
            request.UserType,
            request.Roles,
            cancellationToken);
}
```

### 7.6.4 Application Dependency Injection

`BankingSuite.IamService.Application/DependencyInjection.cs`:

```csharp
using System.Reflection;
using MediatR;
using Microsoft.Extensions.DependencyInjection;

namespace BankingSuite.IamService.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly()));

        // Later: add pipeline behaviors (validation, logging, transactions)
        return services;
    }
}
```

---

## 7.7 IAM Infrastructure: Identity, EF Core and JWT

Infrastructure ties the abstract `IIdentityService` to a real implementation built on Identity + EF Core + JWT.

### 7.7.1 ApplicationUser and IamDbContext

`BankingSuite.IamService.Infrastructure/Identity/ApplicationUser.cs`:

```csharp
using BankingSuite.IamService.Domain.Users;
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IamService.Infrastructure.Identity;

public class ApplicationUser : IdentityUser<Guid>
{
    public string DisplayName { get; set; } = default!;
    public UserType UserType { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}
```

`BankingSuite.IamService.Infrastructure/Persistence/IamDbContext.cs`:

```csharp
using BankingSuite.IamService.Infrastructure.Identity;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IamService.Infrastructure.Persistence;

public class IamDbContext
    : IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>
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
            b.Property(u => u.DisplayName)
                .HasMaxLength(200)
                .IsRequired();

            b.Property(u => u.UserType)
                .IsRequired();

            b.Property(u => u.IsActive)
                .HasDefaultValue(true);

            b.Property(u => u.CreatedAt)
                .IsRequired();

            b.Property(u => u.UpdatedAt);
        });

        // Optional: rename tables
        builder.Entity<ApplicationUser>().ToTable("Users");
        builder.Entity<IdentityRole<Guid>>().ToTable("Roles");
        builder.Entity<IdentityUserRole<Guid>>().ToTable("UserRoles");
        builder.Entity<IdentityUserClaim<Guid>>().ToTable("UserClaims");
        builder.Entity<IdentityUserLogin<Guid>>().ToTable("UserLogins");
        builder.Entity<IdentityUserToken<Guid>>().ToTable("UserTokens");
        builder.Entity<IdentityRoleClaim<Guid>>().ToTable("RoleClaims");
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        var now = DateTime.UtcNow;

        foreach (var entry in ChangeTracker.Entries<ApplicationUser>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = now;
            else if (entry.State == EntityState.Modified)
                entry.Entity.UpdatedAt = now;
        }

        return base.SaveChangesAsync(cancellationToken);
    }
}
```

### 7.7.2 JwtOptions

`BankingSuite.IamService.Infrastructure/Auth/JwtOptions.cs`:

```csharp
namespace BankingSuite.IamService.Infrastructure.Auth;

public class JwtOptions
{
    public string Issuer { get; set; } = default!;
    public string Audience { get; set; } = default!;
    public string Key { get; set; } = default!;
    public int AccessTokenLifetimeMinutes { get; set; } = 60;
}
```

In `BankingSuite.IamService.API/appsettings.json`:

```json
{
  "ConnectionStrings": {
    "IamDatabase": "Host=iam-db;Port=5432;Database=iam_service_db;Username=iam_user;Password=Password123!"
  },
  "Jwt": {
    "Issuer": "AlvorBank.IamService",
    "Audience": "AlvorBank.ApiClients",
    "Key": "SUPER_SECRET_DEMO_KEY_CHANGE_ME",
    "AccessTokenLifetimeMinutes": 60
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### 7.7.3 IdentityService Implementation

`BankingSuite.IamService.Infrastructure/Identity/IdentityService.cs`:

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using BankingSuite.IamService.Application.Auth.Dto;
using BankingSuite.IamService.Application.Common.Interfaces;
using BankingSuite.IamService.Application.Users.Dto;
using BankingSuite.IamService.Domain.Users;
using BankingSuite.IamService.Infrastructure.Auth;
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;

namespace BankingSuite.IamService.Infrastructure.Identity;

public class IdentityService : IIdentityService
{
    private readonly UserManager<ApplicationUser> _userManager;
    private readonly SignInManager<ApplicationUser> _signInManager;
    private readonly RoleManager<IdentityRole<Guid>> _roleManager;
    private readonly JwtOptions _jwtOptions;

    public IdentityService(
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager,
        RoleManager<IdentityRole<Guid>> roleManager,
        IOptions<JwtOptions> jwtOptions)
    {
        _userManager = userManager;
        _signInManager = signInManager;
        _roleManager = roleManager;
        _jwtOptions = jwtOptions.Value;
    }

    public async Task<LoginResultDto?> LoginAsync(
        string email,
        string password,
        CancellationToken cancellationToken = default)
    {
        var user = await _userManager.FindByEmailAsync(email);
        if (user is null || !user.IsActive)
            return null;

        var result = await _signInManager.CheckPasswordSignInAsync(
            user,
            password,
            lockoutOnFailure: true);

        if (!result.Succeeded)
            return null;

        var roles = await _userManager.GetRolesAsync(user);

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new("display_name", user.DisplayName),
            new("user_type", user.UserType.ToString())
        };

        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        var now = DateTime.UtcNow;
        var expires = now.AddMinutes(_jwtOptions.AccessTokenLifetimeMinutes);

        var signingKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtOptions.Key));

        var token = new JwtSecurityToken(
            issuer: _jwtOptions.Issuer,
            audience: _jwtOptions.Audience,
            claims: claims,
            notBefore: now,
            expires: expires,
            signingCredentials: new SigningCredentials(signingKey, SecurityAlgorithms.HmacSha256));

        var accessToken = new JwtSecurityTokenHandler().WriteToken(token);

        return new LoginResultDto(accessToken, expires);
    }

    public async Task<UserDto> CreateUserAsync(
        string email,
        string displayName,
        string password,
        UserType userType,
        IReadOnlyList<string> roles,
        CancellationToken cancellationToken = default)
    {
        foreach (var role in roles)
        {
            if (!await _roleManager.RoleExistsAsync(role))
                throw new InvalidOperationException($"Role '{role}' does not exist.");
        }

        var user = new ApplicationUser
        {
            Id = Guid.NewGuid(),
            Email = email,
            UserName = email,
            DisplayName = displayName,
            UserType = userType,
            IsActive = true
        };

        var createResult = await _userManager.CreateAsync(user, password);
        if (!createResult.Succeeded)
        {
            var errors = string.Join(", ", createResult.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to create user: {errors}");
        }

        var addRolesResult = await _userManager.AddToRolesAsync(user, roles);
        if (!addRolesResult.Succeeded)
        {
            var errors = string.Join(", ", addRolesResult.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to assign roles: {errors}");
        }

        return new UserDto(
            user.Id,
            user.Email!,
            user.DisplayName,
            user.UserType,
            user.IsActive,
            roles.ToList());
    }
}
```

### 7.7.4 Infrastructure Dependency Injection and Seeding

`BankingSuite.IamService.Infrastructure/DependencyInjection.cs`:

```csharp
using BankingSuite.BuildingBlocks.Application.Abstractions;
using BankingSuite.IamService.Application.Common.Interfaces;
using BankingSuite.IamService.Domain.Users;
using BankingSuite.IamService.Infrastructure.Auth;
using BankingSuite.IamService.Infrastructure.Identity;
using BankingSuite.IamService.Infrastructure.Persistence;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace BankingSuite.IamService.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddDbContext<IamDbContext>(options =>
        {
            options.UseNpgsql(configuration.GetConnectionString("IamDatabase"));
        });

        services
            .AddIdentity<ApplicationUser, IdentityRole<Guid>>(options =>
            {
                options.User.RequireUniqueEmail = true;
                options.Password.RequireDigit = true;
                options.Password.RequireLowercase = true;
                options.Password.RequireUppercase = false;
                options.Password.RequireNonAlphanumeric = false;
                options.Password.RequiredLength = 6;

                options.Lockout.MaxFailedAccessAttempts = 5;
                options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
            })
            .AddEntityFrameworkStores<IamDbContext>()
            .AddDefaultTokenProviders();

        services.Configure<JwtOptions>(configuration.GetSection("Jwt"));

        services.AddScoped<IIdentityService, IdentityService>();

        // Simple UnitOfWork for IAM (for future additional entities)
        services.AddScoped<IUnitOfWork, IamUnitOfWork>();

        return services;
    }

    public static async Task SeedIdentityAsync(IServiceProvider services, ILogger logger)
    {
        using var scope = services.CreateScope();
        var roleManager = scope.ServiceProvider.GetRequiredService<RoleManager<IdentityRole<Guid>>>();
        var userManager = scope.ServiceProvider.GetRequiredService<UserManager<ApplicationUser>>();

        foreach (var roleName in RoleNames.All)
        {
            if (!await roleManager.RoleExistsAsync(roleName))
            {
                var result = await roleManager.CreateAsync(new IdentityRole<Guid>
                {
                    Id = Guid.NewGuid(),
                    Name = roleName,
                    NormalizedName = roleName.ToUpperInvariant()
                });

                if (!result.Succeeded)
                {
                    logger.LogError("Failed to create role {Role}: {Errors}", roleName,
                        string.Join(", ", result.Errors.Select(e => e.Description)));
                }
            }
        }

        const string adminEmail = "admin@alvorbank.co.mz";
        const string adminPassword = "Admin123!";

        var adminUser = await userManager.FindByEmailAsync(adminEmail);
        if (adminUser is null)
        {
            adminUser = new ApplicationUser
            {
                Id = Guid.NewGuid(),
                Email = adminEmail,
                UserName = adminEmail,
                DisplayName = "System Administrator",
                UserType = UserType.Employee,
                IsActive = true
            };

            var createResult = await userManager.CreateAsync(adminUser, adminPassword);
            if (!createResult.Succeeded)
            {
                logger.LogError("Failed to create admin user: {Errors}",
                    string.Join(", ", createResult.Errors.Select(e => e.Description)));
                return;
            }

            var roleResult = await userManager.AddToRoleAsync(adminUser, RoleNames.SystemAdmin);
            if (!roleResult.Succeeded)
            {
                logger.LogError("Failed to assign SystemAdmin role: {Errors}",
                    string.Join(", ", roleResult.Errors.Select(e => e.Description)));
            }

            logger.LogInformation("Seeded SystemAdmin user with email {Email}", adminEmail);
        }
    }
}

public class IamUnitOfWork : IUnitOfWork
{
    private readonly IamDbContext _dbContext;

    public IamUnitOfWork(IamDbContext dbContext)
    {
        _dbContext = dbContext;
    }

    public Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
        => _dbContext.SaveChangesAsync(cancellationToken);
}
```

### 7.7.5 Adding EF Core migrations for IAM

Right now, when the IAM service starts, it executes:

```csharp
db.Database.Migrate();
```

in `Program.cs`. This tells EF Core to apply any pending migrations at startup.

If you see this in your Docker or application logs:

```text
No migrations were found in assembly 'BankingSuite.IamService.Infrastructure'.
A migration needs to be added before the database can be updated.
```

it simply means we haven’t created our **initial migration** yet.

We’ll do that now using `dotnet ef`.

First, make sure you have the EF Core tools installed:

```bash
dotnet tool install --global dotnet-ef
```

Add the EF tools package to the IAM Infrastructure project (this keeps design-time services happy):

```bash
cd digital-banking-suite

dotnet add src/backend/Services/IamService/BankingSuite.IamService.Infrastructure \
  package Microsoft.EntityFrameworkCore.Tools

  dotnet add src/backend/Services/IamService/BankingSuite.IamService.API package Microsoft.EntityFrameworkCore.Design
```

Now create the initial IAM migration.

From the backend folder:

```bash
cd src/backend

dotnet ef migrations add InitialIdentity \
  -p Services/IamService/BankingSuite.IamService.Infrastructure \
  -s Services/IamService/BankingSuite.IamService.API
```

Explanation:

- `-p` (**project**) points to the **Infrastructure** project where `IamDbContext` lives and where we want the `Migrations/` folder to be created.
- `-s` (**startup project**) points to the **API** project, which contains `Program.cs` and all configuration.

After running the command you should see a new folder:

```text
src/backend/Services/IamService/BankingSuite.IamService.Infrastructure/
└─ Migrations/
   ├─ 2025XXXXXXXXX_InitialIdentity.cs
   └─ IamDbContextModelSnapshot.cs
```

Now when you:

- Run the API locally (`dotnet run`) **or**
- Start it via Docker Compose:

```bash
cd digital-banking-suite
docker compose -f infra/docker-compose.dev.yml up -d iam-db iam-service
```

EF Core will find the `InitialIdentity` migration and automatically:

- Create the IAM database (if it doesn’t exist)
- Apply the migrations
- Then our seeding logic will insert:
  - All IAM roles (SystemAdmin, BackOfficeAgent, etc.)
  - The initial `SystemAdmin` user (`admin@alvorbank.co.mz`)

If you change IAM’s model in future chapters, you’ll repeat the same pattern:

```bash
cd digital-banking-suite/src/backend

dotnet ef migrations add SomeNewChangeName \
  -p Services/IamService/BankingSuite.IamService.Infrastructure \
  -s Services/IamService/BankingSuite.IamService.API
```

and let `db.Database.Migrate()` apply it at startup.

---

## 7.8 IAM API: FastEndpoints and Program.cs

The API layer is now very thin: it wires up Application + Infrastructure, configures JWT, and exposes endpoints through FastEndpoints.

### 7.8.1 Program.cs

`BankingSuite.IamService.API/Program.cs`:

```csharp
using System.Text;
using BankingSuite.BuildingBlocks.Web.Extensions;
using BankingSuite.IamService.Application;
using BankingSuite.IamService.Infrastructure;
using BankingSuite.IamService.Infrastructure.Auth;
using BankingSuite.IamService.Infrastructure.Persistence;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);

// Layers
builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddBankingFastEndpoints();

// JWT Authentication
builder.Services
    .AddAuthentication(o =>
    {
        o.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
        o.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    })
    .AddJwtBearer((sp, options) =>
    {
        var jwtOptions = sp.GetRequiredService<IOptions<JwtOptions>>().Value;

        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidIssuer = jwtOptions.Issuer,
            ValidAudience = jwtOptions.Audience,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtOptions.Key)),
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateIssuerSigningKey = true,
            ValidateLifetime = true
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

// Apply migrations
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<IamDbContext>();
    db.Database.Migrate();
}

// Seed roles + admin
await DependencyInjection.SeedIdentityAsync(app.Services, app.Logger);

if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}

app.UseAuthentication();
app.UseAuthorization();

app.UseBankingFastEndpoints();

app.Run();
```

### 7.8.2 Login Endpoint

`BankingSuite.IamService.API/Endpoints/Auth/LoginEndpoint.cs`:

```csharp
using BankingSuite.IamService.Application.Auth;
using BankingSuite.IamService.Application.Auth.Dto;
using FastEndpoints;
using MediatR;

namespace BankingSuite.IamService.API.Endpoints.Auth;

public class LoginRequest
{
    public string Email { get; set; } = default!;
    public string Password { get; set; } = default!;
}

public class LoginResponse
{
    public string AccessToken { get; set; } = default!;
    public DateTime ExpiresAt { get; set; }
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
            s.Summary = "Login and receive a JWT access token";
        });
    }

    public override async Task HandleAsync(LoginRequest req, CancellationToken ct)
    {
        var result = await _mediator.Send(new LoginCommand(req.Email, req.Password), ct);

        if (result is null)
        {
            await SendUnauthorizedAsync(ct);
            return;
        }

        await SendAsync(new LoginResponse
        {
            AccessToken = result.AccessToken,
            ExpiresAt = result.ExpiresAt
        }, cancellation: ct);
    }
}
```

### 7.8.3 CreateUser Endpoint (SystemAdmin only)

`BankingSuite.IamService.API/Endpoints/Users/CreateUserEndpoint.cs`:

```csharp
using BankingSuite.IamService.Application.Users;
using BankingSuite.IamService.Application.Users.Dto;
using BankingSuite.IamService.Domain.Users;
using FastEndpoints;
using MediatR;

namespace BankingSuite.IamService.API.Endpoints.Users;

public class CreateUserRequest
{
    public string Email { get; set; } = default!;
    public string DisplayName { get; set; } = default!;
    public string Password { get; set; } = default!;
    public UserType UserType { get; set; }
    public List<string> Roles { get; set; } = new();
}

public class CreateUserResponse
{
    public Guid Id { get; set; }
    public string Email { get; set; } = default!;
    public string DisplayName { get; set; } = default!;
    public UserType UserType { get; set; }
    public bool IsActive { get; set; }
    public List<string> Roles { get; set; } = new();
}

public class CreateUserEndpoint : Endpoint<CreateUserRequest, CreateUserResponse>
{
    private readonly IMediator _mediator;

    public CreateUserEndpoint(IMediator mediator)
    {
        _mediator = mediator;
    }

    public override void Configure()
    {
        Post("/users");
        Roles(RoleNames.SystemAdmin);
        Summary(s =>
        {
            s.Summary = "Create a new IAM user (SystemAdmin only)";
        });
    }

    public override async Task HandleAsync(CreateUserRequest req, CancellationToken ct)
    {
        var command = new CreateUserCommand(
            req.Email,
            req.DisplayName,
            req.Password,
            req.UserType,
            req.Roles
        );

        UserDto user = await _mediator.Send(command, ct);

        await SendCreatedAtAsync(
            "/users/{id}",
            new { id = user.Id },
            new CreateUserResponse
            {
                Id = user.Id,
                Email = user.Email,
                DisplayName = user.DisplayName,
                UserType = user.UserType,
                IsActive = user.IsActive,
                Roles = user.Roles.ToList()
            },
            cancellation: ct);
    }
}
```

At this point IAM is fully wired:

- API → Application → Infrastructure → Identity/EF.
- `/auth/login` returns JWT tokens.
- `/users` allows a SystemAdmin to create new staff or customer logins.

---

## 7.9 Dockerizing IAM and the Database

We want everything to run in containers, like a real banking environment.

### 7.9.1 IAM Dockerfile

In `BankingSuite.IamService.API/` create `Dockerfile`:

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

COPY ../../.. .
WORKDIR /src/src/backend/Services/IamService/BankingSuite.IamService.API

RUN dotnet restore
RUN dotnet publish -c Release -o /app/publish

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080

ENTRYPOINT ["dotnet", "BankingSuite.IamService.API.dll"]
```

Adjust the relative paths if your repo structure differs.

### 7.9.2 docker-compose.dev.yml (infra)

In this book we keep all Docker Compose files under an `infra` folder so that infrastructure and runtime concerns are clearly separated from application source code.

The structure looks like this:

```text
digital-banking-suite/
├─ src/
│  └─ backend/
│     └─ ...
└─ infra/
   └─ docker-compose.dev.yml
```

Open `infra/docker-compose.dev.yml` and add the IAM database and IAM service:

```yaml
version: "3.9"

services:
  iam-db:
    image: postgres:16
    container_name: iam-db
    environment:
      POSTGRES_USER: iam_user
      POSTGRES_PASSWORD: Password123!
      POSTGRES_DB: iam_service_db
    ports:
      - "5433:5432" # external 5433 -> internal 5432
    volumes:
      - iam-db-data:/var/lib/postgresql/data
    networks:
      - banking-net

  iam-service:
    build:
      # The context is the repo root (one level up from infra/)
      context: ..
      dockerfile: src/backend/Services/IamService/BankingSuite.IamService.API/Dockerfile
    container_name: iam-service
    depends_on:
      - iam-db
    environment:
      ASPNETCORE_ENVIRONMENT: "Development"
      ConnectionStrings__IamDatabase: "Host=iam-db;Port=5432;Database=iam_service_db;Username=iam_user;Password=Password123!"
      Jwt__Issuer: "AlvorBank.IamService"
      Jwt__Audience: "AlvorBank.ApiClients"
      Jwt__Key: "SUPER_SECRET_DEMO_KEY_CHANGE_ME"
      Jwt__AccessTokenLifetimeMinutes: "60"
    ports:
      - "5001:8080"
    networks:
      - banking-net

networks:
  banking-net:

volumes:
  iam-db-data:
```

To spin up IAM and its Postgres database from the repo root:

```bash
cd digital-banking-suite
docker compose -f infra/docker-compose.dev.yml up -d iam-db iam-service
```

After the containers are up, IAM is available at:

- `http://localhost:5001/api/auth/login`
- `http://localhost:5001/api/users`

---

## 7.10 Postman Test Collection for IAM

Before we build the Angular frontend, we want a **Postman collection** that exercises our IAM API end-to-end:

- Login as the seeded `SystemAdmin`
- Use the token to create a new back-office user
- (Optionally) log in with the new user to verify

We’ll keep all collections in a dedicated folder:

```text
digital-banking-suite/
└─ postman/
   ├─ iam.postman_collection.json
   └─ environments/
      └─ local.postman_environment.json
```

### 7.10.1 Create the Postman environment

Create a new Postman environment named `DigitalBanking-Local` with variables:

- `iam_base_url` → `http://localhost:5001/api`
- `iam_admin_email` → `admin@alvorbank.co.mz`
- `iam_admin_password` → `Admin123!`
- `iam_access_token` → _initially empty_

You can also export this environment as JSON and commit it under:

- `postman/environments/local.postman_environment.json`

Example environment JSON (shortened):

```json
{
  "name": "DigitalBanking-Local",
  "values": [
    {
      "key": "iam_base_url",
      "value": "http://localhost:5001/api",
      "type": "default",
      "enabled": true
    },
    {
      "key": "iam_admin_email",
      "value": "admin@alvorbank.co.mz",
      "type": "default",
      "enabled": true
    },
    {
      "key": "iam_admin_password",
      "value": "Admin123!",
      "type": "default",
      "enabled": true
    },
    {
      "key": "iam_access_token",
      "value": "",
      "type": "default",
      "enabled": true
    }
  ]
}
```

### 7.10.2 Request 1: IAM – Login (SystemAdmin)

Create a new collection named **Digital Banking – IAM** and add a request:

- **Name**: `IAM - Login (SystemAdmin)`
- **Method**: `POST`
- **URL**: `{{iam_base_url}}/auth/login`
- **Body (raw JSON)**:

```json
{
  "email": "{{iam_admin_email}}",
  "password": "{{iam_admin_password}}"
}
```

In the **Tests** tab, paste:

```javascript
pm.test("Status code is 200", function () {
  pm.response.to.have.status(200);
});

const json = pm.response.json();
pm.test("Response has accessToken", function () {
  pm.expect(json).to.have.property("accessToken");
});

pm.environment.set("iam_access_token", json.accessToken);
```

This test:

- Verifies we get `200 OK`
- Checks the response contains `accessToken`
- Stores the token into the environment variable `iam_access_token`

### 7.10.3 Request 2: IAM – Create BackOffice User

Add a second request to the same collection:

- **Name**: `IAM - Create BackOffice User`
- **Method**: `POST`
- **URL**: `{{iam_base_url}}/users`
- **Auth**: Type = `Bearer Token`, Token = `{{iam_access_token}}`
- **Body (raw JSON)**:

```json
{
  "email": "backoffice.agent+demo@alvorbank.co.mz",
  "displayName": "Demo BackOffice Agent",
  "password": "BackOffice123!",
  "userType": 1,
  "roles": ["BackOfficeAgent"]
}
```

In the **Tests** tab:

```javascript
pm.test("Status code is 201", function () {
  pm.response.to.have.status(201);
});

const json = pm.response.json();
pm.test("Created user has id", function () {
  pm.expect(json.id).to.exist;
});

pm.environment.set("iam_backoffice_email", json.email);
```

This ensures:

- Only a valid `SystemAdmin` token can create users.
- The endpoint returns `201 Created` with a proper `id`.

### 7.10.4 Request 3: IAM – Login (BackOffice User) [Optional]

Add an optional request:

- **Name**: `IAM - Login (BackOfficeAgent)`
- **Method**: `POST`
- **URL**: `{{iam_base_url}}/auth/login`
- **Body**:

```json
{
  "email": "{{iam_backoffice_email}}",
  "password": "BackOffice123!"
}
```

Tests:

```javascript
pm.test("Status code is 200", function () {
  pm.response.to.have.status(200);
});

const json = pm.response.json();
pm.environment.set("iam_backoffice_access_token", json.accessToken);
```

Now your Postman collection gives you:

- A **smoke test** for IAM
- Reusable tokens for manual exploration
- A JSON file (`postman/iam.postman_collection.json`) that can be imported by the whole team

Later we can also wire these Postman collections into **Newman** runs in CI if we want extra API checks.

---

## 7.11 Tests and Coverage ≥ 80% for IAM

Real banking teams don’t merge without tests and coverage.

### 7.11.1 Create Test Projects

Under `tests/backend/IamService` (create folders as needed):

```bash
cd tests/backend

dotnet new xunit -n BankingSuite.IamService.UnitTests
dotnet new xunit -n BankingSuite.IamService.IntegrationTests
```

Add references:

```bash
dotnet add BankingSuite.IamService.UnitTests reference \
  ../../src/backend/Services/IamService/BankingSuite.IamService.Application \
  ../../src/backend/Services/IamService/BankingSuite.IamService.Domain

dotnet add BankingSuite.IamService.IntegrationTests reference \
  ../../src/backend/Services/IamService/BankingSuite.IamService.API
```

Add test libraries:

```bash
dotnet add BankingSuite.IamService.UnitTests package Moq
dotnet add BankingSuite.IamService.IntegrationTests package Microsoft.AspNetCore.Mvc.Testing
```

### 7.11.2 Example Unit Tests

For `LoginCommandHandler`:

```csharp
// BankingSuite.IamService.UnitTests/Auth/LoginCommandHandlerTests.cs
using System.Threading;
using System.Threading.Tasks;
using BankingSuite.IamService.Application.Auth;
using BankingSuite.IamService.Application.Auth.Dto;
using BankingSuite.IamService.Application.Common.Interfaces;
using Moq;
using Xunit;

public class LoginCommandHandlerTests
{
    [Fact]
    public async Task Handle_ReturnsResult_WhenCredentialsAreValid()
    {
        // Arrange
        var identityServiceMock = new Mock<IIdentityService>();
        var expected = new LoginResultDto("token", DateTime.UtcNow.AddMinutes(10));

        identityServiceMock
            .Setup(s => s.LoginAsync("user@alvorbank.co.mz", "password", It.IsAny<CancellationToken>()))
            .ReturnsAsync(expected);

        var handler = new LoginCommandHandler(identityServiceMock.Object);

        var command = new LoginCommand("user@alvorbank.co.mz", "password");

        // Act
        var result = await handler.Handle(command, CancellationToken.None);

        // Assert
        Assert.NotNull(result);
        Assert.Equal("token", result!.AccessToken);
    }
}
```

Similarly, add tests for `CreateUserCommandHandler` and any other important handlers.

### 7.11.3 Example Integration Test

Use `WebApplicationFactory<Program>` in `BankingSuite.IamService.IntegrationTests` to boot the IAM API and test `/auth/login` and `/users`. This validates:

- Wiring of Program.cs
- Identity configuration
- JWT auth pipeline

### 7.11.4 Running Tests and Coverage

From repo root:

```bash
cd digital-banking-suite

dotnet test src/backend/BankingSuite.Backend.sln \
  /p:CollectCoverage=true \
  /p:CoverletOutput=../../../lcov \
  /p:CoverletOutputFormat=lcov
```

Generate the coverage report:

```bash
cd digital-banking-suite

reportgenerator \
  -reports:lcov.info \
  -targetdir:src/backend/coverage-report \
  "-reporttypes:Html;TextSummary"
```

Open `src/backend/coverage-report/index.html` in a browser and verify:

- Overall coverage is at least **80%**.
- IAM projects (`BankingSuite.IamService.*`) individually are above 80% where feasible.

If coverage is too low, add more tests until it passes the threshold.

---

## 7.12 Updating CI and Opening the PR

We now integrate IAM into the CI pipeline and merge like a real team.

### 7.12.1 CI: Build, Test, Coverage Gate

In your CI YAML (for example, `.github/workflows/backend.yml`), ensure you:

- Build the solution.
- Run tests with coverage (as above).
- Optionally enforce the 80% threshold via a coverage tool or script.

Later we can extend the pipeline to:

- Build Docker images for IAM
- (Optionally) run Newman against the IAM Postman collection

### 7.12.2 Git Status and Commit

From repo root:

```bash
git status
git add .
git commit -m "Chapter 07: Final backend structure & IAM Service backend"
git push -u origin feature/chapter-07-iam-backend
```

### 7.12.3 Create PR and Watch CI

- Open a Pull Request:  
  `feature/chapter-07-iam-backend → develop`
- Confirm CI:
  - Builds all backend projects.
  - Runs tests.
  - Generates coverage report.

Only after CI goes green and coverage is **≥80%**, we merge.

```bash
# After PR is approved and merged:
git checkout develop
git pull origin develop
```

The backend structure and IAM are now part of the main development branch.

---

## 7.13 Summary and What’s Next

In this chapter we:

- **Locked in** the final backend architecture:
  - `Domain`, `Application`, `Infrastructure`, `API` per microservice
  - Shared **BuildingBlocks** for DDD, CQRS and web configuration
- Implemented the **IAM Service** end–to–end:
  - Domain: `UserType`, `RoleNames`
  - Application: CQRS with `LoginCommand` and `CreateUserCommand`
  - Infrastructure: ASP.NET Identity, EF Core Postgres, JWT issuing, seeding roles + SystemAdmin
  - API: FastEndpoints for `/auth/login` and `/users`
- **Dockerized** IAM and its Postgres database.
- Added **tests** and enforced a **coverage mindset** (≥80% before merging).
- Created a **Postman collection** to exercise IAM APIs and share with the team.
- Went through a full **branch → CI → PR → merge** workflow, just like a real banking team.

In the **next chapter**, we will:

- Move into the **Angular Nx frontend monorepo**.
- Implement a **real login screen** that talks to IAM.
- Introduce a basic **back–office shell** for SystemAdmin and staff users.
- Wire up role–aware navigation so IAM’s JWT really comes to life in the UI.
