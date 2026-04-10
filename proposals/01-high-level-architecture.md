# High-Level Architecture: Modular Monolith vs. Microservices

## 1. Introduction

EventPass is an event ticketing platform that connects organizers creating live events with buyers who discover, purchase tickets, and attend. The system must handle multiple user roles (Buyer, Organizer, Admin), complex financial flows (Stripe payments, refunds, organizer payouts), high-concurrency flash sales (~5K concurrent users), and integrations with external services (Stripe, SendGrid, Twilio, Google Maps, Cloudflare).

This document evaluates two architectural approaches — **Modular Monolith** and **Microservices** — specifically in the context of EventPass, and recommends the approach that best fits our current constraints.

---

## 2. Approach A: Modular Monolith

### How It Applies to EventPass

In a Modular Monolith, EventPass is deployed as a **single application** composed of internally isolated modules — each module mapping to one of our 7 bounded contexts (Identity, Event Management, Catalog, Ticketing, Orders, Payment, Notification).

All modules run in the same process and share a single PostgreSQL instance (with **schema-per-module** isolation). Communication between modules happens through an **in-process event bus** for asynchronous side-effects (e.g., `OrderConfirmed` triggers ticket status updates and email notifications) and direct interface calls for synchronous queries (e.g., Catalog reads event data through a defined public API exposed by the Event Management module).

**Concretely for EventPass:**

- The **Ticketing module** manages seat reservations with 10-minute TTL locks using Redis. Because it runs in-process with the **Order module**, the reservation-to-checkout flow avoids network overhead — the Order module calls Ticketing's public interface directly to confirm or release reservations.
- The **Payment module** wraps Stripe's API behind an Anti-Corruption Layer (ACL). When a Stripe webhook confirms a payment, the Payment module publishes a `PaymentSucceeded` event on the internal bus. The Order module reacts by confirming the order, which in turn publishes `OrderConfirmed` for Ticketing and Notification to consume — all within the same process, with sub-millisecond event delivery.
- **Flash sale concurrency** (~5K users competing for limited inventory) is handled at the database level with `SELECT ... FOR UPDATE SKIP LOCKED` on the ticket inventory table within the Ticketing module's schema, combined with Redis-backed rate limiting in the API middleware. No distributed locking or cross-service coordination is needed.
- The entire application is packaged as a single Docker image and deployed to **AWS ECS Fargate**, scaling horizontally by adding container instances behind an ALB. Each instance runs the full application — no need to manage per-service scaling policies.

### Key Characteristics

- **Single deployable unit**: One Docker image, one CI/CD pipeline, one deployment target.
- **In-process communication**: Module-to-module calls are function invocations, not HTTP/gRPC requests.
- **Schema-per-module isolation**: Each bounded context owns its PostgreSQL schema, preventing accidental cross-context data coupling while sharing the same database instance.
- **Shared infrastructure**: One connection pool, one cache client, one message bus configuration.
- **Boundary enforcement via code conventions**: Modules expose only public interfaces (`/public` directory); linting rules prevent direct imports of internal module code.

---

## 3. Approach B: Microservices

### How It Applies to EventPass

In a Microservices architecture, each of EventPass's 7 bounded contexts becomes an **independently deployable service** with its own database, its own CI/CD pipeline, and its own scaling configuration.

The services communicate over the network: synchronous calls via REST or gRPC for queries (e.g., Catalog service calls Event Management service to fetch event details) and asynchronous messaging via RabbitMQ for event-driven workflows (e.g., `PaymentSucceeded` event routed from Payment service to Order service).

**Concretely for EventPass:**

- The **Ticketing service** would own its own PostgreSQL database and Redis instance. When a buyer selects tickets, the **Order service** must make an HTTP call to the Ticketing service to create a reservation. This introduces network latency (~5-15ms per call), potential timeout handling, and the need for a circuit breaker pattern in case the Ticketing service is temporarily unavailable.
- The **Payment service** still wraps Stripe behind an ACL, but now `PaymentSucceeded` must be published to RabbitMQ and consumed by the Order service running in a separate process. This introduces eventual consistency — there is a window (typically 50-200ms, but potentially longer under load) where the payment has succeeded but the order is not yet confirmed.
- **Flash sale concurrency** requires distributed coordination. The Ticketing service handles inventory locks locally, but the Order service must handle the case where the Ticketing service's reservation response arrives too late (network jitter) or the Ticketing service is overloaded and shedding requests. Retry logic, idempotency keys, and dead-letter queues become essential.
- Each service is deployed independently to ECS Fargate with its own task definition, auto-scaling policy, and health checks. With 7 services, this means 7 deployment pipelines, 7 sets of CloudWatch alarms, and 7 separate log streams to correlate during debugging.

### Key Characteristics

- **Independent deployment**: Each service ships on its own schedule — valuable for large teams working in parallel.
- **Independent scaling**: The Ticketing service can scale to handle flash sales without scaling the Notification service.
- **Technology heterogeneity**: Each service could use a different language or database — though this introduces operational complexity.
- **Network-based communication**: All inter-service calls traverse the network, requiring retry logic, timeouts, and circuit breakers.
- **Operational overhead**: Service mesh, distributed tracing, centralized logging, and API gateway configuration multiply with each new service.

---

## 4. Comparison

| Dimension | Modular Monolith | Microservices |
|-----------|-----------------|---------------|
| **Deployment Complexity** | Low — single Docker image, one ECS service, one CI/CD pipeline | High — 7 separate deployments, 7 pipelines, coordinated releases for breaking changes |
| **Team Size Fit** | Excellent for 3-5 devs — everyone works in one repo, no cross-service coordination overhead | Optimized for 7+ teams — each team owns a service, but our team of 3-5 would be spread thin across 7 services |
| **Horizontal Scalability** | Good — scales the entire application; sufficient for ~5K concurrent users (our target) | Excellent — fine-grained per-service scaling, but only beneficial at scales we don't yet need |
| **Fault Isolation** | Moderate — a crash in one module crashes the process, but restart is fast (~2s) and scope is limited with proper error boundaries | High — a failing service doesn't crash others, but cascading failures via network calls are a real risk without circuit breakers |
| **Development Speed (Initial)** | Fast — no network protocols to design, no service discovery, no distributed debugging; developers focus on domain logic | Slow — significant upfront investment in infrastructure: API contracts, service mesh, distributed tracing, deployment automation |
| **Development Speed (Long-term)** | Good — module boundaries prevent coupling; if boundaries erode, refactoring is local. Risk: module boundaries may weaken without discipline | Excellent — hard service boundaries prevent coupling by default, but cross-service features require coordinated releases and contract negotiation |
| **Infrastructure Cost** | Low — one ECS service, one RDS instance, one ElastiCache cluster, one Amazon MQ broker | High — 7 ECS services (minimum 1 task each = 7 always-on tasks), potentially 7 database connections, per-service monitoring |
| **Operational Complexity** | Low — one set of logs, one deployment to monitor, one health check endpoint, straightforward debugging with stack traces | High — distributed tracing required (Tempo), log correlation across 7 services, complex debugging for cross-service transaction failures |

---

## 5. Recommendation: Modular Monolith

**We recommend the Modular Monolith architecture for EventPass.** This decision is driven by four concrete constraints:

### 5.1 Team Size (3-5 Developers)

A team of 3-5 developers cannot effectively own and operate 7 independent services. Each microservice requires its own deployment pipeline, monitoring, on-call rotation awareness, and API contract management. With 3 developers, at least 2 people would need to context-switch across 4+ services regularly — the cognitive overhead would consume more engineering time than the independence would save.

In a Modular Monolith, the same team works in a single repository. A developer working on the Ticketing module can trace a bug through Order and Payment modules with a single debugger session and a single set of logs.

### 5.2 Product Phase (Early MVP)

EventPass is in its **initial product phase** — we are still validating the business model, refining user flows, and discovering domain nuances. At this stage, bounded context boundaries may shift. For example, we might discover that "Catalog & Discovery" should absorb recommendation logic currently sketched in "Event Management," or that "Ticketing" and "Order & Checkout" should merge into a single module.

In a Modular Monolith, moving code between modules is a refactoring task — rename directories, update imports, run tests. In Microservices, the same change requires migrating databases, updating API contracts, coordinating deployments, and re-routing events. The cost of getting boundaries wrong is 10x higher with microservices.

### 5.3 Domain Still Under Discovery

The 7 bounded contexts defined in this architecture represent our **current best understanding** of the domain. As EventPass evolves, we may need to split the Ticketing context (e.g., separating "Seat Management" from "Inventory") or merge contexts that turn out to be artificially separated.

The Modular Monolith lets us experiment with boundaries cheaply. Modules communicate through defined interfaces and an internal event bus — the same abstractions that would power microservice communication later. This means the migration path is clear and incremental.

### 5.4 Migration Path to Microservices

The Modular Monolith is not a dead end. The architecture is designed with microservice extraction in mind:

1. **Schema-per-module** on PostgreSQL means each module already owns its data. Extracting a module means pointing it at a new database instance and running the existing migrations.
2. **Internal event bus** uses the same publish/subscribe semantics as RabbitMQ. Swapping the in-process bus for a network-based broker is a configuration change, not a rewrite.
3. **Public interfaces** per module define the API contract. Wrapping these interfaces in REST or gRPC endpoints converts a module into a service.

The trigger for extraction would be: a single module's scaling requirements diverge significantly from the rest (e.g., Ticketing during flash sales), or the team grows beyond 6-7 developers and needs independent deployment cadences.

---

## 6. Summary

| Aspect | Our Choice |
|--------|-----------|
| **Architecture** | Modular Monolith |
| **Rationale** | Team of 3-5, early MVP, domain in discovery, clear migration path |
| **Deployment** | Single Docker image → AWS ECS Fargate |
| **Module Isolation** | Schema-per-module (PostgreSQL), interface-based boundaries, internal event bus |
| **Scaling Strategy** | Horizontal scaling of the entire application via ECS task count |
| **Migration Trigger** | Team >6 devs or module-specific scaling needs beyond 10K concurrent users |
