# ADR-008: Caching Strategy

| Field | Value |
|-------|-------|
| Status | Accepted |
| Date | 2026-04-10 |
| Deciders | Estuardo Sabán |

## Context

EventPass has several hot data paths that benefit significantly from caching:

- **Ticket inventory** (~5K concurrent users during flash sales): Buyers check ticket availability repeatedly. Without caching, each check hits PostgreSQL's `ticketing.tickets` table with a `SELECT COUNT(*) WHERE status = 'AVAILABLE'` — at 5K concurrent users polling every 2 seconds, this generates ~2,500 queries/sec on a single table.
- **Event listings** (popular events page): The Catalog module's denormalized event listings are read-heavy (100:1 read/write ratio) and change only when events are published, updated, or tickets are sold.
- **Search results**: Repeated searches for "concerts in Guatemala City this weekend" should not hit PostgreSQL each time.
- **User sessions**: Auth0 JWT validation caches JWKS public keys, but session metadata (cart contents, last viewed events) needs fast access.

The caching strategy must handle both **read-through caching** (event listings, search results) and **write-through caching** (ticket inventory) with appropriate consistency guarantees.

## Options Considered

### Option 1: Redis (via AWS ElastiCache)

In-memory data store with advanced data structures, pub/sub, and atomic operations.

| Pros | Cons |
|------|------|
| Rich data structures — sorted sets for rankings, hashes for sessions, atomic DECR for inventory | Requires memory management — cached data must fit in RAM (cost scales with memory) |
| Atomic operations — `DECR` for ticket inventory prevents overselling without distributed locks | Additional infrastructure component to manage (mitigated by ElastiCache managed service) |
| Pub/sub for cache invalidation — notify application instances when cached data changes | Data loss on restart if persistence is not configured (mitigated by AOF persistence) |
| Sub-millisecond latency — ideal for flash sale ticket availability checks | Single-threaded per core — complex Lua scripts can block other operations |
| TTL support per key — automatic expiration for different data types | Network hop from ECS to ElastiCache adds ~1ms latency vs. in-process cache |

### Option 2: Memcached

Simple key-value in-memory cache with multi-threaded architecture.

| Pros | Cons |
|------|------|
| Multi-threaded — better utilization of multi-core instances for simple get/set workloads | Key-value only — no data structures, no sorted sets, no atomic DECR |
| Simple protocol — easy to implement and debug | No persistence — all data lost on restart |
| Slightly higher throughput for pure get/set operations | No pub/sub — cache invalidation requires application-level coordination |
| Lower memory overhead per key than Redis | No TTL per key in older versions (though modern versions support it) |
| ElastiCache support for managed deployment | Cannot implement ticket inventory counting or session management without application-level complexity |

### Option 3: In-Memory Cache (Node.js LRU)

Application-level cache using an LRU (Least Recently Used) data structure within the Node.js process.

| Pros | Cons |
|------|------|
| Zero network latency — cache is in the same process | Not shared across ECS task instances — each container has its own cache |
| No additional infrastructure — part of the application | Invalidation is instance-local — writing to one instance's cache doesn't update others |
| Simple implementation — `lru-cache` npm package | Memory competes with application heap — large caches increase GC pressure |
| Effective for single-instance deployments | Useless for ticket inventory — concurrent buyers on different instances see different availability counts |
| No operational overhead | Cache is lost on every deployment/restart |

## Decision

**Redis** (deployed via AWS ElastiCache for managed operation).

Redis's advanced data structures are essential for EventPass's caching needs — particularly atomic `DECR` operations for ticket inventory and sorted sets for event rankings. Memcached's key-value-only model would require building these capabilities at the application level, adding complexity and race condition risks. In-memory LRU caching fails in a horizontally scaled deployment (multiple ECS tasks) because each instance would have an inconsistent view of ticket inventory.

### Caching Strategy by Data Type

| Data | Cache Key Pattern | TTL | Strategy | Justification |
|------|------------------|-----|----------|---------------|
| Ticket inventory count | `ticketing:event:{eventId}:available` | 5s | Write-through | Short TTL balances freshness with DB load; write-through on every ticket status change ensures near-real-time accuracy during flash sales |
| User session metadata | `session:{userId}` | 30min | Write-through | Cart contents and user state need consistency across requests |
| Popular event listings | `catalog:listings:popular` | 60s | Read-through | Popular events page is the highest-traffic endpoint; 60s staleness is acceptable |
| Search results | `catalog:search:{queryHash}` | 30s | Read-through | Repeated searches within 30s return cached results; new events appear within 30s of publication |
| Event detail | `catalog:event:{eventId}` | 120s | Read-through, invalidate on update | Event details change infrequently; invalidated when `EventPublished` or `EventCancelled` events fire |
| Rate limiter counters | `ratelimit:{ip}:{endpoint}` | 60s | Write-through | Sliding window rate limiting for API protection during flash sales |

### Invalidation Patterns

- **TTL-based:** Default for read-through caches (search results, listings). Simple, predictable, eventually consistent.
- **Event-driven invalidation:** For data that changes via domain events. When `TicketInventoryUpdated` fires, the Ticketing module updates the Redis inventory count atomically.
- **Write-through:** For data that must be consistent immediately (session metadata, inventory counts). Application writes to both Redis and PostgreSQL in the same operation.

## Consequences

**Enables:**
- Flash sale performance — ticket availability checks served from Redis (~0.5ms) instead of PostgreSQL (~5-20ms), reducing database load by ~95% during peak traffic.
- Consistent inventory view — all ECS task instances read from the same Redis cluster, preventing overselling.
- Flexible caching — different TTLs and strategies per data type match EventPass's varied consistency requirements.
- Rate limiting — Redis's atomic increment/expire operations enable per-IP, per-endpoint rate limiting without database involvement.

**Prevents:**
- Zero-infrastructure caching — Redis requires an ElastiCache cluster (~$25-50/month for a `cache.t4g.small` instance).
- Guaranteed real-time consistency — ticket inventory has up to 5s staleness for read-through queries (write-through updates are immediate, but read caches may lag).

**Risks and Mitigations:**
- **Risk:** Redis failure causes cache misses, and all traffic hits PostgreSQL directly. **Mitigation:** Application falls back to database queries when Redis is unavailable; ElastiCache Multi-AZ provides automatic failover; circuit breaker pattern on the Redis client to prevent cascading timeouts.
- **Risk:** Stale ticket inventory — buyer sees "2 tickets available" but they are already sold. **Mitigation:** The 5s TTL limits staleness; the actual reservation uses `SELECT ... FOR UPDATE SKIP LOCKED` on PostgreSQL, so stale cache never causes overselling — it only causes a buyer to attempt a reservation that may fail.
- **Risk:** Memory pressure on Redis instance. **Mitigation:** Set `maxmemory-policy allkeys-lru` to evict least-recently-used keys when memory is full; monitor memory usage via Grafana; upgrade instance type if needed.

## Related ADRs

- [ADR-003: Database Strategy](ADR-003-database-strategy.md) — Redis complements PostgreSQL, reducing read load on the database
- [ADR-007: API Design](ADR-007-api-design.md) — REST Cache-Control headers work alongside Redis for CDN-level caching
