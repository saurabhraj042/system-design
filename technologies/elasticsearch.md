# Elasticsearch

## The Core Idea

Elasticsearch is a distributed search and analytics engine built on top of Apache Lucene. When you need relevance-ranked full-text search, fuzzy matching, faceted filters, or geospatial queries — and your data grows too large for PostgreSQL's GIN indexes to handle well — Elasticsearch is the go-to. The catch: it's eventually consistent, not a primary database, and expensive to update data frequently in.

## Mental Model / Analogy

Think of a library catalog system. The library (cluster) holds millions of books (documents) spread across multiple rooms (shards). Each room maintains its own card catalog (Lucene inverted index) — every unique word mapped to the list of books containing it. When a patron asks "find all books with the word 'lazy'," the library doesn't open every book; it checks the card catalog in constant time. A desk at the front (coordinating node) handles all patron requests, figures out which rooms are relevant, collects their answers, and sorts them by relevance.

The "Russian dolls" structure: Elasticsearch Index → Shards → Replicas → Lucene Index → Segments.

---

## Core Concepts

### Documents, Indexes, Mappings, Fields

- **Document**: the basic unit of storage; a JSON object. Like a row in a relational database
- **Index**: a collection of documents with a shared mapping. Like a table
- **Mapping**: the schema — defines field names and their types
- **Field**: a typed attribute on a document

### Field Types: Keyword vs Text

This distinction drives search behavior:

| Type | Stored as | Matches | Use for |
|---|---|---|---|
| `keyword` | Exact string, hash table index | Exact only (`"status"=="active"`) | IDs, status codes, tags, enums |
| `text` | Tokenized, inverted index | Full-text search, partial words | Article content, product descriptions |

A field can have both types simultaneously using `fields` in the mapping — `name.keyword` for exact match, `name` for full-text search.

### REST API

Elasticsearch uses JSON over HTTP:

```bash
# Create/update a document
PUT /products/_doc/123
{ "name": "Blue Widget", "price": 29.99, "category": "hardware" }

# Search
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "widget" } }
      ],
      "filter": [
        { "term":  { "category": "hardware" } },
        { "range": { "price": { "gte": 10, "lte": 50 } } }
      ]
    }
  }
}
```

`must` — must match (affects relevance score); `filter` — must match but doesn't contribute to score (cached aggressively).

**Optimistic concurrency:** Use `_version` on writes. If the version doesn't match what you sent, ES rejects the write with a 409 conflict, letting you retry with fresh data.

---

## Search Features

### Full-Text Search

Elasticsearch tokenizes `text` fields at index time. The query `"match": { "description": "blue widget" }` searches each token independently and scores documents by how well they match. Relevance scoring uses TF-IDF (term frequency-inverse document frequency) by default, which rewards documents where the search term appears often but isn't common across all documents.

### Boolean Queries

```json
{
  "bool": {
    "must":    [{ "match": { "title": "nginx" } }],
    "should":  [{ "match": { "tags":  "tutorial" } }],
    "must_not":[{ "term":  { "status": "draft" } }],
    "filter":  [{ "range": { "date": { "gte": "2024-01-01" } } }]
  }
}
```

- `must`: required, scores count
- `should`: optional boosters (more matches = higher score)
- `must_not`: exclusion
- `filter`: required but not scored (fast, cached)

### Geospatial Search

```json
{
  "query": {
    "geo_distance": {
      "distance": "10km",
      "location": { "lat": 37.77, "lon": -122.41 }
    }
  }
}
```

ES uses `geo_point` fields (lat/lng pair) with BKD tree indexing for efficient radius queries. `geo_shape` supports polygons for things like "is this address inside this neighborhood boundary?" Use `geo_distance` for simple proximity; `geo_shape` with `geo_bounding_box` or `geo_polygon` for complex shapes.

### Sorting

Four ways to sort results:

1. **Field sort**: `"sort": [{ "price": "asc" }]` — sorts by a concrete field value
2. **Script sort**: `"sort": [{ "_script": { "source": "doc['price'].value * doc['rating'].value" } }]` — arbitrary Painless script
3. **Nested sort**: sort by a field inside a nested object (common in e-commerce)
4. **Relevance sort** (default): `_score` descending — most relevant result first (TF-IDF)

### Pagination

Three approaches:

| Method | How | When to use |
|---|---|---|
| `from`/`size` | Skip first N results | Simple pagination; breaks above 10,000 results |
| `search_after` | Stateless cursor using sort values from last page | Infinite scroll, large offsets; results may shift between pages |
| PIT + `search_after` | Point-in-time snapshot, cursor-based | Consistent pagination across data changes |

`from`/`size` is simple but forces ES to load and discard all skipped results — expensive for deep pages. `search_after` with a PIT cursor handles "load more" reliably.

---

## Architecture

### The Russian Dolls

```
Elasticsearch Cluster
 └── Elasticsearch Index
      └── Shards (primary + replicas, across nodes)
           └── Lucene Index (1:1 with shard)
                └── Lucene Segments (immutable containers)
                     ├── Inverted Index (term → doc IDs)
                     └── Doc Values (columnar per-field data)
```

### Shards and Replicas

**Shards** let ES split data across nodes. Different shards hold different documents; searches run across all relevant shards in parallel and results are merged by the coordinating node.

**Replicas** are exact copies of shards for high availability and throughput. If a shard can handle X TPS, Y replicas means the cluster handles X×Y TPS (coordinating node distributes reads across all copies).

Key: ES shards are 1:1 with Lucene indexes. Operations that ES performs on shards — searching, merging, splitting — are ultimately Lucene index operations.

### Node Types

| Node type | Responsibility |
|---|---|
| **Master** | Cluster admin — track nodes, manage index lifecycle, no data |
| **Data** | Store documents and Lucene indexes; handle searches |
| **Coordinating** | Frontend — accept client queries, distribute to data nodes, merge results |
| **Ingest** | Pre-process documents (parse, enrich, transform) before indexing |
| **ML** | Machine learning anomaly detection and inference |

For queries: client → coordinating node → data nodes → coordinating node merges → client.

### Lucene Segments: Immutable Building Blocks

A Lucene index is made of **segments** — immutable containers of indexed data. New documents don't immediately modify existing segments; they go into a new, small segment that's eventually flushed to disk.

**Why immutable?**
- New writes don't block readers (no lock contention)
- Segments can be safely cached in memory or SSD
- Crash recovery is straightforward (known consistent state)
- Enables efficient compression

**Writes:**
- New documents accumulate in an in-memory buffer
- The buffer is periodically flushed to a new segment on disk
- Documents are searchable only after the segment is flushed (by default every 1 second — near-real-time, not real-time)

**Updates** = soft delete the old document + insert new one. The old document is hidden from queries but physically stays in the segment until segment merge.

**Deletes** = mark in a "deleted identifiers" bitset per segment; data remains until merge.

**Segment merging** combines small segments into larger ones, cleans up deletions, and reclaims disk space. ES manages this automatically.

**Important caveat:** Updates and deletes have worse performance than pure inserts because of the soft-delete bookkeeping. Elasticsearch struggles with data that updates frequently (e.g., tracking like counts or impression counters). Don't use ES as your primary database for mutable data.

### Inverted Index

The inverted index maps from tokens (words) to the documents containing them:

```
Documents:          Index (inverted):
ID:12 "lazy dog"   lazy: [12, 53]
ID:53 "lazy days"  dog:  [12]
                   days: [53]
```

Finding all documents with "lazy" = O(1) lookup into the inverted index → returns [12, 53]. Compare to scanning every document = O(n).

The inverted index also stores term frequencies and positions (for phrase queries).

### Doc Values

When you sort results by price, ES needs to read just the `price` field for all matching documents. The inverted index maps tokens to document IDs, not the other way around. **Doc values** store a columnar (column-per-field, contiguous in memory) representation of field values for all documents in a segment — perfect for sorting and aggregations. This is the same principle behind columnar databases like Redshift.

### Query Planner

The coordinating node runs a **query planner** that chooses the most efficient execution strategy based on field statistics. For a phrase query "bill nye" where "nye" appears in 400 documents and "bill" appears in millions, the planner computes the intersection starting from the smaller set (nye), dramatically reducing work.

---

## Key Trade-offs

| What you gain | What you lose |
|---|---|
| Relevance-ranked full-text search | Eventually consistent (results will be stale) |
| Geospatial and range queries | Not a primary database — durability historically weak |
| Horizontal scaling across shards | Updates are expensive (soft delete + reinsert) |
| Aggregations and analytics | Operational complexity of cluster management |
| Near-real-time search (1s refresh) | Must sync with source of truth (drift risk) |

---

## In Your Interview

Elasticsearch typically appears via **Change Data Capture (CDC)** from an authoritative database:

```
PostgreSQL / DynamoDB (source of truth)
        ↓ CDC (Debezium, DynamoDB Streams, etc.)
   Elasticsearch (search index)
        ↓
   Search API queries
```

ES is a secondary read index, not the primary store.

**When Elasticsearch is overkill:**
- Data under ~100k documents → plain SQL with GIN full-text index works fine
- Search patterns are simple (exact match, basic range) → query your primary DB
- Updates are frequent (like counts, view counts) → ES will struggle with constant soft-delete/reinsert churn

**When Elasticsearch is the right call:**
- Relevance scoring matters (ranked results, fuzzy matching, typo tolerance)
- Faceted search with many filter combinations (e-commerce filter UX)
- Large dataset + many concurrent complex queries
- Cross-field search with boolean logic across many fields

**Key rules when proposing ES in an interview:**
1. Don't use it as your primary store — if data needs to persist, store it elsewhere
2. Account for eventual consistency — results will be stale, sometimes significantly
3. Denormalize data in the ES index — avoid needing joins; one query should be enough
4. Keep it in sync — CDC failures cause index drift; plan for re-indexing recovery

---

## Common Gotchas

- **ES is a search engine, not a database** — don't let it be your only copy of the data
- **Writes are expensive for high-update data** — avoid fields that change constantly (counters, status flags that flip often)
- **Deep pagination is slow** — `from: 10000, size: 10` forces ES to load and rank 10,010 results then discard 10,000
- **Segment merge overhead** — heavy write + delete workloads cause ongoing merge pressure; plan disk headroom for temporary merge storage
- **Cluster state can be lost** — if master nodes all fail simultaneously and quorum is broken, the cluster is inoperable

---

## Practice Questions

1. You're building a product catalog search for an e-commerce site. Users can search by text, filter by category/price range, and sort by price or relevance. Design the ES mapping (field types and index structure) to serve these queries efficiently.
2. Your team stores user profile data in PostgreSQL and wants to add full-text search over user bios. Walk through when you'd keep this in PostgreSQL (GIN index) vs. add Elasticsearch. What's the tipping point?
3. A user searches for "nikon d750 lense" (typo in "lens"). Explain how Elasticsearch would handle this vs. a SQL `LIKE '%lense%'` query.
4. Your Elasticsearch index is used for pagination — users scroll through 50,000 search results. Explain why `from/size` breaks at page 200+ and how you'd implement correct deep pagination using PIT + `search_after`.
5. Your ES index and the source PostgreSQL database have diverged — some records in ES are deleted but still appear in search, and some new records don't appear yet. Walk through how you'd detect and fix this without rebuilding the entire index from scratch.

---

## One-Line Summary

Elasticsearch is a distributed search engine built on Lucene — use it when you need relevance ranking, full-text search, or faceted geospatial queries at scale, but always keep it as a CDC-fed secondary index beside your primary database, never the source of truth.
