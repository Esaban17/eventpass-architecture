# Infrastructure Components

Documentation for each infrastructure component used by EventPass. Every document follows a standard 5-section structure: Responsibilities, Why This Project Needs It, Technology Choice (comparison table), Trade-offs (advantages vs. disadvantages), and Integration.

All technology choices are justified against EventPass's specific constraints: team of 3-5 developers, startup budget (~$375/month baseline), ~5K concurrent users during flash sales, and a ticketing domain requiring strong consistency for inventory management.

## Components

| # | Component | Technology | Description |
|---|-----------|-----------|-------------|
| 1 | [API Gateway](01-api-gateway.md) | Kong Gateway (OSS) | Rate limiting for flash sales, JWT verification, CORS, request routing |
| 2 | [CDN](02-cdn.md) | Cloudflare | Static asset delivery, DDoS protection, bot management, edge caching |
| 3 | [Load Balancer](03-load-balancer.md) | AWS ALB | Traffic distribution across ECS tasks, health checks, path-based routing |
| 4 | [Cache](04-cache.md) | Redis (ElastiCache) | Ticket inventory caching, sessions, search results, rate limiter backing store |
| 5 | [Message Queue](05-message-queue.md) | RabbitMQ (Amazon MQ) | Async event processing, QR generation, notifications, batch refunds |
| 6 | [Primary Database](06-primary-database.md) | PostgreSQL (RDS) | Schema-per-module storage, ACID transactions, SKIP LOCKED for concurrency |
| 7 | [Object Storage](07-object-storage.md) | AWS S3 | QR codes, event images, frontend assets, database backups, log archival |
| 8 | [Auth Provider](08-auth-provider.md) | Auth0 | Registration, login, JWT issuance, RBAC, social login, MFA |
| 9 | [Payment Provider](09-payment-provider.md) | Stripe | Payment intents, webhooks, refunds, Connect payouts, PCI compliance |
| 10 | [Notification System](10-notification-system.md) | SendGrid + Twilio | Transactional emails, SMS alerts, dynamic templates, delivery tracking |
| 11 | [Observability Stack](11-observability-stack.md) | Grafana + Prometheus + Loki + Tempo | Metrics, logs, traces, SLO dashboards, alerting |
| 12 | [CI/CD Pipeline](12-cicd-pipeline.md) | GitHub Actions + Docker + ECR + ECS | Automated build, test, deploy with blue-green strategy and auto-rollback |
| 13 | [Background Workers](13-background-workers.md) | BullMQ (Node.js + Redis) | Async task processing: QR generation, emails, payment retry, catalog sync, refund batches |

## Cost Summary (Normal Operation)

| Component | Monthly Cost |
|-----------|-------------|
| ECS Fargate (API + Frontend) | ~$40 |
| ECS Fargate (Workers, 1 task) | ~$20 |
| Kong Gateway (ECS task) | ~$15 |
| RDS PostgreSQL (Multi-AZ) | ~$200 |
| ElastiCache Redis | ~$25 |
| Amazon MQ RabbitMQ | ~$90 |
| S3 + CloudWatch | ~$20 |
| Cloudflare Pro | ~$20 |
| Auth0 | Free (up to 25K MAU) |
| SendGrid | Free (up to 100 emails/day) |
| **Total** | **~$430/month** |
