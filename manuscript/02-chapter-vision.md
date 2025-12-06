# Chapter 01 — The Vision: Building a Real Digital Bank

Before we touch any code, we need a clear vision.

This chapter defines **what** we are building, **why** we’re building it this way, and **what it will look like** when we are done.

By the end of this chapter you should be able to:

- Explain what **Alvor Bank – Banking Suite** is and what capabilities it offers.
- Describe the **core domains** of a digital bank (IAM, Customers, Accounts, Transactions, Payments).
- Understand why we’re using **microservices, Clean Architecture, DDD, TDD and modern DevOps**.
- Visualise the **end-to-end system** we’ll build with **.NET 10, Angular 21, Nx, Docker, RabbitMQ and MassTransit**.
- See how each later chapter fits into this overall vision.

---

## 1.1 What we’re building: Alvor Bank – Banking Suite

The goal of this book is not to build a toy CRUD app.  
We are building a **small but realistic digital banking platform** called **Alvor Bank – Banking Suite**.

Alvor Bank is:

- A fictional, **digital-first bank** that offers basic current and savings accounts and simple payments.
- Accessible through a **back-office portal** (for bank staff) and a **customer web portal**.
- Built with **microservices** and **Clean Architecture**, using **.NET 10** on the backend and **Angular 21 with Nx** on the frontend.
- Integrated using **RabbitMQ** and **MassTransit** for event-driven communication between services.
- Operated with **Docker, GitHub and CI/CD pipelines**, like a real engineering team would.

### 1.1.1 Core capabilities

At a high level, the Banking Suite will support:

- **Identity & Access Management (IAM)**

  - Staff and customer accounts
  - Roles and permissions (e.g., System Admin, Back-Office Agent, Customer)
  - Login with JWT tokens
  - Security features: lockouts after multiple failed attempts, password reset, email confirmation, two-factor authentication

- **Customer Management**

  - Onboarding customers with KYC details
  - Updating customer profiles
  - Tracking verification and risk status

- **Accounts**

  - Creating current and savings accounts
  - Associating accounts with customers
  - Maintaining balances and simple invariants
  - Basic account lifecycle (open, restricted, closed)

- **Transactions & Payments**

  - Posting transactions and transfers between accounts
  - Producing a simple transaction history
  - Initiating **everyday digital banking operations** such as:
    - Buying **electricity tokens**
    - Buying **prepaid airtime/data**
    - Paying **insurance premiums** or other recurring bills
  - Emitting events (e.g., `PaymentInitiated`, `PaymentCompleted`) through **RabbitMQ** so other services (like Notifications) can react

- **User Interfaces**
  - **Back-office portal** for staff to:
    - Log in securely
    - Onboard and manage customers
    - Open and manage accounts
    - Review and approve transactions and payments
  - **Customer portal** for end users to:
    - Log in securely
    - View accounts and balances
    - See recent transactions
    - Perform everyday operations:
      - Buy electricity tokens
      - Buy prepaid airtime/data
      - Pay simple bills like insurance
      - Initiate basic transfers between their own accounts

We are not trying to model every complexity of a real bank (regulatory reporting, FX, complex products), but we want enough detail that the patterns we use are **faithful to real-world systems** and reflect common African and global digital banking use cases.

---

## 1.2 The domains and bounded contexts

To keep complexity under control, we’ll split the system into a few **bounded contexts**, each implemented as a **microservice**.

### 1.2.1 IAM (Identity & Access Management)

IAM is responsible for:

- Users (staff and customers)
- Authentication (login, tokens, 2FA)
- Authorization (roles and permissions)
- Security policies (lockouts, password rules)
- User lifecycle (activate, deactivate, reset password, etc.)

IAM is a **central cross-cutting service**: every other service trusts IAM to tell it **who** is calling and **what they are allowed to do**.

### 1.2.2 Customer

The Customer context covers:

- Customer profiles:
  - Personal details (name, contact info)
  - KYC information (documents, verification status)
- Onboarding workflows:
  - Creating a new customer
  - Updating details
  - Tracking verification and risk flags

The Customer service integrates with IAM (a customer has a login) but owns the **business view of the customer**.

### 1.2.3 Accounts

The Account context focuses on accounts and balances:

- Different account types (current, savings)
- Ownership (which customer(s) own the account)
- Balances and simple rules:
  - No negative balance for basic accounts
  - Simple interest/savings rules (later, optional)
- Account lifecycle (open, block, close)

Account is where money “lives” in the bank’s view.

### 1.2.4 Transactions & Payments

The Transaction/Payment context is where money **moves**:

- Posting accounting entries between accounts
- Simple statements and history
- Ensuring that for every debit there is a corresponding credit
- Initiating **real-life payment flows**:
  - Electricity token purchases
  - Airtime/data top-ups
  - Insurance premium payments and similar simple bills
- Publishing **events** (for example, `TransactionPosted`, `PaymentFailed`) via **RabbitMQ** so:
  - Other services can react (e.g., Notifications, Reporting)
  - We keep services decoupled and scalable

In code, we’ll use **MassTransit** on top of RabbitMQ to simplify message publishing and consumption.

---

## 1.3 Why microservices, Clean Architecture, DDD and TDD?

There are many ways to build a system.  
In this book we deliberately choose a combination that reflects **modern enterprise banking**.

### 1.3.1 Why microservices?

We could build everything as a single monolith, but microservices give us:

- **Clear boundaries**: IAM, Customer, Account, Transaction/Payment each own their data and rules.
- **Independent deployment**: We can evolve IAM or Transaction without redeploying everything.
- **Technology freedom** (in theory): Even though we use .NET 10 everywhere in this book, the architecture allows different technologies per service in real life.
- **Scalability**: Some services (like Transactions/Payments) may need more horsepower than others.
- **Event-driven integration**: Using **RabbitMQ + MassTransit**, services can communicate via events instead of tight synchronous coupling.

However, microservices come with **costs**:

- More moving parts to deploy and monitor
- Network calls instead of in-memory calls
- More complex debugging and tracing

We’ll see both the benefits and the trade-offs as we go.

### 1.3.2 Why Clean Architecture?

Clean Architecture (and similar layered approaches) help us:

- Separate **business rules** from **frameworks** and **infrastructure**.
- Make it easier to:
  - Test core logic without hitting databases or the network.
  - Swap implementations (e.g., different databases or messaging tools) later.
  - Maintain and reason about the codebase over time.

For each service we’ll structure code roughly as:

- **Domain** – Entities, value objects, aggregates, domain services, domain events
- **Application** – Use cases (commands/queries), DTOs, interfaces
- **Infrastructure** – EF Core, PostgreSQL, Identity, RabbitMQ/MassTransit, external integrations
- **API** – FastEndpoints, request/response mapping, authentication and authorization

### 1.3.3 Why DDD (Domain-Driven Design)?

DDD helps us:

- Understand the **language of the business** (what “account”, “customer”, “transaction”, “electricity token purchase” mean in context).
- Draw **boundaries** between parts of the system.
- Respect **invariants** and **rules** (e.g., which operations are allowed on accounts).

We’ll use DDD pragmatically:

- Just enough terminology to be useful (aggregate, value object, domain event).
- Lots of real examples in the banking context.
- No endless theory without code.

### 1.3.4 Why TDD and testing up front?

In a banking environment, bugs are expensive:

- A bad transaction could lose money.
- A broken IAM policy could expose data.
- A lost invariant could corrupt balances.
- A messaging bug could cause payments or notifications to be dropped.

That’s why real teams care about tests and coverage.

In this book:

- We’ll aim for **≥ 80% test coverage** on the core business logic of key services (IAM, Account, Transactions).
- We’ll write **unit tests**, **integration tests**, and **API tests**.
- We’ll add tests around **message publishing and consuming** when we introduce RabbitMQ and MassTransit.
- CI will **fail the build** if tests fail or coverage drops below our threshold.

You’ll see how tests shape the design and give us confidence to refactor.

---

## 1.4 Backend + Frontend + Messaging + DevOps blueprint

Let’s look at how everything fits together.

### 1.4.1 Backend blueprint

On the backend we’ll use:

- **.NET 10 & C# 14**
- **ASP.NET Core** with FastEndpoints / minimal-API style endpoints
- **EF Core 10** with **PostgreSQL** as the database
- **ASP.NET Core Identity** for IAM
- **RabbitMQ** as the message broker
- **MassTransit** as the .NET abstraction over RabbitMQ
- **Docker** for containerization and `docker compose` for local orchestration

Each microservice is its own .NET project (or set of projects), using the Clean Architecture structure described earlier, and services communicate both:

- Synchronously via **HTTP APIs**, and
- Asynchronously via **events over RabbitMQ**.

### 1.4.2 Frontend blueprint

On the frontend we’ll build an **Nx monorepo** with **Angular 21**:

- Multiple Angular apps:
  - `backoffice-app` – for bank staff
  - `customer-app` – for customers
- Shared libraries:
  - `ui` – shared UI components
  - `data-access` – typed API clients
  - `utils` – cross-cutting utilities
  - `auth` – authentication & guards

The **customer portal** will expose real-world digital banking flows:

- Login and registration/onboarding steps
- Viewing accounts and balances
- Viewing transaction history
- Starting **everyday operations** like:
  - Buying electricity tokens
  - Buying prepaid airtime/data
  - Paying simple insurance bills

These flows will call backend APIs, which in turn may publish events via RabbitMQ for other services (like Notifications) to consume.

### 1.4.3 DevOps blueprint

From early in the book we’ll set up:

- **Git branching strategy**:
  - `main` – final, stable code
  - `develop` – integration branch
  - `feature/chapter-XX-...` – chapter branches
- **GitHub Actions** for CI:
  - Build backend and frontend
  - Run tests
  - Generate coverage report
- **Docker images**:
  - One image per microservice
  - Standardised Dockerfiles and compose definitions
  - A RabbitMQ container and PostgreSQL containers for each service

In later parts of the book we’ll enhance this with:

- Image tags and versioning
- Basic hardening (secrets, configuration)
- Hooks for future deployment targets (cloud or Kubernetes)

---

## 1.5 How we will work through the book

A key idea in this book is that each major bounded context is built **end-to-end**:

> **Backend first, then UI, then messaging, then tests and Docker, all within the same “feature block”.**

This matches how many teams ship features in real life.

### 1.5.1 The IAM pattern

We’ll start by building the **IAM service**:

1. Model the IAM domain (users, roles, policies).
2. Implement IAM as a microservice with Clean Architecture.
3. Wire up IAM to PostgreSQL, Identity and JWT.
4. Add tests and coverage.
5. Expose IAM through FastEndpoints.
6. Build the **IAM front-end pieces** in Angular:
   - Login screen
   - JWT handling and storage
   - Role-based navigation
   - Basic user management screens
7. Package and run everything in **Docker**.

Once IAM is in a good shape, we’ll **repeat the pattern** for the other contexts.

### 1.5.2 Repeatable pattern for each microservice

For each major service (Customer, Account, Transaction/Payments) we’ll:

1. Define the **domain model** and workflows.
2. Implement the **backend microservice** (domain + application + infrastructure + API).
3. Integrate with **RabbitMQ/MassTransit** where events make sense.
4. Add **tests and coverage** (including tests around event publishing/consuming where applicable).
5. Build the **corresponding Angular UI**:
   - For Customers: onboarding and management screens.
   - For Accounts: account overview and management UI.
   - For Transactions & Payments: transfer and statement screens plus simple flows to buy electricity tokens, airtime and pay insurance.
6. Run everything together via **Docker + CI**.

You’ll quickly recognise a rhythm:

- Design → Backend → Messaging → Tests → Frontend → Docker → CI.

Once you learn it for IAM, you’ll be comfortable applying it to new services.

---

## 1.6 What you’ll achieve by the end of the book

If you follow this book from start to finish, you will:

- Understand how to **design a digital banking system** around clear bounded contexts.
- Be comfortable structuring **.NET 10 microservices** with **Clean Architecture and DDD**.
- Know how to implement a **full IAM system** with Identity, JWT, 2FA, account lockouts and user lifecycle.
- Be able to build **Angular 21 frontends** in an **Nx monorepo**, sharing UI and data-access libraries across apps.
- Have hands-on experience with **RabbitMQ and MassTransit** for event-driven communication between services.
- Have hands-on experience with **TDD and testing strategy**, including coverage targets and CI enforcement.
- Run and debug a multi-service system in **Docker**, and enhance it with **CI/CD pipelines** that reflect real banking practices.
- Deliver real **customer-facing flows** such as:
  - Viewing accounts and transactions
  - Buying electricity tokens and airtime
  - Paying simple bills like insurance

Most importantly, you’ll have a **complete end-to-end example** you can use:

- As a starting point for your own projects.
- As a portfolio piece in interviews.
- As a reference when you design new systems.

---

## 1.7 Next steps

In the next chapter we’ll zoom into the **domain architecture**:

- Refine the bounded contexts (IAM, Customer, Account, Transaction/Payments).
- Draw the key diagrams that show how services talk to each other—both via HTTP and via RabbitMQ events.
- Identify the core entities and workflows that will drive our implementation.

With the vision of Alvor Bank clear in your mind, we’re ready to start designing the system.

Let’s move from **“what”** to **“how”.**
