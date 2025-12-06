# Chapter 05 — Backend Solution & Clean Architecture Template (.NET 10)

In the previous chapters we:

- Defined the **vision** and **domain architecture** for Alvor Bank – Banking Suite.
- Created the `digital-banking-suite` repository and base folders.
- Set up **Docker** with **PostgreSQL** and **RabbitMQ** using `infra/docker-compose.dev.yml`.

Now we’ll finally create our **.NET 10 backend solution** and a reusable **Clean Architecture template** that every microservice will follow.

By the end of this chapter you will:

- Have a `BankingSuite.Backend.sln` solution under `src/backend/`.
- Have **BuildingBlocks** projects for shared abstractions: Domain, Application, Infrastructure.
- Have an **IAM service skeleton** with Domain, Application, Infrastructure and API projects.
- Be able to run a simple **health endpoint** from `BankingSuite.IamService.API` and see it respond.

> **IDE assumption:** All file and project steps use **Visual Studio 2026**.  
> If you’re using Rider, VS Code or another IDE, create equivalent projects and references using that environment or the `dotnet` CLI.

---

## 5.1 Creating the backend solution

Open **Visual Studio 2026** and make sure the folder `digital-banking-suite` is open (from Chapter 04).

### 5.1.1 Add a new solution

1. In **Solution Explorer**, right-click the root `digital-banking-suite` node.
2. Choose **Add → New Project…**.
3. Search for **“Blank Solution”**.
4. Name it:

   - **Project name:** `BankingSuite.Backend`
   - **Location:** `digital-banking-suite/src/backend`

5. Click **Create**.

Visual Studio will create `src/backend/BankingSuite.Backend.sln` and show an empty solution in Solution Explorer.

---

## 5.2 Creating BuildingBlocks projects

We’ll create three shared class libraries inside a `BuildingBlocks` folder:

- `BankingSuite.BuildingBlocks.Domain`
- `BankingSuite.BuildingBlocks.Application`
- `BankingSuite.BuildingBlocks.Infrastructure`

These will hold reusable patterns (entities, base classes, `Result` type, etc.) shared by all services.

### 5.2.1 Create the `BuildingBlocks` solution folder

1. In Solution Explorer, right-click the solution node **`BankingSuite.Backend`**.
2. Choose **Add → New Solution Folder**.
3. Name it **`BuildingBlocks`**.

### 5.2.2 `BankingSuite.BuildingBlocks.Domain`

1. Right-click the **`BuildingBlocks`** solution folder → **Add → New Project…**.
2. Choose **Class Library** (`C#`, `.NET`).
3. Name it **`BankingSuite.BuildingBlocks.Domain`**.
4. Confirm the **Framework** is **`.NET 10.0` (`net10.0`)**.
5. Click **Create**.

Delete the default `Class1.cs` file (we’ll add our own code).

### 5.2.3 `BankingSuite.BuildingBlocks.Application`

Repeat:

1. Right-click **`BuildingBlocks`** → **Add → New Project…**.
2. Choose **Class Library**.
3. Name it **`BankingSuite.BuildingBlocks.Application`**.
4. Target **`net10.0`**.
5. Create and delete the default `Class1.cs`.

Add a project reference from **Application** to **Domain**:

1. Right-click `BankingSuite.BuildingBlocks.Application` → **Add → Project Reference…**.
2. Tick `BankingSuite.BuildingBlocks.Domain`.
3. Click **OK**.

### 5.2.4 `BankingSuite.BuildingBlocks.Infrastructure`

1. Right-click **`BuildingBlocks`** → **Add → New Project…**.
2. Choose **Class Library**.
3. Name it **`BankingSuite.BuildingBlocks.Infrastructure`**.
4. Target **`net10.0`**.
5. Delete the default `Class1.cs`.

Add project references:

1. Right-click `BankingSuite.BuildingBlocks.Infrastructure` → **Add → Project Reference…**.
2. Tick:
   - `BankingSuite.BuildingBlocks.Domain`
   - `BankingSuite.BuildingBlocks.Application`
3. Click **OK**.

Now Build the solution:

```bash
dotnet build src/backend/BankingSuite.Backend.sln
```

It should succeed.

---

## 5.3 Adding minimal shared types to BuildingBlocks

We’ll add two simple but useful pieces to `BuildingBlocks.Domain`:

- A base **entity** type with an `Id`.
- A functional-style **`Result`** type that represents success/failure.

### 5.3.1 Base entity

In `BankingSuite.BuildingBlocks.Domain`:

1. Right-click the project → **Add → New Folder** → name it **`Abstractions`**.
2. Right-click the `Abstractions` folder → **Add → Class…** → name it **`Entity.cs`**.

Replace its content with:

```csharp
namespace BankingSuite.BuildingBlocks.Domain.Abstractions;

public abstract class Entity<TId>
{
    public TId Id { get; protected set; }

    protected Entity(TId id)
    {
        Id = id;
    }

    protected Entity() { }

    public override bool Equals(object? obj)
    {
        if (obj is not Entity<TId> other)
            return false;

        if (ReferenceEquals(this, other))
            return true;

        if (Equals(Id, default(TId)) || Equals(other.Id, default(TId)))
            return false;

        return Id!.Equals(other.Id);
    }

    public static bool operator ==(Entity<TId>? a, Entity<TId>? b)
    {
        if (a is null && b is null) return true;
        if (a is null || b is null) return false;
        return a.Equals(b);
    }

    public static bool operator !=(Entity<TId>? a, Entity<TId>? b) => !(a == b);

    public override int GetHashCode() => Id?.GetHashCode() ?? 0;
}
```

### 5.3.2 Result type

Still in `BankingSuite.BuildingBlocks.Domain/Abstractions`, add another class **`Result.cs`**:

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

We’ll use `Result` everywhere in the application and domain layers to avoid throwing exceptions for normal control flow.

Build again to ensure everything compiles:

```bash
dotnet build src/backend/BankingSuite.Backend.sln
```

---

## 5.4 Creating the IAM service skeleton

IAM will be our **first real microservice** and the template for others.  
We’ll create four projects under a `Services/IamService` solution folder:

- `BankingSuite.IamService.Domain`
- `BankingSuite.IamService.Application`
- `BankingSuite.IamService.Infrastructure`
- `BankingSuite.IamService.API` (the actual HTTP service)

### 5.4.1 Add the `Services` solution folder

1. In Solution Explorer, right-click the solution `BankingSuite.Backend`.
2. Choose **Add → New Solution Folder**.
3. Name it **`Services`**.

Inside `Services`, add another solution folder **`IamService`**.

### 5.4.2 IAM Domain project

1. Right-click **`IamService`** solution folder → **Add → New Project…**.
2. Choose **Class Library**.
3. Name it **`BankingSuite.IamService.Domain`**.
4. Target **`.NET 10.0`**.
5. Delete `Class1.cs`.

Add a reference to `BuildingBlocks.Domain`:

1. Right-click `BankingSuite.IamService.Domain` → **Add → Project Reference…**.
2. Tick `BankingSuite.BuildingBlocks.Domain`.
3. Click **OK**.

### 5.4.3 IAM Application project

1. Right-click **`IamService`** → **Add → New Project…**.
2. Choose **Class Library**.
3. Name it **`BankingSuite.IamService.Application`**.
4. Target **`net10.0`**.
5. Delete `Class1.cs`.

Add project references:

1. Right-click `BankingSuite.IamService.Application` → **Add → Project Reference…**.
2. Tick:
   - `BankingSuite.BuildingBlocks.Application`
   - `BankingSuite.BuildingBlocks.Domain`
   - `BankingSuite.IamService.Domain`
3. Click **OK**.

### 5.4.4 IAM Infrastructure project

1. Right-click **`IamService`** → **Add → New Project…**.
2. Choose **Class Library**.
3. Name it **`BankingSuite.IamService.Infrastructure`**.
4. Target **`net10.0`**.
5. Delete `Class1.cs`.

Add project references:

1. Right-click `BankingSuite.IamService.Infrastructure` → **Add → Project Reference…**.
2. Tick:
   - `BankingSuite.BuildingBlocks.Infrastructure`
   - `BankingSuite.BuildingBlocks.Application`
   - `BankingSuite.BuildingBlocks.Domain`
   - `BankingSuite.IamService.Application`
   - `BankingSuite.IamService.Domain`
3. Click **OK**.

We’ll later add NuGet packages (EF Core, Identity, MassTransit) to this project.

### 5.4.5 IAM API project (FastEndpoints)

1. Right-click **`IamService`** → **Add → New Project…**.
2. Choose **ASP.NET Core Empty** or **ASP.NET Core Web API** (we will strip it down).
3. Name it **`BankingSuite.IamService.API`**.
4. Target **`net10.0`**.
5. Ensure **Use controllers** is unchecked if you choose Web API template (we’ll use **FastEndpoints** / minimal hosting).

Add project references:

1. Right-click `BankingSuite.IamService.API` → **Add → Project Reference…**.
2. Tick:
   - `BankingSuite.IamService.Infrastructure`
   - `BankingSuite.IamService.Application`
   - `BankingSuite.IamService.Domain`
3. Click **OK**.

Next, add NuGet packages we’ll need for this chapter:

- `FastEndpoints` (for our endpoint framework)
- `FastEndpoints.Swagger` (for Swagger UI)

Right-click `BankingSuite.IamService.API` → **Manage NuGet Packages…** and install:

```text
FastEndpoints
FastEndpoints.Swagger
```

(We’ll add Identity, EF Core and others in Chapter 07 when we fully implement IAM.)

---

## 5.5 Wiring a minimal IAM API with a health endpoint

We’ll now change `Program.cs` in `BankingSuite.IamService.API` to:

- Use FastEndpoints.
- Expose a simple `/health` endpoint returning the service name.
- Be ready for Dependency Injection configuration later.

Open `Program.cs` and replace its content with:

```csharp
using FastEndpoints;

var builder = WebApplication.CreateBuilder(args);

// Add FastEndpoints
builder.Services.AddFastEndpoints();

// Add Swagger for FastEndpoints
builder.Services.AddSwaggerDoc(config =>
{
    config.Title = "Alvor Bank - IAM Service";
    config.Version = "v1";
});

var app = builder.Build();

app.UseFastEndpoints();

// Enable Swagger UI in development
if (app.Environment.IsDevelopment())
{
    app.UseOpenApi();
    app.UseSwaggerUi3(settings =>
    {
        settings.Path = "/swagger";
        settings.DocumentPath = "/swagger/v1/swagger.json";
    });
}

// Minimal health endpoint (no auth yet)
app.MapGet("/health", () => Results.Ok(new
{
    Service = "IamService",
    Status = "Healthy",
    TimestampUtc = DateTime.UtcNow
}));

app.Run();
```

### 5.5.1 Add a simple launch profile

Ensure the IAM API launches on a stable port (we’ll integrate it into Docker later).

Open `Properties/launchSettings.json` in the IAM API project and adjust the `applicationUrl` for the profile you use most (e.g. `http`):

```json
"applicationUrl": "http://localhost:5101"
```

Save the file.

---

## 5.6 Building and running the IAM API

From the repo root, build the backend solution:

```bash
dotnet build src/backend/BankingSuite.Backend.sln
```

To run the IAM API from the command line:

```bash
cd src/backend/Services/IamService/BankingSuite.IamService.API
dotnet run
```

Or from **Visual Studio 2026**:

1. Right-click `BankingSuite.IamService.API` → **Set as Startup Project**.
2. Press **F5** (Debug) or **Ctrl+F5** (Run without debugging).

Open a browser:

```text
http://localhost:5101/health
```

You should see JSON similar to:

```json
{
  "service": "IamService",
  "status": "Healthy",
  "timestampUtc": "2025-01-01T00:00:00Z"
}
```

Swagger UI should be available at:

```text
http://localhost:5101/swagger
```

At this point you have:

- Running .NET 10 IAM API skeleton.
- FastEndpoints + Swagger wired.
- BuildingBlocks projects ready to host shared abstractions.

---

## 5.7 Solution structure recap

Your `src/backend` folder and solution should now look like this:

```text
src/backend/
└─ BankingSuite.Backend.sln

   BuildingBlocks/
   ├─ BankingSuite.BuildingBlocks.Domain/
   ├─ BankingSuite.BuildingBlocks.Application/
   └─ BankingSuite.BuildingBlocks.Infrastructure/

   Services/
   └─ IamService/
      ├─ BankingSuite.IamService.Domain/
      ├─ BankingSuite.IamService.Application/
      ├─ BankingSuite.IamService.Infrastructure/
      └─ BankingSuite.IamService.API/
```

We will reuse this pattern for **CustomerService**, **AccountService**, **TransactionService** and **NotificationService**.

---

## 5.8 Committing and tagging Chapter 05

Follow our usual workflow.

From the repo root:

```bash
git status
git add src/backend
git commit -m "ch05: add backend solution, BuildingBlocks and IAM service skeleton"
```

If you are working in a feature branch (recommended), push it and create a PR into `develop`.  
After merging, tag the state for this chapter:

```bash
git tag -a chapter-05-backend-solution -m "Chapter 05: backend solution and IAM skeleton"
git push origin chapter-05-backend-solution
```

---

## 5.9 What’s next

We now have:

- A **backend solution**.
- Shared **BuildingBlocks** projects.
- An **IAM service skeleton** with a health endpoint.

In the next chapter we will:

- Introduce **backend testing**: xUnit projects, basic unit tests and integration tests for the IAM API health endpoint.
- Integrate **code coverage** with Coverlet and ReportGenerator.
- Create a first **GitHub Actions** workflow that restores, builds, tests and generates coverage reports for the backend.

From that point on, every backend chapter will be built on top of this **test-first, CI-aware** foundation.
