# API Gateway

## The Core Idea

When your backend is split into a dozen microservices, you don't want clients talking to each one directly — that's spaghetti. An API Gateway is the single door into your system: every request enters here, and the gateway decides where it goes. Along the way it can do cross-cutting work (auth, rate limiting, logging) so that each backend service doesn't have to.

## Mental Model / Analogy

Think of airport security. Every passenger — regardless of destination gate — flows through the same security checkpoint. The checkpoint validates your boarding pass (authentication), scans your bags (request validation), and then points you to Terminal A, B, or C (routing). Nobody walks directly onto the tarmac to find their own plane.

The key insight: the checkpoint does zero flying. Its only job is to vet and direct. An API Gateway is the same — thin, fast, and focused on routing and middleware, not business logic.

---

## How It Works

### Request Lifecycle

Every request passes through these stages in order:

```
Client Request
     ↓
1. Validate (well-formed? required headers?)
     ↓
2. Middleware (auth, rate limit, IP check...)
     ↓
3. Route (which backend service handles this?)
     ↓
4. Forward request (HTTP or gRPC translation if needed)
     ↓
5. Transform response (normalize format for client)
     ↓
6. [Optional] Cache response
     ↓
Client Response
```

### 1. Validation

First pass: reject obviously bad requests before they burn backend resources. Malformed JSON, missing required headers, oversized payloads — kill them here, return a helpful error.

### 2. Middleware

The gateway can run shared logic that every service would otherwise duplicate:

| Middleware | What it does |
|---|---|
| Authentication | Verify JWT tokens or API keys |
| Rate limiting | Enforce request caps per user/IP |
| IP allow/deny list | Block known bad actors |
| SSL termination | Decrypt TLS so backends speak plain HTTP internally |
| Logging / tracing | Stamp a correlation ID on every request |
| Response compression | Gzip before returning |

**Interview tip:** The three that actually matter in a system design answer are auth, rate limiting, and IP filtering. Just name them and move on — don't get lost here.

### 3. Routing

The gateway holds a routing table. Each rule maps a pattern (URL path, HTTP method, headers, query params) to a backend service and port:

```yaml
# Concept — not literal config
/api/orders/*  → order-service:8081
/api/users/*   → user-service:8080
/api/payments/*→ payment-service:8082
```

Lookup is O(1) or O(log n) — gateways are designed to do nothing slowly.

### 4. Protocol Translation

If a backend speaks gRPC internally but clients speak REST, the gateway can translate. Uncommon in practice but worth knowing it exists.

### 5. Response Transformation

Backend returns an internal representation; gateway transforms it into the shape clients expect. This lets you restructure your internals without changing the public API contract.

### 6. Caching

Response caching at the gateway makes sense for **non-personalized, stable** data. Cache strategies:
- **Full response caching**: entire response for a given URL stored in Redis/memory
- **Partial caching**: cache just the stable parts of a compound response
- **Invalidation**: TTL-based (expire after N seconds) or event-based (bust on writes)

Do NOT cache user-specific data at the gateway — cache misses become stale-data bugs.

---

## Scaling

**Horizontal scaling** is straightforward: gateways are stateless (no session data lives here), so spin up more instances behind a load balancer as traffic grows.

**Global distribution**: for latency-sensitive global products, deploy gateway clusters per region. Combine with GeoDNS to route users to the nearest cluster. Keep routing rules synchronized across regions via config management.

**Load balancing note**: the gateway load-balances requests to backend service instances (gateway → service). A separate load balancer typically sits in front of gateway instances (client → gateway). In interviews you can collapse this into a single "API Gateway / Load Balancer" box — don't get bogged down in the entry-point plumbing.

---

## Popular Options

| Category | Options |
|---|---|
| Managed cloud | AWS API Gateway, Azure API Management, Google Cloud Endpoints |
| Open source | Kong (NGINX-based, plugin ecosystem), Tyk (GraphQL native), Express Gateway (Node.js) |

---

## When to Use

**Use it when:**
- You have a microservices architecture — without a gateway, clients must discover and call each service directly
- You need centralized auth or rate limiting — better than duplicating that logic in 12 services
- Services use different protocols internally (gRPC) from clients (HTTP)

**Skip it when:**
- You have a simple monolith or a single backend service — a gateway just adds a network hop and a component to maintain
- All your clients are internal services that already know the topology

---

## Common Gotchas

- **Don't put business logic here** — if your gateway is making domain decisions, you've built a smart proxy, not a gateway. The moment you see `if user.tier == 'premium' then route to premium-service`, that logic belongs in a service
- **The gateway is a single point of failure** — run multiple instances; a gateway outage takes down everything
- **Middleware order matters** — auth must run before rate limiting by user; compression must run after response construction
- **Latency budget** — every stage adds a few milliseconds; monitor gateway P99 separately from service P99

---

## Practice Questions

1. You're designing a ride-hailing platform with services for matching, pricing, tracking, and payments. Where does the API Gateway fit in the architecture diagram, and what middleware does it need to run?
2. Your API Gateway is adding 50ms of latency to every request. Walk through the stages where this overhead might be coming from and how you'd diagnose each.
3. A competitor is building a system where each client SDK knows the address of each microservice directly. What are the tradeoffs of this approach vs. an API Gateway?
4. How would you implement request-level rate limiting (e.g., 100 requests/minute per API key) at the gateway layer? What data store would you use?
5. Your system has services in three geographic regions. Describe how global API Gateway distribution works and what happens when a user in Tokyo hits your US-east gateway.

---

## One-Line Summary

An API Gateway is the single entry point into a microservices system — its job is routing and middleware, not business logic — so say it will handle routing and middleware and move on.
