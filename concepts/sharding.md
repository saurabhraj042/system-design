# Sharding

## The Core Idea

Sharding splits your data across multiple database machines so each machine holds only a subset of the total dataset. A single machine can't store or serve all your data fast enough, so you divide it into shards — each shard is a complete, independent database that owns its slice of the data.

The hard part isn't splitting the data; it's choosing how to split it so related data stays together, load distributes evenly, and your most common queries don't require talking to every shard.

## Mental Model / Analogy

Imagine a company's customer support team expanding from one city to multiple regional offices. To handle growth, they split customers alphabetically — the New York office handles A-M, the Austin office handles N-Z. A customer calling in goes directly to their regional office, which has all their history locally. But if the CEO asks "how many customers do we have company-wide?", someone has to call all offices and add up the numbers. That cross-office call is a cross-shard query, and it's expensive.

The shard key (last name initial in this analogy) determines which office owns which customer. A bad shard key creates a hot office — if most customers happen to have last names starting with A-M, New York drowns while Austin sits idle.

---

## Partitioning First

Before sharding (multiple machines), understand partitioning (one machine, multiple logical groups):

**Horizontal partitioning** — split by rows. Table with 100M users becomes two partitions: users 1-50M on one partition, 50M-100M on another. This is the model sharding scales out to multiple machines.

**Vertical partitioning** — split by columns. The `posts` table becomes `posts_core` (id, content, user_id) and `posts_media` (id, media_urls, thumbnail). Separates hot frequently-queried columns from cold ones.

---

## Shard Key Selection

The shard key is the column (or column combination) whose value determines which shard a record goes to. Getting this right is the most important decision in sharding.

**Good shard key criteria:**
- **High cardinality** — enough distinct values to spread data across all shards (user_id: good, country: maybe too few, boolean: terrible)
- **Even distribution** — values spread equally, no natural skew (sequential IDs: good, zip code: depends on your geography)
- **Query alignment** — your most common queries should target a single shard (if you mostly query "posts by user," shard by user_id)

**Examples:**

| Shard key | Assessment |
|-----------|------------|
| `user_id` | Usually good — high cardinality, even (assuming random UUIDs/IDs) |
| `order_id` | Good for order lookups, bad if you need orders-by-user |
| `is_premium` | Terrible — only 2 values, all non-premium traffic hammers one shard |
| `created_at` | Bad for writes — all new records go to the latest time-bucket shard |
| `tenant_id` (SaaS) | Good for multi-tenant — each tenant's data is self-contained |

---

## Sharding Strategies

### Hash-Based Sharding — Default Choice

Apply a hash function to the shard key and use modulo to pick the shard: `shard = hash(user_id) % num_shards`.

**Why it's the default:** Even distribution regardless of what the data looks like. If you have 64 shards and hash distributes uniformly, each shard gets ~1/64 of the data.

**The problem:** Adding or removing shards requires rehashing almost all records. If you go from 64 shards to 65, `hash(key) % 64` and `hash(key) % 65` route most keys to different shards. This is why hash-based sharding is often paired with **consistent hashing** (see the Consistent Hashing notes) — consistent hashing limits data movement when the shard count changes.

### Range-Based Sharding

Assign ranges of key values to shards: shard 1 handles user_ids 1-10M, shard 2 handles 10M-20M, etc.

**Advantage:** Efficient range scans. All posts from a given week are on the same shard. Good for multi-tenant SaaS where you assign tenant ranges.

**Problem:** Access patterns are rarely uniform across ranges. Time-series data with range sharding by timestamp means all writes go to the "current time" shard — a guaranteed hot shard.

### Directory-Based Sharding

Maintain a lookup table that maps shard keys to shards: `{user_id: 1234 → shard_7}`. Any key can be pointed anywhere, moved at will.

**Advantage:** Maximum flexibility. Move a hot user's data to a dedicated shard. Rebalance any individual record without a full resharding operation.

**Problem:** Every database operation requires first consulting the directory — an additional network hop. The directory itself becomes a single point of failure and performance bottleneck. Rarely the right answer in interviews unless the flexibility is a hard requirement.

---

## Sharding Challenges

### Hot Spots (Celebrity Problem)

Even with a good shard key, specific values receive wildly disproportionate traffic. Taylor Swift has 200M followers — any write to her account (new post, bio update) creates a flood of fan activity hitting her user_id's shard.

**Fixes:**
- **Dedicated shard for hot keys:** Move high-traffic entities to their own isolated shard
- **Compound shard keys:** `hash(user_id + date)` spreads one user's traffic across multiple shards over time
- **Application-level fan-out:** Pre-compute and cache results for hot entities, reducing direct shard queries

### Cross-Shard Operations

"Get the top 10 most popular posts globally" requires querying all 64 shards, getting each shard's top 10, merging 640 results, and re-ranking. This scatter-gather pattern is expensive and slow.

**Fixes:**
- **Denormalize:** Maintain a global leaderboard separately in Redis or a single aggregation table that's updated incrementally
- **Accept the hit:** For rare admin-style queries, cross-shard is fine; optimize for the common case
- **Cache results:** Pre-compute global aggregations periodically (top posts of the hour), serve from cache

### Cross-Shard Consistency

A payment that debits one user's shard and credits another user's shard is a distributed transaction. Two-Phase Commit (2PC) makes this atomic but is slow and fragile — if any shard goes down mid-transaction, the whole thing blocks.

**Fixes:**
- **Design to avoid cross-shard transactions:** Keep related data on the same shard. If transfers always go between user_id → user_id, and both users' data is on different shards, you have a problem. Some designs put money flows in a separate ledger table with its own shard key
- **Saga pattern:** Break the transaction into compensating steps. Debit shard A → if credit to shard B fails → issue a compensating debit to undo. Accept that briefly the system is inconsistent
- **Accept eventual consistency:** Follower count discrepancies of a few seconds are fine; financial transaction atomicity is not optional

---

## Modern Database Sharding

Most mature databases handle sharding automatically:

- **Cassandra:** Consistent hash ring with virtual nodes (vnodes). Each physical node owns multiple small segments. On failure, load redistributes evenly across the ring
- **DynamoDB:** AWS-managed hash-based partitioning. Auto-splits partitions when they hit throughput or storage limits. You never manually shard — it just scales
- **MongoDB:** Range-based chunks with a background balancer. The mongos router distributes queries; the config servers track chunk ownership
- **Vitess / Citus:** SQL sharding layers on top of MySQL and PostgreSQL respectively. Use when you want SQL semantics with horizontal scale

---

## Interview Formula

When sharding comes up in an interview:

1. **Establish that sharding is actually needed first** — a single PostgreSQL instance handles 64 TiB and 50k reads/sec. Don't shard until you've proven storage >50 TiB or consistent writes >10k per second
2. **Propose a shard key** and explain why: high cardinality, even distribution, query alignment
3. **Choose a distribution strategy** (hash-based is the default; range-based when range scans matter)
4. **Call out the hot spot risk** for your key choice and your mitigation
5. **Address cross-shard queries** — mention scatter-gather cost and your plan (caching, denormalization, accepting the hit)

---

## Key Trade-offs

| What you gain | What you lose |
|---------------|---------------|
| Horizontal scaling beyond one machine | Cross-shard queries become expensive |
| Storage capacity scales linearly | Distributed transaction complexity |
| Write throughput scales with shard count | Hot shard risk if key skews |
| Fault isolation (one shard failure ≠ total outage) | Operational complexity multiplies by shard count |

## When to Use

- Dataset exceeds ~50 TiB (single-machine storage ceiling)
- Write throughput consistently above ~10k writes/second
- Read throughput exceeds what read replicas can handle
- You have well-defined access patterns that align with a good shard key

## Common Gotchas

- **Premature sharding:** 100M users × 1KB = ~100 GB — easily fits in one database with room to spare. Always size first
- **Wrong shard key is hard to fix:** Resharding is painful. Think carefully upfront — the shard key choice is semi-permanent
- **Time-based shard keys almost always create hot spots:** All new writes go to the current period's shard
- **Cross-shard JOINs don't work:** Your ORM probably allows them but they'll degrade to full scatter-gather at the database layer

## Practice Questions

1. You're designing a URL shortener handling 1 billion shortened URLs. Each URL record is 500 bytes. Walk through why you might or might not shard, and if you shard, what your shard key would be.
2. A social media platform shards posts by `user_id`. Kylie Jenner posts to her 400M followers and her shard gets hammered with read requests. Propose two solutions.
3. Your sharded database has 8 shards by user_id. You get a requirement: "show me all users who signed up in the last 7 days." Walk through how you'd handle this query efficiently.
4. Explain the difference between range-based and hash-based sharding. Give a concrete example where each is the right choice.
5. You've sharded your orders database by user_id, but now you need to generate a daily revenue report across all users. How do you approach this without running a costly scatter-gather query every time?

## One-Line Summary

Sharding distributes data across multiple machines using a shard key — choose a key with high cardinality and even distribution that aligns with your most common queries, default to hash-based distribution, and be prepared for the complexities of hot spots and cross-shard operations.
