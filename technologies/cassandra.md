# Cassandra

## The Core Idea

Cassandra is an open-source, distributed wide-column database originally built at Facebook for inbox search. It handles massive write throughput and horizontal scaling across commodity hardware, trading away ACID transactions and ad-hoc query flexibility in exchange. The whole system is built to be highly available — no single point of failure, and you dial the consistency/availability tradeoff at query time rather than at design time.

## Mental Model / Analogy

Picture a chain of post offices distributed across a city, each holding copies of certain zip codes' mail directories. When you walk into any branch (query coordinator), they can look up what you need — either from their own copy or by routing the question to the right branch. If a branch temporarily closes (node down), the neighboring branches hold on to new mail addressed there (hinted handoff) and deliver it when the branch reopens.

The directories themselves are organized like a filing system: the city/zip code (partition key) determines which physical cabinet you open, and within that cabinet, records are sorted alphabetically (clustering key). You model your filing system around how people ask questions — if staff always ask "show me all mail for zip 98101 sorted by date," that's exactly how you arrange the cabinet.

---

## Data Model

Cassandra's hierarchy: **Keyspace → Table → Row → Column**

- **Keyspace**: top-level container, like a database in PostgreSQL. Defines the replication strategy
- **Table**: schema-defined; has columns with types; rows can have different subsets of columns (wide-column)
- **Row**: identified by its primary key; atomic writes at the row level within a partition
- **Column**: typed value with a name. Every column secretly carries a write timestamp — conflict resolution is "last write wins" by timestamp

The wide-column design means rows don't need to fill every column — a row might have 3 attributes, the next might have 12. Think of it like a JSON object with a defined schema but optional fields.

```
# Think of the data model like a nested JSON
{
  "keyspace:chat": {
    "table:messages": {
      "row(chat_id=1, msg_id=100)": { sender: "alice", text: "hi" },
      "row(chat_id=1, msg_id=101)": { sender: "bob", text: "hey", reaction: "👋" }
    }
  }
}
```

---

## Primary Key

The most critical design decision in Cassandra:

**Partition Key** — one or more columns that determine which node(s) store the row. Cassandra hashes this value to a position on the consistent hash ring. High-cardinality, evenly distributed keys prevent hot partitions.

**Clustering Key** — zero or more columns that define sort order within a partition. Data is sorted by clustering key in a B-tree-like structure on disk, enabling efficient range scans.

```sql
-- Partition key: channel_id; clustering key: message_id (sorted DESC)
CREATE TABLE messages (
  channel_id bigint,
  message_id bigint,
  author_id  bigint,
  content    text,
  PRIMARY KEY (channel_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Composite partition key (two columns together form the partition key)
CREATE TABLE messages_v2 (
  channel_id bigint,
  bucket     int,
  message_id bigint,
  content    text,
  PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

**Important:** The primary key concept maps almost 1:1 with DynamoDB (partition key = partition key, clustering key = sort key).

---

## How It Works

### Partitioning: Consistent Hashing

Cassandra distributes data using a consistent hashing ring. Each node "owns" a range of hash values (tokens). When a row arrives, Cassandra hashes the partition key to get a token, walks the ring clockwise, and writes to the first node whose token range covers that value.

**The problem with naive hashing:** `hash(key) % num_nodes` forces massive data movement when you add/remove a node (affects all keys). Consistent hashing limits reshuffling to only the adjacent segment.

**Virtual nodes (vnodes):** To prevent uneven load distribution, each physical node owns multiple small positions on the ring rather than one large segment. This also lets larger machines claim more positions.

### Replication

Each partition is replicated to N nodes (configured per keyspace):

```sql
ALTER KEYSPACE my_app WITH REPLICATION = {
    'class': 'NetworkTopologyStrategy',
    'datacenter1': 3,   -- 3 replicas in DC1
    'datacenter2': 2    -- 2 replicas in DC2
};
```

- **NetworkTopologyStrategy** (production): rack-aware and DC-aware; places replicas on distinct racks/DCs so a rack failure or datacenter outage doesn't take down all replicas
- **SimpleStrategy** (testing only): scans clockwise for N replicas without rack awareness

### Consistency Levels

Cassandra lets you choose consistency **per-operation**, not globally:

| Level | Meaning | Tradeoff |
|---|---|---|
| ONE | First replica to respond | Fastest, weakest |
| QUORUM | n/2 + 1 replicas must respond | Strong enough when used for both reads and writes |
| LOCAL_QUORUM | Quorum within your local DC | Good for multi-DC without cross-DC latency |
| ALL | Every replica must respond | Strongest, but dies if any replica is down |

**The magic of QUORUM:** With 3 replicas, QUORUM = 2. Write to 2, read from 2 → at least 1 node guaranteed to have seen the latest write. This makes reads always see the most recent committed write.

**No ACID transactions:** Cassandra supports atomic writes at the partition level only (all columns of a row written together). No cross-row transactions. No joins. No referential integrity.

### Query Routing

Any node in the cluster can act as a **coordinator** for a query. The coordinator:
1. Hashes the partition key to find the responsible nodes
2. Forwards the query to those replica nodes
3. Collects responses according to the chosen consistency level
4. Returns the merged result to the client

Nodes know the cluster topology via **gossip** — a peer-to-peer protocol where each node periodically exchanges state with a few others (biased toward "seed" nodes). This eliminates any central metadata server.

### Storage: LSM Tree

Cassandra's write path is why it dominates write-heavy workloads:

```
Write arrives
    ↓
1. Commit Log (write-ahead log on disk — for crash recovery)
    ↓
2. Memtable (in-memory sorted structure — accumulates writes)
    ↓
3. [when Memtable fills] Flush to SSTable (immutable sorted file on disk)
    ↓
4. Commit Log entries for flushed data deleted
```

**Why this is fast:** Step 1 (commit log) is a sequential append — the fastest disk operation. Steps 2-3 are in-memory. Compare to B-tree databases (PostgreSQL, MySQL) which must update random disk pages for every write.

**Reading with LSM:** Cassandra checks the Memtable first (latest data). Then uses **bloom filters** (probabilistic — quickly eliminates SSTables that definitely don't have the key). Then reads SSTables from newest to oldest, stopping when it finds the key.

**Compaction:** Over time, SSTables accumulate and reads get slower (must check more files). Compaction merges SSTables, removes tombstones (deleted rows), and consolidates updates. It also reclaims disk space from old versions of rows.

### Gossip and Failure Detection

Every ~1 second, each node sends its generation/version counters to a few random peers (biased toward seed nodes). This distributes cluster health state throughout the cluster with no single point of failure.

Cassandra uses the **Phi Accrual Failure Detector**: rather than a binary alive/dead decision, it computes a continuous "suspicion score" based on heartbeat timing. A node is "convicted" (writes paused to it) when its score exceeds a threshold, but it's never permanently removed unless an administrator does so — preventing premature rebalancing on temporary network issues.

**Hinted handoffs:** If a replica is temporarily down when a write arrives, the coordinator stores the write as a "hint" and delivers it when the replica comes back online. This maintains write availability without sacrificing data integrity.

---

## Query-Driven Data Modeling

Cassandra's query efficiency is completely tied to its storage layout. It doesn't support JOIN across tables, can't filter efficiently on non-key columns without allowing filtering (dangerous at scale), and every query must specify the partition key.

The design philosophy: model your tables around your application's access patterns, not around your entities.

**4 questions to ask when designing a Cassandra schema:**
1. **Partition key** — what data determines the partition? High cardinality and even distribution
2. **Partition size** — will this partition grow unboundedly? Cap it
3. **Clustering key** — in what order should data be sorted within a partition?
4. **Denormalization** — which data must be duplicated across tables to serve different access patterns?

### Example: Discord Messages

Discord uses Cassandra for message storage. Their schema evolved through a real-world scaling challenge:

```sql
-- V1: Simple schema
CREATE TABLE messages (
  channel_id bigint,
  message_id bigint,   -- Snowflake ID (chronologically sortable UUID)
  author_id  bigint,
  content    text,
  PRIMARY KEY (channel_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

**Problem:** Busy Discord channels (millions of messages) created huge partitions → performance degraded. Cassandra doesn't handle unboundedly large partitions well.

```sql
-- V2: Add time bucketing to bound partition size
-- bucket = floor((message_id - DISCORD_EPOCH) / 10_days_in_ms)
CREATE TABLE messages (
  channel_id bigint,
  bucket     int,       -- 10-day window
  message_id bigint,
  author_id  bigint,
  content    text,
  PRIMARY KEY ((channel_id, bucket), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);
```

Recent messages live in one bucket (single partition query). Scrolling back = query 1-2 adjacent buckets. Partition size is now bounded regardless of channel activity.

**Why message_id instead of created_at?** Snowflake IDs are monotonically increasing UUIDs — no collision possible. A millisecond timestamp can have duplicates if two messages arrive at the same instant, and primary key conflicts in Cassandra silently overwrite the previous row.

---

## Key Trade-offs

| What you gain | What you lose |
|---|---|
| Massive write throughput (LSM tree) | No JOINs; no ad-hoc queries |
| Horizontal scaling with no single master | No ACID transactions (only row-level atomicity) |
| Tunable consistency per operation | Must model schema around access patterns |
| High availability (no SPOF) | Eventual consistency by default; QUORUM adds latency |
| Multi-datacenter replication built-in | Operational complexity of cluster management |

---

## When to Use Cassandra

**Strong fit:**
- High-volume time-series data (IoT sensor readings, activity logs, analytics events)
- Chat/messaging at scale (Discord pattern — time-bucketed, single-partition reads)
- Leaderboards and counters (if using counter columns)
- Any workload where you can define access patterns upfront, need high availability, and write throughput matters
- Multi-region active-active deployments (NetworkTopologyStrategy + LOCAL_QUORUM per DC)

**Poor fit:**
- Complex queries with JOINs or aggregations → PostgreSQL
- ACID transactions across multiple rows/tables → PostgreSQL or DynamoDB
- Ad-hoc analytics or reporting → dedicated analytics DB (Redshift, BigQuery, ClickHouse)
- Small datasets or simple use cases → the operational overhead isn't worth it

**Cassandra vs DynamoDB:** If you're AWS-only and want zero ops burden, use DynamoDB. If you want open-source, multi-cloud, or more control over replication topology, use Cassandra.

---

## Common Gotchas

- **Don't filter on non-key columns** — queries that filter on columns outside the primary key require `ALLOW FILTERING`, which triggers a full-partition scan. Design your keys to match your queries
- **Unbounded partition growth** — a partition key that accumulates data forever (user_id for messages) will eventually hit performance limits; add a time bucket to bound it
- **Last write wins** — concurrent updates to the same column resolve by timestamp; clock skew between nodes can cause surprising behavior
- **Tombstones** — deleted data leaves tombstones that must be read through on every query until compaction runs; heavy deletes can severely hurt read performance
- **QUORUM is not free** — higher consistency = more cross-replica coordination = more latency; don't use QUORUM unless you actually need it

---

## Practice Questions

1. You're building a Twitter-like timeline where users can see all tweets from accounts they follow, sorted by time. How would you model this in Cassandra? What are the tradeoffs of denormalizing vs fan-out on write?
2. Discord's original messages table had `channel_id` as the partition key. Why did this become a problem for busy channels, and how did the bucket column solve it?
3. You have a Cassandra table for ride history: users request "show me my last 50 rides, sorted by date." Design the primary key. What happens if a user takes 10 million rides?
4. Explain the difference between Cassandra's consistency level QUORUM and QUORUM read + QUORUM write. Why does this combination guarantee that reads always see the latest write?
5. A product manager wants to build a report that aggregates ride data by city, filtered by driver rating, sorted by revenue. Explain why this is problematic for Cassandra and what you'd use instead.

---

## One-Line Summary

Cassandra is a distributed wide-column database built for write-heavy workloads at massive scale — design your partition key and clustering key around your exact access patterns, cap your partition sizes, and tune consistency per-operation.
