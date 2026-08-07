# CAP Theorem

## The Core Idea

In a distributed system, you can only guarantee two of three properties simultaneously:
- **Consistency (C):** Every read returns the most recent write (or an error)
- **Availability (A):** Every request to a non-failing node gets a response (though it might not be the latest data)
- **Partition tolerance (P):** The system continues operating when the network splits and nodes can't communicate

The critical insight: **network partitions will happen** in any real distributed system. Cables get cut, routers fail, datacenters lose connectivity. You must design for P. So the actual choice is between **C and A during a network partition**.

## Mental Model / Analogy

Imagine two bank branches sharing the same account records. Normally, they sync every 30 seconds. A fiber cut severs their connection.

**If they choose Consistency:** Branch A goes offline — "I can't guarantee your balance is up to date; please try the other branch." Customers can't transact here until the connection is restored.

**If they choose Availability:** Branch A stays open — "We'll process your withdrawal; we'll reconcile with the main branch when connectivity returns." Customers can transact, but if Branch B already processed a withdrawal from the same account, you've now overdrafted.

This is the fundamental trade-off. Neither choice is universally wrong — it depends on whether an overdraft or a closed branch is worse for your customers.

---

## Why You Can't Avoid P

In any distributed system (meaning: multiple networked machines), partitions are physically inevitable. You can reduce their frequency but not eliminate them. Any system that claims to guarantee both C and A without P is either:
- Running on a single machine (not distributed), or
- Claiming properties it can't actually deliver

So the real question every system architect asks is: **during a network partition, do we choose consistency or availability?**

---

## Choosing Consistency

When stale or conflicting data causes real harm, choose consistency. During a partition, refuse requests rather than risk returning outdated information.

**Examples:**
- **Ticket booking (Ticketmaster):** You can't let two users book seat 12A simultaneously. Return an error if you can't confirm seat availability in real time
- **E-commerce inventory:** If you can't confirm stock counts are current, reject the checkout rather than oversell
- **Financial systems:** An order book showing stale prices could cause massive losses; a trading halt is safer than wrong prices
- **Account balances:** If a user transfers $500, both sides of the transfer must see the correct balance immediately

**Technologies:** PostgreSQL, MySQL (with synchronous replication), Google Spanner (TrueTime), DynamoDB (strong consistency mode). These ensure reads always reflect the latest write — at the cost of refusing to serve reads during a partition.

---

## Choosing Availability

When users being unable to access your service is worse than them seeing slightly stale data, choose availability. During a partition, serve responses from whatever data you have.

**Examples:**
- **Social media profiles:** Seeing a 10-second-old profile photo is acceptable; being unable to load any profile is not
- **Netflix movie catalog:** A movie description that's 5 minutes stale is fine; a black screen is a cancelled subscription
- **Restaurant reviews (Yelp):** Slightly stale hours are okay; the app being down loses a customer
- **Ride-sharing driver locations:** Locations 2 seconds old are fine; showing no drivers is not

**Technologies:** Cassandra, DynamoDB (default multi-AZ mode), Redis Clusters with async replication. These replicate asynchronously, which means replicas might be slightly stale, but the system stays responsive during partitions.

---

## The Interview Question to Always Ask

When deciding C vs A, ask yourself: **"Would it be catastrophic if users briefly saw inconsistent data?"**

- **Yes → Choose Consistency:** booking systems, financial transactions, inventory management, anything involving real money or limited scarce resources
- **No → Choose Availability:** content browsing, social feeds, recommendations, anything where staleness for a few seconds is annoying but not harmful

---

## Advanced: Mixed Models

Real systems need different consistency guarantees for different features. You don't pick one model for the entire system.

**Ticketmaster example:**
- Seat booking endpoint → Consistency (can't double-book)
- Event listing/browsing → Availability (stale events for 30 seconds is fine)

**Tinder example:**
- Swipe matching → Consistency (matching the same person twice creates a terrible experience)
- Profile viewing → Availability (seeing a 1-minute-old bio is fine)

When designing a system, map each core feature to its consistency requirement rather than picking a blanket model.

---

## Consistency Levels Spectrum

Real systems don't flip between "fully consistent" and "fully available" — they operate on a spectrum:

| Level | What it means | When to use |
|-------|---------------|-------------|
| **Strong Consistency** | Every read reflects the latest write; most expensive | Financial transactions, inventory, booking |
| **Causal Consistency** | Causally related events appear in order to all nodes | Comment threads (reply always after the post), chat |
| **Read-Your-Own-Writes** | You always see your own updates immediately; others may lag | User profile updates, posts appearing in your own feed |
| **Eventual Consistency** | The system will converge to a consistent state given time | Follower counts, view counts, notification badges |

Most distributed databases operate at eventual consistency by default (Cassandra, DynamoDB) but offer stronger modes at higher cost. The right level depends on the specific operation.

---

## PACELC Extension

CAP only describes behavior during partitions. PACELC adds normal operation:

- **During a Partition:** choose between Availability and Consistency (this is CAP)
- **Else (normal operation):** choose between Latency and Consistency

During normal operation, synchronous replication (strong consistency) adds replica RTT to every write. Async replication (eventual consistency) is faster but means reads might see stale data.

Most databases have configuration knobs for this trade-off. Cassandra's consistency levels (`ONE`, `QUORUM`, `ALL`) are exactly this dial.

---

## Key Trade-offs

| If you choose Consistency | If you choose Availability |
|---------------------------|---------------------------|
| System may reject requests during partition | System serves requests even with stale data |
| No double-bookings, no overselling | May show inconsistent data temporarily |
| Higher latency (synchronous operations) | Lower latency (async replication) |
| Single leader often required | Active-active possible |

## When to Discuss in an Interview

Bring up CAP during the non-functional requirements phase, before diving into the design. The opening question: "Does this system need strong consistency, or is eventual consistency acceptable?"

This one question determines:
- Which database to choose
- How to configure replication
- Whether you can do active-active multi-region or need a single primary
- Whether you need distributed transactions (2PC, Saga)

## Common Gotchas

- **"We need both C and A"** — Not possible during a partition. Push back and ask which matters more when the network fails
- **Applying one model to the whole system** — A booking feature and a catalog-browse feature can (and should) have different consistency requirements
- **Confusing consistency with durability** — Durability (data survives crashes) is a different guarantee from consistency (all nodes see the same data); all production databases should be durable
- **Thinking CAP doesn't apply during normal operations** — PACELC reminds us that the latency-vs-consistency trade-off exists all the time, not just during failures

## Practice Questions

1. You're designing Ticketmaster. Walk through the consistency requirements for: (a) event search, (b) seat selection, (c) payment processing. Which CAP choice does each require and why?
2. Your team wants to deploy a global active-active database where users in the US and Europe can write simultaneously. What consistency challenges arise and how would you handle them?
3. A social media platform stores follower counts. These counts drift by up to 5% across replicas. Is this acceptable? When would it not be?
4. Cassandra with QUORUM writes and QUORUM reads guarantees you always read your most recent write. Walk through the math of why this works with 3 replicas.
5. You're building a distributed shopping cart. What happens if a user adds an item to their cart in Europe, the network partitions, and they also add from the US? Design a conflict resolution strategy.

## One-Line Summary

In any real distributed system, network partitions are inevitable, so the only real choice is Consistency vs. Availability during a partition — choose consistency for financial/booking systems where stale data causes real harm, and availability for content/social features where brief staleness is tolerable.
