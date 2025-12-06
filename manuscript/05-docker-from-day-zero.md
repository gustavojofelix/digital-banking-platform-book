# Chapter 04 — Docker from Day Zero: Local Infrastructure with PostgreSQL & RabbitMQ

In the previous chapter we prepared our **development environment**, created the `digital-banking-suite` repository and set up a simple folder structure.

Before we create any microservices, we’ll prepare our **local infrastructure** using Docker:

- **PostgreSQL** databases for our services
- **RabbitMQ** as the message broker

We want to be able to run:

- IAM, Customer, Account, Transaction and Notification services against their own databases
- All asynchronous events through **RabbitMQ**

By the end of this chapter you will:

- Have a working `infra/docker-compose.dev.yml` file.
- Be able to start PostgreSQL and RabbitMQ with a single `docker compose` command.
- See containers running and ready to accept connections.
- Tag the repo so this setup is always easy to return to.

> **IDE note:** In this book we assume you are **primarily using Visual Studio 2026**.  
> If you use Rider, VS Code or another IDE, follow similar steps (create files/folders in your IDE or via the command line).

---

## 4.1 Why containers now?

We want our readers (and future team members) to:

- Run **exactly the same infrastructure** on any machine.
- Avoid local “it works on my machine” database setups.
- Treat Docker as a **first-class citizen** from day one.

All backend services will eventually run in containers, but first we containerise:

- **PostgreSQL** – one database per service (IAM, Customer, Account, Transaction, Notification).
- **RabbitMQ** – shared message broker.

We’ll use a single `docker-compose.dev.yml` file in the `infra` folder to manage everything.

---

## 4.2 Open the repository in Visual Studio 2026

1. Start **Visual Studio 2026**.
2. Go to **File → Open → Folder…**.
3. Select the `digital-banking-suite` folder you created in Chapter 03.

Visual Studio will load the folder as a **folder-based solution**, showing `src`, `infra`, `tests`, and `postman` in **Solution Explorer**.

We’ll add files directly under `infra` using Visual Studio.

---

## 4.3 Creating `docker-compose.dev.yml`

In Visual Studio:

1. In **Solution Explorer**, right-click the `infra` folder.
2. Choose **Add → New Item…**.
3. Select **Text File** (or “General → Text File”).
4. Name it **`docker-compose.dev.yml`** and click **Add**.

Now replace the file contents with the following YAML:

```yaml
version: "3.9"

services:
  # PostgreSQL for IAM Service
  iam-db:
    image: postgres:16
    container_name: iam-db
    environment:
      POSTGRES_USER: iam_user
      POSTGRES_PASSWORD: iam_password
      POSTGRES_DB: iam_service_db
    ports:
      - "5433:5432"
    volumes:
      - iam-db-data:/var/lib/postgresql/data
    networks:
      - bankingsuite-network

  # PostgreSQL for Customer Service
  customer-db:
    image: postgres:16
    container_name: customer-db
    environment:
      POSTGRES_USER: customer_user
      POSTGRES_PASSWORD: customer_password
      POSTGRES_DB: customer_service_db
    ports:
      - "5434:5432"
    volumes:
      - customer-db-data:/var/lib/postgresql/data
    networks:
      - bankingsuite-network

  # PostgreSQL for Account Service
  account-db:
    image: postgres:16
    container_name: account-db
    environment:
      POSTGRES_USER: account_user
      POSTGRES_PASSWORD: account_password
      POSTGRES_DB: account_service_db
    ports:
      - "5435:5432"
    volumes:
      - account-db-data:/var/lib/postgresql/data
    networks:
      - bankingsuite-network

  # PostgreSQL for Transaction Service
  transaction-db:
    image: postgres:16
    container_name: transaction-db
    environment:
      POSTGRES_USER: transaction_user
      POSTGRES_PASSWORD: transaction_password
      POSTGRES_DB: transaction_service_db
    ports:
      - "5436:5432"
    volumes:
      - transaction-db-data:/var/lib/postgresql/data
    networks:
      - bankingsuite-network

  # PostgreSQL for Notification Service (optional but keeps pattern consistent)
  notification-db:
    image: postgres:16
    container_name: notification-db
    environment:
      POSTGRES_USER: notification_user
      POSTGRES_PASSWORD: notification_password
      POSTGRES_DB: notification_service_db
    ports:
      - "5437:5432"
    volumes:
      - notification-db-data:/var/lib/postgresql/data
    networks:
      - bankingsuite-network

  # RabbitMQ broker for messaging
  rabbitmq:
    image: rabbitmq:3-management
    container_name: rabbitmq
    ports:
      - "5672:5672" # AMQP
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: rabbitmq
      RABBITMQ_DEFAULT_PASS: rabbitmq
    networks:
      - bankingsuite-network

networks:
  bankingsuite-network:
    driver: bridge

volumes:
  iam-db-data:
  customer-db-data:
  account-db-data:
  transaction-db-data:
  notification-db-data:
```

A few notes:

- We expose each Postgres container on a different local port (`5433`–`5437`) so they don’t conflict.
- All containers join the same `bankingsuite-network` so services can talk to them by container name (`iam-db`, `customer-db`, etc.).
- `rabbitmq:3-management` gives us the standard RabbitMQ **web UI** on port `15672`.

We’ll later add the microservice containers (IAM, Customer, Account, etc.) to this same file.

---

## 4.4 Starting the infrastructure

Open a terminal at the root of the repository (`digital-banking-suite`):

- From Visual Studio:
  - Go to **View → Terminal** (or **View → Other Windows → Terminal**).
  - Make sure the current directory is the repo root (you should see `infra/`, `src/`, etc.).

Run:

```bash
docker compose -f infra/docker-compose.dev.yml up -d
```

This will:

- Pull the Postgres and RabbitMQ images (first run only).
- Start all containers in the background (`-d`).

Check running containers:

```bash
docker ps
```

You should see entries similar to:

```text
CONTAINER ID   IMAGE                   NAMES
...            rabbitmq:3-management   rabbitmq
...            postgres:16             iam-db
...            postgres:16             customer-db
...            postgres:16             account-db
...            postgres:16             transaction-db
...            postgres:16             notification-db
```

---

## 4.5 Verifying PostgreSQL

You can test that one of the Postgres instances is working by using `psql` or any GUI client (e.g. DBeaver, Azure Data Studio). For example:

- **Server/host:** `localhost`
- **Port:** `5433`
- **Database:** `iam_service_db`
- **User:** `iam_user`
- **Password:** `iam_password`

Repeat with ports `5434`–`5437` and the corresponding credentials if you want to verify the others.

We won’t create tables yet — that will happen when we add EF Core migrations in later chapters.

---

## 4.6 Verifying RabbitMQ

Open your browser and navigate to:

```text
http://localhost:15672
```

Log in with:

- **Username:** `rabbitmq`
- **Password:** `rabbitmq`

You should see the RabbitMQ management UI.  
Later, when we publish and consume messages, you’ll see exchanges, queues and messages here.

---

## 4.7 Stopping and cleaning up containers

To stop all containers defined in `docker-compose.dev.yml`:

```bash
docker compose -f infra/docker-compose.dev.yml down
```

This will stop and remove the containers but **keep the volumes** (your data) by default.

If you want to remove volumes as well (for a completely fresh start):

```bash
docker compose -f infra/docker-compose.dev.yml down -v
```

Use this with care — it will delete all Postgres data.

---

## 4.8 Committing and tagging Chapter 04

Let’s follow our standard workflow and commit these changes.

In the **Visual Studio 2026** Git window or any terminal:

```bash
git status
git add infra/docker-compose.dev.yml
git commit -m "ch04: add docker-compose for PostgreSQL and RabbitMQ"
```

Push to GitHub (from `develop` or a feature branch such as `feature/ch04-docker-infra`):

```bash
git push
```

---

## 4.9 Sanity checklist

Before moving on, confirm:

- [x] `docker-compose.dev.yml` exists under `infra/`.
- [x] `docker compose -f infra/docker-compose.dev.yml up -d` starts all Postgres and RabbitMQ containers.
- [x] You can open the RabbitMQ UI at `http://localhost:15672`.
- [x] You can connect to at least one Postgres instance using the provided credentials.
- [x] Changes are committed to Git and (optionally) tagged as `chapter-04-docker-infra`.

If all checks pass, your local infrastructure is ready.

---

## 4.10 What’s next

In the next chapter we will:

- Introduce our **first backend solution and projects** using **Visual Studio 2026**.
- Create the shared **BuildingBlocks** projects that will be reused by all microservices.
- Add a simple **“health check”** API to validate that .NET 10, Docker and PostgreSQL can work together.
- Start wiring in **tests** and **GitHub Actions CI** so that from IAM onward, every chapter builds on a solid foundation.

We now have a reliable, repeatable **containerised infrastructure**.  
Next, we’ll start putting actual .NET 10 code on top of it.
