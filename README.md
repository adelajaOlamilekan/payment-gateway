# Payment Gateway Simulator

A backend payment gateway simulator built in Go, demonstrating core payment infrastructure patterns including idempotent payment processing, asynchronous job execution, webhook delivery with retry, and full transaction auditability.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Local Development](#local-development)
  - [Environment Variables](#environment-variables)
- [Database Migrations](#database-migrations)
- [Running the Services](#running-the-services)
- [API Reference](#api-reference)
- [Architecture](#architecture)
- [Testing](#testing)
- [Deployment](#deployment)
- [Roadmap](#roadmap)
- [Contributing](#contributing)

---

## Overview

The Payment Gateway Simulator is an MVP backend platform that enables merchants to create payment requests, process customer payments via a simulated processor, receive webhook notifications, and manage refunds.

This project is designed to demonstrate production-grade backend patterns in Go — not a real financial integration.

---

## Features

- Merchant registration and API key authentication
- Payment intent creation with idempotency enforcement
- Asynchronous payment processing via background workers
- Simulated payment processor (returns `success`, `failed`, or `pending`)
- Webhook delivery with HMAC-SHA256 signing and automatic retry
- Full and partial refund support
- Immutable transaction history and audit logging
- Admin endpoints for merchant and transaction monitoring
- Merchant data isolation enforced at every layer

---

## Tech Stack

| Layer                | Technology                                                  |
| -------------------- | ----------------------------------------------------------- |
| Language             | Go 1.22+                                                    |
| HTTP Router          | [chi](https://github.com/go-chi/chi)                        |
| Database             | PostgreSQL 15 (AWS RDS)                                     |
| DB Driver            | [pgx/v5](https://github.com/jackc/pgx)                      |
| Query Generation     | [sqlc](https://sqlc.dev)                                    |
| Migrations           | [golang-migrate](https://github.com/golang-migrate/migrate) |
| Job Queue            | [asynq](https://github.com/hibiken/asynq) (Redis-backed)    |
| Cache / Queue Broker | Redis 7 (AWS ElastiCache)                                   |
| Configuration        | [godotenv](https://github.com/joho/godotenv)                |
| IDs                  | [google/uuid](https://github.com/google/uuid)               |
| Testing              | [testify](https://github.com/stretchr/testify)              |

---

## Project Structure

```
payment-gateway/
├── cmd/
│   ├── api/                  # HTTP server entrypoint
│   │   └── main.go
│   └── worker/               # Background worker entrypoint
│       └── main.go
├── internal/
│   ├── config/               # Environment config loader
│   ├── db/
│   │   ├── migrations/       # SQL migration files
│   │   └── queries/          # SQL queries (input to sqlc)
│   ├── domain/               # Core domain structs and types
│   ├── handler/              # HTTP request handlers
│   ├── middleware/           # Auth, logging, recovery middleware
│   ├── repository/           # sqlc-generated database code
│   ├── service/              # Business logic layer
│   ├── worker/               # asynq task definitions and handlers
│   └── webhook/              # Signing, delivery, and retry logic
├── pkg/
│   └── idempotency/          # Redis-backed idempotency key store
├── docker-compose.yml        # Local Postgres + Redis
├── sqlc.yaml                 # sqlc configuration
├── Makefile
├── .env.example
└── README.md
```

---

## Getting Started

### Prerequisites

- [Go 1.22+](https://go.dev/dl/)
- [Docker & Docker Compose](https://docs.docker.com/get-docker/)
- [sqlc](https://docs.sqlc.dev/en/latest/overview/install.html) — `go install github.com/sqlc-dev/sqlc/cmd/sqlc@latest`
- [golang-migrate CLI](https://github.com/golang-migrate/migrate/tree/master/cmd/migrate)

### Local Development

1. **Clone the repository**

```bash
git clone https://github.com/your-username/payment-gateway.git
cd payment-gateway
```

2. **Copy the example environment file**

```bash
cp .env.example .env
```

3. **Start local infrastructure**

```bash
docker-compose up -d
```

This starts PostgreSQL on port `5432` and Redis on port `6379`.

4. **Install Go dependencies**

```bash
go mod download
```

5. **Run database migrations**

```bash
make migrate-up
```

6. **Generate sqlc query code**

```bash
make generate
```

7. **Start the API server**

```bash
make run-api
```

8. **Start the background worker** (separate terminal)

```bash
make run-worker
```

### Environment Variables

Copy `.env.example` to `.env` and fill in the values.

```env
# Server
APP_ENV=development
API_PORT=8080

# Database
DATABASE_URL=postgres://postgres:postgres@localhost:5432/payment_gateway?sslmode=disable

# Redis
REDIS_ADDR=localhost:6379

# Security
WEBHOOK_SIGNING_SECRET=your-signing-secret-here
API_KEY_SALT_ROUNDS=12

# Simulated Processor
PROCESSOR_SUCCESS_RATE=0.85
PROCESSOR_LATENCY_MS=200
```

---

## Database Migrations

```bash
# Run all pending migrations
make migrate-up

# Roll back the last migration
make migrate-down

# Create a new migration file
make migrate-create name=add_webhook_attempts_table
```

---

## Running the Services

The project runs as two separate processes:

| Process           | Command           | Description                                            |
| ----------------- | ----------------- | ------------------------------------------------------ |
| API server        | `make run-api`    | Handles all HTTP requests                              |
| Background worker | `make run-worker` | Processes payments, delivers webhooks, handles retries |

Both must be running for the full payment flow to work.

---

## API Reference

### Authentication

All merchant endpoints require an `X-API-Key` header.

```
X-API-Key: mk_live_xxxxxxxxxxxxxxxxxxxx
```

### Endpoints

#### Merchants

| Method | Path                  | Description                        |
| ------ | --------------------- | ---------------------------------- |
| `POST` | `/merchants/register` | Register a new merchant            |
| `GET`  | `/merchants/me`       | Get authenticated merchant profile |

#### Payment Intents

| Method | Path                            | Description                 |
| ------ | ------------------------------- | --------------------------- |
| `POST` | `/payments/intents`             | Create a payment intent     |
| `GET`  | `/payments/intents`             | List payment intents        |
| `GET`  | `/payments/intents/:id`         | Get a payment intent        |
| `POST` | `/payments/intents/:id/process` | Initiate payment processing |

#### Transactions

| Method | Path                | Description       |
| ------ | ------------------- | ----------------- |
| `GET`  | `/transactions`     | List transactions |
| `GET`  | `/transactions/:id` | Get a transaction |

#### Refunds

| Method | Path       | Description                      |
| ------ | ---------- | -------------------------------- |
| `POST` | `/refunds` | Request a full or partial refund |

#### Admin

| Method  | Path                          | Description                    |
| ------- | ----------------------------- | ------------------------------ |
| `GET`   | `/admin/merchants`            | List all merchants             |
| `PATCH` | `/admin/merchants/:id/status` | Activate or suspend a merchant |
| `GET`   | `/admin/transactions`         | List all transactions          |
| `GET`   | `/admin/webhooks`             | View webhook delivery logs     |

### Idempotency

Payment intent creation supports idempotent requests. Pass a unique key per request:

```
Idempotency-Key: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

Duplicate requests with the same key within 24 hours return the original response.

---

## Architecture

The system separates the synchronous API layer from asynchronous processing:

1. The API server accepts requests, validates them, and writes to PostgreSQL.
2. Payment processing is enqueued as a background job — the API returns `202 Accepted` immediately.
3. The worker picks up the job, calls the simulated processor, updates the transaction state, and enqueues a webhook delivery task.
4. The webhook worker signs the payload, delivers it to the merchant's URL, and retries on failure with exponential backoff.
5. All state transitions are written to an immutable audit log.

**Payment intent status flow:**

```
created → processing → succeeded → refunded
                                 ↘ partially_refunded
               ↘ failed
created → cancelled
```

---

## Testing

```bash
# Run all tests
make test

# Run with coverage report
make test-coverage

# Run a specific package
go test ./internal/service/...
```

Key test areas:

- Idempotency key deduplication
- Payment state transition guards
- Webhook HMAC signature verification
- Refund amount validation
- Async worker retry logic

---

## Deployment

The application is designed for AWS deployment. Recommended setup:

| Service      | AWS Component                                          |
| ------------ | ------------------------------------------------------ |
| API + Worker | ECS Fargate (separate task definitions)                |
| PostgreSQL   | RDS (PostgreSQL 15)                                    |
| Redis        | ElastiCache (Redis 7)                                  |
| Secrets      | AWS Secrets Manager or Parameter Store                 |
| Region       | `af-south-1` (Cape Town) — lowest latency from Nigeria |

**Build the Docker image:**

```bash
docker build -t payment-gateway .
```

**Push to ECR and deploy via ECS** using your preferred CI/CD pipeline (GitHub Actions recommended).

---

## Roadmap

- [ ] Multi-provider payment routing
- [ ] Rate limiting per merchant
- [ ] Circuit breaker for processor calls
- [ ] Fraud scoring engine
- [ ] Ledger accounting and settlement batching
- [ ] Multi-currency support
- [ ] Monitoring dashboard
- [ ] Distributed tracing (OpenTelemetry)
- [ ] Event-driven architecture (Kafka/SNS)

---

## Contributing

This is an MVP project. Contributions, feedback, and issue reports are welcome.

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/your-feature`
3. Commit your changes: `git commit -m 'feat: add your feature'`
4. Push to the branch: `git push origin feat/your-feature`
5. Open a pull request

---

## License

MIT
