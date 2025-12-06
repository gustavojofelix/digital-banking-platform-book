## 8.1 Admin Employee Management (List/Get/Update/Activate/Deactivate)

In this section we turn our IAM service into a **real admin console backend** for Alvor Bank.

So far, our IAM microservice already has:

- `ApplicationUser : IdentityUser<Guid>` and `ApplicationRole`
- A registration endpoint for employees: `POST /auth/employees`
- A login endpoint: `POST /api/iam/auth/login`
- Clean Architecture, CQRS with MediatR, and **FastEndpoints** as the HTTP layer

Now we will let **bank administrators** manage employees:

- List all employees (with paging + filters)
- See employee details
- Update basic profile and roles
- Activate / deactivate employees

We’ll do this using:

- **FastEndpoints** for the HTTP layer
- **MediatR CQRS** in the Application layer
- **ASP.NET Core Identity** for users and roles
- A shared **`PagedResult<T>`** in the BuildingBlocks library
- Modern C# features (records, primary constructors, collection expressions, etc.)

---

### 8.1.1 Branch Strategy for This Section

We keep the same Git discipline we established earlier:

- `main` → always points to the latest **stable, published** book state
- `develop` → integration branch for ongoing **Part II** backend work
- One **chapter/section branch** per major step, kept as a **permanent reference** (we **do not** delete them)

For this section we will use:

- **Branch name:** `part2-chapter08-iam-admin-employee-management`

From the repo root:

```bash
# Make sure you're up to date on develop
git checkout develop
git pull origin develop

# Create a new branch for this section
git checkout -b part2-chapter08-iam-admin-employee-management
```

We’ll do all changes in this branch, run tests, then open a PR into `develop` when we’re done with this section.

---

### 8.1.2 Shared Building Block: `PagedResult<T>`

We want paging to be **consistent** across all microservices. Instead of redefining `PagedResult<T>` inside IAM, we put it once into our **BuildingBlocks** project and reuse it everywhere.

Create (or update) this file:

- `src/backend/building-blocks/BankingSuite.BuildingBlocks.Application/Models/PagedResult.cs`

```csharp
namespace BankingSuite.BuildingBlocks.Application.Models;

public sealed record PagedResult<T>(
    IReadOnlyList<T> Items,
    int PageNumber,
    int PageSize,
    int TotalCount)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => PageNumber < TotalPages;
    public bool HasPreviousPage => PageNumber > 1;

    public static PagedResult<T> Empty(int pageNumber = 1, int pageSize = 20)
        => new([], pageNumber, pageSize, 0); // [] = C# collection expression
}
```

Key points:

- Using a **record with a primary constructor** for immutability and concise syntax
- `[]` is a modern C# **collection expression** used to create an empty list
- Reusable paging metadata (`TotalPages`, `HasNextPage`, `HasPreviousPage`)

In IAM we will simply add:

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
```

whenever we need paged results.

---

### 8.1.3 Extending `ApplicationUser` for Admin Management

Our IAM service is **single-tenant** for a fictitious Alvor Bank, so one bank, one IAM. We still want richer user information and explicit activation state.

Open `ApplicationUser` in the IAM Domain project (path may vary, adjust to your repo):

- `src/backend/services/iam/BankingSuite.IAM.Domain/ApplicationUser.cs`

Make sure it looks like this (or equivalent):

```csharp
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IAM.Domain;

public sealed class ApplicationUser : IdentityUser<Guid>
{
    public string? FullName { get; set; }

    // Explicit active/inactive flag for admin control
    public bool IsActive { get; set; } = true;

    // Optional: audit info; we will update this from the login flow later
    public DateTimeOffset? LastLoginAt { get; set; }
}
```

If you already have some of these properties, just align names (`FullName`, `IsActive`, `LastLoginAt`) so they match this chapter.

If the entity changed, don’t forget to generate and apply a new migration later.

---

### 8.1.4 Employee DTOs (Application Layer)

In this IAM, every `ApplicationUser` is effectively a **bank employee**. We’ll keep DTOs in an `Employees` feature folder.

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Dtos/EmployeeDtos.cs`

```csharp
namespace BankingSuite.IAM.Application.Employees.Dtos;

public sealed class EmployeeSummaryDto
{
    public Guid Id { get; init; }
    public string Email { get; init; } = default!;
    public string? FullName { get; init; }
    public bool EmailConfirmed { get; init; }
    public bool IsActive { get; init; }
    public bool TwoFactorEnabled { get; init; }
    public string[] Roles { get; init; } = [];
}

public sealed class EmployeeDetailsDto : EmployeeSummaryDto
{
    public string? PhoneNumber { get; init; }
    public DateTimeOffset? LockoutEnd { get; init; }
    public DateTimeOffset? LastLoginAt { get; init; }
}
```

DTOs are **read models**, designed specifically for admin screens (and later our Angular IAM UI).  
We include `Roles` so the admin UI can show or edit roles without extra round-trips.

---

### 8.1.5 CQRS Messages: Queries and Commands

We model our use cases as **CQRS messages** using MediatR and modern C# records.

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Queries/ListEmployeesQuery.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Queries/GetEmployeeDetailsQuery.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Commands/UpdateEmployeeCommand.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Commands/EmployeeActivationCommands.cs`

**ListEmployeesQuery**

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
using BankingSuite.IAM.Application.Employees.Dtos;
using MediatR;

namespace BankingSuite.IAM.Application.Employees.Queries;

public sealed record ListEmployeesQuery(
    int PageNumber,
    int PageSize,
    string? Search,
    bool IncludeInactive
) : IRequest<PagedResult<EmployeeSummaryDto>>;
```

**GetEmployeeDetailsQuery**

```csharp
using BankingSuite.IAM.Application.Employees.Dtos;
using MediatR;

namespace BankingSuite.IAM.Application.Employees.Queries;

public sealed record GetEmployeeDetailsQuery(Guid EmployeeId)
    : IRequest<EmployeeDetailsDto?>;
```

**UpdateEmployeeCommand**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Employees.Commands;

public sealed record UpdateEmployeeCommand(
    Guid EmployeeId,
    string? FullName,
    string? PhoneNumber,
    string[] Roles
) : IRequest;
```

**Activate / Deactivate Employee Commands**

```csharp
using MediatR;

namespace BankingSuite.IAM.Application.Employees.Commands;

public sealed record ActivateEmployeeCommand(Guid EmployeeId) : IRequest;

public sealed record DeactivateEmployeeCommand(Guid EmployeeId) : IRequest;
```

These records capture the **intent** of each operation and act as the boundary between HTTP and business logic.

---

### 8.1.6 Handlers: Identity + EF Core + Modern C# (Grouped by Use Case)

Just like we did for the login flow (`Auth/Commands/Login`), we’ll keep each
**command/query and its handler together in the same folder**. This makes it easy
to navigate per use case and keeps the Application layer consistent.

Our structure under `BankingSuite.IAM.Application` will look like this:

- `Employees/Queries/ListEmployees/ListEmployeesQuery.cs`
- `Employees/Queries/ListEmployees/ListEmployeesQueryHandler.cs`
- `Employees/Queries/GetEmployeeDetails/GetEmployeeDetailsQuery.cs`
- `Employees/Queries/GetEmployeeDetails/GetEmployeeDetailsQueryHandler.cs`
- `Employees/Commands/UpdateEmployee/UpdateEmployeeCommand.cs`
- `Employees/Commands/UpdateEmployee/UpdateEmployeeCommandHandler.cs`
- `Employees/Commands/ActivateEmployee/ActivateEmployeeCommand.cs`
- `Employees/Commands/ActivateEmployee/ActivateEmployeeCommandHandler.cs`
- `Employees/Commands/DeactivateEmployee/DeactivateEmployeeCommand.cs`
- `Employees/Commands/DeactivateEmployee/DeactivateEmployeeCommandHandler.cs`

We already created the records in the previous section; now we add the handlers
next to them.

---

#### ListEmployeesQueryHandler

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Queries/ListEmployees/ListEmployeesQueryHandler.cs`

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
using BankingSuite.IAM.Application.Employees.Dtos;
using BankingSuite.IAM.Application.Employees.Queries;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IAM.Application.Employees.Queries.ListEmployees;

public sealed class ListEmployeesQueryHandler
    : IRequestHandler<ListEmployeesQuery, PagedResult<EmployeeSummaryDto>>
{
    private readonly UserManager<ApplicationUser> _userManager;

    public ListEmployeesQueryHandler(UserManager<ApplicationUser> userManager)
        => _userManager = userManager ?? throw new ArgumentNullException(nameof(userManager));

    public async Task<PagedResult<EmployeeSummaryDto>> Handle(
        ListEmployeesQuery request,
        CancellationToken cancellationToken)
    {
        var query = _userManager.Users.AsNoTracking();

        if (!request.IncludeInactive)
        {
            query = query.Where(u => u.IsActive);
        }

        if (!string.IsNullOrWhiteSpace(request.Search))
        {
            var search = request.Search.Trim().ToLowerInvariant();

            query = query.Where(u =>
                u.Email!.ToLower().Contains(search) ||
                (u.FullName != null && u.FullName.ToLower().Contains(search)));
        }

        var totalCount = await query.CountAsync(cancellationToken);

        var users = await query
            .OrderBy(u => u.Email)
            .Skip((request.PageNumber - 1) * request.PageSize)
            .Take(request.PageSize)
            .ToListAsync(cancellationToken);

        var items = new List<EmployeeSummaryDto>(users.Count);

        foreach (var user in users)
        {
            var roles = await _userManager.GetRolesAsync(user);

            items.Add(new EmployeeSummaryDto
            {
                Id = user.Id,
                Email = user.Email!,
                FullName = user.FullName,
                EmailConfirmed = user.EmailConfirmed,
                IsActive = user.IsActive,
                TwoFactorEnabled = user.TwoFactorEnabled,
                Roles = roles.ToArray()
            });
        }

        return new PagedResult<EmployeeSummaryDto>(
            Items: items,
            PageNumber: request.PageNumber,
            PageSize: request.PageSize,
            TotalCount: totalCount);
    }
}
```

> Note the namespace: `Employees.Queries.ListEmployees`.  
> This mirrors the folder layout and keeps types grouped by use case.

---

#### GetEmployeeDetailsQueryHandler

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Queries/GetEmployeeDetails/GetEmployeeDetailsQueryHandler.cs`

```csharp
using BankingSuite.IAM.Application.Employees.Dtos;
using BankingSuite.IAM.Application.Employees.Queries;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IAM.Application.Employees.Queries.GetEmployeeDetails;

public sealed class GetEmployeeDetailsQueryHandler
    : IRequestHandler<GetEmployeeDetailsQuery, EmployeeDetailsDto?>
{
    private readonly UserManager<ApplicationUser> _userManager;

    public GetEmployeeDetailsQueryHandler(UserManager<ApplicationUser> userManager)
        => _userManager = userManager ?? throw new ArgumentNullException(nameof(userManager));

    public async Task<EmployeeDetailsDto?> Handle(
        GetEmployeeDetailsQuery request,
        CancellationToken cancellationToken)
    {
        var user = await _userManager.Users
            .AsNoTracking()
            .FirstOrDefaultAsync(u => u.Id == request.EmployeeId, cancellationToken);

        if (user is null)
            return null;

        var roles = await _userManager.GetRolesAsync(user);

        return new EmployeeDetailsDto
        {
            Id = user.Id,
            Email = user.Email!,
            FullName = user.FullName,
            EmailConfirmed = user.EmailConfirmed,
            IsActive = user.IsActive,
            TwoFactorEnabled = user.TwoFactorEnabled,
            PhoneNumber = user.PhoneNumber,
            LockoutEnd = user.LockoutEnd,
            LastLoginAt = user.LastLoginAt,
            Roles = roles.ToArray()
        };
    }
}
```

---

#### UpdateEmployeeCommandHandler

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Commands/UpdateEmployee/UpdateEmployeeCommandHandler.cs`

```csharp
using BankingSuite.IAM.Application.Employees.Commands;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IAM.Application.Employees.Commands.UpdateEmployee;

public sealed class UpdateEmployeeCommandHandler : IRequestHandler<UpdateEmployeeCommand>
{
    private readonly UserManager<ApplicationUser> _userManager;

    public UpdateEmployeeCommandHandler(UserManager<ApplicationUser> userManager)
        => _userManager = userManager ?? throw new ArgumentNullException(nameof(userManager));

    public async Task<Unit> Handle(UpdateEmployeeCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.Users
            .FirstOrDefaultAsync(u => u.Id == request.EmployeeId, cancellationToken);

        if (user is null)
            throw new KeyNotFoundException("Employee not found.");

        if (request.FullName is not null)
            user.FullName = request.FullName;

        if (request.PhoneNumber is not null)
            user.PhoneNumber = request.PhoneNumber;

        var updateResult = await _userManager.UpdateAsync(user);
        if (!updateResult.Succeeded)
        {
            var errors = string.Join("; ", updateResult.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to update employee: {errors}");
        }

        var currentRoles = await _userManager.GetRolesAsync(user);
        var requestedRoles = request.Roles ?? Array.Empty<string>();

        var rolesToAdd = requestedRoles.Except(currentRoles).ToArray();
        var rolesToRemove = currentRoles.Except(requestedRoles).ToArray();

        if (rolesToAdd.Any())
        {
            var addResult = await _userManager.AddToRolesAsync(user, rolesToAdd);
            if (!addResult.Succeeded)
            {
                var errors = string.Join("; ", addResult.Errors.Select(e => e.Description));
                throw new InvalidOperationException($"Failed to add roles: {errors}");
            }
        }

        if (rolesToRemove.Any())
        {
            var removeResult = await _userManager.RemoveFromRolesAsync(user, rolesToRemove);
            if (!removeResult.Succeeded)
            {
                var errors = string.Join("; ", removeResult.Errors.Select(e => e.Description));
                throw new InvalidOperationException($"Failed to remove roles: {errors}");
            }
        }

        return Unit.Value;
    }
}
```

---

#### Activate / Deactivate Employee Handlers

Create:

- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Commands/ActivateEmployee/ActivateEmployeeCommandHandler.cs`
- `src/backend/services/iam/BankingSuite.IAM.Application/Employees/Commands/DeactivateEmployee/DeactivateEmployeeCommandHandler.cs`

**ActivateEmployeeCommandHandler**

```csharp
using BankingSuite.IAM.Application.Employees.Commands;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IAM.Application.Employees.Commands.ActivateEmployee;

public sealed class ActivateEmployeeCommandHandler : IRequestHandler<ActivateEmployeeCommand>
{
    private readonly UserManager<ApplicationUser> _userManager;

    public ActivateEmployeeCommandHandler(UserManager<ApplicationUser> userManager)
        => _userManager = userManager ?? throw new ArgumentNullException(nameof(userManager));

    public async Task<Unit> Handle(ActivateEmployeeCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.Users
            .FirstOrDefaultAsync(u => u.Id == request.EmployeeId, cancellationToken);

        if (user is null)
            throw new KeyNotFoundException("Employee not found.");

        user.IsActive = true;
        user.LockoutEnd = null;

        var result = await _userManager.UpdateAsync(user);
        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to activate employee: {errors}");
        }

        return Unit.Value;
    }
}
```

**DeactivateEmployeeCommandHandler**

```csharp
using BankingSuite.IAM.Application.Employees.Commands;
using BankingSuite.IAM.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IAM.Application.Employees.Commands.DeactivateEmployee;

public sealed class DeactivateEmployeeCommandHandler : IRequestHandler<DeactivateEmployeeCommand>
{
    private readonly UserManager<ApplicationUser> _userManager;

    public DeactivateEmployeeCommandHandler(UserManager<ApplicationUser> userManager)
        => _userManager = userManager ?? throw new ArgumentNullException(nameof(userManager));

    public async Task<Unit> Handle(DeactivateEmployeeCommand request, CancellationToken cancellationToken)
    {
        var user = await _userManager.Users
            .FirstOrDefaultAsync(u => u.Id == request.EmployeeId, cancellationToken);

        if (user is null)
            throw new KeyNotFoundException("Employee not found.");

        user.IsActive = false;
        user.LockoutEnd = DateTimeOffset.UtcNow.AddYears(100); // effectively frozen

        var result = await _userManager.UpdateAsync(user);
        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to deactivate employee: {errors}");
        }

        return Unit.Value;
    }
}
```

With this structure, **every use case lives in a small, discoverable folder** containing:

- The request model (`Command` or `Query` record)
- The `Handler` that implements it

This matches what we did in earlier chapters for authentication and keeps the Application layer consistent and easy to navigate.

---

### 8.1.7 FastEndpoints: Admin Employees API

Now we expose this functionality via **FastEndpoints**. Each endpoint:

- Accepts an HTTP request DTO
- Converts it to a CQRS command or query
- Sends it via `IMediator`
- Returns the appropriate HTTP response

Create the following files in the IAM API project:

- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Admin/Employees/ListEmployeesEndpoint.cs`
- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Admin/Employees/GetEmployeeEndpoint.cs`
- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Admin/Employees/UpdateEmployeeEndpoint.cs`
- `src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Admin/Employees/EmployeeStatusEndpoints.cs`
- (Optional) `IamAdminGroup` if you use FastEndpoints groups

First, define the request DTOs:

```csharp
namespace BankingSuite.IAM.Api.Endpoints.Admin.Employees;

public sealed class ListEmployeesRequest
{
    public int PageNumber { get; init; } = 1;
    public int PageSize  { get; init; } = 20;
    public string? Search { get; init; }
    public bool IncludeInactive { get; init; } = true;
}

public sealed class EmployeeByIdRequest
{
    public Guid Id { get; init; } // bound from route {id}
}

public sealed class UpdateEmployeeRequest
{
    public Guid Id { get; init; } // route {id}

    public string? FullName { get; init; }
    public string? PhoneNumber { get; init; }
    public string[] Roles { get; init; } = [];
}

public sealed class EmployeeStatusRequest
{
    public Guid Id { get; init; } // route {id}
}
```

**ListEmployeesEndpoint**

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
using BankingSuite.IAM.Application.Employees.Dtos;
using BankingSuite.IAM.Application.Employees.Queries;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Admin.Employees;

public sealed class ListEmployeesEndpoint(IMediator mediator)
    : Endpoint<ListEmployeesRequest, PagedResult<EmployeeSummaryDto>>
{
    public override void Configure()
    {
        Get("/api/iam/admin/employees");
        Roles("IamAdmin", "SuperAdmin"); // adjust role names to your scheme
        // Group<IamAdminGroup>(); // if you use FastEndpoints groups
    }

    public override async Task HandleAsync(ListEmployeesRequest req, CancellationToken ct)
    {
        var result = await mediator.Send(
            new ListEmployeesQuery(
                PageNumber: req.PageNumber,
                PageSize: req.PageSize,
                Search: req.Search,
                IncludeInactive: req.IncludeInactive),
            ct);

        await SendOkAsync(result, ct);
    }
}
```

**GetEmployeeEndpoint**

```csharp
using BankingSuite.IAM.Application.Employees.Dtos;
using BankingSuite.IAM.Application.Employees.Queries;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Admin.Employees;

public sealed class GetEmployeeEndpoint(IMediator mediator)
    : Endpoint<EmployeeByIdRequest, EmployeeDetailsDto>
{
    public override void Configure()
    {
        Get("/api/iam/admin/employees/{id:guid}");
        Roles("IamAdmin", "SuperAdmin");
        // Group<IamAdminGroup>();
    }

    public override async Task HandleAsync(EmployeeByIdRequest req, CancellationToken ct)
    {
        var result = await mediator.Send(
            new GetEmployeeDetailsQuery(req.Id),
            ct);

        if (result is null)
        {
            await SendNotFoundAsync(ct);
            return;
        }

        await SendOkAsync(result, ct);
    }
}
```

**UpdateEmployeeEndpoint**

```csharp
using BankingSuite.IAM.Application.Employees.Commands;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Admin.Employees;

public sealed class UpdateEmployeeEndpoint(IMediator mediator)
    : Endpoint<UpdateEmployeeRequest>
{
    public override void Configure()
    {
        Put("/api/iam/admin/employees/{id:guid}");
        Roles("IamAdmin", "SuperAdmin");
        // Group<IamAdminGroup>();
    }

    public override async Task HandleAsync(UpdateEmployeeRequest req, CancellationToken ct)
    {
        var cmd = new UpdateEmployeeCommand(
            EmployeeId: req.Id,
            FullName: req.FullName,
            PhoneNumber: req.PhoneNumber,
            Roles: req.Roles);

        await mediator.Send(cmd, ct);

        await SendNoContentAsync(ct);
    }
}
```

**Activate / Deactivate Employee Endpoints**

```csharp
using BankingSuite.IAM.Application.Employees.Commands;
using MediatR;

namespace BankingSuite.IAM.Api.Endpoints.Admin.Employees;

public sealed class ActivateEmployeeEndpoint(IMediator mediator)
    : Endpoint<EmployeeStatusRequest>
{
    public override void Configure()
    {
        Post("/api/iam/admin/employees/{id:guid}/activate");
        Roles("IamAdmin", "SuperAdmin");
        // Group<IamAdminGroup>();
    }

    public override async Task HandleAsync(EmployeeStatusRequest req, CancellationToken ct)
    {
        await mediator.Send(new ActivateEmployeeCommand(req.Id), ct);
        await SendNoContentAsync(ct);
    }
}

public sealed class DeactivateEmployeeEndpoint(IMediator mediator)
    : Endpoint<EmployeeStatusRequest>
{
    public override void Configure()
    {
        Post("/api/iam/admin/employees/{id:guid}/deactivate");
        Roles("IamAdmin", "SuperAdmin");
        // Group<IamAdminGroup>();
    }

    public override async Task HandleAsync(EmployeeStatusRequest req, CancellationToken ct)
    {
        await mediator.Send(new DeactivateEmployeeCommand(req.Id), ct);
        await SendNoContentAsync(ct);
    }
}
```

At this point, our IAM backend supports:

- Listing employees with paging and search
- Viewing a specific employee
- Updating profile and roles
- Activating and deactivating employee accounts

All using FastEndpoints + CQRS + Identity + modern C#.

---

### 8.1.8 Quick Local Test, Commit and Push

Build and run the backend (using your usual dev compose or Aspire setup), then hit the new endpoints via Swagger or a REST client:

- `GET    /api/iam/admin/employees`
- `GET    /api/iam/admin/employees/{id}`
- `PUT    /api/iam/admin/employees/{id}`
- `POST   /api/iam/admin/employees/{id}/activate`
- `POST   /api/iam/admin/employees/{id}/deactivate`

When everything looks good:

```bash
git status

git add \
  src/backend/building-blocks/BankingSuite.BuildingBlocks.Application/Models/PagedResult.cs \
  src/backend/services/iam/BankingSuite.IAM.Domain/ApplicationUser.cs \
  src/backend/services/iam/BankingSuite.IAM.Application/Employees/**/* \
  src/backend/services/iam/BankingSuite.IAM.Api/Endpoints/Admin/Employees/**/*

git commit -m "Chapter 08: add IAM admin employee management endpoints"
git push --set-upstream origin part2-chapter08-iam-admin-employee-management
```

In the next section, we’ll build on this by adding **email confirmation**, **resend confirmation**, and later **password flows** and **2FA** on top of the same IAM service.
