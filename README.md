# EventPass — Event Ticketing Platform Architecture

Architecture documentation for **EventPass**, a multi-tenant event ticketing platform enabling event organizers to publish events, manage inventory, and receive payouts, while buyers browse, purchase, and receive QR-coded tickets.

This repository contains the complete software architecture design produced as a graduate-level Software Architecture assignment. All documents follow Domain-Driven Design (DDD) principles with a Modular Monolith deployment model. There is **no application code** — only architecture artifacts.

## Project Constraints

| Dimension | Value |
|-----------|-------|
| Team size | 3–5 developers |
| Monthly infrastructure budget | ~$410/month |
| Concurrent users (normal) | ~500 |
| Concurrent users (flash sales) | ~5,000 |
| Primary market | Latin America (Guatemala, Mexico, Colombia) |
| Consistency requirement | Strong (ticket inventory — no overselling) |

---

## Repository Navigation

| Section | Folder | Contents |
|---------|--------|----------|
| [Proposals](proposals/README.md) | `proposals/` | 4 architectural proposals covering system model, bounded contexts, module decomposition, and data flows |
| [Architecture Decision Records](adrs/README.md) | `adrs/` | 10 ADRs documenting key technology and design decisions with rationale |
| [Diagrams](diagrams/README.md) | `diagrams/` | 4 Mermaid diagrams: system context, context map, data flow sequences, and deployment topology |
| [Infrastructure](infrastructure/README.md) | `infrastructure/` | 12 infrastructure component documents covering every technology in the stack |
| [Backlog](docs/backlog/BACKLOG.md) | `docs/backlog/` | Master project backlog defining all phases, bounded contexts, and grading rubric |

---

## Architecture Overview

EventPass is designed as a **Modular Monolith** — a single deployable unit with strict internal module boundaries enforced by ESLint import rules and schema-per-module PostgreSQL isolation. This model was chosen over microservices because it delivers the required consistency guarantees (ACID transactions across modules for ticket purchase) while matching the team size and budget constraints of an early-stage startup.

### Bounded Contexts

The domain is decomposed into 7 bounded contexts following DDD Strategic Design:

| # | Bounded Context | Classification | Core Responsibility |
|---|----------------|----------------|---------------------|
| 1 | Identity & Access | Generic Subdomain | Authentication, authorization, RBAC via Auth0 |
| 2 | Event Management | Core Domain | Event lifecycle, organizer workflow, approval process |
| 3 | Catalog & Discovery | Supporting Subdomain | Event search, filtering, recommendations |
| 4 | Ticketing | Core Domain | Inventory management, QR code generation, validation |
| 5 | Order & Checkout | Core Domain | Purchase flow, reservation with timeout, order lifecycle |
| 6 | Payment | Supporting Subdomain | Stripe integration, refunds, organizer payouts via Connect |
| 7 | Notification | Generic Subdomain | Email (SendGrid) and SMS (Twilio) transactional delivery |

### Integration Patterns

Inter-context communication uses four DDD integration patterns:

- **Open Host Service / Published Language (OHS/PL)** — Ticketing and Event Management expose stable facades consumed by multiple contexts
- **Anti-Corruption Layer (ACL)** — Payment and Notification wrap external APIs (Stripe, Auth0, SendGrid, Twilio) behind domain interfaces
- **Partnership** — Order & Checkout and Ticketing coordinate via shared domain events with mutual schema agreement
- **Event-Driven** — Domain events (`TicketReserved`, `OrderConfirmed`, `PaymentSucceeded`) propagate state changes asynchronously via RabbitMQ

---

## Key Architecture Decisions

| Dimension | Decision | Reference |
|-----------|----------|-----------|
| Deployment model | Modular Monolith on AWS ECS Fargate | [ADR-001](adrs/ADR-001-deployment-model.md) |
| Communication style | Hybrid sync (REST) + async (RabbitMQ) | [ADR-002](adrs/ADR-002-communication-style.md) |
| Database strategy | Schema-per-module on single PostgreSQL (RDS) | [ADR-003](adrs/ADR-003-database-strategy.md) |
| Message broker | RabbitMQ via Amazon MQ | [ADR-004](adrs/ADR-004-event-bus.md) |
| Authentication | Auth0 (managed identity provider) | [ADR-005](adrs/ADR-005-authentication.md) |
| Observability | Grafana + Prometheus + Loki + Tempo | [ADR-006](adrs/ADR-006-observability.md) |
| API design | REST with versioned JSON endpoints | [ADR-007](adrs/ADR-007-api-design.md) |
| Caching | Redis via ElastiCache (atomic DECR for inventory) | [ADR-008](adrs/ADR-008-caching-strategy.md) |
| CI/CD | GitHub Actions + Docker + ECR + ECS blue-green | [ADR-009](adrs/ADR-009-deployment-cicd.md) |
| Frontend | Next.js App Router (SSR + ISR) | [ADR-010](adrs/ADR-010-frontend-architecture.md) |

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Next.js 14 (App Router) | SSR/ISR event pages, client-side checkout |
| Backend | Node.js + TypeScript (NestJS) | Modular monolith API |
| Primary database | PostgreSQL 16 (RDS Multi-AZ) | Schema-per-module ACID storage |
| Cache | Redis 7 (ElastiCache) | Inventory lock, sessions, search cache |
| Message queue | RabbitMQ (Amazon MQ) | Async event delivery, DLQ |
| Auth | Auth0 | JWT issuance, RBAC, MFA, social login |
| Payments | Stripe + Stripe Connect | Buyer payments, organizer payouts |
| Email | SendGrid | Transactional email with dynamic templates |
| SMS | Twilio | Ticket confirmations and event reminders |
| CDN | Cloudflare | Static assets, DDoS protection, bot management |
| API Gateway | Kong Gateway OSS | Rate limiting, JWT validation, CORS |
| Load balancer | AWS ALB | ECS task routing, health checks |
| Object storage | AWS S3 | QR codes, event images, DB backups |
| Observability | Grafana + Prometheus + Loki + Tempo | Metrics, logs, traces, SLO dashboards |
| CI/CD | GitHub Actions + Docker + ECR + ECS | Build, test, blue-green deploy |
| Container runtime | AWS ECS Fargate | Serverless container execution |

---

## Flash Sale Architecture

The platform's most critical requirement is supporting ~5,000 concurrent users during flash sales without overselling tickets. This is achieved through three layers:

1. **Redis atomic DECR** — `DECR ticket:{eventId}:available` returns the new count atomically. If negative, reject immediately without hitting the database.
2. **PostgreSQL SKIP LOCKED** — Reservation rows selected with `SELECT ... FOR UPDATE SKIP LOCKED` prevent concurrent transactions from blocking each other.
3. **Kong rate limiting** — Configurable per-endpoint limits (e.g., 10 reservation attempts per user per minute) protect against bot-driven inventory exhaustion.

---

## Repository Structure

```
eventpass-architecture/
├── README.md                          # This file
├── docs/
│   └── backlog/
│       └── BACKLOG.md                 # Master project backlog
├── proposals/
│   ├── README.md                      # Proposals index
│   ├── 01-high-level-architecture.md  # Modular Monolith vs Microservices
│   ├── 02-bounded-contexts.md         # 7 bounded contexts with entities
│   ├── 03-service-module-decomposition.md  # TypeScript module structure
│   └── 04-data-flow-and-interactions.md    # End-to-end flow diagrams
├── adrs/
│   ├── README.md                      # ADR index with decision tree
│   ├── ADR-001-deployment-model.md
│   ├── ADR-002-communication-style.md
│   ├── ADR-003-database-strategy.md
│   ├── ADR-004-event-bus.md
│   ├── ADR-005-authentication.md
│   ├── ADR-006-observability.md
│   ├── ADR-007-api-design.md
│   ├── ADR-008-caching-strategy.md
│   ├── ADR-009-deployment-cicd.md
│   └── ADR-010-frontend-architecture.md
├── diagrams/
│   ├── README.md                      # Diagram index
│   ├── 01-system-context.md           # C4 Level 1 context diagram
│   ├── 02-bounded-context-map.md      # DDD context map
│   ├── 03-data-flow.md               # Sequence diagrams
│   └── 04-deployment.md              # Infrastructure topology
└── infrastructure/
    ├── README.md                      # Component index + cost summary
    ├── 01-api-gateway.md              # Kong Gateway OSS
    ├── 02-cdn.md                      # Cloudflare
    ├── 03-load-balancer.md            # AWS ALB
    ├── 04-cache.md                    # Redis (ElastiCache)
    ├── 05-message-queue.md            # RabbitMQ (Amazon MQ)
    ├── 06-primary-database.md         # PostgreSQL (RDS)
    ├── 07-object-storage.md           # AWS S3
    ├── 08-auth-provider.md            # Auth0
    ├── 09-payment-provider.md         # Stripe
    ├── 10-notification-system.md      # SendGrid + Twilio
    ├── 11-observability-stack.md      # Grafana + Prometheus + Loki + Tempo
    └── 12-cicd-pipeline.md            # GitHub Actions + Docker + ECR + ECS
```
