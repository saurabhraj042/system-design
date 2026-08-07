# Scaling Writes

## The Core Idea

Write scaling means reducing the throughput that any single component must absorb. Most systems fail writes before they fail reads — writes mutate state, require durability guarantees, and can't be satisfied from a cache. The goal across every strategy here is the same: spread the incoming write load across more components, or shrink the number of writes each component sees by combining, deferring, or dropping them.

Four strategies in order of increasing complexity: vertical scaling and smarter database choices, horizontal sharding and partitioning, queues and load shedding, and batching with hierarchical aggregation.

## Mental Model / Analogy

Imagine a single cashier lane at a grocery store. As the line grows, you can first upgrade to a faster cashier (vertical scaling). When one cashier still can't keep up, you open more lanes and assign customers by last name (sharding). When there's a temporary spike at lunch, you let the line form and warn customers there may be a wait — the actual checkout still happens, just with a delay (queues). For recurring bulk orders, you ask the stockroom to tally everything once per hour instead of running separate inventory updates after every single item sold (batching). Each strategy applies at a different level of load.

---

## Strategy 1: Vertical Scaling and Database Choices

Before redesigning your architecture, exhaust what a single machine can do. Modern hardware goes further than most engineers expect.

**Check the actual bottleneck first.** The write ceiling differs by bottleneck:
- **Disk I/O bound:** You're hitting the storage layer's write throughput. Move to NVMe SSDs, tune the write-ahead log (WAL) flush interval, or switch to an LSM-tree database that converts random writes to sequential ones.
- **CPU bound:** Each write is doing expensive computation (encryption, validation, complex trigger logic). Profile first; often the fix is removing unnecessary indexes or disabling features you're not using.
- **Network bound:** At 10 Gbps per server, you're pushing ~1.25 GB/s of data. This is rare unless you're writing large payloads at extreme rates.

**Database choices for write-heavy workloads:**

Standard relational databases use B-tree indexes which require random writes. Every insert must update the B-tree in place. LSM-tree databases instead buffer all writes in memory and flush in sorted batches — turning random writes into fast sequential I/O.

- **Cassandra:** LSM-tree based, handles 10,000+ writes/second per node (vs ~1,000 for a tuned PostgreSQL). Designed for append-heavy workloads like time-series and event logs. Trade-off: no joins, eventual consistency model.
- **InfluxDB / TimescaleDB:** Optimized for time-series data. Columns are homogeneous (all floats, all timestamps) enabling better compression and bulk writes.
- **ClickHouse:** Column-store optimized for append-only analytics. Batches inserts internally.

**Tuning your existing database:** Reduce the number of indexes (each index is a separate write per insert), increase WAL flush intervals if you can tolerate slightly longer durability windows, and disable expensive features like triggers and foreign key constraint enforcement at extreme scale.

---

## Strategy 2: Sharding and Partitioning

When a single machine's write capacity is genuinely exhausted, distribute writes across multiple machines.

### Horizontal Sharding (Same Table, Multiple Machines)

Split rows across machines based on a shard key. All writes for a given key go to one machine; different keys go to different machines.

**The shard key problem:** A good shard key distributes writes evenly. A bad one creates hot shards — one machine drowning while others sit idle. Avoid using time-based keys (all current writes go to today's shard) or status fields (all active records pile onto one machine). Prefer high-cardinality keys like user ID or entity ID.

**Implementation options:**
- **Modulo hashing:** `hash(key) % N` → shard index. Simple, but changing N reshuffles almost every key. Redis Cluster uses CRC16 hashed to one of 16,384 virtual "hash slots" and maps slots to nodes — adding a node moves only a subset of slots.
- **Consistent hashing:** Place keys and nodes on a hash ring; each key routes to the nearest node clockwise. Adding or removing a node moves only the keys in the adjacent ring segment (typically 1/N of total keys instead of nearly all of them).

**Resharding (when you need more shards):** Never cut over atomically. Instead:
1. Start writing to both old and new shard layouts simultaneously (dual-write period)
2. Backfill old data to new shards in the background
3. Once backfill is complete and verified, switch reads to new layout
4. Stop writing to old layout

### Vertical Partitioning (Same Table, Split by Column)

Split a wide table into narrower tables by access pattern, not by row. Different parts of the same entity have different write frequencies:

```
posts table (write-once on create):          post_id, user_id, content, media_urls
post_metrics table (high-frequency writes):   post_id, like_count, comment_count, share_count
post_analytics table (append-only events):    post_id, event_type, timestamp, metadata
```

The benefit: high-frequency counters in `post_metrics` don't compete with large content writes in `posts`. Each table can be sized, indexed, and placed on hardware suited to its write pattern.

---

## Strategy 3: Queues and Load Shedding

### Queues for Burst Absorption

Message queues (Kafka, SQS) absorb traffic spikes by acting as a buffer between producers and consumers. Instead of writing directly to the database, producers write to the queue. Consumers drain the queue at the database's sustainable rate.

**Critical constraint:** Queues handle temporary spikes, not permanent excess capacity. If writes arrive at 5,000/sec and your database handles 2,000/sec, a queue buys you time to scale — but the queue will grow without bound until either traffic drops or you add capacity. Using a queue to paper over a sustained capacity gap is a ticking clock.

**Trade-off:** Async writes mean clients don't get immediate confirmation. You need a callback mechanism — webhooks, polling, or a separate notification channel — for clients that need to know when their write completed.

### Load Shedding

When the system is overloaded, intentionally drop low-value writes rather than letting the whole system degrade or crash. Define a priority hierarchy based on business value:

**Example — activity tracking system:**
- **Never drop:** account charges, purchase confirmations (high business value, user-visible)
- **Drop last resort:** click events, session tracking (medium value, recoverable)
- **Drop first:** impression logging, background analytics (low value, approximate is fine)

The system detects overload (queue depth crossing a threshold, latency rising, error rate climbing) and starts rejecting lowest-priority writes first. This keeps the high-value write path healthy even as the system is saturated.

**Real-world pattern:** A delivery tracking service may drop stale location pings (if a driver's position was already logged 2 seconds ago, a duplicate ping from 1 second ago has negative marginal value) while preserving all status change events.

---

## Strategy 4: Batching and Hierarchical Aggregation

### Batching

Instead of writing every event individually, accumulate a window of events and write them together. One write per batch instead of one write per event.

**Three layers where batching can happen:**

**Application layer:** The service holds a buffer of writes in memory and flushes periodically or when the buffer fills.
```
Incoming: write1, write2, write3, write4, write5
Buffered:  [write1...write5]
After 100ms: one INSERT of 5 rows
```

**Intermediate processor:** A dedicated service aggregates multiple events into a single summarized write. A "Like Batcher" receives individual like events for a post, counts them over a 1-minute window, and issues a single `UPDATE posts SET like_count = like_count + 342`. One database write instead of 342.

**Database layer:** Redis, by default, holds writes in memory and flushes to disk every 100ms. ClickHouse and Cassandra also batch internal writes before persisting. These defaults can be tuned for your durability vs. throughput needs.

### Hierarchical Aggregation

For systems where one source generates updates that must reach many consumers (live comments during a streamed event, live scores during a sports match), naive fan-out creates O(viewers) writes for every event.

Hierarchical aggregation introduces intermediate processors that aggregate events from multiple sources before broadcasting to the next layer:

```
Write Processors (receive raw events)
        ↓  (aggregate)
Root Processor (single aggregation point)
        ↓  (broadcast)
Broadcast Nodes (each serve a subset of clients)
        ↓
End Clients
```

A root processor collecting 1,000 likes/second batches them into one summary per second and forwards that summary to broadcast nodes. Each broadcast node fans out to its assigned client subset. This flattens the write spike at every level.

---

## Deep Dives

### Hot Key Problem

A single key receives a disproportionate share of writes (a viral post getting millions of likes simultaneously). No sharding scheme prevents this because all writes legitimately belong on the same key.

**Key splitting:** Distribute writes for `postLikes:viral_post` across N sub-keys: `postLikes:viral_post:0` through `postLikes:viral_post:N-1`. Writers hash into a random sub-key. Readers must query all N sub-keys and sum the results.

**Dynamic splitting:** Writers detect that a key is hot and announce the split to readers before it happens. Readers then check all sub-keys on every read. Alternatively: readers always check all sub-keys as a default — slightly more read work, but no coordination protocol needed.

### Resharding in Production

Adding shards while the system is live requires careful coordination:
1. Decide the new shard layout and announce it to all services
2. Enter dual-write mode: all new writes go to both old and new shard
3. Background job migrates existing data to the new shard layout
4. Verify migration is complete and consistent
5. Switch reads to the new layout; stop dual-writes

The dual-write period ensures no writes are lost during migration. The verification step ensures you don't switch reads before old data has been migrated.

---

## When to Use Each Strategy

- **Vertical + DB choice:** First stop. Confirm you've actually hit the ceiling before adding complexity.
- **Horizontal sharding:** When storage (>50TB) or write rate (>10k/sec) exceeds what one machine can handle. Choose shard key to minimize variance in writes per shard.
- **Vertical partitioning:** When different parts of the same entity have wildly different write frequencies; split them so hot columns don't block cold ones.
- **Queues:** Burst absorption only — temporary traffic spikes that exceed sustained database capacity. Not a substitute for capacity.
- **Load shedding:** When the system must protect its highest-value write path during overload. Requires explicit priority ranking of write types.
- **Batching:** When individual events are low-value enough to aggregate safely. Like counts, view counts, impression logs — all tolerate slight delays.
- **Hierarchical aggregation:** When fan-out is the bottleneck — one event must propagate to millions of consumers.

## Common Gotchas

- **Using queues as a permanent capacity fix** — a queue is a delay, not extra capacity; the write still has to happen eventually
- **Time-based shard keys** — all current writes hit the current time bucket; use entity IDs instead
- **Assuming sharding is always the answer** — run the numbers first; PostgreSQL handles 10k-20k writes/sec on a single well-tuned instance
- **Uniform batching windows** — a fixed 1-second batch window means some writes see up to 1 second of delay; for time-sensitive writes, use adaptive windows or don't batch at all
- **Not planning for hot keys** — shard by user_id, but if one user has 10 million followers posting simultaneously, you still get a hot shard; plan for the p99 case, not just the average

## Practice Questions

1. A social media platform is hitting 8,000 like events per second. Each like is currently a separate `UPDATE posts SET like_count = like_count + 1`. Walk through two strategies to reduce database write load. What are the trade-offs of each?
2. You have a Cassandra cluster with 6 nodes and need to double capacity. Walk through a resharding plan that doesn't require downtime and doesn't lose any writes.
3. A key-value store shows one cache key receiving 50× more writes than any other. Describe the hot-key splitting approach, including both the write and read paths after the split.
4. Design the write path for a real-time leaderboard that must rank 10 million players. How do you handle the situation where 100,000 score updates arrive simultaneously?
5. Your analytics pipeline drops writes during traffic spikes. Walk through a load shedding implementation: how do you detect overload, define write priorities, and decide what to drop?

## One-Line Summary

Scale writes by reducing throughput per component: try vertical scaling and LSM-tree databases first, then shard horizontally by a high-cardinality key, absorb spikes with queues, protect critical paths with load shedding, and aggregate bursty events with batching and hierarchical fan-out.
