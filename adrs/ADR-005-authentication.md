# ADR-005: Authentication

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass requires authentication and authorization for three user roles: Buyer (browse events, purchase tickets, view order history), Organizer (create and manage events, view sales reports, receive payouts), and Admin (moderate platform, manage disputes, approve organizers). The system must support secure registration, login, session management, and role-based access control (RBAC) across all API endpoints.

The Identity & Access bounded context needs to integrate with an authentication provider that handles password hashing, token issuance, brute force protection, and potentially social login (Google, Facebook) for buyers. The team of 3-5 developers should not spend engineering time building and maintaining authentication infrastructure — this is a solved problem with high security stakes.

## Options Considered

### Option 1: Auth0 (Managed Identity Provider)

Cloud-hosted identity platform with SDKs, pre-built login flows, and RBAC support.

| Pros | Cons |
|------|------|
| Zero auth infrastructure to maintain — Auth0 handles password storage, hashing, brute force protection, MFA | Vendor dependency — Auth0 outage blocks all logins for EventPass |
| Free tier covers up to 25K MAU — sufficient for MVP and early growth | Cost scales with users — Enterprise tier starts at $240/month once free tier is exceeded |
| Built-in social login (Google, Facebook, Apple) — reduces buyer registration friction | Customization limits — advanced flows require Auth0 Actions (serverless functions with cold starts) |
| RBAC via custom JWT claims — roles (BUYER, ORGANIZER, ADMIN) embedded in the access token | Latency for token validation depends on Auth0's JWKS endpoint (~50-100ms, cached locally) |
| SOC 2, HIPAA, GDPR compliant — critical if EventPass handles financial data | Learning curve for Auth0's dashboard and rule/action system |

### Option 2: Keycloak (Self-Hosted)

Open-source identity and access management server with RBAC, social login, and federation support.

| Pros | Cons |
|------|------|
| Full control over authentication infrastructure — no vendor dependency | Requires hosting, patching, scaling, and monitoring a Java-based server |
| Open source — no per-user pricing, unlimited MAU | Significant operational burden for a 3-5 person team — Keycloak upgrades are complex |
| Rich customization — themes, custom authenticators, user federation | Memory-hungry — Keycloak (Java/Quarkus) requires ~1-2 GB RAM minimum |
| On-premise data residency — no user data leaves your infrastructure | No managed service on AWS — must run on ECS/EC2 with manual HA configuration |
| Mature ecosystem with extensive documentation | Initial setup and configuration is time-consuming compared to Auth0's SaaS model |

### Option 3: Custom JWT Implementation (bcrypt + jsonwebtoken)

Build authentication from scratch using bcrypt for password hashing and jsonwebtoken for token issuance.

| Pros | Cons |
|------|------|
| Maximum flexibility — complete control over every authentication flow | Security risk — auth is a critical system where vulnerabilities have severe consequences |
| No external dependency — auth works as long as the application is running | Must implement: password hashing, token rotation, brute force protection, MFA, session management |
| Lowest latency — no external service calls for token validation | No social login without additional OAuth2 client implementation per provider |
| No vendor lock-in or pricing concerns | Ongoing maintenance burden — security patches, algorithm updates, compliance requirements |
| Simplest for the most basic use cases | OWASP compliance responsibility falls entirely on the team |

## Decision

**Auth0** (managed identity provider).

Authentication is a Generic Subdomain for EventPass — it is critical but not unique to our business. Building or self-hosting auth infrastructure would divert engineering effort from the core domain (ticketing, event management) without providing competitive advantage.

### Configuration for EventPass

- **Tenant:** `eventpass` on Auth0
- **Applications:** Single SPA application (Next.js frontend) + Machine-to-Machine application (backend API)
- **Roles:** `BUYER`, `ORGANIZER`, `ADMIN` — defined in Auth0 RBAC, included in JWT access token via custom claims
- **Social Connections:** Google and Facebook enabled for buyer registration
- **Database Connection:** Auth0's managed database for email/password authentication
- **MFA:** Optional for buyers, required for organizers and admins
- **Token Configuration:** Access token (JWT, 1h TTL), Refresh token (rotating, 7d TTL)
- **Brute Force Protection:** Enabled — blocks IP after 10 failed attempts
- **Custom Action:** Post-login action adds `role` and `userId` to access token claims

### Integration Pattern

The Identity module's Auth0 adapter handles all Auth0 API interactions. Other modules never call Auth0 directly — they consume the `IdentityFacade` which validates tokens and provides user context. This ACL pattern means replacing Auth0 with another provider requires changes only in the Identity module's infrastructure layer.

## Consequences

**Enables:**
- Immediate security compliance — Auth0 handles OWASP authentication best practices, brute force protection, and credential storage.
- Social login for buyers — reduces registration friction, potentially increasing conversion rates.
- Fast implementation — Auth0's SDKs and pre-built Universal Login page mean authentication is functional within hours, not weeks.
- MFA for organizers and admins — protects accounts that manage events and financial data.

**Prevents:**
- Complete control over authentication flows — advanced customization requires Auth0 Actions.
- On-premise user data — authentication data lives on Auth0's infrastructure (acceptable for EventPass's use case).

**Risks and Mitigations:**
- **Risk:** Auth0 outage blocks all logins. **Mitigation:** Cache JWKS public keys locally with 24h TTL — existing sessions continue to work during Auth0 downtime; new logins are the only operations affected. Auth0's SLA is 99.99% uptime.
- **Risk:** Free tier exceeded as user base grows beyond 25K MAU. **Mitigation:** Auth0's B2C pricing is ~$0.015/MAU — at 50K users, cost is ~$750/month, which is justified by the engineering time saved. Plan migration to Keycloak only if costs become unsustainable (>$2K/month).
- **Risk:** Vendor lock-in — Auth0 token format and APIs become embedded in the codebase. **Mitigation:** Auth0 integration is isolated in the Identity module's `infrastructure/auth0.adapter.ts`. The facade exposes standard interfaces (validateToken, getUserById) that do not expose Auth0-specific concepts.

## Related ADRs

- [ADR-001: Deployment Model](ADR-001-deployment-model.md) — Monolith simplifies auth integration (single application to configure)
- [ADR-007: API Design](ADR-007-api-design.md) — REST API endpoints use Auth0 JWT tokens for authentication
