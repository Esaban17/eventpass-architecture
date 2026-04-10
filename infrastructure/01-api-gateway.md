# API Gateway — Kong Gateway (OSS)

## 1. Responsibilities

- **Rate limiting on purchase endpoints** — Throttle `/events/:id/tickets/reserve` and `/orders` during flash sales to prevent inventory hammering (~5K concurrent users, limited to 100 req/s per IP).
- **Request routing** — Route incoming HTTPS requests to the appropriate ECS Fargate tasks based on path prefix (`/api/*` to backend, `/*` to Next.js frontend).
- **JWT verification** — Validate Auth0-issued JWT tokens on every authenticated request before traffic reaches the application, rejecting expired or malformed tokens at the gateway level.
- **CORS enforcement** — Allow requests only from `eventpass.com` and `*.eventpass.com` origins, blocking unauthorized cross-origin API access.
- **Request/response transformation** — Add request IDs for tracing, enforce content-type headers, strip internal headers from responses.

## 2. Why This Project Needs It

EventPass's flash sale scenario creates a unique traffic pattern: thousands of buyers simultaneously hitting ticket reservation endpoints. Without gateway-level rate limiting, a single overenthusiastic buyer (or bot) could exhaust the Ticketing module's capacity before legitimate users complete their purchases. The API gateway acts as the first line of defense, enforcing per-IP rate limits before requests reach the application's business logic.

Additionally, JWT verification at the gateway means invalid tokens never consume application resources — critical during high-traffic events where every millisecond of backend compute matters.

## 3. Technology Choice

| Dimension | Kong Gateway (OSS) | AWS API Gateway | Traefik |
|-----------|-------------------|-----------------|---------|
| **Managed/Self-hosted** | Self-hosted on ECS (Docker) | Fully managed (AWS) | Self-hosted on ECS (Docker) |
| **Operational complexity** | Moderate — requires ECS task + PostgreSQL for config storage | Low — zero infrastructure management | Low — single binary, file-based config |
| **Cost at our scale** | ~$15/month (ECS task) | ~$3.50/million requests + $1/million WebSocket messages | ~$15/month (ECS task) |
| **Key differentiating feature** | Plugin ecosystem (rate limiting, JWT, CORS, logging, custom Lua plugins) | Native AWS integration (IAM, CloudWatch, Lambda authorizers) | Automatic service discovery with Docker/ECS labels |

**Decision: Kong Gateway (OSS)** — The plugin ecosystem provides the specific capabilities EventPass needs (rate limiting with Redis backend, JWT validation, CORS) without vendor lock-in to AWS. Kong's rate limiting plugin supports Redis-backed distributed counters, ensuring consistent limits across multiple gateway instances during flash sales.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Plugin-based architecture — enable only what you need, disable the rest | Requires managing an additional ECS task and PostgreSQL/Cassandra for configuration storage |
| Redis-backed rate limiting — distributed counters accurate across multiple Kong instances | Higher operational complexity than AWS API Gateway's zero-management model |
| Open source — no per-request pricing, predictable cost regardless of traffic spikes | Kong's declarative config (DB-less mode) is simpler but loses dynamic plugin configuration |
| Extensible with custom Lua plugins if EventPass needs specialized behavior | Learning curve for Kong's admin API and plugin configuration system |
| Can run in DB-less mode with YAML config for simpler operations | No built-in WebSocket support — would need separate configuration if real-time features are added |

## 5. Integration

- **Upstream:** Receives traffic from Cloudflare CDN (see [02-cdn.md](02-cdn.md)) after SSL termination and DDoS filtering.
- **Downstream:** Routes to AWS ALB (see [03-load-balancer.md](03-load-balancer.md)) which distributes to ECS Fargate tasks.
- **Auth0 integration:** Uses the JWT plugin to validate tokens against Auth0's JWKS endpoint. Public keys are cached locally with a 24h TTL to reduce dependency on Auth0 availability.
- **Redis integration:** Rate limiting plugin connects to the same ElastiCache Redis instance (see [04-cache.md](04-cache.md)) used by the application, using a dedicated key prefix (`ratelimit:*`).
- **Observability:** Kong's logging plugin forwards access logs to Loki and request metrics to Prometheus (see [11-observability-stack.md](11-observability-stack.md)).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — Kong runs in the Public VPC subnet alongside the ALB.
