# Architecture Decision Records

ADRs documenting all key architectural decisions for EventPass. Each ADR follows a standard format: Context, Options Considered (with honest pros/cons), Decision, Consequences (including risks and mitigations), and Related ADRs.

All decisions are specific to EventPass's constraints: team of 3-5 developers, startup budget, MVP-stage product, ~5K concurrent users during flash sales, and a ticketing domain with 7 bounded contexts.

## Summary

| ADR | Title | Status | Decision |
|-----|-------|--------|----------|
| [ADR-001](ADR-001-deployment-model.md) | Deployment Model | Accepted | Modular Monolith |
| [ADR-002](ADR-002-communication-style.md) | Communication Style | Accepted | Hybrid — sync facade calls + async events |
| [ADR-003](ADR-003-database-strategy.md) | Database Strategy | Accepted | Schema-per-module on single PostgreSQL |
| [ADR-004](ADR-004-event-bus.md) | Event Bus | Accepted | RabbitMQ (Amazon MQ) |
| [ADR-005](ADR-005-authentication.md) | Authentication | Accepted | Auth0 (managed identity provider) |
| [ADR-006](ADR-006-observability.md) | Observability | Accepted | Grafana + Prometheus + Loki + Tempo |
| [ADR-007](ADR-007-api-design.md) | API Design | Accepted | REST for all external APIs |
| [ADR-008](ADR-008-caching-strategy.md) | Caching Strategy | Accepted | Redis (ElastiCache) with TTL-based invalidation |
| [ADR-009](ADR-009-deployment-cicd.md) | Deployment & CI/CD | Accepted | Docker + AWS ECS Fargate + GitHub Actions |
| [ADR-010](ADR-010-frontend-architecture.md) | Frontend Architecture | Accepted | Next.js App Router (SSR + Client Components) |

## Decision Dependencies

```
ADR-001 (Modular Monolith)
├── ADR-002 (Hybrid communication — in-process facades + events)
│   ├── ADR-004 (RabbitMQ for durable async events)
│   └── ADR-007 (REST for external API only — no inter-service network)
├── ADR-003 (Schema-per-module on shared PostgreSQL)
│   └── ADR-008 (Redis caching complements database strategy)
├── ADR-005 (Auth0 — managed auth for small team)
├── ADR-006 (Grafana stack — open-source, predictable cost)
├── ADR-009 (ECS Fargate — single container deployment)
└── ADR-010 (Next.js — SSR for SEO + client interactivity)
```
