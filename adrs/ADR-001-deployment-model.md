# ADR-001: Deployment Model

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass needs a deployment architecture that supports 7 bounded contexts (Identity, Event Management, Catalog, Ticketing, Orders, Payment, Notification) while accommodating a team of 3-5 developers, a startup budget, and an MVP-stage product where domain boundaries are still being validated. The system must handle ~5K concurrent users during flash sales and ~500 during normal operation.

The deployment model determines how code is organized, built, deployed, and scaled — impacting development velocity, operational complexity, and infrastructure cost.

## Options Considered

### Option 1: Modular Monolith

A single deployable application composed of internally isolated modules, each mapping to one bounded context. Modules communicate in-process via facade calls (synchronous) and an internal event bus (asynchronous). All modules share a single PostgreSQL instance with schema-per-module isolation.

| Pros | Cons |
|------|------|
| Single CI/CD pipeline — one Docker image to build, test, and deploy | Cannot scale individual modules independently (e.g., Ticketing during flash sales vs. Notification) |
| In-process communication eliminates network latency between modules | A crash in one module crashes the entire process (mitigated by error boundaries and fast restarts) |
| Simplified debugging — full stack trace across module boundaries | Risk of accidental coupling if boundary enforcement (linting rules) is not maintained |
| Lower infrastructure cost — one ECS service, one RDS instance | All modules must share the same technology stack (TypeScript/Node.js) |
| Faster development for a 3-5 person team — no service coordination overhead | Deployment requires the entire application to restart, even for changes in a single module |

### Option 2: Microservices

Each bounded context deployed as an independent service with its own database, CI/CD pipeline, and scaling configuration. Services communicate over the network via REST/gRPC (synchronous) and RabbitMQ (asynchronous).

| Pros | Cons |
|------|------|
| Independent scaling — Ticketing can scale for flash sales without scaling Notification | 7 deployment pipelines, 7 ECS task definitions, 7 sets of CloudWatch alarms for a 3-5 person team |
| Hard service boundaries prevent accidental coupling by default | Network latency between services (5-15ms per call) adds up in multi-hop flows like checkout |
| Independent deployment — ship one service without touching others | Distributed tracing required to debug cross-service transaction failures |
| Technology flexibility — each service can use the optimal stack | Eventual consistency between services requires saga patterns and compensation logic |
| Fault isolation — one failing service does not crash others | Startup cost: minimum 7 always-on ECS tasks + 7 database connections + API gateway |

## Decision

**Modular Monolith.**

EventPass is an MVP-stage product with a team of 3-5 developers. The operational overhead of managing 7 independent services — separate pipelines, distributed debugging, network reliability patterns (circuit breakers, retries, timeouts), and saga coordination — would consume more engineering capacity than the team can afford.

The Modular Monolith provides the module isolation we need (schema-per-module, interface-based boundaries, event-driven communication) without the infrastructure complexity. At ~5K concurrent users during flash sales, a horizontally scaled monolith (3-5 ECS tasks behind an ALB) is sufficient — we don't need per-module scaling until traffic patterns diverge significantly.

The architecture is designed for future extraction: each module exposes only public interfaces, communicates via events, and owns its database schema. When a module needs independent scaling (e.g., Ticketing at >10K concurrent users) or the team grows beyond 6-7 developers, extracting that module into a separate service is an incremental operation, not a rewrite.

## Consequences

**Enables:**
- Rapid development velocity — developers focus on domain logic, not infrastructure plumbing.
- Simple operational model — one deployment to monitor, one set of logs, one health check.
- Low infrastructure cost — a single ECS Fargate service with 3-5 tasks handles the expected load.
- Straightforward debugging — full stack traces across module boundaries, no distributed tracing needed initially.

**Prevents:**
- Independent module scaling — all modules scale together. If Ticketing needs 10 instances during a flash sale, Notification also gets 10 instances (wasteful but tolerable at our scale).
- Independent deployment cadence — a change to the Notification module requires redeploying the entire application.
- Technology diversity — all modules must use TypeScript/Node.js.

**Risks and Mitigations:**
- **Risk:** Accidental coupling between modules over time. **Mitigation:** ESLint rules block imports from other modules' `internal/` directories; schema-per-module prevents cross-schema SQL queries; code reviews enforce boundary discipline.
- **Risk:** Single point of failure — a bug in one module can crash the entire process. **Mitigation:** ECS health checks detect crashes and restart containers within ~5 seconds; error boundaries in the API layer prevent unhandled exceptions from propagating.
- **Risk:** Monolith becomes too large to manage. **Mitigation:** Module extraction path is designed in — swap in-process event bus for RabbitMQ, wrap facade calls with HTTP clients, migrate schema to a new database instance.

## Related ADRs

- [ADR-002: Communication Style](ADR-002-communication-style.md) — defines sync vs. async patterns within the monolith
- [ADR-003: Database Strategy](ADR-003-database-strategy.md) — schema-per-module isolation within a single PostgreSQL
- [ADR-009: Deployment & CI/CD](ADR-009-deployment-cicd.md) — Docker + ECS Fargate pipeline for the monolith
