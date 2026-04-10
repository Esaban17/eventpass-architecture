# ADR-006: Observability

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass requires observability across three pillars — metrics, logs, and traces — to monitor system health, debug production issues, and validate SLOs. Critical operations that require monitoring include:

- **Ticket purchase flow:** p99 latency for checkout must stay below 500ms. Reservation TTL (10 min) expiration must be tracked.
- **Flash sale behavior:** Concurrent user count, ticket inventory depletion rate, payment success/failure ratio.
- **Payment processing:** Stripe webhook delivery latency, refund success rate, payout completion rate.
- **Infrastructure health:** ECS task CPU/memory, PostgreSQL connection pool usage, Redis hit rate, RabbitMQ queue depth.

The observability stack must be cost-effective for a startup (no per-host or per-GB pricing that scales unpredictably) and manageable by a 3-5 person team without a dedicated SRE.

### Target SLOs

| Metric | Target |
|--------|--------|
| Event search (p99 latency) | < 200ms |
| Checkout flow (p99 latency) | < 500ms |
| API availability | > 99.9% |
| Payment webhook processing | < 5s from Stripe to confirmation |

## Options Considered

### Option 1: Grafana + Prometheus + Loki + Tempo

Open-source observability stack: Prometheus for metrics, Loki for logs, Tempo for distributed traces, Grafana for visualization.

| Pros | Cons |
|------|------|
| Open source — no per-host/per-GB licensing costs | Self-hosted — requires deployment, configuration, and maintenance of 4 components |
| Unified dashboard in Grafana — metrics, logs, and traces in one place | Initial setup is more complex than SaaS solutions |
| Prometheus is the industry standard for metrics — mature ecosystem, extensive exporters | Storage management for Prometheus TSDB and Loki chunks |
| Loki's log-label model is efficient — indexes labels, not full text (cheaper storage) | Less polished UX compared to Datadog's managed dashboards |
| Tempo provides trace-to-log correlation — critical for debugging multi-module flows | Requires configuring OpenTelemetry instrumentation in the application |
| Team has prior experience with Prometheus | Alert configuration requires manual Alertmanager setup |

### Option 2: Datadog

Fully managed SaaS observability platform with APM, logs, metrics, and traces.

| Pros | Cons |
|------|------|
| Zero infrastructure to manage — SaaS with auto-instrumentation agents | Per-host pricing: ~$15-23/host/month for infrastructure + $31/host/month for APM |
| Polished dashboards, anomaly detection, and built-in alerting | Cost scales with infrastructure — 5 ECS tasks + database monitoring = ~$200-400/month |
| One-click integrations for AWS, PostgreSQL, Redis, RabbitMQ | Log ingestion pricing — $0.10/GB ingested, $1.70/million log events indexed |
| Distributed tracing with automatic service map | Vendor lock-in — Datadog's query language and dashboard format are proprietary |
| Excellent documentation and support | Unpredictable costs during traffic spikes (flash sales increase log and metric volume) |

### Option 3: ELK Stack (Elasticsearch + Logstash + Kibana)

Open-source log management with search capabilities, often extended with APM.

| Pros | Cons |
|------|------|
| Powerful full-text search across logs — Elasticsearch's query DSL is highly capable | Resource-intensive — Elasticsearch requires significant memory and storage (minimum 3 nodes for production) |
| Kibana provides rich log visualization and dashboard capabilities | No native metrics collection — requires adding Metricbeat or Prometheus separately |
| Open source (with caveats — Elastic License 2.0 for newer versions) | Operational complexity — managing Elasticsearch clusters, shard balancing, and index lifecycle |
| Mature ecosystem with extensive community support | Distributed tracing requires Elastic APM — additional component to manage |
| Can serve as both log aggregation and search engine | Cost of running Elasticsearch cluster on AWS: ~$200-500/month for a production-grade setup |

## Decision

**Grafana + Prometheus + Loki + Tempo.**

The open-source Grafana stack provides all three observability pillars without per-host or per-GB pricing — critical for a startup where traffic can spike 10x during flash sales (and so would Datadog costs). The team's existing Prometheus experience reduces the learning curve.

### Deployment on AWS

- **Prometheus:** Runs as a sidecar or separate ECS task, scraping metrics from the EventPass application's `/metrics` endpoint. Uses EC2-attached EBS for TSDB storage.
- **Loki:** Deployed on ECS with S3 backend for log chunk storage (cost-effective for variable log volume).
- **Tempo:** Deployed on ECS with S3 backend for trace storage. Receives traces via OpenTelemetry collector.
- **Grafana:** Single ECS task with persistent storage for dashboards. Connects to Prometheus, Loki, and Tempo as data sources.

### Key Dashboards

| Dashboard | Metrics |
|-----------|---------|
| **SLO Overview** | p99 latency for search and checkout, API availability %, error rate |
| **Flash Sale Monitor** | Concurrent users, ticket inventory depletion rate, reservation count, payment queue depth |
| **Payment Health** | Stripe webhook latency, payment success/failure ratio, refund completion rate, payout status |
| **Infrastructure** | ECS task CPU/memory, PostgreSQL connections/query latency, Redis hit rate/memory, RabbitMQ queue depth |

### Alerting

- **SLO breach:** p99 latency > 200ms (search) or > 500ms (checkout) for 5 minutes → PagerDuty/Slack alert.
- **Error rate spike:** > 1% 5xx errors for 2 minutes → immediate alert.
- **Queue backlog:** RabbitMQ dead-letter queue depth > 0 → warning alert.
- **Infrastructure:** ECS task memory > 85%, PostgreSQL connections > 80% of pool → warning.

## Consequences

**Enables:**
- Predictable costs — no per-host or per-GB pricing surprises during flash sales when log volume spikes.
- Full observability coverage — metrics (Prometheus), logs (Loki), traces (Tempo) with correlation in Grafana.
- SLO monitoring — dashboards and alerts tied to specific business metrics (checkout latency, payment success rate).
- Trace-to-log correlation — when a checkout is slow, trace the request through modules and jump to relevant logs.

**Prevents:**
- Instant setup — unlike Datadog's auto-instrumentation, the Grafana stack requires manual OpenTelemetry configuration and dashboard creation.
- Managed alerting features — no built-in anomaly detection (Datadog's AI-powered alerts).

**Risks and Mitigations:**
- **Risk:** Operational overhead of maintaining 4 observability components. **Mitigation:** Use Amazon Managed Service for Prometheus (AMP) and Amazon Managed Grafana (AMG) to reduce self-management. Loki and Tempo on ECS with S3 backends are lightweight.
- **Risk:** Dashboard and alert configuration is time-consuming. **Mitigation:** Use community Grafana dashboard templates for PostgreSQL, Redis, and Node.js; create EventPass-specific dashboards incrementally starting with the SLO overview.
- **Risk:** Storage costs for Prometheus metrics and Loki logs grow over time. **Mitigation:** Configure retention policies (Prometheus: 15 days, Loki: 30 days, Tempo: 7 days); use S3 lifecycle policies for automatic archival.

## Related ADRs

- [ADR-009: Deployment & CI/CD](ADR-009-deployment-cicd.md) — Observability stack deployed alongside the application on ECS
