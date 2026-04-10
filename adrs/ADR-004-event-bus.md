# ADR-004: Event Bus

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass uses a hybrid communication model (ADR-002) where domain events drive asynchronous side-effects: payment confirmations triggering order updates, order confirmations triggering QR code generation and emails, event cancellations triggering mass refunds and catalog updates. These events need to be reliable — a lost `PaymentSucceeded` event means a buyer pays but never receives their tickets.

Within the Modular Monolith, most events are processed in-process via an in-memory event bus. However, certain operations require durability and retry capability: background jobs (batch refund processing), scheduled tasks (reservation TTL expiration checks), and operations that must survive application restarts. Additionally, this event bus will become the inter-service communication backbone if modules are extracted into microservices (ADR-001).

The expected event volume during flash sales is ~1K events/min at peak, with ~50 events/min during normal operation.

## Options Considered

### Option 1: RabbitMQ

AMQP-based message broker with flexible routing, dead-letter queues, and acknowledgment-based delivery.

| Pros | Cons |
|------|------|
| Flexible routing with exchanges (direct, topic, fanout) — maps naturally to EventPass's event patterns | Not designed for event replay — once consumed, messages are gone (no log-based retention) |
| Native dead-letter queue support — failed events are automatically routed for inspection and retry | Requires operational management (though Amazon MQ provides a managed option) |
| Message acknowledgment ensures at-least-once delivery — critical for payment and refund events | Throughput ceiling (~50K msg/sec) is lower than Kafka, though far beyond our needs |
| Lower operational complexity than Kafka — no partition management, no consumer group coordination | Clustering adds complexity if high availability is needed (mitigated by Amazon MQ) |
| Amazon MQ provides a managed RabbitMQ service on AWS | Memory-based queuing — very large queues can cause memory pressure |

### Option 2: Apache Kafka

Distributed event streaming platform with log-based storage and replay capability.

| Pros | Cons |
|------|------|
| Event replay — can reprocess past events (useful for rebuilding read models like Catalog) | Significant operational complexity — partitions, consumer groups, offset management, ZooKeeper/KRaft |
| Massive throughput — handles millions of events/sec (EventPass needs ~1K/min) | Overkill for our scale — Kafka's strengths emerge at >100K events/sec |
| Built-in event retention — events persist for configurable periods | Learning curve for the team — Kafka's partition semantics require careful design |
| Strong ordering guarantees within partitions | Higher infrastructure cost — AWS MSK starts at ~$150/month for a minimal cluster |
| Excellent for event sourcing patterns | Cold starts for consumers are slower due to log replay |

### Option 3: AWS SQS + SNS

Fully managed AWS messaging services: SNS for pub/sub fan-out, SQS for durable queuing.

| Pros | Cons |
|------|------|
| Zero operational overhead — fully managed, auto-scaling, no capacity planning | No flexible routing — SNS topic filters are limited compared to RabbitMQ exchanges |
| Pay-per-message pricing — cost-effective at low volumes | SQS is pull-based — consumers must poll, adding latency (~100-200ms vs. RabbitMQ's push delivery) |
| Native AWS integration — IAM, CloudWatch, X-Ray | Two services to coordinate (SNS + SQS) instead of one |
| Built-in dead-letter queues on SQS | Vendor lock-in — tightly coupled to AWS, harder to run locally for development |
| Automatic retry with configurable visibility timeout | Message ordering requires FIFO queues, which have throughput limits (300 msg/sec per group) |

## Decision

**RabbitMQ** (deployed via Amazon MQ for managed operation).

At EventPass's event volume (~1K events/min peak), RabbitMQ provides the ideal balance of routing flexibility, delivery guarantees, and operational simplicity. Its exchange/queue model maps naturally to EventPass's event patterns:

- **Topic exchange** for domain events: `events.payment.succeeded`, `events.order.confirmed`, `events.ticket.reserved` — modules subscribe to relevant event patterns.
- **Direct exchange** for command-style messages: targeted background job dispatching (e.g., "process refund for order X").
- **Dead-letter exchange** for failed messages: events that fail processing after 3 retries are routed to a dead-letter queue for inspection and manual replay.

Within the Modular Monolith, the primary event bus is **in-memory** (in-process). RabbitMQ is used specifically for:
1. **Background jobs** that need durability (batch refund processing, payout calculations).
2. **Scheduled tasks** via delayed message plugin (reservation expiration checks).
3. **Future extraction** — when a module becomes an independent service, its events migrate from the in-memory bus to RabbitMQ with no architectural change.

## Consequences

**Enables:**
- Reliable event delivery for critical business operations — payment confirmations, refund processing, and ticket status updates are guaranteed at-least-once.
- Flexible event routing — new consumers (e.g., an analytics module) can subscribe to existing events without modifying publishers.
- Dead-letter queue pattern for operational visibility — failed events are captured, inspected, and replayed.
- Smooth extraction path — the in-memory event bus and RabbitMQ share the same publish/subscribe interface; switching is a configuration change.

**Prevents:**
- Event replay capability — unlike Kafka, consumed messages are removed. If we need to rebuild a read model (e.g., Catalog), we must re-emit events or use database snapshots.
- Massive scale — RabbitMQ handles ~50K msg/sec comfortably, but not millions. This is not a concern at our expected scale.

**Risks and Mitigations:**
- **Risk:** Message loss during RabbitMQ restart. **Mitigation:** Enable persistent messages and durable queues; Amazon MQ provides Multi-AZ deployment with automatic failover.
- **Risk:** Poison messages — an event that always fails processing blocks the queue. **Mitigation:** Dead-letter queue with max retry count (3 attempts); monitoring alerts when DLQ depth > 0; admin tooling for manual retry/discard.
- **Risk:** Event ordering — RabbitMQ does not guarantee strict ordering across consumers. **Mitigation:** For operations requiring ordering (e.g., ticket reservation before order creation), use synchronous facade calls. Events are used for side-effects where ordering is less critical.

## Related ADRs

- [ADR-001: Deployment Model](ADR-001-deployment-model.md) — Monolith uses in-memory bus primarily; RabbitMQ for durable/background jobs
- [ADR-002: Communication Style](ADR-002-communication-style.md) — Hybrid model defines when events vs. sync calls are used
- [ADR-003: Database Strategy](ADR-003-database-strategy.md) — Events enable eventual consistency between module schemas
