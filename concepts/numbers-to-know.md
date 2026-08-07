# Numbers to Know

## The Core Idea

System design interviews require reasoning about scale. Without reference numbers, you can't tell if a proposed design actually works — can a single database handle this traffic? Is 100GB a lot for a cache to store? Do we need to shard? Having rough hardware benchmarks in your head lets you do back-of-envelope math that grounds your decisions in reality.

These are 2026 reference numbers. Treat them as order-of-magnitude anchors, not precise guarantees.

## Mental Model / Analogy

Think of these like a chef's unit conversion chart. A chef doesn't calculate tablespoons to gallons every time — they know that a liter is about a quart, a pound is about 450 grams, a cup is 8 oz. These aren't exact for every situation, but they're accurate enough to plan a recipe. Similarly, "a Redis instance handles ~100k ops/sec" isn't exact for your workload, but it's accurate enough to decide whether you need one Redis node or ten.

---

## Servers

### High-Memory Instances (Keep in mind for caching and in-memory DBs)

| AWS Instance | RAM | vCPUs | Notes |
|-------------|-----|-------|-------|
| M6i.32xlarge | 512 GiB | 128 | General purpose workhorse |
| X1e.32xlarge | 4 TB | 128 | In-memory databases, huge caches |
| U-24tb1.metal | 24 TB | 448 | Extreme in-memory workloads (SAP HANA) |

### High-Storage Instances

| AWS Instance | Local Storage | Notes |
|-------------|--------------|-------|
| i3en.24xlarge | 60 TB NVMe SSD | Fast local SSD, great for read-heavy |
| D3en.12xlarge | 336 TB HDD | Dense storage, cold archival data |
| S3 | Unlimited | Object storage, infinite but higher latency |

### Network

- **Standard network:** ~25 Gbps (about 3 GB/s throughput per server)
- **High-performance network:** 50-100 Gbps (for data-intensive workloads)
- **Network latency within one AZ:** sub-1ms
- **Cross-AZ latency (same region):** 1-2ms
- **Cross-region latency:** 50-150ms (physics of fiber optic cables at ~200,000 km/s)

**Why latency matters:** A query involving 5 cross-AZ hops adds 5-10ms. A design with 3 cross-region hops is adding 150-450ms before any computation.

---

## Caching (Redis / Memcached)

### Capacity

- **Memory per instance:** up to ~1 TB per Redis node
- **Latency:** sub-1ms reads (typically 0.1-0.5ms)
- **Throughput:** 100,000-200,000 ops/sec per Redis instance (Graviton-based, pipelining)

### When to Scale Out (Add More Redis Nodes)

| Trigger | Threshold |
|---------|-----------|
| Dataset size | > 1 TB |
| Throughput | > 100k ops/sec consistently |
| Latency | Requirement < 0.5ms (may need in-process cache instead) |

### What This Means in Practice

If you have 10M users each with a 1KB session: 10 TB of session data. That won't fit on one Redis node. You need Redis Cluster (sharding across multiple nodes).

If you have 10M users but only 100K active at any time with 1KB sessions: 100 GB, easily fits on one Redis node with room to spare.

---

## Databases (PostgreSQL / MySQL / Aurora)

### Capacity

- **Storage per instance:** up to 64 TiB (Aurora scales to 256 TiB)
- **Read latency (cached):** 1-5ms
- **Read latency (disk):** 5-30ms
- **Write latency (committed):** 5-15ms per transaction

### Throughput

| Metric | Approximate limit |
|--------|------------------|
| Reads per second | Up to 50,000 TPS (Aurora/RDS with read replicas) |
| Writes per second | 10,000-20,000 TPS (Aurora) |
| Concurrent connections | 5,000-20,000 (with PgBouncer connection pooling) |

### When to Shard

| Trigger | Threshold |
|---------|-----------|
| Storage | > 50 TiB of data |
| Write throughput | > 10,000 writes/sec consistently |
| Query complexity | Queries taking > 100ms despite good indexing |

### What This Means in Practice

A social media app with 100M users × 1KB average profile = 100 GB. **No need to shard — easily fits in one database.**

The same app with 100M users each having 200 posts × 2KB per post = 40 TB. **Approaching the limit. Consider sharding.**

---

## App Servers

### Capacity

| Metric | Approximate value |
|--------|------------------|
| Concurrent connections | 100,000+ (with async frameworks like Node.js, Go, Nginx) |
| CPU cores | 8-64 cores per instance |
| RAM | 64-512 GB (up to 2 TB on large instances) |
| Network | 25 Gbps standard |

### Operations

- **Container startup time:** 30-60 seconds (relevant for auto-scaling responsiveness)
- **P99 response target:** < 100ms for most API endpoints

### When to Scale

| Trigger | Threshold |
|---------|-----------|
| CPU utilization | > 70-80% sustained |
| Memory utilization | > 70-80% |
| Response latency | Exceeds your SLA |
| Error rate | Rising |

---

## Message Queues (Kafka)

### Capacity

| Metric | Approximate value |
|--------|------------------|
| Messages per second per broker | Up to 1 million msgs/sec |
| End-to-end latency | 1-5ms |
| Message size | 1KB to 10MB per message |
| Storage per broker | Up to 50 TB |
| Retention | Days to weeks typically |

### When to Scale

| Trigger | Threshold |
|---------|-----------|
| Throughput | Approaching 800k msgs/sec per broker |
| Partition count | Approaching 200k per cluster |
| Consumer lag | Growing consistently (consumers can't keep up) |

### What This Means in Practice

A logging pipeline receiving 100k events/sec with 1KB events = 100MB/s. One Kafka broker at ~10% capacity. Fine.

The same pipeline at 800k events/sec = 800MB/s. Time to add brokers or optimize consumers.

---

## Quick Math Shortcuts

**Storage sizing:**
- 1 million items × 1KB = 1 GB
- 1 billion items × 1KB = 1 TB
- 1 billion items × 1MB = 1 PB

**Throughput sizing:**
- 1M users, each making 10 requests/day = 10M requests/day = ~115 requests/second
- 1M DAU, each spending 30 min/day × 1 request/10s = 1M × 180 requests/day = ~2,000 requests/second peak
- Assume 2-3× peak vs average for sizing headroom

---

## Common Interview Mistakes

### 1. Premature Sharding

Candidates hear "100 million users" and immediately say "we need to shard." Run the numbers first:
- 100M users × 1KB = 100GB — fits in one database with room for 50TB more
- 100M users × 10KB = 1TB — still fits in one database (Aurora supports 256TB)
- 100M users, 1000 writes/day = 1,000 WPS — well within single-database range (10-20k WPS capacity)

**Rule:** Don't propose sharding until you've established storage > 50TB or writes > 10k/sec.

### 2. Overestimating Latency

SSD-based databases with good indexing return results in 1-5ms. Candidates sometimes add a Redis cache "for performance" when the underlying database query would already be 2ms. A cache that saves 2ms adds 2ms of cache management overhead and is net-negative.

**Rule:** Cache only when DB latency exceeds your requirement, not preemptively.

### 3. Over-Engineering Writes

Candidates hear "we need to handle 5,000 writes per second" and immediately propose Kafka. PostgreSQL handles 10,000-20,000 writes/sec. 5k WPS is comfortably within range.

**Rule:** Don't add Kafka unless the write rate exceeds DB write capacity, you need async processing, or you need event replay capabilities.

---

## Practice Questions

1. A photo-sharing app has 50M users. Each user uploads an average of 20 photos, each stored at 2MB (after compression). Calculate total storage. Do you need object storage (like S3) or can a database handle this?
2. A notification service sends push notifications to 100M users when a major event happens. They all trigger within 60 seconds. How many messages per second is that, and can a single Kafka broker handle it?
3. Your API receives 10M requests per day. Peak traffic is 5× average. Calculate requests per second at peak. How many app servers do you need assuming each handles 1,000 concurrent connections?
4. You're building a session store for 5M concurrent logged-in users, each with 2KB of session data. Size the Redis requirement. How does this change if it's 50M concurrent users?
5. A financial system processes 3,000 transactions per second. Each transaction is 500 bytes and must be retained for 7 years. Calculate: (a) annual storage growth, (b) whether one PostgreSQL instance can handle the write throughput.

## One-Line Summary

Know that a single database handles 50 TiB and 10-20k WPS, a Redis node handles 100k ops/sec with 1 TB memory, and a Kafka broker handles 1M msgs/sec — run these numbers first before sharding, caching, or adding queues.
