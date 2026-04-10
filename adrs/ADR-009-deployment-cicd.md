# ADR-009: Deployment & CI/CD

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass is a Modular Monolith (ADR-001) deployed as a single Docker container. The deployment pipeline must handle:

- **Build and test:** TypeScript compilation, linting (including module boundary enforcement), unit tests, and integration tests.
- **Container management:** Build Docker images, push to a registry, deploy to a container orchestration platform.
- **Environment management:** Staging environment for pre-production validation, production environment with zero-downtime deployments.
- **Rollback capability:** Quick rollback to the previous version if a deployment causes issues.
- **Cost efficiency:** Startup budget — avoid paying for unused compute capacity during low-traffic periods.

The team of 3-5 developers uses GitHub for source control, making GitHub-native CI/CD integration a natural fit. The deployment target must support horizontal scaling (3-5 container instances during flash sales, 1-2 during normal operation).

## Options Considered

### Option 1: Docker + AWS ECS Fargate + GitHub Actions

Serverless container orchestration with GitHub-native CI/CD.

| Pros | Cons |
|------|------|
| Fargate eliminates server management — no EC2 instances to patch, scale, or monitor | Fargate pricing per vCPU/memory-hour is higher than EC2 (~30% premium for the managed service) |
| GitHub Actions integrates natively with the repository — triggers on push, PR, and manual dispatch | Cold start latency — new Fargate tasks take 30-60s to start (mitigated by keeping minimum tasks running) |
| Auto-scaling based on CPU/memory or custom CloudWatch metrics | Less control over the underlying infrastructure than EC2 or Kubernetes |
| Blue-green deployment via ECS service update — zero-downtime deployments with automatic rollback | GitHub Actions free tier has limited minutes (2,000/month for private repos) |
| Pay-per-use — scale to 0 tasks during development, 1-2 in normal operation, 5+ during flash sales | ECS-specific configuration — task definitions, service configurations, target group management |

### Option 2: Docker + Kubernetes (Amazon EKS)

Container orchestration with full control via Kubernetes API.

| Pros | Cons |
|------|------|
| Industry-standard orchestration — extensive ecosystem of tools, operators, and community support | EKS control plane cost: $73/month + node costs (~$100+/month minimum) |
| Fine-grained control over scheduling, networking, and resource allocation | Significant operational complexity — cluster upgrades, RBAC, network policies, Helm charts |
| HPA (Horizontal Pod Autoscaler) for sophisticated scaling policies | Overkill for a single application — Kubernetes shines with 10+ services, not 1 monolith |
| Built-in secret management, config maps, and service discovery | Team of 3-5 needs Kubernetes expertise — steep learning curve for day-to-day operations |
| Portable across cloud providers | Debugging Kubernetes issues (pod scheduling, networking, RBAC) requires specialized knowledge |

### Option 3: Serverless (AWS Lambda)

Event-driven compute with zero infrastructure management.

| Pros | Cons |
|------|------|
| Zero infrastructure — no containers, no orchestration, no scaling configuration | Cold start latency (100-500ms) is problematic for user-facing API responses |
| Pay-per-invocation — extremely cost-effective at low traffic | Not suitable for a monolith — Lambda's 250MB deployment package and 15-minute execution limits constrain application architecture |
| Auto-scaling to any load without configuration | Persistent connections (PostgreSQL, Redis) require Lambda-specific solutions (RDS Proxy, connection pooling) |
| Built-in integration with API Gateway for REST endpoints | Requires restructuring the monolith into Lambda-compatible handlers — significant refactoring |
| AWS manages patching, scaling, and availability | WebSocket connections (for real-time features) require separate API Gateway WebSocket configuration |

## Decision

**Docker + AWS ECS Fargate + GitHub Actions.**

ECS Fargate is the natural fit for a Modular Monolith: one container image, serverless infrastructure management, and straightforward horizontal scaling. Kubernetes is powerful but adds operational complexity that a 3-5 person team cannot justify for a single application. Lambda's constraints (cold starts, connection management, package size limits) conflict with the monolith architecture.

### CI/CD Pipeline

```
┌─────────┐    ┌──────┐    ┌───────┐    ┌──────────┐    ┌──────────┐    ┌────────────┐    ┌────────────┐
│  Push   │ →  │ Lint │ →  │ Test  │ →  │  Build   │ →  │ Push to  │ →  │  Deploy    │ →  │  Deploy    │
│ to main │    │      │    │       │    │  Docker  │    │   ECR    │    │  Staging   │    │ Production │
└─────────┘    └──────┘    └───────┘    └──────────┘    └──────────┘    └────────────┘    └────────────┘
                  │             │                                            │                   │
                  ▼             ▼                                            ▼                   ▼
              ESLint +     Unit +                                       Smoke tests         Blue-green
              boundary   integration                                    validate            deployment
              checks     tests                                          health              with rollback
```

**Pipeline Stages:**

| Stage | Tool | Actions | Failure Behavior |
|-------|------|---------|-----------------|
| **Lint** | ESLint | TypeScript type checking, module boundary violations, code style | Fail fast — block pipeline |
| **Test** | Jest | Unit tests + integration tests (PostgreSQL test container) | Fail — block pipeline, report coverage |
| **Build** | Docker | Multi-stage build: `npm ci → npm run build → production image` | Fail — block pipeline |
| **Push** | AWS ECR | Tag image with commit SHA and `latest`, push to ECR registry | Fail — block pipeline |
| **Deploy Staging** | ECS | Update staging service with new task definition | Fail — alert team, stop pipeline |
| **Smoke Tests** | Jest/curl | Health check, key API endpoints, basic purchase flow | Fail — rollback staging, alert team |
| **Deploy Production** | ECS | Blue-green deployment — new tasks start, old tasks drain | Fail — automatic rollback to previous task definition |

### Infrastructure Configuration

```yaml
# ECS Service Configuration
service:
  name: eventpass-api
  launch_type: FARGATE
  desired_count: 2  # Normal operation
  
task_definition:
  cpu: 512          # 0.5 vCPU
  memory: 1024      # 1 GB
  container:
    image: ${ECR_REPO}/eventpass:${COMMIT_SHA}
    port: 3000
    health_check:
      path: /health
      interval: 30s
      timeout: 5s
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://...
      - REDIS_URL=redis://...
      - RABBITMQ_URL=amqp://...
      
auto_scaling:
  min: 1
  max: 8
  target_cpu: 70%
  scale_in_cooldown: 300s
  scale_out_cooldown: 60s
```

### Rollback Strategy

- **Automatic:** ECS monitors health checks during blue-green deployment. If new tasks fail health checks within 5 minutes, ECS automatically rolls back to the previous task definition.
- **Manual:** `aws ecs update-service --force-new-deployment` with the previous task definition revision.
- **Database migrations:** Forward-only migrations with backward-compatible schema changes. If a migration causes issues, deploy a new forward migration to fix it — never roll back migrations.

## Consequences

**Enables:**
- Zero-downtime deployments via ECS blue-green strategy — buyers are never interrupted during flash sales.
- Cost-efficient scaling — 1 task during quiet hours ($15/month), 8 tasks during flash sales (~$120/month for the duration).
- GitHub-native workflow — developers push code, GitHub Actions handles the rest, ECS deploys automatically.
- Simple rollback — one CLI command or automatic rollback via health check failures.

**Prevents:**
- Fine-grained deployment control — cannot deploy a single module independently (consequence of the monolith architecture).
- Multi-cloud portability — ECS is AWS-specific. Migration to GCP or Azure would require re-platforming to Cloud Run or ACI.

**Risks and Mitigations:**
- **Risk:** GitHub Actions free tier limit (2,000 min/month) exceeded with frequent pushes. **Mitigation:** Optimize CI by caching node_modules and Docker layers; run lint and tests in parallel; consider GitHub Actions paid plan ($4/1,000 additional minutes) if needed.
- **Risk:** ECS Fargate cold start (30-60s) during rapid scale-out events. **Mitigation:** Keep minimum 2 tasks running during expected flash sale windows; pre-scale 30 minutes before announced events.
- **Risk:** Failed database migration blocks deployment. **Mitigation:** All migrations are backward-compatible and forward-only; test migrations against a staging database clone before production; use pg_dump snapshots for recovery.

## Related ADRs

- [ADR-001: Deployment Model](ADR-001-deployment-model.md) — Modular Monolith deployed as a single container
- [ADR-006: Observability](ADR-006-observability.md) — Monitoring stack deployed alongside the application on ECS
