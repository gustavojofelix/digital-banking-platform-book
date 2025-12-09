## 8.5 Integration Tests to Push IAM Coverage Toward 80%+

In this section we **focus only on tests**.

Our goal is to treat the IAM service like a **real banking backend**:

- Exercise the **actual HTTP endpoints** (FastEndpoints)
- Hit the full pipeline: routing → middleware → authentication/authorization → handlers → EF Core + Identity
- Cover the most important flows:
  - Admin employee management
  - Email confirmation & password flows
  - 2FA login and 2FA settings
- Push IAM test coverage towards **80%+**

We’ll reuse the same tooling and patterns from earlier chapters:

- `xUnit` test projects
- `WebApplicationFactory<Program>` (ASP.NET Core test host)
- Test doubles for infrastructure dependencies (e.g. email sender)
- `dotnet test` with Coverlet and `reportgenerator` to generate coverage reports

> **Note**  
> In this book structure, this file `08-04-iam-integration-tests-and-coverage.md` complements:
>
> - `08-01-iam-admin-employee-management.md`
> - `08-02-iam-email-confirmation-and-password-flows.md`
> - `08-03-iam-2fa-and-security-hardening.md`  
>   You can remove/trim the old mini “tests” subsection from `08-03` and point readers here instead.

---

### 8.5.1 Branch Strategy for IAM Integration Tests

We keep our Git discipline:

- `main` → **published** & stable
- `develop` → integration branch for ongoing Part II work
- One branch per section → permanent reference for the chapter

For IAM integration tests:

- **Branch name:** `part2-chapter08-iam-integration-tests-and-coverage`

From the repo root:

```bash
git checkout develop
git pull origin develop

git checkout -b part2-chapter08-iam-integration-tests-and-coverage
```

All test-related changes for IAM in this chapter go into this branch, which you’ll later merge into `develop`.

---

### 8.5.2 IAM Integration Test Project Setup

We’ll create a dedicated test project:

- `tests/BankingSuite.IAM.IntegrationTests`

This project will:

- Reference the IAM API project
- Spin up the **full IAM API** in-memory using `WebApplicationFactory<Program>`
- Override configuration for a test database and test email sender
- Provide helper methods to authenticate as admin/employee

From the solution root (where your `.sln` lives):

```bash
cd src/backend

dotnet new xunit -n BankingSuite.IAM.IntegrationTests -o ../tests/BankingSuite.IAM.IntegrationTests

# Add project reference to IAM API
dotnet add ../tests/BankingSuite.IAM.IntegrationTests/BankingSuite.IAM.IntegrationTests.csproj \
  reference services/iam/BankingSuite.IAM.Api/BankingSuite.IAM.Api.csproj
```

Now open `BankingSuite.IAM.IntegrationTests.csproj` and ensure it roughly looks like this:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="9.0.0" />
    <PackageReference Include="FluentAssertions" Version="6.12.0" />
    <PackageReference Include="Respawn" Version="6.1.0" />
    <!-- Add Testcontainers for PostgreSQL if you want true containerized DB -->
    <!-- <PackageReference Include="Testcontainers.PostgreSql" Version="3.9.0" /> -->
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\backend\services\iam\BankingSuite.IAM.Api\BankingSuite.IAM.Api.csproj" />
  </ItemGroup>

</Project>
```

> **Note:**  
> Versions here are illustrative; align them with the rest of your solution.

---

### 8.5.3 IAM Test Host: Custom WebApplicationFactory

We need a **test host** that:

- Bootstraps the IAM API (`Program`)
- Uses a test database (in-memory or a test PostgreSQL database)
- Replaces `IEmailSender` with a test double that stores “sent emails” in memory

Create:

- `tests/BankingSuite.IAM.IntegrationTests/Infrastructure/IamApiFactory.cs`

```csharp
using System.Net.Http.Headers;
using BankingSuite.IAM.Application.Common.Interfaces;
using BankingSuite.IAM.Domain;
using BankingSuite.IAM.Infrastructure.Persistence;
using FastEndpoints;
using FastEndpoints.Swagger;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace BankingSuite.IAM.IntegrationTests.Infrastructure;

public sealed class IamApiFactory : WebApplicationFactory<Program>
{
    public List<TestEmailMessage> SentEmails { get; } = [];

    protected override IHost CreateHost(IHostBuilder builder)
    {
        // Use a different environment if needed
        builder.UseEnvironment("IntegrationTests");

        builder.ConfigureServices(services =>
        {
            // Replace real IEmailSender with a test double
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(IEmailSender));

            if (descriptor is not null)
                services.Remove(descriptor);

            services.AddSingleton<IEmailSender>(_ => new TestEmailSender(SentEmails));

            // Option A: Replace DB context with in-memory database for tests
            var dbDescriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<IamDbContext>));

            if (dbDescriptor is not null)
                services.Remove(dbDescriptor);

            services.AddDbContext<IamDbContext>(options =>
            {
                options.UseInMemoryDatabase("iam_integration_tests_db");
            });
        });

        var host = base.CreateHost(builder);

        // Ensure database created and seeded
        using var scope = host.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<IamDbContext>();
        db.Database.EnsureCreated();

        return host;
    }
}

public sealed record TestEmailMessage(string To, string Subject, string HtmlBody);

public sealed class TestEmailSender(List<TestEmailMessage> store) : IEmailSender
{
    private readonly List<TestEmailMessage> _store = store;

    public Task SendEmailAsync(string toEmail, string subject, string htmlBody, CancellationToken cancellationToken = default)
    {
        _store.Add(new TestEmailMessage(toEmail, subject, htmlBody));
        return Task.CompletedTask;
    }
}
```

This gives us:

- A reusable IAM API factory
- A simple way to **inspect emails** sent during tests (for confirmation/reset/2FA codes)
- An in-memory database (good for coverage & speed; you can swap to PostgreSQL via Testcontainers if desired)

---

### 8.5.4 Test Helpers: Users, Auth, and JSON

To keep tests clean, add a small set of helpers:

- Create a base class for IAM integration tests
- Methods to:
  - Create an admin user
  - Authenticate as admin or normal employee
  - Send JSON requests

Create:

- `tests/BankingSuite.IAM.IntegrationTests/Infrastructure/IntegrationTestBase.cs`

```csharp
using System.Net.Http.Json;
using BankingSuite.IAM.Domain;
using BankingSuite.IAM.Infrastructure.Persistence;
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.DependencyInjection;

namespace BankingSuite.IAM.IntegrationTests.Infrastructure;

public abstract class IntegrationTestBase : IClassFixture<IamApiFactory>, IAsyncLifetime
{
    protected readonly IamApiFactory Factory;
    protected readonly HttpClient Client;

    protected IntegrationTestBase(IamApiFactory factory)
    {
        Factory = factory;
        Client = factory.CreateClient();
    }

    public virtual async Task InitializeAsync()
    {
        // Optional: clear DB between tests if using a shared factory
        using var scope = Factory.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<IamDbContext>();

        db.Users.RemoveRange(db.Users);
        db.Roles.RemoveRange(db.Roles);
        await db.SaveChangesAsync();
    }

    public Task DisposeAsync() => Task.CompletedTask;

    protected async Task<ApplicationUser> CreateEmployeeAsync(
        string email,
        string password,
        bool isActive = true,
        bool emailConfirmed = true,
        bool twoFactorEnabled = false,
        string? fullName = null)
    {
        using var scope = Factory.Services.CreateScope();
        var userManager = scope.ServiceProvider.GetRequiredService<UserManager<ApplicationUser>>();

        var user = new ApplicationUser
        {
            UserName = email,
            Email = email,
            FullName = fullName ?? email,
            IsActive = isActive,
            EmailConfirmed = emailConfirmed,
            TwoFactorEnabled = twoFactorEnabled
        };

        var result = await userManager.CreateAsync(user, password);
        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            throw new InvalidOperationException($"Failed to create test user: {errors}");
        }

        return user;
    }

    protected async Task AddRoleAsync(ApplicationUser user, string roleName)
    {
        using var scope = Factory.Services.CreateScope();
        var userManager = scope.ServiceProvider.GetRequiredService<UserManager<ApplicationUser>>();
        var roleManager = scope.ServiceProvider.GetRequiredService<RoleManager<ApplicationRole>>();

        if (!await roleManager.RoleExistsAsync(roleName))
        {
            await roleManager.CreateAsync(new ApplicationRole { Name = roleName });
        }

        await userManager.AddToRoleAsync(user, roleName);
    }

    protected async Task<string> AuthenticateAsync(string email, string password)
    {
        var response = await Client.PostAsJsonAsync("/api/iam/auth/login", new
        {
            email,
            password
        });

        response.EnsureSuccessStatusCode();

        var json = await response.Content.ReadFromJsonAsync<LoginResponseForTests>();

        if (json is null || json.AccessToken is null)
            throw new InvalidOperationException("Login did not return an access token.");

        Client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", json.AccessToken);

        return json.AccessToken;
    }

    private sealed class LoginResponseForTests
    {
        public bool RequiresTwoFactor { get; init; }
        public Guid? UserId { get; init; }
        public string? AccessToken { get; init; }
        public DateTimeOffset? ExpiresAt { get; init; }
    }
}
```

> Note:  
> For 2FA tests, we’ll slightly change how we handle login responses (since `RequiresTwoFactor` might be `true`). We’ll show that when we get to the 2FA test examples.

---

### 8.5.5 Tests for Admin Employee Management

We start with the admin endpoints implemented in `08-01`.

Create:

- `tests/BankingSuite.IAM.IntegrationTests/Admin/AdminEmployeesTests.cs`

```csharp
using System.Net;
using System.Net.Http.Json;
using BankingSuite.IAM.Domain;
using BankingSuite.IAM.IntegrationTests.Infrastructure;
using FluentAssertions;

namespace BankingSuite.IAM.IntegrationTests.Admin;

public sealed class AdminEmployeesTests(IamApiFactory factory) : IntegrationTestBase(factory)
{
    private const string AdminRole = "IamAdmin";
    private const string AdminEmail = "admin@alvorbank.test";
    private const string AdminPassword = "Admin123!";

    [Fact]
    public async Task ListEmployees_ReturnsPagedResult_ForAdmin()
    {
        // Arrange
        var admin = await CreateEmployeeAsync(AdminEmail, AdminPassword, emailConfirmed: true);
        await AddRoleAsync(admin, AdminRole);

        var emp1 = await CreateEmployeeAsync("e1@alvorbank.test", "Password123!", emailConfirmed: true);
        var emp2 = await CreateEmployeeAsync("e2@alvorbank.test", "Password123!", emailConfirmed: true, isActive: false);

        await AuthenticateAsync(AdminEmail, AdminPassword);

        // Act
        var response = await Client.GetAsync("/api/iam/admin/employees?pageNumber=1&pageSize=10&includeInactive=true");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);

        var result = await response.Content.ReadFromJsonAsync<PagedEmployeesResponse>();

        result.Should().NotBeNull();
        result!.Items.Should().HaveCount(3); // admin + 2 employees
        result.Items.Any(e => e.Email == emp1.Email).Should().BeTrue();
        result.Items.Any(e => e.Email == emp2.Email).Should().BeTrue();
    }

    [Fact]
    public async Task DeactivateEmployee_ThenLogin_ShouldFail()
    {
        // Arrange
        var admin = await CreateEmployeeAsync(AdminEmail, AdminPassword, emailConfirmed: true);
        await AddRoleAsync(admin, AdminRole);

        var employee = await CreateEmployeeAsync("user@alvorbank.test", "Password123!", emailConfirmed: true);

        await AuthenticateAsync(AdminEmail, AdminPassword);

        // Act: deactivate the employee
        var deactivateResponse = await Client.PostAsync($"/api/iam/admin/employees/{employee.Id}/deactivate", content: null);
        deactivateResponse.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Try to log in as that employee
        var loginResponse = await Client.PostAsJsonAsync("/api/iam/auth/login", new
        {
            email = employee.Email,
            password = "Password123!"
        });

        // Assert: login should fail (we expect a 4xx)
        loginResponse.StatusCode.Should().Be(HttpStatusCode.BadRequest)
            .Or.Be(HttpStatusCode.Unauthorized);
    }

    private sealed class PagedEmployeesResponse
    {
        public List<EmployeeSummary> Items { get; init; } = [];
        public int TotalCount { get; init; }
        public int PageNumber { get; init; }
        public int PageSize { get; init; }
    }

    private sealed class EmployeeSummary
    {
        public Guid Id { get; init; }
        public string Email { get; init; } = string.Empty;
        public bool IsActive { get; init; }
    }
}
```

This covers:

- Admin-only listing
- Activation/Deactivation behavior reflected in login

You can add more tests for **UpdateEmployee** and **GetEmployee** to further improve coverage.

---

### 8.5.6 Tests for Email Confirmation & Password Flows

Now we target the flows in `08-02`.

Create:

- `tests/BankingSuite.IAM.IntegrationTests/Auth/EmailConfirmationAndPasswordTests.cs`

```csharp
using System.Net;
using System.Net.Http.Json;
using System.Web;
using BankingSuite.IAM.IntegrationTests.Infrastructure;
using FluentAssertions;

namespace BankingSuite.IAM.IntegrationTests.Auth;

public sealed class EmailConfirmationAndPasswordTests(IamApiFactory factory) : IntegrationTestBase(factory)
{
    private const string Password = "Password123!";

    [Fact]
    public async Task ResendConfirmation_ThenConfirmEmail_ShouldMarkEmailAsConfirmed()
    {
        // Arrange
        var email = "employee@alvorbank.test";
        var user = await CreateEmployeeAsync(email, Password, emailConfirmed: false);

        // Act: request resend confirmation
        var resendResponse = await Client.PostAsJsonAsync("/api/iam/auth/resend-confirmation", new
        {
            email
        });

        resendResponse.StatusCode.Should().Be(HttpStatusCode.OK);
        Factory.SentEmails.Should().NotBeEmpty();

        var confirmationEmail = Factory.SentEmails.Last();
        confirmationEmail.To.Should().Be(email);

        // Extract link and token from email body (simple parse for demo)
        var link = ExtractFirstHref(confirmationEmail.HtmlBody);
        link.Should().NotBeNullOrWhiteSpace();

        var uri = new Uri(link);
        var query = HttpUtility.ParseQueryString(uri.Query);
        var userId = Guid.Parse(query["userId"]!);
        var token = query["token"]!;

        // Call confirm-email endpoint
        var confirmResponse = await Client.GetAsync($"/api/iam/auth/confirm-email?userId={userId}&token={HttpUtility.UrlEncode(token)}");

        confirmResponse.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Try to log in (should work if email is now confirmed)
        var loginResponse = await Client.PostAsJsonAsync("/api/iam/auth/login", new
        {
            email,
            password = Password
        });

        loginResponse.EnsureSuccessStatusCode();
    }

    [Fact]
    public async Task ForgotPassword_ThenResetPassword_ShouldAllowLoginWithNewPassword()
    {
        // Arrange
        var email = "reset@alvorbank.test";
        var user = await CreateEmployeeAsync(email, Password, emailConfirmed: true);

        // Act: forgot password
        var forgotResponse = await Client.PostAsJsonAsync("/api/iam/auth/forgot-password", new
        {
            email
        });

        forgotResponse.StatusCode.Should().Be(HttpStatusCode.OK);

        Factory.SentEmails.Should().NotBeEmpty();
        var resetEmail = Factory.SentEmails.Last();
        resetEmail.To.Should().Be(email);

        var link = ExtractFirstHref(resetEmail.HtmlBody);
        link.Should().NotBeNullOrWhiteSpace();

        var uri = new Uri(link);
        var query = HttpUtility.ParseQueryString(uri.Query);
        var token = query["token"]!;
        var encodedEmail = query["email"]!;
        var decodedEmail = HttpUtility.UrlDecode(encodedEmail);

        // Reset password
        const string newPassword = "NewPassword123!";

        var resetResponse = await Client.PostAsJsonAsync("/api/iam/auth/reset-password", new
        {
            email = decodedEmail,
            token,
            newPassword
        });

        resetResponse.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Try to log in with the new password
        var loginResponse = await Client.PostAsJsonAsync("/api/iam/auth/login", new
        {
            email = decodedEmail,
            password = newPassword
        });

        loginResponse.EnsureSuccessStatusCode();
    }

    private static string ExtractFirstHref(string html)
    {
        const string marker = "href=\"";
        var start = html.IndexOf(marker, StringComparison.OrdinalIgnoreCase);
        if (start < 0) return string.Empty;
        start += marker.Length;
        var end = html.IndexOf("\"", start, StringComparison.OrdinalIgnoreCase);
        if (end < 0) return string.Empty;
        return html[start..end];
    }
}
```

This covers:

- Resend confirmation → confirm email → login
- Forgot password → reset password → login with new password

You can add more tests for **invalid tokens**, **unconfirmed users**, etc.

---

### 8.5.7 Tests for 2FA Login & Settings

Finally, we test the 2FA flows from `08-03`.

Create:

- `tests/BankingSuite.IAM.IntegrationTests/Auth/TwoFactorTests.cs`

```csharp
using System.Net;
using System.Net.Http.Json;
using BankingSuite.IAM.IntegrationTests.Infrastructure;
using FluentAssertions;

namespace BankingSuite.IAM.IntegrationTests.Auth;

public sealed class TwoFactorTests(IamApiFactory factory) : IntegrationTestBase(factory)
{
    private const string Password = "Password123!";

    [Fact]
    public async Task Login_WithTwoFactorEnabled_ShouldRequire2FA_ThenVerify()
    {
        // Arrange: create a confirmed, active user with 2FA enabled
        var email = "2fa@alvorbank.test";
        var user = await CreateEmployeeAsync(email, Password, emailConfirmed: true, twoFactorEnabled: true);

        // Act: login with password
        var loginResponse = await Client.PostAsJsonAsync("/api/iam/auth/login", new
        {
            email,
            password = Password
        });

        loginResponse.StatusCode.Should().Be(HttpStatusCode.OK);

        var loginResult = await loginResponse.Content.ReadFromJsonAsync<LoginResultResponse>();
        loginResult.Should().NotBeNull();
        loginResult!.RequiresTwoFactor.Should().BeTrue();
        loginResult.UserId.Should().NotBeNull();
        loginResult.AccessToken.Should().BeNull();

        // The login should have sent a 2FA email
        Factory.SentEmails.Should().NotBeEmpty();
        var twoFaEmail = Factory.SentEmails.Last();
        twoFaEmail.To.Should().Be(email);

        // Extract the code from the email body (the code is usually plain text, not in a link)
        var code = ExtractCodeFromBody(twoFaEmail.HtmlBody);
        code.Should().NotBeNullOrWhiteSpace();

        // Call 2FA verify endpoint
        var verifyResponse = await Client.PostAsJsonAsync("/api/iam/auth/2fa/verify", new
        {
            userId = loginResult.UserId,
            code
        });

        verifyResponse.StatusCode.Should().Be(HttpStatusCode.OK);

        var verifyResult = await verifyResponse.Content.ReadFromJsonAsync<LoginResultResponse>();
        verifyResult.Should().NotBeNull();
        verifyResult!.RequiresTwoFactor.Should().BeFalse();
        verifyResult.AccessToken.Should().NotBeNullOrWhiteSpace();
    }

    [Fact]
    public async Task EnableAndDisableTwoFactor_ShouldUpdateUserFlag()
    {
        // Arrange
        var email = "2fasettings@alvorbank.test";
        var user = await CreateEmployeeAsync(email, Password, emailConfirmed: true);

        // Login to get token
        var loginResponse = await Client.PostAsJsonAsync("/api/iam/auth/login", new
        {
            email,
            password = Password
        });

        loginResponse.EnsureSuccessStatusCode();

        var loginResult = await loginResponse.Content.ReadFromJsonAsync<LoginResultResponse>();
        var token = loginResult!.AccessToken!;

        Client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

        // Act: enable 2FA
        var enableResponse = await Client.PostAsJsonAsync("/api/iam/auth/2fa/enable", new
        {
            currentPassword = Password
        });

        enableResponse.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Try login again, now it should require 2FA
        Client.DefaultRequestHeaders.Authorization = null;

        var login2Response = await Client.PostAsJsonAsync("/api/iam/auth/login", new
        {
            email,
            password = Password
        });

        login2Response.StatusCode.Should().Be(HttpStatusCode.OK);

        var login2Result = await login2Response.Content.ReadFromJsonAsync<LoginResultResponse>();
        login2Result!.RequiresTwoFactor.Should().BeTrue();

        // Act: disable 2FA
        Client.DefaultRequestHeaders.Authorization =
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token);

        var disableResponse = await Client.PostAsJsonAsync("/api/iam/auth/2fa/disable", new
        {
            currentPassword = Password
        });

        disableResponse.StatusCode.Should().Be(HttpStatusCode.NoContent);
    }

    private sealed class LoginResultResponse
    {
        public bool RequiresTwoFactor { get; init; }
        public Guid? UserId { get; init; }
        public string? AccessToken { get; init; }
        public DateTimeOffset? ExpiresAt { get; init; }
    }

    private static string ExtractCodeFromBody(string htmlBody)
    {
        // For demo purposes we assume the code is in a <strong> tag.
        const string marker = "<strong>";
        var start = htmlBody.IndexOf(marker, StringComparison.OrdinalIgnoreCase);
        if (start < 0) return string.Empty;
        start += marker.Length;
        var end = htmlBody.IndexOf("</strong>", start, StringComparison.OrdinalIgnoreCase);
        if (end < 0) return string.Empty;
        return htmlBody[start..end].Trim();
    }
}
```

These tests cover:

- Login with 2FA required, reading the code from email, verifying via `/api/iam/auth/2fa/verify`
- Basic enable/disable 2FA behavior for a logged-in employee

You can expand with negative tests:

- Wrong 2FA code → should fail
- Wrong password when enabling/disabling → should fail
- Inactive or unconfirmed users → cannot complete login/2FA

---

### 8.5.8 Running Coverage for IAM

To run coverage for the entire backend (including IAM), re-use the commands from earlier in Part II, adapted to your repo layout.

From the **repository root**, for example:

```bash
cd digital-banking-suite

dotnet test src/backend/BankingSuite.Backend.sln \
  /p:CollectCoverage=true \
  /p:CoverletOutput=../../../lcov \
  /p:CoverletOutputFormat=lcov

reportgenerator \
  -reports:lcov.info \
  -targetdir:src/backend/coverage-report \
  "-reporttypes:Html;TextSummary"
```

Open the generated HTML report:

- `src/backend/coverage-report/index.html`

Look for:

- **BankingSuite.IAM.Api**
- **BankingSuite.IAM.Application**
- **BankingSuite.IAM.Domain**

Your goal in this chapter is to:

- Identify **uncovered IAM code paths**
- Add or refine integration tests until IAM coverage approaches or exceeds **80%**

Once tests are passing and coverage is where you want it:

```bash
git status

git add tests/BankingSuite.IAM.IntegrationTests/**

git commit -m "Chapter 08: add IAM integration tests and increase coverage"
git push --set-upstream origin part2-chapter08-iam-integration-tests-and-coverage
```

At this point, Chapter 08 is complete:

- `08-01`: Admin employee management
- `08-02`: Email confirmation & password flows
- `08-03`: 2FA & security hardening
- `08-04`: Integration tests to push IAM coverage toward 80%+

In the next major part of the book, we’ll move to the **Angular 21 + Nx frontend**, building a full IAM UI that consumes these endpoints and behaves like a genuine banking employee portal.
