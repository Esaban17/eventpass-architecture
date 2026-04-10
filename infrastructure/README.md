# Infrastructure Components

Documentation for each infrastructure component used by EventPass.

> Documents will be added in Phase 4.

| # | Component | Technology Chosen | Document |
|---|-----------|------------------|----------|
| 1 | API Gateway | Kong Gateway (OSS) | `01-api-gateway.md` |
| 2 | CDN | Cloudflare | `02-cdn.md` |
| 3 | Load Balancer | AWS ALB | `03-load-balancer.md` |
| 4 | Cache | Redis (ElastiCache) | `04-cache.md` |
| 5 | Message Queue | RabbitMQ (Amazon MQ) | `05-message-queue.md` |
| 6 | Primary Database | PostgreSQL (RDS) | `06-primary-database.md` |
| 7 | Object Storage | AWS S3 | `07-object-storage.md` |
| 8 | Auth Provider | Auth0 | `08-auth-provider.md` |
| 9 | Payment Provider | Stripe | `09-payment-provider.md` |
| 10 | Notification System | SendGrid + Twilio | `10-notification-system.md` |
| 11 | Observability Stack | Grafana + Prometheus + Loki + Tempo | `11-observability-stack.md` |
| 12 | CI/CD Pipeline | GitHub Actions + Docker + ECR + ECS | `12-cicd-pipeline.md` |
