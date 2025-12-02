# Chapter 5 — Backend Solution Structure & Shared Building Blocks

In the previous chapter, we containerized our first placeholder service (`AccountService.Api`) and wired it into Docker and Docker Compose.

In this chapter, we’ll give the backend a **proper home**:

- Create a **backend solution file** (`.sln`) under `src/backend`.
- Add `AccountService.Api` to this solution.
- Introduce a **shared building blocks project** for common patterns (entities, value objects, Result type).
- Wire everything into CI so GitHub Actions can build the backend solution.

By the end of this chapter, you will have:

- A clear **solution structure** for the backend.
- A **`BuildingBlocks`** library ready to be reused by future microservices.
- CI updated to restore and build your backend solution as part of every PR.

---

## 5.1 Git Setup for This Chapter (Feature Branch)

As before, this chapter’s work lives in its own feature branch.

> **Each build chapter gets its own feature branch.**  
> We only merge back to `develop` after the chapter is complete and CI is green.

For this chapter we’ll use:

- Branch name: `feature/ch05-backend-structure`

From your local clone:

```bash
cd digital-banking-suite

# Make sure you're on develop and up to date
git checkout develop
git pull origin develop

# Create a new branch for this chapter
git checkout -b feature/ch05-backend-structure
```

All changes for this chapter (solution files, new projects, CI updates, docs) will happen on this branch.

T> Keeping a separate branch per chapter makes it easy to review your work, roll back if needed, and see CI runs per chapter in GitHub.

---

## 5.2 What We’re Building in This Chapter

Right now, your backend is just:

- A folder structure created in Chapter 3.
- A placeholder `AccountService.Api` project created in Chapter 4.
- Docker and docker-compose configuration.

We’re missing:

- A **solution file** that ties backend projects together.
- A **shared library** for reusable building blocks.
- A CI step that **builds** the backend, not only restores dependencies.

In this chapter, we’ll:

1. Create `BankingSuite.Backend.sln` under `src/backend`.
2. Add `AccountService.Api` to that solution.
3. Create a `BuildingBlocks` class library for shared building blocks.
4. Implement simple, reusable base types:
   - `Entity`
   - `ValueObject`
   - `Result` (for returning success/failure)
5. Reference `BuildingBlocks` from `AccountService.Api`.
6. Update GitHub Actions to **build the solution**.

I> These building blocks are intentionally **minimal**. We’ll evolve them as we introduce real domain models and domain events later in the book.

---

## 5.3 Creating the Backend Solution

We want a single solution file that will eventually include:

- AccountService
- IAMService
- CustomerService
- TransactionService
- NotificationService
- Shared libraries (e.g., BuildingBlocks)

From the repo root:

```bash
cd digital-banking-suite

# Go to backend folder
cd src/backend

# Create a new empty solution
dotnet new sln -n BankingSuite.Backend
```

This will create:

- `src/backend/BankingSuite.Backend.sln`

At this point, your backend structure looks roughly like:

```text
src/
  backend/
    BankingSuite.Backend.sln
    AccountService/
      AccountService.Api/
        AccountService.Api.csproj
        Dockerfile
        ...
```

T> We keep **one solution per logical layer**. Here it’s the backend solution. Later, you might have a separate solution for tests, or a combined solution for end-to-end builds if you prefer.

---

## 5.4 Adding AccountService.Api to the Solution

Next, we add the existing `AccountService.Api` project to `BankingSuite.Backend.sln`.

Still inside `src/backend`:

```bash
cd digital-banking-suite/src/backend

dotnet sln BankingSuite.Backend.sln add \
  ./AccountService/AccountService.Api/AccountService.Api.csproj
```

You can verify the solution contents with:

```bash
dotnet sln BankingSuite.Backend.sln list
```

You should see something like:

```text
Projects in solution 'BankingSuite.Backend.sln':
    AccountService/AccountService.Api/AccountService.Api.csproj
```

Now you can build the backend solution locally:

```bash
dotnet build BankingSuite.Backend.sln
```

If everything is set up correctly, this should complete successfully.

---

## 5.5 Creating the BuildingBlocks Project

We’ll now add a **shared library** to hold building blocks that multiple services can reuse.

From `src/backend`:

```bash
cd digital-banking-suite/src/backend

# Create the BuildingBlocks class library
dotnet new classlib \
  -n BuildingBlocks \
  -o BuildingBlocks
```

This creates:

```text
src/backend/
  BankingSuite.Backend.sln
  BuildingBlocks/
    BuildingBlocks.csproj
    Class1.cs
  AccountService/
    AccountService.Api/
      AccountService.Api.csproj
```

Now add `BuildingBlocks` to the solution:

```bash
dotnet sln BankingSuite.Backend.sln add ./BuildingBlocks/BuildingBlocks.csproj
```

We don’t need `Class1.cs`, so remove it:

```bash
rm BuildingBlocks/Class1.cs
```

---

## 5.6 Implementing Shared Building Blocks

We’ll add three simple, reusable base types:

- `Entity` – base class for entities with an `Id`.
- `ValueObject` – base class for value objects with proper equality semantics.
- `Result` – a simple success/failure wrapper with an optional error message.

These are intentionally small but powerful. They’ll be used heavily when we start modeling real domain logic.

### 5.6.1 The `Entity` Base Class

Create `src/backend/BuildingBlocks/Entity.cs`:

```csharp
namespace BuildingBlocks;

public abstract class Entity
{
    public Guid Id { get; protected set; }

    protected Entity()
    {
        Id = Guid.NewGuid();
    }

    protected Entity(Guid id)
    {
        Id = id == Guid.Empty ? Guid.NewGuid() : id;
    }

    public override bool Equals(object? obj)
    {
        if (obj is not Entity other)
            return false;

        if (ReferenceEquals(this, other))
            return true;

        if (GetType() != other.GetType())
            return false;

        return Id.Equals(other.Id);
    }

    public static bool operator ==(Entity? a, Entity? b)
    {
        if (a is null && b is null)
            return true;

        if (a is null || b is null)
            return false;

        return a.Equals(b);
    }

    public static bool operator !=(Entity? a, Entity? b) => !(a == b);

    public override int GetHashCode() => Id.GetHashCode();
}
```

T> Later, specific aggregates like `Account`, `Customer`, and `Transaction` will inherit from `Entity`.

### 5.6.2 The `ValueObject` Base Class

Create `src/backend/BuildingBlocks/ValueObject.cs`:

```csharp
namespace BuildingBlocks;

public abstract class ValueObject
{
    protected abstract IEnumerable<object?> GetEqualityComponents();

    public override bool Equals(object? obj)
    {
        if (obj is null || obj.GetType() != GetType())
            return false;

        var other = (ValueObject)obj;

        return GetEqualityComponents()
            .SequenceEqual(other.GetEqualityComponents());
    }

    public override int GetHashCode()
    {
        return GetEqualityComponents()
            .Aggregate(1, (current, obj) =>
            {
                unchecked
                {
                    return current * 23 + (obj?.GetHashCode() ?? 0);
                }
            });
    }

    public static bool operator ==(ValueObject? a, ValueObject? b)
    {
        if (a is null && b is null)
            return true;

        if (a is null || b is null)
            return false;

        return a.Equals(b);
    }

    public static bool operator !=(ValueObject? a, ValueObject? b) => !(a == b);
}
```

T> A `Money` type, or an `Address` type, is a good example of a value object we’ll create later.

### 5.6.3 The `Result` Type

Create `src/backend/BuildingBlocks/Result.cs`:

```csharp
namespace BuildingBlocks;

public class Result
{
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public string? Error { get; }

    protected Result(bool isSuccess, string? error)
    {
        if (isSuccess && error is not null)
            throw new InvalidOperationException("Successful result cannot have an error.");

        if (!isSuccess && string.IsNullOrWhiteSpace(error))
            throw new InvalidOperationException("Failure result must have an error message.");

        IsSuccess = isSuccess;
        Error = error;
    }

    public static Result Success() => new Result(true, null);

    public static Result Failure(string error) => new Result(false, error);
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
        new Result<T>(true, value, null);

    public static new Result<T> Failure(string error) =>
        new Result<T>(false, default, error);
}
```

This will allow us to write domain logic like:

```csharp
Result<Account> result = Account.Open(...);

if (result.IsFailure)
{
    // handle error
}
```

Later we’ll build domain services and application handlers that return `Result`/`Result<T>` to indicate success or failure in a consistent way.

---

## 5.7 Referencing BuildingBlocks from AccountService.Api

Now we want `AccountService.Api` to be able to use these building blocks.

From `src/backend`:

```bash
cd digital-banking-suite/src/backend

dotnet add ./AccountService/AccountService.Api/AccountService.Api.csproj \
  reference ./BuildingBlocks/BuildingBlocks.csproj
```

You can open `AccountService.Api.csproj` and verify a `<ProjectReference>` to `BuildingBlocks.csproj` was added.

Let’s also confirm everything builds:

```bash
dotnet build BankingSuite.Backend.sln
```

If there are any compile errors in our new `BuildingBlocks` code, fix them now. You shouldn’t see any if you copied the code as-is.

I> We’re not using `Entity`, `ValueObject`, or `Result` inside `AccountService.Api` yet. That’s okay. They’ll become essential when we introduce real domain models in upcoming chapters (for IAM, Customers, Accounts, and Transactions).

---

## 5.8 Updating the CI Pipeline to Build the Backend Solution

In Chapter 4, we fixed the CI restore step to restore all backend `.csproj` files. Now that we have a **backend solution file**, we want CI to:

1. Restore backend projects.
2. **Build** the backend solution.

Open `.github/workflows/ci.yml` and:

1. Keep the existing “restore backend projects” step from Chapter 4.
2. Add a new step **after** restore to build the backend solution if it exists.

Add this step:

```yaml
- name: Build backend solution (if present)
  run: |
    if [ -f "./src/backend/BankingSuite.Backend.sln" ]; then
      echo "Building backend solution..."
      dotnet build ./src/backend/BankingSuite.Backend.sln --configuration Release
    else
      echo "No backend solution found yet. Skipping build."
    fi
```

From now on, every PR will:

- Restore any backend project(s) under `src/backend`.
- Build `BankingSuite.Backend.sln` if it exists.

T> As we add more backend projects (IAM, Customer, Transaction, Notification), we only need to add them to the solution. CI will automatically build all of them as part of the solution build.

---

## 5.9 Local Sanity Check (Before Committing)

Before we start committing, let’s quickly run through the key commands locally.

From the repo root:

```bash
cd digital-banking-suite

# Restore all backend projects (similar to CI)
projects=$(find ./src/backend -name "*.csproj")
for proj in $projects; do
  echo "Restoring $proj"
  dotnet restore "$proj"
done

# Build the backend solution
dotnet build ./src/backend/BankingSuite.Backend.sln --configuration Release
```

If both restore and build succeed locally:

- Your solution is wired correctly.
- `BuildingBlocks` compiles.
- `AccountService.Api` references `BuildingBlocks` without issues.

---

## 5.10 Wrap-up: PR, CI & Merge to `develop`

At this point you should have, on the branch  
`feature/ch05-backend-structure`:

- `BankingSuite.Backend.sln` under `src/backend`.
- A new `BuildingBlocks` class library with:
  - `Entity`
  - `ValueObject`
  - `Result` / `Result<T>`
- `AccountService.Api` added to the solution and referencing `BuildingBlocks`.
- CI updated to build the backend solution.

Now we finish the Git workflow for the chapter.

### 5.10.1 Committing Your Changes

Stage and commit your work in logical chunks. For example:

```bash
git add src/backend/BankingSuite.Backend.sln
git add src/backend/AccountService/AccountService.Api/AccountService.Api.csproj
git commit -m "chore(backend): add backend solution and include AccountService.Api"

git add src/backend/BuildingBlocks/*
git add src/backend/BuildingBlocks/BuildingBlocks.csproj
git commit -m "feat(building-blocks): add Entity, ValueObject and Result primitives"

git add .github/workflows/ci.yml
git commit -m "chore(ci): build backend solution in pipeline"
```

T> As before, you don’t need to match these messages exactly; just keep them **clear and scoped**. Avoid “misc changes” or “stuff”.

### 5.10.2 Pushing and Opening a Pull Request

Push the branch to GitHub:

```bash
git push -u origin feature/ch05-backend-structure
```

Then, in GitHub:

1. Open a Pull Request from  
   `feature/ch05-backend-structure` → `develop`
2. Give it a clear title, for example:  
   **"Chapter 5 – Backend Solution Structure & Shared Building Blocks"**
3. Briefly describe what this PR adds:
   - Backend solution (`BankingSuite.Backend.sln`)
   - `BuildingBlocks` library with base types
   - CI update to build backend solution

### 5.10.3 Watching the CI Pipeline

When you open the PR, the CI workflow (`.github/workflows/ci.yml`) should run automatically.

At this stage, CI will:

- Restore all `.csproj` files under `src/backend`.
- Build `BankingSuite.Backend.sln` if it exists.

Later chapters will extend CI to:

- Run unit tests (`dotnet test`).
- Build Docker images.
- Potentially run linters and static analysis.

W> Do not merge into `develop` if the pipeline is red. Fix the issue (build errors, missing files, etc.), push again, and wait for CI to pass.

### 5.10.4 Merging to `develop`

Once the pipeline is green:

1. Merge the PR into `develop`.
2. Optionally delete the feature branch:

   - in GitHub (Delete branch button), and/or
   - locally:

   ```bash
   git checkout develop
   git pull origin develop
   git branch -d feature/ch05-backend-structure
   ```

At this point:

- `develop` contains all Chapter 5 changes.
- CI has already validated the backend structure and building blocks.
- You are ready to start the next chapter from a clean, green baseline.

---

## 5.11 Summary & What’s Next

In this chapter, we:

- Created a **backend solution**: `src/backend/BankingSuite.Backend.sln`.
- Added the existing `AccountService.Api` project to the backend solution.
- Introduced a reusable **`BuildingBlocks`** library containing:
  - `Entity` base class
  - `ValueObject` base class
  - `Result` / `Result<T>` types for success/failure modeling
- Updated `AccountService.Api` to reference `BuildingBlocks`.
- Updated CI to **build the backend solution** on every PR.

These are foundational steps for a maintainable, domain-driven backend. All future services (IAM, Customer, Transaction, Notification) will:

- Join this same backend solution.
- Reuse the same building blocks.

In the next chapter, we will:

- Introduce **testing foundations** for the backend:
  - Create a test project (e.g., `AccountService.UnitTests` or `BuildingBlocks.Tests`).
  - Add the first meaningful tests (for our building blocks or basic API behavior).
  - Wire `dotnet test` into CI to ensure every PR runs tests.

From now on, the backend isn’t just “some projects in a folder”.  
It’s a structured solution with shared building blocks, built and validated by CI on every change.
