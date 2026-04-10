# Primary Database — PostgreSQL (AWS RDS)

## 1. Responsibilities

- **Transactional data storage for all 7 modules** — Store users, events, tickets, orders, payments, notifications, and catalog data with ACID guarantees. Schema-per-module isolation: `identity.*`, `events.*`, `catalog.*`, `ticketing.*`, `orders.*`, `payments.*`, `notifications.*`.
- **Inventory consistency under concurrency** — Use `SELECT ... FOR UPDATE SKIP LOCKED` on `ticketing.tickets` to handle ~5K concurrent buyers competing for limited ticket inventory during flash sales without deadlocks.
- **Full-text search** — PostgreSQL's `tsvector` and GIN indexes power the Catalog module's event search functionality, avoiding the need for a separate search engine at current scale.
- **Audit trail and compliance** — Store complete order and payment history with timestamps for financial auditing. Point-in-time recovery (PITR) enables restoring the database to any second within the retention window.
- **Geospatial queries** — PostGIS extension enables venue location searches ("events near me") in the Catalog module using `ST_DWithin` queries on venue coordinates.

## 2. Why This Project Needs It

EventPass's core transaction — ticket reservation and purchase — requires strict ACID guarantees. When a buyer reserves a ticket, the database must atomically update the ticket status from AVAILABLE to RESERVED and set the 10-minute TTL. If two buyers attempt to reserve the same ticket simultaneously, exactly one must succeed and the other must receive an immediate "unavailable" response. PostgreSQL's row-level locking with `SKIP LOCKED` provides this guarantee without blocking concurrent transactions on other tickets.

The schema-per-module strategy (ADR-003) requires a database that supports multiple schemas within a single instance. PostgreSQL's native schema support (`CREATE SCHEMA ticketing;`) provides logical isolation without the operational overhead of separate database servers.

## 3. Technology Choice

| Dimension | PostgreSQL (RDS) | MySQL (RDS) | MongoDB Atlas |
|-----------|-----------------|-------------|---------------|
| **Managed/Self-hosted** | Fully managed (AWS RDS) | Fully managed (AWS RDS) | Fully managed (MongoDB Atlas) |
| **Operational complexity** | Low — RDS handles backups, patching, failover | Low — RDS managed, similar to PostgreSQL | Low — Atlas managed, different operational model |
| **Cost at our scale** | ~$200/month (db.r6g.large, Multi-AZ) | ~$180/month (db.r6g.large, Multi-AZ) | ~$170/month (M10 dedicated cluster) |
| **Key differentiating feature** | Advanced locking (SKIP LOCKED), native schema isolation, PostGIS, tsvector, JSONB | Simpler replication, widely adopted, good read performance | Document model, flexible schema, horizontal scaling via sharding |

**Decision: PostgreSQL (RDS)** — The `SKIP LOCKED` locking mode is critical for EventPass's flash sale concurrency model — MySQL does not support it natively. PostgreSQL's schema isolation aligns with the schema-per-module strategy, and built-in `tsvector` eliminates the need for a separate search engine at current scale. MongoDB's document model would require denormalization that conflicts with our relational bounded contexts (e.g., Order references TicketType, which references Event).

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| ACID transactions across all operations — critical for financial data (orders, payments, refunds) | Vertical scaling only — single instance must handle all 7 modules' load |
| `SKIP LOCKED` enables non-blocking concurrent ticket reservations | Schema-per-module is a convention — no database-level enforcement prevents cross-schema queries |
| Native full-text search (tsvector) eliminates need for Elasticsearch at current scale | Connection pool shared across all modules — one module's connection leak affects others |
| PostGIS extension provides geospatial queries without additional infrastructure | Single point of failure — mitigated by Multi-AZ but failover takes 60-120 seconds |
| Schema isolation provides clear module boundaries with minimal operational overhead | Cannot use different database technologies per module (e.g., time-series for analytics) |

## 5. Integration

- **All 7 modules** connect via the shared connection pool in `shared/infrastructure/database/connection.ts`. Each module's repository layer sets `search_path` to its own schema. See [ADR-003](../adrs/ADR-003-database-strategy.md).
- **Connection pool:** Using `pg-pool` with 50 connections allocated proportionally — Ticketing (30%), Orders (20%), Catalog (20%), Payment (15%), Identity (10%), Notification (3%), Events (2%).
- **Redis integration:** Hot data (ticket inventory, popular listings) is cached in Redis (see [04-cache.md](04-cache.md)) to reduce read load on PostgreSQL.
- **Backups:** RDS automated backups with 7-day retention + manual snapshots before deployments. PITR granularity: 5 minutes. Snapshots stored on S3 (see [07-object-storage.md](07-object-storage.md)).
- **Monitoring:** PostgreSQL metrics (connections, query latency, replication lag, IOPS) exported to Prometheus via `pg_stat_statements`. Dashboards and alerts in Grafana. See [11-observability-stack.md](11-observability-stack.md).
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — RDS runs in the Managed Data Services zone with Multi-AZ standby.
