# Introduction

Software changes fast. Banking changes slowly.

This tension makes digital banking one of the best domains to learn **architecture**, **testing**, and **DevOps** that can survive more than one framework release.

This book invites you to build a **real digital banking platform**—not a demo, not a todo list, but a small version of what an actual bank could run in 2026:

- Multiple **microservices** with clear bounded contexts
- A modern **Angular 21** frontend in an **Nx monorepo**
- A hardened **IAM service** with login, roles, 2FA and lockouts
- **Docker**, **CI/CD**, **tests and coverage** from day one

You’ll use the latest tools: **.NET 10, C# 14, Visual Studio 2026, Angular 21, Nx and GitHub Actions**.  
But more importantly, you’ll learn patterns you can carry into any modern backend system.

---

## The core idea: learn by building a bank

Instead of teaching patterns in isolation, we’ll grow a single system called **Alvor Bank – Banking Suite**.

You will:

- Design the **domain** and map it to bounded contexts.
- Implement those contexts as separate **microservices** with Clean Architecture and DDD.
- Expose them through **HTTP APIs** and an **API gateway**.
- Authenticate and authorize everyone through a dedicated **IAM service**.
- Build a **back-office portal** for bank staff and a **customer portal** for end users.
- Run everything locally with **Docker** and ship it through **CI/CD pipelines**.

Most importantly, you’ll always see how backend and frontend come together:

> For each major bounded context (IAM, Customers, Accounts, Transactions), we’ll build the **backend first**, then **immediately wire the Angular UI** so you can see the feature working end-to-end.

By the end, you’ll have a working platform and, more importantly, the confidence to build similar systems in your own career.

---

## The tech stack at a glance

We’ll use a modern stack that matches what many teams will be running in 2026 and beyond:

- **Backend**

  - .NET 10, C# 14
  - ASP.NET Core with FastEndpoints (or minimal-API style)
  - EF Core 10 + PostgreSQL
  - ASP.NET Core Identity, JWT, role-based auth, 2FA
  - Docker & docker compose

- **Frontend**

  - Angular 21
  - Nx monorepo (apps + shared libs)
  - Typed API clients, feature modules, shared UI components

- **DevOps & Tooling**
  - Visual Studio 2026 (or latest Visual Studio) & VS Code
  - Git & GitHub
  - GitHub Actions for CI/CD
  - Postman collections & Newman for API testing
  - Code coverage with Coverlet and ReportGenerator

We’ll keep the versions current at the time of writing (for example, **.NET 10.x**, **Angular 21.x**). When you read this, use the latest compatible minor versions.

---

## The microservices you’ll build

The Banking Suite will be composed of several services:

- **IAM Service** – Identity & Access Management for staff and customers:  
  Login, roles, permissions, JWT tokens, account lockouts, password reset, email confirmation, and two-factor authentication.

- **Customer Service** – Customer onboarding and KYC:  
  Customer profiles, documents, verification status, and basic lifecycle.

- **Account Service** – Accounts and balances:  
  Current and savings accounts, ownership, simple invariants (no negative balances unless explicitly allowed), basic account lifecycle.

- **Transaction Service** – Money moving between accounts:  
  Posting transactions, transfers, simple statements and audit-friendly history.

- **API Gateway** – A thin layer that exposes a single entry point to all backend services.

- **Angular 21 Frontend (Nx monorepo)** – Two primary experiences:
  - A **back-office portal** for Alvor Bank employees.
  - A **customer portal** for retail users.

Each service follows the same Clean Architecture pattern:

- **Domain** – Entities, value objects, aggregates, domain services, events
- **Application** – Use cases expressed as CQRS commands and queries
- **Infrastructure** – EF Core, Identity, messaging, external integrations
- **API** – HTTP endpoints, auth, request/response mapping

And for each bounded context, we’ll **pair the backend with its frontend**:

- Build IAM backend → build IAM login & admin UI.
- Build Customer Service backend → build customer onboarding & management UI.
- Build Account & Transaction services → build account overview, transaction history and simple transfer flows.

---

## Repositories and how to follow along

To keep things organised, we’ll use **separate repositories**:

1. **Book repository** – Contains the manuscript, diagrams and assets for this book (the Markdown you’re reading now).

2. **Main source code repository** – Contains the full **Alvor Bank – Banking Suite** solution as it evolves, for example:

   - `digital-banking-suite/`
     - `src/backend/` – IAM, Customer, Account, Transaction and related services
     - `src/frontend/` – Angular 21 + Nx apps and libs
     - `infra/` – Docker compose and infrastructure scripts
     - `tests/` – Unit, integration and API test projects
     - `postman/` – Postman collections and environments

3. **Chapter snapshots** – Instead of dozens of long-lived branches, we’ll use **Git tags** (or a lightweight “chapter snapshots” repo) so you can check out the exact state of the code after each chapter, for example:

   - `chapter-01-vision`
   - `chapter-03-dev-environment`
   - `chapter-07-iam-backend`
   - `chapter-08-iam-frontend`
   - `chapter-10-customer-service`
   - etc.

In the book, when you see folder structures or commands, assume they refer to the **main source code repo**. When you want to compare your progress, you can fetch the corresponding tag and inspect how things look at that point.

---

## How the chapters are organised

This book is divided into several parts that mirror how you might run a real project.  
From Part III onward we deliberately **pair back-end and front-end work** for each microservice.

### Part I — Understanding & Designing Modern Digital Banking Systems

We start with **concepts and design**:

- Why microservices for a bank?
- What are the core bounded contexts?
- What does a basic banking workflow look like?
- How do we set up a clean development environment and branching strategy?

You’ll define the vision for Alvor Bank, sketch the domain, and prepare your environment (tooling, repos, branches).

### Part II — Foundations: Environment, Containers, Testing & CI/CD

Before we dive into individual services, we set the **foundation**:

- Install and verify **.NET 10, Angular 21, Nx, Visual Studio 2026**.
- Create the main repo and branch strategy.
- Introduce **Docker** and **docker compose** for local infrastructure.
- Define a **testing strategy**: unit, integration, API tests.
- Set up a **CI pipeline** (GitHub Actions) that builds, tests and reports coverage.
- Introduce shared **BuildingBlocks** and the common microservice template.

By the end of Part II you’ll have a stable skeleton: a repeatable backend structure and a working CI pipeline.

### Part III — IAM Service: Secure Access End-to-End

Here we build **Identity & Access Management**, including its UI:

- Finalise the backend microservice structure with **IAM** as the template.
- Implement IAM Domain, Application, Infrastructure and API (login, roles, 2FA, lockouts).
- Add **Postman collections** and tests for IAM.
- Set up the **Angular 21 + Nx workspace**.
- Build the **IAM front-end pieces**: login page, JWT handling, role-based navigation, basic admin screens for managing users.

By the end of Part III you’ll have a working login flow end-to-end, with tokens issued by IAM and consumed by the Angular app.

### Part IV — Customer Onboarding & Management (Backend + UI)

Next we focus on **Customers**:

- Design and implement the **Customer Service** backend: entities, onboarding flows, KYC data, customer status.
- Expose customer APIs through the gateway and secure them with IAM roles.
- Build the **customer management UI** in Angular:
  - Back-office screens for creating and reviewing customers.
  - A first version of a **customer self-service onboarding** flow.

Again, we close the loop by pairing backend and UI before moving on.

### Part V — Accounts & Transactions (Backend + UI)

Now we model **money and movement**:

- Implement **Account Service**: accounts, balances, simple invariants and lifecycle.
- Implement **Transaction Service**: posting transactions, basic transfers and history.
- Create **Angular views** for:
  - Account overview and details.
  - Transaction history.
  - A simple “make a transfer” flow that uses the Transaction Service.

By the end of Part V, a customer can log in, see their accounts and perform simple operations through the web UI.

### Part VI — DevOps & Hardening

Finally we focus on **operational excellence**:

- Polish Docker images and compose files.
- Enhance CI/CD pipelines to build and push images, run tests and enforce coverage.
- Add logging, health checks and simple monitoring.
- Discuss basic hardening: secrets, configuration, minimal surface area.
- Explore options for deploying to cloud platforms or Kubernetes.

By the end of Part VI, you will have travelled from blank repo to a **deployable, testable, observable digital banking suite**.

---

## How to work through this book

You can approach this book in different ways:

- **End-to-end builder**  
  Follow chapters in order, building everything as you go. This is the most immersive path and the one recommended if you want the full experience.

- **Backend-first**  
  When we reach a paired backend + frontend section (for example, IAM), you can focus on the backend chapters first, then come back to the UI chapters later. The snapshot tags make it easy to pick up where you left off.

- **Frontend-focused**  
  Read just enough of Parts I–II to understand the APIs and domain, then follow each microservice’s UI chapter to see how Angular 21 + Nx consume those APIs.

Whichever path you choose, treat the code as if it were your **day job**:

- Create branches.
- Keep tests green.
- Run Docker regularly.
- Use `git diff` and tags to understand how the system evolves.

---

## What you need before starting

You don’t need to be an expert in everything, but these basics will make your journey smoother:

- Comfortable reading and writing **C#**
- Basic understanding of **HTTP and REST APIs**
- Some experience with **JavaScript/TypeScript** (for the Angular parts)
- Familiarity with **Git** and basic branching
- Willingness to open a terminal and run some commands

If you don’t know Angular well, that’s fine—you’ll pick up a lot by building the specific screens and flows we need for the banking suite.

---

## A note on versions and real-world use

The examples in this book target:

- **.NET 10** and **C# 14**
- **Angular 21**
- **Visual Studio 2026** (or the latest VS / VS Code)
- **EF Core 10**, **PostgreSQL**, **Docker**, **GitHub Actions**

In real projects, you might use slightly different versions, different CI servers, or other cloud providers. That’s fine.

The goal is that, after working through these chapters, you’ll understand **why** the system is structured this way and be able to adapt the ideas to your own stack and constraints.

---

## Let’s begin

In the next chapter, we’ll set the vision for Alvor Bank:

- What are we building?
- Who are the users?
- What capabilities does a modern digital bank need?

From there, we’ll design the domain and gradually turn that vision into code—pairing backend and frontend for each microservice so you always see the full story.

Let’s build a real digital bank.
