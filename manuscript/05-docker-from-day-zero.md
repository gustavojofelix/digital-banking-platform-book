# Chapter 4 — Docker from Day Zero

In this chapter, we stop talking _about_ containers and actually start running our Banking Suite _inside_ containers.

By the end of this chapter, you will:

- Understand why containers are non-negotiable for a modern banking platform.
- Build and run a minimal backend service in Docker.
- Create a reusable multi-stage Dockerfile for .NET 9 microservices.
- Wire up a first `docker-compose.yml` to run the API + database together.
- Adopt simple patterns you’ll reuse for every service in this book.

We’ll focus on one service as a concrete example: **AccountService**.

I> In this chapter, **AccountService is a minimal placeholder service** used purely to demonstrate Docker and basic infrastructure patterns. In later chapters—_after we design IAM and Customer services properly_—we’ll come back and rebuild AccountService as the real, domain-rich service that depends on them.

---

## 4.1 Git Setup for This Chapter (Feature Branch)

Before we write any code or YAML, we follow the same rule as in previous chapters:

> **Each build chapter gets its own feature branch.**  
> We only merge back to `develop` after the chapter is complete and CI is green.

For this chapter we’ll use:

- Branch name: `feature/ch04-docker-from-day-zero`

From your local clone:

```bash
cd digital-banking-suite

# Make sure you're on develop and up to date
git checkout develop
git pull origin develop

# Create a new branch for this chapter
git checkout -b feature/ch04-docker-from-day-zero
```

All the changes you make in this chapter (code, Dockerfiles, compose files, docs) should happen on this branch.

T> Keeping a separate branch per chapter makes it easy to review your work, roll back if needed, and see CI runs per chapter in GitHub.

---

## 4.2 Why Containers for a Digital Banking Platform?

You could technically build this whole system without containers.

You’d also:

- Argue with “it works on my machine”.
- Spend days recreating environments for new developers.
- Fight version mismatches between your laptop, test servers, and production.

Banking systems are:

- **Distributed** – multiple services interacting across the network.
- **Integration-heavy** – databases, message brokers, API gateways, etc.
- **Sensitive** – you want predictable, repeatable deployments.

Containers give us:

- **Consistency** – “runs the same way on every machine.”
- **Isolation** – each microservice has its own runtime, dependencies, and environment.
- **Reproducible environments** – `docker compose up` is enough for a new developer to see the system running.
- **Cloud-readiness** – whether you deploy to Kubernetes, Azure Container Apps, AWS ECS, or something else, containers are the standard unit of deployment.

I> We’ll use **Docker** as the container runtime. Everything we do here will work in your local machine and in most cloud container platforms.

---

## 4.3 Assumptions & Prerequisites

By now you should already have (from Chapter 3):

- **Docker Desktop** (Windows/macOS) or **Docker Engine** (Linux) installed.
- The `digital-banking-suite` repository created with the basic folder structure:

```text
digital-banking-suite/
README.md
src/
  backend/
  frontend/
infra/
scripts/
docs/
.github/
  workflows/
```

Let’s quickly verify Docker:

```bash
docker --version
docker info
```

---

## 4.4 Our First Service in a Container (Placeholder AccountService)

We’ll start by creating a **minimal .NET 9 Web API** for `AccountService`. It won’t contain real banking logic yet – this chapter’s focus is on **containerization**, not domain rules or service dependencies.

In the real system:

- AccountService will **depend on**:
  - **IAMService** (for authenticated users and roles), and
  - **CustomerService** (for customer profiles).
- We’ll design and implement IAM and Customer **first**, then come back to give AccountService real behavior, aggregates, and invariants.

For now, you can think of this AccountService as:

> “A thin placeholder API whose job is to help us learn how to build and run services in Docker.”

From the root of your repo:

```bash
cd digital-banking-suite

# Create a folder for AccountService
mkdir -p src/backend/AccountService

# Create a minimal Web API in that folder
dotnet new webapi \
  -n AccountService.Api \
  -o src/backend/AccountService/AccountService.Api
```

This will generate a basic .NET Web API with a default `Program.cs`, controllers, and HTTPS setup.

Let’s do a quick sanity check (still without Docker):

```bash
cd src/backend/AccountService/AccountService.Api
dotnet run
```

Open your browser at `https://localhost:5001` or whatever URL the console shows. You should see the default API response or Swagger (depending on the template version).

Once you’re satisfied that:

- The project builds
- The API runs

…press `Ctrl + C` to stop the app. Now we’re ready to move it into a container.

I> Later chapters will **replace this placeholder behavior** with the real AccountService: accounts belonging to customers, protected by IAM, backed by PostgreSQL, and integrated with Transactions and Notifications.

---

## 4.5 Writing a Multi-Stage Dockerfile for .NET 9

We’ll use a **multi-stage Dockerfile**, which:

- Uses a **SDK image** to restore, build, and publish the app.
- Uses a **lighter runtime image** to actually run it.

This gives us smaller, more secure images.

Create a file at:

`src/backend/AccountService/AccountService.Api/Dockerfile`

with the following content:

```dockerfile
# ===========================
# 1. Build stage
# ===========================
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy csproj and restore as distinct layers
COPY ["AccountService.Api.csproj", "./"]
RUN dotnet restore "AccountService.Api.csproj"

# Copy the rest of the source code
COPY . .

# Build the project
RUN dotnet build "AccountService.Api.csproj" -c Release -o /app/build

# ===========================
# 2. Publish stage
# ===========================
FROM build AS publish
RUN dotnet publish "AccountService.Api.csproj" -c Release -o /app/publish /p:UseAppHost=false

# ===========================
# 3. Runtime stage
# ===========================
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app

# Set a non-root user if desired (good practice for production)
# USER app

COPY --from=publish /app/publish .
EXPOSE 8080

# ASP.NET Core defaults to port 8080 in some container scenarios;
# adjust ASPNETCORE_URLS if needed.
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "AccountService.Api.dll"]
```

A few important notes:

- We keep the Dockerfile **close to the API project** so each service can have its own.
- We explicitly set `EXPOSE 8080` and `ASPNETCORE_URLS`. We’ll map this port from Docker later.
- We use `UseAppHost=false` to keep the image lighter (no platform-specific app host).

T> This Dockerfile will become the **template** for our other backend services. Later, we’ll factor out common patterns, but for now it’s okay to keep one Dockerfile per service.

---

## 4.6 Building & Running the AccountService Container

From the `AccountService.Api` folder:

```bash
cd src/backend/AccountService/AccountService.Api

# Build the Docker image
docker build -t banking-suite/account-service:dev .
```

If the build succeeds, you’ll see something like:

> Successfully built \<image-id>  
> Successfully tagged banking-suite/account-service:dev

Now run the container:

```bash
docker run --rm -p 5000:8080 banking-suite/account-service:dev
```

What this does:

- `--rm` → remove the container when it stops.
- `-p 5000:8080` → map port 5000 on your host to 8080 inside the container.
- `banking-suite/account-service:dev` → the image we just built.

Open `http://localhost:5000/swagger` (or the base URL, depending on the template). You should see the API running **inside a container**.

Hit `Ctrl + C` to stop the container.

I> From now on, whenever you see `docker run -p 5000:8080 banking-suite/account-service:dev`, you’ll know that you’re running the exact same environment that will run in staging or production.

---

## 4.7 Dockerfile Template for Other Microservices

We don’t want to reinvent the Dockerfile every time. Here’s the **generic pattern** we’ll reuse:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

COPY ["<ServiceName>.Api.csproj", "./"]
RUN dotnet restore "<ServiceName>.Api.csproj"

COPY . .
RUN dotnet build "<ServiceName>.Api.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "<ServiceName>.Api.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "<ServiceName>.Api.dll"]
```

When you scaffold `IAMService.Api`, `CustomerService.Api`, etc., you’ll:

- Copy this Dockerfile.
- Replace `<ServiceName>` with the actual project name.

Later in the book, we’ll:

- Introduce **environment-specific configuration** (dev/staging/prod).
- Inject connection strings and secrets using environment variables + secret managers.
- Add **health checks** and other Docker awareness.

---

## 4.8 Introducing docker-compose for the Banking Suite

Running one service with `docker run` is okay, but we’re building a **suite**:

- AccountService
- IAMService
- CustomerService
- TransactionService
- NotificationService
- PostgreSQL
- RabbitMQ
- API Gateway

We’ll orchestrate these with **Docker Compose**.

For this chapter, we’ll start with a **minimal compose file** that runs:

- `account-service`
- `postgres` (for future use; we’ll wire it in the next chapters)

Create `infra/docker-compose.dev.yml`:

```yaml
version: "3.9"

services:
  account-service:
    image: banking-suite/account-service:dev
    container_name: account-service
    build:
      context: ../src/backend/AccountService/AccountService.Api
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: "Development"
      # Placeholder DB connection string, will be used later
      ConnectionStrings__DefaultConnection: "Host=postgres;Port=5432;Database=accounts_db;Username=accounts_user;Password=accounts_password"
    depends_on:
      - postgres

  postgres:
    image: postgres:16
    container_name: postgres-accounts
    restart: always
    environment:
      POSTGRES_DB: "accounts_db"
      POSTGRES_USER: "accounts_user"
      POSTGRES_PASSWORD: "accounts_password"
    ports:
      - "5432:5432"
    volumes:
      - accounts_pgdata:/var/lib/postgresql/data

volumes:
  accounts_pgdata:
```

A few important points:

- `build` → Compose will build the image from the `AccountService.Api` folder using the `Dockerfile` we wrote.
- `depends_on` → ensures Postgres starts before AccountService (not a health check, just order).
- `environment` → we’re already using the standard `ConnectionStrings:DefaultConnection` pattern via `__` double underscores.

T> Even though we’re not yet connecting AccountService to PostgreSQL, this setup prepares the ground. In future chapters we’ll add Entity Framework Core, migrations, and actual database operations.

To run this stack:

```bash
cd infra
docker compose -f docker-compose.dev.yml up --build
```

Then open:

- `http://localhost:5000/swagger` → AccountService.
- The Postgres container is also running and ready, even if the API doesn’t use it yet.

To stop:

```bash
docker compose -f docker-compose.dev.yml down
```

---

## 4.9 Handling Configuration & Secrets (The Right Way Later)

In this chapter, we’ve hard-coded values like:

- Database names.
- Usernames and passwords.

This is **acceptable only for local development** and learning.

In a real banking system, we’ll:

- Move secrets to environment variables or secret stores (Azure Key Vault, AWS Secrets Manager, etc.).
- Use **per-environment configuration files**.
- Introduce **Docker secrets** or cloud-specific secret mechanisms.

For now, our rules are:

- Hard-coded secrets are allowed **only** inside `docker-compose.dev.yml` and only for your local machine.
- Never commit real production secrets to the repository.
- Later we will refactor these into a proper configuration strategy.

W> Never re-use these sample passwords for anything outside your local dev environment. Treat them as intentionally weak and disposable.

---

## 4.10 Common Docker Commands You’ll Use Constantly

Here’s your **day-to-day cheat sheet**:

Build an image manually:

```bash
docker build -t banking-suite/account-service:dev src/backend/AccountService/AccountService.Api
```

Run a single container:

```bash
docker run --rm -p 5000:8080 banking-suite/account-service:dev
```

See running containers:

```bash
docker ps
```

See all containers (including stopped):

```bash
docker ps -a
```

Stop and remove containers created by compose:

```bash
cd infra
docker compose -f docker-compose.dev.yml down
```

Remove dangling images:

```bash
docker image prune
```

View logs from a service in Compose:

```bash
cd infra
docker compose -f docker-compose.dev.yml logs -f account-service
```

T> When something doesn’t work, your first stops should be:  
T> `docker ps` (is it running?) and `docker logs <container-name>` (what went wrong?).

---

## 4.11 Wrap-up: PR, CI & Merge to `develop`

At this point you should have, on the branch  
`feature/ch04-docker-from-day-zero`:

- The placeholder `AccountService.Api` project.
- A working multi-stage `Dockerfile`.
- The `infra/docker-compose.dev.yml` file.
- This chapter’s markdown file (if you keep docs in the repo).

Now we finish the Git workflow for the chapter.

### 4.11.1 Committing Your Changes

Stage and commit your work in logical chunks. For example:

```bash
git add src/backend/AccountService/AccountService.Api/*
git commit -m "feat(docker): add placeholder AccountService.Api"

git add infra/docker-compose.dev.yml
git commit -m "chore(docker): add dev docker-compose stack"

git add docs/04-docker-from-day-zero.md
git commit -m "docs(ch04): document Docker from day zero"
```

T> You don’t need to perfectly match these messages, but keep them **clear and scoped**. Avoid “misc changes” or “stuff”.

### 4.11.2 Pushing and Opening a Pull Request

Push the branch to GitHub:

```bash
git push -u origin feature/ch04-docker-from-day-zero
```

Then, in GitHub:

1. Open a Pull Request from  
   `feature/ch04-docker-from-day-zero` → `develop`
2. Give it a clear title, for example:  
   **"Chapter 4 – Docker from Day Zero"**
3. Briefly describe what this PR adds:
   - Placeholder AccountService
   - Dockerfile and docker-compose.dev.yml
   - Documentation for Chapter 4

### 4.11.3 Watching the CI Pipeline

When you open the PR, the CI workflow (`.github/workflows/ci.yml`) should run automatically.

At this stage, CI might only:

- Restore dependencies.
- Build the solution (or individual projects).

Later in the book we will enhance CI to:

- Build Docker images.
- Run unit and integration tests.
- Optionally run formatters and linters.

W> Do not merge into `develop` if the pipeline is red. Fix the issue (build errors, missing files, etc.), push again, and wait for CI to pass.

### 4.11.4 Merging to `develop`

Once the pipeline is green:

1. Merge the PR into `develop`.
2. Optionally delete the feature branch:

   - in GitHub (Delete branch button), and/or
   - locally:

   ```bash
   git checkout develop
   git pull origin develop
   git branch -d feature/ch04-docker-from-day-zero
   ```

At this point:

- `develop` contains all Chapter 4 changes.
- CI has already validated the chapter’s code and Docker setup.
- You are ready to start the next chapter from a clean, green baseline.

---

## 4.12 Summary & What’s Next

In this chapter, we:

- Created our **first backend service** (`AccountService.Api`) as a **placeholder** to demonstrate containerization.
- Wrote a **multi-stage Dockerfile** for .NET 9, which we’ll reuse across microservices.
- Built and ran the service in a container using `docker build` and `docker run`.
- Introduced a minimal **Docker Compose** setup (`docker-compose.dev.yml`) to orchestrate:
  - `account-service`
  - `postgres` (as a future database backend)
- Practiced the **per-chapter Git workflow**:
  - Create branch → implement → push → PR → CI → merge to `develop`.

Remember:

I> This is not the final, domain-complete AccountService. In later chapters, after we’ve built **IAMService** and **CustomerService**, we’ll come back and turn AccountService into the real thing: accounts owned by customers, protected by IAM, persisted in PostgreSQL, and integrated with Transactions and Notifications.

In the next chapters of Part II, we will:

- Wire in **testing** (unit and integration tests that can run against containers).
- Evolve our **CI pipeline** to build and test these images on every push.
- Introduce a more complete **backend solution structure** and a shared kernel for domain-driven design.

From this point on, think of Docker as your default runtime. If the Banking Suite isn’t running in containers, it’s not really running.
