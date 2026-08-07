# Database Indexing

## The Core Idea

A database index is a separate data structure that the database maintains to find records faster. Without an index, every query must scan every row in the table. With an index, the database can jump directly to matching records in O(log n) time or better.

The trade-off: indexes speed up reads but slow down writes, because every insert/update/delete must also update every index on that table. More indexes = faster queries, slower writes, more storage.

## Mental Model / Analogy

Think of navigating a city without and with GPS. Without GPS (no index), you drive every street until you find your destination — efficient for knowing the whole city, terrible for finding one address. With GPS (with an index), you type the address and GPS routes you directly. But maintaining GPS requires infrastructure — satellites, map updates, routing servers. That's the write overhead of indexes: the "map" (index) must stay updated whenever new addresses (rows) are added.

Different index types are like different navigation tools: a phone book (B-tree) is sorted alphabetically and great for range queries ("all restaurants between A and M"). A GPS grid system (geospatial index) finds nearby locations in 2D space. A search engine (inverted index) maps keywords to documents.

---

## Why Indexes Matter: Physical Storage

Without indexes, the database stores rows in **heap files** — unordered pages on disk. To find all posts by user_id = 123:
1. Load page 1 from disk into RAM — check every row
2. Load page 2 from disk into RAM — check every row  
3. Repeat for all N pages

With millions of rows, this is catastrophically slow. An index is a separate structure that points to where the matching rows live, so the database loads only relevant pages.

**The key insight:** disk reads are orders of magnitude slower than RAM access. Good indexes minimize disk I/O by loading only the relevant pages.

---

## Index Types

### B-Tree Indexes — The Default

A B-tree is a self-balancing tree where each node can have many children (hundreds in practice, sized to fill an 8KB disk page). Each level of the tree dramatically narrows the search.

```
                    [50]
                   /    \
             [25]          [75]
            /    \        /    \
          [10]   [40]  [60]   [90]
```

**Characteristics:**
- O(log n) lookup for any query
- Handles equality queries: `WHERE user_id = 123`
- Handles range queries: `WHERE created_at > '2024-01-01'`
- Handles sorting: `ORDER BY price DESC`
- Keeps data sorted, so range scans are sequential reads on disk (fast)

**This is your default for most columns.** PostgreSQL uses B-trees for all primary keys, unique constraints, and most regular indexes. MongoDB's default index type is a B+ tree variant.

```sql
CREATE INDEX idx_posts_user_id ON posts (user_id);
CREATE INDEX idx_posts_created_at ON posts (created_at DESC);
```

### LSM Trees (Log-Structured Merge Trees) — Write-Optimized

LSM trees are designed for write-heavy workloads. Instead of updating a B-tree in place on every write, all writes are first accumulated in memory (fast), then flushed to disk in sorted batches (sequential writes, also fast).

**Write path:**
```
Incoming write
     ↓
1. Append to WAL (Write-Ahead Log) — sequential disk write for durability
     ↓
2. Write to Memtable (sorted in-memory structure, like a red-black tree)
     ↓
3. When Memtable fills → flush to immutable SSTable on disk
     ↓
4. Background compaction merges multiple SSTables periodically
```

**Why writes are fast:** Steps 1 (sequential log append) and 2 (in-memory) are the critical path. Random writes — the B-tree's Achilles heel — are avoided entirely.

**Read trade-off:** Reading a key requires checking the Memtable + potentially many SSTables. Two optimizations:
- **Bloom filters:** Probabilistic data structure for each SSTable. "Does key 123 definitely NOT exist in this SSTable?" — if yes, skip the SSTable entirely. Very effective at reducing unnecessary disk reads.
- **Sparse indexes:** Track the range of keys in each SSTable block, enabling quick range elimination.

**Compaction strategies:**
- **Size-tiered:** Merge SSTables of similar size. Simple but temporarily uses 2× storage during merge.
- **Leveled:** Maintain strict size limits per level; more CPU overhead but better read performance and more predictable space usage.

**Used by:** Cassandra, RocksDB (used inside many systems), DynamoDB internals, HBase.

In interviews: propose LSM trees when you need extremely high write throughput and can tolerate slightly slower reads. The classic trade-off is write speed vs read speed vs storage amplification.

### Hash Indexes

A persistent hash map: `hash(key) → file offset`. O(1) exact-match lookups.

**When it wins:** Pure key-value lookups with no range queries and no sorting needed. `WHERE user_id = 123` is instant; `WHERE user_id > 100` is impossible (hash destroys ordering).

**In practice:** Redis uses hash tables for its core key-value operations (all in memory). PostgreSQL supports hash indexes but B-trees are nearly as fast for equality queries, so hash indexes are rarely used. The main database use case is in-memory exact-match lookups where B-tree overhead matters.

### Geospatial Indexes

Standard B-trees can't handle 2D location queries efficiently. `WHERE latitude BETWEEN 37.7 AND 37.8 AND longitude BETWEEN -122.5 AND -122.4` requires two separate B-tree range scans and intersecting them — slow.

Three approaches, from simplest to most powerful:

**Geohash:** Encode 2D lat/lng into a 1D base32 string that preserves proximity. Nearby locations share a common prefix. This makes geohash values indexable with a standard B-tree: `WHERE geohash LIKE 'abc%'` finds all points in that region.

```
San Francisco: 9q8yy  (geohash at ~2.4km precision)
A block away:  9q8yy  (same prefix = nearby)
Tokyo:         xn76h  (completely different prefix = far away)
```

**Redis uses geohash** internally for `GEOSEARCH` commands. Limitation: regions near geohash cell boundaries may be in adjacent cells with different prefixes.

**Quadtree:** Recursively subdivide 2D space into four quadrants. Dense areas get subdivided further; sparse areas stay as large cells. Good for adaptive resolution but less common in modern databases.

**R-Tree:** Organizes space using flexible, potentially overlapping bounding rectangles that adapt to data clusters. Can handle both point queries (find nearby restaurants) and shape queries (is this building inside this flood zone?).

**PostgreSQL + PostGIS uses R-trees** (GiST index). MySQL also uses R-trees. The most capable option for complex geospatial workloads. Good for any system where users search for nearby things.

**Interview default:** If you're using PostgreSQL, mention a GiST index with PostGIS. If using Redis for a simpler system, mention `GEOSEARCH` with the built-in geohash. If you only want to know one approach to explain deeply, learn geohashing.

### Inverted Indexes

Used for full-text search. The structure is: `word → [list of document IDs containing that word]`.

```
"lazy" → [doc12, doc53, doc97]
"dog"  → [doc12, doc44]
"days" → [doc53]
```

**Analysis pipeline:**
1. **Tokenize:** split text into words
2. **Lowercase:** normalize case
3. **Remove stop words:** filter "the", "a", "is" (too common to be useful)
4. **Stemming:** reduce words to root form ("running" → "run", "boxes" → "box")

**Scoring (TF-IDF):** Documents that contain the search term many times (high term frequency) and where the term is rare across all documents (high inverse document frequency) score highest.

**Capabilities:** fuzzy matching (handles typos), phrase queries ("exact phrase"), boolean operators (AND/OR/NOT).

**Trade-offs:** Substantial storage overhead (the inverted index can be as large as or larger than the original data). Expensive to update — every insert/update requires re-indexing the changed text. This is why Elasticsearch struggles with frequently-updated data.

**Used by:** Elasticsearch/Lucene (GitHub search, Slack search, e-commerce product search), PostgreSQL GIN indexes (built-in full-text search).

**Interview trigger:** "Users can search post content" or "full-text search" → inverted index.

---

## Composite Indexes

The most common optimization pattern in practice. Instead of separate indexes on multiple columns, combine them into one index:

```sql
-- Suboptimal: two separate indexes
CREATE INDEX idx_user ON posts (user_id);
CREATE INDEX idx_time ON posts (created_at);

-- Better: one composite index
CREATE INDEX idx_user_time ON posts (user_id, created_at);
```

For a query like `WHERE user_id = 123 AND created_at > '2024-01-01' ORDER BY created_at DESC`, a composite index on `(user_id, created_at)` handles filtering AND sorting in a single index scan. Two separate indexes require the database to scan both, intersect the results, and re-sort.

**The order matters critically:** An index on `(user_id, created_at)` is excellent for queries filtering by user_id. It's useless for queries that only filter by created_at — B-trees can only use the index from left to right through its prefix columns.

**Rule of thumb:** Put the most selective column first (user_id narrows to one user's posts; created_at alone is less selective across all users). Exception: if you always sort by a column, including it last in the composite index eliminates the sort operation.

**Common patterns:**
- Order history: `(customer_id, order_date)`
- Event processing: `(status, priority, created_at)`
- Activity feeds: `(user_id, type, timestamp)`

---

## Covering Indexes

A covering index includes all columns needed by a query — not just the columns being filtered or sorted. The database can answer the query entirely from the index without touching the main table (no "heap fetch").

```sql
-- Regular index: DB finds matching posts, then fetches each row for the likes column
CREATE INDEX idx_user_time ON posts (user_id, created_at);

-- Covering index: DB answers query entirely from the index  
CREATE INDEX idx_user_time_likes ON posts (user_id, created_at) INCLUDE (likes);

-- This query now never touches the main table:
SELECT likes FROM posts WHERE user_id = 42 ORDER BY created_at DESC LIMIT 20;
```

**When it's worth it:** Frequently-run queries that only need a small subset of columns from a large table. Common for social feed queries, leaderboards, and dashboards.

**Interview note (2026):** Modern database query optimizers have gotten smarter at minimizing heap fetches even without covering indexes. Covering indexes are a useful optimization but a niche one — have a specific reason before proposing them. When in doubt, stick to simpler indexing strategies.

---

## Index Selection Flowchart

When asked what index to use:

1. Table has < 10k rows → full table scan is fine, no index needed
2. Full-text search → inverted index (Elasticsearch or PostgreSQL GIN)
3. Location data → geospatial index (geohash for Redis, PostGIS R-tree for PostgreSQL)
4. In-memory exact-match only → hash index
5. Everything else → B-tree
   - Multiple columns queried together → composite B-tree index
   - Heavy reads on few columns → consider a covering index

---

## Write Trade-offs

Every index on a table adds overhead to every insert, update, or delete:
- The database must update the index structure for every row modification
- More indexes = slower writes

**General guidance:**
- Add indexes that serve your most critical read queries
- Remove indexes that don't serve any query (dead weight on write performance)
- Tables with very high write rates (event logs, telemetry) should have minimal indexes
- Tables with heavy read access and less frequent writes (product catalog, user profiles) can have more indexes

---

## Interview Application

When you propose a database schema, call out your indexes explicitly:

"I'll put an index on `posts.user_id` to serve the timeline query. I'll also add a composite index on `(user_id, created_at DESC)` so that the `GET /users/{id}/posts` endpoint can sort by recency in a single index scan without a separate sort operation."

For full-text search: "Since users can search post content, I'll use Elasticsearch fed via CDC from our primary database, which maintains an inverted index for relevance-ranked text search."

For nearby search: "I'll use a GiST index with PostGIS for driver location queries. If we use Redis, `GEOSEARCH` with geohash handles this natively."

---

## Key Trade-offs

| Index type | Reads | Writes | Range queries | Memory |
|-----------|-------|--------|---------------|--------|
| B-tree | O(log n) | Overhead to maintain | Yes | Moderate |
| LSM tree | Slightly slower | Very fast | Yes | Moderate |
| Hash index | O(1) | Overhead | No | High (all in memory) |
| Geospatial | Fast proximity | Overhead | Yes (2D) | Moderate |
| Inverted | O(1) lookup | High (re-indexing) | No | High (large index) |

## When to Use

- **B-tree:** Default for any indexed column; equality and range queries on structured data
- **LSM tree:** Write-heavy workloads (Cassandra, RocksDB, DynamoDB internals)
- **Hash:** Pure key-value in-memory lookups (Redis)
- **Geospatial:** Any query involving proximity or location (ride-sharing, food delivery, map search)
- **Inverted:** Full-text search (search boxes, document retrieval, Slack/GitHub search)

## Common Gotchas

- **Indexing every column** — more indexes slow down writes and waste storage; index only what queries need
- **Forgetting composite index column order** — `(user_id, created_at)` doesn't help a query filtering only on `created_at`
- **Using a LIKE query with a leading wildcard** — `WHERE name LIKE '%smith'` can't use a B-tree; `WHERE name LIKE 'smith%'` can
- **Not indexing foreign keys** — PostgreSQL auto-indexes primary keys but NOT foreign key columns; explicitly add indexes on FK columns used in JOINs
- **Covering indexes as a first resort** — modern optimizers are smart; reach for covering indexes only for proven bottlenecks

## Practice Questions

1. Your `posts` table has 50M rows and the query `SELECT * FROM posts WHERE user_id = 123 ORDER BY created_at DESC LIMIT 20` takes 800ms. Walk through exactly why it's slow and what index you'd add.
2. You need to build "find all restaurants within 5km of the user's current location." Compare three approaches: two separate B-tree range scans on lat/lng, geohash with a B-tree, and an R-tree. Which do you choose and why?
3. A product search needs to match "runnning shoes" (typo) and rank results by relevance. Why can't a B-tree index serve this, and what does an inverted index provide that makes this possible?
4. You're designing a messaging app. The most common query is "all messages in channel X, sorted by time, last 50." Design the index. Now add a requirement: "all messages sent by user Y across all channels." How does this change things?
5. An e-commerce platform indexes 6 columns on the `orders` table for various queries. Insert performance is degrading. Walk through how you'd audit the indexes and decide which to remove.

## One-Line Summary

B-trees are the default index for reads — use composite indexes when multiple columns are queried together, geospatial indexes for location queries, inverted indexes for full-text search, and LSM trees (Cassandra/RocksDB) when write throughput is the bottleneck.
