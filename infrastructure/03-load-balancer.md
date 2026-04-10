# Load Balancer — AWS Application Load Balancer (ALB)

## 1. Responsibilities

- **Traffic distribution** — Distribute incoming requests across 2-8 ECS Fargate tasks running the EventPass Modular Monolith, using round-robin with slow-start to avoid overwhelming newly launched tasks during scale-out events.
- **Health checking** — Poll each ECS task's `/health` endpoint every 30 seconds. Unhealthy tasks (3 consecutive failures) are removed from the target group, and ECS replaces them automatically.
- **Path-based routing** — Route `/api/*` requests to the backend API target group and `/*` requests to the Next.js frontend target group, enabling independent scaling of frontend and backend.
- **TLS termination** — Terminate HTTPS connections from Cloudflare using AWS Certificate Manager (ACM) certificates, passing plain HTTP to ECS tasks within the VPC.
- **Connection draining** — When a task is being replaced (deployment or scaling), the ALB drains existing connections over 30 seconds before deregistering the task, preventing in-flight requests from failing.

## 2. Why This Project Needs It

EventPass runs multiple ECS Fargate tasks to handle concurrent traffic. During flash sales, the system scales from 2 to 8 tasks within minutes. The ALB ensures new tasks receive traffic only after passing health checks, preventing buyers from hitting a container that is still initializing its database connections or loading configuration. Without a load balancer, scaling would require manual DNS updates or client-side logic — unacceptable for a real-time purchasing flow.

The path-based routing is critical for EventPass's architecture: the Next.js frontend (SSR) and the Node.js backend API are separate ECS services that scale independently. The ALB routes requests to the correct service without requiring separate domains or ports.

## 3. Technology Choice

| Dimension | AWS ALB | Nginx (self-managed) | HAProxy |
|-----------|---------|---------------------|---------|
| **Managed/Self-hosted** | Fully managed (AWS) | Self-hosted on ECS/EC2 | Self-hosted on ECS/EC2 |
| **Operational complexity** | Low — AWS manages patching, scaling, HA across AZs | High — must manage instances, config, SSL certs, scaling | High — must manage instances, config, health checks |
| **Cost at our scale** | ~$22/month (fixed) + $0.008/LCU-hour | ~$15/month (ECS task) + operational time | ~$15/month (ECS task) + operational time |
| **Key differentiating feature** | Native ECS integration (automatic target registration), WAF support, access logs to S3 | Maximum configurability, Lua scripting, caching | Best-in-class performance for raw TCP/HTTP, advanced health checks |

**Decision: AWS ALB** — Native integration with ECS Fargate is the decisive factor. When ECS launches a new task, the ALB automatically registers it in the target group after it passes health checks. With Nginx or HAProxy, this registration would require additional tooling (service discovery, consul-template, or custom scripts).

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Zero operational management — AWS handles HA, patching, and cross-AZ distribution | Fixed cost (~$22/month) even during zero-traffic periods |
| Native ECS integration — tasks auto-register/deregister on scale events | Less configuration flexibility than Nginx — no custom Lua logic or advanced caching |
| Built-in access logs to S3 — useful for debugging and audit trails | AWS-specific — cannot be used outside AWS ecosystem |
| WebSocket support — ready for future real-time features (live ticket availability) | No built-in rate limiting — must use Kong (see [01-api-gateway.md](01-api-gateway.md)) or AWS WAF |
| Blue-green deployment support via target group switching | Additional LCU charges during traffic spikes (though minimal at our scale) |

## 5. Integration

- **Upstream:** Receives traffic from Cloudflare CDN (see [02-cdn.md](02-cdn.md)) or directly from Kong API Gateway (see [01-api-gateway.md](01-api-gateway.md)) depending on the routing configuration.
- **Downstream:** Routes to two ECS target groups — the EventPass API (backend monolith) and the Next.js frontend, both in the Private/Public VPC subnet.
- **ECS integration:** Uses ECS service discovery to automatically register new Fargate tasks as they start and deregister tasks being terminated during deployments or scale-in.
- **SSL/TLS:** ACM certificate for `*.eventpass.com` attached to the ALB HTTPS listener (port 443). Internal traffic to ECS tasks uses HTTP (port 3000) within the VPC.
- **Health checks:** `GET /health` on port 3000, healthy threshold: 2, unhealthy threshold: 3, interval: 30s, timeout: 5s.
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — ALB sits in the Public VPC subnet between Cloudflare and ECS tasks.
