# ADR-003: Database Strategy

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass's 7 bounded contexts each own distinct data: users and profiles (Identity), events and venues (Event Management), ticket inventory with concurrency-sensitive reservations (Ticketing), orders and purchase history (Orders), payment records with Stripe references (Payment), search indexes (Catalog), and notification logs (Notification).

The database strategy must balance data isolation between contexts (preventing accidental coupling) with operational simplicity (a team of 3-5 developers managing the infrastructure). The Ticketing module requires strong consistency for inventory management during flash sales (~5K concurrent users), while the Catalog module prioritizes read performance over transactional guarantees.

## Options Considered

### Option 1: Single Shared Database and Schema

All modules use one PostgreSQL database with one shared schema. Tables like `users`, `events`, `tickets`, and `orders` coexist in the `public` schema.

| Pros | Cons |
|------|------|
| Simplest operational model — one database, one connection pool, one backup strategy | No data isolation — any module can JOIN across all tables, creating hidden coupling |
| Cross-module queries are trivial (e.g., `SELECT * FROM orders JOIN users`) | Schema changes in one domain can break other modules (e.g., renaming `users.email` affects every module that queries it) |
| Single transaction scope — ACID guarantees across module boundaries | Impossible to extract a module without untangling shared table dependencies |
| Easiest for a small team to manage | No clear ownership — who is responsible for the `users` table if both Identity and Orders reference it? |

### Option 2: Schema-per-Module on a Shared PostgreSQL Instance

Each module owns a PostgreSQL schema within a single database instance: `identity.*`, `events.*`, `catalog.*`, `ticketing.*`, `orders.*`, `payments.*`, `notifications.*`. Cross-schema access is prohibited at the application level.

| Pros | Cons |
|------|------|
| Logical isolation — each module owns its schema, tables, and migrations | Requires discipline to avoid cross-schema queries (no database-level enforcement without row-level security) |
| One database instance to manage — single backup, single connection pool, single failover | All schemas share the same PostgreSQL instance — a resource-intensive query in Catalog can impact Ticketing performance |
| Migrations are scoped per module — Identity's migration doesn't affect Ticketing's schema | Cannot scale databases independently per module |
| Natural extraction path — move a schema to a new database instance when needed | Shared connection pool means one module's connection leak affects all modules |
| Cross-module data access forced through facades — maintains bounded context integrity | Slightly more complex migration management (7 migration directories vs. 1) |

### Option 3: Database-per-Service

Each bounded context gets its own PostgreSQL database instance (or potentially different database technologies per module).

| Pros | Cons |
|------|------|
| Complete data isolation — no possibility of cross-module queries | 7 database instances: 7 RDS costs (~$50-100/month each), 7 backup policies, 7 connection configurations |
| Independent scaling — Ticketing can use a larger instance during flash sales | No cross-module transactions — requires saga patterns for operations spanning multiple modules |
| Technology diversity — Catalog could use Elasticsearch, Notification could use DynamoDB | Team of 3-5 developers managing 7 databases is operationally expensive |
| Natural fit for future microservices | Connection management complexity — each module needs its own pool configuration |

## Decision

**Option 2: Schema-per-module on a single PostgreSQL instance.**

This provides the data isolation needed to maintain bounded context integrity without the operational overhead of managing 7 separate database instances. At EventPass's current scale (~500 normal users, ~5K flash sale peak), a single RDS `db.r6g.large` instance handles the combined load of all 7 modules comfortably.

### Schema Layout

```sql
CREATE SCHEMA identity;      -- Users, Profiles, OrganizerVerifications
CREATE SCHEMA events;        -- Events, Venues, Categories
CREATE SCHEMA catalog;       -- EventListings, SearchIndexes, FeaturedEvents
CREATE SCHEMA ticketing;     -- TicketTypes, Tickets, SeatReservations
CREATE SCHEMA orders;        -- Orders, OrderItems, Carts
CREATE SCHEMA payments;      -- PaymentIntents, Refunds, OrganizerPayouts
CREATE SCHEMA notifications; -- Templates, Logs, Preferences
```

### Access Rules

1. Each module's database layer uses `SET search_path TO <module_schema>` — queries default to the module's own schema.
2. Cross-schema queries (`SELECT * FROM identity.users`) are prohibited at the application level. ESLint rules flag raw SQL containing schema prefixes outside the owning module.
3. Cross-module data access goes through the module's public facade: `IdentityFacade.getUserById(id)`, not `SELECT * FROM identity.users WHERE id = $1`.

## Consequences

**Enables:**
- Clear data ownership — each module's migration directory manages only its own schema, reducing conflict risk.
- Simple extraction path — when a module is extracted to a separate service, its schema is migrated to a new database instance with minimal changes.
- Single operational footprint — one RDS instance with automated backups, one connection pool, one monitoring dashboard.
- Cross-module transaction support when truly needed — possible via database-level transactions since all schemas are on the same instance (used sparingly, e.g., during reservation-to-order creation).

**Prevents:**
- Independent database scaling — if Ticketing needs more database resources during flash sales, we must scale the entire instance.
- Technology diversity — all modules must use PostgreSQL (though this is acceptable since PostgreSQL handles all our use cases: relational data, full-text search via `tsvector`, JSON storage, and point-in-time recovery).

**Risks and Mitigations:**
- **Risk:** Developers write cross-schema queries bypassing the facade. **Mitigation:** ESLint rules flag schema-prefixed SQL; code review checklist includes "no cross-schema queries"; database user per schema with restricted permissions if discipline erodes.
- **Risk:** Resource contention — a slow Catalog query impacts Ticketing's inventory operations. **Mitigation:** Connection pool limits per module (e.g., Ticketing gets 30% of the pool); read replicas for Catalog's search-heavy workload; Redis caching reduces database load for hot paths.
- **Risk:** Single point of failure — if the database goes down, all modules are affected. **Mitigation:** RDS Multi-AZ deployment with automatic failover; point-in-time recovery; connection retry logic in the shared infrastructure layer.

## Related ADRs

- [ADR-001: Deployment Model](ADR-001-deployment-model.md) — Modular Monolith aligns with shared database infrastructure
- [ADR-004: Event Bus](ADR-004-event-bus.md) — events enable eventual consistency between modules without cross-schema queries
