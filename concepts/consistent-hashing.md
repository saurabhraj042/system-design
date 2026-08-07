# Consistent Hashing

## The Core Idea

Consistent hashing is a technique for distributing keys across a set of servers so that when a server is added or removed, only a small fraction of keys need to move. Regular modulo hashing (assign key to `hash(key) % N`) requires almost all data to relocate when N changes. Consistent hashing limits reshuffling to just the keys that were previously owned by the changing node.

It's the fundamental technique behind distributed databases, caches, and CDNs — any system where you need to split data across many nodes and add or remove nodes without full data migration.

## Mental Model / Analogy

Picture a circular subway line with stops numbered 0 to 100. Four database servers are placed at stops 15, 40, 65, and 90. When a piece of data arrives, you hash it to get a stop number (say, stop 72) and walk clockwise until you reach the next server — DB4 at stop 90 owns stop 72's data.

When you add a new server at stop 80: only data that hashed between stops 65 and 80 needs to move from DB4 to the new server. Everything else stays exactly where it is. Remove DB1 at stop 15: only DB1's data (stops 90-15) moves to DB2 at stop 40.

Compare to a straight number line with modulo: changing from 4 to 5 servers changes the formula from `hash(key) % 4` to `hash(key) % 5`, which reassigns almost every key to a different server.

---

## How It Works

### The Hash Ring

Arrange both servers and data keys in a circular hash space (conceptually 0 to 2^32-1 or 0 to 2^64-1 in practice):

```
              0 / 2^32
                 │
              DB4 (90°)
       DB1(15°)  ●    
      ●               ●  DB3 (65°)
         DB2 (40°)
              ●
```

When a key arrives:
1. Hash the key to get a position on the ring (e.g., key "user:1234" → position 55)
2. Walk clockwise to find the nearest server
3. That server owns the key (DB3 at 65 owns keys hashing to 55)

### Adding a Node

New DB5 placed at position 80:
- Keys that previously hashed between 65 and 80 were on DB3 (at 90)
- Now those keys move to DB5
- Everything else stays unchanged
- Data movement ≈ `1/N` of total data (where N is the new total number of nodes)

### Removing a Node

DB1 at position 15 is taken offline:
- All keys that were on DB1 (hashing to positions 90-15 clockwise) move to DB2
- Everything else is unaffected
- Data movement ≈ `1/N` of total data

---

## Virtual Nodes: Solving Uneven Distribution

Basic consistent hashing has a problem: each physical node owns one contiguous arc of the ring, so arc lengths are uneven. One node might own 30% of the ring; another only 5%.

**Virtual nodes (vnodes)** solve this by giving each physical server multiple positions on the ring:

```
DB1 maps to ring positions: 15, 38, 72, 91
DB2 maps to ring positions: 8, 29, 55, 84
DB3 maps to ring positions: 19, 44, 67, 96
```

Each physical node owns many small arcs rather than one large one. As the number of virtual nodes per server increases, distribution becomes statistically more uniform.

**Additional benefits of vnodes:**

When DB2 fails: its 4 virtual node positions are distributed to different neighbors — some go to DB1, some to DB3. Load spreads evenly across the remaining cluster instead of all crashing into one neighbor.

When a new server joins: it takes a few arcs from each existing server, so the data migration is spread across the cluster. No single server does all the work of handing off data.

Cassandra uses vnodes. The number of vnodes per server (`num_tokens` in Cassandra config) controls how finely the load is balanced — higher values mean more even distribution but more metadata overhead.

---

## Still Not a Complete Solution: Hot Spots

Consistent hashing distributes **keys** evenly, but not necessarily **traffic**. A key that receives 100× more requests than average (Taylor Swift's user account during a concert announcement) will always route to the same server, regardless of how well keys are distributed on the ring.

**Fix 1 — Read replicas:** Replicate hot keys to multiple servers. Reads can be served from any replica.

**Fix 2 — Key-space salting:** Instead of storing `user:taylorswift`, store `user:taylorswift:0` through `user:taylorswift:9`. Reads and writes are randomly distributed across 10 different ring positions, and thus potentially 10 different servers.

**Fix 3 — Adaptive rebalancing:** DynamoDB does this automatically. It monitors per-partition throughput and dynamically adjusts partition ownership (effectively moving virtual nodes) to balance hot partitions.

---

## How Data Actually Moves: The Replication Story

In practice, data movement during node changes is often handled differently than the pure ring model suggests:

**Node failure in production systems:** Replication handles this. DynamoDB replicates each partition to 3 availability zones using Raft consensus. Cassandra replicates to N consecutive ring neighbors (configured via replication factor). When a node fails, a replica already has the data — nothing needs to move.

**Planned capacity changes:** When adding servers to expand capacity, data does physically migrate. But modern systems do this incrementally in the background with rate limiting, so the cluster stays responsive during reshuffling.

---

## Real-World Usage

| System | Approach |
|--------|----------|
| **Cassandra** | Hash ring with Murmur3, vnodes for balance, configurable replication factor |
| **DynamoDB** | Consistent hashing internally; AWS abstracts the ring from users entirely |
| **CDNs (Akamai, Cloudflare)** | Consistent hashing to route requests to nearby edge nodes |
| **Redis Cluster** | NOT consistent hashing — uses 16,384 fixed hash slots (CRC16(key) mod 16384) assigned to nodes. Different trade-off: simpler but requires config updates when nodes change |

Redis Cluster's hash slot approach is worth knowing: it's deterministic and avoids the need for virtual nodes, but moving slots during cluster resize is an explicit config operation, not automatic.

---

## Interview Application

Bring up consistent hashing when you're designing any distributed system with many nodes where you need to:
- Partition data across multiple databases or cache nodes
- Add/remove capacity without mass migration
- Route requests to the correct node without a central routing table

Common trigger questions:
- "How would you shard a distributed cache?"
- "How does DynamoDB decide which server handles a given key?"
- "What happens to the system when you add a new database node?"

---

## Key Trade-offs

| What you gain | What you lose |
|---------------|---------------|
| Minimal data movement on node changes | Traffic hot spots still possible |
| No central routing table (decentralized) | Virtual node metadata overhead |
| Automatic load balancing with vnodes | Uneven arcs with basic version |
| Works for any key-based distributed system | Implementation is more complex than modulo |

## When to Use

- Building a distributed cache (Redis Cluster, Memcached cluster)
- Designing a distributed key-value database
- Load balancing stateful services where a key must always go to the same server
- CDN edge routing where nearby servers are preferred

## Common Gotchas

- **Confusing key distribution with traffic distribution** — even key spread doesn't prevent hot keys; those need separate solutions
- **Forgetting replication means no data movement on failure** — in production, replicas handle node failure without the ring reshuffling
- **Redis Cluster uses hash slots, not a ring** — know the distinction if Redis comes up
- **Virtual nodes aren't free** — each additional vnode adds metadata overhead; the right number depends on cluster size and distribution requirements

## Practice Questions

1. Your Redis cluster has 6 nodes using consistent hashing. You want to add 2 more nodes. Walk through what data moves, which nodes are involved, and how you'd minimize the impact on production traffic.
2. Explain why `hash(key) % N` is problematic when N changes, using a concrete example with 3 items and going from 3 nodes to 4 nodes.
3. DynamoDB auto-scales your partition when it hits throughput limits. How does this relate to consistent hashing? What's the mechanism that allows this to work transparently to your application?
4. Cassandra with replication factor 3 and 5 nodes uses consistent hashing. When node 3 fails, what happens to: (a) reads for keys it owned, (b) writes for keys it owned, and (c) the ring topology?
5. You're designing a distributed rate limiter. Each user's request count must go to the same server to count correctly. How would consistent hashing help here? What's the failure mode if a server goes down?

## One-Line Summary

Consistent hashing places both data and servers on a circular ring so that adding or removing a server only relocates ~1/N of the data rather than rehashing everything — solve residual hot spots with virtual nodes and key-space salting.
