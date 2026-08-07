# Caching

## The Core Idea

A cache is a faster, smaller storage layer that sits in front of a slower one. Instead of hitting a database on every request, you keep frequently-needed data in memory where reads are ~1ms instead of ~50ms. The tradeoff: the cache might be stale, and it adds complexity around invalidation.

The most important question before adding a cache: **is your bottleneck actually reads?** Caching doesn't help write-heavy systems or systems where every read needs fresh data.

## Mental Model / Analogy

Think of a chef's mise en place — the prep work done before service starts. Common ingredients (diced onions, minced garlic) are prepped and placed in small containers on the counter, not stored in the walk-in cooler. During the dinner rush, the chef grabs from the counter (cache hit), not the cooler (database). But if the kitchen runs out of prepped garlic, the chef has to go get more from the cooler and prep it (cache miss + repopulate). And if a dish is 86'd for the night, the chef clears that prep container (cache invalidation).

---

## Where to Cache

**External Cache (Redis / Memcached)** — The default for system design interviews. Shared across all app server instances, so any instance can serve from the same cache. Redis supports data structures, pub/sub, and persistence. Memcached is simpler and faster for pure key-value. Target this layer first.

**CDN** — Caches static assets (images, videos, CSS, JS) at edge locations geographically close to users. Reduces cross-region latency from 250-300ms to 20-40ms. Works as a read-through cache — on miss, the CDN fetches from origin and stores it. Best for content that changes infrequently.

**Client-Side** — Browser localStorage, session storage, mobile app local storage. Eliminates network round-trips for data that only the current user needs (user preferences, auth tokens, drafts). Limited to a single device; cleared when user logs out.

**In-Process (Application-Level)** — A map/dictionary inside the app server's memory. Fastest possible reads (no network hop), but not shared across instances. Only appropriate for truly global, rarely-changing data: config flags, feature switches, rate limit counters. If you have 50 app servers, you have 50 copies of this cache — data updates must propagate to all of them.

---

## Cache Architectures

### Cache-Aside (Lazy Loading) — Default Choice

The application manages the cache manually:

```
Read: Check cache → cache hit → return
                  → cache miss → fetch from DB → write to cache → return
Write: Write to DB → (optionally invalidate/update cache)
```

**Why it's the default:** Simple, only caches data that's actually needed, works for any data store. The downside is a cold cache on startup — first reads are always slow until the cache warms up.

### Write-Through

Every write goes to the cache first, and the cache synchronously writes to the database before confirming success. Cache and database are always in sync. Slower writes (two hops), and you cache data that may never be read. Best for cases where the write path is also the hot read path.

### Write-Behind (Write-Back)

The application writes to the cache, and the cache asynchronously flushes to the database in the background. Fastest writes possible. Risk: if the cache crashes before flushing, those writes are lost. Use for high-write throughput where brief data loss is acceptable (like a game leaderboard).

### Read-Through

The cache acts as a proxy — the app only talks to the cache. On a miss, the cache itself fetches from the database and populates itself. The app never touches the database directly. CDNs are the most common real-world read-through cache. Less common at the app layer.

---

## Eviction Policies

When the cache is full, something must go:

| Policy | Mechanism | Best for |
|--------|-----------|----------|
| **LRU** (Least Recently Used) | Remove the item not accessed in the longest time | General purpose — most interview situations |
| **LFU** (Least Frequently Used) | Track access count, remove the least-accessed | Consistently popular items (viral videos, top playlists) |
| **FIFO** | Remove oldest-inserted item regardless of usage | Rare; ignores access patterns |
| **TTL** | Expire items after a fixed time window | Freshness requirements; combined with LRU/LFU |

Redis default eviction is **allkeys-lru** — when memory is full, it removes the least recently used key across all keys. For most workloads, this is correct.

---

## Cache Problems and Solutions

### Cache Stampede (Thundering Herd)

A very popular item's cache entry expires. Simultaneously, thousands of requests miss and slam the database to rebuild the same entry.

**Fix 1 — Request coalescing (single-flight):** Only let one request rebuild the cache entry. All other requests with the same key wait and get the result when the first request completes. Many frameworks have this built in.

**Fix 2 — Cache warming:** Proactively refresh popular entries before they expire, not after. Requires knowing what's popular ahead of time (works for TTL-based caching, not write-invalidation-based).

### Cache Consistency

Your database is the source of truth. If a user updates their profile, the cache holds the old version until it's invalidated.

**Fix — Invalidate on write:** When data is written to the database, delete the corresponding cache entry. The next read will miss and repopulate from the database. This is simpler than trying to update the cache atomically with the DB write.

**Accept eventual consistency** for data where brief staleness is fine: feed posts, notification counts, analytics dashboards. Set a TTL that matches your tolerance (5 seconds for counts, 5 minutes for recommendations).

### Hot Keys

A single cache key receives disproportionately high traffic (e.g., a celebrity's profile page with millions of followers). Even though it's in cache, a single Redis node serving that key becomes a bottleneck.

**Fix 1 — Key replication:** Store the hot key as `user:taylorswift:0` through `user:taylorswift:9`, and randomly select one to read from. This spreads load across multiple cache nodes.

**Fix 2 — In-process local cache fallback:** Add a small local cache inside each app server for the hottest keys. The app server checks local cache first, then Redis. Each server absorbs a fraction of the load.

---

## Interview Checklist

When proposing a cache in an interview, walk through 5 questions:

1. **What's the bottleneck?** (Confirm it's reads, not writes — run the numbers)
2. **What to cache?** (Read-heavy data that's expensive to compute and infrequently changed)
3. **Which layer?** (External Redis for shared app data, CDN for static media)
4. **Eviction policy?** (LRU + TTL is the safe default unless you have specific hotness patterns)
5. **What are the downsides?** (Staleness window, thundering herd on expiry, Redis failure behavior — app should gracefully fall back to DB if Redis is unavailable)

---

## Key Trade-offs

| What you gain | What you lose |
|---------------|---------------|
| Sub-millisecond reads | Stale data risk |
| Reduced DB load | Cache invalidation complexity |
| Higher throughput | Additional infrastructure to manage |
| Lower latency at scale | Cold start (empty cache) cost |

## When to Use

- Read-to-write ratio is high (10:1 or more)
- Queries are expensive and repeatedly asked (feed generation, search results, recommendation scores)
- Data changes infrequently relative to how often it's read (product catalog, user profiles)
- Acceptable to show data that's a few seconds/minutes stale

## Common Gotchas

- **Caching solves reads, not writes** — if you're write-bound, a cache won't help and may hurt
- **Double write consistency** — writing to cache and DB separately is not atomic; failures between the two leave them out of sync. Delete the cache entry instead of updating it
- **Short TTLs don't solve cache stampede** — they make it worse by causing more frequent simultaneous expirations
- **Redis single-threaded** — one slow command (like a large `KEYS *` scan) blocks all other commands. Avoid scanning large keyspaces in production

## Practice Questions

1. Your news feed API returns the same timeline for most users but recalculates it from scratch on every request. Propose a caching strategy. How would you handle the case where a followed user posts something new?
2. You run a CDN-cached product catalog page. A product goes out of stock. How do you ensure users see the updated status within 30 seconds given CDN TTLs?
3. A viral tweet causes 500,000 requests/second for `user:id:12345`. Your single Redis node handling that key is at 100% CPU. Walk through two approaches to resolve this without changing the database.
4. You're designing a rate limiter that allows 100 requests per user per minute. Why is an in-process counter (instead of Redis) problematic if you have 20 app servers? How would you make it work?
5. Your cache miss rate has jumped from 5% to 60% overnight. Walk through how you'd diagnose this — what metrics would you check and what are the three most likely root causes?

## One-Line Summary

Cache is a fast, memory-backed layer in front of your database — default to external Redis with cache-aside pattern and LRU eviction, invalidate on write, and treat thundering herd and hot keys as the two problems most likely to bite you at scale.
