# Preface

The banking world is changing faster than ever.

New digital banks launch every year. Traditional banks modernise decades of legacy systems. Customers expect real-time payments, instant onboarding, mobile everything and rock-solid security.

Behind all of that, there are engineers like you and me—trying to design systems that are **secure**, **reliable**, **testable**, and **change-friendly**.

This book is my way of saying:

> “Let’s stop building TODO apps and _actually_ build the kind of system a real bank could use.”

You are going to build a **real digital banking suite** step by step, using a modern stack:

- **.NET 10 & C# 14** for the backend
- **Angular 21 with Nx** for the frontend
- **Visual Studio 2026 / VS Code** as your tools
- **Docker, GitHub, CI/CD, testing and coverage** from day one

Instead of scattered examples, we’ll grow a **single real project** from a blank repo into a small but production-grade digital banking platform.

---

## Why this book now?

Most resources still teach:

- A single API project with controllers and EF
- Little or no realistic domain modelling
- Maybe a few tests, but no real testing strategy
- DevOps as an afterthought (if it appears at all)

That’s not how banks work in 2026.

Banks care about:

- **Bounded contexts** and clear domain boundaries
- **Microservices & integration patterns**
- **Security, IAM, 2FA, lockouts, auditability**
- **Testing and observability**
- **Reproducible environments** (Docker)
- **Reliable pipelines** (build → test → deploy)

This book exists because I wanted something I could hand to a developer and say:

> “Follow this from start to finish and you’ll understand how to architect, implement, test and ship a realistic banking system.”

And if you’re preparing your career for **2026 and beyond**, that’s exactly the kind of experience you want to have.

---

## What this book is (and isn’t)

This book **is**:

- A **full-stack, end-to-end project**: you will build the backend **and** the frontend **and** the DevOps around it.
- A **microservices + Clean Architecture + DDD** guide, grounded in a single concrete domain: **digital banking**.
- Opinionated about **testing**: we aim for **≥ 80% coverage** on key services and treat failing tests as blockers.
- Written with **real team practices** in mind: branches, PRs, CI, coverage thresholds, code review mindset.

This book is **not**:

- A quick syntax reference for .NET 10 or Angular 21.
- A purely theoretical DDD or architecture book.
- A book that stops after “Hello World” and leaves you to figure out the rest.

You’ll see real code, real trade-offs and realistic constraints.

---

## Who this book is for

This book is for you if:

- You’re a **backend or full-stack .NET developer** who wants to go beyond basic Web API and learn **microservices, Clean Architecture and DDD** in a realistic way.
- You’re an **Angular developer** who wants to understand how frontends fit into a serious backend architecture using **Nx monorepos** and multiple apps.
- You’re a **senior engineer or aspiring architect** who wants a concrete blueprint for **greenfield digital banking systems**.
- You’re actively preparing your **career for 2026** and want hands-on experience with **.NET 10, C# 14, Angular 21, Visual Studio 2026 and modern DevOps**.

You don’t need to be a banking expert.

You do need:

- Familiarity with **C#** and basic web APIs
- Basic understanding of **HTTP, JSON and REST**
- Some experience with **SQL databases**
- Willingness to write tests and follow a structured workflow

If that’s you, you’re in exactly the right place.

---

## What you’ll build: Alvor Bank – Banking Suite

Throughout the book we’ll build **Alvor Bank – Banking Suite**, a fictional but realistic **digital bank** composed of several microservices:

- **IAM Service** – Identity & Access Management for bank employees and customers (users, roles, JWT, 2FA, lockouts, lifecycle).
- **Customer Service** – Customer onboarding, KYC information, customer profile updates.
- **Account Service** – Current and savings accounts, balances, invariants, account lifecycle.
- **Transaction Service** – Posting transactions, transfers, simple statements and audit trails.
- **API Gateway** – A single entry point for all backend services.
- **Angular 21 Frontend (Nx monorepo)**:
  - A **back-office portal** for Alvor Bank employees.
  - A **customer portal** for self-service: view accounts, submit requests, etc.

Each microservice follows the same shape:

- **Domain** (DDD): entities, value objects, aggregates, events
- **Application** (CQRS): commands, queries, handlers, DTOs
- **Infrastructure**: EF Core, PostgreSQL, Identity, messaging
- **API**: FastEndpoints (or minimal-API style) as the HTTP surface

And everything is wired together with **Docker**, **docker compose**, and **CI pipelines**.

---

## Repositories and source code layout

To keep things clean and realistic, we’ll separate concerns into different Git repositories:

1. **Book repository** – contains the manuscript, diagrams and Leanpub/Markdown files (this is the repo you’re reading from right now).
2. **Main source code repository** – contains the full **Alvor Bank – Banking Suite** solution as it evolves throughout the book (for example: `digital-banking-suite/`).
3. **Chapter reference repository or tags** – instead of cluttering the main repo with dozens of branches, we’ll use **tags** or a separate **“chapter snapshots” repo** so you can checkout the state of the code at the end of each chapter.

A simple, reader-friendly setup is:

- One public repo for the full solution (`digital-banking-suite`), with tags like `chapter-03`, `chapter-07` and so on.
- Optionally, a **private “solutions” repo** with bonus material or extended branches for readers who purchased the book.

In this preface and throughout the book, when we show folder structures we’ll be referring to the **main source code repo**, for example:

- `digital-banking-suite/` – project root
  - `src/backend/` – all backend services (IAM, Customer, Account, Transactions, etc.)
  - `src/frontend/` – Angular 21 + Nx workspace
  - `infra/` – infrastructure (Docker compose files, later maybe Kubernetes manifests)
  - `tests/` – shared test projects (unit, integration, API)
  - `postman/` – Postman collections and environments

The **book repo** stays separate so you can evolve the text (and translations) independently from the code.

---

## How the project and chapters are structured

Each **backend service** shares a consistent structure so you can recognise patterns instead of learning everything from scratch each time.

Each **chapter** follows a predictable rhythm:

1. **Create a feature branch** for the chapter in the main source repo.
2. **Implement a vertical slice** of functionality (backend and sometimes frontend).
3. **Write tests** (unit, integration, API, as appropriate).
4. **Run everything in Docker** locally and explore the system.
5. **Run CI**, check coverage and fix anything that breaks.
6. **Open a PR**, imagine it being reviewed by a real team, then **merge to `develop`**.

This rhythm will train you to think and work like a real banking team.

---

## Tooling you’ll need

You don’t need a fancy machine, but you do need a few key tools installed:

- **.NET 10 SDK** (for example 10.0.100 or higher)
- **Node.js** (a recent LTS suitable for Angular 21)
- **Angular CLI 21** (installed globally or via `npx`)
- **Nx CLI** (for managing the monorepo)
- **Docker Desktop** or a Docker engine that can run Linux containers
- **Git**
- **Visual Studio 2026** (or latest Visual Studio) **or** **VS Code** with the C#, Angular and Docker extensions
- **Postman** (or any HTTP client) for testing APIs

In the early chapters we’ll walk through installing and verifying these.

---

## How to read and work through this book

You can read this book in different ways depending on your background.

### If you’re primarily a backend developer

- Don’t skip the **architecture and domain chapters** in Part I — they set the language we use everywhere.
- Focus deeply on backend chapters: **IAM, Customer, Account, Transactions**, and the shared **BuildingBlocks**.
- Skim the frontend details at first if you like, then come back when you’re ready to wire the UI.

### If you’re primarily a frontend developer

- Read the domain and IAM chapters to understand what the backend provides.
- Pay close attention to the **Angular 21 + Nx** chapters, especially how we generate clients, share types, and enforce boundaries in the monorepo.
- Use the backend chapters as a reference when you’re integrating specific endpoints.

### If you’re aiming for architecture / tech lead roles

- Read everything.
- Pay extra attention to:
  - How we define **bounded contexts** and map them to microservices.
  - How we structure **tests** and **CI pipelines**.
  - How we handle **IAM, security, and cross-cutting concerns** (logging, health checks, migrations).

Regardless of your path, the principle is the same:

> **Follow the steps, run the commands, read the code, and treat this like a real project.**

---

## Conventions used in the book

A few conventions we’ll use throughout:

- **Code blocks** use triple tildes:

  ```csharp
  public class Example
  {
      // ...
  }
  ```

- Shell commands are shown like this:

  ```bash
  dotnet new webapi -n ExampleService
  ```

- We assume a Linux-style shell for commands (even on Windows). If you use PowerShell, you may need small adjustments.
- Git branches follow a simple pattern:

  - `main` – production / final state
  - `develop` – integration branch
  - `feature/chapter-XX-something` – chapter feature branches

- We care about **tests and coverage**.  
  Chapters will explicitly call out commands like:

  ```bash
  dotnet test src/backend/BankingSuite.Backend.sln \
    /p:CollectCoverage=true \
    /p:CoverletOutput=../../../lcov \
    /p:CoverletOutputFormat=lcov
  ```

  and we expect **coverage ≥ 80%** for key microservices before merging.

---

## A note about versions (2026 and beyond)

At the time of writing, the stack we use is:

- **.NET 10**, **C# 14**
- **Angular 21**
- **Visual Studio 2026** (or latest VS) / **VS Code**
- **EF Core 10**, **PostgreSQL**, **Docker** and **GitHub Actions**

These will evolve over time.

Whenever you see a specific version (for example, a NuGet or NPM package), treat it as:

> “Use the latest stable **10.x / 21.x** version that matches this example.”

If a newer version changes an API slightly, don’t panic; the architecture and approach stay the same.

---

## Final word before we start

If you follow this book from start to finish, you will:

- Design a realistic banking system.
- Implement multiple microservices with Clean Architecture and DDD.
- Secure the platform with a full IAM service (login, roles, 2FA, lockouts, etc.).
- Build out **Customer, Account and Transaction** services with real business rules.
- Ship everything through Docker and CI like a real team.

And you’ll do it using the **2026 tooling** that modern teams care about: **.NET 10, C# 14, Angular 21, Nx, Visual Studio 2026, and modern DevOps practices**.

Let’s build a real digital bank.
