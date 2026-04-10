# CDN — Cloudflare

## 1. Responsibilities

- **Static asset delivery** — Serve Next.js static files (JavaScript bundles, CSS, fonts, images) from edge locations closest to users, reducing latency from ~200ms (origin) to ~20ms (edge).
- **Event image caching** — Cache organizer-uploaded event poster images and venue photos at the edge with 24h TTL, reducing S3 egress costs and origin load.
- **DDoS protection during flash sales** — Absorb volumetric attacks and bot traffic before they reach AWS infrastructure. Critical when a popular event goes on sale and attracts both legitimate buyers and ticket scalper bots.
- **SSL/TLS termination** — Manage the `eventpass.com` SSL certificate and encrypt all traffic between users and the edge. Origin traffic uses Cloudflare's full (strict) SSL mode.
- **Edge caching for public API responses** — Cache GET requests to `/api/catalog/events` and `/api/catalog/search` at the edge with short TTLs (30-60s), reducing backend load for the highest-traffic read endpoints.

## 2. Why This Project Needs It

Event ticketing platforms have bursty, predictable traffic patterns: a popular concert announcement can drive 10x normal traffic within minutes. Without a CDN, every request hits the origin (ECS Fargate in a single AWS region), creating a bottleneck that degrades the experience for all users. Cloudflare's edge network (300+ cities) absorbs the read-heavy traffic (event browsing, image loading) while only purchase-related write requests reach the origin.

Additionally, ticket scalper bots are a real threat in the ticketing industry. Cloudflare's bot management and rate limiting at the edge prevent automated purchasing scripts from consuming inventory before human buyers.

## 3. Technology Choice

| Dimension | Cloudflare | AWS CloudFront | Fastly |
|-----------|------------|----------------|--------|
| **Managed/Self-hosted** | Fully managed SaaS | Fully managed (AWS) | Fully managed SaaS |
| **Operational complexity** | Low — DNS change + page rules | Low — integrated with S3/ALB via AWS console | Low — VCL config for custom logic |
| **Cost at our scale** | Free plan for basic CDN; Pro at $20/month for WAF + bot management | ~$0.085/GB for first 10TB + $0.0075/10K requests | ~$0.12/GB + $0.009/10K requests |
| **Key differentiating feature** | Integrated DDoS + bot management + WAF at no extra cost (Pro plan) | Native AWS integration (S3 origin, Lambda@Edge) | Real-time log streaming, VCL-based edge compute |

**Decision: Cloudflare** — The integrated DDoS protection and bot management are critical for a ticketing platform where flash sales attract both legitimate users and scalper bots. Cloudflare's Pro plan ($20/month) includes WAF, bot score, and rate limiting — capabilities that would require separate AWS services (WAF at $5/month + $1/million requests, Shield Advanced at $3K/month) on CloudFront.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Integrated DDoS + WAF + bot management in a single product | DNS must be managed through Cloudflare (proxied mode) — adds a dependency layer |
| Free plan covers basic CDN needs; Pro ($20/month) adds WAF and bot scoring | Less control over cache behavior compared to Fastly's VCL — Cloudflare uses page rules and cache rules |
| 300+ edge locations globally — excellent coverage for Latin American users (Guatemala, Mexico, Colombia) | Origin-pull model means first request after cache expiry hits the origin — no push-based invalidation |
| Workers (edge compute) available if EventPass needs edge-side logic in the future | Cloudflare sits in front of all traffic — a Cloudflare outage affects the entire platform |
| Always-on DDoS protection — no additional configuration needed | Cache purge propagation takes 2-5 seconds across all edge locations |

## 5. Integration

- **Upstream:** Users' browsers connect to Cloudflare edge locations via HTTPS (`eventpass.com`). Cloudflare terminates SSL and applies WAF/bot rules.
- **Downstream:** Cloudflare proxies dynamic requests (API calls, SSR pages) to the AWS ALB (see [03-load-balancer.md](03-load-balancer.md)). Static assets are served from edge cache or fetched from the S3 origin (see [07-object-storage.md](07-object-storage.md)).
- **Cache configuration:** Page rules define caching behavior per path: `/api/catalog/*` cached 30s at edge; `/api/orders/*` and `/api/payments/*` bypassed (never cached); static assets (`/_next/static/*`) cached 30 days with immutable headers.
- **Bot management:** Cloudflare's bot score identifies automated traffic. Requests with bot score < 30 are challenged with a CAPTCHA before reaching the `/tickets/reserve` endpoint during flash sales.
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — Cloudflare sits in the Public Internet zone, upstream of all AWS infrastructure.
