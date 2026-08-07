# ZooKeeper

## The Core Idea

ZooKeeper is a distributed coordination service — a consistent, highly available shared brain for distributed systems. When you have multiple servers that need to agree on something (who's the leader, what's in the config, is this service still alive), ZooKeeper provides a single source of truth that all of them can read from and write to with strong consistency guarantees.

It's not a database. It stores small amounts of coordination metadata — typically under 1MB per entry, and your whole dataset fits in memory. What it provides is the infrastructure for distributed agreement: config management, service discovery, leader election, and distributed locks.

## Mental Model / Analogy

Imagine a chat app where a hundred server instances need to coordinate. Which server is handling which chat room? When a server dies, who detects it? When you update a configuration flag, how does every server learn about it without restarting?

Without ZooKeeper, each solution is a mess: polling a database wastes resources, hard-coded config requires restarts, custom health checks are fragile. ZooKeeper is the bulletin board in the break room — everyone can write to it, everyone gets notified when it changes, and it's always accurate.

The key insight: ZooKeeper doesn't hold your business data. It holds the small facts your system needs to agree on to function: "these nodes are alive," "this config is current," "this server is the leader."

---

## Data Model: ZNodes

ZooKeeper organizes data as a **hierarchical namespace** (like a file system tree):

```
/
├── /config
│   ├── /config/database-url
│   └── /config/feature-flags
├── /services
│   ├── /services/payment-api (ephemeral)
│   ├── /services/payment-api-2 (ephemeral)
│   └── /services/user-api (ephemeral)
└── /locks
    └── /locks/inventory-update
```

Each node in this tree is a **ZNode**. Three ZNode types:

| Type | Lifetime | Sequential? | Use case |
|---|---|---|---|
| **Persistent** | Until explicitly deleted | No | Config, namespace |
| **Ephemeral** | Deleted when client session ends | No | Service registration, failure detection |
| **Sequential** | Persistent or ephemeral, gets a monotonically increasing number appended | Yes | Leader election, distributed locks |

**Ephemeral nodes are the killer feature.** When a service registers an ephemeral node and then crashes (network failure, OOM, power loss), ZooKeeper detects the session timeout and automatically removes the node. No manual cleanup, no zombie entries — the act of deletion itself signals failure to every watcher.

---

## Watches: Push Notifications Without Polling

Clients don't poll ZooKeeper for changes. Instead, they **register a watch** on a ZNode:

```
Client registers: "watch /config/feature-flags"

Someone updates /config/feature-flags
        ↓
ZooKeeper pushes notification to client
        ↓
Client re-reads the ZNode and processes the update
Client re-registers the watch (watches are one-shot)
```

Watches fire once and must be re-registered. This eliminates constant polling while ensuring clients eventually see every change. The tradeoff: if two updates happen before the client re-registers, it will see only the latest value (not every intermediate state).

---

## Key Capabilities

### 1. Configuration Management

```
/config/max-connections  → "100"
/config/feature-flags    → {"dark_mode": true, "new_checkout": false}
```

Every service reads its config from ZooKeeper and registers a watch. When you update a config ZNode, all watchers are notified simultaneously — no restart required. Compare to: environment variables (need restart), config DB tables (polling overhead), shared Redis (not ACID, no push notifications).

**Interview application:** In a microservices architecture, changing a feature flag or database connection string once in ZooKeeper instantly propagates to all running service instances.

### 2. Service Discovery

Services register ephemeral nodes when they start:
```
/services/payment-api     → {"host": "10.0.1.42", "port": 8080}
/services/payment-api-2   → {"host": "10.0.1.43", "port": 8080}
/services/payment-api-3   → {"host": "10.0.1.44", "port": 8080}
```

When a service goes down, its ephemeral node disappears. The API Gateway (or other consumers) watching `/services/payment-api` gets notified and stops routing to it.

No heartbeat polling loop needed. Failure detection is O(1) in time — when the session expires (typically 10-30 seconds), the node is gone.

### 3. Leader Election

All candidates create sequential ephemeral nodes under a common path:
```
/election/candidate-00001  (sequential, ephemeral)
/election/candidate-00002
/election/candidate-00003
```

The candidate with the **lowest sequence number is the leader**. Each non-leader watches the node just below it (not the leader directly, to avoid "herd effect" — everyone rushing to re-elect when the leader changes).

When the leader dies → its ephemeral node is deleted → only the candidate watching it gets notified → that candidate checks if it now has the lowest number → if yes, it becomes the new leader.

This is the pattern that Kafka historically used (before KRaft) for broker leader election.

### 4. Distributed Locks

The same sequential ephemeral node pattern works for locks:
```
/locks/resource-42/lock-00001  (ephemeral, sequential)
/locks/resource-42/lock-00002
/locks/resource-42/lock-00003
```

The holder of the lowest-numbered node holds the lock. Others watch the node just below them and wait. When the lock holder releases (deletes its node) or dies (session expires), the next one takes over.

**ZooKeeper vs Redis for locks:**
- ZooKeeper: strict ordering, hierarchical lock structures, watches on the lock hierarchy, no split-brain risk from async replication. Use when correctness is paramount and the lock might be held for seconds or minutes
- Redis: much higher throughput, simpler setup, lower latency. Use when the lock is held for milliseconds and performance matters more than absolute correctness

---

## How ZooKeeper Works

### Ensemble and Quorum

ZooKeeper runs as an **ensemble** of 3, 5, or 7 servers (always odd — to get a clear majority). One server is elected **leader** by the ensemble; the rest are **followers**.

**All writes go through the leader.** Reads can go to any follower (fast, but may be slightly stale). Use the `sync` operation before a critical read to ensure you're reading the latest committed value.

**Writes require quorum:** a write is committed only when a majority of servers (⌊n/2⌋ + 1) have acknowledged it. With 5 servers, 3 must ack. This means the ensemble can tolerate losing any minority of servers without losing write availability.

### ZAB Protocol (ZooKeeper Atomic Broadcast)

ZooKeeper uses **ZAB** for consensus — not Paxos, but similar in intent:

**Phase 1 — Leader Election:** When the ensemble starts or loses a leader, servers elect a new leader. The candidate with the highest transaction ID (most up-to-date log) wins. This ensures no committed data is lost during leader transitions.

**Phase 2 — Atomic Broadcast:** The leader receives all write requests, assigns each a unique zxid (transaction ID), and broadcasts it to followers. When a majority ack, the leader commits and notifies followers. This guarantees all followers see writes in the same order.

### Consistency Guarantees

ZooKeeper provides:
- **Sequential consistency**: all updates appear to happen in the order they were sent by each client
- **Atomicity**: updates succeed or fail completely
- **Single system image**: a client sees the same view regardless of which follower it connects to (within a session)
- **Durability**: committed writes survive server restarts
- **Timeliness**: within a configurable timeout, clients see the latest committed writes

Reads from followers are fast but can be slightly stale. For read-your-own-writes consistency, use `sync()` before the read.

### Sessions and Failure Handling

Clients maintain a **session** with the ensemble via heartbeats. If a client stops heartbeating (network partition, crash), the session expires after the configured timeout (typically 10-30 seconds) and all that client's ephemeral nodes are deleted.

ZooKeeper persists state via a **transaction log** (WAL) and periodic **snapshots**. On restart, it replays the log from the last snapshot. Use a dedicated disk for the transaction log — competing I/O is a major performance killer.

### Limitations

**Hot spotting:** Many clients watching the same ZNode (common in leader election and locks) can cause a thundering herd when that node changes. Scale-sensitive designs must account for this.

**Performance ceiling:** Writes are sequentially ordered through the leader and require majority quorum — throughput is limited. ZooKeeper isn't designed for high write volumes. Keep ZNodes small (under 1MB; ideally much smaller). Everything must fit in memory.

**Operational complexity:** Java configuration, JVM tuning, dedicated disks, monitoring session timeouts — ZooKeeper is "simple to use but complex to operate." One maintainer's words, not mine.

---

## ZooKeeper in the Modern World

ZooKeeper was introduced in 2008 and powered the distributed systems ecosystem for over a decade. Things have shifted:

**Where ZooKeeper is still used:**
- Apache HBase, Hadoop, SolrCloud, Storm, NiFi, Pulsar — all built on ZooKeeper
- ClickHouse uses ZooKeeper for replication coordination and DDL execution

**Kafka's transition to KRaft:** Kafka removed ZooKeeper as a dependency in Kafka 3.x (KRaft mode). ZooKeeper was a significant operational burden — separate cluster to manage, scalability bottleneck for metadata, extra failure point. KRaft uses a Raft-based protocol built into Kafka itself.

**Modern alternatives:**

**etcd** — powers Kubernetes. Distributed key-value store with strong consistency, modern HTTP/JSON and gRPC APIs, optimized for small datasets and high read volumes. The cloud-native default for config management and service discovery.

**Consul** (HashiCorp) — extends beyond ZooKeeper's scope with built-in health checking, service mesh, and network infrastructure automation. Dynamically configures load balancers and firewalls based on service state. More comprehensive than ZooKeeper's focused coordination.

**Cloud provider solutions** — AWS Parameter Store + ECS CloudMap, Azure App Configuration, Google Cloud Datastore. Fully managed with zero ops burden; deepens vendor lock-in but eliminates entire operational domains.

---

## When to Use ZooKeeper (In Your Interview)

**Don't reach for ZooKeeper by default.** It's most relevant in:

**Deep infrastructure design problems:** "Design a distributed message queue" or "Design a distributed task scheduler" — ZooKeeper is the coordination brain. In a distributed queue design, ZooKeeper handles broker registration (ephemeral nodes), partition leader election (sequential nodes), consumer group membership tracking, and broker failure detection (session expiration). This is the pattern Kafka used for years before KRaft.

**Smart routing with colocation constraints:** In a real-time chat or live video app, users in the same room should connect to the same server to minimize cross-server fan-out. ZooKeeper stores the mapping of room → server. The API Gateway queries ZooKeeper on each new connection to route the user to the correct server. When a server reaches capacity, ZooKeeper coordinates expanding the room to an additional server.

**Hierarchical distributed locks:** When resources have parent-child relationships (a distributed file system where locking a directory should lock all its files), ZooKeeper's hierarchical watch model enables correct nested lock acquisition with deadlock prevention. Redis doesn't handle this case well.

**Not worth it for:**
- Simple config management → environment variables or a managed parameter store is fine
- Service discovery in cloud environments → your cloud provider's service registry handles this
- Basic distributed locks in low-stakes scenarios → Redis SET NX EX is simpler and faster

---

## Practice Questions

1. You're designing a chat app where WebSocket connections must be colocated per room for efficiency (minimize cross-server pub/sub). Walk through how ZooKeeper would enable smart routing — what ZNodes exist, what watches are registered, and what happens when the server assigned to a room crashes.
2. Five services compete to become the leader for processing a work queue. Walk through the ZooKeeper sequential ephemeral node pattern for leader election, including what happens when the current leader's server fails mid-processing.
3. Kafka used ZooKeeper for years before introducing KRaft. What specifically did ZooKeeper handle in the Kafka architecture, and what was the motivation to replace it?
4. Compare ZooKeeper and etcd for service discovery in a Kubernetes-style environment. When would you prefer each?
5. You're using ZooKeeper for distributed locking in a distributed file system where operations on directories must lock all contained files. Why is Redis an insufficient solution here, and how does ZooKeeper's watch model on multiple nodes solve it?

---

## One-Line Summary

ZooKeeper is a battle-tested distributed coordination service for leader election, service discovery, config management, and hierarchical locks — reach for it in deep infrastructure design problems or when your modern alternatives (etcd, Consul, cloud-managed services) aren't available.
