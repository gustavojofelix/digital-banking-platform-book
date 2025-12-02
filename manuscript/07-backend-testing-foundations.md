# Chapter 6 — Backend Testing Foundations

So far in Part II, we:

- Containerized our first placeholder backend service (`AccountService.Api`) with Docker and docker-compose.
- Created a **backend solution** (`BankingSuite.Backend.sln`) and a **shared `BuildingBlocks` library** containing `Entity`, `ValueObject`, and `Result`.

In this chapter, we add the next critical piece: **automated tests**.

By the end of this chapter, you will:

- Understand the basic **testing strategy** for the Banking Suite backend.
- Create a **test project** for the backend (`BuildingBlocks.UnitTests`).
- Add the first **meaningful tests** for `Result` and `ValueObject`.
- Wire backend tests into **GitHub Actions** so every PR runs tests.

---

## 6.1 Git Setup for This Chapter (Feature Branch)

As always, this chapter’s work lives in its own feature branch.

> **Each build chapter gets its own feature branch.**  
> We only merge back to `develop` after the chapter is complete and CI is green.

For this chapter we’ll use:

- Branch name: `feature/ch06-backend-testing-foundations`

From your local clone:

```bash
cd digital-banking-suite

# Make sure you're on develop and up to date
git checkout develop
git pull origin develop

# Create a new branch for this chapter
git checkout -b feature/ch06-backend-testing-foundations
```

All changes for this chapter (test project, test code, CI updates, docs) will happen on this branch.

T> This chapter is intentionally small but important. You’ll feel its impact every time CI catches a bug before it hits `develop`.

---

## 6.2 Testing Strategy for the Banking Suite

We’re building a **real** banking platform, not a toy project. That means we care about:

- **Fast feedback** – tests should run on every PR.
- **Clear layers** – unit tests for core domain and building blocks, integration tests for services.
- **Confidence** – catching regressions before they reach production.

We’ll use a layered approach:

- **Unit tests**:
  - Pure C# tests.
  - No network, no Docker, no database.
  - Focused on domain logic and building blocks (`Entity`, `ValueObject`, `Result`, aggregates, domain services).
- **Integration tests** (later chapters):
  - Exercising microservices via HTTP or messaging.
  - Using test containers or docker-compose for dependencies (PostgreSQL, RabbitMQ).

In this chapter, we start with:

- **Unit tests for `BuildingBlocks`**:
  - `Result` – ensure success/failure behaves correctly.
  - `ValueObject` – equality semantics.

I> Starting with shared building blocks is a great way to ensure the foundation is solid before we build business-critical logic on top of it.

---

## 6.3 Creating the Backend Test Project

We’ll begin by creating a dedicated **xUnit** test project for the `BuildingBlocks` library.

From the backend folder:

```bash
cd digital-banking-suite/src/backend

# Create xUnit test project for BuildingBlocks
dotnet new xunit \
  -n BuildingBlocks.UnitTests \
  -o BuildingBlocks.UnitTests
```

This creates:

```text
src/backend/
  BankingSuite.Backend.sln
  BuildingBlocks/
    BuildingBlocks.csproj
    Entity.cs
    ValueObject.cs
    Result.cs
  BuildingBlocks.UnitTests/
    BuildingBlocks.UnitTests.csproj
    UnitTest1.cs
  AccountService/
    AccountService.Api/
      AccountService.Api.csproj
```

Now add the test project to the backend solution:

```bash
dotnet sln BankingSuite.Backend.sln add ./BuildingBlocks.UnitTests/BuildingBlocks.UnitTests.csproj
```

We also need the test project to reference the `BuildingBlocks` project:

```bash
dotnet add ./BuildingBlocks.UnitTests/BuildingBlocks.UnitTests.csproj \
  reference ./BuildingBlocks/BuildingBlocks.csproj
```

You can now build the solution to verify everything compiles:

```bash
dotnet build BankingSuite.Backend.sln
```

If this succeeds, the test project and building blocks are wired correctly.

T> For now, we’re only adding tests for `BuildingBlocks`. Later, we’ll create more test projects like `AccountService.UnitTests`, `IAMService.UnitTests`, etc.

---

## 6.4 Cleaning Up the Default Test Class

The `dotnet new xunit` template creates a default `UnitTest1.cs`. Let’s remove it and create our own tests.

From `src/backend`:

```bash
cd digital-banking-suite/src/backend

rm BuildingBlocks.UnitTests/UnitTest1.cs
```

Now we’ll add our own test classes in the next sections.

---

## 6.5 Testing the Result Type

We want to be confident that `Result` and `Result<T>` behave as expected:

- `Result.Success()` → `IsSuccess == true`, `IsFailure == false`, `Error == null`.
- `Result.Failure("error")` → `IsSuccess == false`, `IsFailure == true`, `Error` contains our message.
- The same semantics for `Result<T>`.

Create `src/backend/BuildingBlocks.UnitTests/ResultTests.cs`:

```csharp
using BuildingBlocks;
using Xunit;

namespace BuildingBlocks.UnitTests;

public class ResultTests
{
    [Fact]
    public void Success_Should_Set_IsSuccess_True_And_IsFailure_False()
    {
        // Act
        var result = Result.Success();

        // Assert
        Assert.True(result.IsSuccess);
        Assert.False(result.IsFailure);
        Assert.Null(result.Error);
    }

    [Fact]
    public void Failure_Should_Set_IsSuccess_False_And_Error_Message()
    {
        // Arrange
        var errorMessage = "Something went wrong";

        // Act
        var result = Result.Failure(errorMessage);

        // Assert
        Assert.False(result.IsSuccess);
        Assert.True(result.IsFailure);
        Assert.Equal(errorMessage, result.Error);
    }

    [Fact]
    public void ResultOfT_Success_Should_Contain_Value()
    {
        // Arrange
        var value = 42;

        // Act
        var result = Result<int>.Success(value);

        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal(value, result.Value);
        Assert.Null(result.Error);
    }

    [Fact]
    public void ResultOfT_Failure_Should_Not_Contain_Value()
    {
        // Arrange
        var errorMessage = "Failure with generic type";

        // Act
        var result = Result<int>.Failure(errorMessage);

        // Assert
        Assert.True(result.IsFailure);
        Assert.Equal(errorMessage, result.Error);
        Assert.Equal(default, result.Value);
    }
}
```

Run the tests locally:

```bash
cd digital-banking-suite/src/backend
dotnet test BankingSuite.Backend.sln
```

You should see the test project discovered and all tests passing.

---

## 6.6 Testing ValueObject Equality

Next, let’s verify that our `ValueObject` base class behaves correctly for equality and inequality.

To do that, we’ll:

1. Create a simple test-only value object, e.g. `Money`.
2. Write tests that compare different instances.

Create `src/backend/BuildingBlocks.UnitTests/TestMoney.cs`:

```csharp
using BuildingBlocks;

namespace BuildingBlocks.UnitTests;

// Test-only value object to verify equality semantics
public sealed class TestMoney : ValueObject
{
    public decimal Amount { get; }
    public string Currency { get; }

    public TestMoney(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Amount;
        yield return Currency;
    }
}
```

Now create `src/backend/BuildingBlocks.UnitTests/ValueObjectTests.cs`:

```csharp
using Xunit;

namespace BuildingBlocks.UnitTests;

public class ValueObjectTests
{
    [Fact]
    public void Two_ValueObjects_With_Same_Values_Should_Be_Equal()
    {
        // Arrange
        var a = new TestMoney(100m, "USD");
        var b = new TestMoney(100m, "USD");

        // Act & Assert
        Assert.Equal(a, b);
        Assert.True(a == b);
        Assert.False(a != b);
    }

    [Fact]
    public void Two_ValueObjects_With_Different_Values_Should_Not_Be_Equal()
    {
        // Arrange
        var a = new TestMoney(100m, "USD");
        var b = new TestMoney(200m, "USD");

        // Act & Assert
        Assert.NotEqual(a, b);
        Assert.True(a != b);
        Assert.False(a == b);
    }

    [Fact]
    public void ValueObject_Should_Have_Consistent_HashCode_For_Equal_Values()
    {
        // Arrange
        var a = new TestMoney(50m, "EUR");
        var b = new TestMoney(50m, "EUR");

        // Act
        var hashA = a.GetHashCode();
        var hashB = b.GetHashCode();

        // Assert
        Assert.Equal(hashA, hashB);
    }
}
```

Run the tests again:

```bash
cd digital-banking-suite/src/backend
dotnet test BankingSuite.Backend.sln
```

You should now see a few more tests running and passing.

I> These tests are small, but they validate the “gravity” of your building blocks. Every future domain model that relies on `ValueObject` and `Result` will inherit these guarantees.

---

## 6.7 (Optional) Testing Entity Equality

Testing `Entity` is similar: we want to ensure entities with the same `Id` are considered equal.

This is optional at this stage, but it’s a good pattern to show.

Create `src/backend/BuildingBlocks.UnitTests/TestEntity.cs`:

```csharp
using BuildingBlocks;

namespace BuildingBlocks.UnitTests;

// Simple test-only entity for equality tests
public sealed class TestEntity : Entity
{
    public string Name { get; }

    public TestEntity(string name)
        : base()
    {
        Name = name;
    }

    public TestEntity(Guid id, string name)
        : base(id)
    {
        Name = name;
    }
}
```

Now create `src/backend/BuildingBlocks.UnitTests/EntityTests.cs`:

```csharp
using Xunit;

namespace BuildingBlocks.UnitTests;

public class EntityTests
{
    [Fact]
    public void Entities_With_Same_Id_Should_Be_Equal()
    {
        // Arrange
        var id = Guid.NewGuid();
        var entityA = new TestEntity(id, "A");
        var entityB = new TestEntity(id, "B");

        // Act & Assert
        Assert.Equal(entityA, entityB);
        Assert.True(entityA == entityB);
        Assert.False(entityA != entityB);
    }

    [Fact]
    public void Entities_With_Different_Id_Should_Not_Be_Equal()
    {
        // Arrange
        var entityA = new TestEntity(Guid.NewGuid(), "A");
        var entityB = new TestEntity(Guid.NewGuid(), "A");

        // Act & Assert
        Assert.NotEqual(entityA, entityB);
        Assert.True(entityA != entityB);
        Assert.False(entityA == entityB);
    }
}
```

Run tests again:

```bash
cd digital-banking-suite/src/backend
dotnet test BankingSuite.Backend.sln
```

All tests should still pass.

---

## 6.8 Updating CI to Run Backend Tests

In Chapter 5, we updated CI to:

- Restore all backend projects.
- Build `BankingSuite.Backend.sln`.

Now that we have tests, we want CI to also:

- **Run backend tests** using `dotnet test` on the solution.

Open `.github/workflows/ci.yml` and add a new step **after** the backend build step:

```yaml
- name: Run backend tests (if backend solution exists)
  run: |
    if [ -f "./src/backend/BankingSuite.Backend.sln" ]; then
      echo "Running backend tests..."
      dotnet test ./src/backend/BankingSuite.Backend.sln --configuration Release --no-build
    else
      echo "No backend solution found yet. Skipping tests."
    fi
```

This step:

- Checks if `BankingSuite.Backend.sln` exists.
- If it does, runs all tests in the solution in **Release** configuration.
- Uses `--no-build` to avoid rebuilding (the previous step already built the solution).

From now on, every PR will:

- Restore backend projects.
- Build the backend solution.
- Run all backend tests.

W> If tests fail, the CI pipeline fails. That’s **exactly what we want** — no broken code should merge into `develop`.

---

## 6.9 Local Sanity Check Before Committing

Before committing, run the whole flow locally:

```bash
cd digital-banking-suite/src/backend

# Build solution
dotnet build BankingSuite.Backend.sln --configuration Release

# Run tests
dotnet test BankingSuite.Backend.sln --configuration Release --no-build
```

If both commands succeed, you’re ready to commit.

---

## 6.10 Wrap-up: PR, CI & Merge to `develop`

At this point you should have, on the branch  
`feature/ch06-backend-testing-foundations`:

- A new test project: `BuildingBlocks.UnitTests`.
- Tests for:
  - `Result` / `Result<T>`
  - `ValueObject` equality (with `TestMoney`)
  - (Optional) `Entity` equality (with `TestEntity`)
- `BuildingBlocks.UnitTests` added to `BankingSuite.Backend.sln`.
- CI updated to run `dotnet test` for the backend.

Now we finish the Git workflow for the chapter.

### 6.10.1 Committing Your Changes

Stage and commit your work in logical chunks. For example:

```bash
git add src/backend/BuildingBlocks.UnitTests/*
git add src/backend/BuildingBlocks.UnitTests/BuildingBlocks.UnitTests.csproj
git commit -m "test(building-blocks): add unit tests for Result, ValueObject, and Entity"

git add .github/workflows/ci.yml
git commit -m "chore(ci): run backend tests in pipeline"
```

T> As always, keep commits **focused and descriptive**. It makes PR review and future archaeology much easier.

### 6.10.2 Pushing and Opening a Pull Request

Push the branch to GitHub:

```bash
git push -u origin feature/ch06-backend-testing-foundations
```

Then, in GitHub:

1. Open a Pull Request from  
   `feature/ch06-backend-testing-foundations` → `develop`
2. Give it a clear title, for example:  
   **"Chapter 6 – Backend Testing Foundations"**
3. Briefly describe what this PR adds:
   - `BuildingBlocks.UnitTests` project
   - Unit tests for `Result`, `ValueObject`, and `Entity`
   - CI update to run backend tests
   - Chapter 6 documentation

### 6.10.3 Watching the CI Pipeline

When you open the PR, the CI workflow should:

- Restore all backend projects.
- Build `BankingSuite.Backend.sln`.
- Run `dotnet test` against the backend solution.

If any test fails, the pipeline will be red. Fix the issue locally, push again, and wait for CI to turn green.

W> Never merge a red pipeline. Broken tests in `develop` are a productivity killer.

### 6.10.4 Merging to `develop`

Once the pipeline is green:

1. Merge the PR into `develop`.
2. Optionally delete the feature branch:

   - in GitHub (Delete branch button), and/or
   - locally:

   ```bash
   git checkout develop
   git pull origin develop
   git branch -d feature/ch06-backend-testing-foundations
   ```

At this point:

- `develop` contains all Chapter 6 changes.
- CI is now watching your backend with tests on every PR.
- You’re ready to start the next chapter from a clean, green baseline.

---

## 6.11 Summary & What’s Next

In this chapter, we:

- Defined a **testing strategy** for the backend:
  - Unit tests first, integration tests later.
- Created a `BuildingBlocks.UnitTests` project using xUnit.
- Added **unit tests** for:
  - `Result` / `Result<T>` success and failure.
  - `ValueObject` equality semantics (using a test `TestMoney` value object).
  - Entity equality (optional, using `TestEntity`).
- Added the test project to `BankingSuite.Backend.sln`.
- Updated CI to run `dotnet test` on the backend solution in every PR.

From now on:

- Any change that breaks `Result`, `ValueObject`, or `Entity` will be caught immediately by tests and CI.
- Future services (IAM, Customer, Account, Transaction) will build on top of a **tested foundation**.

In the next chapter, we’ll start moving closer to the **real banking domain**:

- Design and implement the first real bounded context as code — most likely **IAMService** or **CustomerService** foundations.
- Use the building blocks and testing patterns we’ve created here.
- Continue evolving CI, Docker, and our solution structure as we add real behavior.

Your backend now has **both structure and tests**.  
It’s no longer just “compiles”; it’s **actively protected** by a growing safety net of unit tests.
