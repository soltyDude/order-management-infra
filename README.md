# Order Management Platform

E-commerce order processing system built with microservices architecture. Handles the full order lifecycle — from placement through payment to inventory reservation — using event-driven communication and distributed transaction patterns.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client                               │
│                    POST /api/v1/orders                       │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│                   order-service :8080                        │
│              (Saga Orchestrator, CQRS)                       │
│                                                             │
│  • Accepts orders, manages lifecycle (state machine)        │
│  • Orchestrates CreateOrderSaga                             │
│  • Denormalized read model for zero-JOIN queries            │
│                                                             │
│  Patterns: Saga Orchestration · Outbox · State Machine      │
│            CQRS · Optimistic Locking                        │
├────────────┬────────────────────────────┬───────────────────┤
│            │          Kafka             │                    │
│     ┌──────┴──────┐             ┌──────┴──────┐            │
│     ▼             │             ▼             │            │
│ payment.charge    │      inventory.reserve    │            │
│ .requested        │      .requested           │            │
│     │             │             │             │            │
└─────┼─────────────┼─────────────┼─────────────┼────────────┘
      ▼             │             ▼             │
┌───────────────────┴──┐  ┌──────────────────────┴───────────┐
│ payment-service :8081│  │ inventory-service :8082           │
│                      │  │                                   │
│ • Idempotent payment │  │ • Atomic stock reservation        │
│   processing         │  │   (optimistic locking + retry)    │
│ • Strategy pattern   │  │ • Redis cache (60s TTL)           │
│   for payment methods│  │ • Reservation expiry (15 min TTL) │
│ • Circuit breaker on │  │ • Scheduled cleanup jobs          │
│   external gateway   │  │                                   │
│                      │  │ Patterns: Optimistic Lock ·       │
│ Patterns: Strategy · │  │   Cache-Aside · Outbox            │
│   Idempotency ·      │  │                                   │
│   Circuit Breaker ·  │  │         PostgreSQL                │
│   Outbox             │  │         + Redis                   │
│                      │  │                                   │
│      PostgreSQL      │  └───────────────────────────────────┘
└──────────────────────┘
```

Each service owns its database. No shared state. Services communicate exclusively through Kafka events — no synchronous inter-service HTTP calls in the primary flow.

## Saga Flow: Create Order

**Happy path:**

```
Client → POST /orders → order-service saves Order (PENDING)
                              │
                              ├──► Kafka: payment.charge.requested
                              │         payment-service processes charge
                              │    ◄── Kafka: payment.processed
                              │
                              │    Order → PAYMENT_CONFIRMED
                              │
                              ├──► Kafka: inventory.reserve.requested
                              │         inventory-service reserves stock
                              │    ◄── Kafka: inventory.reserved
                              │
                              │    Order → CONFIRMED ✓
```

**Compensation flows:**
- Payment fails → order cancelled, no inventory action needed
- Inventory fails → order cancelled, refund triggered via `payment.refund.requested`
- User cancels confirmed order → refund + stock release triggered in parallel

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 17 |
| Framework | Spring Boot 3.x |
| Databases | PostgreSQL 16 (per-service isolation) |
| Messaging | Apache Kafka (KRaft mode, no Zookeeper) |
| Cache | Redis 7 |
| Migrations | Flyway (SQL-based) |
| Auth | Stateless JWT (Spring Security) |
| Resilience | Resilience4j (circuit breaker, retry, bulkhead) |
| Tracing | Micrometer + OpenTelemetry → Zipkin |
| Metrics | Prometheus + Grafana |
| Build | Maven |
| Containers | Docker (multi-stage, ~180MB images) |
| CI | GitHub Actions |

## Repositories

| Repo | Description | Status |
|------|------------|--------|
| **order-management-infra** (this repo) | Docker Compose orchestration, documentation | ✅ Infrastructure ready |
| [order-service](https://github.com/soltyDude/order-service) | Saga orchestrator, order lifecycle, CQRS | ✅ Phase 1 complete |
| [payment-service](https://github.com/soltyDude/payment-service) | Idempotent payments, Strategy pattern | 🔲 Phase 2 planned |
| [inventory-service](https://github.com/soltyDude/inventory-service) | Stock management, optimistic locking, Redis cache | 🔲 Phase 3 planned |

## Project Status

**Phase 1 — order-service core: complete.** REST API works end-to-end, data persists in PostgreSQL, CQRS read model populates on writes. Outbox events are saved transactionally (Kafka publishing comes in Phase 4).

### Roadmap

| Phase | Scope | Status |
|-------|-------|--------|
| 0 — Foundation | Documentation, Docker Compose, service bootstraps | ✅ Done |
| 1 — order-service | Domain model, persistence, CQRS, REST API | ✅ Done |
| 2 — payment-service | Strategy pattern, idempotency, gateway stub | 🔲 Next |
| 3 — inventory-service | Optimistic locking, Redis cache, reservation expiry | 🔲 Planned |
| 4 — Kafka integration | Outbox poller, consumers, saga orchestration, DLQ | 🔲 Planned |
| 5 — Cross-cutting | JWT security, Resilience4j, distributed tracing, metrics | 🔲 Planned |
| 6 — Quality & DevOps | Testcontainers, ArchUnit, CI/CD, Dockerfiles | 🔲 Planned |

## Quick Start

```bash
# 1. Start infrastructure
git clone https://github.com/soltyDude/order-management-infra.git
cd order-management-infra
docker compose up -d

# 2. Verify infrastructure
docker compose ps          # all containers healthy
curl localhost:9411         # Zipkin UI
curl localhost:9090         # Prometheus UI
curl localhost:3000         # Grafana (admin/admin)

# 3. Start order-service
git clone https://github.com/soltyDude/order-service.git
cd order-service
mvn spring-boot:run

# 4. Test
curl localhost:8080/actuator/health
```

### Infrastructure Components

| Service | Port | Purpose |
|---------|------|---------|
| postgres-orders | 5432 | order-service database (`orders_db`) |
| postgres-payments | 5433 | payment-service database (`payments_db`) |
| postgres-inventory | 5434 | inventory-service database (`inventory_db`) |
| kafka | 9092 | Event streaming (KRaft mode) |
| redis | 6379 | Stock level cache |
| zipkin | 9411 | Distributed tracing UI |
| prometheus | 9090 | Metrics collection |
| grafana | 3000 | Monitoring dashboards |

## Key Design Decisions

Every architectural choice is documented in [ADR records](docs/adr-full.md). Highlights:

**Why Saga Orchestration over Choreography?** Create Order involves 3 steps with compensations. Orchestration keeps the entire flow in one place — easier to debug, log, and extend than choreography where logic is scattered across services. ([ADR-002](docs/adr-full.md))

**Why Outbox Pattern?** Saving an order and publishing a Kafka event must be atomic. Dual-write (save to DB, then send to Kafka) risks data loss if Kafka send fails after DB commit. Outbox writes the event to a database table in the same transaction, then a poller publishes it. ([ADR-003](docs/adr-full.md))

**Why CQRS?** GET endpoints need denormalized data (order + items + payment status + reservation status). The write model is normalized (orders + order_items). A separate read model table (`order_read_model`) serves reads with zero JOINs. ([ADR-004](docs/adr-full.md))

**Why separate databases?** True data isolation — impossible to accidentally create cross-service foreign keys or JOINs. Each service owns its data completely. ([ADR-010](docs/adr-full.md))

## API Overview

### order-service (port 8080)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/orders` | Create order (starts saga) |
| GET | `/api/v1/orders/{id}` | Order details (CQRS read model) |
| GET | `/api/v1/orders` | List user's orders (paginated) |
| PATCH | `/api/v1/orders/{id}/cancel` | Cancel order (triggers compensations) |
| GET | `/api/v1/orders/{id}/status` | Lightweight status polling |

### payment-service (port 8081)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/payments` | Process payment (idempotent) |
| GET | `/api/v1/payments/{id}` | Payment details |
| GET | `/api/v1/payments` | List payments (filtered) |
| POST | `/api/v1/payments/{id}/refund` | Manual refund (admin) |

### inventory-service (port 8082)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/inventory/{productId}` | Stock level (Redis cached) |
| GET | `/api/v1/inventory` | Product catalog (admin) |
| POST | `/api/v1/inventory/reserve` | Reserve stock (optimistic locking) |
| POST | `/api/v1/inventory/release` | Release reservation (compensation) |
| POST | `/api/v1/inventory/confirm` | Confirm reservation |
| PUT | `/api/v1/inventory/{productId}/stock` | Update stock (admin) |

## Order State Machine

```
PENDING ──► PAYMENT_PROCESSING ──► PAYMENT_CONFIRMED ──► INVENTORY_RESERVING ──► CONFIRMED ──► SHIPPED ──► DELIVERED
   │               │                      │                      │                    │
   └───────────────┴──────────────────────┴──────────────────────┴────────────────────┘
                                    CANCELLED (terminal)
```

Cancellable from: PENDING, PAYMENT_PROCESSING, PAYMENT_CONFIRMED, INVENTORY_RESERVING, CONFIRMED.
Not cancellable: SHIPPED, DELIVERED, CANCELLED.

Each transition is enforced by the state machine — invalid transitions throw `InvalidOrderStateException`.

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture](docs/architecture-full.md) | System design, service responsibilities, patterns |
| [API Contract](docs/api-contract-full.md) | Full REST API specification with examples |
| [Database Schema](docs/db-schema-full.md) | All tables, indexes, constraints, migrations |
| [Event Catalog](docs/event-catalog-full.md) | Kafka topics, event schemas, DLQ strategy |
| [Security](docs/security-full.md) | JWT auth, RBAC, ownership checks, CORS |
| [ADR](docs/adr-full.md) | Architecture Decision Records |
| [Conventions](docs/conventions-full.md) | Code style, naming, git workflow |
| [Glossary](docs/glossary-full.md) | Domain terminology |

## What This Project Demonstrates

**Distributed Systems Patterns:** Saga Orchestration with compensating transactions. Outbox Pattern for atomic event publishing. CQRS for optimized reads. Consumer idempotency via processed events table.

**Production Practices:** Optimistic locking for concurrent stock updates. Circuit breakers on external calls. Idempotency keys for safe retries. Dead Letter Queues for poison messages. Structured logging with correlation IDs.

**Code Quality:** State machine for order lifecycle — no invalid transitions possible. Strategy pattern for payment methods (OCP). Clean layering enforced by ArchUnit. BigDecimal for all monetary values. Flyway for version-controlled migrations.

**Infrastructure:** Database-per-service isolation. KRaft Kafka (no Zookeeper). Redis cache-aside with explicit eviction. Docker multi-stage builds. Health checks for container orchestration.
