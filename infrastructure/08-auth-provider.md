# Auth Provider — Auth0

## 1. Responsibilities

- **User registration and login** — Handle email/password registration, social login (Google, Facebook), and credential storage for all three user roles (Buyer, Organizer, Admin).
- **JWT token issuance** — Issue access tokens (1h TTL) and rotating refresh tokens (7d TTL) with custom claims containing `userId`, `role`, and `organizerStatus`.
- **Multi-factor authentication (MFA)** — Optional TOTP-based MFA for buyers, mandatory for organizers and admins who manage events and financial data.
- **Role-based access control (RBAC)** — Define and enforce three roles: `BUYER` (browse, purchase), `ORGANIZER` (create events, view sales), `ADMIN` (moderate, approve organizers). Roles propagated via JWT claims.
- **Brute force protection** — Block IPs after 10 failed login attempts, detect credential stuffing attacks, and enforce password complexity policies.

## 2. Why This Project Needs It

Authentication is the highest-stakes infrastructure component in EventPass: a vulnerability in auth exposes financial data (payment history, organizer payouts), personal information (email, phone, purchase history), and platform integrity (an attacker with admin access could cancel events or approve fraudulent organizers).

A team of 3-5 developers should not build and maintain authentication infrastructure — the risk of security vulnerabilities (improper password hashing, JWT validation bugs, session fixation) far outweighs the cost of a managed provider. Auth0 handles OWASP authentication best practices, compliance (SOC 2, GDPR), and continuously patches against emerging threats.

## 3. Technology Choice

| Dimension | Auth0 | Keycloak | AWS Cognito |
|-----------|-------|----------|-------------|
| **Managed/Self-hosted** | Fully managed SaaS | Self-hosted (Java/Quarkus on ECS) | Fully managed (AWS) |
| **Operational complexity** | Low — zero infrastructure, dashboard-based config | High — must manage JVM, database, upgrades, HA | Low — AWS console-based config |
| **Cost at our scale** | Free up to 25K MAU; then ~$0.015/MAU | Free (OSS) + ~$50/month hosting (ECS + RDS) | Free up to 50K MAU; then $0.0055/MAU |
| **Key differentiating feature** | Universal Login, social connections, Actions (serverless hooks), extensive SDK ecosystem | Full customization, user federation, on-premise data | Native AWS integration (IAM, API Gateway, Amplify) |

**Decision: Auth0** — The managed model eliminates authentication infrastructure from the team's operational scope. Auth0's free tier (25K MAU) covers EventPass's MVP phase, and the Universal Login page provides a secure, customizable authentication UI out of the box. Cognito is cheaper at scale but has a limited customization model and weaker social login support. Keycloak provides maximum control but requires hosting a Java application — an operational burden that contradicts the "minimize infrastructure" philosophy of a small team.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Zero auth infrastructure to manage — Auth0 handles password storage, hashing, and brute force protection | Vendor dependency — Auth0 outage blocks new logins (mitigated by JWKS caching for existing sessions) |
| Pre-built Universal Login UI — secure by default, customizable with CSS/HTML | Cost increases with user growth — at 50K MAU, cost is ~$750/month |
| Social login (Google, Facebook) reduces registration friction for buyers | Customization limits — advanced auth flows require Auth0 Actions (serverless with cold starts) |
| Built-in MFA support — TOTP, SMS, email verification | Authentication data lives on Auth0's infrastructure — no on-premise option |
| Comprehensive SDKs for Node.js (backend) and React (frontend) | Auth0's dashboard UX has a learning curve for complex RBAC configurations |

## 5. Integration

- **Identity module:** The `infrastructure/auth0.adapter.ts` wraps all Auth0 API interactions behind the `IdentityFacade`. Other modules never call Auth0 directly. See [ADR-005](../adrs/ADR-005-authentication.md).
- **API Gateway (Kong):** JWT validation plugin verifies Auth0-issued tokens using the JWKS endpoint. Public keys cached for 24h. See [01-api-gateway.md](01-api-gateway.md).
- **Auth0 configuration:**
  - Tenant: `eventpass.auth0.com`
  - Applications: SPA (Next.js frontend), Machine-to-Machine (backend API)
  - Database connection: Auth0-managed for email/password
  - Social connections: Google, Facebook
  - Post-login Action: injects `role` and `userId` into access token claims
- **Frontend (Next.js):** Uses `@auth0/nextjs-auth0` SDK for login/logout flows and token management. See [ADR-010](../adrs/ADR-010-frontend-architecture.md).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — Auth0 is an external service accessed from the Private VPC via NAT Gateway.
