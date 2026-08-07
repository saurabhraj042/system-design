# PostgreSQL

## The Core Idea

PostgreSQL is the default choice for a system design interview database. It's a fully-featured relational database: ACID transactions, rich querying with JOINs, mature indexing, built-in full-text search, JSONB for flexible schemas, and PostGIS for geospatial. It's not the fastest for every workload, but it handles "typical application" needs well enough that you should always start here and justify any deviation.

## Mental Model / Analogy

PostgreSQL is like a well-organized spreadsheet building with very strict filing rules. Each floor is a table, each row is a record, and the building manager (query planner) figures out the fastest way to find any record using the building's index directory. The building enforces strict rules: you can't delete a tenant (row) that's still referenced elsewhere, you can't overdraw an account balance past zero, and every transaction either completes entirely or is rolled back — no half-finished moves.

The B-tree index is the building directory: sorted alphabetically so lookups are O(log n). A GIN index is like a keyword index in a book — given a word, instantly find every page that contains it.

---

## Data Model

### Relational Tables

PostgreSQL stores data in tables (also called relations). Each column has a declared type; each row is one complete record. Tables connect via foreign keys:

```sql
-- Users own posts; PostgreSQL enforces this automatically
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    username   VARCHAR(50) UNIQUE NOT NULL,
    email      VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE posts (
    id         SERIAL PRIMARY KEY,
    user_id    INTEGER REFERENCES users(id),
    content    TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Relationship types** (know these for interviews):
- **One-to-One**: user → profile settings
- **One-to-Many**: user → posts (one user, many posts; `REFERENCES` on the many side)
- **Many-to-Many**: users ↔ posts they've liked (join table with two foreign keys)

**Normalization vs. denormalization:** The default is normalized — each fact lives once, relationships link to it. Sometimes you intentionally denormalize (store `like_count` on the posts table instead of counting the likes table) for read performance. This is a deliberate trade-off worth discussing in interviews.

### JSONB

For flexible/variable schemas, JSONB stores arbitrary JSON in a binary format with full indexing support:

```sql
ALTER TABLE products ADD COLUMN metadata JSONB;

-- GIN index on entire JSONB for flexible queries
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- Query inside JSONB
SELECT * FROM products WHERE metadata @> '{"color": "blue"}';
```

Useful when different products have completely different attribute sets (electronics vs. clothing vs. furniture) and you don't want 200 nullable columns.

---

## ACID Properties

PostgreSQL is strictly ACID-compliant. In interviews, know what each letter means and why it matters:

**Atomicity (All or Nothing):** A transaction's operations either all succeed or all roll back. A money transfer that debits one account but crashes before crediting the other will automatically roll back — the debit never happened.

**Consistency (Data Integrity):** Transactions can only bring the database from one valid state to another. If you have `CHECK (balance >= 0)`, PostgreSQL rejects any transaction that would make a balance negative — unlike NoSQL databases where you enforce this in application code.

```sql
CREATE TABLE accounts (
    account_id TEXT PRIMARY KEY,
    balance    DECIMAL CHECK (balance >= 0),
    owner_id   INTEGER REFERENCES users(id)
);
```

**Isolation (Concurrent Transactions):** Concurrent transactions don't interfere with each other. PostgreSQL uses MVCC (Multi-Version Concurrency Control) — each transaction sees a consistent snapshot of the data as it existed when the transaction started, regardless of what other transactions are doing concurrently.

Isolation levels and what they prevent:

| Isolation Level | Dirty Read | Nonrepeatable Read | Phantom Read | Notes |
|---|---|---|---|---|
| Read Uncommitted | Allowed (but not in PG) | Possible | Possible | PG treats same as Read Committed |
| **Read Committed** (default) | Not possible | Possible | Possible | Default; fine for most use cases |
| Repeatable Read | Not possible | Not possible | Allowed (but not in PG) | PG's SSI prevents phantom reads too |
| Serializable | Not possible | Not possible | Not possible | Full isolation; highest cost |

**Durability (Permanent Storage):** Once PostgreSQL says a transaction committed, the data is guaranteed to survive crashes and power failures. This is achieved via Write-Ahead Logging (WAL): changes are first written to the WAL on disk, then the transaction is confirmed. Even if PostgreSQL crashes before writing to the actual data pages, it can replay the WAL on restart.

**Why ACID matters in interviews:**
- Financial transactions → need ACID to prevent double-spending or lost money
- User authentication → need ACID to prevent race conditions on account creation
- Social media likes → can tolerate eventual consistency (counts being slightly off is fine)
- Analytics data → can trade consistency for performance

---

## Indexing

Indexing is where PostgreSQL shows its depth. Different index types for different query patterns:

### B-tree (Default)

Best for exact lookups, range queries, and sorting:
```sql
CREATE INDEX idx_posts_user_id    ON posts (user_id);
CREATE INDEX idx_posts_created_at ON posts (created_at DESC);

-- Range query: O(log n) instead of O(n)
SELECT * FROM posts WHERE created_at > NOW() - INTERVAL '7 days';
```

### GIN Index for Full-Text Search

```sql
-- Add a computed tsvector column for full-text search
ALTER TABLE articles ADD COLUMN search_vector TSVECTOR;
UPDATE articles SET search_vector = to_tsvector('english', title || ' ' || content);
CREATE INDEX idx_articles_search ON articles USING GIN (search_vector);

-- Query
SELECT * FROM articles WHERE search_vector @@ plainto_tsquery('english', 'distributed systems');
```

GIN (Generalized Inverted Index) stores term → row mappings, like Elasticsearch's inverted index. Performance is comparable to ES for simple full-text search on tables up to tens of millions of rows.

### PostGIS for Geospatial (GiST Index)

```sql
CREATE EXTENSION postgis;
ALTER TABLE drivers ADD COLUMN location GEOGRAPHY(Point, 4326);
CREATE INDEX idx_drivers_location ON drivers USING GIST (location);

-- Nearby search (drivers within 5km)
SELECT * FROM drivers
WHERE ST_DWithin(location, ST_MakePoint(-122.41, 37.77)::geography, 5000);
```

PostGIS can replace Elasticsearch or dedicated geo databases for many use cases.

### Covering Indexes (INCLUDE)

Avoid fetching the full row when the query only needs indexed columns:
```sql
-- Without INCLUDE: index lookup → fetch row from table
-- With INCLUDE: answered entirely from the index
CREATE INDEX idx_posts_user_created ON posts (user_id, created_at DESC)
INCLUDE (title, like_count);

-- This query is now served entirely from the index:
SELECT title, like_count FROM posts
WHERE user_id = 42 ORDER BY created_at DESC LIMIT 20;
```

### Partial Indexes (WHERE Clause)

Index only a subset of rows:
```sql
-- Only index unread notifications (the hot path)
CREATE INDEX idx_notifications_unread ON notifications (user_id, created_at)
WHERE read = false;
```

Dramatically smaller index, faster lookups, only helps queries that include the partial condition.

---

## Read Performance

Approximate limits for a well-configured single PostgreSQL instance:
- Simple indexed lookup: ~50,000 reads/sec/core
- Multi-table joins: thousands/sec
- Full-text search: fine up to tens of millions of rows
- Tables become unwieldy beyond ~100M rows without partitioning

### Read Scaling Strategies

**Read replicas:** Stream WAL to async followers; reads go to followers, writes to primary. Adds horizontal read capacity. Replica reads are slightly stale (async replication lag).

**Connection pooling (PgBouncer):** PostgreSQL is process-per-connection; opening thousands of raw connections is expensive. PgBouncer multiplexes many client connections over a small pool of real PostgreSQL connections. Essential at scale.

**Caching layer:** Put Redis in front for frequently-read, rarely-changed data. Cache invalidation is your problem.

---

## Write Performance

The PostgreSQL write path:
```
Client COMMIT
     ↓
1. Buffer Cache + WAL entry created (in memory)
     ↓
2. WAL flushed to disk (synchronous — this is the commit latency bottleneck)
     ↓
3. "Committed" returned to client
     ↓
4. Background Writer → data pages written to disk (async, later)
     ↓
5. Index Updates (background)
```

The disk write at step 2 is unavoidable for ACID durability. Every index on a table adds overhead to every write (index B-tree must be updated).

Throughput benchmarks (rough, depends on hardware):
- Simple inserts: ~5,000/sec/core
- Updates with indexes: ~1,000-2,000/sec/core
- Complex transactions: hundreds/sec
- Bulk (COPY): tens of thousands/sec

### Write Scaling Strategies

**Vertical scaling:** Faster disk (NVMe SSD), more RAM (bigger buffer cache reduces I/O), more CPU cores. Cheapest first step.

**Batch writes:** Instead of 1,000 individual INSERTs, send them in batches of 100 or use `COPY`. Amortizes WAL overhead.

**Write offloading:** Accept writes into Kafka, buffer them, and drain into PostgreSQL at a controlled rate. Smooths traffic spikes and provides natural backpressure.

**Table partitioning:** Partition by time range to keep hot data (recent partitions) in memory, and enable dropping/archiving old partitions:
```sql
CREATE TABLE events (
    id         BIGINT,
    created_at TIMESTAMPTZ NOT NULL,
    payload    JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_01 PARTITION OF events
FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

**Sharding:** Distribute tables across multiple PostgreSQL instances by shard key (e.g., `user_id % 8`). Manual sharding is painful; Citus (now distributed PostgreSQL) automates this. Only reach for this after exhausting other options.

---

## Concurrency and Locking

### Row-Level Locking

```sql
-- Pessimistic: lock rows for update within transaction
BEGIN;
SELECT * FROM accounts WHERE user_id = 42 FOR UPDATE;  -- holds lock
UPDATE accounts SET balance = balance - 100 WHERE user_id = 42;
COMMIT;
```

`FOR UPDATE` prevents other transactions from reading or modifying the same row until you commit.

### Optimistic Concurrency Control

Better for low-contention scenarios:
```sql
-- Add a version column to track concurrent updates
ALTER TABLE products ADD COLUMN version INTEGER DEFAULT 0;

-- Update only if version matches what you read
UPDATE products
SET price = 29.99, version = version + 1
WHERE id = 123 AND version = 5;  -- fails if someone else updated in between
```

If `rowcount = 0`, the version didn't match — retry from the top with fresh data. No locks held while you think.

---

## Replication

**Async replication** (default): Primary sends WAL to replicas after committing. Fastest. If primary crashes before WAL reaches a replica, those writes are lost.

**Sync replication**: Primary waits for at least one replica to confirm WAL receipt before returning success to the client. Slower (adds replica RTT to every write), but zero data loss on primary crash.

Typical production setup: sync replication to one replica (for HA and zero data loss), async to the rest (for read scaling without impacting write latency).

---

## When to Use PostgreSQL

**Default choice for:** structured relational data, complex queries with joins, ACID transactions, rich querying flexibility, anything where you're not sure what queries you'll need. PostgreSQL's rich feature set can eliminate the need for additional systems:
- Full-text search → GIN index may replace Elasticsearch
- Flexible metadata → JSONB may replace MongoDB
- Geospatial → PostGIS may replace a dedicated geo database

**When to consider alternatives:**

**Extreme write throughput** (millions/sec): Each write requires a WAL flush and index update — I/O bottleneck even with NVMe. Consider Cassandra (LSM tree, write-optimized) or Redis (in-memory counters).

**Global multi-region active-active**: PostgreSQL has a single primary writer. Distributed writes across multiple regions means one region is always the master, others are read replicas. True active-active creates conflict resolution headaches. Consider CockroachDB (global ACID), Cassandra (eventual consistency), or DynamoDB Global Tables (managed active-active).

**Pure key-value patterns**: If you're only doing `GET key` / `SET key` with no joins or aggregations, PostgreSQL's MVCC + WAL overhead is unnecessary. Redis handles it faster; DynamoDB handles it at AWS scale.

> "Scalability alone is not a good reason to choose an alternative to PostgreSQL. PostgreSQL can handle significant scale with proper design."

---

## Common Gotchas

- **N+1 query problem**: loading 100 posts, then querying the author separately for each post = 101 queries. Use `JOIN` or eager loading to fetch in one query
- **EXPLAIN ANALYZE before optimizing**: don't guess where the slow query is; ask PostgreSQL to show you the query plan with actual timing
- **Vacuum and autovacuum**: MVCC means dead rows (old versions) accumulate; VACUUM cleans them up. If autovacuum falls behind, table bloat slows queries. Don't disable it
- **Connection count**: PostgreSQL forks a process per connection; 10,000 raw connections will exhaust OS resources. Use PgBouncer
- **Index on foreign key**: PostgreSQL doesn't auto-index foreign key columns; you often need to add indexes explicitly on the many-side of relationships for fast JOIN performance

---

## Practice Questions

1. You're building a fintech app where users transfer money between accounts. Design the PostgreSQL schema and explain which isolation level and locking mechanism you'd use to prevent double-spending or negative balances.
2. Your social media app has a `posts` table with 200 million rows. Users can search post content, filter by author, and sort by likes. What indexes would you create and in what order would you prioritize them?
3. An API that fetches the last 20 posts by a user is taking 800ms. Walk through how you'd diagnose this using `EXPLAIN ANALYZE` and what your first three optimization attempts would be.
4. Your system needs geospatial search: users want to find restaurants within 2km of their location. Walk through how you'd implement this in PostgreSQL with PostGIS. At what scale would you consider migrating to a dedicated geo service?
5. A product manager wants to add full-text search across 5 million product descriptions. Walk through building this in PostgreSQL with a GIN index. When would this not be enough and you'd need Elasticsearch?

---

## One-Line Summary

PostgreSQL is the right default — start with it, use JSONB for flexible schemas, PostGIS for geo, GIN for full-text search, read replicas for scale, and only deviate when you have a concrete requirement it can't meet.
