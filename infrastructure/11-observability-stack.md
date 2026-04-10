# Observability Stack — Grafana + Prometheus + Loki + Tempo

## 1. Responsibilities

- **Metrics collection (Prometheus)** — Scrape application metrics from the `/metrics` endpoint on each ECS task every 15 seconds: HTTP request latency (p50, p95, p99), request count by endpoint and status code, active connections, ticket inventory levels, and queue depth.
- **Log aggregation (Loki)** — Collect structured JSON logs (via Pino logger) from all ECS tasks. Logs are labeled by module (identity, ticketing, orders, payment), severity (info, warn, error), and request ID for correlation.
- **Distributed tracing (Tempo)** — Collect OpenTelemetry traces spanning the full request lifecycle: API request → module facade calls → database queries → Redis operations → external API calls. Critical for debugging multi-module flows like checkout.
- **Unified visualization (Grafana)** — Single dashboard platform connecting all three data sources. Purpose-built dashboards for SLO monitoring, flash sale operations, payment health, and infrastructure capacity.
- **Alerting** — Prometheus Alertmanager triggers alerts to Slack and PagerDuty when SLOs are breached: p99 latency > 200ms (search), p99 > 500ms (checkout), error rate > 1%, or RabbitMQ DLQ depth > 0.

## 2. Why This Project Needs It

EventPass's ticket purchase flow crosses 4 bounded contexts (Ticketing → Orders → Payment → Notification) and involves external services (Stripe, SendGrid). When a buyer reports "my payment went through but I didn't get my ticket," debugging requires correlating: the Stripe webhook arrival time, the Payment module's event publishing, the Orders module's confirmation, the Ticketing module's QR generation, and the Notification module's email delivery. Without distributed tracing and correlated logs, this investigation would require manual log searching across multiple outputs — impractical during a flash sale with thousands of concurrent transactions.

SLO monitoring is equally critical: if checkout latency degrades during a flash sale, the team needs to identify the bottleneck (database? Redis? Stripe API?) within minutes, not hours.

## 3. Technology Choice

| Dimension | Grafana + Prometheus + Loki + Tempo | Datadog | ELK Stack (Elasticsearch + Logstash + Kibana) |
|-----------|-------------------------------------|---------|-----------------------------------------------|
| **Managed/Self-hosted** | Self-hosted on ECS (or AWS managed options for Prometheus/Grafana) | Fully managed SaaS | Self-hosted on EC2/ECS (or Elastic Cloud) |
| **Operational complexity** | Moderate — 4 components to deploy and configure | Low — zero infrastructure, auto-instrumentation | High — Elasticsearch cluster management, shard balancing |
| **Cost at our scale** | ~$50/month (ECS tasks + S3 storage) | ~$200-400/month (per-host + log ingestion + APM) | ~$200-500/month (Elasticsearch cluster on EC2) |
| **Key differentiating feature** | Open-source, no per-host pricing, Grafana's trace-to-log correlation | One-click setup, AI-powered anomaly detection, 600+ integrations | Powerful full-text log search, Kibana visualization, APM |

**Decision: Grafana + Prometheus + Loki + Tempo** — The open-source stack provides all three observability pillars at predictable cost. Datadog's per-host pricing ($15-31/host/month) would make flash sale scaling expensive: scaling from 2 to 8 ECS tasks temporarily triples the monitoring bill. ELK's Elasticsearch cluster requires significant memory and operational expertise for a team of 3-5 developers.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Predictable cost — no per-host or per-GB ingestion pricing that spikes during flash sales | Initial setup time — must deploy 4 components and configure dashboards manually |
| Grafana's trace-to-log-to-metrics correlation — click from a slow trace to related logs and metrics in one UI | No AI-powered anomaly detection — alerts are rule-based, not ML-driven (unlike Datadog) |
| Loki's label-based indexing — cheaper storage than Elasticsearch's full-text indexing | Loki is not suitable for full-text log search across all fields — search is label-scoped |
| Prometheus is the industry standard — extensive exporter ecosystem for PostgreSQL, Redis, RabbitMQ, Node.js | Prometheus's local storage is not highly available — mitigated by AWS Managed Prometheus (AMP) |
| Open-source — no vendor lock-in, community-supported dashboards and alerting templates | Dashboard creation requires Grafana expertise — no pre-built dashboards for EventPass-specific metrics |

## 5. Integration

- **Application instrumentation:**
  - Metrics: `prom-client` library exposes custom metrics at `/metrics` endpoint (request latency histogram, ticket inventory gauge, payment success/failure counter).
  - Logs: Pino logger outputs structured JSON with `module`, `requestId`, `traceId` fields. Loki agent collects from stdout.
  - Traces: `@opentelemetry/sdk-node` auto-instruments HTTP, PostgreSQL (`pg`), Redis (`ioredis`), and AMQP (`amqplib`) operations.
- **Key dashboards:**
  - **SLO Overview:** p99 latency for `/catalog/search` (<200ms target) and `/orders` (<500ms target), API availability %, error budget remaining.
  - **Flash Sale Monitor:** Concurrent users (active connections), ticket inventory depletion rate, reservation count, payment processing queue depth.
  - **Payment Health:** Stripe webhook latency, payment success/failure ratio by error type, refund completion rate, payout status.
  - **Infrastructure:** ECS task CPU/memory per task, PostgreSQL connections and query latency, Redis memory usage and hit rate, RabbitMQ queue depth and consumer count.
- **Alert rules:**
  - `slo:search_latency_p99 > 200ms for 5m` → Slack #alerts
  - `slo:checkout_latency_p99 > 500ms for 5m` → PagerDuty
  - `rate(http_requests_total{status=~"5.."}[2m]) > 0.01` → PagerDuty
  - `rabbitmq_queue_messages{queue="dlx"} > 0` → Slack #alerts
  - `ecs_task_memory_utilization > 85%` → Slack #infra
- **Storage backend:** Prometheus TSDB on EBS (15-day retention). Loki chunks and Tempo traces on S3 (30-day and 7-day retention respectively). S3 lifecycle policies archive to Glacier after 90 days. See [07-object-storage.md](07-object-storage.md).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — Observability stack runs as ECS tasks in the Private VPC, with Grafana exposed through the ALB for dashboard access.
