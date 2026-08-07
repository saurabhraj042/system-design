# Networking Essentials

## The Core Idea

Every request your system processes travels through a networking stack. Understanding how that stack works — and the trade-offs between protocols at each layer — determines what technology you reach for when a design needs real-time communication, geographic scale, or resilience under failure.

In interviews, networking knowledge shows up in three questions: "What protocol should the client use to talk to the server?" "How do we load balance across many servers?" "What happens when a dependency fails?"

## Mental Model / Analogy

The networking stack is like a postal system with different shipping tiers. TCP is Priority Mail with delivery confirmation — slower, costs more overhead, but every letter is guaranteed to arrive in order. UDP is bulk mail — faster and cheaper, but some letters get lost and the post office doesn't retry. HTTP is the standardized envelope format that sits on top of TCP: it defines where to put the address (URL), how to state your request (GET/POST), and what the response looks like.

Higher layers in the stack are like additional services you can add: HTTPS adds tamper-evident packaging; WebSockets turn a letter exchange into an open phone line; gRPC is a standardized form submission system that both sides agree on in advance.

---

## The Foundation: TCP vs UDP

These are the two transport protocols you'll choose between for everything above them.

### TCP (Transmission Control Protocol)

TCP provides **reliable, ordered delivery**. Before any data flows, both sides complete a three-way handshake (SYN → SYN-ACK → ACK). The protocol handles:
- Packet acknowledgment and retransmission on loss
- In-order delivery (reorders packets that arrive out of sequence)
- Flow control (receiver can signal the sender to slow down)
- Congestion control (adapts to network conditions)

**Use TCP when:** data integrity and order matter — HTTP, database connections, file transfers, anything where a lost packet would corrupt the result.

**Cost:** The handshake adds round-trip latency before data flows. Retransmission adds more latency when packets are lost.

### UDP (User Datagram Protocol)

UDP is **connectionless and best-effort**. No handshake, no acknowledgment, no ordering guarantees. Packets may be lost, duplicated, or arrive out of order without notification.

**Use UDP when:** speed matters more than perfection, and the application can handle loss itself — live video/audio (a dropped frame is better than stalling), online gaming (a stale game state is better than waiting for retransmission), DNS queries (tiny payloads, client just retries if no response arrives).

**WebRTC is the primary case in system design** where you'll encounter UDP directly — it uses UDP for media streams because a brief video glitch is far preferable to a 500ms stall.

---

## HTTP / HTTPS

**HTTP** is the application-layer request-response protocol used by virtually every web API. The client sends a request (verb + URL + headers + optional body); the server responds (status code + headers + body).

Key versions:
- **HTTP/1.1:** One request per TCP connection at a time (with `keep-alive` for reuse). Suffers from head-of-line blocking — a slow request blocks all subsequent ones on that connection
- **HTTP/2:** Multiplexes multiple requests over a single TCP connection simultaneously. Much better for parallel asset loading. gRPC uses HTTP/2
- **HTTP/3:** Replaces TCP with QUIC (a UDP-based protocol). Eliminates head-of-line blocking at the transport layer. Faster connection establishment (0-RTT in some cases)

**HTTPS** = HTTP + TLS (Transport Layer Security). Adds encryption, authentication (server identity via certificates), and integrity. Non-negotiable for any production system.

---

## Application Protocols

### Long Polling

The client sends a request and the server holds it open until there's new data to return (or a timeout). When the response finally arrives, the client immediately sends another request.

**When to use:** Simple server-push with minimal infrastructure change. Works with standard HTTP, no new server support needed.

**Downsides:** Lots of open connections held on the server. Each response requires re-establishing the "polling" request. Not great at scale.

### gRPC

RPC framework from Google using Protocol Buffers (binary serialization) over HTTP/2. Compared to JSON REST:
- **Faster:** binary protobuf is significantly smaller and faster to serialize/deserialize than JSON
- **Strongly typed:** `.proto` files define the contract; client and server stubs generated automatically
- **Streaming support:** unary, server-streaming, client-streaming, bidirectional streaming modes
- **Browser limitation:** browsers don't natively support gRPC (no direct HTTP/2 from browser JS today); a gRPC-Web proxy is required

**Use gRPC for:** internal service-to-service communication, microservices backends, anywhere performance matters and you control both sides. Not for client-facing APIs unless you use gRPC-Web.

### Server-Sent Events (SSE)

SSE is built on top of regular HTTP. The server starts a response and sends data in chunks over time, keeping the single HTTP connection open:

```
data: {"event": "price_update", "ticker": "AAPL", "price": 182.50}

data: {"event": "price_update", "ticker": "AAPL", "price": 183.10}
```

The client uses the `EventSource` browser API which handles reconnection automatically — if the connection drops, it reconnects and passes the ID of the last message received so the server can resend missed events.

**Use SSE when:** the server needs to push updates to clients continuously and communication is one-directional (server → client). Live notifications, auction price updates, sports scores, stock tickers.

**Limitations:** SSE connections time out due to load balancers and proxies. The EventSource API handles reconnection but you need server-side logic to track message IDs and replay missed events. Some proxies buffer the response and batch everything, defeating the streaming purpose.

### WebSockets

WebSockets start as an HTTP request with an `Upgrade: websocket` header. After both sides agree, the protocol changes and you have a persistent, bidirectional binary channel. Either side can send data at any time without waiting for the other.

```
// Connection flow
Client → HTTP GET /socket (with Upgrade header)
Server → 101 Switching Protocols
Client ↔ Server: binary frames (any time, either direction)
```

**Use WebSockets when:** communication is high-frequency, persistent, and bidirectional — multiplayer games, collaborative editing (like Figma), chat systems with real-time replies, live collaborative whiteboards.

**Important interview warning:** Don't reach for WebSockets by default. Stateful connections at scale are expensive — you need sticky routing (user always goes to the same server) or a pub/sub layer to broadcast across servers. Load balancers need explicit WebSocket support. Justify why SSE isn't sufficient before proposing WebSockets.

**L4 load balancers are preferred for WebSockets** (they preserve the TCP connection); L7 load balancers terminate and re-establish the HTTP connection, which complicates the upgrade handshake.

### WebRTC

WebRTC enables direct peer-to-peer communication between browsers, primarily for audio/video. The only application-layer protocol that uses UDP directly.

**Connection setup (4 steps):**
1. Both clients connect to a **signaling server** (via WebSocket) to find each other and exchange connection details
2. Each client contacts a **STUN server** to discover its public IP and port (NAT traversal)
3. Clients exchange this connection info through the signaling server
4. Clients establish a direct peer-to-peer connection and send media

**STUN** ("Session Traversal Utilities for NAT") helps clients punch through NAT devices to get publicly routable addresses.

**TURN** ("Traversal Using Relays around NAT") is the fallback: if direct connection fails (strict firewalls), traffic is relayed through a TURN server. More expensive but always works.

**Use WebRTC for:** video/audio calling (Zoom, Google Meet, Discord video), conferencing. Most other "collaborative" use cases (shared documents, whiteboards) are better served by WebSockets because they need a central server anyway.

---

## Load Balancing

Horizontal scaling means adding more servers — but you need a way to route requests across them.

### Client-Side Load Balancing

The client itself decides which server to send each request to. This works via a **service registry**: the client fetches the list of available servers, picks one, and calls it directly.

**Advantages:** No extra network hop; the client can pick the fastest server.

**Examples:**
- **Redis Cluster:** clients connect to any node, learn the cluster topology (gossip protocol), then route subsequent requests directly to the right node based on hash slots
- **DNS load balancing:** the DNS resolver returns a rotated list of IP addresses; each client gets a different ordering and therefore hits different servers. TTL controls how stale the list can be

**Best for:** internal microservices (gRPC has client-side load balancing built in), or situations where clients are controlled and small in number.

### Dedicated Load Balancers

A server or hardware device that sits between all clients and all backends. Clients only see the load balancer's address.

**Advantage:** Immediate updates when servers are added/removed (no waiting for client cache TTLs), fine-grained routing control.

**Disadvantage:** One extra network hop per request; the load balancer itself can be a bottleneck (though hardware LBs handle hundreds of millions of requests/sec).

#### Layer 4 (L4) Load Balancers

Operate at the **transport layer** (TCP/UDP). Route based on IP address and port without inspecting the packet contents. The LB proxies TCP connections at the transport level.

- Fast and efficient (minimal packet inspection)
- Persistent TCP connections: once a client's TCP connection routes to server A, all subsequent requests in that session go to server A
- **Best for:** WebSocket connections (preserve the persistent TCP connection), high-performance scenarios where protocol inspection isn't needed
- **Example:** AWS Network Load Balancer (NLB)

#### Layer 7 (L7) Load Balancers

Operate at the **application layer** (HTTP). Terminate the incoming connection and open a new one to the backend. Can route based on URL paths, headers, cookies, or request content.

- Can route `/api/` requests to API servers and `/static/` requests to CDN/file servers
- Can ensure user sessions stick to the same backend (based on cookies)
- More CPU-intensive due to packet inspection
- **Best for:** HTTP-based traffic, A/B testing, path-based routing, anything that needs to inspect request content
- **Example:** AWS Application Load Balancer (ALB)

#### Health Checks and Algorithms

Load balancers monitor backend health:
- **TCP health check:** Can the server accept a new TCP connection?
- **HTTP health check:** Does `GET /health` return a 200 status?

If a server fails health checks, the LB stops routing to it until it recovers.

**Routing algorithms:**
| Algorithm | How | When to use |
|-----------|-----|-------------|
| **Round Robin** | Sequential distribution | Stateless backends, simple default |
| **Random** | Random distribution | Similar to round robin |
| **Least Connections** | Smallest active connection count | SSE/WebSocket servers (prevents one server from hoarding all long-lived connections) |
| **Least Response Time** | Fastest responding server | Mixed server speeds |
| **IP Hash** | Hash of client IP | Session persistence without sticky cookies |

For **persistent connection workloads** (WebSockets, SSE), use **Least Connections** — without it, the first server to receive connections ends up accumulating all of them while newer servers sit idle.

---

## Regionalization and Latency

Light travels through fiber optic cables at ~200,000 km/s. New York to London (~5,600 km) has a theoretical minimum round-trip of ~56ms just from physics — before any processing. For global users, this physical constraint drives architecture decisions.

**Content Delivery Networks (CDNs):** Edge servers positioned close to users worldwide. For cacheable content (static files, images, video), CDNs serve from the nearest edge location — turning 200ms cross-region latency into 5-20ms local latency. CDNs are read-through caches: on miss, they fetch from origin and store for future requests.

**Regional partitioning:** For data that's inherently local (rides for Uber in Miami, posts for users in Tokyo), split your infrastructure by region. Miami servers query Miami databases; Tokyo servers query Tokyo databases. Same-region queries are fast (<1ms); cross-region queries are avoided by design. Uber's insight: a Miami rider will never book a driver currently in New York, so Miami data never needs to be queried from a Tokyo server.

---

## Failure Handling

The fallacy of "the network is reliable" kills distributed systems. Always design assuming network calls will fail, be delayed, or return unexpected results.

### Timeouts and Retries

Set a timeout on every network call. If a response doesn't arrive in the expected window, give up and try again.

**Retries are safe only for idempotent operations.** GET requests are idempotent (fetching the same data twice causes no harm). A POST to charge a credit card is not — retrying it charges twice. For non-idempotent operations, use an **idempotency key**: a unique client-generated ID for the request. The server checks if it's seen this key before processing, and returns the previous result if so.

### Exponential Backoff with Jitter

Naive retries can make failures worse: if 10,000 requests fail simultaneously, they all retry at the same time, potentially overwhelming the recovering service again.

**Exponential backoff:** After each failed retry, wait progressively longer (1s → 2s → 4s → 8s...). This gives the failing service time to recover.

**Jitter:** Add randomness to the wait time (e.g., `wait = base * 2^attempt + random(0, 1000ms)`). This desynchronizes all the retrying clients so they don't all slam the service in the same instant. In interviews: "retry with exponential backoff and jitter" is the magic phrase.

### Idempotency

Build your APIs to be idempotent wherever possible. The retry problem becomes a non-problem: any request can be retried safely any number of times and the system ends up in the same state.

For operations that can't be inherently idempotent (creating a new record), add idempotency keys. The client generates a unique ID per request intention; the server uses it to detect and ignore duplicate submissions.

### Circuit Breakers

When a downstream service is failing, retries can create a **cascading failure**: the failing service gets hammered with retries just as it's trying to recover, preventing it from ever getting back up.

Circuit breakers prevent this by fast-failing requests to a known-bad service without even attempting the call:

**States:**
1. **Closed (normal):** All requests pass through. The circuit tracks failure rate
2. **Open (failing):** When failures exceed a threshold, the circuit "trips." All subsequent requests immediately fail without calling the dependency. The service can start recovering
3. **Half-open (testing recovery):** After a timeout, a few test requests are allowed through. If they succeed, the circuit closes. If they fail, it opens again

**Use circuit breakers on:** external API calls, database connections, any service-to-service call where failure is possible. They appear in deep-dive questions about reliability, disaster recovery, and handling dependency failures.

---

## Key Trade-offs by Protocol

| Protocol | Direction | Persistent? | Best for |
|----------|-----------|-------------|----------|
| HTTP REST | Client → Server (req/resp) | No | Standard APIs |
| Long Polling | Server → Client (delayed) | No | Simple push, low frequency |
| SSE | Server → Client (stream) | Yes | Notifications, live scores, price feeds |
| WebSockets | Bidirectional | Yes | Chat, gaming, collaborative editing |
| WebRTC | Peer-to-peer | Yes | Video/audio calling |
| gRPC | Client → Server (or streaming) | HTTP/2 | Internal service-to-service |

## When to Use What

- **Most things:** HTTP REST
- **Server needs to push events to client (read-only for client):** SSE
- **Real-time, high-frequency, bidirectional:** WebSockets (justify it)
- **Audio/video calling:** WebRTC (only use case)
- **Internal microservice communication:** gRPC
- **L4 load balancer:** WebSockets and other persistent-connection protocols
- **L7 load balancer:** Everything HTTP-based

## Common Gotchas

- **Proposing WebSockets when SSE suffices** — WebSockets require stateful infrastructure; SSE is simpler and usually enough for server-push notifications
- **Forgetting load balancer WebSocket support** — WebSocket upgrades can fail at L7 load balancers that don't support them; specify L4 (NLB) when designing WebSocket-heavy systems
- **Retrying non-idempotent operations without idempotency keys** — causes duplicate charges, double-booking, or duplicate records
- **No jitter on retries** — synchronized retries can cause a retry storm that crashes the recovering service
- **Skipping circuit breakers in dependency chains** — one slow downstream service without a circuit breaker can cause all its callers to hang, creating a cascading failure across the system

## Practice Questions

1. You're designing a live auction system. Bidders must see new bids within 1 second. Walk through the choice between long polling, SSE, and WebSockets. What's your final choice and why?
2. Your payment service makes calls to a third-party bank API that occasionally takes 30+ seconds to respond, causing your API to time out. Design a retry strategy including timeout values, retry count, backoff, and when to give up entirely.
3. A global video platform has servers in North America, Europe, and Asia. Users in Tokyo streaming a video hosted in Virginia experience 200ms latency. Walk through a CDN strategy to reduce this.
4. You're building a multiplayer game with 10,000 concurrent players. Each player sends position updates 60 times/second. Compare TCP vs UDP for this use case and justify your choice.
5. Your microservice architecture has 15 services. Service A depends on Service B which depends on Service C. Service C starts timing out. Without circuit breakers, what happens? Design the circuit breaker placement to prevent cascading failure.

## One-Line Summary

Networking essentials for system design: use HTTP REST by default, SSE for server-push and WebSockets for bidirectional real-time, L4 load balancers for persistent connections and L7 for HTTP routing, and always design for failure with timeouts, exponential backoff with jitter, idempotency keys, and circuit breakers.
