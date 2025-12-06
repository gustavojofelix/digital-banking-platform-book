# Chapter 06 — Backend Tests & CI with Code Coverage

In the previous chapter we created:

- The `BankingSuite.Backend.sln` solution
- Shared **BuildingBlocks** projects
- The **IAM service skeleton** with a `/health` endpoint using FastEndpoints

Now we’ll add:

- **Unit & integration test projects** for the IAM service
- **Coverlet** for code coverage
- **ReportGenerator** for HTML coverage reports
- A **GitHub Actions** workflow to run tests and generate coverage in CI

By the end of this chapter you will:

- Run backend tests from **Visual Studio 2026** and the **CLI**
- Generate a **coverage report** locally
- Have a working **CI pipeline** that runs tests and publishes coverage artifacts
- Tag the repository state for this chapter

> **IDE note:** All project/file steps assume **Visual Studio 2026**.  
> If you’re using another IDE, create equivalent projects and references using your tooling or the `dotnet` CLI.

---

## 6.1 Create test projects for IAM

We’ll add two projects under the `tests` folder:

- `BankingSuite.IamService.UnitTests`
- `BankingSuite.IamService.IntegrationTests`

And a `Tests` solution folder so they’re organised inside the solution.

### 6.1.1 Add the `Tests` solution folder

1. In **Solution Explorer**, right-click the **`BankingSuite.Backend`** solution.
2. Choose **Add → New Solution Folder**.
3. Name it **`Tests`**.

This is just a logical folder in the solution (not a physical one).

---

### 6.1.2 Create `BankingSuite.IamService.UnitTests` (xUnit)

1. Right-click the **`Tests`** solution folder → **Add → New Project…**
2. Search for **“xUnit Test Project”**.
3. Select **xUnit Test Project (.NET)** and click **Next**.
4. Set:
   - **Project name:** `BankingSuite.IamService.UnitTests`
   - **Location:** `digital-banking-suite/tests`
5. Click **Next**, ensure **Framework** is **`.NET 10.0 (net10.0)`**, then click **Create**.

Visual Studio creates:

```text
tests/BankingSuite.IamService.UnitTests/
```

Delete the default `UnitTest1.cs`.

#### Add references and packages

1. Right-click `BankingSuite.IamService.UnitTests` → **Add → Project Reference…**
2. Tick:
   - `BankingSuite.IamService.Domain`
   - `BankingSuite.IamService.Application`
   - `BankingSuite.BuildingBlocks.Domain`
3. Click **OK**.

Now add helpful NuGet packages:

- Right-click `BankingSuite.IamService.UnitTests` → **Manage NuGet Packages…** → **Browse** tab, install:
  - `xunit` (if not already there)
  - `xunit.runner.visualstudio`
  - `FluentAssertions`
  - `coverlet.msbuild`

We’ll use `coverlet.msbuild` when we run `dotnet test` with coverage.

---

### 6.1.3 Create `BankingSuite.IamService.IntegrationTests` (xUnit)

Repeat the process:

1. Right-click the **`Tests`** solution folder → **Add → New Project…**
2. Choose **xUnit Test Project**.
3. Set:
   - **Project name:** `BankingSuite.IamService.IntegrationTests`
   - **Location:** `digital-banking-suite/tests`
4. Target **`.NET 10.0`** and click **Create**.

Delete the default `UnitTest1.cs`.

#### Add references and packages

1. Right-click `BankingSuite.IamService.IntegrationTests` → **Add → Project Reference…**
2. Tick:
   - `BankingSuite.IamService.API`
   - `BankingSuite.IamService.Infrastructure`
   - `BankingSuite.IamService.Application`
   - `BankingSuite.IamService.Domain`
3. Click **OK**.

Then add NuGet packages:

- `xunit`
- `xunit.runner.visualstudio`
- `FluentAssertions`
- `Microsoft.AspNetCore.Mvc.Testing`
- `coverlet.msbuild`

> `Microsoft.AspNetCore.Mvc.Testing` gives us `WebApplicationFactory<T>` which we’ll use to host the IAM API in-memory for integration tests.

---

## 6.2 Make `Program` test-friendly

To use `WebApplicationFactory<Program>`, our entry point (`Program`) must be visible to the test project.

Open `src/backend/Services/IamService/BankingSuite.IamService.API/Program.cs` and, at the very bottom of the file, add:

```csharp
public partial class Program;
```

The full file should still contain the FastEndpoints setup and end with this partial class declaration.

This allows tests to reference `Program` as the entry point.

---

## 6.3 Add a sample unit test (Result type)

We already have a `Result` type in `BankingSuite.BuildingBlocks.Domain.Abstractions`. Let’s write a simple unit test for it.

In `BankingSuite.IamService.UnitTests`:

1. Right-click the project → **Add → New Folder** → name it `Common`.
2. Right-click the `Common` folder → **Add → Class…** → name it `ResultTests.cs`.

Replace with:

```csharp
using BankingSuite.BuildingBlocks.Domain.Abstractions;
using FluentAssertions;
using Xunit;

namespace BankingSuite.IamService.UnitTests.Common;

public class ResultTests
{
    [Fact]
    public void Success_Should_Set_IsSuccess_True_And_Error_Null()
    {
        // Arrange & Act
        var result = Result.Success();

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.IsFailure.Should().BeFalse();
        result.Error.Should().BeNull();
    }

    [Fact]
    public void Failure_Should_Set_IsSuccess_False_And_Error_Message()
    {
        // Arrange
        const string errorMessage = "Something went wrong";

        // Act
        var result = Result.Failure(errorMessage);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be(errorMessage);
    }

    [Fact]
    public void Generic_Success_Should_Expose_Value()
    {
        // Arrange
        const string expected = "OK";

        // Act
        var result = Result.Success(expected);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().Be(expected);
        result.Error.Should().BeNull();
    }
}
```

This is a simple example, but it demonstrates the pattern we’ll use for domain and application tests.

---

## 6.4 Add an integration test for `/health`

Now we’ll test the IAM API’s `/health` endpoint.

In `BankingSuite.IamService.IntegrationTests`:

1. Right-click the project → **Add → New Folder** → name it `Health`.
2. Right-click the `Health` folder → **Add → Class…** → name it `HealthEndpointTests.cs`.

Replace with:

```csharp
using System.Net;
using System.Net.Http.Json;
using FluentAssertions;
using Microsoft.AspNetCore.Mvc.Testing;
using Xunit;

namespace BankingSuite.IamService.IntegrationTests.Health;

public class HealthEndpointTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public HealthEndpointTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Health_Should_Return_Healthy_Status()
    {
        // Act
        var response = await _client.GetAsync("/health");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var payload = await response.Content.ReadFromJsonAsync<HealthResponse>();

        payload.Should().NotBeNull();
        payload!.Service.Should().Be("IamService");
        payload.Status.Should().Be("Healthy");
    }

    private sealed class HealthResponse
    {
        public string? Service { get; set; }
        public string? Status { get; set; }
        public DateTime TimestampUtc { get; set; }
    }
}
```

This test:

- Boots the IAM API in-memory using `WebApplicationFactory<Program>`.
- Sends a GET request to `/health`.
- Asserts the status is `200 OK` and the payload matches what we return from `HealthCheckEndpoint`.

---

## 6.5 Run tests from Visual Studio 2026

1. Open **Test → Test Explorer**.
2. Click **Run All Tests**.

You should see:

- 3 passing tests in `ResultTests`.
- 1 passing test in `HealthEndpointTests`.

If any tests fail, re-check `Program.cs` (partial class) and that all references/packages are in place.

---

## 6.6 Install ReportGenerator tool

We’ll use `ReportGenerator` to generate HTML coverage reports from the coverage data produced by Coverlet.

From a terminal at the **repo root** (`digital-banking-suite`):

```bash
dotnet tool install --global dotnet-reportgenerator-globaltool
```

Verify:

```bash
reportgenerator -h
```

If the command is not found, ensure your `~/.dotnet/tools` is on your `PATH`.

> In CI, we’ll also install this tool using `dotnet tool install --global`.

---

## 6.7 Run tests with coverage locally

We’ll use the `coverlet.msbuild` integration via `dotnet test`.

From the **repo root** (`digital-banking-suite`):

```bash
dotnet test src/backend/BankingSuite.Backend.sln ^
  /p:CollectCoverage=true ^
  /p:CoverletOutput=../../lcov ^
  /p:CoverletOutputFormat=lcov
```

> On Linux/macOS, replace `^` with `\` for line continuation.

Explanation:

- `CollectCoverage=true` tells Coverlet to instrument the code.
- `CoverletOutput=../../lcov` will (for our layout) generate an `lcov.info` file at the repository root.
- `CoverletOutputFormat=lcov` uses the `lcov` format, which we’ll feed to `ReportGenerator`.

After this command succeeds, you should see `lcov.info` in the root of `digital-banking-suite`.

Now generate an HTML report:

```bash
reportgenerator ^
  -reports:lcov.info ^
  -targetdir:src/backend/coverage-report ^
  "-reporttypes:Html;TextSummary"
```

On Linux/macOS:

```bash
reportgenerator \
  -reports:lcov.info \
  -targetdir:src/backend/coverage-report \
  "-reporttypes:Html;TextSummary"
```

This will:

- Create `src/backend/coverage-report/index.html` (and related files).
- Print a **TextSummary** to the console (including overall line coverage).

Open `src/backend/coverage-report/index.html` in your browser to explore coverage per project, namespace and class.

> As we add more tests and microservices, this report becomes much more interesting.

---

## 6.8 Add backend tests workflow in GitHub Actions

Now we’ll add a CI workflow that runs on pushes/PRs and:

- Restores dependencies
- Builds the backend solution
- Runs tests with coverage
- Generates an HTML coverage report
- Uploads the report as a build artifact

### 6.8.1 Create the workflow folder and file

Inside the `digital-banking-suite` folder, create:

```text
.github/
└─ workflows/
   └─ backend-tests.yml
```

You can:

- Create `.github` and `workflows` via File Explorer or terminal,
- Then in Visual Studio 2026, right-click `workflows` → **Add → New Item…** → **Text File**, name it `backend-tests.yml`.

### 6.8.2 Backend tests workflow (`backend-tests.yml`)

Paste the following:

```yaml
name: Backend Tests & Coverage

on:
  push:
    branches:
      - develop
      - main
      - feature/**
  pull_request:
    branches:
      - develop
      - main

jobs:
  tests:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET 10
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "10.0.x"

      - name: Restore dependencies
        run: dotnet restore src/backend/BankingSuite.Backend.sln

      - name: Build
        run: dotnet build src/backend/BankingSuite.Backend.sln --no-restore

      - name: Install ReportGenerator tool
        run: dotnet tool install --global dotnet-reportgenerator-globaltool

      - name: Run tests with coverage
        run: |
          dotnet test src/backend/BankingSuite.Backend.sln \
            /p:CollectCoverage=true \
            /p:CoverletOutput=../../../lcov \
            /p:CoverletOutputFormat=lcov

      - name: Generate coverage report
        run: |
          reportgenerator \
            -reports:lcov.info \
            -targetdir:src/backend/coverage-report \
            "-reporttypes:Html;TextSummary"
        env:
          PATH: $PATH:~/.dotnet/tools

      - name: Upload coverage report artifact
        uses: actions/upload-artifact@v4
        with:
          name: backend-coverage-report
          path: src/backend/coverage-report
```

Notes:

- We install .NET 10 on the runner.
- We install `reportgenerator` as a global dotnet tool.
- `PATH` is extended so the `reportgenerator` command is found.
- The HTML report is uploaded as an artifact named `backend-coverage-report`.

We are **not yet enforcing** a minimum coverage threshold in CI — we’ll add stricter rules later once we have more tests. For now, we just generate the report and read the summary.

---

## 6.9 Commit and tag Chapter 06

As usual, follow the Git workflow.

From the repo root:

```bash
git status
git add src/backend tests .github
git commit -m "ch06: add IAM tests, coverage and backend CI workflow"
```

If you are using a feature branch (recommended):

```bash
git push -u origin feature/ch06-backend-tests
```

Create a PR into `develop`, review and merge it.

Then tag the state for this chapter:

```bash
git checkout develop
git pull

git tag -a chapter-06-backend-tests-ci -m "Chapter 06: backend tests, coverage and CI"
git push origin chapter-06-backend-tests-ci
```

Now you and readers can always restore the repo to the exact state at the end of this chapter.

---

## 6.10 Sanity checklist

Before moving on, confirm:

- [x] `BankingSuite.IamService.UnitTests` and `BankingSuite.IamService.IntegrationTests` exist under `tests/`.
- [x] Tests run successfully in **Visual Studio 2026**.
- [x] `dotnet test ... /p:CollectCoverage=true ...` succeeds from the command line.
- [x] `lcov.info` is generated and `reportgenerator` produces `src/backend/coverage-report/index.html`.
- [x] The **GitHub Actions** workflow (`backend-tests.yml`) runs on pushes/PRs to `develop` and uploads the coverage artifact.

If everything looks good, your backend foundation is now:

- **Tested** (unit + integration)
- **Measurable** (coverage)
- **CI-aware** (GitHub Actions pipeline)

---

## 6.11 What’s next

In the next chapter we will:

- Finalise the **backend structure and building blocks**
- Start implementing the **IAM service** properly:
  - Identity models
  - ASP.NET Core Identity setup
  - EF Core mappings and migrations
  - Real endpoints for registration, login, user management

From this point on, every backend feature we add will:

- Have tests
- Participate in coverage
- Be validated automatically by our CI pipeline

Exactly the behaviour we want in a **real banking team**.
