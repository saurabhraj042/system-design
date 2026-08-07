# Real-Time Updates

## The Core Idea

Getting live data to clients is fundamentally a two-hop problem. First hop: how does an update travel from the server to each connected client? Second hop: how does the server learn that an update happened in the first place? Both hops have multiple solutions, and the right combination depends on your latency requirements and how much state lives on each connection.

The single biggest mistake is jumping straight to WebSockets. Most systems need much less than that.

## Mental Model / Analogy

Think of a newsroom bulletin board. The first hop is the delivery method: you can walk by and check it periodically (polling), have an editor hold you at the door until there's news (long polling), subscribe to a one-way announcement feed (SSE), or have a two-way radio where the editor and you can both talk anytime (WebSockets). The second hop is how the editor gets the story: they can keep checking the wire service themselves (pull-based polling), always know which desk handles which beat (consistent hashing), or post stories to a shared message board that all interested editors are watching (pub/sub). Picking the right delivery method and the right intake channel independently gives you more flexibility than treating them as one decision.

---

## Hop 1: Client Update Protocols

### Simple Polling

The client sends a request every N seconds asking "anything new?" The server queries a database and returns whatever changed since the client's last-seen timestamp.

**Characteristics:** Completely stateless. Easy to scale — any server can handle any request. Works with any load balancer. The simplest architecture that can possibly work.

**When to reach for it:** When latency of several seconds is acceptable. Order status pages, activity summaries, dashboards that refresh every minute. If polling every few seconds meets your needs, don't add complexity — this approach avoids both hops entirely.

**Watch the math:** 1 million clients polling every 10 seconds = 100,000 reads/second against your database. Easy to miss in capacity planning.

**Disadvantages:** Latency equal to your polling interval. Wasted DB queries when nothing has changed.

---

### Long Polling

The client sends a request and the server intentionally holds it open — sitting on the response until there's data to return or a timeout fires (typically 15–30 seconds). When the response arrives, the client immediately sends another request.

**Characteristics:** Still stateless on the server side (no persistent connection). Works with standard HTTP infrastructure. Good match for infrequent but time-sensitive updates and for async process completion (waiting for a payment to confirm, waiting for a job to finish).

**The sequential update problem:** If two updates arrive 10ms apart while the server is processing the first response, the client will receive them 290ms apart — the round-trip time to re-establish the next long poll. Not suitable for high-frequency update streams.

---

### Server-Sent Events (SSE)

The server starts an HTTP response and keeps it open, streaming chunks of data as they become available using `Transfer-Encoding: chunked`. The browser's `EventSource` API handles this natively, auto-reconnecting when the connection drops and sending the last received event ID so the server can replay missed events.

**Characteristics:** One-way (server → client only). Built on plain HTTP — works through most proxies and load balancers without special configuration. Popular for AI token streaming. Connection lifetime is typically 30–60 seconds before client reconnects.

**Infrastructure note:** Some proxies buffer chunked responses and batch them, defeating the streaming effect. At the load balancer, use "Least Connections" routing so that long-lived SSE connections distribute evenly rather than piling up on whichever server was added first.

**When to use:** Notification feeds, live scores, price tickers, AI output streaming — any scenario where the server needs to push a continuous stream and the client doesn't need to send data back.

---

### WebSockets

The client initiates an HTTP request with an `Upgrade: websocket` header. After a 101 Switching Protocols handshake, both sides have a persistent, bidirectional TCP channel. Either side can send a frame at any time without waiting for the other to request first.

**Characteristics:** Bidirectional, persistent, low-overhead per message (no HTTP headers on each frame). Requires stateful infrastructure — the client is pinned to one server for the lifetime of the connection.

**Load balancer requirement:** Use a Layer 4 (TCP-level) load balancer, not a Layer 7 HTTP load balancer. L7 LBs terminate and re-establish connections, which breaks the upgrade handshake. Use "Least Connections" algorithm so new connections distribute across servers rather than all routing to servers that came up first.

**Architecture tip:** Terminate WebSocket connections at a dedicated WebSocket service tier. Everything else in your system — auth, data services, APIs — can remain stateless. The WebSocket layer handles the statefulness; the rest of the system doesn't need to know about it.

**When to use:** Chat, multiplayer games, collaborative document editing, any scenario requiring both high-frequency updates AND client-to-server messages. Don't reach for WebSockets when SSE is sufficient — stateful connections are expensive to scale.

**Reconnection handling:** On reconnect, the client needs to catch up on any updates it missed. Track a sequence number or timestamp per connection and replay missed messages from a Redis stream or equivalent.

---

### WebRTC

WebRTC enables direct browser-to-browser (peer-to-peer) communication, primarily for audio and video. Unlike every other option here, it uses UDP rather than TCP — a dropped frame is far preferable to a 500ms stall while TCP retransmits.

**Connection setup (four steps):**
1. Both peers connect to a signaling server (via WebSocket or SSE) to find each other
2. Each peer contacts a STUN server to discover its publicly routable IP and port (NAT traversal via "hole punching")
3. Peers exchange this connection info through the signaling server
4. Peers establish a direct connection and stream data

**STUN** discovers public addresses. **TURN** is the fallback relay when direct connection fails (strict firewalls, symmetric NAT) — traffic bounces through a central server.

**When to use:** Video/audio calling (the only valid case in most interviews). Peer-to-peer data sharing when you need to reduce server bandwidth — clients establish direct connections and offload traffic from your servers. The signaling server itself is lightweight and needs only a standard WebSocket connection.

**When not to use:** Any scenario where a central server needs to coordinate, store, or audit the data. Most "collaborative" features (shared whiteboards, live document edits) still benefit from a central server and WebSockets is simpler.

---

### Decision Flowchart

```
Latency sensitive (sub-second)?
├── No → Simple Polling (done, minimal complexity)
└── Yes → Frequent bidirectional communication?
         ├── No → SSE (one-way push, simpler)
         └── Yes → Peer-to-peer audio/video?
                  ├── Yes → WebRTC
                  └── No → WebSockets
```

---

## Hop 2: How the Server Gets Triggered

When a client uses long polling, SSE, or WebSockets, it maintains a persistent connection to one specific server. That server needs to be notified when an update relevant to that client occurs — even if the update originated on a completely different server. There are three patterns for this.

### Pull-Based Polling (Server Polls a DB)

The simplest second-hop approach: the update source writes events to a shared database, and the client-facing servers query that database on each client poll.

```
Update Source → DB → Server (queries on poll) → Client
```

This decouples the producer from the consumer entirely. The client-facing server doesn't need to know where updates come from or which other servers exist. The downside is that you've traded real-time delivery for polling latency, and every client poll adds DB read load.

**Best for:** Simple polling architectures where you've already accepted latency. Doesn't make sense with SSE or WebSockets because you lose the benefit of the persistent connection.

---

### Push via Consistent Hashing

With SSE and WebSocket connections, the server that holds the connection must be the one to write to it — you can't just send the message to a random server. When an update needs to reach a user, you need to find which server holds that user's connection.

**Simple hashing approach:** Assign each user to server `hash(userId) % N`. A coordination service (Zookeeper, etcd) maintains the server list and assigns each server a number 0 through N-1. When a message needs to reach User C, the update service hashes User C's ID to find the right server and delivers the message there directly.

**The scaling problem:** If you add or remove a server, N changes, and `hash(userId) % N` produces different results for nearly every user. Everyone has to reconnect to a new server — expensive churn that disrupts all active sessions.

**Consistent hashing solution:** Place both servers and users on a virtual hash ring (0 to 2^32). Each user connects to the nearest server clockwise around the ring. When a server is added or removed, only users in the adjacent ring segment need to migrate — typically 1/N of all users instead of nearly all of them.

**Scaling operations require coordination:**
1. Signal the start of a scaling event; record old and new server assignments
2. Gradually disconnect affected clients and have them reconnect to their new server
3. During the transition window, send messages to both old and new server
4. Signal completion and update the coordination service

**When to use:** When each connection carries significant per-user state that's expensive to rebuild — a collaborative editing session that loads a document, applies all pending operations, and syncs with collaborators. Consistent hashing keeps all that state on one server while allowing scaling. If your connections are lightweight (just forwarding messages, no per-connection state), use pub/sub instead.

---

### Push via Pub/Sub

A message broker (Redis pub/sub, Kafka) acts as the intermediary. Client-facing "endpoint servers" are lightweight — they hold connections and subscribe to the broker, but carry no connection-specific state. Any endpoint server can serve any client.

**Connection flow:**
1. Client connects to any endpoint server (standard "Least Connections" LB works fine — no routing required)
2. Endpoint server creates a topic subscription for that client in the pub/sub service

**Update flow:**
1. Update server publishes the message to the relevant topic
2. Pub/sub service broadcasts to all subscribed endpoint servers
3. The endpoint server that holds that client's connection forwards the message over the existing connection

**Architecture benefit:** Endpoint servers are interchangeable. You don't need to know which server holds a given client — you publish to a topic and the pub/sub layer handles delivery. This keeps your endpoint tier stateless enough to scale with a simple load balancer.

**Scaling the pub/sub layer:** Redis pub/sub becomes a single point of failure. Redis Cluster shards subscriptions by key across multiple nodes, scaling both throughput and reliability. Each endpoint server connects to all nodes in the cluster, so messages published to any shard reach the right endpoint server.

**When to use:** Broadcasting updates to many clients when you don't need to maintain per-connection state. Chat applications, notification systems, live feeds. Simpler operationally than consistent hashing.

**Latency impact of pub/sub:** Adds roughly <10ms compared to direct delivery. Usually acceptable.

---

## Deep Dives

### Connection Failures and Reconnection

WebSocket connections break silently — a client might believe it's still connected while the server has already cleaned up the connection object. Implement heartbeat pings (server → client every 30s, client responds) to detect "zombie" connections quickly.

On reconnect, clients must catch up on missed events. Track a sequence number or timestamp per user. On reconnect, the client sends its last-received sequence number; the server replays everything since. Redis Streams are a natural fit — they're an ordered log per key with consumer group semantics.

### The Fan-Out Problem (Celebrity Accounts)

When a user with millions of followers posts an update, naively writing that update to each follower's feed creates millions of individual writes that can crash your system.

Better approach: cache the update once at the source and distribute through a hierarchical tree. Regional servers pull the update from origin and push to their local clients. Write processors aggregate events and pass them to a root processor, which fans out to broadcast nodes, which deliver to end clients. This reduces the load on any single component and prevents the O(N_followers) spike from reaching your database.

### Message Ordering Across Servers

When multiple servers handle updates, two messages sent milliseconds apart may arrive out of order if they travel different network paths. For most product-style interviews, the right answer is to funnel related messages through a single server or Kafka partition — total ordering within a partition is guaranteed, and you trade a small amount of scalability for a much simpler system. Vector clocks exist for fully distributed ordering, but they belong in infrastructure-level designs, not "Design a Chat App" interviews.

---

## When to Use

- **Chat applications:** WebSockets for bidirectional communication, pub/sub to route messages to the right endpoint server, Redis Streams to replay missed messages on reconnect
- **Live comments / high fan-out events:** SSE for delivery, hierarchical aggregation to prevent write storms
- **Collaborative document editing:** WebSockets for low-latency bidirectional updates, consistent hashing to co-locate document state on one server, CRDTs or Operational Transforms for conflict resolution
- **Live dashboards and analytics:** SSE for one-way data push, polling if updates are infrequent enough
- **Multiplayer games:** WebSockets for server-coordinated state, WebRTC for peer-to-peer when server bandwidth is the bottleneck

## When NOT to Use

Avoid real-time update infrastructure when a polling model meets the requirement. Polling eliminates both hops — no persistent connection management, no pub/sub routing, no reconnection logic. Senior interviewers value simplicity; reaching for WebSockets when a 5-second polling interval satisfies the user need is a signal of over-engineering, not sophistication.

---

## Key Trade-offs

| Protocol | Direction | Stateful? | Best for |
|----------|-----------|-----------|---------|
| Simple polling | Pull | No | Delay-tolerant updates |
| Long polling | Server push (delayed) | No | Infrequent push, async completion |
| SSE | Server → client | Yes (connection) | Notification feeds, AI streaming |
| WebSockets | Bidirectional | Yes (connection) | Chat, games, collaborative editing |
| WebRTC | Peer-to-peer | Yes (connection) | Audio/video calling only |

| Second-hop pattern | State location | Best for |
|-------------------|---------------|---------|
| Pull-based polling | Database | Simple polling architectures |
| Consistent hashing | Endpoint servers | Per-connection heavy state |
| Pub/sub | Broker | Lightweight broadcast, easy scaling |

## Common Gotchas

- **Defaulting to WebSockets** — SSE is simpler for one-way push and sufficient for most notification use cases
- **Forgetting the read load of polling** — 1M clients polling every 10s = 100k read TPS; this surprises people
- **Using L7 load balancer with WebSockets** — breaks the upgrade handshake; specify L4 (NLB)
- **Not planning for reconnection** — what a client misses during a disconnect must be catchable on reconnect; design this from the start
- **Consistent hashing without a migration plan** — be ready to explain how you handle scaling events without dropping messages or connections
- **Pub/sub as a single point of failure** — address Redis Cluster or Kafka replication when the interviewer asks about availability

## Practice Questions

1. Design a live sports score feed for 10 million concurrent viewers. Walk through your choice of client protocol and server-side trigger mechanism. What changes if viewers can also post comments?
2. You're building a chat application. A message sent by User A needs to reach User B, who is connected to a different server. Walk through two approaches to route the message to the right server. What are the trade-offs?
3. A WebSocket server has 50,000 active connections. You need to scale from 3 to 5 servers. Using consistent hashing, walk through what happens to connections during the scaling event and how you avoid dropped messages.
4. Your notification system polls a database every 5 seconds for 2 million mobile clients. At what point does this become problematic, and what would you replace it with?
5. A collaborative whiteboard has 500 users editing simultaneously. Every cursor move must be visible to all other users within 100ms. Design the real-time update architecture from client protocol through server-side delivery.

## One-Line Summary

Solve real-time updates in two hops: start with SSE for one-way server push or WebSockets for bidirectional, then route updates to the right server via pub/sub (lightweight connections, broadcast) or consistent hashing (heavy per-connection state); default to simple polling when latency requirements allow it.
