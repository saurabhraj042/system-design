# Kafka

## The Core Idea

Kafka is a distributed, append-only log that producers write to and consumers read from. It can act as a message queue (each message processed by one consumer) or an event stream (multiple independent consumers read the same log). Its superpower is durability + throughput at scale: millions of messages per second, replicated across machines, with configurable retention so consumers can replay history.

## Mental Model / Analogy

Imagine a newspaper publishing operation. The newsroom (producers) generates stories continuously. The paper is divided into sections: Sports, Finance, Tech (topics). Each section is printed across several pages (partitions) so multiple printing presses can work in parallel. Subscribers (consumers) pick which sections they care about. Your bookmark (offset) marks exactly where you stopped reading — drop it and pick up right where you left off even after a week away.

The key: once a story is printed, it stays in the archive (retention policy). Multiple departments can read the same story independently and at their own pace — the Sports desk reading it doesn't remove it from what the Analytics team will read later.

---

## Core Concepts

| Concept | What it is |
|---|---|
| **Broker** | A single Kafka server. A cluster has multiple brokers |
| **Topic** | Logical category for messages (like a named channel) |
| **Partition** | Physical append-only log within a topic; ordered sequence of messages |
| **Offset** | Sequential ID for a message within a partition — how consumers track position |
| **Producer** | Writes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | Set of consumers where each partition is assigned to exactly one member |

**Topic vs partition:** A topic organizes data logically. A partition scales it physically — you can put different partitions of the same topic on different brokers, letting them be read in parallel.

---

## How It Works

### Message Structure

Every Kafka message (record) has:
- **Value** — the payload (required in practice)
- **Key** — determines which partition the message lands in (hash(key) % num_partitions)
- **Timestamp** — when it was created
- **Headers** — optional metadata key-value pairs

**The key is the most important design decision you'll make.** If you don't set a key, Kafka distributes messages round-robin. If you do, all messages with the same key always land in the same partition — giving you ordering guarantees for that key.

### Partitions Are Append-Only Logs

```
Partition A: [msg0] [msg1] [msg2] [msg3] → always appended, never modified
                ↑                    ↑
           offset=0             offset=3
```

Consumers track their position by committing offsets back to Kafka. If a consumer crashes, it restarts from the last committed offset. This gives **at-least-once delivery**: a message might be reprocessed if the consumer crashes after processing but before committing.

### Consumer Groups

Multiple consumers can subscribe to the same topic independently — each consumer group gets its own copy of every message. Within a group, each partition is assigned to exactly one consumer (no duplicate processing within the group).

```
Topic: user-events (3 partitions)

Consumer Group A (analytics):    P0→C1, P1→C2, P2→C3
Consumer Group B (notifications): P0→C4, P1→C4, P2→C5
```

If you add a 4th consumer to group A but there are only 3 partitions, the 4th consumer sits idle — you can never have more active consumers in a group than partitions.

### Replication

Each partition has one leader (handles all reads/writes) and N-1 follower replicas on other brokers. On crash, a follower becomes the new leader.

Producer acknowledgment modes:
- `acks=0` — fire and forget (fastest, messages can be lost)
- `acks=1` — leader acknowledges (lost if leader crashes before replication)
- `acks=all` — all in-sync replicas (ISR) acknowledge (strongest guarantee)

---

## When to Use Kafka

### As a Message Queue
Use when you need:
- **Async processing** — decouple slow work (video transcoding, ML inference) from the fast API path
- **Ordered processing per entity** — partition by user ID or order ID for per-entity ordering
- **Producer-consumer rate decoupling** — producers can burst ahead; consumers drain at their own pace

### As an Event Stream
Use when you need:
- **Fan-out to multiple independent consumers** — analytics, fraud detection, notifications all read the same events
- **Replay** — new consumer can catch up from the beginning or any point in history
- **Real-time aggregation** — window over a stream of clicks, transactions, sensor readings

---

## Scaling

### Partitioning Strategy

Your partition key is your primary scaling lever:

```
partition = hash(key) % num_partitions
```

A bad key → hot partitions → one broker overwhelmed while others are idle.

Good partition keys are **high-cardinality and evenly distributed**: user_id, order_id, session_id. Bad keys: country (a few dominate), status (only 3-4 values).

### Handling Hot Partitions

When one key gets disproportionate traffic (a viral post, a celebrity account):

| Strategy | Tradeoff |
|---|---|
| No key (round-robin default) | Loses ordering guarantees |
| Random salting (key + random suffix) | Spreads load; consumer must re-aggregate across partitions |
| Compound key (key + region/shard) | Better distribution; slightly more complex routing |
| Back-pressure (slow the producer) | Simple; only works if producer can tolerate it |

### Sizing a Broker

A single broker can handle roughly 1TB of storage and ~1M messages/second (hardware-dependent). If your design stays under that, you likely don't need to optimize partitioning. If you exceed it, add brokers and increase partition count.

**Don't store large blobs in Kafka** — messages should be metadata/events (under 1MB ideally), not the actual files. Store files in S3 and put the S3 path in Kafka.

---

## Fault Tolerance

### When a Broker Dies

The controller reassigns that broker's leader partitions to one of its in-sync followers. With `acks=all` and `replication_factor=3`, you can lose a broker with zero data loss.

### When a Consumer Dies

Kafka tracks the last committed offset. The consumer group rebalances — remaining consumers take over the failed consumer's partitions. Messages after the last committed offset get reprocessed (at-least-once). If exact-once matters: use idempotent producers + transactional APIs.

### Retry Patterns

**Producer retries:** Built-in with backoff. Enable `idempotent=true` to prevent duplicate messages when retrying.

**Consumer retries:** Kafka doesn't retry for you — implement it yourself:
```
Main Topic → fails → Retry Topic → fails again after N retries → DLQ Topic
```
A separate consumer processes the retry/DLQ topics. This avoids blocking the main consumer on a poison message.

---

## Retention Policies

Kafka is not a database — messages don't live forever by default. Two options:
- **Time-based**: `retention.ms` (default: 7 days)
- **Size-based**: `retention.bytes`

For event sourcing or audit logs, extend retention significantly. For pure job queues, short retention is fine. Beware: storage cost scales with retention × throughput.

---

## Key Trade-offs

| What you gain | What you lose |
|---|---|
| Decoupling: producers and consumers scale independently | Operational complexity: brokers, ZooKeeper/KRaft, monitoring |
| Durability: replication protects against broker loss | Exactly-once requires careful configuration |
| Fan-out: many consumer groups from one topic | Kafka is always available, sometimes consistent |
| Replay: rewind and reprocess from any offset | Not a good fit for large blobs or complex queries |

---

## When NOT to Use Kafka

- Simple job queues where SQS or Redis Streams would work and you don't need replay
- Short-lived events that don't need fan-out (pick SQS/RabbitMQ, save the operational burden)
- Storing large files — Kafka messages should be tiny pointers to files stored elsewhere

---

## Practice Questions

1. You're building a notification system where every user action (like, comment, follow) should trigger notifications. How would you partition the Kafka topic and why?
2. Your ad-click aggregation pipeline uses Kafka with `ad_id` as the partition key. One ad from a major campaign gets 10× the clicks of all others combined. Walk through the hot partition problem and three ways to solve it.
3. A consumer processes payment events from Kafka. It marks a payment as complete in the database, then crashes before committing its Kafka offset. What happens when the consumer restarts, and how do you make the outcome safe?
4. Explain the difference between Kafka's at-least-once and exactly-once delivery semantics. When would you pay the cost of exactly-once?
5. When would you use Redis Streams instead of Kafka, and when would you pick Kafka over Redis Streams?

---

## One-Line Summary

Kafka is an append-only distributed log: pick your partition key carefully (ordering lives there), use `acks=all` for durability, and choose between message-queue mode (one consumer group drains it) and stream mode (multiple consumer groups each read independently).
