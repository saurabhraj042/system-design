# Data Modeling

## The Core Idea

Data modeling is deciding how your application's data is structured, stored, and related. In a system design interview, this isn't a formal database normalization exercise — it's showing you can design a schema that supports your system's requirements, won't collapse under expected load, and is aligned with your chosen database type.

Your data model shows up twice in the interview: first when identifying **core entities** during requirements, and again when sketching a **schema diagram** alongside your database in the high-level design. Include key fields, relationships, and a note on indexing and sharding for the main query patterns.

## Mental Model / Analogy

Designing a data model is like organizing a home library. You choose your shelving system (database type) based on how you'll use it: a bookcase with labeled categories works great if you browse by genre (SQL with indexes); binders organized by project work better if you always access a whole project together (document database); index cards with cross-references help if you constantly jump between related topics (graph database).

The key insight: the "right" organization depends on how you access information, not just what information you have. A library organized alphabetically is terrible if you always search by topic; organized by topic is terrible if you always look up specific titles.

---

## Picking the Right Database Type

The default is a **relational database**. Resist the temptation to use exotic database types to show off — most problems fit naturally into tables with relationships. Start with PostgreSQL and deviate only when requirements clearly demand it.

### Relational Databases (SQL) — Your Default

Data organized into tables with rows and columns. Relationships enforced via foreign keys. ACID transactions guaranteed.

**When to use:**
- Structured data with clear relationships
- Need for complex queries (JOINs, aggregations, filtering)
- Transactions across multiple records (payments, inventory)
- You're not sure what queries you'll need (SQL handles anything)

**Schema example (social media app):**
```
Users:    userId (PK), username, email, createdAt
Posts:    postId (PK), userId (FK → users.userId, index), content, mediaUrls, createdAt (index)
Comments: commentId (PK), postId (FK, index), userId (FK, index), content, createdAt
Likes:    userId (FK → users), postId (FK → posts)  [composite PK prevents duplicates]
```

**Technologies:** PostgreSQL (default), MySQL, SQLite

### Document Databases — For Flexible Schemas

Stores data as JSON-like documents. Flexible structure — different documents can have different fields. Related data is often embedded in the same document rather than normalized across tables.

**When to use:**
- Schema changes frequently (rapidly evolving product)
- Data is deeply nested and would require many JOINs in SQL
- Records have highly variable structure (user profiles with very different attribute sets)

**Data modeling impact:** Embed related data within documents to avoid expensive lookups across collections. This trades storage space and update complexity for read performance.

```json
{
  "_id": "507f1f77bcf86cd799439011",
  "username": "john_doe",
  "posts": [
    { "content": "Hello world!", "createdAt": "2024-01-01T10:00:00Z" },
    { "content": "My second post", "createdAt": "2024-01-01T10:05:00Z" }
  ]
}
```

**Important interview caveat:** System design interviews define clear, stable requirements upfront — which removes the primary reason to choose document databases ("schema might change"). Only pick MongoDB if the interviewer explicitly mentions variable or rapidly changing data structures.

**Technologies:** MongoDB, Firestore, CouchDB

### Key-Value Stores — For Simple, Fast Lookups

Fetch values by exact key match. Extremely fast, limited query capability. Schema is essentially flat — you duplicate data across different keys to support different access patterns.

**When to use:**
- Caching (Redis in front of PostgreSQL)
- Session storage (user session ID → session data)
- Feature flags, rate limit counters, leaderboards

**Clarification:** "Over SQL" is misleading — in practice, you often use both. PostgreSQL as your source of truth, Redis in front for hot data.

**Technologies:** Redis, DynamoDB, Memcached

### Wide-Column Databases — For Massive Write Throughput

Rows can have different sets of columns within a column family. Optimized for enormous write volumes and time-series data.

**When to use:**
- Time-series data (IoT sensors, metrics, event logs)
- Analytics workloads with massive append-mostly write patterns
- Systems where you're willing to design rigidly around access patterns in exchange for horizontal scalability

**Data modeling impact:** Design around query patterns even more than SQL. Duplicate data across column families to support different access patterns. Time is a first-class design dimension.

**Technologies:** Cassandra, HBase

### Graph Databases — Almost Never in Interviews

Stores data as nodes and edges, optimized for relationship traversal.

**When to use:** Almost never in a system design interview. Classic use cases (social networks, recommendation engines) are actually solved with SQL at companies like Facebook and LinkedIn. Graph databases add operational complexity that isn't justified by the query benefits.

**Technologies:** Neo4j, Amazon Neptune

---

## Schema Design Fundamentals

### Start with Requirements: Three Factors

Every schema decision flows from three factors:

**Data volume** — How much data exists? 100M users × 1KB = 100GB, fine for one database. 100M users with 200 posts each × 2KB = 40TB, approaching limits. Volume affects whether you need partitioning, archival, or multiple data stores.

**Access patterns** — How will data be queried? A news feed loading "recent posts by followed users" needs denormalized data or well-designed indexes. An analytics dashboard aggregating across time periods needs a different schema entirely. Your access patterns come directly from your API endpoints: ask "what query will I need to run for each endpoint?"

**Consistency requirements** — How tightly coupled must the data be? Financial transactions need strong consistency (no partial charges) — keep related data in the same database with ACID guarantees. Activity feed events can handle eventual consistency — allow data to be spread across systems with async replication.

In interviews, connect schema choices back to these factors explicitly: "Since we need to load feeds quickly and likes can be eventually consistent, I'll denormalize like counts into the posts table."

### Entities, Keys, and Relationships

For each core entity, define:

**Primary key:** The unique identifier for each record. Use system-generated IDs (user_id, post_id) not business data (email addresses). Emails can change; system-generated keys don't.

```
users:    id (PK), username, email, createdAt
posts:    id (PK), user_id (FK → users.id), content, createdAt
comments: id (PK), post_id (FK → posts.id), user_id (FK → users.id), content
likes:    user_id (FK → users.id), post_id (FK → posts.id)  [composite PK]
```

**Relationship types:**
- **One-to-many (1:N):** a user has many posts — foreign key on the "many" side (`posts.user_id`)
- **Many-to-many (N:M):** users like many posts, posts are liked by many users — junction table with two foreign keys (`likes` table above)
- **One-to-one (1:1):** rare in practice; often a sign the two tables should be merged

**Foreign keys** enforce referential integrity at the database level — a post can't reference a user that doesn't exist. At very large scale, some systems drop foreign keys for write performance and enforce integrity at the application layer. Mention this trade-off if scale is discussed.

**Constraints** (`NOT NULL`, `UNIQUE`, `CHECK`) enforce correctness at the database level: emails must be unique, prices must be positive. They add write overhead but protect data quality.

---

## Indexing for Access Patterns

Without an index, every query scans the entire table. Indexes let the database jump to matching records.

Mention indexes explicitly in your schema design — it shows you're thinking about real query performance.

**Index your query columns:**
```
Index on posts.user_id         → quickly find all posts by a user
Index on posts.created_at      → load recent posts chronologically
Composite index on (user_id, created_at) → efficiently load a user's recent posts
```

**Connect indexes to API endpoints:** "The `GET /users/{id}/posts` endpoint queries by user_id sorted by created_at, so I'll create a composite index on `(user_id, created_at DESC)`."

For a deeper understanding of index types (B-trees, LSM trees, geospatial indexes), see the [Database Indexing](database-indexing.md) notes.

---

## Normalization vs Denormalization

**Normalization:** Each piece of information lives in exactly one place. User data lives only in the `users` table; posts reference users via foreign key. Prevents update anomalies — change a username in one place and it's updated everywhere.

**Denormalization:** Deliberately duplicate data across tables for read performance. Store `author_username` in the `posts` table instead of joining to `users` on every post read.

```
Normalized posts:
posts: post_id, user_id (FK), content, created_at
→ To display post: JOIN posts + users

Denormalized posts:
posts: post_id, user_id, author_username, content, created_at  
→ To display post: just read from posts (but if username changes, update every post)
```

**Default:** Start normalized. Denormalize only when you've identified a specific read performance bottleneck.

**When denormalization makes sense:**
- Analytics and reporting systems (aggregating data that changes rarely)
- Event logs and audit trails (snapshots that shouldn't change)
- Heavily read-optimized systems where consistency is less critical (search indexes)

**Alternative to denormalizing your source of truth:** Put a cache in front. Keep the database normalized; store pre-computed joins and aggregations in Redis. The source of truth stays clean; read performance is handled by the cache.

---

## Scaling and Sharding

When data outgrows one machine (>50TB or >10k writes/sec), you need to shard.

**Shard by the primary access pattern:** If you mostly query "posts by user," shard by `user_id`. This keeps all of one user's data on the same shard, avoiding cross-shard queries for the most common case.

**Avoid time-based sharding for write-heavy systems:** If you shard posts by month, all current writes go to the current month's shard — a hot shard. Time-range sharding works better for archival workloads where recent data is read-heavy but writes are spread out.

**Avoid cross-shard queries whenever possible:** If your timeline feature needs "posts from users I follow" and users are on different shards, you'll need to query multiple shards and merge results. This is expensive. Solutions: denormalize the timeline (store a user's feed separately), use an intermediate aggregation service, or accept the performance hit for rare queries.

---

## Interview Workflow

When introducing your data model in an interview:

1. **Identify core entities** during requirements — "For a social media app, the core entities are users, posts, comments, and follows"
2. **Choose database type** and justify: "I'll use PostgreSQL — the data is structured, we need ACID for follow/unfollow operations, and it scales to our estimated volume without sharding initially"
3. **Sketch the schema** with key fields:
   - Include primary keys and foreign keys
   - Include fields the API needs
   - Mark which columns need indexes and why
4. **Note normalization decisions:** "I'll denormalize like_count into the posts table since counting the likes table on every post read would be expensive and counts can be eventually consistent"
5. **Address scaling:** "If we need to shard, we'd shard by user_id since most queries are user-scoped"

---

## Key Trade-offs

| Database type | Strengths | Weaknesses |
|--------------|-----------|------------|
| Relational (SQL) | Complex queries, ACID, flexible | Harder to scale horizontally |
| Document | Flexible schema, nested data | No joins, consistency challenges |
| Key-Value | O(1) lookup speed | Very limited query capability |
| Wide-Column | Massive write scale | Rigid access patterns, no ad-hoc queries |
| Graph | Relationship traversal | Operational complexity, rarely needed |

## Common Gotchas

- **Graph databases in interviews** — avoid unless the interviewer explicitly asks for complex relationship traversal; even social networks use SQL at scale
- **Sharding by created_at** — creates a hot shard for all current writes; use entity IDs instead
- **Forgetting indexes** — saying "we'll query by user_id" without mentioning an index on user_id is a gap; indexes don't happen automatically (except for primary keys)
- **Premature denormalization** — start normalized; denormalize only when you've established a read performance problem, not preemptively
- **All foreign keys at all scales** — valid at moderate scale, but at massive scale some teams drop foreign keys for write throughput and enforce integrity in application code

## Practice Questions

1. Design the data model for a ride-sharing app (like Uber). Identify core entities, sketch key fields and relationships, and propose indexes for the most common queries (nearby drivers, ride history by user).
2. Your social media app's post feed query does a JOIN across 4 tables and takes 200ms. Walk through a denormalization strategy to reduce this to a single-table read.
3. You're designing a multi-tenant SaaS application with 10,000 customers. Each customer has their own isolated dataset. Compare three approaches: separate database per tenant, separate schema per tenant, shared tables with tenant_id column. What are the trade-offs for data modeling?
4. Explain when you would embed a comments array inside a posts document (document DB style) vs. keep comments in a separate collection. What queries are better/worse under each approach?
5. You're building a leaderboard for a game with 10M players, updated in real-time. Design the data model. How does your answer change if the leaderboard only needs to show the top 100?

## One-Line Summary

Default to relational (PostgreSQL), identify entities from your API requirements, design indexes to match your actual query patterns, start normalized and denormalize only for proven read bottlenecks, and shard by the primary access pattern only when storage or write throughput requires it.
