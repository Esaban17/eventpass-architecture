# ADR-002: Communication Style

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass's 7 bounded contexts need to communicate within a Modular Monolith (ADR-001). Some interactions require immediate responses (buyer browsing events, checking ticket availability), while others are side-effects that should not block the primary operation (sending confirmation emails, updating search indexes, generating QR codes).

The communication style determines latency characteristics, coupling between modules, failure handling complexity, and the ease of extracting modules into independent services in the future.

## Options Considered

### Option 1: Primarily Synchronous (REST-style Facade Calls)

All inter-module communication happens via direct facade method calls. Events are only used for notifications.

| Pros | Cons |
|------|------|
| Simple mental model — call a method, get a response | Tight temporal coupling — if the Payment module is slow, the checkout request blocks |
| Easy to trace execution flow with a debugger | Side-effects (email, catalog update) execute in the request path, increasing response time |
| No eventual consistency concerns | Difficult to extract modules later — each extraction requires converting calls to HTTP |
| Familiar pattern for most developers | The Notification module would need to be called synchronously after every order, adding latency |

### Option 2: Primarily Event-Driven

All inter-module communication happens via the internal event bus. REST is only used for external API queries.

| Pros | Cons |
|------|------|
| Maximum decoupling — modules react to events without knowing who published them | Simple queries become complex — checking ticket availability requires publishing an event and waiting for a response |
| Easy to add new consumers without modifying the publisher | Debugging event flows is harder — no single call stack to follow |
| Natural fit for future microservice extraction | Eventual consistency everywhere — the buyer sees "order pending" even after successful payment until the event is processed |
| Side-effects are naturally non-blocking | Overengineered for a monolith — in-process facade calls are simpler and faster for synchronous needs |

### Option 3: Hybrid — Synchronous for Commands/Queries, Asynchronous for Side-Effects

Direct facade calls for operations that need immediate responses. Internal event bus for operations that are side-effects or cross-cutting concerns.

| Pros | Cons |
|------|------|
| Best of both worlds — fast responses for user-facing operations, decoupled side-effects | Two communication patterns to understand and maintain |
| Non-blocking side-effects — email, catalog update, QR generation don't slow down checkout | Developers must decide which pattern to use for each interaction (requires clear guidelines) |
| Natural extraction path — synchronous calls become HTTP, events become RabbitMQ messages | Some flows mix both patterns (e.g., checkout is sync, but confirmation is async), adding complexity |
| Matches the actual domain — some operations are inherently sync (check availability), others are inherently async (send email) | Risk of inconsistent application if guidelines are not enforced |

## Decision

**Option 3: Hybrid communication — synchronous facade calls for commands and queries, asynchronous events for side-effects.**

This matches EventPass's domain reality. When a buyer checks ticket availability, they need an immediate answer — an event-driven approach would add unnecessary latency and complexity. When an order is confirmed, sending the confirmation email and updating the search index are side-effects that should not block the response to the buyer.

### Concrete Application in EventPass

**Synchronous (facade calls):**
- Browse events → `CatalogFacade.searchEvents()`
- Check ticket availability → `TicketingFacade.getAvailability()`
- Reserve tickets → `TicketingFacade.reserveTickets()`
- Create order → `OrdersFacade.createOrder()`
- Validate user token → `IdentityFacade.validateToken()`

**Asynchronous (internal event bus):**
- `PaymentSucceeded` → Orders confirms order, Ticketing marks tickets SOLD
- `OrderConfirmed` → Notification sends confirmation email, Ticketing generates QR codes
- `EventPublished` → Catalog updates event listing
- `EventCancelled` → Orders initiates mass refund, Ticketing cancels tickets, Catalog removes listing
- `UserRegistered` → Notification sends welcome email
- `TicketInventoryUpdated` → Catalog updates available ticket counts

### Decision Rule

> If the caller **needs the result to proceed**, use a synchronous facade call.
> If the operation is a **side-effect that another module should react to**, publish a domain event.

## Consequences

**Enables:**
- Fast user-facing responses — checkout flow completes without waiting for email delivery or search index updates.
- Clear separation between core transaction logic (sync) and reactive side-effects (async).
- Straightforward migration path — sync calls become HTTP requests, async events become RabbitMQ messages.
- New side-effects can be added by subscribing to existing events without modifying the publisher.

**Prevents:**
- Pure event-driven architecture — some operations will always be synchronous, which is appropriate for a monolith.
- Uniform communication pattern — developers must understand both patterns.

**Risks and Mitigations:**
- **Risk:** Inconsistent pattern application — a developer uses a sync call where an event would be more appropriate. **Mitigation:** Document the decision rule in the module development guidelines; review in PRs.
- **Risk:** Event handler failures go unnoticed. **Mitigation:** Structured logging for all event handlers; dead-letter mechanism for failed events; health monitoring via Grafana dashboards.
- **Risk:** Circular event chains (Module A publishes event → Module B reacts → publishes event → Module A reacts). **Mitigation:** Events represent facts about the past (e.g., `OrderConfirmed`), not commands. A module should never react to its own published events.

## Related ADRs

- [ADR-001: Deployment Model](ADR-001-deployment-model.md) — Modular Monolith determines in-process communication
- [ADR-004: Event Bus](ADR-004-event-bus.md) — RabbitMQ as the durable event transport for background jobs
- [ADR-007: API Design](ADR-007-api-design.md) — REST for external API, facade calls for internal sync communication
