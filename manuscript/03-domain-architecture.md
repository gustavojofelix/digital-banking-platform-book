# Chapter 2 — Domain Architecture of a Digital Bank

In Chapter 1, we defined the **vision** for the Banking Suite and saw the high-level architecture: frontend, API gateway, microservices, messaging, and databases.

In this chapter, we zoom in on the **domain itself**:

- How to slice the system into **bounded contexts**
- How those contexts collaborate using **events and APIs**
- The difference between **domain models** and **integration models**
- The key **business workflows** that drive the design
- How everything fits into a **system and container view**

By the end of this chapter, you’ll have a clear mental model of the domains that shape the code you’ll write later.

---

## 2.1 From Business Capabilities to Bounded Contexts

Before thinking about projects, folders, or databases, we start with **business capabilities**:

- Onboard a new customer
- Open and manage bank accounts
- Move money (deposits, withdrawals, transfers)
- Authenticate users and control what they can do
- Notify customers about important events

These capabilities naturally group into **bounded contexts**. A bounded context is a part of the system where:

- A **specific domain language** is used consistently
- A **domain model** makes sense and is internally coherent
- The **rules** are clear and locally owned

For the Banking Suite, our main bounded contexts are:

- **IAM (Identity & Access Management)**
- **Customers**
- **Accounts**
- **Transactions**
- **Notifications**

We treat each bounded context as a **separate microservice** in the implementation, but the concept is architectural first, technical second.

---

## 2.2 Bounded Context: IAM

The **IAM context** is responsible for:

- User registration and login
- Password management and security policies
- Roles and permissions (customer, admin, operator)
- (Optionally) Tenants / organizations if you run the platform as SaaS

Inside the IAM context:

- A **User** is someone who can log in and act in the system.
- A **Role** defines what users can do (e.g., `Customer`, `BankAdmin`).
- **Permissions** or policies may define more granular access rules.

Importantly:

- IAM **does not own** customer profile data (addresses, phone numbers, etc.).
- IAM **does not know** about accounts, balances, or transactions.
- It only knows **who** someone is (from an auth point of view) and **what they can do**.

When another context needs to know “who is this user?”, it interacts with IAM via:

- **Tokens and claims** (e.g., JWT) for authentication
- Optionally, **read-only APIs** for querying identity-related information

---

## 2.3 Bounded Context: Customers

The **Customer context** represents the bank’s relationship with individuals or businesses.

Responsibilities:

- Store customer profiles (name, email, phone, address)
- Associate customers with IAM users, where relevant
- Track basic KYC-style information (very simplified for this book)
- Provide customer information to other contexts (e.g., Accounts, Notifications)

In this context:

- A **Customer** is a person or organization that owns one or more **Accounts**.
- A **CustomerId** is the primary identifier shared with other contexts.

The Customer context doesn’t:

- Handle login — that’s IAM.
- Manage accounts or balances — that’s Accounts.
- Record money movement — that’s Transactions.

It is purely **who** the bank serves.

---

## 2.4 Bounded Context: Accounts

The **Accounts context** models how money is stored and labeled for a customer.

Responsibilities:

- Create and manage bank accounts
- Enforce account lifecycle: `Opened → Active → Frozen → Closed`
- Manage account types: current/checking, savings, etc.
- Maintain **derived balances** based on transactions
- Enforce simple account-level rules (e.g., block transactions on frozen accounts)

Key concepts:

- **Account** — belongs to a **Customer**, has an account number and type.
- **AccountStatus** — indicates whether the account is usable.
- **Balance** — often derived from Transactions, but cached in Accounts for performance.

The Accounts context is tightly linked to Transactions but has its own rules:

- You may **not** be allowed to perform certain transactions depending on account status.
- Some actions (like opening or closing an account) emit **domain events**.

---

## 2.5 Bounded Context: Transactions

The **Transactions context** is where actual **money movement** is recorded.

Responsibilities:

- Record **debits and credits** to accounts.
- Ensure that each transaction is **consistent and traceable**.
- Support different transaction types:
  - Deposit
  - Withdrawal
  - Transfer between internal accounts
- Expose transaction history for a given account.

Key concepts:

- **Transaction** — an atomic event that affects one or more accounts.
- **Ledger** — the logical stream of transactions that defines balances.
- **Reference / Correlation IDs** — used for reconciliation and idempotency.

The Transactions context does not:

- Decide who is allowed to perform an action — that’s IAM + Accounts.
- Handle notifications — that’s the Notifications context.
- Own customer information — that’s Customers.

It just answers: **“What happened to the money, and when?”**

---

## 2.6 Supporting Context: Notifications

The **Notifications context** is not core to banking, but it is crucial to a good digital banking experience.

Responsibilities:

- Receive events like `TransactionCompleted`, `AccountOpened`, `PasswordChanged`
- Transform them into **human-friendly messages**
- Send emails (and possibly SMS or push notifications in the future)

The Notification context:

- Does not initiate business actions.
- Reacts to what happens elsewhere.
- Is naturally **event-driven**.

This context is a great example of how you can add features **without changing core services**, by subscribing to their events instead.

---

## 2.7 Event-Driven Design in the Banking Suite

Event-driven design is a natural fit for banking. Many things that happen are **facts** about the past that other parts of the system care about:

- An account was created.
- A transaction was completed.
- A password was reset.
- A customer updated their contact email.

We represent these as **events**:

- `AccountOpened`
- `TransactionCompleted`
- `CustomerEmailUpdated`
- `UserPasswordChanged`

### Why Events?

Events help us:

- Decouple services: the Transactions service doesn’t need to know who listens to `TransactionCompleted`.
- Scale independently: Notifications can scale based on event volume.
- Add features without touching core flows (e.g., add analytics or reporting later).

### A Simple Event Flow Example

Consider a **deposit**:

1. The frontend calls the **API Gateway**, which forwards to **Transaction Service**.
2. Transaction Service:
   - Validates the request
   - Records a new `Transaction` in its ledger
   - Publishes a `TransactionCompleted` event
3. **Account Service** subscribes to the event and updates the account’s cached balance.
4. **Notification Service** also subscribes and sends an email to the customer.

Visually, you can represent this flow with a diagram such as:

![Event-Driven Flow: Deposit → Event → Notification](event-flow.png)

---

## 2.8 Domain Models vs Integration Models

One of the most common mistakes in microservice systems is to treat **API contracts** and **domain models** as if they were the same thing.

They are not.

### Domain Model

- Lives inside a bounded context.
- Encodes business rules and invariants.
- Can be rich and behavior-focused (methods, invariants, domain events).
- Can evolve **independently**, as long as its public interfaces remain consistent.

Example (simplified):

```csharp
public class Account
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public decimal Balance { get; private set; }
    public AccountStatus Status { get; private set; }

    public void ApplyCredit(decimal amount)
    {
        // business rules, validations, domain events...
    }

    public void ApplyDebit(decimal amount)
    {
        // business rules, overdraft rules, etc...
    }
}
```

### Integration Model (DTO / Contract)

- Represents how data is sent over the wire between services or to the frontend.
- Needs to be **versioned carefully**, because other systems depend on it.
- Often flatter and more explicit than domain models.

Example:

```csharp
public sealed class AccountDto
{
    public Guid Id { get; init; }
    public string AccountNumber { get; init; } = default!;
    public string Type { get; init; } = default!;
    public string Status { get; init; } = default!;
    public decimal Balance { get; init; }
}
```

Key point:

> The **domain model** serves the **internal business rules** of the bounded context.  
> The **integration model** serves the **outside world** (APIs, events, other services).

Keeping them separate gives you flexibility to evolve your internals without breaking everyone who consumes your APIs.

## 2.9 High-Level Business Workflows

To validate our domain architecture, we run it through a few core workflows.

### Workflow 1 — Open a New Account

1. User logs in via **IAM**.
2. Frontend calls **Customer Service** to fetch or create a customer profile.
3. Frontend calls **Account Service** to open a new account for that customer.
4. Account Service:
   - Validates customer eligibility (simplified)
   - Creates an `Account` in its own database
   - Emits `AccountOpened` event
5. Other services (e.g., Notifications) may react to `AccountOpened`.

Bounded contexts involved: **IAM, Customers, Accounts, Notifications**.

---

### Workflow 2 — Deposit Money

1. User initiates a deposit from the frontend.
2. Gateway forwards the request to **Transaction Service**.
3. Transaction Service:
   - Validates the account and amount
   - Records a `Transaction` in its ledger
   - Emits `TransactionCompleted`
4. **Account Service** listens to `TransactionCompleted` and updates balances.
5. **Notification Service** sends a “Deposit received” email to the customer.

Bounded contexts involved: **IAM (auth), Transactions, Accounts, Notifications**.

---

### Workflow 3 — Transfer Between Internal Accounts

1. User selects a source and destination account.
2. Frontend calls **Transaction Service** to create a transfer.
3. Transaction Service:
   - Validates that both accounts exist and are active
   - Ensures the source account has sufficient balance (via Accounts or cached info)
   - Creates a transfer transaction (debit source, credit destination)
   - Emits `TransactionCompleted` (or separate events for debit/credit)
4. **Account Service** updates both account balances.
5. **Notification Service** may notify the account owner(s).

Bounded contexts involved: **IAM, Transactions, Accounts, Notifications**.

These workflows validate that our bounded contexts are **cohesive internally** and **cooperate cleanly** via events and APIs.

---

## 2.10 System Context & Container View

To communicate the architecture to other people (and to future you), it’s useful to have a couple of static diagrams:

- A **System Context diagram** — the Banking Suite as a single system and the external actors around it.
- A **Container diagram** — the main containers (frontend, gateway, microservices, databases, message broker) and how they relate.

![System Context Diagram](c4-system-context.png)

_System Context — the Banking Suite as seen by users and external systems._

![Container Diagram](c4-container.png)

_Container View — frontend, gateway, microservices, databases, and messaging._

> You don’t need to memorize every box and arrow. The purpose of these diagrams is to keep the team aligned on the **big picture** while you dive into implementation details.

---

## 2.11 Summary & What’s Next

In this chapter, you:

- Identified the main **bounded contexts** of a digital bank: IAM, Customers, Accounts, Transactions, Notifications.
- Saw how each context has a **clear responsibility** and owns its own data and rules.
- Learned how **events** like `AccountOpened` and `TransactionCompleted` are used to integrate services in an **event-driven** way.
- Understood the difference between **domain models** and **integration models** and why they shouldn’t be the same thing.
- Walked through core **business workflows** (open account, deposit, transfer) and mapped them to contexts.
- Saw how the system fits into **system context** and **container** views.

You now have a solid understanding of **what** you’re building in domain terms and **how** the pieces relate.

In the next chapter, we’ll make this architecture concrete on your machine:

- Set up your development environment
- Create the GitHub repository and base structure
- Define the branch strategy you’ll use for the rest of the book

From there, we move from architecture to code — with a clear direction already in place.
