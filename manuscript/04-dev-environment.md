# Chapter 3 — Development Environment, Repository Setup & Branch Strategy

In the previous chapters, you designed the **vision** and **domain architecture** for the Banking Suite.  
In this chapter, we make everything concrete on your machine:

- Install the core tools (.NET 9, Node.js, Angular, Nx, Docker, Git)
- Create the **main GitHub repository** for the Banking Suite
- Define a **branch strategy** you’ll use throughout the book
- Establish **coding standards & commit conventions**
- Add simple **developer onboarding scripts** so new team members can get started quickly

By the end of this chapter, you’ll have a working project skeleton with version control, agreed rules, and a repeatable setup — just like a real team would.

---

## 3.1 Chapter Goals

After this chapter, you should:

- Have all required tools installed and verified.
- Have a **GitHub repository** created and linked to your local clone.
- Have a **base folder structure** for the Banking Suite.
- Understand and apply a **lightweight Git workflow** (`main`, `develop`, `feature/*`).
- Use consistent **coding standards and commit messages** from day one.
- Be able to run a **single onboarding script** to prepare the environment.

You won’t write much domain logic yet — this chapter is about building a foundation you can trust as the system grows.

## 3.2 Tooling Checklist

Before creating the repository, install the following tools in the **same major versions** used in this book.  
Newer _patch_ versions (for example `9.0.302` instead of `9.0.301`) are usually fine, but try to stay on the same major line.

### Required tools and versions

- **.NET 9 SDK — 9.0.301**

  - Download: <https://dotnet.microsoft.com/en-us/download/dotnet/9.0>

- **Node.js — 24.11.1 (LTS)**

  - Download: <https://nodejs.org/en>
  - Make sure you choose the **LTS** build, not the “Current” release.

- **Angular CLI — 21.0.1**

  - Install globally after Node.js is installed:
    ```bash
    npm install -g @angular/cli@21.0.1
    ```

- **Nx CLI — 21.5.1**

  - Install globally:
    ```bash
    npm install -g nx@21.5.1
    ```

- **Docker Desktop — tested with 4.41.x (or newer stable)**

  - Download for your OS: <https://docs.docker.com/get-started/introduction/get-docker-desktop/>

- **Git — 2.52.0**

  - Download: <https://git-scm.com/downloads>

- **Editor / IDE**
  - Any modern editor works; examples in this book use **Visual Studio Code** and **JetBrains Rider**.

### Verifying your installation

After installing everything, open a terminal (PowerShell, bash, etc.) and run:

```bash
dotnet --version
node --version
npm --version
ng version
nx --version
git --version
docker --version
```

## 3.3 Creating the Banking Suite Repository

In this section, we’ll create the GitHub repository that will hold all source code for the **Digital Banking Suite**.  
This is the repo you’ll use for the rest of the book.

We’ll:

- Create a new empty repository on GitHub.
- Initialize a local repository and connect it to GitHub.
- Add a minimal `README.md` so the repo is not empty.

You can choose any name you like; in this chapter we’ll assume:

- Repository name: `digital-banking-suite`

### 3.3.1 Create the GitHub Repository

1. Go to GitHub and click **New repository**.
2. Set:
   - **Repository name:** `digital-banking-suite`
   - **Visibility:** public or private (your choice)
3. Leave the options for README, `.gitignore`, and license **unchecked**.
4. After the repo is created, copy the repository URL (SSH or HTTPS), for example:
   - `git@github.com:your-username/digital-banking-suite.git`
   - or `https://github.com/your-username/digital-banking-suite.git`

### 3.3.2 Initialize the Local Repository

Choose a parent folder on your machine (for example `C:\dev` or `~/dev`) and run:

```bash
mkdir digital-banking-suite
cd digital-banking-suite
git init -b main
git remote add origin <your-repo-url>
```

- `git init -b main` initializes a new Git repo with `main` as the default branch.
- `git remote add origin ...` tells Git where your GitHub repository lives.

Next, create a minimal `README.md` so the repository isn’t empty:

```bash
echo "# Digital Banking Suite" > README.md
```

Stage, commit, and push the file:

```bash
git add README.md
git commit -m "chore: initial repository"
git push -u origin main
```

At this point you have:

- A `main` branch on GitHub containing a basic README.
- A local clone wired to that remote.
- A clean starting point for the rest of the chapter.

## 3.4 Base Folder Structure

Before we add any services, we set up a **high-level structure** that will grow with the platform.

From the root of `digital-banking-suite`, create this structure:

```bash
mkdir -p src/backend
mkdir -p src/frontend
mkdir -p infra
mkdir -p scripts
mkdir -p docs
mkdir -p .github/workflows
```

Your folder tree should now look like:

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

We’ll use these folders for:

- `src/backend` — .NET 9 microservices, shared libraries, and API Gateway
- `src/frontend` — Angular + Nx workspace
- `infra` — Docker Compose files, database/messaging definitions, infrastructure scripts
- `scripts` — onboarding and helper scripts
- `docs` — architecture notes, diagrams, ADRs (Architecture Decision Records)
- `.github/workflows` — CI pipelines

Commit the structure:

```bash
git add .
git commit -m "chore: scaffold repository structure"
git push
```

## 3.5 Branch Strategy: main, develop, and feature branches

To keep the project organized, we’ll use a simple, battle-tested branching model:

- `main` — stable, release-ready code.
- `develop` — integration branch for ongoing work.
- `feature/*` — short-lived branches for specific tasks or chapters.

### 3.5.1 Create the `develop` Branch

From your local repo:

```bash
git checkout -b develop
git push -u origin develop
```

From now on:

- You branch off `develop` to work on new features.
- You open a Pull Request from `feature/*` → `develop`.
- When a milestone is complete and green, you merge `develop` → `main`.

### 3.5.2 Chapter-Based Feature Branches

Since this is a book-based project, it’s useful to organize work by chapter:

- `feature/ch03-dev-environment`
- `feature/ch04-domain-architecture`
- `feature/ch05-security-basics`
- `...`

For this chapter, create a branch:

```bash
git checkout develop
git checkout -b feature/ch03-dev-environment
```

You’ll do all Chapter 3 work on this branch, then:

- Commit regularly
- Open a PR into `develop` when done
- Merge once you’re satisfied or once all tests pass

I> **Note:** Even if you’re working alone, treat yourself like a team: use branches, write PR descriptions, and keep a clean history. It pays off later.

---

## 3.6 Coding Standards & Commit Conventions

Before writing serious code, we agree on how that code and history should look.

### 3.6.1 C# / .NET Coding Standards

You can evolve this over time, but here’s a good starting set:

- Use **PascalCase** for classes, methods, and public properties.
- Use **camelCase** for local variables and private fields (optionally `_camelCase` for private fields).
- Prefer **expression-bodied members** where clear, but not at the cost of readability.
- Avoid “god classes” and static helper bags — prefer **domain-driven entities and services**.
- Keep files focused: one class per file, unless types are tightly coupled.

Add a minimal `.editorconfig` at the repo root:

```ini
root = true

[*.cs]
indent_style = space
indent_size = 4
charset = utf-8-bom
end_of_line = lf
insert_final_newline = true
dotnet_sort_system_directives_first = true
dotnet_style_qualification_for_field = false
dotnet_style_qualification_for_property = false
dotnet_style_qualification_for_method = false
dotnet_style_qualification_for_event = false
```

This ensures consistent formatting across editors.

### 3.6.2 TypeScript / Angular Conventions

- Use **TypeScript strict mode**.
- Use **PascalCase** for components and services; **camelCase** for variables and functions.
- Keep modules/components small and focused.
- Use **feature-based folder structure** within the Angular workspace (we’ll set this up later).

### 3.6.3 Commit Message Style (Conventional Commits)

We’ll use a slightly simplified **Conventional Commits** style:

- `feat:` — a new feature
- `fix:` — a bug fix
- `chore:` — repo/config changes (no behavior)
- `docs:` — documentation updates
- `test:` — adding or updating tests
- `refactor:` — change that improves code without changing behavior

Examples:

```text
feat: add base folder structure for backend services
chore: configure editorconfig for csharp and typescript
docs: explain branch strategy in README
```

W> **Warning:** Avoid commits like `update stuff` or `misc changes`.  
Worse than no commit history is a **meaningless** one.

---

## 3.7 Developer Onboarding Scripts

Real projects avoid “it works on my machine” by giving new developers a **single command** that:

- Checks prerequisites (or at least logs them)
- Restores backend and frontend dependencies
- Prepares local infrastructure (e.g., Docker Compose)

We’ll start simple and evolve these scripts as the project grows.

### 3.7.1 Script Layout

Create two scripts in the `scripts` folder:

- `scripts/bootstrap.ps1` — for Windows / PowerShell
- `scripts/bootstrap.sh` — for macOS / Linux

`scripts/bootstrap.ps1` (PowerShell)

```bash
Write-Host "=== Digital Banking Suite: Developer Bootstrap (PowerShell) ==="

Write-Host "`n[1/3] Checking basic tool availability..."
dotnet --version
node --version
npm --version
git --version
docker --version

Write-Host "`n[2/3] Restoring backend and frontend dependencies (if projects exist)..."

if (Test-Path "./src/backend") {
    Write-Host "-> Backend folder found, running dotnet restore..."
    dotnet restore ./src/backend 2>$null
} else {
    Write-Host "-> Backend folder not found yet. Will be created in later chapters."
}

if (Test-Path "./src/frontend") {
    Write-Host "-> Frontend folder found, running npm install (via Angular/Nx workspace)..."
    Push-Location ./src/frontend
    npm install
    Pop-Location
} else {
    Write-Host "-> Frontend folder not found yet. Will be created in later chapters."
}

Write-Host "`n[3/3] Starting local infrastructure with Docker Compose (if defined)..."

if (Test-Path "./infra/docker-compose.yml") {
    docker compose -f ./infra/docker-compose.yml up -d
} else {
    Write-Host "-> No docker-compose.yml found yet. Infrastructure will be added later."
}

Write-Host "`nBootstrap complete. You are ready to follow the next chapters."

```

`scripts/bootstrap.sh` (bash)

```bash
#!/usr/bin/env bash
set -e

echo "=== Digital Banking Suite: Developer Bootstrap (bash) ==="

echo
echo "[1/3] Checking basic tool availability..."
dotnet --version || echo "dotnet not found"
node --version || echo "node not found"
npm --version || echo "npm not found"
git --version || echo "git not found"
docker --version || echo "docker not found"

echo
echo "[2/3] Restoring backend and frontend dependencies (if projects exist)..."

if [ -d "./src/backend" ]; then
 echo "-> Backend folder found, running dotnet restore..."
 dotnet restore ./src/backend || true
else
 echo "-> Backend folder not found yet. Will be created in later chapters."
fi

if [ -d "./src/frontend" ]; then
 echo "-> Frontend folder found, running npm install (via Angular/Nx workspace)..."
 cd ./src/frontend
 npm install || true
 cd - > /dev/null
else
 echo "-> Frontend folder not found yet. Will be created in later chapters."
fi

echo
echo "[3/3] Starting local infrastructure with Docker Compose (if defined)..."

if [ -f "./infra/docker-compose.yml" ]; then
 docker compose -f ./infra/docker-compose.yml up -d
else
 echo "-> No docker-compose.yml found yet. Infrastructure will be added later."
fi

echo
echo "Bootstrap complete. You are ready to follow the next chapters."

```

Make the bash script executable:

```bash
chmod +x scripts/bootstrap.sh
```

From now on, a new team member can:

```bash
git clone <your-repo-url>
cd digital-banking-suite
./scripts/bootstrap.sh   # or: pwsh ./scripts/bootstrap.ps1
```

## 3.8 A Minimal CI Pipeline Skeleton

We’ll go deep into CI/CD later, but it’s useful to wire a **very simple pipeline** now so that every PR is validated.

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
    branches: [develop]
  push:
    branches: [develop]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "9.0.x"

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "lts/*"

      - name: Restore backend (if any)
        run: |
          if [ -d "./src/backend" ]; then
            dotnet restore ./src/backend
          else
            echo "No backend projects yet."
          fi

      - name: Restore frontend (if any)
        run: |
          if [ -d "./src/frontend" ]; then
            cd src/frontend
            npm install
          else
            echo "No frontend workspace yet."
          fi
```

Right now this pipeline mainly ensures:

- The repo checks out correctly
- The tools install successfully
- Restore steps work (or are skipped gracefully)

We’ll add real **build + test + lint + Docker** steps as soon as we create actual projects.

Commit and push:

```bash
git add .github/workflows/ci.yml
git commit -m "chore: add initial CI pipeline skeleton"
git push
```

### 3.8.1 Seeing the Pipeline in Action

Before we move on to the next chapter, let’s actually **see the CI pipeline run**.

1. Make sure all your Chapter 3 work is committed on the feature branch:

```bash
git status
git add .
git commit -m "chore: finish chapter 3 setup"
```

2.  On GitHub, open a **Pull Request**:

    - Base branch: `develop`
    - Compare branch: `feature/ch03-dev-environment`

3.  When you create the PR, GitHub will trigger the **CI workflow** (the `ci.yml` file you added).  
    You should see a “Checks” section appear on the PR and the workflow run.

4.  After the CI checks are **green**, merge the PR into `develop`.

5.  Following are the screenshots of the github steps to create a PR (pretty mucha self explanatory, just follow the seuence of the images, the important sections I have highleted in red square).

        - On Gihub repository of your project select the `Code` tab, make sure you selcted the correct branch, in our case is `feature/ch03-dev-environment` and the click on the `Compare & pull request` green button.

    ![PR Step1](pr1.png) - On the next screen make sure you select base branch `develop` and compare to `feature/ch03-dev-environment` then click on the `Create pull request` green button.
    ![PR Step1](pr2.png) - On the next screen you will see that the pipeline was triggered and that there are no conflicts. Click on the `Merge Pull request`.
    ![PR Step1](pr3.png) - Next step click the `Confirm merge` green button.
    ![PR Step1](pr4.png) - Now when we go back to the reposiotry and select the `Action` tab yous should see the workflows for your pipeline running.
    ![PR Step1](pr5.png) - And finally you can see the details of the pipeline by clicking on the `build` step.
    ![PR Step1](pr6.png) - And...
    ![PR Step1](pr7.png)

From now on, we’ll **keep feature branches as chapter references** instead of deleting them, for example:

- `feature/ch03-dev-environment`
- `feature/ch04-domain-architecture`
- `feature/ch05-security-basics`
- `...`

When you’re ready to start the next chapter, create a new feature branch from `develop`, such as:

```bash
git checkout develop
git pull
git checkout -b feature/ch04-backend-foundation
```

## 3.9 Summary & What’s Next

In this chapter, you:

- Installed and verified the **core toolchain**: .NET 9, Node.js, Angular, Nx, Docker, Git.
- Created the **main GitHub repository** and scaffolded a sensible folder structure.
- Defined a **branching strategy** with `main`, `develop`, and `feature/*`.
- Set up **coding standards and commit conventions** so the codebase stays consistent.
- Added **bootstrap scripts** for easy developer onboarding.
- Wired up a minimal **CI pipeline skeleton** that will grow with the project.

You now have a development environment and repository that feel like a real-world team setup — not just a local folder with random files.

In the next chapter, we’ll start creating the **backend foundation** for our services, beginning with:

- The first .NET solution structure
- Shared building blocks for Clean Architecture
- The initial microservice skeletons that will host our banking domains
