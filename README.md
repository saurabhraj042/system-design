# System Design Notes

A structured set of study notes for software engineering system design interviews. Everything here is written in student voice — paraphrased concepts, original analogies, and self-authored practice questions. Each file is designed to be read in under 30 minutes and used as a reference during active interview prep.

---

## How to Use This Repo

1. **Start with Concepts** — these are the building blocks. Read them in any order, but cover all of them before moving to Patterns.
2. **Move to Patterns** — each pattern is a recurring problem that shows up across many system design questions. Learn to recognize when a problem calls for a pattern.
3. **Use Technologies as a reference** — don't memorize these upfront. Come back when a concept or pattern mentions a specific technology and you want to understand it more deeply.
4. **Practice with the questions** — every file ends with 5 practice questions. Write out your answers; don't just read and assume you could answer them.

---

## Concepts

Core ideas that appear everywhere in system design. Every senior engineer is expected to know these fluently.

| Topic | What You'll Learn |
|-------|------------------|
| [Networking Essentials](concepts/networking-essentials.md) | TCP vs UDP, HTTP/1.1/2/3, WebSockets, SSE, WebRTC, load balancing (L4 vs L7), CDNs, failure handling (retries, circuit breakers) |
| [API Design](concepts/api-design.md) | REST, GraphQL, gRPC — when to use each; pagination, versioning, authentication, rate limiting |
| [Data Modeling](concepts/data-modeling.md) | Choosing the right database type; schema design; normalization vs denormalization; indexing for access patterns; sharding strategies |
| [Database Indexing](concepts/database-indexing.md) | B-trees, LSM trees, hash indexes, geospatial indexes, inverted indexes; composite and covering indexes; write trade-offs |
| [Caching](concepts/caching.md) | Cache-aside, write-through, write-behind; TTL; eviction policies; cache invalidation strategies |
| [Sharding](concepts/sharding.md) | Horizontal vs vertical partitioning; shard key selection; consistent hashing; resharding |
| [Consistent Hashing](concepts/consistent-hashing.md) | Hash rings; virtual nodes; why modulo hashing breaks during scaling |
| [CAP Theorem](concepts/cap-theorem.md) | Consistency, availability, partition tolerance; CP vs AP systems; what the theorem actually means in practice |
| [Numbers to Know](concepts/numbers-to-know.md) | Hardware benchmarks for 2026: database TPS, Redis ops/sec, Kafka throughput, storage limits — the numbers you need to sanity-check your designs |

---

## Patterns

Recurring architectural problems that interviewers probe deeply. Each pattern covers when to apply it, how it works, and the deep-dive questions you're likely to face.

| Pattern | When It Comes Up |
|---------|-----------------|
| [Scaling Writes](patterns/scaling-writes.md) | High write throughput; social media activity; IoT sensors; event logging; any system where writes are the bottleneck |
| [Scaling Reads](patterns/scaling-reads.md) | High read traffic; slow queries; feed generation; search; any system where read latency or volume is the bottleneck |
| [Real-Time Updates](patterns/real-time-updates.md) | Chat; live scores; notifications; collaborative editing; any system where clients need data pushed to them |
| [Multi-Step Processes](patterns/multi-step-processes.md) | Payment flows; order fulfillment; approval workflows; any operation that spans multiple services and must survive failures |
| [Dealing with Contention](patterns/dealing-with-contention.md) | Race conditions; double-booking; ticket reservations; inventory management; anywhere two users compete for the same resource |
| [Managing Long-Running Tasks](patterns/managing-long-running-tasks.md) | Video processing; report generation; ML inference; any operation too slow to complete in an HTTP request |
| [Handling Large Blobs](patterns/handling-large-blobs.md) | File uploads; image storage; video hosting; any system dealing with multi-megabyte or multi-gigabyte objects |

---

## Technologies

Deep dives on specific technologies. Use these when you need to justify a technology choice in an interview or understand how a tool works under the hood.

| Technology | Use Case |
|-----------|---------|
| [Redis](technologies/redis.md) | Caching, pub/sub, rate limiting, session storage, leaderboards |
| [Kafka](technologies/kafka.md) | Event streaming, async decoupling, log aggregation, real-time pipelines |
| [PostgreSQL](technologies/postgresql.md) | Relational data, ACID transactions, general-purpose primary database |
| [Cassandra](technologies/cassandra.md) | Write-heavy workloads, time-series, globally distributed data |
| [DynamoDB](technologies/dynamodb.md) | Serverless key-value/document store, AWS-native, single-digit ms at any scale |
| [Elasticsearch](technologies/elasticsearch.md) | Full-text search, log analytics, faceted filtering |
| [Zookeeper](technologies/zookeeper.md) | Distributed coordination, leader election, configuration management |
| [API Gateway](technologies/api-gateway.md) | Request routing, rate limiting, auth, protocol translation at the edge |

---

## Interview Approach

**Requirements (5–10 min):** Ask about scale (users, requests/sec, data size), consistency requirements, and latency targets. Write these down. Every design decision should reference back to them.

**High-level design (10–15 min):** Sketch the major components — clients, load balancers, services, databases, caches. Identify where each pattern applies. Don't over-specify yet.

**Deep dives (15–20 min):** The interviewer will pick 1–2 areas to probe. Common targets: database choice and indexing, caching strategy, how you handle failures, scaling the bottleneck component. These are where the pattern notes pay off.

**Sanity-check with numbers:** Use the [Numbers to Know](concepts/numbers-to-know.md) reference to validate your design. A single Redis node handles 100k ops/sec. A single PostgreSQL instance handles 10–20k writes/sec. If your back-of-envelope math exceeds these, you need more nodes or a different strategy.

---

## Suggested Study Order

**Week 1 — Foundations**
- Networking Essentials → API Design → Data Modeling → Database Indexing

**Week 2 — Scaling Concepts**
- Caching → Sharding → Consistent Hashing → CAP Theorem → Numbers to Know

**Week 3 — Patterns**
- Scaling Reads → Scaling Writes → Real-Time Updates

**Week 4 — Advanced Patterns + Technologies**
- Multi-Step Processes → Dealing with Contention → Managing Long-Running Tasks → Handling Large Blobs
- Read relevant technology deep dives as they come up in patterns
