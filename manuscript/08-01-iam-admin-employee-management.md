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

In **Visual Studio 2026**:

1. In **Solution Explorer**, locate the project:  
   `BankingSuite.BuildingBlocks.Application`.
2. If you don’t already have a `Models` folder:
   - Right-click **BankingSuite.BuildingBlocks.Application**  
     → **Add** → **New Folder…**  
     → name it `Models`.
3. Right-click the `Models` folder  
   → **Add** → **Class…**  
   → name it `PagedResult.cs`.
4. Replace the contents of `PagedResult.cs` with:

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

In IAM (and other services) we will simply add:

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
```

whenever we need paged results.

---

### 8.1.3 Extending `ApplicationUser` for Admin Management

Our IAM service is **single-tenant** for a fictitious Alvor Bank, so one bank, one IAM. We still want richer user information and explicit activation state.

In **Visual Studio 2026**, open the IAM Domain project:

1. In **Solution Explorer**, locate:  
   `BankingSuite.IamService.Domain`.
2. Find and open the `ApplicationUser` class file:  
   `ApplicationUser.cs` (path may look like  
   `src/backend/services/iam/BankingSuite.IamService.Domain/ApplicationUser.cs`).

Make sure it looks like this (or equivalent):

```csharp
using Microsoft.AspNetCore.Identity;

namespace BankingSuite.IamService.Domain;

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

In **Visual Studio 2026**:

1. In **Solution Explorer**, locate the project:  
   `BankingSuite.IamService.Application`.
2. If you don’t already have an `Employees` folder:
   - Right-click **BankingSuite.IamService.Application**  
     → **Add** → **New Folder…**  
     → name it `Employees`.
3. Under `Employees`, create a `Dtos` folder:
   - Right-click the `Employees` folder  
     → **Add** → **New Folder…**  
     → name it `Dtos`.
4. Right-click the `Dtos` folder  
   → **Add** → **Class…**  
   → name it `EmployeeDtos.cs`.
5. Replace the contents of `EmployeeDtos.cs` with:

```csharp
namespace BankingSuite.IamService.Application.Employees.Dtos;

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

In **Visual Studio 2026**, inside `BankingSuite.IamService.Application`:

1. Ensure you have the `Employees` folder (from the previous step).
2. Under `Employees`, create a `Queries` folder:
   - Right-click `Employees`  
     → **Add** → **New Folder…**  
     → name it `Queries`.
3. Under `Employees`, create a `Commands` folder:
   - Right-click `Employees`  
     → **Add** → **New Folder…**  
     → name it `Commands`.

Now add the individual classes.

#### ListEmployeesQuery

1. Right-click the `Queries` folder  
   → **Add** → **Class…**  
   → name it `ListEmployeesQuery.cs`.
2. Replace the contents of `ListEmployeesQuery.cs` with:

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
using BankingSuite.IamService.Application.Employees.Dtos;
using MediatR;

namespace BankingSuite.IamService.Application.Employees.Queries;

public sealed record ListEmployeesQuery(
    int PageNumber,
    int PageSize,
    string? Search,
    bool IncludeInactive
) : IRequest<PagedResult<EmployeeSummaryDto>>;
```

#### GetEmployeeDetailsQuery

1. Right-click the `Queries` folder again  
   → **Add** → **Class…**  
   → name it `GetEmployeeDetailsQuery.cs`.
2. Replace the contents with:

```csharp
using BankingSuite.IamService.Application.Employees.Dtos;
using MediatR;

namespace BankingSuite.IamService.Application.Employees.Queries;

public sealed record GetEmployeeDetailsQuery(Guid EmployeeId)
    : IRequest<EmployeeDetailsDto?>;
```

#### UpdateEmployeeCommand

1. Right-click the `Commands` folder  
   → **Add** → **Class…**  
   → name it `UpdateEmployeeCommand.cs`.
2. Replace the contents with:

```csharp
using MediatR;

namespace BankingSuite.IamService.Application.Employees.Commands;

public sealed record UpdateEmployeeCommand(
    Guid EmployeeId,
    string? FullName,
    string? PhoneNumber,
    string[] Roles
) : IRequest;
```

#### Activate / Deactivate Employee Commands

1. Right-click the `Commands` folder  
   → **Add** → **Class…**  
   → name it `EmployeeActivationCommands.cs`.
2. Replace the contents with:

```csharp
using MediatR;

namespace BankingSuite.IamService.Application.Employees.Commands;

public sealed record ActivateEmployeeCommand(Guid EmployeeId) : IRequest;

public sealed record DeactivateEmployeeCommand(Guid EmployeeId) : IRequest;
```

These records capture the **intent** of each operation and act as the boundary between HTTP and business logic.

---

### 8.1.6 Handlers: Identity + EF Core + Modern C# (Grouped by Use Case)

Just like we did for the login flow (`Auth/Commands/Login`), we’ll keep each
**command/query and its handler together in the same folder**. This makes it easy
to navigate per use case and keeps the Application layer consistent.

Our structure under `BankingSuite.IamService.Application` will look like this:

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

In **Visual Studio 2026**:

1. Under `Employees` → `Queries`, create a folder:
   - Right-click the `Queries` folder  
     → **Add** → **New Folder…**  
     → name it `ListEmployees`.
2. Move `ListEmployeesQuery.cs` into this new `ListEmployees` folder (optional but recommended for consistency).
3. Right-click the `ListEmployees` folder  
   → **Add** → **Class…**  
   → name it `ListEmployeesQueryHandler.cs`.
4. Replace the contents of `ListEmployeesQueryHandler.cs` with:

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
using BankingSuite.IamService.Application.Employees.Dtos;
using BankingSuite.IamService.Application.Employees.Queries;
using BankingSuite.IamService.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IamService.Application.Employees.Queries.ListEmployees;

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

In **Visual Studio 2026**:

1. Under `Employees` → `Queries`, create another folder:
   - Right-click the `Queries` folder  
     → **Add** → **New Folder…**  
     → name it `GetEmployeeDetails`.
2. Move `GetEmployeeDetailsQuery.cs` into this `GetEmployeeDetails` folder.
3. Right-click the `GetEmployeeDetails` folder  
   → **Add** → **Class…**  
   → name it `GetEmployeeDetailsQueryHandler.cs`.
4. Replace the contents of `GetEmployeeDetailsQueryHandler.cs` with:

```csharp
using BankingSuite.IamService.Application.Employees.Dtos;
using BankingSuite.IamService.Application.Employees.Queries;
using BankingSuite.IamService.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IamService.Application.Employees.Queries.GetEmployeeDetails;

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

In **Visual Studio 2026**:

1. Under `Employees` → `Commands`, create a folder:
   - Right-click the `Commands` folder  
     → **Add** → **New Folder…**  
     → name it `UpdateEmployee`.
2. Move `UpdateEmployeeCommand.cs` into this `UpdateEmployee` folder.
3. Right-click the `UpdateEmployee` folder  
   → **Add** → **Class…**  
   → name it `UpdateEmployeeCommandHandler.cs`.
4. Replace the contents with:

```csharp
using BankingSuite.IamService.Application.Employees.Commands;
using BankingSuite.IamService.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IamService.Application.Employees.Commands.UpdateEmployee;

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

In **Visual Studio 2026**:

1. Under `Employees` → `Commands`, create a folder:
   - Right-click the `Commands` folder  
     → **Add** → **New Folder…**  
     → name it `ActivateEmployee`.
2. Move `ActivateEmployeeCommand` (inside `EmployeeActivationCommands.cs`) into its own file if you prefer, or leave the commands file as-is and just place the handler here.
3. Right-click the `ActivateEmployee` folder  
   → **Add** → **Class…**  
   → name it `ActivateEmployeeCommandHandler.cs`.
4. Replace the contents of `ActivateEmployeeCommandHandler.cs` with:

```csharp
using BankingSuite.IamService.Application.Employees.Commands;
using BankingSuite.IamService.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IamService.Application.Employees.Commands.ActivateEmployee;

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

5. Under `Employees` → `Commands`, create another folder:
   - Right-click the `Commands` folder  
     → **Add** → **New Folder…**  
     → name it `DeactivateEmployee`.
6. Right-click the `DeactivateEmployee` folder  
   → **Add** → **Class…**  
   → name it `DeactivateEmployeeCommandHandler.cs`.
7. Replace the contents with:

```csharp
using BankingSuite.IamService.Application.Employees.Commands;
using BankingSuite.IamService.Domain;
using MediatR;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace BankingSuite.IamService.Application.Employees.Commands.DeactivateEmployee;

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

In **Visual Studio 2026**, inside the IAM API project:

1. In **Solution Explorer**, locate the project:  
   `BankingSuite.IamService.Api`.
2. Ensure you have an `Endpoints` folder. If not:
   - Right-click `BankingSuite.IamService.Api`  
     → **Add** → **New Folder…**  
     → name it `Endpoints`.
3. Under `Endpoints`, create an `Admin` folder:
   - Right-click `Endpoints`  
     → **Add** → **New Folder…**  
     → name it `Admin`.
4. Under `Admin`, create an `Employees` folder:
   - Right-click `Admin`  
     → **Add** → **New Folder…**  
     → name it `Employees`.

We’ll add:

- A file for the admin employee request DTOs.
- One file per endpoint.

#### Admin Employee Request DTOs

1. Right-click the `Employees` folder  
   → **Add** → **Class…**  
   → name it `AdminEmployeeRequests.cs`.
2. Replace the contents with:

```csharp
namespace BankingSuite.IamService.Api.Endpoints.Admin.Employees;

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

#### ListEmployeesEndpoint

1. Right-click the `Employees` folder  
   → **Add** → **Class…**  
   → name it `ListEmployeesEndpoint.cs`.
2. Replace the contents with:

```csharp
using BankingSuite.BuildingBlocks.Application.Models;
using BankingSuite.IamService.Application.Employees.Dtos;
using BankingSuite.IamService.Application.Employees.Queries;
using MediatR;

namespace BankingSuite.IamService.Api.Endpoints.Admin.Employees;

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

#### GetEmployeeEndpoint

1. Right-click the `Employees` folder  
   → **Add** → **Class…**  
   → name it `GetEmployeeEndpoint.cs`.
2. Replace the contents with:

```csharp
using BankingSuite.IamService.Application.Employees.Dtos;
using BankingSuite.IamService.Application.Employees.Queries;
using MediatR;

namespace BankingSuite.IamService.Api.Endpoints.Admin.Employees;

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

#### UpdateEmployeeEndpoint

1. Right-click the `Employees` folder  
   → **Add** → **Class…**  
   → name it `UpdateEmployeeEndpoint.cs`.
2. Replace the contents with:

```csharp
using BankingSuite.IamService.Application.Employees.Commands;
using MediatR;

namespace BankingSuite.IamService.Api.Endpoints.Admin.Employees;

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

#### Activate / Deactivate Employee Endpoints

1. Right-click the `Employees` folder  
   → **Add** → **Class…**  
   → name it `EmployeeStatusEndpoints.cs`.
2. Replace the contents with:

```csharp
using BankingSuite.IamService.Application.Employees.Commands;
using MediatR;

namespace BankingSuite.IamService.Api.Endpoints.Admin.Employees;

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

### 8.1.8 Update Database: Migration & Seed Initial IAM Data

Before we hit the new admin endpoints, we want the database to be in a **known good state**:

- The `AspNetUsers` table includes our new fields (`FullName`, `IsActive`, `LastLoginAt`).
- Default IAM roles exist (`Employee`, `IamAdmin`, `SuperAdmin`).
- A default **admin employee** exists so we can log into the admin UI later.

We’ll:

1. Add an EF Core migration for the new `ApplicationUser` fields.
2. Create a **database seeder** that ensures roles and a default admin user are present.
3. Hook the seeder into `Program.cs`.

> In the next section (originally 8.1.8, now 8.1.9) we’ll run a quick local test, then commit and push.

---

#### 8.1.8.1 Add EF Core Migration for IamService

We assume:

- `IamDbContext` lives in the **Infrastructure** project  
  `BankingSuite.IamService.Infrastructure`
- The API project is `BankingSuite.IamService.Api`.

From the repo root:

```bash
cd src/backend

dotnet ef migrations add AddAdminEmployeeFieldsToApplicationUser \
  -p services/iamservice/BankingSuite.IamService.Infrastructure/BankingSuite.IamService.Infrastructure.csproj \
  -s services/iamservice/BankingSuite.IamService.Api/BankingSuite.IamService.Api.csproj
```

This will generate a new migration in:

- `src/backend/services/iamservice/BankingSuite.IamService.Infrastructure/Migrations/`

> If you prefer Visual Studio 2026:  
> **Tools → NuGet Package Manager → Package Manager Console** and run the same `dotnet ef` command from the `src/backend` folder.

Don’t worry about the exact auto-generated migration code; the important thing is that it adds/updates the columns for:

- `FullName`
- `IsActive`
- `LastLoginAt`

We’ll let the seeder handle **roles and users** at runtime.

---

#### 8.1.8.2 Create IamService Database Seeder (Roles + Admin User)

We’ll add a small **Infrastructure** helper that:

- Applies migrations on startup.
- Ensures the default roles exist.
- Creates a default admin employee if needed and assigns roles.

In **Visual Studio 2026**, inside the IamService Infrastructure project:

1. In **Solution Explorer**, locate:  
   `src/backend/services/iamservice/BankingSuite.IamService.Infrastructure`.
2. Expand the `Persistence` folder (where `IamDbContext` lives).  
   If it doesn’t exist, right-click the project → **Add** → **New Folder…** → name it `Persistence`.
3. Right-click the `Persistence` folder → **Add** → **Class…**  
   Name it: `IamDbContextSeed.cs`.

Replace the contents with:

```csharp
using BankingSuite.IamService.Domain;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;

namespace BankingSuite.IamService.Infrastructure.Persistence;

public static class IamDbContextSeed
{
    public static async Task SeedAsync(IServiceProvider services)
    {
        using var scope = services.CreateScope();

        var context     = scope.ServiceProvider.GetRequiredService<IamDbContext>();
        var roleManager = scope.ServiceProvider.GetRequiredService<RoleManager<ApplicationRole>>();
        var userManager = scope.ServiceProvider.GetRequiredService<UserManager<ApplicationUser>>();

        // Apply pending migrations (safe to call multiple times)
        await context.Database.MigrateAsync();

        string[] roles =
        [
            "Employee",
            "IamAdmin",
            "SuperAdmin"
        ];

        foreach (var roleName in roles)
        {
            if (!await roleManager.RoleExistsAsync(roleName))
            {
                await roleManager.CreateAsync(new ApplicationRole { Name = roleName });
            }
        }

        const string adminEmail    = "admin@alvorbank.test";
        const string adminPassword = "Admin123!";

        var admin = await userManager.FindByEmailAsync(adminEmail);

        if (admin is null)
        {
            admin = new ApplicationUser
            {
                UserName         = adminEmail,
                Email            = adminEmail,
                FullName         = "Alvor Bank IAM Admin",
                EmailConfirmed   = true,
                IsActive         = true,
                TwoFactorEnabled = false
            };

            var createResult = await userManager.CreateAsync(admin, adminPassword);

            if (!createResult.Succeeded)
            {
                var errors = string.Join("; ", createResult.Errors.Select(e => e.Description));
                throw new InvalidOperationException($"Failed to create default admin user: {errors}");
            }
        }

        var desiredRoles = new[] { "Employee", "IamAdmin", "SuperAdmin" };
        var currentRoles = await userManager.GetRolesAsync(admin);
        var missingRoles = desiredRoles.Except(currentRoles).ToArray();

        if (missingRoles.Length > 0)
        {
            var addRolesResult = await userManager.AddToRolesAsync(admin, missingRoles);

            if (!addRolesResult.Succeeded)
            {
                var errors = string.Join("; ", addRolesResult.Errors.Select(e => e.Description));
                throw new InvalidOperationException($"Failed to assign roles to admin user: {errors}");
            }
        }
    }
}
```

Key points:

- We call `Database.MigrateAsync()` so the app always applies migrations on startup.
- We create three roles: `Employee`, `IamAdmin`, `SuperAdmin`.
- We create a default IAM admin user:
  - Email: `admin@alvorbank.test`
  - Password: `Admin123!` (dev/demo only – in production you’d rotate this)
- We assign **all three roles** to the admin user.

---

#### 8.1.8.3 Hook the Seeder Into Program.cs

Finally, we call the seeder during IamService API startup.

In **Visual Studio 2026**, inside the IamService API project:

1. Locate `src/backend/services/iam/BankingSuite.IamService.Api/Program.cs`.
2. At the top of the file, add:

   ```csharp
   using BankingSuite.IamService.Infrastructure.Persistence;
   ```

3. After `var app = builder.Build();` (and before `app.Run();`), call the seeder:

   ```csharp
   var app = builder.Build();

   // Apply migrations and seed IAM data (roles + default admin)
   await IamDbContextSeed.SeedAsync(app.Services);

   // existing middleware/endpoint configuration
   app.UseAuthentication();
   app.UseAuthorization();

   app.UseFastEndpoints();
   app.UseOpenApi();
   app.UseSwaggerUi3();

   app.Run();
   ```

> Because we’re using top-level statements in .NET 10, `Program` is already an `async Task Main`, so `await IamDbContextSeed.SeedAsync(...)` is valid.

Now, any time you run the IamService API locally:

- The latest migrations are applied.
- The roles exist.
- The default admin user exists and has the correct roles.

In the **next section** (8.1.9), we’ll do a **quick local test**, then commit these changes and push the `part2-chapter08-iam-admin-employee-management` branch.

---

### 8.1.9 Quick Local Test, Commit and Push

Build and run the backend (using your usual dev compose), then hit the new endpoints via Swagger or a REST client:

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
  src/backend/services/iam/BankingSuite.IamService.Domain/ApplicationUser.cs \
  src/backend/services/iam/BankingSuite.IamService.Application/Employees/**/* \
  src/backend/services/iam/BankingSuite.IamService.Api/Endpoints/Admin/Employees/**/*

git commit -m "Chapter 08: add IAM admin employee management endpoints"
git push --set-upstream origin part2-chapter08-iam-admin-employee-management
```

In the next section, we’ll build on this by adding **email confirmation**, **resend confirmation**, and later **password flows** and **2FA** on top of the same IAM service.
