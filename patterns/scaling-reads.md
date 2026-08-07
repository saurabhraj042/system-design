# Scaling Reads

## The Core Idea

Read scaling means making data retrieval faster or distributing read load across more machines. Unlike writes, reads don't mutate state — which opens options not available for writes: caching, read replicas, and CDN edge serving. The right strategy depends on where the bottleneck is. Most systems that struggle under read load haven't exhausted their single-database options yet.

Three strategies in order of complexity: optimize reads within your existing database, scale horizontally with replicas or sharding, and add external caching.

## Mental Model / Analogy

Think of a reference library. First, you reorganize the card catalog so every book is easy to find (add indexes, use better hardware). When the single desk still can't keep up, you hire more librarians who each have their own copy of the catalog (read replicas). For the most popular books, you make a copy and place it in the lobby where anyone can grab it instantly without talking to a librarian at all (caching). Each layer handles more volume than the one before, but also adds more infrastructure to maintain.

---

## Strategy 1: Optimize Within Your Database

Most read bottlenecks are solved here — don't add infrastructure you don't need.

### Indexes

Without an index, a query scans every row in a table (O(n)). With a B-tree index, the database jumps to matching rows in O(log n) via a sorted tree structure. The improvement for a 412,000-row table queried by email goes from a sequential scan of the entire table to an index seek with a cost under 10.

**When to add an index:**
- Columns that appear in `WHERE` clauses
- Columns used in `JOIN` conditions
- Columns referenced in `ORDER BY` when sorted results are needed

**Composite indexes:** A composite index on `(user_id, created_at)` serves the query "posts by this user, sorted by time" in a single index scan. Column order matters: the index is traversable from left to right only. An index on `(user_id, created_at)` helps queries filtering on `user_id` alone — but does nothing for queries filtering on `created_at` alone.

**Rule of thumb:** Under-indexing kills more applications than over-indexing. Add indexes for your critical query patterns. Remove indexes that serve no query — they add overhead to every write.

### Hardware Upgrades

- **SSD over HDD:** 10–100× faster random I/O. Most cloud databases already use SSDs, but verify your storage class.
- **More RAM:** The database keeps recently read pages in its buffer pool. If your working dataset fits in RAM, reads never hit disk. Adding RAM is often faster and cheaper than resharding.

### Denormalization and Materialized Views

**Denormalization:** Store redundant data to eliminate expensive JOIN operations at read time.

```sql
-- Normalized: reading an order requires JOINing orders + customers + products
SELECT o.id, c.name, p.name, o.quantity, o.total
FROM orders o JOIN customers c ON ... JOIN products p ON ...

-- Denormalized order_summary table (written once on order creation):
SELECT * FROM order_summary WHERE order_id = ?  -- single table read
```

The trade-off: writes become more expensive because you must update multiple tables. Changes to source data (a customer's name changes) require updating every denormalized copy.

**Materialized views:** Precomputed aggregations stored as real tables, refreshed on a schedule or on write. A product's average rating, computed from millions of reviews at query time, becomes a single row read if you maintain a `product_avg_rating` materialized view that refreshes when new reviews arrive. Use these when an aggregation is expensive to compute at query time but the underlying data changes infrequently.

---

## Strategy 2: Scale Horizontally

**Threshold to consider horizontal scaling:** approximately 50,000–100,000 reads/second with good indexes. Below that, the strategies in Strategy 1 almost always suffice.

### Read Replicas

Add follower databases that receive every write from the primary via replication log. Route all write traffic to the primary; distribute read traffic across followers.

**Synchronous replication:** The primary waits for at least one follower to confirm before acknowledging a write. Guarantees no data loss and no stale reads. Slower writes.

**Asynchronous replication:** The primary acknowledges immediately; followers catch up in the background. Faster writes. Risk: if the primary fails before a follower applies its latest log entries, those writes are lost. More common risk: **replication lag** — a read replica may return data that's seconds behind the primary. Always a deep-dive topic in interviews.

**When replication lag matters:** "Read your own writes" consistency. A user submits a form, then immediately reloads the page — if the read hits a replica that hasn't replicated yet, the user sees stale data. Solutions: route reads-after-writes for the same user to the primary for a short window, or use synchronous replication for critical paths.

### Database Sharding

When storage or read throughput exceeds what even a replica set can handle, partition data across multiple databases.

**Functional sharding:** Split by domain. Instead of one monolithic database, have a `users` database, a `posts` database, and a `likes` database. Each service team owns their database. This is often the first sharding move — it's low-risk and aligns with service boundaries.

**Geographic sharding:** Partition data by region. Users in the US read from US databases; EU users from EU databases. Cross-region latency (50–150ms) is eliminated for the common case because users' data lives in their region. Works when users' data is inherently regional (a Miami user will never need data from a Tokyo shard in normal operation).

**Horizontal sharding by entity ID:** Split rows of the same table across machines based on a hash of the primary key. Distributes both storage and read load. Requires the query to know which shard to target — cross-shard queries (e.g., "posts from all users I follow") are expensive and require scatter-gather.

---

## Strategy 3: External Caching

Caching serves read results from fast memory (Redis, Memcached) instead of the database. Latency drops from 5–30ms (database disk read) to 0.1–1ms (cache hit).

### Application-Level Caching (Cache-Aside)

The application code is responsible for populating and invalidating the cache.

```
Read path:  check cache → hit: return immediately
                        → miss: query DB, write to cache, return
Write path: update DB → invalidate or update cache entry
```

**TTL-based expiration:** Set a time-to-live on every cache entry. Entries expire automatically, guaranteeing the cache can never be more than TTL seconds stale. Simple to implement. Works for data where brief staleness is acceptable (product catalogs, user profiles).

**Write-through:** On every write, update both the database and cache simultaneously. Reads always hit the cache. Downside: cache fills with entries that may never be read (you're caching writes that nobody reads).

**Write-behind (write-back):** Write to cache first; database write happens asynchronously. Lowest write latency. Risk: if the cache node fails before the async write completes, data is lost. Use only when occasional data loss is acceptable.

**Tagged invalidation:** Group related cache entries under a tag. When an entity updates, invalidate all entries with that tag. Useful for complex objects that power many different views.

**Versioned keys:** Instead of deleting or updating a cache entry on write, increment a version number. Old key `event:123:v41` is simply abandoned; new writes and reads use `event:123:v42`. Old entries expire via TTL. Avoids cache invalidation races where a slow writer and a fast reader can race to produce inconsistency.

### CDN / Edge Caching

CDNs cache content at edge servers geographically close to users. Latency drops from 200ms (cross-region origin fetch) to under 10ms (edge cache hit).

**Constraint:** CDNs serve the same response to all requesters. Use CDN caching only for content that's identical across users — public API responses, static assets, publicly accessible feeds. Never for user-specific data (a user's private inbox, personalized recommendations).

---

## Deep Dives

### Slow-Growing Dataset — Add Indexes First

Before adding any infrastructure, check if a missing index explains the slow query. A table with 412,000 rows queried by email without an index costs a full sequential scan. Adding an index on email reduces query cost to ~8.45 (index seek + fetch). This is a free performance gain that takes seconds to apply.

`EXPLAIN` (PostgreSQL) or `EXPLAIN ANALYZE` shows the query plan. If you see "Seq Scan" on a large table for a selective query, add an index.

### Cache Stampede (Thundering Herd)

When a popular cache entry expires, many requests simultaneously miss and all query the database — multiplying load by the number of concurrent readers.

**Probabilistic early refresh:** As a cache entry ages, introduce a small random chance of refreshing it before it expires — higher probability as expiration approaches. The entry stays warm most of the time; a small number of requests trigger refreshes before the TTL runs out.

**Background refresh process:** A separate background job refreshes popular keys before they expire, completely decoupled from the request path. Cache entries are never cold for clients; the refresh job absorbs the database load.

### Hot Cache Key (Request Coalescing vs. Fanout)

When a single popular cache key receives thousands of simultaneous misses:

**Request coalescing:** When multiple concurrent requests miss on the same key, only one proceeds to query the database. The rest wait and receive the result from the one DB query. Reduces N simultaneous DB queries to 1 (or at most N where N = number of app servers, since coalescing is local to each server).

**Cache key fanout:** Store the same data across multiple cache keys (`feed:taylor_swift:1` through `feed:taylor_swift:10`). Each client randomly picks one. Hot load is distributed across 10 keys instead of concentrated on one. Trade-off: more cache memory usage; invalidation requires clearing all 10 keys.

### Cache Invalidation for Immediate Consistency

Typical TTL-based caching tolerates several seconds of staleness. When a user updates their data and expects to see the change immediately on next load, TTL-based expiration fails.

**Cache versioning:** On each write to the database, atomically increment a version counter in the same transaction. Cache keys include the version number (`event:123:v42`). Readers check the version before serving from cache; if the cached version is behind, they re-fetch. Old cache entries expire via TTL and are never explicitly deleted — avoiding the race condition where an invalidation message arrives before the new write is committed.

**Deleted item tracking:** For cases where items are deleted but old cache entries still exist, maintain a small set of recently-deleted IDs. Before serving a cached feed item, check if its ID appears in the deleted set. This prevents phantom reads without requiring immediate global cache invalidation.

---

## When to Use Each Strategy

- **Indexes:** Always the first step. Run `EXPLAIN` before adding any infrastructure.
- **Better hardware:** Second step — SSD and RAM upgrades are cheap relative to architectural changes.
- **Denormalization / materialized views:** When you have specific expensive queries that run frequently and the source data doesn't change constantly.
- **Read replicas:** When you need to distribute read load across multiple machines and replication lag is acceptable for most reads.
- **Sharding:** When storage (>50TB) or read volume exceeds what a primary + replica set can handle.
- **Application cache:** When database read latency exceeds your SLA and the data can tolerate some staleness.
- **CDN:** When the same response is served to many users and latency from geography is significant.

## Common Gotchas

- **Adding a cache before checking indexes** — a missing index can cause 200ms queries; adding an index may be all you need
- **Caching user-specific data at the CDN** — CDNs serve one response to all; never cache private or personalized data at the edge
- **Ignoring replication lag** — asynchronous replicas lag by seconds; reads-after-writes must account for this
- **Not planning for cache misses** — cache hit rate of 95% sounds great but at 100k RPS that's 5,000 cache misses/sec hitting your database; the database must handle that load
- **No expiration policy** — a cache without TTL grows unbounded and eventually evicts the entries you most need

## Practice Questions

1. A user profile page takes 800ms. The query joins 4 tables and returns 2KB of data. The table has 50 million rows. Walk through three different interventions in order of complexity and expected impact.
2. Your read replicas have a 3-second replication lag. A user changes their username and immediately refreshes — they see the old name. Describe two approaches to solve "read your own writes" without switching to synchronous replication.
3. A sports app serves live scores to 5 million concurrent users. The same score data is requested millions of times per second. Walk through a caching strategy that handles cache stampede when scores update every 30 seconds.
4. Design the cache invalidation strategy for a social media feed where posts can be deleted or edited. The user expects edits and deletes to be reflected within 1 second, but new posts can have a 5-second delay.
5. An e-commerce site has a "Products you may like" API that queries a recommendation model for each user. The model runs in 200ms. This endpoint is called on every page load. Design a caching strategy given that recommendations only need to refresh every 10 minutes per user.

## One-Line Summary

Scale reads by exhausting single-machine options first (indexes, hardware, denormalization), then distributing load with read replicas or sharding, and finally serving hot data from Redis or CDN caches with explicit TTL and invalidation strategies.
