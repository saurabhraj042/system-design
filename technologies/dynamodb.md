# DynamoDB

## The Core Idea

DynamoDB is AWS's fully-managed NoSQL database: no servers to patch, no capacity planning beyond choosing your throughput mode, automatic scaling and replication. The catch is that it forces you to define your access patterns upfront — the partition key you choose at table-creation time determines where data lives and what queries are fast. Get that right and it handles just about anything; get it wrong and you're stuck with full-table scans.

## Mental Model / Analogy

Picture a hospital's physical records room with filing cabinets — one drawer per patient (partition key = patient ID). Within each drawer, records are sorted chronologically (sort key = visit date). Finding patient 7734's records from last March is instant: go to drawer 7734, flip to March. But if you need "all patients who saw Dr. Chen last Tuesday," you'd have to open every drawer and check — that's a scan, and it's slow.

Secondary indexes are like maintaining cross-reference binders: a GSI might be sorted by doctor name across all patients (different physical storage, separate binder room), while an LSI might sort each patient's records by diagnosis code instead of date (same drawer, different tab).

---

## Data Model

DynamoDB organizes data as: **Tables → Items → Attributes**

- **Table**: top-level container, defined by a primary key
- **Item**: a row; each item up to 400KB; no fixed schema (different items can have different attributes)
- **Attribute**: key-value pair; scalar types (String, Number, Boolean) or nested (Maps, Lists)

DynamoDB is **schema-less**: you define only the primary key at table creation. Any item can have any additional attributes. This flexibility requires you to enforce structure at the application layer.

---

## Keys

### Partition Key (Required)

DynamoDB hashes the partition key to determine which physical storage node holds the item. Items with the same partition key are stored together (on the same partition). Choosing a key with high cardinality and even distribution is critical — a "hot" partition key (like `status: 'active'`) concentrates writes on one node and throttles you.

### Sort Key (Optional)

When combined with the partition key, forms a composite primary key. Items sharing a partition key are ordered by sort key using a B-tree within the partition. This enables efficient range queries like "give me all messages in chat_42 with message_id between 1000 and 2000."

**Best practice for sort keys:** Use a monotonically increasing ID (UUID v7, ULID, Snowflake) rather than a timestamp alone. Timestamps don't guarantee uniqueness — two messages can arrive in the same millisecond.

```
Table: messages
Partition key: chat_id      → routes to physical partition
Sort key: message_id        → ordered within that partition
```

### What's happening under the hood?

- **Partition key → hash → node**: a centralized partition metadata service maps hash values to storage nodes (similar concept to consistent hashing, but DynamoDB uses a central map, not a peer-to-peer ring)
- **Sort key → B-tree**: within a partition, items are stored in a B-tree indexed by sort key, enabling O(log n) range queries

---

## Secondary Indexes

When you need to query by something other than the partition key:

| Feature | GSI (Global Secondary Index) | LSI (Local Secondary Index) |
|---|---|---|
| Partition key | Different from base table | Same as base table |
| Sort key | Any attribute (optional) | Different from base table sort key |
| Physical storage | Separate partitions, replicated independently | Co-located with base table |
| Consistency | Eventually consistent only | Strongly consistent reads supported |
| Throughput | Separate RCU/WCU from base table | Shares base table throughput |
| Size limit | None | 10GB per partition key |
| When created | Any time; can delete and recreate | At table creation only; permanent |
| Max per table | 20 GSIs | 5 LSIs |

**When to add a GSI:** You need to query by an attribute that isn't the partition key. Example: messages table partitioned by `chat_id`, but you need to find all messages by a specific `user_id` across all chats → GSI on `user_id`.

**When to add an LSI:** You need to sort items within a partition by a different attribute. Example: messages partitioned by `chat_id`, sort key is `message_id`, but you also want to find messages with the most attachments within a chat → LSI on `num_attachments`. Must be defined at table creation.

---

## Reading Data

Two operations:

**Query** — efficient, uses partition key (and optionally filters on sort key). DynamoDB reads only matching items. Always prefer this.

**Scan** — reads every item in the table/index and filters client-side. Expensive. Avoid for anything in production at scale; the cost is proportional to table size, not result size.

**Important billing detail:** Even with `ProjectionExpression` (select specific attributes), you're billed for the full item read from storage. Normalize large attributes into separate items if you frequently need to read small slices of big items.

---

## Consistency Models

Chosen **per-request**, not per table:

| Mode | How | Latency | Cost | Available on GSIs? |
|---|---|---|---|---|
| Eventually consistent (default) | Any of 3 replicas serves the read | Lower | 0.5 RCU per 4KB | Yes |
| Strongly consistent | Routes to partition leader (always has latest data) | Slightly higher | 1 RCU per 4KB | No |

Strong consistency costs 2× as many RCUs. GSIs are always eventually consistent because they replicate asynchronously.

---

## Transactions

DynamoDB supports ACID transactions via `TransactWriteItems` and `TransactGetItems`:
- Serializable isolation
- Up to 100 items across multiple tables in a single transaction
- 2× the normal WCU/RCU cost

The "NoSQL means no transactions" criticism is outdated — DynamoDB has had full transactions since 2018.

---

## Scaling and Architecture

DynamoDB auto-scales at the partition level:
- When a partition hits its capacity ceiling (3,000 RCU or 1,000 WCU), DynamoDB splits it
- Data is redistributed; the partition metadata service tracks the new mapping
- This happens transparently — no downtime

**Fault tolerance:** Each item is automatically replicated across 3 Availability Zones using Multi-Paxos consensus. One leader per partition handles all writes; a follower is promoted if the leader fails. For cross-region replication, enable **Global Tables**.

**Pricing models:**
- **On-demand**: pay per request; good for unpredictable traffic
- **Provisioned**: reserve RCU/WCU hourly; cheaper for predictable loads; supports auto-scaling

One partition supports 3,000 RCU and 1,000 WCU. A single RCU = one strongly consistent read of up to 4KB (or two eventually consistent reads). A single WCU = one write of up to 1KB.

---

## Advanced Features

### DAX (DynamoDB Accelerator)

Purpose-built in-memory cache that sits in front of DynamoDB. Microsecond reads for read-heavy workloads. Drop-in SDK replacement (API-compatible). Maintains an item cache and a query cache. Caveat: DAX doesn't cache strongly consistent reads — those bypass the cache and hit DynamoDB directly. Also: if you write to DynamoDB directly (bypassing DAX), the DAX cache won't be invalidated until TTL expiry.

### DynamoDB Streams

Built-in CDC (change data capture): every insert, update, and delete on a table is recorded in a stream in near-real-time. Common uses:
- Sync a DynamoDB table to Elasticsearch for full-text search
- Trigger Lambda functions on data changes (send notifications, invalidate caches)
- Real-time analytics (pipe to Kinesis → Firehose → S3/Redshift)

---

## When to Use DynamoDB

**Good fit:**
- You need key-value or document storage with predictable, simple access patterns
- You need to scale horizontally to massive throughput without managing infra
- You're in AWS and want managed everything (backups, replication, scaling)
- Your team wants single-digit millisecond latency at any scale

**Not a good fit:**
- You need flexible ad-hoc queries, joins, or complex aggregations → use PostgreSQL
- You need strong consistency on Global Secondary Indexes → LSIs or redesign
- You have very high write rates and cost is a concern (high-volume writes at 1WCU/1KB add up fast)
- Your interviewer wants vendor-neutral answers → propose Cassandra or another open-source alternative

---

## Common Gotchas

- **Scans are not free** — billed per KB read, not per result returned; a scan on a 100GB table costs as much whether it returns 1 item or 10 million
- **LSIs can't be added later** — plan your local secondary indexes at design time or you'll be rebuilding the table
- **GSIs are eventually consistent** — if strong consistency matters for a query path, you can't use a GSI there
- **400KB item limit** — large blobs (attachments, full documents) belong in S3; store the S3 key in DynamoDB
- **Hot partition keys** — `status`, `date`, `country` are almost always bad partition keys; pick something with high cardinality

---

## Practice Questions

1. You're designing a messaging app. Each chat has thousands of messages; you need to load the last 50 messages for a chat, and also see all messages a specific user has sent across all chats. Design the DynamoDB table and the indexes needed.
2. What's the difference between a GSI and an LSI in terms of consistency, cost, and when you can create them? Give a concrete example where you'd choose each.
3. A table has `user_id` as partition key and `created_at` (timestamp) as sort key. Explain why using timestamp as a sort key for a high-throughput table is risky and how you'd fix it.
4. Your DynamoDB table has 10 million items and you need to find all records where `status = 'pending'`. Walk through why this is a problem and two architectural approaches to solve it.
5. When would you enable DAX in front of DynamoDB, and what's the failure mode you need to be aware of when updates bypass DAX and go directly to DynamoDB?

---

## One-Line Summary

DynamoDB is AWS's fully-managed NoSQL powerhouse — design your partition key and secondary indexes around your access patterns upfront, because that decision determines everything about its performance and cost.
