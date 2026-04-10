# Proposals

Architecture proposal documents for EventPass. These documents establish the foundational architectural decisions, domain decomposition, code organization, and key interaction flows for the platform.

## Documents

| # | Document | Description |
|---|----------|-------------|
| 1 | [High-Level Architecture](01-high-level-architecture.md) | Compares Modular Monolith vs. Microservices for EventPass with 8-dimension analysis and recommendation for Modular Monolith |
| 2 | [Bounded Contexts](02-bounded-contexts.md) | Defines 7 DDD bounded contexts with typed entities, domain events, integration patterns, and a Mermaid context map |
| 3 | [Service/Module Decomposition](03-service-module-decomposition.md) | Details the Modular Monolith code structure: module ownership, public APIs, boundary enforcement, and future extraction path |
| 4 | [Data Flow and Interactions](04-data-flow-and-interactions.md) | Documents 4 end-to-end flows (registration, ticket purchase, payment retry, event cancellation) with sequence diagrams |

## Key Decisions Made

- **Architecture:** Modular Monolith (team of 3-5 devs, MVP phase, clear migration path to microservices)
- **Domain Model:** 7 bounded contexts (3 Core, 2 Supporting, 2 Generic)
- **Module Communication:** Facade calls (sync) + internal event bus (async)
- **Boundary Enforcement:** ESLint rules preventing cross-module internal imports
- **Database Isolation:** Schema-per-module on single PostgreSQL instance
