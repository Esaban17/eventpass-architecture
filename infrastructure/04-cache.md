# Cache — Redis (AWS ElastiCache)

## 1. Responsibilities

- **Ticket inventory caching** — Store real-time available ticket counts per event (`ticketing:event:{id}:available`) with 5s TTL. Atomic `DECR` operations prevent overselling during flash sales without requiring a database round-trip for every availability check.
- **User session storage** — Store cart contents, last viewed events, and session metadata (`session:{userId}`) with 30-minute TTL, enabling fast retrieval across all ECS task instances.
- **Search result caching** — Cache Catalog module query results (`catalog:search:{queryHash}`) with 30s TTL, reducing PostgreSQL load for repeated searches like "concerts in Guatemala City."
- **Popular event listings** — Cache the homepage popular events response (`catalog:listings:popular`) with 60s TTL, serving the highest-traffic endpoint entirely from memory.
- **Rate limiter backing store** — Power the Kong API Gateway's distributed rate limiting counters (`ratelimit:{ip}:{endpoint}`) with atomic `INCR` and TTL operations.

## 2. Why This Project Needs It

During a flash sale, ~5K concurrent users repeatedly check ticket availability while attempting to reserve and purchase tickets. Without Redis, each availability check would execute `SELECT COUNT(*) FROM ticketing.tickets WHERE event_id = $1 AND status = 'AVAILABLE'` — generating ~2,500 queries/second on a single table. Redis reduces this to ~50 queries/second (one per 5s TTL window), with the remaining requests served from memory in <1ms.

The atomic `DECR` operation is essential for inventory accuracy: when the Ticketing module writes a ticket status change to PostgreSQL, it simultaneously decrements the Redis counter. This write-through pattern ensures all ECS task instances see the same inventory count — preventing the scenario where two instances independently sell the "last" ticket.

## 3. Technology Choice

| Dimension | Redis (ElastiCache) | Memcached (ElastiCache) | KeyDB |
|-----------|--------------------|-----------------------|-------|
| **Managed/Self-hosted** | Fully managed (AWS ElastiCache) | Fully managed (AWS ElastiCache) | Self-hosted on ECS |
| **Operational complexity** | Low — AWS handles patching, failover, backups | Low — AWS managed, simpler config | Moderate — must manage ECS task, no managed service |
| **Cost at our scale** | ~$25/month (cache.t4g.small) | ~$20/month (cache.t4g.small) | ~$15/month (ECS task) |
| **Key differentiating feature** | Advanced data structures (sorted sets, hashes, streams), atomic operations, pub/sub | Multi-threaded — higher throughput for simple get/set | Redis-compatible with multi-threading and active-active replication |

**Decision: Redis (ElastiCache)** — Redis's atomic `DECR` operation is non-negotiable for ticket inventory management. Memcached's key-value-only model would require application-level locking to achieve the same inventory consistency, introducing race conditions. KeyDB is Redis-compatible but lacks a managed AWS service, adding operational burden.

## 4. Trade-offs

| Advantages | Disadvantages |
|------------|---------------|
| Atomic operations (DECR, INCR) prevent inventory race conditions during concurrent purchases | All cached data must fit in RAM — cache.t4g.small provides 1.37GB (sufficient for current scale but requires monitoring) |
| Sub-millisecond read latency — ticket availability responses are nearly instant | Additional infrastructure component with its own failure mode — Redis outage degrades (but does not break) the system |
| Pub/sub capability available for future cache invalidation patterns | Single-threaded per core — complex Lua scripts or large KEYS scans can block other operations |
| ElastiCache provides automatic failover with Multi-AZ replica | Network hop (~1ms) from ECS to ElastiCache — not zero-latency like in-process caching |
| TTL per key enables different freshness guarantees for different data types | Data loss possible on failover if using in-memory only (mitigated by AOF persistence) |

## 5. Integration

- **Ticketing module:** Write-through for inventory counts — every ticket status change (AVAILABLE→RESERVED, RESERVED→SOLD) updates both PostgreSQL and Redis atomically. See [ADR-008](../adrs/ADR-008-caching-strategy.md).
- **Catalog module:** Read-through for search results and event listings — cache miss triggers a database query, result stored in Redis with appropriate TTL.
- **API Gateway (Kong):** Shares the same ElastiCache instance for distributed rate limiting counters, using a `ratelimit:` key prefix. See [01-api-gateway.md](01-api-gateway.md).
- **Identity module:** Stores session metadata and cached Auth0 JWKS public keys.
- **Fallback behavior:** If Redis is unavailable, the application falls back to direct PostgreSQL queries. A circuit breaker on the Redis client prevents cascading timeouts — after 3 connection failures in 10 seconds, Redis is bypassed for 30 seconds before retry.
- **Deployment diagram reference:** See [diagrams/04-deployment.md](../diagrams/04-deployment.md) — ElastiCache Redis runs in the Managed Data Services zone within the Private VPC.
