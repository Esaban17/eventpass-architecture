# CI/CD Pipeline — GitHub Actions + Docker + AWS ECR + ECS

## 1. Responsibilities

- **Automated build pipeline** — On every push to `main`: lint (ESLint + module boundary checks), test (unit + integration with PostgreSQL test container), build Docker image, push to ECR.
- **Staging deployment** — After successful build, deploy to the staging ECS service for pre-production validation. Run smoke tests against staging endpoints.
- **Production deployment with blue-green** — After staging smoke tests pass, deploy to production ECS service using blue-green strategy: new tasks start alongside old tasks, traffic shifts after health checks pass, old tasks drain and terminate.
- **Automatic rollback** — If new production tasks fail health checks within 5 minutes, ECS automatically rolls back to the previous task definition. No manual intervention required.
- **Docker image management** — Multi-stage Dockerfile for optimized production images. Images tagged with commit SHA and `latest`, stored in ECR with lifecycle policy (keep last 20 images).

## 2. Why This Project Needs It

EventPass's Modular Monolith requires a reliable deployment pipeline that prevents broken code from reaching production — a deployment that breaks the checkout flow during a flash sale directly impacts revenue and user trust. The pipeline enforces quality gates: module boundary violations caught by ESLint, regressions caught by tests, and infrastructure issues caught by staging smoke tests.

Blue-green deployment is critical because EventPass cannot afford downtime: buyers may be mid-checkout when a deployment occurs. The blue-green strategy ensures the old version continues serving requests while the new version starts and passes health checks, then traffic shifts seamlessly.

## 3. Technology Choice

| Dimension | GitHub Actions + Docker + ECR + ECS | GitLab CI | CircleCI |
|-----------|-------------------------------------|-----------|----------|
| **Managed/Self-hosted** | Fully managed (GitHub + AWS) | Fully managed SaaS or self-hosted | Fully managed SaaS |
| **Operational complexity** | Low — YAML workflows in repo, native GitHub integration | Low-moderate — similar YAML, but separate platform from code hosting | Low — YAML config, Docker-native |
| **Cost at our scale** | Free: 2,000 min/month (private repos). Paid: $4/1K additional min | Free: 400 min/month. Paid: $19/user/month | Free: 6,000 credits/month. Paid: $15/month for additional |
| **Key differentiating feature** | Native GitHub integration (PR checks, deployment environments, GITHUB_TOKEN), marketplace actions | Built-in container registry, merge request pipelines, DevSecOps features | Docker layer caching, parallel workflows, SSH debugging |

**Decision: GitHub Actions + Docker + ECR + ECS** — EventPass's source code is hosted on GitHub, making GitHub Actions the natural CI/CD choice. The native integration provides PR status checks (blocking merges that fail lint/test), deployment environments with approval gates, and the `GITHUB_TOKEN` for authenticated API calls without managing separate credentials. GitLab CI would require migrating the repository or setting up mirroring. CircleCI is capable but adds another vendor relationship.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Native GitHub integration — PR checks, deployment status, commit status badges | Free tier limited to 2,000 min/month — may need paid plan with frequent pushes |
| YAML workflows version-controlled alongside code — pipeline changes go through the same review process | GitHub Actions marketplace actions are third-party — must audit for security |
| Deployment environments with protection rules — production deployments require manual approval or passing staging | Debugging failed workflows requires reading logs in GitHub UI — no SSH into runner |
| Docker layer caching reduces build times from ~5 min to ~2 min | Secrets management is GitHub-native — cannot use external secret managers without additional setup |
| ECS blue-green deployment provides zero-downtime releases with automatic rollback | Tightly coupled to GitHub — migrating to GitLab/Bitbucket requires rewriting all workflows |

## 5. Integration

- **Source control:** Workflows triggered on push to `main` (deploy) and pull request (lint + test only). Feature branches do not trigger deployments.
- **Pipeline stages:**

| Stage | Duration | Tools | Failure Action |
|-------|----------|-------|---------------|
| Lint | ~30s | ESLint (TypeScript + module boundaries) | Block pipeline |
| Test | ~2min | Jest + PostgreSQL test container (docker-compose) | Block pipeline, report coverage |
| Build | ~2min | Multi-stage Dockerfile (`npm ci → build → production`) | Block pipeline |
| Push | ~30s | `docker push` to ECR with commit SHA tag | Block pipeline |
| Deploy Staging | ~2min | `aws ecs update-service` for staging | Alert team, stop pipeline |
| Smoke Tests | ~1min | curl health check + key API endpoints on staging | Rollback staging, alert team |
| Deploy Production | ~3min | `aws ecs update-service` with blue-green | Automatic rollback on health check failure |

- **AWS credentials:** GitHub Actions uses OIDC federation with AWS IAM role — no long-lived access keys stored in GitHub Secrets. The IAM role has permissions limited to ECR push and ECS service update.
- **Docker configuration:**
  ```dockerfile
  # Multi-stage build
  FROM node:22-alpine AS builder
  WORKDIR /app
  COPY package*.json ./
  RUN npm ci
  COPY . .
  RUN npm run build

  FROM node:22-alpine AS production
  WORKDIR /app
  COPY --from=builder /app/dist ./dist
  COPY --from=builder /app/node_modules ./node_modules
  EXPOSE 3000
  CMD ["node", "dist/main.js"]
  ```
- **ECR lifecycle policy:** Retain the 20 most recent images; expire untagged images after 1 day.
- **Monitoring:** Pipeline execution time, success/failure rate, and deployment frequency tracked as custom metrics in Prometheus. Deployment events annotated on Grafana dashboards. See [11-observability-stack.md](11-observability-stack.md).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — GitHub Actions pushes to ECR, which ECS pulls from during deployment.
