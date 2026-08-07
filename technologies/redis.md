# Redis

## The Core Idea

Redis is an in-memory data structure store that executes operations in microseconds. It's single-threaded (one command at a time, no lock contention) and keeps everything in RAM. The tradeoff is deliberate: it's blazingly fast, but it's not your system of record — acknowledged writes can vanish on a crash. Use Redis where speed matters more than durability.

## Mental Model / Analogy

Think of a chef's mise en place — all ingredients pre-measured and arranged in small bowls directly on the counter before service starts. Reaching into a bowl is instant; walking to the pantry (disk) takes seconds. Redis is your counter-top: the working memory for your distributed system. You put the things you need right now where you can grab them fast, not everything you've ever stored.

The insight: mise en place only holds what fits on the counter. Redis only holds what fits in RAM. Design accordingly.

---

## The Basics

### Data Model

Every Redis value lives at a string key. The value's type determines what operations you can run on it:

| Type | Fits when... | Example use |
|---|---|---|
| String | Simple key-value, counters | Rate limit counter, session token |
| Hash | Object/dictionary with fields | User profile `{name, email, tier}` |
| List | Ordered sequence | Message history, activity feed |
| Set | Unique membership | Users who've seen a notification |
| Sorted Set | Ranked members with scores | Leaderboard, sliding-window rate limit |
| Stream | Append-only log with consumer groups | Event sourcing, work queue |
| Geospatial | Lat/lng indexed by geohash | Proximity search (nearby drivers) |

### Infrastructure Modes

```
Single node:    [Redis]                 – dev/test

Replicated:     [Primary] → [Secondary] – HA, async replication
                                          (failover via Sentinel)

Cluster:        [P0|S0] [P1|S1] [P2|S2] – horizontal scale
                  keys hash to one of 16,384 slots
                  each slot owned by one primary
```

**Critical replication fact:** Redis replication is **asynchronous**. The primary acknowledges your write before the replica sees it. If the primary crashes and a replica gets promoted, the last few acknowledged writes may be gone. Redis is not a system of record.

**Cluster key routing:** Clients cache the slot→node map locally. Hash a key to get its slot; go directly to the right node. If the slot moved (rebalancing/failover), the node returns a `MOVED` redirect and the client updates its map.

**Hash tags:** When two keys must live on the same node (e.g., for a `MULTI` transaction), wrap the meaningful part in `{braces}`: `{user:123}:posts` and `{user:123}:likes` hash on `user:123` and land on the same slot.

---

## Capabilities

### Cache

The most common Redis role. Pattern: check Redis first; on miss, load from DB and cache with a TTL.

```
SET product:123 <json_blob> EX 300  # expires in 5 minutes
GET product:123                      # returns nil if expired
```

Configure `maxmemory-policy allkeys-lru` so Redis evicts the least-recently-used keys when memory fills — don't let it reject writes.

**Hot key problem**: a single wildly popular key hammers one node. Solutions:
- **Client-side cache**: each app server caches the hot key locally (tolerate short stale window)
- **Key copies**: store the same value at `product:123:1` through `product:123:10`; readers pick one at random; writers fan out to all copies
- **Read replicas**: configure clients to read from replicas (only helps for read-hot keys, not write-hot)

### Distributed Lock

Use Redis to hold exclusivity across multiple app servers. The idiom:

```
# Acquire: set if not exists, with TTL
SET lock:resource:42 my-unique-token NX EX 30

# Release: check your token, then delete — atomically via Lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
```

`NX` = only set if key doesn't exist. `EX 30` = auto-expire after 30 seconds so a crashed holder releases the lock. The unique token ensures you only delete your own lock, not someone else's that acquired it after yours expired.

**The Redlock problem:** Single-node locks can be granted twice if the primary crashes right after granting and before the replica replicates. Redlock (majority quorum across independent nodes) reduces this but doesn't eliminate it. Practical rule: treat Redis locks as a performance optimization, not a correctness guarantee. For true correctness, enforce the invariant at the database layer too.

### Leaderboard

Sorted sets maintain ordered members with O(log N) insert/update:

```
# Score = number of points; member = user ID
ZADD game:leaderboard 9400 "player_77"
ZADD game:leaderboard 8200 "player_12"
ZREVRANK game:leaderboard "player_77"  # → 0 (top rank)
ZRANGE game:leaderboard 0 9 REV WITHSCORES  # top 10
```

### Rate Limiting

**Fixed window** — simplest:
```
# One Lua script to keep INCR + EXPIRE atomic:
# 1. INCR counter:user:42:window:1722470400
# 2. If count == 1: EXPIRE key 60  (only set expiry on first request)
# 3. If count > limit: reject
```

Why a Lua script? If you `INCR` then `EXPIRE` separately and the process crashes between them, the counter never expires and the user gets permanently rate-limited.

**Sliding window** — more accurate:
```
# Sorted set per user, score = timestamp, member = request ID
ZREMRANGEBYSCORE user:42:requests 0 (now - window_ms)  # drop old
ZADD user:42:requests now request_id
ZCARD user:42:requests  # count in window
```

### Proximity Search

```
GEOADD active-drivers -122.4194 37.7749 "driver:88"  # lng lat member
GEOSEARCH active-drivers FROMLONLAT -122.41 37.78 BYRADIUS 5 km  # search
```

Under the hood: geohash encodes lat/lng into a single integer stored in a sorted set. The search is O(N + log M) where N is candidates in the bounding box, M is those within the exact radius.

### Pub/Sub

Real-time fire-and-forget messaging:
```
PUBLISH notifications:user:42 '{"type":"like"}'  # publisher
SUBSCRIBE notifications:user:42                   # subscriber
```

Each subscriber holds one connection per node (not per channel). Messages are **not persisted**: a subscriber that's offline misses messages entirely. For delivery guarantees, use Redis Streams or Kafka instead.

Since Redis 7, use sharded Pub/Sub (`SPUBLISH`/`SSUBSCRIBE`) in cluster mode — routes channels to slots so capacity scales with the cluster.

### Event Sourcing / Work Queue (Streams)

Redis Streams are Kafka-lite: append-only log with consumer groups, offset tracking, and claim-based failure recovery.

```
XADD jobs * type encode video_id v99     # producer adds item
XREADGROUP GROUP workers consumer1 ... STREAMS jobs >  # consume
XACK jobs workers <message-id>           # acknowledge
XCLAIM jobs workers consumer2 <old-id>  # claim timed-out item
```

**Streams vs Kafka:** Use Streams when Redis is already in your design and queue volume is modest. Upgrade to Kafka when you need long retention, replay for many independent consumers, or throughput at a scale where losing messages is unacceptable.

---

## Key Trade-offs

| What you gain | What you lose |
|---|---|
| Sub-millisecond reads/writes | No joins, no cross-key queries |
| Rich data structures fit naturally | Data size limited to RAM |
| Dead-simple API, easy to reason about | Async replication; acknowledged writes can vanish |
| 100k+ writes/sec per node | No durable, replayable streams at Kafka scale |

---

## When NOT to Use Redis

- **System of record**: use a database with synchronous disk writes (PostgreSQL, MySQL)
- **Data > RAM budget**: memory is expensive; large datasets belong on disk
- **Complex queries**: no joins, no aggregations across keys, no ad-hoc filters
- **Durable fan-out streams**: Kafka with `acks=all` and long retention is the right tool

---

## Practice Questions

1. You're building a "daily active users" counter that must be accurate to within ±1%. Walk through how you'd implement it in Redis, and explain the tradeoff between a simple counter and a HyperLogLog.
2. Your leaderboard has 50 million players. Describe the full data model and the operations needed to show any player their rank and the top 100 players globally.
3. A user hits your API 10,000 times in 60 seconds; your limit is 100/minute. Explain both a fixed-window and a sliding-window implementation in Redis, and why a plain `INCR` + `EXPIRE` without a Lua script is buggy.
4. You're using Redis for distributed locks in a payment system. A network partition causes the primary to die just after granting a lock. Explain the failure mode and how you'd defend against it at the application layer.
5. Describe the hot key problem in a Redis cluster and explain three different mitigation strategies, noting which works for read-hot keys vs write-hot keys.

---

## One-Line Summary

Redis is an in-memory data structure store — blazingly fast, surprisingly versatile (cache, locks, leaderboards, rate limits, pub/sub, geo search) — but it is not your system of record, and everything must fit in RAM.
