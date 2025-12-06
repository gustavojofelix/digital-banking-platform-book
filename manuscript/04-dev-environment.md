# Chapter 03 — Development Environment, Repository Setup & Branch Strategy

In Chapters 01–02 we defined the **vision** and **domain architecture** for Alvor Bank – Banking Suite.

Now it’s time to prepare our **development environment** and create the **main source-code repository** where all backend, frontend and infrastructure code will live.

By the end of this chapter you will have:

- All core tools installed and verified (`.NET 10`, `Angular 21`, `Nx`, `Node`, `Git`, `Docker`).
- A clean **folder structure** for the project.
- A **GitHub repository** called `digital-banking-suite` initialised and pushed.
- A **branch strategy** (`main`, `develop`, `feature/chXX-*`) that we will use in every chapter.
- The first **Git tag** representing the state of the repo at the end of this chapter.

From the next chapters onward we will start adding real code into this structure.

---

## 3.1 Tools you need

Make sure you have the following tools installed before you proceed.

We don’t go into OS-specific installation steps here; follow the official documentation for your platform, then use the commands below to verify.

### 3.1.1 .NET 10 SDK

Install the **.NET 10 SDK** (for example `10.0.100` or higher).

Verify:

```bash
dotnet --version
```

You should see something like:

```text
10.0.100
```

Also check basic runtime info:

```bash
dotnet --info
```

We will target the `net10.0` framework in all backend projects.

---

### 3.1.2 Node.js and npm

Install a recent **Node.js LTS** (for Angular 21, Node 20 LTS is a good baseline).

Verify:

```bash
node -v
npm -v
```

Expected output (your exact versions may differ):

```text
v20.x.x
10.x.x
```

---

### 3.1.3 Angular CLI 21

Install **Angular CLI 21** globally (or use `npx` later if you prefer local tools).

```bash
npm install -g @angular/cli@21
```

Verify:

```bash
ng version
```

Make sure the global Angular CLI version shows `21.x.x`.

---

### 3.1.4 Nx CLI

We’ll use **Nx** to manage the Angular monorepo.

Install Nx globally:

```bash
npm install -g nx
```

Verify:

```bash
nx --version
```

You should see a recent Nx version (e.g. `20.x.x` or similar).

---

### 3.1.5 Git

We’ll use **Git** for version control and **GitHub** as the remote.

Verify:

```bash
git --version
```

You should see something like:

```text
git version 2.x.x
```

---

### 3.1.6 Docker

We’ll run PostgreSQL, RabbitMQ and all microservices in **Docker containers**.

Install **Docker Desktop** (on Windows/macOS) or Docker Engine (on Linux).

Verify:

```bash
docker --version
docker compose version
```

Make sure both commands succeed.

---

### 3.1.7 IDEs and tools

You’ll need at least one of:

- **Visual Studio 2026** (or latest Visual Studio) with:
  - .NET development workload
  - Docker tools
- **VS Code** with:
  - C# extension (for .NET 10)
  - Angular/TypeScript extensions
  - Docker extension

And optionally:

- **Postman** (or equivalent HTTP client) — for manual and automated API testing later.

---

## 3.2 Repository strategy

We’ll use **separate repositories** for the **book** and the **code**:

1. **Book repository**

   - Contains this manuscript in Markdown and any diagrams.
   - You might call it `digital-banking-suite-book` or similar.

2. **Main source-code repository**

   - Contains all backend, frontend, infrastructure and tests.
   - We’ll call it `digital-banking-suite` throughout the book.

3. **Chapter snapshots (tags)**
   - Instead of many permanent feature branches, we’ll use **Git tags** like `chapter-03-env-and-repo`, `chapter-07-iam-backend`, etc.
   - This lets you (and readers) check out the exact repo state at the end of each chapter.

In this chapter we focus on the **main source-code repository**.

---

## 3.3 Creating the main source-code repository

Choose a folder where you keep your development projects, for example:

```text
C:\Projects\Alvordev\
~/Projects/
```

Create and initialise the main repo:

```bash
cd /path/to/your/projects

mkdir digital-banking-suite
cd digital-banking-suite

git init
```

> Throughout the book we’ll assume the root path is `digital-banking-suite/`.

---

## 3.4 Initial folder structure

Create a minimal folder structure to keep backend, frontend and infrastructure organised.

From the `digital-banking-suite` root:

```bash
mkdir -p src/backend
mkdir -p src/frontend
mkdir -p infra
mkdir -p tests
mkdir -p postman
```

Your tree now looks like:

```text
digital-banking-suite/
├─ src/
│  ├─ backend/
│  └─ frontend/
├─ infra/
├─ tests/
└─ postman/
```

We’ll add more subfolders as we start building microservices and frontends.

---

## 3.5 Basic repo housekeeping (.gitignore, README)

Before we push anything, let’s add a basic `.gitignore` and `README.md`.

### 3.5.1 .gitignore

Create `.gitignore` at the root:

```bash
touch .gitignore
```

Open it in your editor and add:

```text
# .NET build artifacts
bin/
obj/

# Visual Studio / VS Code
.vs/
.vscode/

# Node / Angular
node_modules/
dist/
tmp/

# Coverage reports
coverage/
**/coverage-report/

# Docker
**/*.log

# OS files
.DS_Store
Thumbs.db
```

We’ll add more entries later if needed (e.g., for secrets files).

---

### 3.5.2 README.md

Create a simple `README.md`:

```bash
touch README.md
```

Add:

```markdown
# Alvor Bank – Banking Suite

This repository contains the source code for **Alvor Bank – Banking Suite**, the sample system built in the book:

> **Build a Real Digital Bank: Banking Suite with .NET 10 & Angular 21**

Tech stack:

- .NET 10, C# 14
- ASP.NET Core microservices with Clean Architecture and DDD
- Angular 21 + Nx monorepo (back-office and customer portals)
- PostgreSQL, RabbitMQ, Docker
- GitHub Actions for CI/CD

Folder layout:

- `src/backend` – backend services (IAM, Customer, Account, Transaction, Notification)
- `src/frontend` – Angular 21 + Nx workspace
- `infra` – docker-compose and infrastructure scripts
- `tests` – unit, integration and API tests
- `postman` – Postman collections and environments
```

This will evolve as we add more components.

---

## 3.6 Branch strategy

We’ll use a **lightweight GitFlow-style** branch strategy:

- `main`

  - Always points to the latest **stable, tagged** version.
  - Represents “book-complete” snapshots.

- `develop`

  - Integration branch where we merge chapter feature branches.
  - CI runs here on every push and PR.

- `feature/chXX-short-description`
  - Short-lived branches per chapter or feature.
  - Example: `feature/ch03-env-and-repo`, `feature/ch07-iam-backend`.

### 3.6.1 Create `main` and `develop` branches

From your `digital-banking-suite` root:

```bash
# We are currently on the default branch created by 'git init'
git add .
git commit -m "ch03: initial repository structure"

git branch -M main
git checkout -b develop
```

Now `main` holds the very first commit, and `develop` is ready for active work.

---

## 3.7 Connecting to GitHub

Create a new empty repository on GitHub called **`digital-banking-suite`** (no README, no .gitignore — we already have them locally).

Then add the remote and push:

```bash
git remote add origin git@github.com:<your-username>/digital-banking-suite.git

# Push both main and develop
git push -u origin main
git push -u origin develop
```

Replace `<your-username>` with your actual GitHub username (or the organisation name).

---

## 3.8 Workflow per chapter

From now on, every chapter in this book will follow a **repeatable workflow**:

1. **Create a feature branch** from `develop`.
2. **Make changes** (code, tests, infra) described in the chapter.
3. **Run tests and coverage** locally (once we have projects).
4. **Commit regularly** with meaningful messages.
5. **Open a Pull Request** from the feature branch into `develop`.
6. Let **CI** run and pass.
7. **Merge** the PR.
8. **Tag** the resulting commit to mark the chapter end state.

### 3.8.1 Creating the feature branch for this chapter

We’re already on `develop`, but let’s simulate the pattern we’ll follow from now on.

From `develop`:

```bash
git checkout -b feature/ch03-env-and-repo
```

For this chapter we’ve already created the basic structure, so we just commit and merge as if we had been working on this branch from the start.

Stage and commit (if there are any remaining changes):

```bash
git status        # check if anything changed
git add .
git commit -m "ch03: setup environment folders and git basics"
```

Push the feature branch:

```bash
git push -u origin feature/ch03-env-and-repo
```

Then on GitHub:

- Open a **Pull Request** from `feature/ch03-env-and-repo` into `develop`.
- There is not much yet, but this sets the habit early.
- Merge the PR.

After merging, sync `develop` locally:

```bash
git checkout develop
git pull
```

---

## 3.9 Tagging the chapter

To help readers, I will create a **Git tag** for the state of the repository at the end of this chapter.

From `develop`:

```bash
git tag -a chapter-03-env-and-repo -m "Chapter 03: environment, repo structure and branch strategy"
git push origin chapter-03-env-and-repo
```

Later chapters will tell readers:

> “If you get stuck, you can check out tag `chapter-03-env-and-repo` to see the canonical state of the repo at the end of this chapter.”

---

## 3.10 Sanity checks

Before moving on, run a few sanity checks to ensure all tools are available.

From anywhere:

```bash
dotnet --version
node -v
npm -v
ng version
nx --version
docker --version
docker compose version
git --version
```

You should now have:

- A working `.NET 10` SDK.
- Node + npm installed.
- Angular CLI 21 and Nx CLI installed.
- Docker available.
- Git connected to your GitHub repo.

Your `digital-banking-suite` repository is ready to host:

- Backend microservices in `src/backend/`.
- Angular 21 Nx workspace in `src/frontend/`.
- Infrastructure (including PostgreSQL and RabbitMQ) in `infra/`.
- Tests in `tests/`.
- Postman collections in `postman/`.

---

## 3.11 What’s next

In the next chapters we will:

- Introduce **Docker from day one** for local infrastructure.
- Set up **GitHub Actions CI** that runs `dotnet test`, coverage and (later) frontend builds.
- Create the first shared **BuildingBlocks** projects in the backend.
- Scaffold the skeleton of the **IAM service**, which will become our reference template for all other microservices.

You now have the **foundation** in place: tools, repository and workflow.

In Chapter 04 we’ll start putting containers and pipelines around this skeleton so that every piece of code we add later will run in a **repeatable, production-like environment**.
