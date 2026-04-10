# ADR-007: API Design

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass exposes an API consumed by two clients: the Next.js frontend (SSR pages and client-side interactions) and external webhooks (Stripe payment notifications). The API must support CRUD operations for events, ticket browsing and purchasing, order management, and user authentication flows.

Within the Modular Monolith (ADR-001), there is no inter-service network communication — modules communicate in-process via facades (ADR-002). The API design decision applies exclusively to the **external-facing API** between the frontend/webhooks and the backend application.

The API must be easy to consume from React Server Components (SSR) and client-side React Query (mutations), well-documented for potential future third-party integrations (e.g., event listing partners), and consistent across all 7 bounded contexts.

## Options Considered

### Option 1: REST for Everything

RESTful API with JSON payloads, HTTP methods for CRUD, and standard status codes.

| Pros | Cons |
|------|------|
| Universal familiarity — every developer on the team knows REST | Over-fetching — event listing pages may need data from multiple endpoints (event + tickets + venue) |
| Excellent tooling — OpenAPI/Swagger for documentation, Postman for testing | No real-time capabilities — requires polling or WebSocket addition for live updates (e.g., ticket availability during flash sales) |
| Cache-friendly — HTTP caching (ETags, Cache-Control) works natively with CDN and Redis | Versioning complexity — breaking changes require URL versioning (`/v1/events`) or header-based negotiation |
| Natural fit for Stripe webhooks (HTTP POST to a REST endpoint) | Multiple round trips for complex pages (mitigated by Server Components doing batch fetches) |
| Easy to secure with standard middleware (JWT validation, rate limiting) | No type safety between client and server without additional tooling (e.g., OpenAPI codegen) |

### Option 2: GraphQL for Frontend + REST for Webhooks

GraphQL API for all frontend queries and mutations, REST endpoints only for external webhooks.

| Pros | Cons |
|------|------|
| Eliminates over-fetching — frontend requests exactly the fields it needs | Additional complexity — GraphQL server (Apollo/Yoga), schema design, resolver implementation |
| Single endpoint reduces client-side coordination — one request for event + tickets + venue | N+1 query problem — resolvers can generate excessive database queries without DataLoader |
| Built-in type system — schema serves as API documentation and enables codegen | Caching is harder — HTTP caching doesn't work with POST-only GraphQL; requires Apollo Cache or similar |
| Real-time subscriptions for live ticket availability during flash sales | Security surface area — query depth/complexity limits needed to prevent denial-of-service |
| Excellent developer experience with GraphQL Playground/Explorer | Team has limited GraphQL experience — learning curve for schema design and resolver patterns |

### Option 3: REST External + gRPC Internal

REST for the external API (frontend and webhooks), gRPC for internal module-to-module communication.

| Pros | Cons |
|------|------|
| gRPC's Protocol Buffers provide type-safe inter-service contracts | In a Modular Monolith, internal communication is in-process — gRPC adds unnecessary serialization overhead |
| Binary protocol is faster than JSON for high-throughput internal communication | Two API technologies to maintain — REST for external, gRPC for internal |
| Streaming support for real-time data flows | Frontend cannot consume gRPC directly — requires a REST gateway or gRPC-Web |
| Strong tooling for code generation across languages | Over-engineering for current architecture — gRPC benefits emerge with network-based inter-service calls |
| Natural evolution path when extracting modules to microservices | Protocol Buffers schema management adds development overhead |

## Decision

**REST for everything** (external API in the current Modular Monolith phase).

In a Modular Monolith, internal module communication is in-process via facade calls — there is no network boundary where gRPC or GraphQL would provide benefits. The external API is the only network-facing surface, and REST is the simplest, most widely understood choice.

### API Design Conventions

**Base URL:** `https://api.eventpass.com/v1`

**Resource Structure:**

| Module | Endpoints | Methods |
|--------|-----------|---------|
| Identity | `/auth/register`, `/auth/login`, `/users/:id`, `/admin/organizers/:id/approve` | POST, GET, PATCH |
| Event Management | `/events`, `/events/:id`, `/events/:id/cancel`, `/venues` | GET, POST, PUT, PATCH |
| Catalog | `/catalog/events`, `/catalog/search`, `/catalog/featured` | GET |
| Ticketing | `/events/:id/tickets`, `/events/:id/tickets/reserve`, `/tickets/:id/validate` | GET, POST |
| Orders | `/orders`, `/orders/:id`, `/cart` | GET, POST, PUT, DELETE |
| Payment | `/payments/intent`, `/payments/retry`, `/webhooks/stripe` | POST |
| Notification | `/notifications/preferences` | GET, PUT |

**Standards:**
- JSON request/response bodies with `Content-Type: application/json`.
- HTTP status codes: 200 (OK), 201 (Created), 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found), 409 (Conflict — e.g., tickets already reserved), 429 (Rate Limited), 500 (Server Error).
- Pagination: cursor-based for listing endpoints (`?cursor=xxx&limit=20`).
- Error format: `{ "error": { "code": "TICKETS_UNAVAILABLE", "message": "...", "details": {...} } }`.
- API documentation via OpenAPI 3.0 specification, auto-generated from route definitions.

### Future Considerations

If EventPass evolves to need real-time ticket availability during flash sales, we will add **WebSocket support** as a supplementary channel (not a replacement for REST). If modules are extracted into microservices, **gRPC** becomes the natural choice for inter-service communication — the facade interfaces already define the contracts.

## Consequences

**Enables:**
- Fast implementation — REST is the most familiar API pattern for the team and the Next.js ecosystem.
- Cache-friendly architecture — event listings and search results cacheable via HTTP Cache-Control headers and CDN (Cloudflare).
- Standard security patterns — JWT Bearer tokens in Authorization header, rate limiting via middleware, CORS configuration.
- OpenAPI documentation — machine-readable API specification enables future third-party integrations and SDK generation.

**Prevents:**
- Efficient data fetching for complex views — event detail pages may require 2-3 API calls (event + tickets + venue). Mitigated by Next.js Server Components batching these on the server side.
- Real-time updates without additional infrastructure — polling is the default; WebSocket or SSE would need to be added separately.

**Risks and Mitigations:**
- **Risk:** Over-fetching on mobile clients with limited bandwidth. **Mitigation:** Use sparse fieldsets (`?fields=id,title,minPrice`) for listing endpoints; compress responses with gzip/brotli.
- **Risk:** API versioning becomes complex as the product evolves. **Mitigation:** Start with `/v1` prefix; use additive changes (new fields, new endpoints) instead of breaking changes; deprecation headers for sunset fields.
- **Risk:** Inconsistent API design across modules. **Mitigation:** Shared API conventions document; code review checklist; consider a shared request/response validation middleware.

## Related ADRs

- [ADR-001: Deployment Model](ADR-001-deployment-model.md) — Monolith eliminates need for inter-service API (gRPC unnecessary)
- [ADR-002: Communication Style](ADR-002-communication-style.md) — Sync facade calls replace internal API; REST is external only
- [ADR-005: Authentication](ADR-005-authentication.md) — Auth0 JWT tokens secure REST endpoints
