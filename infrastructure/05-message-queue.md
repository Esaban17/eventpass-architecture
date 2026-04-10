# Message Queue — RabbitMQ (Amazon MQ)

## 1. Responsibilities

- **Asynchronous payment processing** — Decouple Stripe webhook handling from order confirmation. When `payment_intent.succeeded` arrives, the Payment module publishes `PaymentSucceeded` to RabbitMQ for durable processing by the Orders module.
- **QR code generation** — After ticket purchase confirmation, the Ticketing module consumes `OrderConfirmed` events from RabbitMQ to generate QR codes asynchronously, avoiding blocking the checkout response.
- **Notification delivery** — The Notification module consumes domain events (`OrderConfirmed`, `PaymentFailed`, `EventCancelled`, `RefundCompleted`) from RabbitMQ and dispatches emails via SendGrid and SMS via Twilio.
- **Catalog index updates** — The Catalog module consumes `EventPublished`, `EventCancelled`, and `TicketInventoryUpdated` events to maintain its denormalized search index without coupling to upstream modules.
- **Dead-letter queue processing** — Failed messages (3 retries exhausted) are routed to dead-letter exchanges for inspection, alerting, and manual replay via admin tooling.

## 2. Why This Project Needs It

EventPass's ticket purchase flow generates a chain of side-effects: update order status → mark tickets as sold → generate QR codes → send confirmation email → update catalog. If these operations were synchronous, a SendGrid outage would block the checkout response — the buyer would see a timeout even though their payment succeeded. RabbitMQ ensures that side-effects are processed independently: if email delivery fails, the buyer still receives their order confirmation immediately, and the email is retried from the dead-letter queue.

During event cancellations with mass refunds (up to 2,000 orders), RabbitMQ queues `RefundRequested` events and the Payment module processes them at Stripe's rate limit (100 req/sec) without overwhelming the system.

## 3. Technology Choice

| Dimension | RabbitMQ (Amazon MQ) | Apache Kafka (MSK) | AWS SQS + SNS |
|-----------|---------------------|-------------------|---------------|
| **Managed/Self-hosted** | Managed (Amazon MQ) | Managed (Amazon MSK) | Fully managed (AWS) |
| **Operational complexity** | Low-moderate — Amazon MQ handles broker management | Moderate-high — partitions, consumer groups, retention policies | Low — zero infrastructure management |
| **Cost at our scale** | ~$90/month (mq.m5.large) | ~$150/month (minimum MSK cluster) | ~$5/month at our event volume |
| **Key differentiating feature** | Flexible routing (topic/direct/fanout exchanges), native dead-letter queues, message acknowledgment | Event replay, log-based retention, massive throughput (millions/sec) | Zero-ops, pay-per-message, native AWS integration |

**Decision: RabbitMQ (Amazon MQ)** — RabbitMQ's exchange/queue routing model maps naturally to EventPass's event patterns. Topic exchanges enable flexible subscriptions (`events.payment.*`, `events.order.*`), and native dead-letter queue support provides reliable failure handling. Kafka's event replay and massive throughput are overkill at ~1K events/min. SQS+SNS lacks flexible routing and requires managing two services.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Flexible routing via exchanges — topic, direct, and fanout patterns support all EventPass event flows | No event replay — once consumed, messages are gone (unlike Kafka's log-based retention) |
| Native dead-letter queues with configurable retry counts — essential for payment and refund reliability | Amazon MQ cost (~$90/month) is higher than SQS for low-volume scenarios |
| Message acknowledgment ensures at-least-once delivery — no lost payment confirmations | RabbitMQ's memory-based queuing can cause pressure under very large queue backlogs |
| Familiar AMQP protocol — team experience and extensive client library support (amqplib for Node.js) | Single broker is a SPOF — mitigated by Amazon MQ's Multi-AZ deployment |
| Amazon MQ handles broker management, patching, and Multi-AZ failover | Less cloud-native than SQS — does not auto-scale with traffic |

## 5. Integration

- **In-process event bus bridge:** Within the Modular Monolith, the primary event bus is in-memory (in-process). Events that require durability or background processing are bridged to RabbitMQ by the `shared/events/rabbitmq-event-bus.ts` adapter. See [ADR-004](../adrs/ADR-004-event-bus.md).
- **Exchange topology:**
  - `eventpass.topic` (topic exchange) — domain events routed by pattern: `events.payment.succeeded`, `events.order.confirmed`, etc.
  - `eventpass.dlx` (fanout exchange) — dead-letter exchange receiving messages after 3 failed processing attempts.
- **Consumer modules:** Payment (consumes `OrderCreated`), Orders (consumes `PaymentSucceeded`, `TicketReservationExpired`), Ticketing (consumes `OrderConfirmed`, `EventCancelled`), Notification (consumes all notification-triggering events), Catalog (consumes `EventPublished`, `TicketInventoryUpdated`).
- **Monitoring:** Queue depth, consumer count, and dead-letter queue depth exported to Prometheus via RabbitMQ's built-in metrics endpoint. Alerts configured in Grafana for DLQ depth > 0. See [11-observability-stack.md](11-observability-stack.md).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — Amazon MQ runs in the Managed Data Services zone.
