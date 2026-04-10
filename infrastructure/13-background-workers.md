# Background Workers — BullMQ (Node.js + Redis)

## 1. Responsibilities

- **Asynchronous task execution** — Process tasks that are too slow or too unreliable for the request-response cycle: QR code generation (~1-3s per ticket), transactional email delivery via SendGrid (~1-2s), SMS sending via Twilio (~1-2s), and payment retry attempts with exponential backoff (~3-10s).
- **Queue-based workload distribution** — Six purpose-built queues distribute work by domain concern: `ticket_generation`, `email`, `payment_retry`, `catalog_sync`, `refund_batch`, and `analytics`. Each queue has independent concurrency settings, retry policies, and dead-letter handling.
- **Batch processing for mass operations** — Process mass refunds when an event is cancelled: the `refund_batch` queue receives refund jobs in batches of 50, processes them sequentially with Stripe's API (respecting rate limits), and publishes `RefundCompleted` events for each successful refund.
- **Independent scaling from the web server** — Workers run the same Modular Monolith codebase in "worker mode" (`npm run start:worker`), deployed as separate ECS Fargate tasks that scale based on queue depth rather than HTTP traffic. During flash sales, `ticket_generation` and `catalog_sync` workers scale up independently.
- **Reliable job execution with retry and dead-letter** — Failed jobs are retried with exponential backoff (3 attempts, delays of 5s → 30s → 120s). After 3 failures, jobs move to the dead-letter queue (DLQ) for manual investigation via Bull Board UI. DLQ depth > 0 triggers a Slack alert.

## 2. Why This Project Needs It

EventPass's ticket purchase flow generates multiple side-effects that must not block the buyer's checkout experience. When a buyer completes a purchase, the system must: generate a QR code image (CPU-intensive, ~2s), upload it to S3, send a confirmation email with the QR attachment, send an SMS confirmation, and update the Catalog search index with the new inventory count. Executing all of these synchronously in the checkout response would add 5-10 seconds of latency — unacceptable for a platform competing for flash sale conversions.

Additionally, event cancellation triggers mass refunds that can involve hundreds or thousands of orders. Processing these synchronously would timeout any HTTP request. Background workers process refunds in controlled batches, respecting Stripe's rate limits (100 requests/second), and report progress through domain events that the Notification module consumes to inform each buyer.

The worker approach also isolates failures: if SendGrid is temporarily down, email jobs accumulate in the queue and are processed when the service recovers — the buyer's purchase confirmation is not affected.

## 3. Technology Choice

| Dimension | BullMQ (Node.js + Redis) | Celery (Python) | Temporal (Polyglot) |
|-----------|--------------------------|------------------|---------------------|
| **Managed/Self-hosted** | Self-hosted on ECS, backed by existing ElastiCache Redis | Self-hosted, requires Redis or RabbitMQ broker | Self-hosted or Temporal Cloud ($200+/month) |
| **Operational complexity** | Low — runs same Node.js codebase, uses existing Redis instance | High — requires Python runtime alongside Node.js stack, separate dependency management | High — new server component (Temporal server), new SDK, new deployment pipeline |
| **Cost at our scale** | ~$0 incremental (uses existing ElastiCache Redis; worker ECS tasks ~$20/month) | ~$20/month (worker tasks) + operational overhead of polyglot stack | ~$200+/month (Temporal Cloud) or ~$100/month (self-hosted on ECS) |
| **Key differentiating feature** | Same TypeScript codebase — workers import module logic directly, no serialization boundary. Bull Board UI for queue visibility | Mature ecosystem with 10+ years of production use, rich task routing and scheduling | Durable execution with automatic state persistence, ideal for long-running workflows spanning days |

**Decision: BullMQ (Node.js + Redis)** — BullMQ is the natural choice for EventPass's Modular Monolith because workers run the same TypeScript codebase in a different execution mode. A QR generation worker imports the Ticketing module's `QrCodeService` directly — no HTTP calls, no serialization, no API contracts to maintain. This aligns with the Modular Monolith philosophy (ADR-001): one codebase, multiple execution modes. Celery would introduce Python as a second runtime, requiring separate dependency management, testing pipelines, and deployment configurations. Temporal offers powerful durable execution semantics but adds a significant new infrastructure component — overkill for EventPass's relatively straightforward background tasks (none span multiple days or require human-in-the-loop workflows).

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Same TypeScript codebase — workers reuse module logic directly, no code duplication or API serialization boundary | Redis as job store is not durable to Redis restarts — mitigated by ElastiCache Multi-AZ with automatic failover |
| Backed by existing ElastiCache Redis — no new infrastructure component to deploy or manage | BullMQ is Node.js-only — if EventPass ever migrates a module to a different language, that module's workers must remain in Node.js or switch to a polyglot solution |
| Bull Board UI provides real-time visibility into queue depths, active jobs, failed jobs, and retry status | No built-in workflow orchestration — complex multi-step workflows (e.g., refund → notify → update catalog) are coordinated via domain events, not a workflow engine |
| Independent scaling — worker ECS tasks scale on queue depth, not HTTP traffic | Queue depth monitoring requires custom Prometheus metrics — not built into BullMQ out of the box (implemented via `bull-prom-metrics` library) |
| Exponential backoff retry with configurable attempts — resilient to transient external service failures (SendGrid, Twilio, Stripe) | Memory pressure on Redis during flash sales — ticket generation and catalog sync queues can accumulate thousands of jobs. Mitigated by queue concurrency limits and backpressure |

## 5. Integration

- **Execution mode:** Workers run the same Docker image as the web server with a different entrypoint: `CMD ["node", "dist/worker.js"]` instead of `CMD ["node", "dist/main.js"]`. The `worker.js` entrypoint initializes BullMQ processors for each queue instead of starting the HTTP server.
- **Queue definitions and responsibilities:**

| Queue | Trigger Event | Processing Time | Concurrency | Retry Policy |
|-------|--------------|-----------------|-------------|-------------|
| `ticket_generation` | `OrderConfirmed` | 1–3s per ticket | 5 concurrent | 3 attempts, backoff 5s→30s→120s |
| `email` | `OrderConfirmed`, `PaymentFailed`, `EventCancelled`, `RefundCompleted` | 1–2s | 10 concurrent | 3 attempts, backoff 5s→30s→120s |
| `payment_retry` | `PaymentFailed` | 3–10s | 3 concurrent | 3 attempts, backoff 30s→120s→300s |
| `catalog_sync` | `EventPublished`, `TicketInventoryUpdated`, `EventCancelled` | 1–5s | 5 concurrent | 3 attempts, backoff 5s→30s→120s |
| `refund_batch` | `EventCancelled` (produces batch of refund jobs) | 10–30s per batch of 50 | 2 concurrent | 3 attempts, backoff 60s→300s→600s |
| `analytics` | `OrderConfirmed`, `TicketValidated`, `EventCompleted` | 15–60s | 2 concurrent | 3 attempts, backoff 30s→120s→300s |

- **Scaling strategy:** Worker ECS tasks scale independently from web server tasks. During normal operation: 1 worker task handles all queues. During flash sales: auto-scaling adds 2–3 worker tasks when `ticket_generation` queue depth exceeds 100 jobs or `catalog_sync` queue depth exceeds 50 jobs.
- **Monitoring:** Custom Prometheus metrics exported via `bull-prom-metrics`: `bullmq_queue_completed_total`, `bullmq_queue_failed_total`, `bullmq_queue_active_count`, `bullmq_queue_waiting_count`. Bull Board UI exposed at `/admin/queues` (admin-only access via Auth0 RBAC). Alert rules: `bullmq_queue_waiting_count{queue="*_dlq"} > 0` → Slack #alerts. See [11-observability-stack.md](11-observability-stack.md).
- **Connection to RabbitMQ:** BullMQ uses Redis (not RabbitMQ) as its job store. RabbitMQ (ADR-004) handles durable inter-module domain events; BullMQ handles task-specific job processing. The flow: RabbitMQ delivers `OrderConfirmed` → Notification module's consumer enqueues an email job in BullMQ → BullMQ worker processes the job via SendGrid. This separation keeps domain event delivery (RabbitMQ) independent from task execution (BullMQ).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — Workers run as separate ECS Fargate tasks in the Private VPC, sharing the same Docker image as the API but with a different entrypoint.
