# Merezh Platform

A microservices platform for managing users, wallets, orders, and payments.

📖 In Russian: [перевод на русский](https://github.com/CkutlsGit/merezh/blob/main/README.ru.md)

## 📋 Overview

Merezh is a learning microservices project built with **Java 21 + Spring Boot 3**. The platform consists of **six services** communicating over **HTTP**. The project demonstrates:

- separation into bounded contexts,
- a single entry point (Gateway) with JWT authentication,
- a distributed saga `order → payment → wallet → payment → order`,
- idempotency, pessimistic locking, health checks.

The project is built as a **portfolio** to showcase microservices design skills.

## 🏗️ Architecture

```
                        ┌─────────────────────┐
                        │      Client         │
                        └──────────┬──────────┘
                                   │ HTTP
                                   ▼
                        ┌─────────────────────┐
                        │      Gateway        │  ← single entry point
                        │      :8000          │  ← JWT, roles, routing
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
      ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
      │  User Service │    │  Auth Service │    │ Wallet Service│
      │    :8080      │◄───│    :8081      │    │    :8083      │
      └───────┬───────┘    └───────┬───────┘    └───────┬───────┘
              │                    │                    │
              ▼                    ▼                    ▼
      ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
      │   Postgres    │    │   Postgres    │    │   Postgres    │
      │  userservice  │    │  authservice  │    │  walletservice│
      └───────────────┘    └───────────────┘    └───────────────┘

              ┌────────────────────┐
              │                    │
              ▼                    ▼
      ┌───────────────┐    ┌───────────────┐
      │ Order Service │───▶│Payment Service│
      │    :8085      │    │    :8084      │
      └───────┬───────┘    └───────┬───────┘
              │                    │
              ▼                    ▼
      ┌───────────────┐    ┌───────────────┐
      │   Postgres    │    │   Postgres    │
      │  orderservice │    │paymentservice │
      └───────────────┘    └───────────────┘
```

**All services live in the Docker `dbnet` network.** Only the **Gateway** port is exposed (`127.0.0.1:8000`). Internal services are not accessible from outside.

## 🚀 Technology Stack

**Backend**

- Java 21 - core language
- Spring Boot 3 - application framework
- Spring Data JPA - database access and ORM
- Spring Security - authentication and authorization (in Gateway)
- Spring Security Crypto - BCrypt
- JJWT - JWT tokens
- RestTemplate - synchronous HTTP calls
- Jakarta Validation - request validation
- Lombok - boilerplate reduction

**Databases**

- PostgreSQL - one database per service (database-per-service)

**DevOps**

- Docker - containerization
- Docker Compose - orchestration
- Spring Boot Actuator - health checks (liveness/readiness)

**Testing**

- JUnit 5
- Mockito

## 🧩 Services

| Service             | Port | Purpose                                          | README                               |
|---------------------|------|--------------------------------------------------|--------------------------------------|
| **Gateway**         | 8000 | Single entry point, JWT, routing                 | [README](https://github.com/CkutlsGit/merezh-gateway)  |
| **User Service**    | 8080 | User storage and management                      | [README](https://github.com/CkutlsGit/merezh-userservice)     |
| **Auth Service**    | 8081 | Authentication, JWT, refresh tokens              | [README](https://github.com/CkutlsGit/merezh-authservice)     |
| **Wallet Service**  | 8083 | Wallets and balances                             | [README](https://github.com/CkutlsGit/merezh-walletservice)   |
| **Order Service**   | 8085 | Orders and order items                           | [README](https://github.com/CkutlsGit/merezh-orderservice)    |
| **Payment Service** | 8084 | Payment processing                               | [README](https://github.com/CkutlsGit/merezh-paymentservice)  |

## 🛠️ Quick Start

### Prerequisites

- Docker
- Docker Compose

### Run
Copy each service repository and run the `docker compose up --build` command in the gateway.

## 📚 API

The single entry point is the **Gateway** (`http://localhost:8000`). All requests go through it.

### Authentication

| Method | Endpoint                | Description                       | Access        |
|--------|-------------------------|-----------------------------------|---------------|
| POST   | `/api/v1/auth/register` | Register                          | Public        |
| POST   | `/api/v1/auth/login`    | Login, get a token pair           | Public        |
| POST   | `/api/v1/auth/refresh`  | Refresh tokens using refresh      | Public        |
| POST   | `/api/v1/auth/logout`   | Logout                            | Authenticated |

### Users

| Method | Endpoint              | Description              | Access |
|--------|-----------------------|--------------------------|--------|
| GET    | `/api/v1/users`       | Get all users            | ADMIN  |
| GET    | `/api/v1/users/{id}`  | Get user by ID           | ADMIN  |
| DELETE | `/api/v1/users/{id}`  | Delete user              | ADMIN  |

### Wallets

| Method | Endpoint                       | Description          | Access        |
|--------|--------------------------------|----------------------|---------------|
| POST   | `/api/v1/wallets/create`       | Create a wallet      | Authenticated |
| GET    | `/api/v1/wallets/balance`      | Get balance          | Authenticated |
| POST   | `/api/v1/wallets/balance/sum`  | Top up balance       | Authenticated |
| POST   | `/api/v1/wallets/balance/sub`  | Withdraw funds       | Authenticated |
| DELETE | `/api/v1/wallets/delete/{id}`  | Delete a wallet      | ADMIN         |

### Orders

| Method | Endpoint                  | Description                    | Access        |
|--------|---------------------------|--------------------------------|---------------|
| GET    | `/api/v1/orders/get/{id}` | Get order by ID                | Authenticated |
| GET    | `/api/v1/orders/get/user` | Get user orders                | Authenticated |
| POST   | `/api/v1/orders/create`   | Create an order                | Authenticated |

### Payments

| Method | Endpoint                         | Description               | Access        |
|--------|----------------------------------|---------------------------|---------------|
| GET    | `/api/v1/payments/get/{id}`      | Get payment by ID         | Authenticated |
| GET    | `/api/v1/payments/get/user`      | Get user payments         | Authenticated |
| POST   | `/api/v1/payments/pay/{orderId}` | Pay for an order          | Authenticated |

**Internal endpoints** (`/users/create`, `/users/validate`, `/orders/update`, `/payments/place`) are **not exposed** through the Gateway - they are only for service-to-service communication.

## 🔄 Saga: Order Lifecycle

The platform implements a **distributed saga** (choreography) for order creation and payment:

```
┌──────────┐   1. POST /create        ┌──────────────┐
│  Client  │─────────────────────────▶│ Order Service│
└──────────┘                          └──────┬───────┘
                                             │
                                             │ 2. save Order(PAYMENT_WAITING)
                                             │
                                             │ 3. POST /place
                                             ▼
                                      ┌──────────────┐
                                      │Payment Service│
                                      └──────┬───────┘
                                             │
                                             │ 4. save Payment(WAITING)
                                             │
                                             │ 5. return WAITING
                                             ▼
                                      ┌──────────────┐
                                      │ Order Service│
                                      └──────────────┘

  === Later, the user pays ===

┌──────────┐   6. POST /pay/{orderId}  ┌──────────────┐
│  Client  │─────────────────────────▶│Payment Service│
└──────────┘                          └──────┬───────┘
                                             │
                                             │ 7. POST /balance/sub
                                             ▼
                                      ┌──────────────┐
                                      │Wallet Service│
                                      └──────┬───────┘
                                             │
                                             │ 8. debit
                                             │
                                             │ 9. return balance
                                             ▼
                                      ┌──────────────┐
                                      │Payment Service│
                                      └──────┬───────┘
                                             │
                                             │ 10. Payment = SUCCESS
                                             │
                                             │ 11. POST /update (finally)
                                             ▼
                                      ┌──────────────┐
                                      │ Order Service│
                                      └──────┬───────┘
                                             │
                                             │ 12. Order = PAYMENT_SUCCESS
                                             ▼
```

**Implementation notes:**

- `@Transactional(noRollbackFor = {HttpClientErrorException, HttpServerErrorException})` in Payment - status is persisted even on a network error.
- Callback to Order Service is **always** executed via `finally`.
- On 4xx from Wallet - payment is marked `FAILED`; on 5xx - `WAITING`.
- On repeated `/place` for an existing payment, `existsOrder` is triggered (retry of the debit if `WAITING`).

## 🔒 Security

- **Gateway is the single entry point.** Internal services run only inside the Docker `dbnet` network.
- **JWT authentication.** Access (30 min) + Refresh (7 days). HS256 signature.
- **Refresh token rotation.** On every `/refresh`, the old hash is replaced with a new one.
- **Refresh tokens are stored as SHA-256 hashes** - in case of a DB leak, raw tokens are not compromised.
- **Passwords are stored as BCrypt hashes.** Hashing in Auth Service.
- **`X-User-Id` spoofing is impossible** - the client header is stripped at the Gateway.
- **Role-based authorization** - `hasRole("ADMIN")` for admin endpoints.
- **Internal endpoints are not proxied** - accessible only inside `dbnet`.

## ⚙️ Configuration

All secrets (DB passwords, JWT secret) are set in `docker-compose.yml` (in plain text for this learning project).
